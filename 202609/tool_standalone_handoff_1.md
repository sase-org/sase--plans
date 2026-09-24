---
tier: tale
size: medium
title: 'sase-17p.2 standalone-handoff: sase tool run -H over a plain durable proc'
goal: Outside agents, sase tool run -H reserves a ToolRun fail-closed, hands it to
  a plain durable proc whose hidden worker claims and runs the frozen invocation through
  the same executor body as foreground runs, behind the tool_handoff beta flag.
proposed_by: bbugyi200.athena.sase-17p.2
bead: sase-17p.2
status: done
---

- **BEAD:**
  [sase-17p.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17p/sase-17p.2.md)

# Plan: phase `standalone-handoff` (sase-17p.2) of epic sase-17p

Implements phase 2 of `plan:202609/tool_e2_durable_handoff.md` ("Launch protocol",
decisions 2–6 and 12, and section "2. standalone-handoff"). The epic plan is the
authority; this tale only pins the concrete module layout and the facts found while
surveying the current tree.

## Facts established during survey

- Phase 1 (sase-17p.1, closed) landed the sase-core contract as sase-core commit
  `9956773` ("add reservation, claim, stop requests, and owner-aware settlement"), which
  is now on sase-core `origin/master`. sase's `sase-core-revision.txt` still pins
  `eef7ca4…` (one commit earlier) because phase 1 could not ratchet before the push
  (bead note #1). This phase must move the pin first, or CI's "Check pinned core
  bindings" step fails once `tool_run_claim` gains a caller.
- `src/sase/core/tool_run.py` already exposes `tool_run_claim` and
  `tool_run_request_stop`. The Justfile symvision stage whitelists both as
  `--epic-symbol 'sase-17p(...)'`. This phase removes only the `tool_run_claim` entry
  (and fixes the comment above it); `tool_run_request_stop` stays for phase
  `lifecycle-controls`. `sase bead epic-symbols sase-17p.2` reports no entries keyed to
  the phase itself.
- Core begin validation for `launch_mode: handoff` requires: `commit_running: false`, a
  launch envelope, non-empty `owner_kind`/`owner_id`, no `parent_run_id`, envelope
  `argv` == `private_argv or display_argv`, matching `display_argv`, `private_argv`,
  `tool_name`, `extra_args`, `adhoc == (tool_name is None)`, and a named definition that
  re-digests to the definition digest. It stores the launcher pid/boot/identity as
  `launcher`. Claim overwrites `wrapper_pid`/`boot_id`/`process_start_identity` with the
  worker's and sets `owner_log_path`; outcomes are `claimed` (with `launch`), `refused`
  (`not_created`/`owner_mismatch`/`already_claimed`/`not_handoff`), or `stopped`
  (settled `signaled`/`stop_requested`, "command was not run").
- Core finish validates `terminal_cause` against state: `exited` ↔ succeeded/failed,
  `interrupt` ↔ interrupted, `signal`/`timeout` ↔ signaled, `stop_requested` ↔
  signaled/interrupted, `launch_failed` ↔ failed (also the only allowed
  `created → failed`).
- Core reconcile already never settles a `created` hand-off on a dead launcher without
  an owner fact; no owner facts are sent until phase `settlement`.
- The proc supervisor copies `os.environ`, applies the request `env` overlay, then sets
  `SASE_PROC_ID`/`SASE_PROC_LOG_PATH`; a non-monitor `followup` block settles as
  `followup_outcome: pending` and nothing else consumes it.
- `sase proc run`'s attribution (`_infer_attribution`, `_resolve_session_id`) is private
  to `src/sase/main/proc_handler.py`; symvision forbids importing it across files.
- The linked sase-core checkout may lack a compiled `sase_core_rs` extension in a fresh
  workspace; run `just install` (long; generous timeout) before running tests.

## Changes

### 0. Core pin and flag

- Run `just ratchet-core-revision` (it moves the pin to sase-core's remote HEAD) and
  confirm the new SHA contains `9956773` (`git merge-base --is-ancestor` in the linked
  checkout opened with `sase repo open sase-core`). Then `just install` so the local
  venv has the matching binding.
- `sase flag new tool_handoff -k beta -z small` with the epic's three sentences:
  - `--when-enabled`: `sase tool run -H` hands a run off to a durable proc, and monitor
    starts reserve their ToolRun.
  - `--when-disabled`: `-H` is refused with a one-line message naming the flag, and
    monitor starts keep E1.5 wrapping.
  - `--remove-when`: the E2 epic (sase-17p) lands with the ToolRun smoke hand-off fault
    matrix green. Paste the printed registry entry into
    `src/sase/feature_flags/registry.py` (this is the sanctioned flag-bead path; it is
    not a follow-up task bead). Run `tools/check_feature_flags` via `just check`.

### 1. Executor split (`src/sase/tool/executor.py`, `executor_recording.py`)

- `executor_recording.py`: factor the begin request into a public
  `build_begin_request(run_id, *, resolved, owner_kind, owner_id, parent_run_id, events_path, stdout_path, stderr_path) -> dict`
  used by `begin_tool_run` and by the hand-off reservation. `finish_tool_run` gains
  `terminal_cause: str | None = None`, sent when set.
- `executor.py`: keep `execute_tool_run` (validation, resolve, ownership, reconcile with
  reaping, signal handlers) and the foreground prologue (paths, fail-open begin,
  metrics). Move everything after begin into one shared body, `run_recorded_body(ctx)`,
  where `ctx` is a small frozen dataclass carrying `run_id`, `recorded`, `resolved`,
  `has_owner`, `owns_output`, `compact`, `tail_lines`, `events_path`, `stdout_path`,
  `stderr_path`, and an optional `stop_recorded: Callable[[], bool]`. The body keeps the
  pre-spawn signal check, spawn, observe, pumps, sinks, stage ingestion, sampler,
  fingerprints, finish, footer, and metrics byte-for-byte; the only behavior change is
  `terminal_cause` on every finish it writes:
  - pre-spawn signal: `interrupt` (SIGINT) / `signal` (SIGTERM), or `stop_requested`
    when `stop_recorded()` is true;
  - spawn failure 127/126: `launch_failed`;
  - after wait: derived from `settle_wait_code`'s state — `interrupted` → `interrupt`,
    `signaled` → `signal`, `succeeded`/`failed` → `exited`. Existing executor tests must
    stay green unmodified; add assertions that foreground runs now record
    `terminal_cause` (`exited`, `launch_failed`, `interrupt`).
- `argv.py`: `resolve_run_argv(words, *, cwd: str | Path | None = None)`. `None` keeps
  today's path exactly (`load_project_tool_catalog()`, `discover_project_root()`). A
  given `cwd` uses `load_project_tool_catalog_at(cwd)` and `discover_project_root(cwd)`;
  for an ad-hoc run with an explicit cwd, `ResolvedToolArgv.cwd` is that directory so
  the frozen envelope never depends on the worker's cwd. Foreground callers pass
  nothing.
- Keep `executor.py` under the toobig thresholds; if the split pushes it over, move the
  shared body into `src/sase/tool/executor_body.py`.

### 2. Launch module (`src/sase/tool/handoff.py`)

Single module both legs (this phase's `-H`, phase 3's monitor start) use:

- `envelope_from_resolved(resolved)` / `resolved_from_envelope(envelope)` (tuples ↔
  lists; `cwd`, `digest`, `adhoc` preserved).
- `reserve_handoff_run(resolved, *, owner_kind, owner_id) -> HandoffReservation`
  (frozen: `run_id`, `owner_kind`, `owner_id`, `events_path`, `error: str | None`,
  property `reserved`). It mints the run id, prepares the events path with
  `prepare_run_paths(run_id, owns_output=False)`, and calls `tool_run_begin` with
  `build_begin_request(...)` plus `launch_mode: "handoff"`, `launch: envelope`,
  `commit_running: False`, and no `parent_run_id` (a hand-off is an ownership root). Any
  exception or a non-`created` result becomes `error`; nothing is raised.
- `worker_argv(run_id)` → `[sys.executable, "-m", "sase", "tool", "_adopt", run_id]`.
- `worker_env_overlay()` → `{"SASE_TOOL_RUN_ID": "", "SASE_TOOL_RUN_EVENTS": ""}`.
- `owner_tags(run_id)` → `["tool-run", f"tool-run:{run_id}"]`;
  `owner_request_fingerprint(run_id)` → `f"tool-run:{run_id}"`.
- `settle_launch_failure(run_id, message)` →
  `finish_tool_run(state="failed", exit_code=None, terminal_cause="launch_failed", diagnostics=["command was not run", message])`,
  returning the bool.

### 3. Adopting worker (`sase tool _adopt RUN`)

- Parser: hidden `_adopt` subparser (`help=argparse.SUPPRESS`, positional `RUN`) in
  `src/sase/main/parser_tool.py`; the public metavar list stays `{list,run,runs,show}`.
  `tool_handler.py` dispatches it lazily to the new module.
- `src/sase/tool/adopt.py::execute_adopted_run(run_id) -> int`:
  - owner from `SASE_MONITOR_ID` (kind `monitor`) else `SASE_PROC_ID` (kind `proc`);
    neither → print "sase tool _adopt: no owner in the environment; run nothing" and
    exit 2 without touching the store;
  - install SIGINT/SIGTERM handlers (`SignalState`) **before** claiming;
  - claim with the worker's pid/boot/process identity and
    `owner_log_path=SASE_PROC_LOG_PATH`; ignore inherited `SASE_TOOL_RUN_ID`/`EVENTS`;
  - `refused` → print
    `sase tool _adopt: claim refused for RUN (<refusal>); command was not run` to
    stderr, exit 2; `stopped` → print that the run was stopped before it started, exit
    143 (nothing spawned); a claim exception → print it, exit 1, run nothing (never run
    without a claim);
  - `claimed` → rebuild `ResolvedToolArgv` from the envelope, build ownership in owner
    mode (`owns_output=False`, `compact=False`, `parent_run_id=None`), take the events
    path from the claimed run's `logs.events_path`, and call the shared body with
    `stop_recorded` = "the run has a `stop_request` (via `tool_run_show`) or the owner
    proc row has `stop_requested_at`". Worker stderr lines (`sase tool run RUN`, stage
    lines, footer) are therefore unchanged.

### 4. `sase tool run -H/--hand-off`

- Parser: add `-H/--hand-off` (store_true) to `run`; change `-T` default to `None` so an
  explicit `-T` is detectable (handler resolves `None` → 200). Update the run
  description/epilog: `-H` hands off to a durable proc and returns at once; options must
  precede `TOOL`/`--`; with `-H`, `-q` prints only the run id, `-v` and `-T` are usage
  errors; exit codes 0 accepted, 1 not started (reservation or submit failed), 2 usage
  or refusal. `ToolRunCliRequest` gains `hand_off: bool = False` and
  `tail_lines_explicit: bool = False` (defaults keep existing callers valid).
- `src/sase/tool/handoff_launch.py::execute_handoff(request) -> int`, reached from
  `execute_tool_run` via a lazy import (keeps `test_import_weight` green):
  1. flag off → one line naming `tool_handoff` and `sase flag enable tool_handoff`, exit
     2;
  2. `-v` or explicit `-T` → usage error, exit 2;
  3. `SASE_AGENT` set → exit 2 and print exactly
     `sase monitor start -p verify --reason '<why>' -- sase tool run <words>` (words
     shell-quoted); `resolve_ownership(quiet=False)` finds a live monitor/proc owner →
     exit 2 naming that owner; when the owner is a proc whose `origin` is `ace` (a TUI
     `!` command) the message says it is already a detached proc and to drop `-H`.
     Nothing is reserved in any refusal;
  4. resolve with `resolve_run_argv(words, cwd=Path.cwd())` (usage error → exit 2);
  5. pre-allocate `proc_id = new_proc_id()`;
     `reserve_handoff_run(owner_kind="proc", owner_id=proc_id)`; on failure print that
     nothing was started, the reason, and the foreground form `sase tool run <words>`,
     exit 1 (fail-closed);
  6. `submit_proc_request(ProcSubmitRequest(argv=worker_argv, command=<redacted logical form: sase tool run NAME [extra display args] | sase tool run -- <display_argv>>, label=f"tool:{name}" or "tool:ad-hoc", cwd=Path.cwd(), origin="tool-run", proc_id=proc_id, project/workspace_num/session_id from the shared attribution helper, tags=owner_tags, env=worker_env_overlay(), request_fingerprint=owner_request_fingerprint, followup={"kind": "tool-run", "run_id": run_id}))`;
     `ProcSubmitError`/any exception → `settle_launch_failure(run_id, str(exc))`, print,
     exit 1;
  7. acknowledge after `submit_proc_request` returns (reservation committed, barrier
     released): `-q` prints only the full run id; otherwise print the run id, tool, proc
     id, a note that the run may still be starting, and hints `sase tool show RUN -F`,
     `sase tool wait RUN`, `sase tool stop RUN` (plus `sase proc show <proc> --follow`,
     which works today). Exit 0. The proc leg relies on the proc runner's existing
     `detach_scope`; no tool-local cgroup code.
- Shared attribution: move `_infer_attribution` from `proc_handler.py` into a public
  `infer_proc_attribution(cwd, project)` in `src/sase/procs/attribution.py` (exported
  from `sase.procs` the same way its siblings are); `proc_handler.py` and the hand-off
  both call it. Session id: `resolve_session_ref(None)` with `SessionRefError` → `None`.
  `sase proc run` behavior stays identical.

### 5. Liveness (`src/sase/tool/liveness.py`)

For an unsettled run with `state == "created"` and `launch_mode == "handoff"`, a `dead`
wrapper (launcher) observation is downgraded to `unknown` with reason "launcher exit is
not proof of launch failure" unless an owner fact accompanies it (none do until phase
`settlement`, which may pair them). Unit-test that fact shape.

### 6. Symvision

Remove `--epic-symbol 'sase-17p(tool_run_claim)'` from the Justfile and reword its
comment to name only `tool_run_request_stop` / phase sase-17p.4. Every new public symbol
above has a non-test caller in this phase; make anything else private. Re-run
`sase bead epic-symbols sase-17p.2` before closing.

## Tests

New `tests/tool/test_handoff.py` (and, if it grows past toobig limits,
`tests/tool/test_adopt.py`), isolated `SASE_HOME`, private cwd, `SASE_AGENT`,
`SASE_AGENT_NAME`, `SASE_MONITOR_ID`, `SASE_PROC_ID`, `SASE_TOOL_RUN_ID` removed, flag
toggled with `override_flags` / `SASE_FEATURE_FLAGS`:

- worker paths with a fake owner env: claimed (runs the frozen argv, records
  `owner_kind=proc`, `owner_log_path`, `terminal_cause=exited`), refused
  (`owner_mismatch` → exit 2, nothing spawned — assert via a marker file the command
  would create), stopped (stop request before claim → `signaled`/`stop_requested`,
  nothing spawned), and no owner env → exit 2 with the store untouched;
- `-H` end to end through a real proc supervisor: returns before a slow fixture command
  (sleep then `exit 7`) finishes; `wait_for_proc` then the run settles `failed` with
  exit 7 under `owner_kind=proc`, `launch_mode=handoff`; exactly one ToolRun and one
  proc tagged `tool-run:RUN`; `-q` prints only the run id;
- a secret-bearing ad-hoc argv (`--token SECRET`) never appears in `sase proc show -j`
  output / the proc row (argv is only the worker argv, command is redacted);
- an unwritable store (point `SASE_HOME/tools` at a non-writable path or make
  `tool_run_begin` raise) → `-H` exits 1 and no proc row exists, while foreground
  `sase tool run -- true` in the same state still exits 0 (fail-open);
- a catalog edit between reservation and claim still executes the frozen argv (reserve
  via `reserve_handoff_run`, rewrite `sase/sase.yml`, then run the worker in-process
  with a fake owner env);
- submit failure → run settles `failed`/`launch_failed` with "command was not run";
- refusals: inside an agent (prints the exact monitor form, exit 2, no run), inside a
  live proc owner (exit 2), TUI `!` proc owner message; `-v` and explicit `-T` usage
  errors;
- both flag states (off: exit 2 naming `tool_handoff`, no run, no proc);
- foreground `terminal_cause` assertions added to `tests/tool/test_executor.py`;
- liveness downgrade in `tests/tool/test_liveness.py`;
- `resolve_run_argv(..., cwd=...)` resolves another directory's catalog in
  `tests/tool/test_argv.py`;
- parser: `_adopt` hidden from `sase tool --help`, `-H` present in `sase tool run -h`.

## Verification

`just fmt`, then `sase tool run check` (generous explicit timeout) from the sase
checkout. Because master is often red for unrelated reasons, judge the phase by the
targeted tests above plus a manual smoke: in a temp git project with an isolated
`SASE_HOME` and the flag enabled, `sase tool run -H -- sh -c 'sleep 2; exit 3'` returns
immediately, and `sase tool show RUN -j` later shows `failed`, exit 3,
`owner_kind: proc`, `launch_mode: handoff`, `terminal_cause: exited`.

## Out of scope (other phases)

Monitor-start reservation (sase-17p.3), `stop`/`show -F`/`wait` and `show` rendering of
the new fields (sase-17p.4), owner facts, timeout/stop causes after spawn, and
notifications (sase-17p.5), docs, memory, skill source, and flag removal (sase-17p.6).
Record anything else discovered as `PROPOSED FOLLOW-UP:` notes on sase-17p.2; do not
create task beads.
