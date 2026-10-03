---
tier: tale
title: Isolate detached agents from the launching terminal's systemd scope
goal:
  One OOM kill (or stop) of the TUI's tmux or terminal systemd scope no longer SIGTERMs
  every agent launched from it. Detached SASE work escapes into its own scope with
  OOMPolicy=continue, and external kills are recorded as external, with OOM evidence.
size: medium
proposed_by: bbugyi200.apollo.4t
create_time: 2026-10-03 12:36:45
status: wip
---

# Plan: Isolate detached agents from the launching terminal's systemd scope

## Incident and root cause

On 2026-10-03 at 16:24:28Z, the kernel OOM killer on apollo (31 GB RAM, no swap) killed
a process inside the tmux pane scope `tmux-spawn-<id>.scope` that hosted the ACE TUI.
The user journal shows the sequence:

```text
tmux-spawn-1a05ddb8-….scope: A process of this unit has been killed by the OOM killer.
tmux-spawn-1a05ddb8-….scope: Stopping timed out. Killing.            (90 s later, SIGKILL)
tmux-spawn-1a05ddb8-….scope: Failed with result 'oom-kill'.
tmux-spawn-1a05ddb8-….scope: Consumed … 29.0G memory peak
```

The default `DefaultOOMPolicy=stop` made systemd stop the whole scope after one OOM
kill, so it sent SIGTERM to every process in that cgroup. That included the TUI and all
three agents launched from it: `4s.f0` (codex, 10 s into its turn), `4r.f0` (codex), and
`bob-cli-3u.land` (claude). All three ended with `outcome: killed`. None had a
`.sase_user_kill_pending` marker, and none of their own commands sent the signal.

The code-level cause is `src/sase/detach_scope.py`. It wraps a detached launch in
`systemd-run --user --scope` only when the launching process already runs inside a
SASE-owned unit (`sase.service` or `sase-*`). That rule came from the service-host epic
(sase-11y.1, `plan:202609/service_host_1.md`), which targeted `systemctl --user restart`
of the service host. A launch from a terminal or tmux pane keeps the "ordinary
new-session detach": `setsid` moves the process to a new session but leaves it in the
launcher's cgroup. Every agent launched from the TUI therefore shares the fate of the
TUI's tmux pane scope.

There is a second gap. SASE's own transient scopes inherit `OOMPolicy=stop`. So even an
agent already in its own `sase-agent-*` scope is torn down entirely when the kernel
OOM-kills one `rustc` or test binary in its tree.

A third gap is diagnostic. `_handle_killed_iteration` in
`src/sase/axe/run_agent_exec.py` treats a SIGTERM with no user-kill intent and no
handoff marker as a user kill (`AGENT_KILLS{reason="user"}`, `outcome: killed`, log line
`agent was killed`). As a result, nothing told the user that this was an external
teardown.

## Goals

1. Long-lived detached work escapes into its own transient scope on Linux whenever the
   launcher runs under a SASE-owned unit **or** the user's systemd manager is reachable.
   That covers tmux pane scopes, terminal `app-*.scope` units, and ssh `session-N.scope`
   units.
2. SASE transient scopes and the SASE service unit use `OOMPolicy=continue`, so one
   OOM-killed process no longer stops the whole unit.
3. A SIGTERM that is neither a user kill nor a handoff is recorded as an **external**
   kill. When the runner's cgroup recorded OOM kills during the run, that evidence is
   recorded too.

## Non-goals

- Reducing memory pressure itself: swap, per-agent `MemoryHigh=`/`MemoryMax=`, or
  admission based on memory. Per-agent scopes make those possible later; they are out of
  scope here.
- Automatically reviving externally killed agents.
- TUI or fleet presentation of the new kill-source field. The sase-core agent-scan wire
  does not use `deny_unknown_fields`, so additive `done.json` keys are safe and can be
  surfaced in a follow-up.
- macOS behavior, which already always detaches with `setsid`.

## Design

### 1. Escape policy in `src/sase/detach_scope.py`

Keep the existing early returns: the disable env vars (`SASE_DETACH_SCOPE_DISABLE`, plus
the legacy `SASE_AXE_DISABLE_SYSTEMD_SCOPE`), macOS, and non-Linux. On Linux, change the
escape decision:

- Escape when `systemd-run` is on `PATH` **and** one of these holds:
  1. The parent unit is SASE-owned. This is the existing rule, unchanged.
  2. The user's systemd manager is reachable. The check passes if either:
     - the launching process's cgroup-v2 path contains a `user@<uid>.service` component
       for `uid == os.getuid()`, or
     - `<runtime_dir>/systemd/private` exists, where `runtime_dir` is
       `$XDG_RUNTIME_DIR`, falling back to `/run/user/<uid>`. `systemd-run --user` uses
       this private socket.
- Otherwise keep today's no-op command (containers, CI without a user bus, no systemd).
- Parse the full cgroup path for the `user@<uid>.service` check. Today's
  `_unit_from_cgroup_path` returns only the innermost unit, so add a small helper next
  to it. Make the runtime dir injectable the same way `proc_root` is (for example a
  `runtime_dir: Path | None = None` keyword), so tests never touch the real `/run/user`.
- Add an `escape_reason: str | None` field to `_DetachScopeCommand` with the values
  `"sase_unit"`, `"user_manager"`, or `None`. Keep `parent_unit` as is.

Changes to the `systemd-run` argv:

- Add `--property=OOMPolicy=continue` when the systemd version is at least **243**.
  `systemd.scope(5)` documents scope `OOMPolicy=` as "Added in version 243"; apollo runs
  systemd 255.
- Detect the version by parsing the first line of `systemd-run --version`
  (`systemd NNN (…)`). Cache the result per process, keyed by the resolved `systemd-run`
  path (for example with `functools.cache`), and bound it with a short subprocess
  timeout. If the probe fails, times out, or cannot be parsed, omit the property. Never
  fail the launch because of the probe.
- Expose the probe as a patchable helper so unit tests can force a version.

No call site should need code changes, because every detach point already goes through
`detach_scope`:

- `agent/launch_spawn.py`
- `agent/launch_admission_coordinator.py`
- `procs/spawn.py`
- `monitor/spawn.py`
- `ace/hooks/execution.py`
- `ace/scheduler/workflows_runner/starter.py`
- `ace/scheduler/mentor_runner.py`
- `file_hooks/dispatch.py`
- `integrations/chat_install.py`
- `service/control.py`

Still, audit each one. Callers that branch on `launch.method == "systemd-run"` (the
pid-file paths in `procs/spawn.py` and `monitor/spawn.py`, and `chat_install.py`)
already run escaped under `sase.service`; confirm that their pid-identity and lock-fd
assumptions also hold when the parent is a tmux or terminal scope. `systemd-run --scope`
execs in place, and `test_live_scope_pid_unchanged_through_detach` plus
`test_live_scope_preserves_lock_fd_through_detach` cover that.

The plan adds no runtime fallback for a `systemd-run` invocation that fails before exec.
The reachability check is the guard, and `SASE_DETACH_SCOPE_DISABLE=1` remains the
documented escape hatch. If the audit finds a call site where a failed scope creation
would silently lose work, record it as a follow-up rather than widening this tale.

### 2. Service unit `OOMPolicy`

In `src/sase/service/platform_units.py::render_systemd_unit`, add `OOMPolicy=continue`
to the `[Service]` section, next to `KillMode=mixed`. The host already supervises and
restarts its own procs, so one OOM-killed routine child should not stop the whole host.
If the host main process itself is OOM-killed, the unit still fails and
`Restart=on-failure` restarts it. Installed units are compared by `content_current`
(`service/platform_definition.py`), so existing installs show as stale until the normal
service install or init path rewrites them. Confirm that this is what happens and
mention it in the docs.

### 3. External-kill provenance

- Put the classification in one helper that every kill path uses, for example
  `classify_runner_kill(artifacts_dir, ...) -> KillProvenance` in a new small module
  under `src/sase/axe/`. `_handle_killed_iteration` (`run_agent_exec.py`), the retry
  path (`run_agent_exec_retry.py`, which sets `loop_outcome = "killed"`), and
  `run_agent_runner.py` (which sets `exec_outcome = "killed"`) must all report the same
  provenance.
  - `user`: a user-kill intent marker exists (today's first branch).
  - `handoff`: a plan, questions, monitor, gate, or pipe marker predates the kill (the
    existing handoff branches; their outcome is unchanged).
  - `external`: everything else.
- For the external fallthrough, increment `AGENT_KILLS.labels(reason="external")`
  instead of `reason="user"`, and update the metric's help text in
  `src/sase/telemetry/metrics.py`. Keep `outcome: "killed"` unchanged, because
  lifecycle, retry, and UI consumers key on it.
- Add the additive keys `kill_source` (`"user"` or `"external"`) and, when available,
  `kill_evidence` to `done.json` through the existing done-marker writer
  (`runner_artifacts.write_done_marker` / `run_agent_exec_markers.py`). Omit both keys
  for runs that were not killed.
- OOM evidence:
  - At agent-runner start, where the runner installs its SIGTERM handler, snapshot the
    runner's cgroup-v2 path from `/proc/self/cgroup` and the hierarchical `oom_kill`
    counter from `/sys/fs/cgroup/<path>/memory.events`.
  - When classifying an external kill, re-read the counter. If it increased, set
    `kill_evidence = {"cgroup_unit": <unit>, "oom_kill_delta": <n>}`.
  - Every read is best effort: a missing cgroup v2 mount, permission errors, or parse
    failures mean no evidence and never raise.
  - Make the sysfs and proc roots injectable for tests.
- After the existing `Received SIGTERM - agent was killed` line, print one
  classification line from the kill-handling code, not from the signal handler. For
  example:
  - `Kill source: external (no user-kill intent or handoff marker)`
  - `…; cgroup tmux-spawn-….scope recorded 1 OOM kill since start` when OOM evidence
    exists.

### 4. Docs

- Rewrite the detach-scope paragraph in `docs/axe.md` (currently "Long-lived work that
  SASE detaches … Outside a SASE-owned unit, children keep the ordinary new-session
  detach …"). It must cover:
  - the new escape condition (SASE-owned unit **or** reachable user manager)
  - `OOMPolicy=continue` on SASE scopes and the service unit
  - why the change was made: one OOM kill in a terminal or tmux scope used to SIGTERM
    every agent launched from it
  - the `SASE_DETACH_SCOPE_DISABLE=1` escape hatch
- Update any matching wording in `docs/configuration.md` and `docs/tool.md`, which both
  mention the scope or cgroup behavior.
- Document `kill_source` / `kill_evidence` wherever the done-marker fields are
  described, if such a place exists.

## Tests

- `tests/test_detach_scope.py`:
  - Keep a no-op case for a parent outside any user manager. Use a cgroup path not under
    `user@<uid>.service` and an injected runtime dir that lacks `systemd/private`. This
    replaces the current assumption in `test_detach_scope_noops_outside_sase_cgroup`.
  - Escapes from `0::/user.slice/user-1000.slice/user@1000.service/tmux-spawn-abc.scope`
    with `os.getuid` patched to 1000. Assert that the argv starts with
    `systemd-run --user --scope`, that `escaped` is true, that
    `escape_reason == "user_manager"`, and that `parent_unit == "tmux-spawn-abc.scope"`.
  - Does not use the cgroup rule when the path names `user@1001.service` but the uid is
    1000 and no socket exists.
  - Escapes from `session-2.scope` when the injected runtime dir contains
    `systemd/private`.
  - The SASE-owned-unit path still escapes, with `escape_reason == "sase_unit"`.
  - `--property=OOMPolicy=continue` is present when the patched version probe returns
    255, and absent for 241, `None`, or an unparsable probe.
  - The disable env vars still win over the user-manager rule.
- Extend the live cgroup test, or add a sibling. When the test process is under the user
  manager (skip otherwise), the escaped child's cgroup must differ from the parent's.
  From inside the child, while it is still alive, run
  `systemctl --user show <own unit> -p OOMPolicy` and expect `OOMPolicy=continue` when
  the version is at least 243.
- Kill provenance: run `_handle_killed_iteration` (and the retry and runner paths via
  the shared helper) with no markers. Expect `external`,
  `AGENT_KILLS{reason="external"}`, and the additive `done.json` keys. With a user
  intent marker, expect `user`. With a handoff marker, expect handoff behavior
  unchanged. Test OOM evidence with a fake sysfs root where `oom_kill` goes from 0 to 1,
  and test that no evidence is recorded when the file is missing.
- `tests/service/test_service_platform_linux.py`: assert that `OOMPolicy=continue` is in
  the rendered unit.
- The session-wide `tests/conftest.py` guard keeps disabling detach scopes for ordinary
  tests. Do not loosen it.

## Verification

- Run the repo's standard verification as described in the `lint_and_test` reference
  memory.
- Manual check on a Linux host with a user manager:
  1. From a scratch tmux pane, run `sase run` (or launch from the TUI).
  2. Confirm that `/proc/<runner pid>/cgroup` names a `sase-agent-*.scope` and that
     `systemctl --user show <that scope> -p OOMPolicy` prints `OOMPolicy=continue`.
  3. Run `systemctl --user stop <scratch pane's tmux-spawn scope>` and confirm the agent
     keeps running.
  4. Separately, SIGTERM a scratch agent runner by hand. Confirm that `done.json` gets
     `kill_source: "external"` and that the runner log prints the classification line.
