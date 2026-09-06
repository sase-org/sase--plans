---
tier: tale
title: Fix `,X` by re-keying launch records to durable proc ids
goal: '`,X` kills and edits the last accepted launch in real ACE sessions because
  launch records are re-keyed from the submit-time placeholder proc id to the durable
  proc id, so completion stamping and deferred kills find their record.'
size: medium
proposed_by: bbugyi200.athena.0h3
status: done
---

# Fix `,X` by re-keying launch records from placeholder to durable proc ids

## Problem and root cause

The `,X` (kill-and-edit-last-launch) keymap still does not work in real ACE sessions,
even after the `202609/kill_and_edit_lifecycle.md` repairs landed. The keymap dispatch,
target resolution, and deferred-kill state machine are all correct and fully covered by
passing tests (92 focused tests pass on the current tree). The defect is one layer
below, in how launch records are keyed.

**Root cause (confirmed from code and from live runtime logs):**

1. `_submit_durable_proc` (`src/sase/ace/tui/actions/_proc_action_submission.py`)
   returns the observer's short-lived _placeholder_ row from
   `ProcObserver.register_pending`, whose `proc_id` is `pending-<uuid4hex>`
   (`src/sase/ace/tui/proc_observer.py`). The real durable proc id (for example
   `psmymmrmsqxd`) only exists after the threaded submit worker returns a handle.
2. `_submit_resolved_launch` (`.../agent_workflow/_launch_submission.py`) and the bulk
   path (`.../agent_workflow/_launch_bulk.py`) call `push_launch_record` with that
   placeholder id (`proc_info.proc_id` / `slot.proc.proc_id`), so every
   `LaunchRecord.proc_ids` and `submitted_prompts` key is a `pending-*` id.
   `_submit_launch_proc` (`.../agent_workflow/_launch_procs.py`) also keys
   `_launch_submitted_prompts` (the payloadless-failure recovery prompt map) by the
   placeholder id.
3. When the submit worker returns, `_on_durable_submit_worker_completed`
   (`src/sase/ace/tui/actions/_proc_action_completion.py`) re-keys
   `_proc_completion_callbacks` from the placeholder id to the durable id — but nothing
   re-keys the launch record.
4. Proc completion is then delivered with the durable id: `_on_launch_proc_complete`
   calls `stamp_launch_record_results(self, proc_id, ...)` and
   `apply_deferred_launch_kill_on_completion(self, proc_id, ...)`, both of which look
   the record up via `launch_record_for_proc_id` — which matches
   `proc_id in record.proc_ids`. The durable id never matches the stored placeholder id,
   so both calls silently no-op (`launch_record_for_proc_id` returns `None`).
5. The record therefore never leaves `IN_FLIGHT`. Every `,X` press routes to
   `_begin_inflight_deferred_kill`, toasts
   `Will kill "<name>" when its launch finishes`, registers a `KILL_PENDING` intent that
   no completion callback can ever match, and the 180-second
   `PENDING_LAUNCH_KILL_TIMEOUT_SECONDS` timer abandons it with
   `"<name>" took too long to finish launching`.

**Runtime evidence** (host toast log `~/.sase/logs/tui_toasts.jsonl` and durable proc
store `~/.sase/procs/procs.jsonl`, 2026-09-06 UTC): launch proc `psmymmrmsqxd` was
created 20:49:06 and finished successfully 20:49:13 (its message `Started 1 agent(s)`
was toasted at 20:49:14, so the completion callback ran). The user pressed `,X` at
20:49:52 — 38 seconds after completion — and still got the in-flight branch
(`Will kill "bob-cli" when its launch finishes`); the timeout warning fired at 20:52:53,
exactly 180 seconds later, and the user had to kill and relaunch by hand. This proves
the record was never stamped despite a successful, delivered completion.

The existing test suites pass because they construct records and deliver completions
with the _same_ proc id; no test drives the placeholder-to-durable handle transition.
The placeholder mechanism (2026-08-15, `feat(ace): observe durable procs read-only`)
predates the `,X` feature (2026-09-04), so `,X` has never worked against a real durable
launch.

## Scope

One bounded fix implemented directly by one agent. This is TUI orchestration state
(session-scoped launch bookkeeping), not shared backend behavior, so it stays in this
repo's Python TUI layer; no `sase-core` change is needed. Keymap meanings, confirmation
flows, timeout values, and default bindings are unchanged. Do not redesign the launch
record state machine; only fix the identity plumbing and its direct consequences.

## Required behavior

- A launch record created at submit time becomes addressable by the durable proc id as
  soon as the durable handle is known, so completion stamping, deferred-kill
  application, and prompt-recovery lookups all find their record.
- A `,X` press in the window before the durable handle arrives (record still keyed by
  the placeholder) must still work: the pending-kill intent, its timeout timer, and the
  later completion must all converge on the same record after the re-key.
- A submit that fails before producing a handle keeps today's behavior: the failure
  completion is delivered under the placeholder id and marks the record `FAILED`.
- Bulk launch records (multiple proc ids on one record) re-key each slot independently.
- No new UI-thread blocking work; the re-key is a dict/tuple mutation on the UI thread
  from an existing UI-thread callback site.

## Implementation

### 1. Add a handle-arrival hook to the generic proc submission layer

- `src/sase/ace/tui/actions/_proc_action_types.py`: add an optional
  `on_handle: Callable[[str, str], None] | None = None` field to `ProcCallbackConfig`
  (arguments: placeholder id, durable proc id).
- `src/sase/ace/tui/actions/_proc_action_submission.py`: accept `on_handle` in
  `_submit_durable_proc` and store it on the `ProcCallbackConfig`.
- `src/sase/ace/tui/actions/_proc_action_completion.py`: in
  `_on_durable_submit_worker_completed`, after re-keying `_proc_completion_callbacks`
  and calling `observer.register_submitted`, invoke
  `config.on_handle(placeholder_id, result.handle.proc_id)` when set, guarded so an
  exception in the hook cannot break completion routing (log it, as sibling paths do).
  The hook must not fire on the no-handle failure path.
- `src/sase/ace/tui/actions/agent_durable.py`: pass `on_handle` through
  `submit_agent_launch` to `_submit_durable_proc`. Other submitters are untouched.

### 2. Re-key launch bookkeeping when the handle arrives

- `src/sase/ace/tui/actions/agent_workflow/_launch_records.py`: add
  `rename_launch_record_proc_id(app, old_proc_id, new_proc_id)` that finds the owning
  record via `launch_record_for_proc_id` and rewrites, preserving order:
  - the entry in `record.proc_ids`,
  - the key in `record.submitted_prompts`,
  - defensively, keys in `record.results` and `record.failed_proc_ids` (they should not
    exist before the handle, but a rename must never strand them),
  - the matching `app._pending_launch_kill_timers` entry, whose key is the record's
    `proc_ids` tuple (a `,X` pressed before the handle arrives creates it under the old
    tuple). It must be a no-op when no record owns `old_proc_id` and must not touch
    `handled_result_keys` / `kill_*_result_keys` (those are result-keyed, not
    proc-keyed). Export it via `__all__`.
- `src/sase/ace/tui/actions/agent_workflow/_launch_procs.py`: in `_submit_launch_proc`,
  pass an `on_handle` closure to `submit_agent_launch` that (a) re-keys
  `self._launch_submitted_prompts` from the placeholder id to the durable id, and (b)
  calls `rename_launch_record_proc_id`. This single wiring point covers both the
  single-launch path (`_launch_submission.py`) and the bulk path (`_launch_bulk.py`),
  since both submit through `_submit_launch_proc`.

### 3. Make the pending-kill timeout rename-safe

`src/sase/ace/tui/actions/agent_workflow/_kill_last_launch_deferred.py` currently
captures `record.proc_ids` in the `set_timer` closure and the timeout handler bails when
`record.proc_ids != proc_ids`. After a rename that guard orphans the timer and can leave
the record `KILL_PENDING` forever if the completion also never arrives, which would
silently park future launches through `has_pending_launch_kill`. Rework
`register_pending_launch_kill` / `_pending_launch_kill_timed_out` /
`_stop_pending_kill_timer` so the timer is associated with the record object itself
rather than a snapshot of its proc-id tuple (for example, key the timers map by a stable
token stored on the timer registration, or resolve the record by iterating the session
stack for the one holding the registration). The timeout must still: only act on a
record that is still `KILL_PENDING`, stop cleanly when `_finish_pending_launch_kill`
runs first, and release relaunch holds exactly as today.

### 4. Regression tests

Extend the existing focused suites (splitting files if size limits require, matching
current test style with fake apps):

- `tests/ace/tui/test_launch_records.py` (or a sibling): `rename_launch_record_proc_id`
  rewrites `proc_ids`, `submitted_prompts`, defensive `results` / `failed_proc_ids`
  keys, and the pending-kill timer key; no-ops without an owning record; preserves
  multi-proc order when renaming one slot of a bulk record.
- A completion-layer test (`tests/` sibling of the proc action tests): submitting with
  `on_handle` set fires the hook exactly once with (placeholder id, durable id) on the
  handle path and never on the submit-failure path, and callbacks still deliver.
- `tests/ace/tui/test_kill_and_edit_last_launch_dispatch.py` /
  `test_kill_and_edit_inflight.py`: an end-to-end shaped regression that mirrors the
  observed failure — push a record under a placeholder id, deliver the handle re-key,
  then stamp results under the durable id and assert the record resolves and `,X`
  targets it immediately (no deferred-kill toast); and the race variant — `,X` pressed
  while the record is still placeholder-keyed, then handle re-key, then completion under
  the durable id, asserting the deferred kill executes and the timeout timer is stopped
  rather than orphaned.
- A bulk variant asserting each slot re-keys independently and
  `_mount_inflight_launch_prompt` still finds per-proc submitted prompts afterward.

No real agents may be killed by tests; keep the established fake-kill/cleanup stubs.

## Verification

- Read `sase/memory/lint_and_test.md` and `sase/memory/tui_perf.md` through the
  `/sase_memory_read` skill in the implementing turn before finishing.
- Run `just install` if this clone's virtualenv is stale, then the focused suites above
  and `just check`. If scoped selection escalates or broadens unusually, run
  `just check-full` only through the `/sase_monitor` skill with the TESTING/TESTED
  status pair.

## Acceptance

All of these together, on top of the existing 92 passing focused tests:

- A resolved real launch (record pushed under a placeholder id, completion delivered
  under the durable id) is immediately targetable by `,X` with no deferred-kill detour.
- A `,X` pressed mid-flight kills the launch's concrete results from the completion
  callback instead of timing out after 180 seconds.
- A submit failure still marks the record `FAILED` under the placeholder id.
- The payloadless-failure recovery prompt map is keyed by the id the completion handler
  actually pops, so `_pop_launch_submitted_prompt` finds its entry.
- No launch record can remain `KILL_PENDING` with an orphaned timeout timer after a
  proc-id rename.
