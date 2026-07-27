---
tier: tale
title: Defer the planner completion notification until the epic launch settles
goal: A planner agent whose epic plan was approved sends its completion notification
  only once the detached `sase bead work` launch task has settled and the row has
  moved from EPIC APPROVED to EPIC CREATED, and no completion notification is ever
  lost when nothing settles the launch.
create_time: 2026-07-27 06:50:33
status: wip
---

- **PROMPT:** [202607/prompts/defer_epic_completion_notification.md](prompts/defer_epic_completion_notification.md)

# Defer the planner completion notification until the epic launch task settles

## Goal

When a planner agent's epic plan is approved, the agent's "completed" notification must not fire until the detached
`sase bead work` launch task has settled — i.e. until the planner row's status has moved from `EPIC APPROVED` to
`EPIC CREATED`. Today the notification fires the moment the planner runner finalizes, while the launch is still running.

## Background

Approving an epic plan runs two independent processes:

1. **Host side.** The gate side effect submits a detached background task
   (`sase bead work <plan> --yes-to-all --artifacts-dir <dir> --cl-name <name>`) via `submit_epic_launch_task`
   (`src/sase/bead/epic_launch.py:101`). The gate response is written _before_ side effects run
   (`src/sase/notification_gates/executor.py:205` then `:212`), so the planner runner unblocks from `wait_for_gate`
   before the task id is even known.
2. **Runner side.** The planner runner sees `action == "epic"`, returns the `epic_approved` loop outcome
   (`src/sase/axe/run_agent_exec_plan_accept.py:460`), and finalizes. `send_completion_notification`
   (`src/sase/axe/run_agent_runner_finalize.py:205`) returns early only for `plan_rejected`, so the
   `"<model> @<agent> completed: <workflow>"` notification is emitted immediately.

Later, the detached task finishes and `finish_epic_launch` (`src/sase/bead/epic_launch.py:207`) back-fills
`epic_bead_id` into `agent_meta.json`, which is what flips the TUI row to `EPIC CREATED`
(`src/sase/ace/tui/models/_agent_status_family.py:307-317`), and sends a separate `epic-launch` notification.

So the user currently gets the completion toast early and a second `epic-launch` toast later. The fix is to hand the
completion notification off to whoever settles the launch.

## Design

Introduce a small durable handoff store keyed by planner-agent identity. The runner _defers_ its completion notification
into the store instead of sending it; the process that settles the epic launch _claims_ it and sends it, folding the
launch outcome into the notes. A backstop chop flushes any deferral that nothing ever claimed, so a notification can
never be silently swallowed.

### Correlation key

Both sides know a planner artifacts directory, but not necessarily the same path string: the runner may have promoted
the agent into a workflow directory (`promote_to_workflow`), while the host resolved the dir from the notification's
`agent_timestamp` (`resolve_plan_agent_artifacts_dir`, `src/sase/_plan_approval_artifacts.py:37`). Derive the key from
the parsed identity instead of the raw path, using the existing Rust-backed parser `parse_agent_artifact_path`
(`src/sase/core/agent_artifact_paths.py:88`), which returns `project_name` and `timestamp` and already handles both the
legacy and day-sharded layouts:

```
key = f"{project_name}__{timestamp}"
```

If the path does not parse, there is no usable key — fail open and send the notification immediately.

### Store

A new directory under the SASE notifications root (use `sase.core.paths.sase_subdir("notifications")` so it sits next to
`notifications.jsonl`), e.g. `notifications/epic_completions/`, holding two flat JSON files per key:

- `<key>.pending.json` — `{"key", "artifacts_dir", "created_at", "plan_file"?, "payload": {...}}` where `payload` is the
  fully-built completion notification (`sender`, `cl_name`, `success`, `notes`, `action`, `action_data`, `extra_files`,
  `silent`, `tags`) — every field is already JSON-safe.
- `<key>.settled.json` — `{"success", "epic_id", "plan_file", "detail", "settled_at"}`.

Serialize both sides' critical sections with the existing sibling-lock helper
`sase.logs._bounded.log_file_lock(<pending path>)` (`src/sase/logs/_bounded.py:67`), which is already the primitive
`submit_epic_launch_task` uses.

### Protocol

**Runner (defer).** In `send_completion_notification`, split payload construction from sending, then:

```
if outcome == "epic_approved":
    with lock(key):
        settled = pop(<key>.settled.json)      # launch already finished
        if settled is None:
            write(<key>.pending.json, payload)
            return                              # do not send now
    # settled: fall through and send immediately; the launch outcome was
    # already reported by finish_epic_launch, so do not re-fold it into notes
```

Every failure in this block (unparseable path, OSError, lock failure) must fall through to the existing immediate send.
Never swallow a notification because the handoff store misbehaved.

**Launch task (claim).** In `finish_epic_launch`, after the existing `_update_epic_launch_metadata` back-fill so the row
is already `EPIC CREATED` when the toast lands:

```
with lock(key):
    pending = pop(<key>.pending.json)
    if pending is None:
        write(<key>.settled.json, outcome)
if pending is not None:
    send(pending.payload, with the epic outcome folded into notes)
    # skip the standalone epic-launch notification — one toast, not two
else:
    send the existing epic-launch notification unchanged
```

Folded notes on success: append `Epic <epic_id> launched from <plan>.md` and `Plan: <archived_plan_path>` to the
existing completion notes. On failure: append `Epic launch failed: <detail>` and `Resume with: <argv>`, and drop the
`done` tag so the row is not presented as a clean success. Keep `action = JumpToAgent` so `<enter>` still jumps to the
planner row.

`finish_epic_launch` returns early when `artifacts_dir is None and not cl_name` and when `result.dry_run` is set; those
paths must not write a settle marker, and correspondingly the runner must not have deferred for a dry run (it cannot — a
dry run never produces an approved epic outcome).

### Backstop chop

A deferral is orphaned when nothing ever calls `finish_epic_launch` for it: `epic_launch_mode == "skip"`, a host-side
`prepare_epic_launch` submit failure (the gate response already exists at that point, so the runner still reports
`epic_approved`), a `sase task kill` on the launch, or a crash/reboot. Add a builtin chop that flushes those.

- New `src/sase/scripts/sase_chop_epic_launch_flush.py` following the `sase_chop_managed_tmp_reap.py` shape
  (`@builtin_chop(...)`, `runtime.emit_summary(...)`, `main()` calling `run_builtin_chop`).
- Register the console script in `pyproject.toml` alongside the other `sase_chop_*` entries.
- Add the chop to the `waits` lumberjack lane in `src/sase/default_config.yml` (10s interval) with `run_every: "30s"`,
  and give it the two-paragraph summary-first description the other chops use.
- Logic: for each `<key>.pending.json` older than a grace period (90s), flush it unless an active detached task tagged
  `epic`+`launch` still owns it. Ownership is read straight off the task row — `task.command` contains
  `--artifacts-dir <dir>`, so parse that dir through the same key derivation and compare keys
  (`read_tasks(status=ACTIVE_TASK_STATUSES, kind=DETACHED_TASK_KIND)`, as `_active_epic_launch_for_plan` already does).
  Flushed notes say the launch outcome is unknown and include the `build_epic_launch_argv` resume command.
- Also reap `<key>.settled.json` files older than a longer TTL (1 hour) so unclaimed settle markers do not accumulate.

## Steps

1. **Add the handoff module.** Create `src/sase/bead/epic_launch_handoff.py` with the key derivation, store paths, and
   the three locked primitives: `defer_epic_completion(artifacts_dir, payload) -> bool`,
   `claim_epic_completion(artifacts_dir, *, outcome) -> DeferredCompletion | None`, and the sweep helper the chop uses
   (`iter_orphaned_deferrals(...)` / `flush_orphaned_deferrals(...)`). Keep every public entry point fail-open: on any
   exception the caller must be told to send immediately.

2. **Split payload construction in the runner.** Refactor `send_completion_notification`
   (`src/sase/axe/run_agent_runner_finalize.py:205`) so it builds a serializable payload dataclass and then sends it,
   and insert the `epic_approved` deferral branch. Keep the existing `plan_rejected` early return and the existing
   signature — the lifecycle call site (`src/sase/axe/run_agent_runner_lifecycle.py:272`) and its tests pass keyword
   arguments and must keep working unchanged.

3. **Claim on settle.** Update `finish_epic_launch` (`src/sase/bead/epic_launch.py:207`) to claim the deferral after the
   metadata back-fill, send the folded completion notification when it claims one, write the settle marker when it does
   not, and keep the standalone `epic-launch` notification only on the not-claimed path. Preserve the existing
   best-effort posture: a handoff failure must never turn a successful launch into a CLI error.

4. **Add the backstop chop.** New chop script, `pyproject.toml` entry, `default_config.yml` lane entry, plus the
   `docs/axe.md` chop table row and the `docs/configuration.md` lumberjack listing that mirror the config.

5. **Verify no collateral artifact-dir coupling.** The handoff files live under the notifications root, not the agent
   artifacts dir, so no marker wire, artifact index, or dismissed-bundle change is needed. Confirm this by checking that
   nothing in `src/sase/core/agent_scan_wire_markers.py` or the explicit-artifact listing used by
   `_completion_explicit_artifact_paths` enumerates unknown files.

## Testing

Unit tests (new files under `tests/`, matching existing naming):

- Key derivation agrees across a runner-side promoted workflow dir and a host-resolved `ace-run` dir for the same agent,
  and returns `None` for a non-artifact path.
- Runner defers on `outcome == "epic_approved"`: no notification is appended, a `<key>.pending.json` exists, and its
  payload round-trips to the same arguments the immediate path would have used.
- Runner sends immediately when the store is unusable (patch the store dir to a read-only/nonexistent path) — the
  fail-open guarantee.
- `finish_epic_launch` claims a pending deferral: exactly one notification is appended, it carries `JumpToAgent` and the
  planner's `cl_name`/`raw_suffix`, its notes contain both the original completion line and the epic line, and no
  separate `epic-launch` notification is sent.
- `finish_epic_launch` with no pending deferral writes a settle marker and sends the existing `epic-launch`
  notification; a runner that defers afterwards consumes the marker and sends its completion notification once, without
  re-reporting the epic outcome.
- Failure path: `finish_epic_launch(error=...)` claims the deferral, appends the failure detail and resume command, and
  omits the `done` tag.
- Chop: a pending deferral younger than the grace period with an active epic-launch task is left alone; one past the
  grace period with no owning task is flushed exactly once; stale settle markers are reaped.

Add a chop output-contract case in `tests/test_axe_chop_output_contract.py` and register the chop name in
`tests/test_chop_sdk.py` alongside the existing builtins.

Then run `just install` followed by `just check`.

## Risks and notes

- **Double-send.** Both the claim and the defer paths delete the file they consume inside the same lock, so at most one
  sender wins. The chop takes the same lock before flushing.
- **Lost notification.** The only way to lose one is for the store write to succeed and then every claimer to disappear;
  the chop's grace-period sweep is what closes that hole. Prefer a duplicate over a loss anywhere the design is
  ambiguous.
- **Toast/row ordering.** `finish_epic_launch` back-fills `epic_bead_id` before notifying, so the row is already
  `EPIC CREATED` on disk when the toast is appended. ACE's notification handling already re-reads the targeted row
  (`refresh_notification_agent_or_request`), so no TUI change is required.
- **Rust core boundary.** The whole epic-launch and notification-sending flow this touches is Python
  (`sase/notifications/senders.py`, `sase/bead/epic_launch.py`, `sase/axe/run_agent_runner_finalize.py`); the only core
  call needed is the existing `parse_agent_artifact_path` binding. No `sase-core` change.
- **Scope.** Only the epic path changes. `plan_rejected`, `approve`, `tale`, and `commit` completion notifications keep
  their current immediate behavior.
