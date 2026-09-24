---
tier: tale
title: Same-tick unread marker for completed agents
goal:
  A finished agent's Agents-tab node shows its terminal status and unread marker on the
  same notification-poll tick that shows the completion toast, without waiting for the
  serialized Agents load lane.
size: medium
proposed_by: bbugyi200.athena.0r3
create_time: 2026-09-24 14:35:09
status: wip
---

# Show the Agents-tab unread marker on the same tick as the completion toast

## Symptom

An agent finishes. ACE shows the completion toast right away, but the Agents-tab node
for that agent keeps its running status, with no unread marker, for many seconds
afterward. The contract is that the unread marker (and the terminal status it depends
on) appears on the same notification-poll tick as the toast.

## Evidence (athena, 2026-09-24, from `~/.sase/logs`)

`tui_toasts.jsonl` gives toast times. `tui_agent_loads.jsonl` logs every Agents load
that took 2 s or more (`tui_agent_load_slow`). Lining the two up shows the exact
artifact-delta load that the toast tick requests finishing well after the toast:

| toast                  | loads in the Agents lane (seconds relative to the toast)                                                                            | earliest dot |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| 18:00:24 `@0qz`        | `notification` delta +0.9 → +8.0 (disk 2.3, prep 4.6)                                                                               | ~+8 s        |
| 18:00:35 `@sase-17y.4` | `inflight_poll` delta −1.2 → +8.6 still running, then `notification` delta +8.9 → +14.8                                             | ~+9–15 s     |
| 18:02:49 `@sase-17z.2` | `inflight_poll` delta −4.2 → +10.9 (disk 9.3, prep 5.5) still running, then `notification` delta +11.1 → +20.7 (disk 8.1, prep 1.2) | ~+11–21 s    |

Today's slow loads alone kept the single Agents load lane busy 23–31% of every hour.
Among the slow ones, `notification` exact deltas had disk time p50 3.3 s and p90 9.1 s,
and `inflight_poll` deltas p50 2.9 s, max 69.6 s. A one-directory delta takes about 0.3
s in an idle process. In the busy TUI process, it also pays for thread and GIL
contention and O(roster) prep.

## Root cause

The unread marker is a join of two facts, and the toast tick has only one of them.

1. `_poll_agent_completions_once` (`_notification_polling.py`) toasts the arrival and
   calls `_reconcile_unread_from_completion_notifications`. That projection
   (`_notification_unread_projection.py`) marks a node unread only when the **loaded**
   node's status passes `is_unread_completed_status`. This gate is correct and tested
   (`test_reconcile_skips_running_agent_rows`), and it stays.
2. The terminal status can only enter the roster through an Agents load. The toast tick
   asks for one through `request_notification_agents_refresh` →
   `_schedule_agent_artifact_delta_refresh`. That load runs on the one serialized
   `_agents_loading` lane: it waits behind whatever load is in flight, and
   `_drain_pending_agents_refresh_work` starts a pending broad refresh before a pending
   delta. Its own disk and prep stages then take seconds under production load.

So the marker lands one or two slow loads after the toast. The runner already wrote
`done.json` before it sent the notification (`finalize_runner_shutdown` runs after
`_run_agent`'s exec finalize). That means the terminal state is on disk at toast time.
ACE just doesn't read it on that tick.

### Why the gate-notification fix does not close this

The pending-gate fix (`511a6c5ae`, plan `202609/pending_gate_status_refresh.md`) closed
a different gap: the gate arrival requested **no** exact delta at all. Completions
already request a same-tick exact delta (`is_active_agent_refresh_notification` →
`completion_notification_delta_dirs`), so copying that fix changes nothing here. The
remaining latency is the load lane itself. This tale reuses that fix's shape instead:
disk work stays on the notification-poll worker hop that
`_prepare_notification_reconciliation` already runs.

### Secondary drop

In `_run_auto_refresh_surfaces` (`event_refresh/_auto_refresh_surfaces.py`), the tick's
own `_poll_agent_completions()` can be the poll that sees an arrival. That happens when
it wins the race with the watcher-scheduled poll, or when the watcher is inactive. The
tick then returns early for transient input modes or `self._agents_loading` before it
reaches `request_notification_agents_refresh`. The activity cursor is already marked
delivered, so that arrival's exact delta is never requested. The row then waits for the
in-flight-marker or watcher backstops, or for the 300 s sanity pass.

## Fix

Resolve the notified node's terminal status authoritatively, and cheaply, on the poll's
existing worker hop. Apply it to the loaded rows on the same tick as a short-lived
**arrival status overlay**, then let the existing unread projection and selective row
patch show the marker. The exact delta still runs and remains the authoritative apply.
This follows `tui_perf` rules 1, 4, 5, and 6: disk work off the loop, optimistic
in-memory state plus background reload, and selective patches. It adds no new refresh
spine, no broad load, and no change to `AGENTS_LOAD_MIN_INTERVAL_SECONDS` or
`FULL_SANITY_REFRESH_SECONDS`.

### 1. Worker-side exact status probe

Add `src/sase/ace/tui/actions/agents/_notification_completion_arrival.py` with a pure,
worker-thread function, for example
`prepare_completion_arrival_overlays(app, notifications) -> CompletionArrivalPrep` (a
frozen dataclass holding `overlays: tuple[(identity, status), ...]` and
`artifact_dirs: tuple[Path, ...]`). Call it from `_prepare_notification_reconciliation`
alongside `prepare_pending_gate_notification_refresh`, passing the full current
snapshot's notifications (`notifications`), and return its result in that tuple.

Candidate selection is driven by mismatches, so it also covers muted arrivals and heals
itself on later polls:

- Take exact keys `(cl_name, raw_suffix)` from notifications where
  `is_active_agent_completion_notification` holds and `raw_suffix` is non-empty. Skip
  `cl_name`-only rows. The runner always stamps `raw_suffix`.
- Resolve each key against `loaded_real_agent_roster(app)` (local rows only; skip rows
  with no local artifact dir). Find the owning node with
  `agent_node_projection_index(roster).owner_for_identity(row.identity)`. A standalone
  row owns itself.
- Keep only candidates whose owning node is **not** `is_unread_completed_status`. That
  is exactly the gate the unread projection is waiting on. When nothing mismatches,
  return an empty result without touching disk (the common case).
- For each candidate, collect the artifact dirs of the node and all of its projection's
  `owned_rows`, excluding workflow-step children, via `agent_artifact_dir`. Include
  every loaded family member so that `apply_status_overrides` in normalization mirrors
  the family root the same way a full load would.
- Make one batched call to
  `load_artifact_delta_agents(dirs, update_index=False, patch_snapshot=[])` from
  `models/agent_loader.py`. The call is read-only: no index writes. Pass
  `patch_snapshot=[]`, not `None`: `None` triggers an uncached `find_all_patches`, and
  status does not depend on patch lookups (precedent:
  `_load_plan_agents_from_artifact_snapshot`).
- Emit an overlay `(identity, scanned.status)` for the candidate's node and its notified
  row, but only where the scanned row exists and
  `is_unread_completed_status(scanned.status)`. Take the label (`DONE`, `FAILED`,
  `TALE DONE`, …) from the loader. Never infer it from the notification's `action`.
- Wrap the probe in
  `tui_trace("notification.completion_arrival", candidates=…, overlays=…, dirs=…)` so
  the fix can be measured with `SASE_TUI_TRACE=1`.

### 2. Same-tick install in the poll

In `_poll_agent_completions_once`, after the worker hop returns and the toasts are
emitted, and before `_reconcile_unread_from_completion_notifications`, install the
overlays in a small mixin method (it can live in the new module):

- Re-capture the live roster, because the await may have interleaved with an apply (rule
  4). For each overlay, find the live row by identity in `loaded_real_agent_roster`.
  Skip it if it is missing or already `is_unread_completed_status`.
- Set `row.status = status` on the live object. This is the same live-row mutation that
  `_apply_remote_optimistic_state` uses. Then record it in a new app dict
  `_agents_arrival_status_overlays: dict[identity, _ArrivalOverlay]`, where
  `_ArrivalOverlay` is `(status, survives_stale_apply=bool(self._agents_loading))`.
  Initialize the dict in `_state_init_agents.py` and declare it in `_loading_state.py`.
- Schedule the paired authoritative reload:
  `self._schedule_agent_artifact_delta_refresh(list(prep.artifact_dirs), source="notification")`.
  It coalesces with the caller's own request through
  `_merge_agent_artifact_delta_request`. This matters because the overlaid row no longer
  looks in-flight, which drops it from the in-flight-marker backstop.
- Let the existing `_reconcile_unread_from_completion_notifications(notifications)` mark
  the node unread, now that the terminal gate passes. Then repaint through the existing
  selective path. Extend
  `_patch_unread_completed_agent_changes(before, *, status_changed=())` so that
  status-changed identities are patched the same way as unread-changed ones, including
  their node and clan ancestors. Keep its current
  `_refresh_agents_display(list_changed=True, defer_detail=True)` fallback for
  membership changes. Do not call a disk load. Off the Agents tab it stays state-only,
  as today.

### 3. Overlay lifetime: survive only the stale in-flight load

A family node's terminal status is not monotonic: a later `%id` or fork can attach a new
running member. The overlay must therefore never outlive the first authoritative load,
and must not use `_agent_status_overrides`. Those overrides are cleared by "loaded
status is terminal", which would pin a wrongly-DONE family, and a refilter clears them
against the already-mutated row. They also belong to user-intent overrides (`RUNNING`,
`ANSWERED`, remote labels) whose semantics must not change.

In the UI-thread load-apply seam (`_apply_loaded_agents_prepared_inner` in
`_loading_apply.py`, the single seam that full, bounded/viewport, and artifact-delta
loads share), install a new roster, then call `reconcile_arrival_status_overlays(self)`
before `_finalize_agent_list` runs `_sync_unread_completed_agents`. For each overlay:

- If `survives_stale_apply` is set, this apply belongs to the load that was already in
  flight when the overlay was installed, so its read may predate `done.json`. Set
  `row.status = overlay.status` on the newly installed row, but only if that row is
  present and not `is_unread_completed_status`. Then clear `survives_stale_apply`. This
  prevents the marker from flickering off and back on.
- Otherwise the load started after install and is authoritative. Drop the overlay and
  let the loaded row speak, whatever its status.

The lane is serialized, so at most one stale apply exists per overlay, and every overlay
is gone after about two loads. Refilters (`_refilter_agents`, fleet reprojection) reuse
the mutated live objects and need no hook. Also drop overlays for identities that are
dismissed or killed, next to the existing `_agent_status_overrides` pops.

### 4. Never drop an observed arrival's delta

In `_run_auto_refresh_surfaces`, when the tick's own poll returned
`new_agent_notification`, call `request_notification_agents_refresh(...)`, with the same
arguments it uses today, before returning on the transient-input-mode and
`_agents_loading` early exits. The request only queues behind the in-flight load. The
watcher-scheduled poll path (`_run_scheduled_notification_poll`) already requests in
those modes, so this restores parity. Keep today's suppression when the tick itself runs
a broad load.

### 5. Docs

Extend the "A newly arrived completion notification also drives a targeted Agents-tab
refresh" paragraph in `docs/notifications.md` with two points. First, the finished node
shows its terminal status and unread marker on the toast tick, from an exact read-only
status probe. Second, the targeted reload that follows is authoritative.

## Constraints and non-goals

- Keep the unread projection's terminal-status gate and its matching rules unchanged.
- Keep all disk access off the event loop: roster reads plus `agent_artifact_dir` and
  the scan happen on the existing worker hop only. Add no new pump or timer callbacks.
- No broad load, no new refresh loop, and no change to the load lane's scheduling order.
  Making the lane itself faster is separate work, recorded as a discovered issue on epic
  `sase-124` (Agents tab freshness).
- Pending-gate arrivals (TALE/EPIC/QUESTION) insert a **new** member row, so an overlay
  cannot help them. They stay on their exact-delta path.
- Settlement senders (`epic-launch`, `monitor-settlement`) are out of scope. They can
  continue a family, so they keep today's delta path.

## Tests

Build the artifact dirs with real on-disk fixtures, reusing the helpers in
`tests/ace/tui/test_agent_family_status_convergence_repro.py` and
`tests/test_notification_pending_gate_refresh.py`. Drive the real
`_poll_agent_completions_once` with `tests/_notification_toasts_helpers._FakeApp` where
possible.

- New `tests/test_notification_completion_arrival.py`:
  - A standalone RUNNING row plus a new `JumpToAgent` completion plus `done.json` on
    disk. After one poll, the row is `DONE` and in `_unread_completed_agent_ids`, and an
    exact `notification` delta for its dir is scheduled but not yet executed. This
    proves the marker does not wait for the delta.
  - `ViewErrorReport` with a failed done marker produces `FAILED` and unread.
  - Family: the coder member finishes. The family root takes the loader's mirrored label
    (for example `TALE DONE`) and becomes unread in the same poll.
  - Family with another loaded member still in flight: no node overlay and no unread.
  - The scan still shows in-flight (no `done.json`): no overlay and no unread. A later
    poll with the marker present then overlays.
  - A `cl_name`-only notification, or an already-terminal node: no disk scan (assert
    through a recording monkeypatch on `load_artifact_delta_agents`).
  - The scan and `agent_artifact_dir` run on the worker thread, not the event-loop
    thread (same technique as `_record_scan_threads`).
- Overlay lifetime, in `tests/ace/tui/` next to the unread tests:
  - Installed while `_agents_loading` is set, then a stale load applies fresh RUNNING
    objects for that identity. The row stays terminal and unread. The next load applies
    a RUNNING family (a new member attached) and the overlay is dropped: the row shows
    RUNNING and is not unread.
  - Installed with no load in flight: the next load drops it and loaded status wins.
  - Existing `_agent_status_overrides` behavior is unchanged
    (`tests/ace/tui/test_agent_notification_status_overrides.py` stays green).
- `_patch_unread_completed_agent_changes(status_changed=…)` patches a status-changed
  member row that did not change unread state, and falls back to the display rebuild on
  a membership change.
- Auto-refresh drop, in
  `tests/ace/tui/test_event_handlers_auto_refresh_notifications.py`: when the tick's own
  poll observes a completion while `_agents_loading` is set, or while a hint mode is
  active, `request_notification_agents_refresh` is still called. With no arrival, a
  quiet tick still reloads no surfaces.

## Verification

- Read `sase/memory/lint_and_test.md` and run the verification it prescribes. Fix any
  new symvision or private-use findings, for example by giving test-imported helpers
  public names.
- Optional live check: with `SASE_TUI_TRACE=1`, finish a trivial agent. The
  `notification.completion_arrival` span should report one overlay on the toast tick,
  and the row should flip to done and unread without waiting for the
  `notification`-sourced delta.
