---
tier: tale
title:
  "E2 settlement: settle hand-off ToolRuns truthfully after crashes and deliver once"
goal:
  Every handed-off ToolRun settles from owner facts, stop/timeout intent, and proc or
  monitor settlement to an authoritative outcome or typed uncertainty, proc-owned
  hand-offs publish exactly one notification, and expired owner logs are reported
  explicitly.
size: medium
proposed_by: bbugyi200.athena.sase-17p.5
bead: sase-17p.5
status: done
---

- **PARENT:**
  [202609/tool_e2_durable_handoff.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e2_durable_handoff.md)
- **BEAD:**
  [sase-17p.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17p/sase-17p.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-17p.5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-17p.5.md)
- **COMMITS:**
  - [7173669](https://github.com/sase-org/sase/commit/71736697dcb97b14cceb96cf687dfa24d6454d17)
    — feat(tool): settle hand-off ToolRuns from owner facts and deliver once
    (sase-17p.5)

# Phase sase-17p.5 (settlement): settle hand-off runs truthfully after crashes and deliver once

Parent epic: `sase-17p` (E2 durable ToolRun hand-off), epic plan
`plan:202609/tool_e2_durable_handoff.md`, section "5. settlement" plus the binding
contracts "Reconciliation", "Output, logs, and retention", and decisions 7–9. Phases
core-contract, standalone-handoff, monitor-handoff, and lifecycle-controls are landed.

## What already exists (do not redo)

- sase-core (pin `9956773…`, already in `sase-core-revision.txt`) implements the whole
  owner-aware reconcile table in `tool_run/store/reconcile.rs`:
  `ToolRunLivenessFactWire` has an optional
  `owner: {kind, id, state: active|terminal|missing|unknown, exit_code?, termination_reason?, stop_requested?}`.
  The reconcile result has a `settled` list of
  `{run_id, state, terminal_cause, settled_by}`. Core emits reap candidates only for
  runs with no owner. Finish accepts `terminal_cause=timeout` with state `signaled`, and
  it allows an exit code and signal alongside it. **No sase-core change is needed in
  this phase.** If one turns out to be required, stop and record it as a
  `PROPOSED FOLLOW-UP:` note; do not edit core.
- `src/sase/tool/liveness.py` sends only wrapper facts today. It downgrades a dead
  launcher for a `created` hand-off to `unknown`. It also looks at `wrapper_pid`, which
  is NULL for a `created` hand-off: core stores the launcher identity in
  `run["launcher"]` (`{pid, boot_id, process_start_identity}`).
- `src/sase/tool/adopt.py` (worker) and `src/sase/tool/executor.py`
  (`run_recorded_body`, `RecordedRunContext.stop_recorded`) already record `interrupt`,
  `signal`, `exited`, and `stop_requested`.
- The proc supervisor (`src/sase/procs/supervisor.py`) records `termination_reason`
  values `success`, `error`, `stop`, `total-timeout`, `idle-timeout`, `launch-failure`,
  and `supervisor-loss`. Proc reconcile (`procs/submission.py::_reconcile_proc_shell`)
  adds `reboot`. Every path ends in `settle_proc_shell` (`procs/settlement.py`), whose
  `_settle_followup` checkpoint handles the monitor kind. The hand-off proc is submitted
  with `followup={"kind": "tool-run", "run_id": RUN}` (`tool/handoff_launch.py`).
- A monitor's proc row has `proc_id == monitor_id`, and monitor meta carries
  `monitor_tool_run_id`. Monitor settlement runs through
  `monitor/proc_adapter.py::settle_monitor_followup(state)`.
- `sase tool show -l` already prints "proc owner log is missing or expired" when the row
  exists but the file does not. When the proc row is pruned, it only prints "proc owner
  … was not found". `show` (human and `-j`) says nothing about owner retention.

## Changes

### 1. Termination intent in the proc supervisor (`src/sase/procs/`)

- `runtime.py`: add `proc_termination_intent_path(proc_id)`
  (`<runtime dir>/termination-intent.json`),
  `write_termination_intent(proc_id, intent)`, and a reader
  `read_termination_intent(proc_id) -> str | None`. Intent is one of `stop`,
  `total-timeout`, `idle-timeout`, stored with a `timestamp`. The first write wins:
  never overwrite an existing file. The writer and the reader both swallow `OSError`
  (the writer must never break the signal path) and use `write_json_atomic` /
  `read_json_object`. Add them to `__all__`.
- `supervisor.py`: give `_Termination` the proc id. `request()` (the SIGTERM/SIGINT
  handler) writes intent `stop` **before** `_signal_child()`. `trigger_timeout(kind)`
  takes the `_TimeoutKind` and writes `f"{kind}-timeout"` before it signals. Update
  `_wait_for_child` to pass the kind. The outcome dicts and `termination_reason` values
  are unchanged.

### 2. Owner facts (`src/sase/tool/owner.py`, new)

One module owns owner observation for both reconcile and `show`:

- `observe_owner_fact(run) -> dict | None`: returns a fact only for
  `launch_mode == "handoff"` runs whose `owner_kind` is `proc` or `monitor`, or `None`
  otherwise. Foreground and nested runs never get owner facts (epic "Reconciliation").
  It reads `sase.procs.store.get_proc(owner_id)` lazily; monitor rows share the id.
  - A missing row → `missing`.
  - Any exception → `unknown`.
  - Status in `TERMINAL_PROC_STATUSES` → `terminal`, with `exit_code`,
    `termination_reason` from `proc.result["termination_reason"]` (when it is a dict),
    and
    `stop_requested = bool(proc.stop_requested_at) or read_termination_intent(id) == "stop"`.
  - Anything else (including `settling`) → `active`.
- `owner_fact_from_settlement(kind, owner_id, state) -> dict`: builds a `terminal` fact
  from a proc settlement `state` dict (`exit_code`, `termination_reason`,
  `stop_requested`), with `stop_requested` also true for intent `stop`. Settlement runs
  before `finish_proc`, while the row still reads `settling` (active), so settlement
  hooks must pass the fact they already hold.
- `owner_retention(run) -> dict`:
  `{"owner": "none"|"retained"|"pruned"|"unknown", "log": "none"|"retained"|"missing"|"not-recorded", "log_path": str|None}`.
  The log path is the recorded `logs.owner_log_path`, else the live row's `log_path`.
  For monitors, fall back to the existing `_monitor_output_path` reader in `control.py`.
  A missing proc row for a proc or monitor owner is `pruned`.

### 3. Reconcile with owner facts (`src/sase/tool/liveness.py`)

- In `reconcile_unsettled_tool_runs`, attach `fact["owner"] = observe_owner_fact(run)`
  when it is non-`None`.
- For a `created` hand-off, observe the **launcher**: build the liveness fact from
  `run["launcher"]` (pid, boot_id, process_start_identity) with the same identity and
  boot checks as `_observe_wrapper`. Refactor the pid/identity check into a helper both
  paths share; do not duplicate it. Report a dead launcher truthfully (`dead`) **only**
  when the owner fact is `missing` or `terminal`. Otherwise keep today's `unknown`
  downgrade, so a dead launcher alone is never proof. Core then applies "owner missing
  and launcher dead → `launch_failed`".
- Defense in depth in `_maybe_reap`: skip, with a diagnostic, any candidate whose run
  has an `owner_kind`. Collect the owned run ids while listing. Python never signals a
  group whose run has an owner, even if core ever emitted one.
- After a successful reconcile, pass every `result["settled"]` entry to the delivery
  helper (§6). It filters to proc-owned hand-offs itself. Failures are swallowed into
  diagnostics.
- Add `reconcile_handoff_run(run_id, owner_fact) -> dict`, a targeted single-run pass
  for settlement hooks. It loads the run with `tool_run_show`, returns early unless the
  run is an unsettled hand-off, and builds the worker or launcher liveness fact as above
  with the supplied owner fact. It then calls `tool_run_reconcile` with that one fact,
  never reaps, never raises (returns a diagnostics envelope), and never launches
  anything.

### 4. Worker: stop and timeout causes from the owner's intent; finish race

- `executor.py`: add `timeout_recorded: Callable[[], bool] | None = None` to
  `RecordedRunContext` with a `_timeout_requested(ctx)` twin of `_stop_requested`. Stop
  keeps precedence. In the pre-spawn signal branch, a timeout settles `signaled`, exit
  143, SIGTERM, cause `timeout`. After wait, a `signaled` state records
  `stop_requested`, then `timeout`, then `signal`. `timeout` keeps the exit code and
  signal (core allows it). The foreground path is unchanged: it passes no timeout probe.
- `adopt.py`: read the intent through `read_termination_intent(SASE_PROC_ID)`. The
  supervisor sets `SASE_PROC_ID` for monitor procs too, and the id equals the monitor
  id.
  - The stop probe is true for intent `stop`, a durable ToolRun stop request, or
    `proc.stop_requested_at`. It is false when the intent is a timeout: first intent
    wins.
  - The new timeout probe is true for `total-timeout` or `idle-timeout`.
  - Remove the dead `_ = ownership` scaffolding only if it is truly unused; keep
    behavior.
- `executor_recording.py::finish_tool_run`: when `tool_run_finish` raises, check
  `tool_run_show(run_id)`. If the run is already terminal (reconcile or another settler
  won the race), return `True`: "already settled" is success, never an
  incomplete-evidence warning. Otherwise return `False` as today.
  `handoff.settle_launch_failure` inherits this.
- After `run_recorded_body` returns in the worker, if `owner_kind == "proc"`, call the
  delivery helper (§6) for the run. It is best-effort and never changes the exit code.

### 5. Settle from the owner's side

- `procs/settlement.py::_settle_followup`: add a `tool-run` branch after the monitor
  check, taken when `policy.get("kind") == "tool-run"`. Lazily import
  `sase.tool.settlement.settle_tool_run_followup(state)` (new module). That function:
  - builds `owner_fact_from_settlement("proc", state["proc_id"], state)`;
  - calls `reconcile_handoff_run`;
  - then calls delivery for proc-owned hand-offs whenever the run is settled, even when
    the worker already settled it;
  - sets `state["followup_outcome"]` to `tool-run-settled`, `tool-run-unsettled`, or
    `tool-run-error`, with `state["followup_error"]` on error.

  The branch must be wrapped so no ToolRun or notification error propagates: settlement
  must never wedge. It must also stay idempotent under the existing checkpoint resume.
  The existing `stop`/`reboot`/`supervisor-loss` "suppressed" rule stays for
  non-tool-run policies only.

- `monitor/proc_adapter.py::settle_monitor_followup`: before the follow-up decision
  (before the `if meta.get("monitor_followup_outcome")` block, so resumed settlements
  also reconcile), when `meta.get("monitor_tool_run_id")` is set, call
  `settle_monitor_tool_run(run_id, monitor_id, state)` from `sase.tool.settlement`. It
  reconciles with `owner_fact_from_settlement("monitor", monitor_id, state)`, is
  best-effort, **never notifies**, and never touches the continuation.

### 6. Deliver once (`src/sase/tool/notify.py`, new)

`deliver_handoff_settlement(run_id | run) -> str` returns `published`,
`already_published`, `skipped`, or `failed`.

- It acts only when the run is settled, `launch_mode == "handoff"`, and
  `owner_kind == "proc"`. Monitor-owned runs always return `skipped`.
- The id is `str(uuid5(NAMESPACE_URL, f"sase:tool-run-settled:{run_id}"))`.
- Follow the exactly-once pattern of
  `bead/epic_launch_handoff.py::_publish_deferred_monitor_completion`:
  - hold `log_file_lock(tools_dir() / "tool_run_notifications")` (one shared lock file);
  - check durability with `load_notifications(include_dismissed=True)` (same logic as
    `epic_launch_handoff_notifications.notification_is_durable`, imported lazily or
    reimplemented in a few lines);
  - `append_notification`;
  - on exception, re-check durability before reporting `failed`.
- The notification is `sender="tool-run"`, `tags=["tool-run"]`, with notes carrying:
  - the tool name (`ad-hoc` when unnamed), state, terminal cause, human duration, and
    exit code when present;
  - one line `sase tool show RUN`.

  `action_data={"run_id": RUN, "command": "sase tool show RUN"}`, and `files` holds the
  owner log path when it exists. Leave `action` as `None`. There is no generic "run
  command" action, and an unknown action string makes the TUI warn. A TUI action is E5
  scope, so record it as a `PROPOSED FOLLOW-UP:`.

- Callers: the worker (§4), proc settlement (§5), and reconcile passes that settle a run
  (§3). The one id collapses them.

### 7. Report expired or pruned owner logs explicitly (`src/sase/tool/query.py`, `control.py`)

- `show` / `show -F` final envelope: add
  `envelope["owner_retention"] = owner_retention(run)`. For the human view, add an
  `OWNERRET` line such as `proc 2f3a…: pruned; log not retained` or
  `retained; log missing`. It is printed only for owner-bound runs.
- `show -l`: when the proc row is gone, fall back to the recorded `logs.owner_log_path`.
  Replay it if the file still exists; otherwise print
  `sase: proc owner <id> is no longer retained (pruned); its log <path> is missing`.
  Monitors behave the same way. Keep the existing messages for the
  row-exists/file-missing case.
- `show -F`: when an owner-bound run has no readable output-of-record path, print one
  explicit stderr notice (owner or log no longer retained) once, instead of streaming
  nothing silently.

### 8. sase-145

sase-145 is still `ready`. The phase-1 note already records its acceptance. Verify that
`sase tool show RUN -j` and the human `DIAG` block render persisted spawn, ingest, and
log-write diagnostics; add a rendering test only if one is missing. Per this worker's
instructions (close only sase-17p.5), **do not close sase-145**. Record
`PROPOSED FOLLOW-UP: close sase-145 — <evidence>` on sase-17p.5 instead.

## Tests

Add them in a new `tests/tool/test_settlement.py`, plus focused additions to
`tests/tool/test_liveness.py`, `tests/procs/` (the supervisor-intent test next to the
existing supervisor tests), and `tests/monitor/` for the monitor hook. Use an isolated
`SASE_HOME` (see `_clean_env` in `tests/tool/test_handoff.py`) and fixture commands
only. Each epic test row maps to a test:

1. Kill the launcher after reservation and before submit: a reserved proc-owned run with
   a dead launcher identity and no proc row → `failed`/`launch_failed`, "command was not
   run".
2. Launcher exits after submit: `-H` returns, and the run settles normally through the
   worker (extend or reuse the e2e in `test_handoff.py`).
3. Worker dies while the owner is alive and the owner settles with a non-negative exit
   code (simulate: a claimed run with a dead worker identity and a terminal proc owner
   fact `error`/exit 3) → `failed`, exit 3, `settled_by: owner`, diagnostic "recovered
   from owner result…". Also cover a worker SIGKILLed by signal (owner exit code
   negative): assert what core decides (`lost`, no fabricated success, nothing re-run).
   Note: core maps that to `owner_lost`; if that reads wrong, record a
   `PROPOSED FOLLOW-UP:` for core rather than changing it.
4. Supervisor SIGKILLed or supervisor-loss → `lost`/`owner_lost`, no rerun (settle via
   `reconcile_proc_shells` or a `supervisor-loss` settlement fact).
5. Total and idle timeouts (real supervisor with a tiny `timeout_seconds` /
   `idle_timeout_seconds`, or a worker with a timeout intent file plus SIGTERM) →
   `signaled`/`timeout`. Also a unit test that the supervisor writes the intent before
   signaling, and that the first intent wins.
6. Stop before the barrier → `signaled`/`stop_requested`, "command was not run".
7. A worker finish that fails on a busy or raising store → recovered from the owner
   through the settlement hook. Also a unit test that `finish_tool_run` returns `True`
   when the run is already terminal.
8. Repeated reconcile passes and a resumed settlement (call `settle_tool_run_followup`
   twice, plus worker delivery) → exactly one notification with the deterministic id.
   Monitor-owned runs produce none.
9. Pruned owner row → `show -j` has `owner_retention.owner == "pruned"`, the summary is
   intact, and `show -l` names the missing log.
10. Reboot via a boot-id mismatch on the worker identity plus a `reboot` owner fact →
    `lost`, nothing re-executed.
11. `_maybe_reap` never signals an owned run's candidate (monkeypatch `os.killpg` and
    assert it is not called).
12. Proc settlement hook errors (monkeypatch the reconcile to raise) still finish the
    proc row.

The existing suites must stay green untouched: `tests/tool/`,
`tests/core/test_tool_run_store.py`, `tests/monitor/test_monitor_tool_handoff.py`,
`tests/procs/`, and `tests/monitor/`.

## Verification and bookkeeping

- Read `sase/memory/lint_and_test.md` (via `/sase_memory_read`) before finishing, and
  run the verification it prescribes (`sase tool run check`). Also run targeted pytest
  for the suites above.
- New public symbols must have callers (see `sase/memory/symvision.md`). Run
  `sase bead epic-symbols sase-17p.5`, which must stay empty.
- Add a CHANGELOG entry if the lint rules require one.
- No config fields, no CLI options, and no memory or skill-source edits in this phase.
  Docs, memory, and the flag removal belong to phase acceptance-and-adoption.
- Record `PROPOSED FOLLOW-UP:` notes on sase-17p.5 for sase-145 closure, the TUI
  notification action, and anything else discovered. Close only sase-17p.5.
