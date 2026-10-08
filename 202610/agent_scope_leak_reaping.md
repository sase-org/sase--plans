---
tier: epic
title: Agent scopes reap every process an agent leaks
goal: 'No process an agent starts outlives its agent runner unless SASE deliberately
  escaped it into its own systemd scope. Leftovers die when the runner exits (and
  between in-process successor turns), and a five-minute backstop reaps sase-agent
  scopes whose runner died without cleaning up, while shared user daemons (ssh-agent,
  gpg-agent, ssh ControlMaster, tmux server) are never killed.

  '
decisions:
  scope_decision_record:
    ask: Add a decisions-web record stating that an agent's scope bounds every process
      the agent starts?
    memory:
    - decisions
    default: false
    answer: false
  turn_sweep:
    ask: Also sweep leaked processes between in-process successor turns (plan to coder,
      pipe handoff)?
    default: true
    why: Otherwise a planner's leaked loop keeps spinning through an hours-long %auto
      coder turn
    answer: true
phases:
- id: escape-helpers
  title: Escape long-lived SASE helpers from the agent scope
  depends_on: []
  size: small
  description: 'escape-helpers: route the four fire-and-forget spawns reachable from
    an agent (background trash delete, goals fetch worker, federation worker daemon,
    tmux session bootstrap) through detach_scope so the new sweep can never kill them,
    with detach-scope worker tests.'
- id: runner-teardown
  title: Agent runner sweeps its own scope
  depends_on:
  - escape-helpers
  size: medium
  description: 'runner-teardown: add the shared scope-sweep module, the agent_scope_teardown
    config block, and the runner hooks that kill non-descendant, non-spared processes
    in the runner''s own sase-agent scope before shutdown finalization and between
    in-process successor turns.'
- id: scope-reaper
  title: Orphaned agent scope reaper job
  depends_on:
  - runner-teardown
  size: medium
  description: 'scope-reaper: add a checks-routine job that discovers sase-agent scopes
    with no live runner and sweeps their non-spared processes, with a dry-run core
    API, docs, and a live systemd test.'
proposed_by: bbugyi200.apollo.5s
decided_by: auto
create_time: 2026-10-08 06:37:31
status: done
bead_id: sase-1i4
---

- **PROMPT:** [prompts/202610/agent_scope_leak_reaping.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/agent_scope_leak_reaping.md)
- **BEAD:** [sase-1i4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1i4/README.md)

# Plan: Agent scopes reap every process an agent leaks

## Background

The incident report `research:202610/athena_cpu_saturation_orphaned_load_loops.md`
(bob-cli research sidecar) found athena pinned at 64/64 cores and 95 °C for about 14
hours. Agent `bob-cli-5k.5` (muse provider) started 64 `(while :; do :; done) &` busy
loops to load-test a fix. Its shell tool timed out and killed only the leader shell, so
the loops were reparented to PID 1. The agent's `ps`-based cleanup missed them because
their `args` read `/bin/sh -c …`. The runner then exited (SIGTERM monitor handoff), and
nothing in SASE ever stopped the agent's `sase-agent-<pid>-<ns>.scope`, which stayed
"active (running)" holding only the loops (about 754 CPU-hours).

This is not a one-off. A read-only survey of apollo during planning found two more
`sase-agent-*` scopes with no live runner:

- A scope about 3.9 days old holding a leaked
  `/bin/sh …/agent-tmp/…/waiting-worker --internal-root-worker …` polling loop
  (`sleep 0.05`, 16 CPU-minutes so far). It also keeps that run's launch scratch alive.
- A scope about 20 hours old holding `ssh-agent -a ~/.ssh/agent.sock`, the user's shared
  SSH agent, which an agent's shell lazily started. This must **not** be killed.

Also on apollo, `ssh: ~/.ssh/cm-git@ssh.github.com:443 [mux]` is a ControlMaster
(`ControlMaster auto`, `ControlPersist 10m` in the chezmoi-managed ssh config) that
every git-over-ssh operation multiplexes through. Whichever process first runs `git` or
`ssh` owns the master in its cgroup, so if an agent happens to start it, killing it
would cut other agents' in-flight fetches and pushes.

### Facts the design relies on (verified in code)

- `src/sase/agent/launch_spawn.py` wraps every runner launch in
  `detach_scope(..., unit_prefix="sase-agent")`
  (`systemd-run --user --scope --collect --unit=sase-agent-<launcher pid>-<time_ns>`).
  `systemd-run --scope` execs the runner in place, so the runner pid is the scope's
  first process. Its cmdline has `run_agent_runner.py` as `argv[1]`.
- Every successor that is a separate process already escapes into a **new** scope,
  because `detach_scope` always escapes when the caller sits in a SASE-owned unit. That
  covers retry and continuation respawns, gate and monitor follow-ups, `sase run`
  children, monitors (`sase-monitor`), procs and detached ToolRuns (`sase-proc`), bead
  sync, attachment upload, and file hooks. In-process successors (pipe handoff, plan
  re-plan, the accepted-plan coder under `%auto`, auto-answered questions) loop inside
  `_run_execution_loop_bound` in `src/sase/axe/run_agent_exec.py`.
- Handoff commands (`kill_agent_runner_group` in `src/sase/main/utils.py`) write their
  marker, `killpg` the runner group, and `sys.exit(0)` immediately, so nothing durable
  runs in them after the SIGTERM.
- `main()` in `src/sase/axe/run_agent_runner.py` funnels every non-SIGKILL exit through
  one outer `finally` (`finalize_runner_shutdown`, then `cleanup_launch_scratch`).
  `finalize_runner_shutdown` writes `done.json` and releases the workspace.
- Four fire-and-forget spawns reachable from an agent do **not** escape today (see the
  escape-helpers phase).
- No existing code enumerates systemd scopes or reads `cgroup.procs`. Reusable pieces:
  - `src/sase/agent/process_tree.py`: `read_process_row`, `read_process_table`,
    `descendants_of`, zombie handling.
  - `src/sase/core/process_identity.py`: `process_identity_token` for pid pinning.
  - `src/sase/detach_scope.py`: `_process_systemd_unit`, `_unit_from_cgroup_path`,
    `_detach_scope_disabled`.
  - The TERM, grace, then KILL loop in `src/sase/llm_provider/_subprocess_reap.py`
    (`_sweep`).

## Design

**Contract:** an agent runner's `sase-agent-*` scope bounds the lifetime of every
process the agent starts. Work that must outlive the runner must escape through
`detach_scope` into its own scope. Anything still in the scope that is not the runner,
not one of the runner's live descendants, and not a spared shared daemon is a leak and
gets terminated.

The same pure selection rule is used at three points:

1. **Runner exit:** the runner sweeps its own scope as the first step of its shutdown
   `finally`, so a released workspace never hands the next agent a predecessor's live
   leftovers.
2. **Turn boundary:** the runner also sweeps between in-process successor turns.
   > [!decision] turn_sweep
3. **Backstop reaper:** a five-minute `checks` job sweeps scopes whose runner is gone
   (SIGKILL, OOM, crash, or a stuck provider that was still a runner descendant at
   exit).

### Selection rule (shared by all three)

Given the scope's members (from `cgroup.procs`, skipping zombies and unreadable pids):

- **Protected:** the protect root, when one is given (the runner itself for in-runner
  sweeps), plus its live `ppid` descendants within the member set. The reaper passes no
  root.
- **Spared:** members whose `/proc/<pid>/comm` **or** space-joined `cmdline` matches any
  configured spare pattern (`re.search`), plus their descendants, so tmux panes survive
  with their server. Default patterns:
  - `^ssh-agent$` (comm)
  - `^gpg-agent$` (comm)
  - `^tmux: server$` (comm; the server keeps its original client cmdline)
  - `^ssh: .*\[mux\]$` (cmdline; the ControlMaster proctitle)

  Verify each against live processes with `ps -o comm=,args=` while implementing. An
  invalid user pattern is logged and ignored; it never makes the sweep fail closed or
  open wholesale.

- **Targets:** every remaining member. Each target is pinned with
  `process_identity_token` and signalled only while that token still matches, so a
  recycled pid is never hit.
- **Termination:** SIGTERM all targets, poll until `term_grace_seconds`, then SIGKILL
  survivors. Re-read `cgroup.procs` and repeat (SIGKILL directly) for members that
  appeared mid-sweep, up to 3 rounds. The total run is hard-bounded to a few seconds
  past the grace.

### Safety rails

- In-runner sweeps run only when **all** of these hold:
  - the platform is Linux;
  - cgroup v2 is unified (a `0::` line in `/proc/self/cgroup`);
  - the process's own unit matches `sase-agent-*.scope`;
  - `_detach_scope_disabled()` is false (otherwise children may not have escaped);
  - `agent_scope_teardown.enabled` is true;
  - the **current process is the runner**: its own cmdline has `run_agent_runner.py` as
    an argv basename.

  The last check matters because an agent running `pytest` or any `sase` CLI from inside
  its own scope must never sweep the scope and kill its own provider or runner. Tests
  also get `SASE_DETACH_SCOPE_DISABLE=1` from the autouse fixture in
  `tests/conftest.py`.

- The sweep entry points never raise. They log and return a result object instead.
- The reaper skips any scope containing a runner (an argv basename of
  `run_agent_runner.py`) or a `systemd-run` process (the pre-exec window). It also skips
  scopes younger than `reaper_min_scope_age_seconds`, measured from the trailing
  `<time_ns>` in the unit name. Unparsable names are skipped.
- Under pytest (`PYTEST_CURRENT_TEST` set), an unfiltered reaper scan of the real cgroup
  tree is refused. Tests must pass an explicit fake root or an explicit unit filter,
  mirroring `service_lifecycle_blocked_in_tests`.
- The reaper signals pids rather than running `systemctl --user stop`. Stopping the unit
  would also kill spared daemons. Once its last process exits, `--collect`
  garbage-collects the scope.

### Alternatives rejected

- **`systemctl --user stop` the scope at runner exit** (the research report's first
  suggestion): this kills ssh-agent, ControlMaster and tmux servers that other sessions
  share, as observed on apollo.
- **Matching the launch scratch key in each process's environ** (what
  `agent/process_tree.py` does for user kills): escaped monitors, procs and other
  escaped helpers inherit the key and must survive a normal exit. Cgroup membership is
  precisely "did not escape".
- **Fixing muse's shell-tool timeout or adding agent guidance only:** neither is under
  SASE's control nor enforceable. The scope sweep makes any provider's leak harmless.
- **Rust core (`sase-core`):** this is host process lifecycle beside `detach_scope.py`,
  `_subprocess_reap.py` and `process_tree.py`, all of which are Python. No frontend
  needs it to match, so no `sase_core` or binding change is needed.

## Phase escape-helpers — Escape long-lived SASE helpers from the agent scope

Wrap each of these spawns with
`detach_scope(argv, description=..., unit_prefix=..., start_new_session=...)`. Then pass
the returned `argv` and `start_new_session` to the existing `Popen` or runner, following
the pattern in `src/sase/ace/hooks/execution.py` and
`src/sase/bead/attachments/background.py`. Keep the Windows branches unchanged.

1. `_delete_paths_in_background` in `src/sase/_linked_repo_workspaces.py`, using unit
   prefix `sase-trash-delete`. It is reached from runner workspace prep. If killed
   partway, `*.sase-reclone-trash-*` and `*.sase-corrupt-trash-*` clones would leak,
   because no sweeper covers them.
2. The goals fetch worker spawn in `src/sase/goals/fetch_worker.py` (the
   `subprocess.Popen` of `python -m sase.goals.fetch_worker`) and `_spawn_fetch_worker`
   in `src/sase/main/goal_fast_path.py`, using unit prefix `sase-goal-fetch`. Prefer
   making the fast path call one shared spawn helper in `fetch_worker.py` instead of
   keeping two copies.
3. The federation worker daemon in `ensure_started` in
   `src/sase/dispatch/federation/_supervisor.py` (`self.popen(...)`), using unit prefix
   `sase-federation`. It is shared over a socket by the TUI and other clients. Because
   `systemd-run --scope` execs in place, `self._proc.pid` still names the worker.
4. The `tmux new-session -d` bootstrap in `_create_agents_tmux_session_with_bootstrap`
   in `src/sase/main/ace_tmux_session.py`, using unit prefix `sase-tmux`. When it starts
   the server, the server then lives in its own scope instead of the agent's.
   `systemd-run --scope --quiet` keeps the `-P -F` stdout.

Tests: extend `tests/test_detach_scope_background_workers.py`, using the
`fake_popen_class`, `escaped_launch` and `noop_launch` helpers in
`tests/_detach_scope_helpers.py`. Add one test per site that asserts the detach kwargs
(description and unit prefix), that Popen receives the scoped argv, and the no-escape
(noop) behaviour.

## Phase runner-teardown — Agent runner sweeps its own scope

### New module `src/sase/agent/scope_sweep.py`

All filesystem roots are injectable (`proc_root`, `cgroup_root`, plus a `kill` callable
and a clock) so unit tests never signal real processes.

- `AGENT_SCOPE_UNIT_PREFIX = "sase-agent"`. Use it in `src/sase/agent/launch_spawn.py`
  in place of the literal. Also add `RUNNER_SCRIPT_NAME = "run_agent_runner.py"`.
- `ScopeMember(pid, ppid, comm, argv, identity)` and
  `read_scope_members(scope_dir, *, proc_root)`.
- `is_agent_runner(member)`: true when any of the first three argv entries has basename
  `RUNNER_SCRIPT_NAME`.
- `plan_scope_sweep(members, *, protect_root, spare_patterns) -> ScopeSweepPlan(targets, spared, protected)`:
  the pure selection rule above.
- `execute_scope_sweep(plan, scope_dir, *, grace_seconds, ...) -> ScopeSweepResult`:
  TERM, grace, KILL, up to 3 re-read rounds, with identity pinning. Reuse or factor the
  `_sweep` idea from `llm_provider/_subprocess_reap.py` rather than inventing a third
  variant.
- `own_agent_scope(...) -> Path | None`: applies every in-runner safety rail and returns
  the scope's cgroup directory under `/sys/fs/cgroup`. Promote the private unit-parsing
  helpers in `detach_scope.py` to public names, or import them deliberately, instead of
  copying them.
- `sweep_own_agent_scope(*, context: Literal["exit", "turn"]) -> ScopeSweepResult | None`:
  config-gated and never raises. When it terminated anything, it prints one line to the
  runner's stdout (the agent output log) and logs at INFO. The line looks like
  `[scope-sweep exit] terminated N leaked process(es) in <unit>:` and lists up to 10
  entries of `pid comm argv[:120]`, so the leaking agent's own log shows what it left
  behind.

### Config

In `src/sase/default_config.yml` (near `runner_slots`), add a top-level block with
comments. Add matching entries in `src/sase/config/sase.schema.json`, typed getters with
defaults in `src/sase/config/_settings_system.py` (exported the same way as the
`managed_tmp` getters), and the mirror in `docs/configuration.md`.

```yaml
agent_scope_teardown:
  enabled: true
  term_grace_seconds: 3
  reaper_min_scope_age_seconds: 120
  spare_process_patterns:
    - "^ssh-agent$"
    - "^gpg-agent$"
    - "^tmux: server$"
    - '^ssh: .*\[mux\]$'
```

This is a config field, not a feature flag. Users may permanently choose to disable it
or extend the spare list, and the fix must be on by default.

### Runner hooks

- In `main()` in `src/sase/axe/run_agent_runner.py`, make
  `sweep_own_agent_scope(context="exit")` the first statement of the outer `finally`,
  inside its own `try/except Exception` that logs. Keep the existing
  `finalize_runner_shutdown` and `cleanup_launch_scratch` nesting unchanged after it.
  Running first means workspace release in `finalize_runner_shutdown` and scratch
  cleanup (which keeps scratch while a live process carries the launch key) both see a
  scope without leftovers.
- > [!decision] turn_sweep

  In `_run_execution_loop_bound` in `src/sase/axe/run_agent_exec.py`, call
  `sweep_own_agent_scope(context="turn")` at the top of every `while True` iteration
  **after the first**. No provider is running at that point; the runner and its live
  descendants stay protected.

### Docs

- Update the detached-work paragraph in `docs/architecture.md` (around the "On Linux,
  when detached work starts inside a SASE-owned systemd unit or scope" text) with the
  contract: the agent scope bounds the agent's processes, long-lived work must use
  `detach_scope`, and shared daemons are spared.
- Add the same contract as a short paragraph to the `src/sase/detach_scope.py` module
  docstring, so the next person adding a background spawn sees it.

### Memory

> [!decision] scope_decision_record

If approved, use `/sase_memory_write` (plan approval is the authorization) to add a
decisions-web strand (for example
`sase/memory/decisions/agent-scope-bounds-processes.md`). It states:

- **Claim:** an agent's `sase-agent` scope bounds every process the agent starts;
  leftovers are swept at runner exit and by the reaper, and long-lived work must escape
  via `detach_scope`.
- **Why:** the alternatives above.
- **Cost:** the spare list needs maintenance, the mechanism is Linux/cgroup-v2 only, and
  a turn may not leave background processes for its successor.
- **Reopens when:** a legitimate need for agent-started processes to outlive the runner
  appears that cannot use `detach_scope`.

Link `[[decisions/single-turn-agents]]`, then run `sase memory init`.

### Tests

- `tests/test_agent_scope_sweep.py`, using a fake `/proc` and cgroup tree built under
  `tmp_path` in the style of `tests/test_detach_scope_command.py`. Cover:
  - protect-root descendants;
  - spare by comm and by cmdline, plus spared descendants;
  - zombie and unreadable pids;
  - identity pinning, where a recycled pid is not signalled;
  - a member that appears in a later round;
  - grace escalation to SIGKILL;
  - every `own_agent_scope` gate (non-Linux, cgroup v1, a non-`sase-agent` unit, a
    non-runner argv, a disabled env, a disabled config);
  - invalid spare regexes;
  - "never raises".
- A real-signal test that is safe without systemd: the test spawns its own orphaned
  `sleep` loop, writes its pids into a temp `cgroup.procs`, and runs
  `execute_scope_sweep` against the real `/proc`. It may only ever target pids the test
  created.
- Runner wiring tests: monkeypatch `sweep_own_agent_scope` and assert that it runs
  before `finalize_runner_shutdown` on success, exception, `SystemExit` and user-kill
  paths, that its exceptions are swallowed, and that cleanup still runs. Assert that the
  loop calls it only from the second iteration on. Extend the existing runner `main()`
  and exec-loop tests.

## Phase scope-reaper — Orphaned agent scope reaper job

### Core API (in `src/sase/agent/scope_sweep.py`)

- `discover_agent_scopes(*, cgroup_root=None, only_units=None, proc_root) -> list[AgentScope(unit, path, created_ns)]`:
  - Locate the user-manager subtree from the `user@<uid>.service` component of
    `/proc/self/cgroup`, falling back to
    `/sys/fs/cgroup/user.slice/user-<uid>.slice/user@<uid>.service`.
  - Search at most 3 levels deep (normally `app.slice/`) for `sase-agent-*.scope`
    directories.
  - Return nothing, with a reason, on non-Linux hosts or without cgroup v2.
  - Enforce the pytest guard.
- `reap_orphaned_agent_scopes(*, apply: bool, min_age_seconds, ...) -> ReapResult`,
  following the `apply=` precedent of `sweep_orphan_proc_runtime_dirs`. Classify each
  scope as one of:
  - `skipped_young`
  - `live`: a runner or `systemd-run` member
  - `empty`
  - `spared_only`
  - `reapable`

  For reapable scopes, it runs `plan_scope_sweep(protect_root=None)` and, with `apply`,
  also runs `execute_scope_sweep`. Per reaped scope it records the unit, a best-effort
  `SASE_AGENT_NAME` read from the first target's `/proc/<pid>/environ`, the target
  count, and up to 5 `comm argv[:120]` entries.

### Job

- `src/sase/scripts/sase_chop_orphan_agent_scope_reap.py`:
  `@builtin_chop("orphan_agent_scope_reap")`, modelled on
  `sase_chop_proc_runtime_sweep.py`.
  - Write one `runtime.log` line per reaped scope.
  - Summary counters: `scanned`, `live`, `skipped_young`, `spared_only`,
    `reaped_scopes`, `terminated`, `errors`.
  - Reason `"disabled"` when `agent_scope_teardown.enabled` is false;
    `"nothing_eligible"` when nothing was reaped.
- Register it in `src/sase/default_config.yml` under `axe` routine `checks` (300 s
  cadence, `timeout: "2m"`) with a full description. It keeps the default `always`
  trigger, because process death leaves no filesystem event (the same rationale as
  `stale_running_cleanup`).
- Add both console scripts to `pyproject.toml`: `sase_chop_orphan_agent_scope_reap` and
  `sase_job_orphan_agent_scope_reap`, pointing at
  `sase.scripts.sase_chop_orphan_agent_scope_reap:main`.
- Docs:
  - the `docs/axe.md` job tables (the checks-routine table and any job list that
    enumerates `checks` jobs);
  - the `docs/configuration.md` mirror of the `checks` routine;
  - a sentence in `docs/troubleshooting` or `docs/axe.md` on spotting leaks with
    `systemctl --user list-units 'sase-agent-*'` and running the job by hand with
    `sase axe job run orphan_agent_scope_reap`.

### Tests

- Unit tests with a fake cgroup tree covering:
  - discovery depth and fallback root;
  - the pytest guard;
  - age parsing from unit names;
  - each classification;
  - `apply=False` signalling nothing;
  - a scope with a live runner plus leftovers left untouched.
- Chop contract tests mirroring
  `tests/test_axe_chop_output_contract_digest_and_reap.py`. Update
  `tests/test_axe_default_chop_triggers.py` and any lane/trigger pin it needs;
  `tests/test_axe_lumberjack_config.py` enforces the pyproject entries.
- A live test next to `tests/test_detach_scope_live.py`. Skip it unless the host is
  Linux with cgroup v2, a reachable user manager and `systemd-run`, and clear
  `SASE_DETACH_SCOPE_DISABLE` locally.
  - Start
    `systemd-run --user --scope --collect --unit=sase-agent-<pid>-<time_ns> -- sh -c '(while :; do sleep 0.2; done) & exit 0'`.
  - Call `reap_orphaned_agent_scopes(apply=True, min_age_seconds=0, only_units={unit})`.
  - Assert the loop pid is gone and the scope disappears within a few seconds.
  - In a `finally`, kill the loop pid directly if the test failed.
- Host sanity check before finishing, read-only: run
  `reap_orphaned_agent_scopes(apply=False)` on the current host and include the
  classification in the phase notes. It must not run with `apply=True` against the real
  tree. On apollo, expect the stale `waiting-worker` scope to classify as reapable and
  the `ssh-agent` scope as `spared_only`.

## Verification

Each phase runs `just check` (through `sase tool run`, per the guarded-recipe rules) and
fixes what it breaks. `just check-full` is not run unless explicitly requested. The land
agent confirms:

- the three phases' tests pass together;
- the `checks` routine lists the new job;
- `sase axe job run orphan_agent_scope_reap` works on the host and its summary looks
  right. That run applies for real, so it is the deployment moment. It is expected to
  clean up the stale apollo `waiting-worker` scope and must leave the `ssh-agent` scope
  alone.

## Out of scope (record as follow-ups, not in this epic)

- Hosts without systemd user scopes or cgroup v2, including macOS. The sweep is a
  documented no-op there.
- Extending the reaper to `sase-monitor` and `sase-proc` scopes. They need their own
  owner-process signatures.
- muse's shell tool killing only the leader on timeout. That is a provider bug; the
  sweep makes it harmless.
- The other athena findings in the report: the stale Pushgateway series, the two waiting
  runners using about 13 % of a core each, and the TUI's unreaped zombie children.
