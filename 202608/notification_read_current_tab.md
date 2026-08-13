---
tier: tale
title: Scope notification-panel R to the current tab
goal:
  Pressing R marks only the active notification tab read, including rows beyond the
  modal page, without blocking the TUI.
size: medium
proposed_by: bbugyi200.athena.001
create_time: 2026-08-13 17:49:18
status: wip
---

# Scope notification-panel `R` to the current tab

## Goal

Change the notifications panel so pressing `R` marks every unread notification owned by
the tab that was active when the key was pressed, while notifications in every other tab
remain unread. This must apply to the whole canonical tab, not only the modal's bounded
100-row page, and the write must not block Textual's event loop.

## Current behavior and root cause

- `NotificationModal.BINDINGS` maps `R` to `read_all` and advertises "Read All".
- `NotificationBasicActionsMixin.action_read_all()` calls the modal's `_mark_all_read()`
  adapter, which delegates to the store-wide `mark_all_read()` mutation, then marks
  every row in the modal's in-memory dataset as read.
- The modal already has canonical, single-owner tab membership from the Rust-backed
  classifier, represented locally by `_active_notification_tag` and
  `_notification_tab_keys`. The General tab is represented as `None` in the modal but as
  `"general"` in the core.
- The unread provider caps the modal page at 100 rows. Persisting only the IDs currently
  loaded in the modal would therefore leave overflow rows in the same tab unread and
  make them reappear the next time the panel opens.
- The current `R` handler performs the store rewrite synchronously on the UI thread. The
  replacement should use the modal's existing tracked notification-state task
  infrastructure so an unbounded store scan/rewrite cannot stall key handling.

## Required behavior

1. Capture the active tab at dispatch time. A user may switch tabs while persistence is
   running; completion must still apply to the originally requested tab.
2. Under one notification-store lock, mark as read every non-dismissed, non-silent,
   currently unread row whose canonical `tab_key_for()` result equals the captured core
   tab key. This includes matching rows beyond the modal page limit. Already-read rows,
   hidden dismissed/silent rows, and rows owned by other tabs are unchanged.
3. On successful completion, update only the matching rows present in the modal's
   in-memory dataset, preserve whichever tab and row are current at completion time,
   rebuild the display, and request the existing authoritative notification refresh so
   indicator chips and agent unread projections converge with the store.
4. On persistence failure, leave modal-local read flags unchanged and show an error.
   Duplicate notification-state writes remain serialized by the existing tracked-task
   deduplication/exclusive scope.
5. Rename the binding/action/help copy from "read all" to "read tab". Keep the public
   store-wide `mark_all_read()` API for its existing non-modal callers and benchmarks.
   No configurable keymap value changes, so `default_config.yml` does not need an edit.

## Implementation

### 1. Add an atomic tab-scoped read mutation to `sase-core`

- Open the linked `sase-core` repository through the required repository workflow.
- Extend `NotificationStateUpdateWire` in `crates/sase_core/src/notifications/wire.rs`
  with a tagged `mark_tab_read` variant carrying the canonical `tab_key`.
- Handle that variant in `crates/sase_core/src/notifications/store.rs` while the store's
  existing exclusive lock is held. Reuse `notifications::tabs::tab_key_for()` as the
  sole membership rule; do not duplicate panel/tag/gate/error/mute/snooze precedence in
  Python or in the store mutation.
- Preserve the existing outcome contract: matched/changed counts describe eligible
  unread rows actually targeted, count-only calls omit returned rows, and the file is
  rewritten only when a row changes.
- Add Rust store and PyO3 binding coverage proving serialization/deserialization,
  count-only behavior, idempotence, exclusion of dismissed/silent/read and other-tab
  rows, and multiple matching rows. Include enough matching rows in a store-level case
  to demonstrate that the operation has no UI page-size boundary.

### 2. Expose the mutation through the Python notification facade

- Add `tab_key` to `NotificationStateUpdateWire` and its JSON projection in
  `src/sase/core/notification_store_wire.py`.
- Add a public `mark_tab_read(tab_key: str) -> int` wrapper in
  `src/sase/notifications/store.py` that sends one count-only `mark_tab_read` update and
  returns the changed count. Export it from `src/sase/notifications/__init__.py`.
- Add facade/wire and notification-store tests that assert the exact tagged payload,
  cache invalidation through the normal update path, idempotent counts, complete
  same-tab coverage, and preservation of unrelated tabs. Use the real local extension
  integration test where the existing notification-store suite already exercises new
  state-update variants.

### 3. Route `R` through a tracked, tab-stable modal action

- Add or expose one small conversion helper in
  `src/sase/ace/tui/modals/notification_modal_tags.py` that maps the modal tag
  vocabulary back to the core key (`None` to `"general"`; every other key unchanged).
- Replace the modal's `_mark_all_read()` adapter/import with a `_mark_tab_read()`
  adapter, and change the binding to `("R", "read_tab", "Read Tab")`.
- Implement `action_read_tab()` in
  `src/sase/ace/tui/modals/notification_modal_basic_actions.py`. At dispatch, coerce and
  capture the active tab/core key plus the IDs of loaded rows owned by that tab, then
  submit the store write through `_submit_notification_state_task()` using the existing
  `notification-state` dedup key and exclusive scope. Extend the typed notification
  mutation result only as needed to represent a read operation and its captured target.
- In the UI-thread completion callback, first request an authoritative refresh. If the
  write failed, report it without changing local rows. If the modal is still active,
  re-read the current tab/highlight, mark only the captured matching in-memory rows as
  read, and rebuild around the current selection (or the first visible row if that
  selection disappeared). Do not force the UI back to the originally targeted tab.
- Update `DEFAULT_HINT_TEXT` from `R: read all` to `R: read tab`. Gate/question-specific
  hints currently omit `R`, so leave them unchanged unless the implementation discovers
  an existing consistency assertion that requires the same copy there.

### 4. Lock the behavior down with modal tests

- Update the binding/footer assertions to require `read_tab` / `Read Tab` /
  `R: read tab` and reject the old `read_all` wording.
- Add focused action tests with notifications in at least two canonical tabs proving
  that dispatch sends the captured core tab key and does not mutate the other tab.
- Exercise the General-tab `None` to `"general"` conversion and a synthetic/panel or tag
  tab so the conversion is not accidentally special-cased to one tab kind.
- Drive the tracked-task callback separately to prove persistence is off the UI thread,
  an error leaves all local flags unchanged, success updates only the captured tab,
  refresh is scheduled, and switching/highlighting another tab before completion does
  not switch the user back or mark that tab read.
- Keep a regression assertion that the store-wide `mark_all_read()` behavior still
  exists independently of the modal action.

## Verification

1. In the linked `sase-core` repository, run its required whole-workspace `just check`
   (not a `cargo test -p sase_core` shortcut) so core, PyO3, formatting, clippy, and
   binding tests all run.
2. In `sase`, run `just install` after the core changes so the workspace virtualenv uses
   the updated local `sase_core_rs` extension.
3. Run focused Python tests for notification-store state updates/core facade and the
   notification modal binding, tab, and read-action behavior.
4. Run `just test-visual` because the default notification footer wording appears in PNG
   snapshots. Inspect `.pytest_cache/sase-visual/` artifacts, accept only the
   intentional `R: read tab` footer diffs with `--sase-update-visual-snapshots`, and
   rerun the visual suite to exact equality.
5. Run `just check`. If its scoped lane escalates, reports an unusual selection, or the
   changed core/wire surfaces are in the broadening set, run `just check-full` through
   `/sase_monitor` with a concrete `--next` action, as required by the project workflow.

## Acceptance criteria

- Pressing `R` on any notification tab marks all eligible unread rows in that canonical
  tab read, including rows beyond the modal's 100-row page, and leaves every other tab's
  rows unread.
- The tab targeted is the tab active at keypress time even if the user navigates before
  persistence completes; completion preserves the user's then-current tab and cursor.
- The mutation is one atomic core store update and does not perform disk I/O on the
  Textual event loop.
- General, gate/panel/tag, Errors, Muted, and Snoozed membership continues to use the
  existing canonical precedence rather than a new UI-side classifier.
- The binding/help text says "Read Tab", store-wide `mark_all_read()` remains available,
  and all Rust, Python, scoped/full (as required), and visual verification passes.
