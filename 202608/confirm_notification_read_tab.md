---
tier: tale
title: Confirm notification-tab read actions
goal:
  Require an explicit, cancel-default danger confirmation before R marks the captured
  notification tab read, without weakening the existing whole-tab or asynchronous
  behavior.
size: small
proposed_by: bbugyi200.athena.001.f1
create_time: 2026-08-13 19:38:42
status: wip
---

# Confirm notification-tab read actions

## Goal

Change the notification panel's `R` action so it first asks the user to confirm marking
the current tab read. The confirmation must identify the captured tab, make clear that
the operation affects the entire canonical tab (including rows outside the modal's
bounded page), and default to cancellation. Confirming must preserve the existing
off-thread, tab-scoped mutation; canceling must have no side effects.

## Current behavior and risk

- `NotificationModal.BINDINGS` maps `R` to `action_read_tab()`, and the footer correctly
  advertises `R: read tab`.
- `action_read_tab()` immediately captures `_active_notification_tag`, converts it to
  the core tab key, captures the loaded row IDs owned by that tab, and submits
  `_mark_tab_read()` through the existing tracked notification-state task path.
- The core mutation deliberately covers every eligible unread row in the canonical tab,
  not just the at-most-100 rows loaded in the modal. On a broad tab such as Gates,
  Errors, or a busy project/panel tab, one accidental keypress can therefore clear far
  more notifications than are visible.
- The repository already has a shared `ConfirmActionModal`/`ConfirmKind.DANGER` surface.
  Danger confirmations focus Cancel by default and support `y`, `n`, Escape, and the
  corresponding buttons, so this action should reuse that behavior rather than add
  notification-specific confirmation state or keybindings.

## Required behavior

1. Pressing `R` on a nonempty notification tab opens one danger-styled confirmation and
   performs no store mutation, local read-state change, or tracked-task submission yet.
2. The prompt identifies the tab by its user-facing label (for example, `General`,
   `Gates`, `Beads`, or a formatted tag label), says that every unread notification in
   the tab will be marked read, and warns that rows not currently loaded are included
   and the action cannot be undone from ACE. Do not show the modal's row count as a
   total, because the modal page is bounded while the core mutation is not.
3. The confirmation is `ConfirmKind.DANGER`, uses an action-specific confirm label such
   as `Mark read`, uses `Cancel` for the negative action, and explicitly defaults to
   Cancel.
4. Only an explicit `True` confirmation dispatches the read. `False`, `None`, Escape,
   `n`, `q`, and the Cancel button leave both persistent and in-memory state unchanged.
5. The target is frozen when `R` is pressed: the callback must retain the captured core
   tab key and captured loaded IDs rather than re-reading `_active_notification_tag` or
   reclassifying targets after the confirmation closes. The subsequent background
   completion must continue to preserve any tab/highlight changes made while the write
   is in flight.
6. If the active tab has no live loaded rows, its tab record is stale/missing, or the
   notification modal is no longer active when the callback runs, do not open or
   dispatch a destructive operation.
7. Keep the existing atomic Rust `mark_tab_read` mutation, Python facade, tracked-task
   deduplication/exclusive scope, authoritative refresh, error handling, `R` binding,
   and `R: read tab` help copy unchanged.

## Implementation

### 1. Separate confirmation from the existing tracked mutation

- In `src/sase/ace/tui/modals/notification_modal_basic_actions.py`, import and reuse
  `ConfirmActionModal` and `ConfirmKind`; do not introduce another confirmation modal or
  overload the existing inline `y`/`n` state used for privileged notification
  dismissals.
- At the start of `action_read_tab()`, refresh the current tab classification once via
  the existing `_tag_tabs()` path. Resolve the active `NotificationTagTab` record from
  that snapshot so the prompt uses its canonical user-facing `label`, and capture:
  - the modal tag active when `R` was pressed;
  - its converted core tab key (`None` remains the core `general` key); and
  - the tuple of currently loaded notification IDs owned by that modal tag.
- Return without prompting if the active tab record or captured ID tuple is empty. Do
  not use the tab record's count in confirmation copy: it describes the in-memory page,
  while `_mark_tab_read()` intentionally operates beyond that page.
- Push a `ConfirmActionModal` whose title/copy clearly describes marking one tab read,
  whose subject names the captured display label, and whose danger/default/button
  configuration follows the required behavior above.
- Treat only `confirmed is True` as approval. Before dispatch, ensure the originating
  notification modal is still active enough to receive the eventual completion.

### 2. Preserve tab identity across confirmation and persistence

- Extract the current tracked-task body from `action_read_tab()` into a focused private
  dispatcher that accepts the already captured core tab key and ID tuple. The
  confirmation callback calls this dispatcher; it must not recalculate the active tab or
  target IDs.
- Keep the dispatcher's `_mark_tab_read(core_tab_key)` call and
  `_submit_notification_state_task(label="Read tab", ...)` contract intact, including
  the `notification-state` dedup key and exclusive scope supplied by the shared action
  support mixin.
- Leave `_complete_read_tab()` behavior intact: request an authoritative refresh, report
  failures without flipping local flags, mark only captured loaded IDs on success, and
  rebuild around the user's then-current tab/highlight without switching back to the
  confirmed tab.
- Do not change the Rust wire/store, Python notification-store facade, keymap binding,
  footer text, or `default_config.yml`; those surfaces already implement the intended
  tab-scoped operation.

### 3. Extend focused notification-modal coverage

- Update `tests/test_notification_modal_read_tab.py` so the dispatch tests drive the new
  two-stage flow: inspect the `ConfirmActionModal` passed to `push_screen`, invoke its
  dismissal callback explicitly, and then inspect/run the tracked task.
- Add assertions that the prompt:
  - appears before any tracked submission or `_mark_tab_read()` call;
  - is danger-styled with Cancel as the default;
  - uses `Mark read`/`Cancel` actions;
  - names the correct user-facing label for General and at least one synthetic,
    panel-owned, or ordinary tag tab; and
  - warns about the entire tab and unloaded rows without presenting the visible count as
    the total.
- Cover both `False` and `None` dismissal results and prove each performs no task
  submission, store call, local `.read` mutation, or refresh.
- On `True`, prove the tracked task receives the core key and ID tuple captured at the
  original `R` press. Retain the existing General-key conversion, panel/tag behavior,
  off-UI-thread persistence, error, success, authoritative-refresh, and tab-switch
  completion tests, adjusting their setup only as needed for the confirmation callback.
- Add a stale/empty target regression showing that no confirmation is pushed when there
  is nothing valid to mark. Continue relying on the shared confirm-dialog tests for
  generic `y`/`n`/Escape/button mechanics rather than duplicating that component's
  coverage.

## Verification

1. Run `just install` in the `sase` workspace before verification, as required for an
   ephemeral workspace.
2. Run focused tests for the read-tab flow, notification modal binding/help contract,
   tab classification, and the shared confirmation dialog, including at least:
   `tests/test_notification_modal_read_tab.py`,
   `tests/test_notification_modal_action_bindings.py`,
   `tests/test_notification_modal_mark_and_tabs.py`, and
   `tests/ace/tui/modals/test_confirm_dialog.py`.
3. Run `just check` after the file changes. If the scoped selector escalates or reports
   unusual selection, follow the repository's two-speed verification rule and run
   `just check-full` only through `/sase_monitor` with a concrete next action.
4. No new PNG golden is required when the implementation only composes the already
   snapshot-covered shared danger dialog without changing its CSS or rendering. If the
   implementation changes shared dialog presentation or adds notification-specific
   visual structure, run the affected visual snapshots, inspect the generated diffs,
   accept only intentional changes, and rerun to exact equality.

## Acceptance criteria

- One press of `R` cannot mark anything read until the user explicitly confirms a danger
  prompt whose default action is Cancel.
- The prompt names the captured tab and accurately warns that the whole tab, including
  notifications outside the visible page, is affected and cannot be restored from ACE.
- Every cancellation path is side-effect free.
- Confirmation marks the tab that was active when `R` was pressed, through the existing
  atomic core mutation and tracked background task, even if UI state changes before the
  write completes.
- Other tabs remain unread, completion does not navigate the user, failures do not
  change local read flags, and the binding/help text remains `Read Tab` / `R: read tab`.
- Focused tests and the required repository check pass.
