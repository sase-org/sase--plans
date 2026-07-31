---
tier: tale
title: Complete marked-set notification actions
goal:
  Notification marks consistently target atomic bulk dismiss, mute, unmute, and snooze operations without blocking the
  TUI.
proposed_by: bbugyi200.athena.qj
create_time: 2026-07-31 13:15:31
status: done
---

- **PROMPT:** [202607/prompts/bulk_notification_mark_actions.md](prompts/bulk_notification_mark_actions.md)

# Plan: Complete marked-set notification actions

## Goal

Make the notification modal's `m` marks a consistent target selector for every row-state action that sensibly applies to
a selected set. Bulk dismiss via `x` already honors marks; extend the same behavior to mute/unmute via `M` and snooze
via `s`, while preserving all existing no-mark behavior.

This is a `tale`, not an `epic`: the Rust store operation, Python wrapper, and modal behavior are one tightly coupled
vertical slice. Splitting them across independent phases would create temporary wire/API mismatches and add handoff cost
without useful parallelism.

## Current behavior and constraints

- `NotificationModal` stores stable notification IDs in `_marked_notification_ids`, renders the mark, advances after
  `m`, and clears tab-scoped marks when the user explicitly switches tabs.
- `x` checks for marks and calls the atomic `mark_many_dismissed` store operation. `M` and `s` ignore marks and mutate
  only the highlighted row.
- The prior mark/bulk-dismiss tale explicitly left mark-aware mute and snooze out of scope; this tale closes that gap.
- Notification tabs make mute state homogeneous within a selectable tab: muted rows live only in `Muted`, and all other
  tabs contain unmuted rows. Even so, the bulk mute API should define a safe mixed-state fallback.
- `R` remains “Read All” and already targets the complete inbox without marks. Selection, debug, response, attachment,
  and navigation actions are not bulk row-state mutations and remain single-row operations.
- Notification persistence belongs in `sase-core`. Each Rust state update acquires the store lock and rewrites once, so
  bulk actions must use one multi-ID update rather than loop over the single-ID Python APIs.
- Persistence must not block Textual's event loop. Submit marked-set writes through the app's tracked background-task
  path (with a notification-state dedup/exclusive scope), and apply modal state only on the UI-thread completion
  callback. One marked action must produce one store update and at most one list rebuild.
- Do not manually change `sase-core` workspace/crate versions or path-dependency pins; release-plz owns them. The new
  tagged update variants are additive and do not require a notification wire schema-version bump.

## Product semantics

### Target resolution

Add one deterministic helper that resolves the action target at dispatch time:

1. If live marked IDs exist, return those notifications in modal dataset order and report a marked-set action.
2. If `_marked_notification_ids` contains only stale IDs, clear those stale marks and fall back to the highlighted row
   rather than silently acting on an empty set.
3. If there are no marks, return only the highlighted row, preserving today's behavior.

Snapshot target IDs before opening the snooze picker or submitting persistence. IDs, not list indices, must cross any
modal or worker boundary.

### Mute/unmute (`M`)

- With no marks, keep the current highlighted-row toggle and existing toast wording.
- With marks, set every target to one common state: mute all if any target is unmuted; otherwise unmute all. Under the
  normal tab invariant this simply toggles the active tab's state, while the mixed-state rule is deterministic if
  stale/injected state ever violates that invariant.
- Bulk unmute clears `snooze_until` for every target, matching the single-row store contract.
- Use one plural count-aware toast (`Muted N notifications`, `Unmuted N notifications`, or an unmute message noting that
  snoozes were cancelled when any target had a deadline).

### Snooze (`s`)

- Resolve/snapshot the target before pushing `SnoozeDurationModal`, but do not persist anything until the picker returns
  a duration or datetime.
- A cancellation changes neither rows nor marks and retains the existing cancellation toast.
- Convert a relative duration to one absolute timezone-aware deadline once, then apply that exact deadline to every
  target through one store update. All affected rows become `muted=True` and share the same ISO `snooze_until` value.
- Keep the existing single-row toast; use a plural count-aware toast for a marked set.

### Marks, tabs, and highlighting

- On successful bulk mute/unmute or snooze, remove the acted-on IDs from the mark set. Do not clear unrelated marks that
  could have been added while persistence was in flight.
- Recompute tabs once after mutating all matching in-memory rows. If the current tab still exists, preserve the current
  highlighted notification when it remains visible; otherwise select the nearest unacted-on row using visual order, then
  fall back to the first visible row. If the tab disappears, use the existing nearest-tab coercion and its first visible
  row.
- A completion callback must re-resolve IDs against the current modal dataset and current tab/highlight instead of
  trusting pre-worker indices. If the modal has closed, persistence still succeeds but the callback performs no widget
  work.
- On persistence failure, leave in-memory rows and marks unchanged and show an error toast. Guard/deduplicate
  overlapping notification-state writes so a second action cannot race the first against stale modal state.

## Implementation

### 1. Add atomic multi-ID operations in `sase-core`

Update `crates/sase_core/src/notifications/wire.rs` with additive tagged variants equivalent to:

- `MarkManyMuted { ids: Vec<String>, muted: bool }`
- `MarkManySnoozed { ids: Vec<String>, until: String }`

Handle both variants in `crates/sase_core/src/notifications/store.rs` under the existing single locked read/modify/write
transaction. Normalize IDs to a set, count each matching stored row once, preserve the single-row changed-count
semantics, clear snooze deadlines when unmuting, and set `muted=true` plus the shared deadline when snoozing.

Extend `crates/sase_core/tests/notification_store_parity.rs` to cover multiple matches, missing and duplicate IDs,
already-correct rows, bulk unmute cancelling snoozes, bulk snooze using one deadline, and accurate matched/changed
counts. Add a PyO3 update-deserialization smoke test only if the existing generic binding coverage does not exercise the
new tagged variants; no gateway endpoint change is needed because the TUI reaches these operations through the generic
Python binding.

### 2. Expose Python bulk store APIs

In `src/sase/notifications/store.py`, add `mark_many_muted(ids, muted=True) -> int` and
`mark_many_snoozed(ids, until) -> int`. Mirror `mark_many_dismissed`: materialize the iterable once, skip the Rust call
for an empty input, send one `NotificationStateUpdateWire`, use the count-only facade, and return `matched_count`.
Export both from `src/sase/notifications/__init__.py`.

The existing Python wire record already carries `ids`, `muted`, and `until`; add serialization assertions in
`tests/test_core_facade/test_notification_store.py` rather than introducing parallel wire fields. Extend
`tests/notification_store/test_mute_snooze.py` with round-trip, empty-input/no-call, missing/duplicate-ID,
shared-deadline, and route-through-one-Rust-update coverage for both public APIs. Retain the single-ID APIs for
compatibility.

### 3. Make modal mute and snooze mark-aware

In `src/sase/ace/tui/modals/notification_modal.py`, import the new bulk APIs and provide patchable adapter methods next
to `_mark_many_dismissed`. Keep the persistence boundary behind modal methods so unit tests can assert one call without
touching the real store.

In `src/sase/ace/tui/modals/notification_modal_actions.py`:

- Factor target-ID resolution and bulk reclassification/rebuild logic instead of duplicating it between `M` and `s`.
- Branch `action_toggle_mute` and `action_snooze` on whether resolved targets came from marks; retain their current
  single-row contract when no marks exist.
- Submit persistence off the event loop through one small notification-state tracked task. Return a typed result to a
  UI-thread completion callback, disable automatic global reload/toast behavior, and use a stable dedup/exclusive scope
  so overlapping modal writes are rejected cleanly.
- On successful completion, mutate every still-live target object, reconcile only the acted-on marks, tabs, and current
  highlight, rebuild once, and emit the correct singular/plural toast. On failure, keep modal state intact and report
  the error.
- Ensure the snooze callback closes over target IDs and one computed absolute deadline, never a mutable index or one
  notification object.

If a tiny dataclass is useful for the worker request/result, keep it beside the modal state actions unless it has a
genuine consumer outside this modal. Do not move presentation-only selection/tab logic into Rust.

### 4. Tests and documentation

Expand `tests/test_notification_modal_mute_snooze.py` and, where shared mark/tab helpers fit better,
`tests/test_notification_modal_mark_and_tabs.py` to cover:

- `M` with two marked unmuted rows performs one bulk mute, updates both rows, clears only acted-on marks, rebuilds once,
  and preserves/highlights the nearest visible unmarked row.
- `M` from `Muted` bulk-unmutes, cancels mixed snooze deadlines, and handles the last-row/tab-disappears case.
- The defined mixed-state fallback converges all marked targets to muted.
- `s` opens one picker for the marked set, cancellation is a no-op, and a relative or absolute choice produces one bulk
  call with the same deadline for every ID.
- The async completion path revalidates IDs/current selection, does not disturb unrelated/new marks, no-ops widget work
  after modal close, rejects an overlapping mutation, and leaves state untouched on failure.
- No-mark `M` and `s` retain all existing single-row calls, state changes, navigation, and toast text.
- Stale marked IDs are pruned and do not cause an empty bulk write.

Update `docs/notifications.md` so the keybinding table and “Marks and Bulk Dismiss” section become a general “Marks and
Bulk Actions” contract covering `x`, `M`, and `s`, including mark consumption and the bulk mute rule. Update the
mute/snooze examples with plural behavior. The modal hint strings already expose `m`, `x`, `M`, and `s`, so wording need
not change unless implementation reveals misleading text; recheck the ACE `?` help popup per the nested instructions and
update it only if notification-modal behavior is documented there. No visual golden should change unless an intentional
visible label/hint change is introduced.

## Validation

1. Run focused Rust formatting/tests for the notification store while iterating, then run `just rust-check` from the
   `sase` checkout so format, clippy, and the full linked-core workspace tests pass.
2. Run `just install` to rebuild/install the linked `sase-core` extension before Python verification.
3. Run the focused Python store/facade/modal tests named above, including the real-extension round trips.
4. Run `just check` from the `sase` checkout as the final required validation. If an intentional visible TUI change was
   made, also run `just test-visual` and inspect/update snapshots only when the diff matches the plan.
5. Recheck both repository worktrees and report the `sase` and `sase-core` changes/tests separately; do not modify
   release-plz-owned version fields.

## Expected files

`sase-core`:

- `crates/sase_core/src/notifications/wire.rs`
- `crates/sase_core/src/notifications/store.rs`
- `crates/sase_core/tests/notification_store_parity.rs`
- `crates/sase_core_py/src/lib.rs` only if a focused binding smoke test is needed

`sase`:

- `src/sase/notifications/store.py`
- `src/sase/notifications/__init__.py`
- `src/sase/ace/tui/modals/notification_modal.py`
- `src/sase/ace/tui/modals/notification_modal_actions.py`
- `tests/notification_store/test_mute_snooze.py`
- `tests/test_core_facade/test_notification_store.py`
- `tests/test_notification_modal_mute_snooze.py`
- `tests/test_notification_modal_mark_and_tabs.py` if shared selection/tab cases belong there
- `docs/notifications.md`

## Non-goals

- Changing `R` from “Read All” into “read marked.”
- Bulk-opening/answering/debugging notifications or applying attachment actions to multiple rows.
- Persisting marks across tabs or modal sessions, or adding a separate clear-marks binding.
- Adding mobile/gateway endpoints for bulk state updates.
- Manually bumping Rust crate/package versions or the published `sase-core-rs` dependency window.
