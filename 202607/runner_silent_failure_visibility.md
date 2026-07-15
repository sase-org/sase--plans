---
tier: epic
title: Eliminate silent runner deaths; always surface failure output in the TUI
goal: 'Agents that die for any reason always surface actionable failure output in
  the ace TUI, and the post-wait runner code refresh no longer kills agents by replaying
  argv that references an already-deleted temp prompt file.

  '
phases:
- id: reexec-prompt-fix
  title: Survive the post-wait runner re-exec (root cause)
  depends_on: []
- id: runner-error-coverage
  title: Record failure artifacts on every runner exit path
  depends_on:
  - reexec-prompt-fix
- id: tui-failure-fallback
  title: TUI fallback failure output for artifact-less agent deaths
  depends_on:
  - runner-error-coverage
create_time: 2026-07-15 18:36:16
status: wip
---

# Plan: Eliminate silent runner deaths; always surface failure output in the TUI

## Incident and root cause (diagnosed, evidence on disk)

Agent `sase-65.2` (pid 2395029, artifacts dir
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/15/20260715180341/`) showed FAILED in the ace TUI with **no
failure output**. Its run log (`~/.sase/workflows/202607/gh_sase-org__sase_ace-run-260715_180341.txt`) ends with:

```
All dependencies satisfied, proceeding with workflow
Refreshing sase runner code after dependency wait: 59b11cd1... -> ede79bc9...
Error reading prompt file: [Errno 2] No such file or directory: '/home/bryan/tmp/sase/sase_ace_prompt_MM9IjS.md'
```

The failure chain:

1. The runner (`src/sase/axe/run_agent_runner.py`) reads its prompt from a temp file at `main()` startup via
   `read_prompt_file` (`src/sase/axe/run_agent_runner_cli.py:55`), which **unlinks the file in a `finally`** — the file
   is a one-shot TUI→runner handoff.
2. The agent had `%w:sase-65.1`, so it blocked in `wait_for_dependencies` for ~15 minutes. During that wait, other
   agents landed commits on the editable sase checkout's master.
3. `refresh_runner_code_after_wait` (`src/sase/axe/run_agent_runner_refresh.py:58`, added 2026-07-13 in `f7cbca6fd`)
   detected the HEAD move and re-exec'd via `os.execv(sys.executable, [sys.executable, *sys.argv])` — replaying the
   **original argv, including the deleted prompt-file path**.
4. The re-exec'd process hit `read_prompt_file` → `FileNotFoundError` → `sys.exit(1)` at `run_agent_runner_cli.py:62`.
   That exit happens **before** the error-recording `try` block (`run_agent_runner.py:205`), so no failed `done.json`
   (`write_error_done_marker`), no `error_report.md`, and no completion notification were written. `workflow_state.json`
   stayed `"running"` with zero steps.
5. The TUI loaders (`src/sase/ace/tui/models/_loaders/_workflow_loaders.py:155-167` and
   `_workflow_snapshot_loaders.py:95-102`) synthesize FAILED from "active status + dead pid + no in-progress step", but
   source `error_message`/`error_traceback` only from `workflow_state.json`'s `error`/`traceback` or a failed step — all
   absent here. The detail-pane ERROR block, and even the `Output:` log-path breadcrumb, render only
   `if agent.error_message` (`src/sase/ace/tui/widgets/prompt_panel/_workflow_render.py:124-130`, similar gate in
   `_agent_display_header.py`), so the pane shows nothing.

Because the refresh feature is only two days old and fires whenever a dependency wait spans any commit to the editable
checkout, **every waiting agent in an active development session is exposed** — which is exactly why these no-output
failures have suddenly become common.

## Design

Three layers, in dependency order: remove the root cause, then make the runner record failure artifacts wherever it can,
then give the TUI a universal fallback for deaths no in-process handler can catch (SIGKILL/OOM, import-time crashes on
freshly pulled code, malformed argv).

### Phase `reexec-prompt-fix` — Survive the post-wait runner re-exec

Make the re-exec path self-consistent with the one-shot prompt-file contract:

- **Re-materialize the prompt file before `os.execv`.** Pass the already-read prompt content and `args.prompt_file` path
  into `refresh_runner_code_after_wait`. Immediately before exec, rewrite the prompt file (same path, same content) so
  the replayed argv is valid again. The refreshed pass re-reads and re-unlinks it, preserving existing semantics.
- **Refresh must never kill a run.** If re-writing the prompt file fails, skip the refresh (log a warning, continue on
  the old code). The refresh is an optimization, not a correctness requirement.
- **Defensive fallback read.** The runner already persists the prompt as `submitted_xprompt.md` in the artifacts dir
  (`write_submitted_xprompt_artifact`). Teach `read_prompt_file` (or its caller) that when the temp file is missing and
  `SASE_RUNNER_CODE_REFRESHED_ENV` is set, it may fall back to the artifacts copy (derivable from argv via
  `setup_artifacts_directory` inputs) instead of dying. This protects against any future one-shot-argv drift.
- Audit `sys.argv` for other one-shot resources replayed by the exec (none known today; document the constraint in
  `run_agent_runner_refresh.py` so future argv additions consider it).

Testing: unit tests for `refresh_runner_code_after_wait` with `os.execv` monkeypatched — asserts the prompt file exists
with identical content at exec time, argv unchanged, one-shot env guard set; asserts refresh is skipped (no exec) when
the rewrite fails; asserts the guard prevents double refresh. Unit test for the `read_prompt_file` fallback path.

### Phase `runner-error-coverage` — Record failure artifacts on every runner exit path

Narrow the window where the runner can exit without writing failure data:

- **Hoist error recording to cover `main()`'s bootstrap.** Today lines 102–203 of `run_agent_runner.py` (arg parsing,
  prompt read, telemetry init, `setup_artifacts_directory`, `preprocess_prompt_xprompts`) run before the recording
  `try`. Restructure so that everything after argument parsing runs under a handler that records errors via
  `record_runner_error`/`write_error_done_marker`. Reorder so the artifacts dir is established as early as possible (it
  is a pure function of argv), and make the error context tolerate not-yet-populated fields.
- **Stop calling `sys.exit` without recording.** `read_prompt_file` should raise (letting the recording handler classify
  and persist it) rather than print-and-exit. Sweep the runner bootstrap for the same pattern (e.g.
  `claim_deferred_workspace`'s `sys.exit(1)` is already inside the `try`; verify nothing else escapes).
- **Persist the run-log path for consumers.** Write the runner's `output_path` (argv) into `agent_meta.json` alongside
  the existing fields, early — before any dependency wait — so any frontend can locate the raw stdout/stderr log without
  re-deriving the sharded path. (The TUI phase consumes this; legacy runs without the field still work via derivation.)
- Genuine limits remain: a malformed-argv exit before the timestamp is known, and SIGKILL/OOM/import-time crashes,
  cannot write artifacts. Those are exactly what the TUI phase covers.

Testing: tests that drive `main()` (or an extracted bootstrap helper) with a missing prompt file / failing bootstrap
step in a temp sandbox and assert a failed `done.json` with `error` + `traceback` is written and
`finalize_runner_shutdown` still runs (workspace release, notification). Assert user-kill semantics
(`is_user_kill_exit`) are unchanged.

### Phase `tui-failure-fallback` — TUI fallback failure output for artifact-less deaths

Guarantee the Agents-tab detail pane always shows failure output for FAILED agents, no matter how the process died:

- **Loader-level fallback.** In both loaders that synthesize FAILED (`_workflow_loaders.py` and
  `_workflow_snapshot_loaders.py`), when no `error_message`/`error_traceback` can be sourced from `workflow_state.json`
  or a failed step, attach a fallback: the tail (last ~40 lines, size-capped) of the run output log, prefixed with an
  explicit explanation such as "Runner exited without recording an error (crash, kill, or startup failure). Last
  output:". Locate the log via the new `agent_meta.json` `output_path` field, falling back to derivation from
  `cl_name` + launch timestamp (`safe_launch_name` + `sharded_path("workflows", ...)`, the policy in
  `src/sase/agent/launch_spawn.py:192-193`). If the log is missing or empty, still attach an explicit message naming the
  paths checked — the pane must never be silently blank for a FAILED agent.
- **Always show the `Output:` breadcrumb for FAILED agents.** In `_workflow_render.py` and `_agent_display_header.py`,
  un-nest the `Output:` path line from the `if agent.error_message` gate so a FAILED agent always points at its raw log,
  and render the ERROR section whenever `display_status == FAILED` (using the fallback text above).
- **Performance constraints (per `memory/tui_perf.md`).** Loaders already run off the event loop in the background
  refresh path — keep the tail-read there, never in a render/message handler. Cache tail reads keyed by
  `(path, mtime, size)` and only for agents synthesized as FAILED (rare, bounded set); dead runs' logs never change, so
  each is read once. No new refresh code paths; no full-rebuild triggers. Verify with `SASE_TUI_PERF=1` / existing bench
  tests that j/k p95 is unaffected.
- **Boundary note.** The tail-read fallback is host-side display enrichment over a host-local log file — analogous to
  the sharding policy that already lives in Python — so it stays in this repo's loaders/adapters and must be applied to
  both the filesystem and wire/index loader paths so the panes agree. If implementation reveals the wire snapshot is
  constructed in Rust core such that Python cannot enrich it uniformly, stop and flag rather than forking behavior
  between the two paths.
- Do not change KILLED/DONE classification or send new notifications (out of scope; failure notifications for silent
  deaths can be a follow-up).

Testing: loader unit tests for the synthesized-FAILED fallback (log present → tail attached; log absent → explicit
message; error already present → untouched; mtime cache hit path). Render tests asserting the ERROR section and
`Output:` line appear for a FAILED agent without recorded error data. Add/refresh a PNG visual snapshot for the FAILED
detail pane if the existing suite covers that pane.

## Risks

- **Re-exec semantics drift**: the refresh pass intentionally re-resolves xprompts; re-materializing the identical raw
  prompt preserves that (resolution happens after read). Tests pin argv and content equality.
- **Restructuring `main()`** risks disturbing kill/finalize ordering; keep the existing `except SystemExit` user-kill
  branch and `finally` shutdown exactly as-is, and lean on existing runner tests plus the new bootstrap-failure tests.
- **Loader IO regressions**: bounded by the FAILED-only, mtime-keyed cache; benches guard the j/k budget.

Each phase must pass `just check` (and the visual suite where touched) before completion.
