---
tier: epic
title: Fix monitor follow-up output excerpts and host PATH loss in agent-shell launches
goal: 'Monitor follow-up prompts show the real tail of the monitored command''s output,
  and agents launched from inside an agent''s harness shell (monitor follow-ups, handoffs,
  nested launches) resolve provider CLIs with the same PATH a host launch would have.

  '
phases:
- id: monitor-output-tail
  title: Show the real output tail in monitor follow-up prompts
  depends_on: []
  size: small
  description: 'monitor-output-tail: populate retained-log segment ranges when no
    retention snapshot exists, read legacy null-range metadata, and excerpt the tail
    rather than the head.'
- id: host-path-launch-env
  title: Restore the host PATH for agent launches made from agent shells
  depends_on: []
  size: medium
  description: 'host-path-launch-env: runners stamp a SASE_HOST_PATH_SNAPSHOT of their
    PATH and spawn_agent_subprocess restores it, so login-shell PATH resets cannot
    hide provider CLIs.'
decisions:
  path_merge_order:
    ask: Should restored host PATH entries take precedence over the spawning shell's
      PATH entries?
    choices:
      host_first: Agent-shell launches get the host's PATH order; shell-only dirs
        are appended
      shell_first: Shell PATH keeps precedence; only missing host dirs are appended
    default: host_first
    why: An agent launched from an agent shell should be indistinguishable from a
      host launch
    answer: host_first
  monitored_command_path:
    ask: Should the monitored command itself also run with the restored host PATH?
    default: false
    why: A monitor stands in for the agent running the command, so it keeps the agent
      shell env
    answer: false
proposed_by: bbugyi200.athena.133
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-23 09:29:08
status: done
bead_id: sase-1nk
---

- **PROMPT:** [prompts/202610/monitor_followup_output_and_host_path.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/monitor_followup_output_and_host_path.md)
- **BEAD:** [sase-1nk](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nk/README.md)

# Fix monitor follow-up output excerpts and host PATH loss in agent-shell launches

## Background: what broke `bob-cli-as.4`

`bob-cli-as.4` (Codex, `gpt-6-luna`) finished its implementation turn and started a
`verify` monitor running `npm test` in its linked `bob-plugins` checkout. The test run
failed (exit 1: 5 failed, 14 cancelled), and the monitor's `failed` branch launched the
follow-up agent `bob-cli-as.4--1`. That follow-up died before its first token:

```
FileNotFoundError: Unable to launch Codex CLI executable 'codex'. Set SASE_CODEX_PATH
to the Codex binary, ensure 'codex' is discoverable on PATH, or set NVM_BIN so SASE
can try NVM_BIN/codex.
```

The investigation found two independent SASE bugs. Fixing either one would not fix the
other.

### Bug 1: every proc-backed monitor follow-up prompt replaces the output with a gap notice

The follow-up prompt's `## Last 200 lines of output` section contained only:

```
[retained output gap: bytes 0:671302 are unavailable]
```

But the retained log at `diagnostics/retained_logs/live_reply.md` was complete (671302
bytes, `complete: true`, `drain_confirmed: true`).

Root cause, in `src/sase/monitor/diagnostics.py`:

- `freeze_retained_log_metadata(..., retention=None)` assigns each segment's `range`
  with `_pop_segment_range(ranges, size)`. When `retention` is `None`, `ranges` is
  empty, so every segment gets `"range": null`. The top-level `retained_ranges` field is
  still filled in from `_fallback_retained_ranges`, so the metadata looks complete.
- `read_retained_log_range` skips every segment whose `range` is not a dict, so it
  reports the whole requested range as a gap.
- The proc-backed monitor path always passes `retention=None`, at
  `src/sase/monitor/proc_adapter.py` (the settle path) and
  `src/sase/monitor/reconcile.py`. Only the legacy in-process `supervise.py` path passes
  a real retention snapshot, and the existing test
  (`tests/monitor/test_monitor_diagnostics.py::test_supervisor_freezes_stage_manifest_and_retained_log_metadata`)
  covers only that path. It also asserts only `metadata["available"] is True`, which is
  true even when the text is just a gap notice.

Impact: in October, all 999 retained-log metadata files on this host have null segment
ranges, and 284 monitor follow-up prompts carry `retained output gap: bytes 0:...` in
place of output. The bug dates back to `e3feeb1ec1` (2026-09-11).

There is a latent second defect in `src/sase/monitor/followup_output.py`.
`frozen_output_text` reads the retained ranges forward from their start, up to
`max_chars` (12 KiB by default, `TOTAL_RAW_EXCERPT_MAX_BYTES`). It ignores the tail-line
count it computes. If only the ranges were fixed, the prompt would show the first 12 KiB
of the log under a "Last N lines" heading. For `npm test` that is the build and early
passing tests, never the failures at the end.

### Bug 2: launches made from inside a Codex agent shell lose nvm-installed provider CLIs

The chain that lost `codex`:

1. The epic phase runners were spawned from the service-host context. Their env comes
   from `~/.sase/service/env`, which captures `PATH` (nvm bin first) but not `NVM_BIN`
   (`CAPTURED_BASE_ENV_NAMES` in `src/sase/service/env.py`).
2. Codex runs every shell command as `/bin/zsh -lc '<cmd>'`. On this host the login
   profile (`~/.profile` → `/etc/profile`) rebuilds PATH from scratch and drops
   `~/.config/nvm/versions/node/v22.14.0/bin`. nvm itself is only loaded from `.zshrc`.
3. `sase monitor start` ran in that login shell. The proc supervisor it spawned
   (`src/sase/procs/spawn.py::_supervisor_env`, `os.environ.copy()`) inherited the reset
   PATH.
4. When the monitor settled, the supervisor launched the follow-up through
   `src/sase/monitor/followup.py` → `spawn_agent_session_successor` →
   `src/sase/agent/launch_spawn.py::spawn_agent_subprocess`. That builds the child env
   from `dict(os.environ)`, so the follow-up runner got the reset PATH and no `NVM_BIN`.
5. `resolve_codex_executable` (`src/sase/llm_provider/_codex_instructions.py`) tries
   `SASE_CODEX_PATH`, then PATH, then `NVM_BIN/codex`. All three missed, so it returned
   the bare string `codex` and `Popen` failed.

Evidence: the monitored `npm test` ran under `/usr/bin/node` v20.19.2, fingerprinted
from the Node test-runner stack line numbers, instead of nvm's v22.14.0. So PATH had
already lost the nvm bin directory when the supervisor started.

This is not Codex-specific on the launch side. `gemini`, `qwen`, `opencode` and `codex`
are installed only in the nvm bin directory, so any follow-up, handoff, or `sase run`
launched from a Codex shell that routes to one of them fails the same way. The other
codex phases of the same epic (`bob-cli-as.6`, `bob-cli-as.7`) were spawned the same way
and will hit this if they use monitors.

Design principle: a SASE agent launch is host-owned. An agent spawned by a process that
happens to live inside an agent's harness shell (monitor supervisor, handoff, nested
`sase run`) should get the same PATH a host launch would have had, no matter what the
harness's login shell did to PATH. Login shells reset PATH, but they leave unknown
environment variables alone. So the runner records its own PATH in a dedicated variable,
and `spawn_agent_subprocess` restores it.

## Phases

### Phase `monitor-output-tail`: Show the real output tail in monitor follow-up prompts

Make the follow-up prompt's raw-output section show the actual last lines of the
monitored command's output for proc-backed monitors, and keep reading older artifacts
that were frozen with null segment ranges.

Changes:

1. `src/sase/monitor/diagnostics.py::freeze_retained_log_metadata`: when `retention` is
   `None`, give each copied segment a contiguous range in segment order (`.1` rotated
   first, then active), the same coordinates `_fallback_retained_ranges` already
   produces for the top-level `retained_ranges`. Segment ranges and top-level ranges
   must agree. Leave the `retention is not None` path unchanged.
2. `src/sase/monitor/diagnostics.py::read_retained_log_range`: tolerate the legacy
   shape. If every segment's `range` is null, derive contiguous ranges from the
   segments' `bytes` in order before reading, so monitors frozen before this fix
   (including `m7p0bx9p29vb` from `bob-cli-as.4--mon`) become readable, for example on a
   reconcile/re-delivery or `sase monitor show`. Do not rewrite metadata on disk.
3. `src/sase/monitor/followup_output.py::frozen_output_text`: select the **tail**, not
   the head. Walk the retained ranges from the end and read at most `max_chars` bytes
   ending at the last retained byte, splitting across ranges when the last one is
   shorter than the budget. Keep the ranges in ascending order in the result, and keep
   gap notices between non-contiguous ranges. A small helper in `diagnostics.py`, such
   as `read_retained_log_tail(artifacts_dir, max_bytes=...)`, is fine if it keeps
   `frozen_output_text` simple. Starting mid-line or mid-UTF-8 character is acceptable
   (`untrusted_output_section` already trims to `tail_lines`), but drop a leading
   partial line when that is easy.

Tests (extend `tests/monitor/test_monitor_diagnostics.py` and the existing follow-up
prompt tests under `tests/monitor/`):

- `freeze_retained_log_metadata(..., retention=None)` writes non-null segment ranges
  that match `retained_ranges`, for active-only and rotated+active logs.
- `read_retained_log_range` returns real bytes, not a gap notice, for fresh
  `retention=None` metadata and for hand-written legacy metadata with `"range": null`
  segments.
- `frozen_output_text` on a log larger than the 12 KiB budget contains a sentinel line
  written at the end of the log and does not contain one written at the start.
- Tighten the existing supervisor test so it asserts the ranged text contains the
  command's output, not just `available is True`.
- One end-to-end test through the proc-backed settle path (`proc_adapter`) asserting
  that the composed follow-up prompt's `Last N lines of output` section contains the
  command's final output lines and no `retained output gap` notice.

Optional manual check: run `frozen_output_text` against the real artifacts of
`bob-cli-as.4--mon` under
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202610/23/20261023090457` with
its stored monitor result. The excerpt should end with the `npm test` summary
(`# tests 3472`, `# fail 5`, `# cancelled 14`).

### Phase `host-path-launch-env`: Restore the host PATH for agent launches made from agent shells

Make every agent launched through `spawn_agent_subprocess` get its launching runner's
PATH back, even when the spawn happens in a process descended from a harness login shell
that reset PATH.

Changes:

1. Add an env contract constant in `src/sase/env_contracts.py`:
   `SASE_HOST_PATH_SNAPSHOT_ENV = "SASE_HOST_PATH_SNAPSHOT"`. The name deliberately
   avoids the `SASE_<PROVIDER>_PATH` override pattern (`provider_path_env_var`) and
   every prefix the env-hygiene scrubbers drop (`SASE_AGENT_`, `SASE_PROC_`,
   `SASE_TOOL_`, `SASE_MONITOR_`, `SASE_CHOP_`, `SASE_JOB_`, and the
   workspace-preallocation suffixes). Proc supervisors, monitored commands and follow-up
   spawns must all inherit it untouched.
2. Runner boot (`src/sase/axe/run_agent_runner.py`, at the earliest point where the
   process env is final and before any provider subprocess starts): set
   `os.environ[SASE_HOST_PATH_SNAPSHOT]` to the runner's own `PATH`. The runner's PATH
   is the host PATH: either the host launched it directly, or step 3 restored it. Every
   provider CLI, and every shell command the provider runs, then inherits the snapshot.
   Confirm there is no other runner entry point that skips this. If there is, stamp
   there too.
3. `src/sase/agent/launch_spawn.py`: add `_restore_host_path_env(env)` and call it in
   the `env_shape` stage after the `_remove_inherited_*` calls and before `extra_env` is
   re-applied. That way an explicit `PATH` in `extra_env` still wins. If the snapshot is
   present and non-empty, rebuild `PATH` according to the `path_merge_order` decision,
   deduplicating entries in order and skipping empty ones. If the snapshot is absent, as
   for TUI and service launches, leave PATH unchanged.

   > [!decision] path_merge_order = host_first `PATH = snapshot entries` followed by any
   > entries from the current PATH that are not already present.

   > [!decision] path_merge_order = shell_first `PATH = current entries` followed by any
   > snapshot entries that are missing.

4. Monitored command environment, per the `monitored_command_path` decision (nvm's Node
   22 vs. system Node 20 under Codex is the visible difference):

   > [!decision] monitored_command_path = no Do not touch `src/sase/procs/spawn.py` or
   > the monitor proc env. The monitored command keeps the agent shell's PATH, and the
   > snapshot just rides along in its env.

   > [!decision] monitored_command_path Also apply the same restore to the proc
   > supervisor env built in `src/sase/procs/spawn.py::_supervisor_env`, so the
   > monitored command runs with the host PATH. Add a test that a monitored command sees
   > a directory that exists only in the snapshot.

5. Document `SASE_HOST_PATH_SNAPSHOT` as an internal, SASE-managed variable in
   `docs/reference/environment-variables.md`. Read the `docs_writing` reference memory
   first. In `docs/reference/providers/codex.md`, add one sentence saying launches made
   from inside an agent's shell restore the runner's PATH, so nvm-only Codex installs
   keep resolving.

Tests:

- Unit (`tests/agent/` near the existing `launch_spawn` env-shaping tests): with
  `PATH=/usr/bin:/bin` and a snapshot of `<tmp>/nvmbin:/usr/bin:/bin`, the shaped
  subprocess env has `<tmp>/nvmbin` per the chosen order, no duplicates, and an
  `extra_env` `PATH` still overrides. With no snapshot, PATH is unchanged.
- Regression for the actual failure: put a fake executable `codex` only in a temp dir
  that is in the snapshot but not in the munged PATH, with `NVM_BIN` and
  `SASE_CODEX_PATH` unset. After env shaping, `resolve_codex_executable()` evaluated
  under the shaped env returns the temp-dir path.
- Runner boot stamps `SASE_HOST_PATH_SNAPSHOT` equal to its PATH. Unit-test the helper
  that does it, not the full runner.
- The snapshot survives `scrub_agent_identity_env`, `scrub_chop_context_env`,
  `scrub_proc_operation_env` and `scrub_executor_ownership_env`, so a future prefix
  change cannot silently drop it.

## Out of scope (recorded for the reviewer)

- **Login profile resets PATH (dotfiles, `chezmoi`).** `~/.profile` sources
  `/etc/profile`, which overwrites PATH, and nvm is only initialized in `.zshrc`. So
  every Codex shell command (`zsh -lc`) runs system Node v20.19.2 instead of nvm's
  v22.14.0, and cannot see `codex`/`gemini`/`qwen`/`opencode`. Phase
  `host-path-launch-env` makes SASE launches immune to this. The Node-version mismatch
  inside Codex agent shells is a separate dotfiles fix, for example saving the inherited
  PATH before sourcing `/etc/profile` and merging it back. It was checked not to cause
  this run's test failures: the 4 `test-ledger-tools-unblocked-glyph.cjs` failures and 1
  `test-navigation-decision-card-handlers.cjs` failure reproduce on clean `bob-plugins`
  master under both Node 20 and Node 22.
- **Capturing `NVM_BIN` in the service env.** It would only help Codex, which is the
  only resolver with an `NVM_BIN` fallback, and only for service-launched runners. The
  snapshot restore covers every provider and every launch context.
- **Recovering `bob-cli-as.4`.** Its implementation is still uncommitted in its
  workspace, with a real regression from this phase: 14 cancelled tests in
  `test-navigation-decision-card-handlers.cjs` ("Promise resolution is still pending").
  Re-running its follow-up is an operational step, not part of this plan.
