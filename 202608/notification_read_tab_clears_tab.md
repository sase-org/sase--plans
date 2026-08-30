---
tier: tale
title: Clear the notification tab in place when R marks it read
goal:
  Confirming R in the notification modal immediately removes the marked-read rows and
  their now-empty tab from the open panel, so the user never has to close and reopen the
  panel to see that the tab is gone.
size: small
proposed_by: bbugyi200.athena.0gb
create_time: 2026-08-30 11:11:35
status: wip
---

# Plan: Clear the notification tab in place when `R` marks it read

## Problem

In the ACE notification modal, `R` (`read_tab`) marks every unread notification in the
active tab as read. It opens a `y`/`n` danger confirmation ("Mark Notification Tab
Read?"), and on confirmation dispatches a durable `sase notify apply-state-many read`
proc scoped to the tab.

The store write is correct, but the open modal never reflects it:

- The modal is built from the **unread** page only
  (`src/sase/ace/tui/actions/agents/_notification_provider_direct.py:87` filters
  `not n.read and not n.silent`), so a marked-read row is gone the next time the panel
  is opened.
- `_complete_read_tab`
  (`src/sase/ace/tui/modals/notification_modal_basic_actions.py:262`) only flips
  `notification.read = True` on the in-memory rows and rebuilds the list. The rows stay
  in `self._notifications`, so they stay in the tab, and the tab strip keeps the tab
  with its full count (tab counts come from
  `classify_notification_modal_tabs(self._notifications)` in
  `notification_modal.py:_tag_tabs`, which does not filter on `read`).

The only visible effect of a successful `R` is that each row loses its `*` unread marker
— and there is no toast at all. The user has to close the panel and reopen it to learn
that the tab was emptied and no longer exists.

Every other mutating action in this modal already removes or reclassifies its rows
locally: `_dismiss_notification_by_index` and `_bulk_dismiss_marked_ids` drop rows and
re-coerce the active tab; `_complete_bulk_toggle_mute` mutates rows and calls
`_rebuild_after_bulk_notification_reclassification` so a row that moved to the Muted tab
moves in the visible strip too. `_complete_read_tab` is the one path that leaves the
modal showing state the store no longer has.

## Approach

Make a successful tab read behave, locally, like a bulk dismiss: drop the acted rows
from the modal's dataset, let the existing tab-coercion helper retarget the active tab,
and confirm the write with a toast.

This is presentation-only. The store semantics (`mark_tab_read`, tab classification) are
already correct and live in the Rust core; nothing in `../sase-core` changes.

**Rejected alternative:** re-pull the authoritative unread page from the provider on
completion and replace `self._notifications`. It would also self-heal any divergence
between the modal's frozen page and the store, but it breaks the modal's frozen-page
contract (rows arriving mid-session would appear under the cursor, and the highlight
would churn), and it diverges from how every other mutation path in this modal updates
itself. Not worth it for this bug.

## Implementation

### 1. Rewrite `_complete_read_tab`

File: `src/sase/ace/tui/modals/notification_modal_basic_actions.py`.

Replace the body of `_complete_read_tab` (currently lines 262-281) so that, after the
existing failure and still-active guards, it mirrors `_complete_bulk_toggle_mute` in
`notification_modal_mute_actions.py` but removes rows instead of mutating them:

```python
    def _complete_read_tab(self: Any, result: NotificationMutationResult) -> None:
        """Apply a completed tab-scoped read mutation on the UI thread."""
        self._request_authoritative_notification_refresh()
        if not result.success:
            self.notify(f"Could not mark tab read: {result.message}", severity="error")
            return
        if not self._notification_modal_still_active():
            return

        acted_ids = set(result.ids)
        previous_tabs = self._tag_tabs()
        current = self._get_highlighted_notification()
        preferred_id = current.id if current is not None else None
        indices = [i for i, n in enumerate(self._notifications) if n.id in acted_ids]
        replacement_id = self._replacement_notification_id_after_bulk_dismiss(indices)
        for notification in self._notifications:
            if notification.id in acted_ids:
                notification.read = True
        self._notifications = [n for n in self._notifications if n.id not in acted_ids]
        self._forget_removed_notification_targets(acted_ids)
        self._rebuild_after_bulk_notification_reclassification(
            previous_tabs=previous_tabs,
            replacement_notification_id=replacement_id,
            preferred_notification_id=preferred_id,
        )
        count = result.matched_count or len(indices)
        if count:
            self.notify(f"Marked {count} notification{'s' if count != 1 else ''} read")
```

Ordering constraints that must hold:

- `previous_tabs` and `replacement_id` are computed **before** removal.
  `_replacement_notification_id_after_bulk_dismiss` reads
  `_visual_notification_index_order()`, which is scoped to the currently active tab, and
  `_tag_tabs()` also refreshes `_notification_tab_keys` for the pre-removal set.
- `preferred_id` needs no "was it removed?" guard:
  `_rebuild_after_bulk_notification_reclassification` resolves it through
  `_visible_notification_index_for_id`, which returns `None` for an id no longer in
  `self._notifications`, and then falls back to `replacement_id` and finally to
  `_first_visible_notification_index()`.
- Keep `notification.read = True` on the acted rows before dropping them. The modal
  shares those `Notification` objects with the caller's page list, and it keeps the
  detached objects truthful.
- Rebinding `self._notifications` to a new list is the same pattern
  `_bulk_dismiss_marked_ids` already uses.

Toast count: `result.matched_count` is the store-side changed count (set by both
`_run_apply_state_many` in `src/sase/ops/commands/notify.py` and the direct fallback in
`notification_modal_action_support.py`), so it can legitimately exceed the number of
visible rows — the confirmation dialog already warns that the write covers rows ACE has
not loaded, and reporting the real number is the honest confirmation. Fall back to
`len(indices)` when a completion payload is missing, and skip the toast entirely when
both are zero.

### 2. Add `_forget_removed_notification_targets`

Add a small private helper to `NotificationBasicActionsMixin` (same file), placed next
to `_bulk_dismiss_marked_ids`, that drops modal-local state pointing at rows that no
longer exist:

```python
    def _forget_removed_notification_targets(
        self: Any, removed_ids: set[str]
    ) -> None:
        """Drop marks and pending confirmations aimed at rows that are gone."""
        self._marked_notification_ids.difference_update(removed_ids)
        if self._pending_confirm_notification_id in removed_ids:
            self._pending_confirm_notification_id = None
        pending_ids = self._pending_confirm_notification_ids
        if pending_ids is not None:
            remaining = [n_id for n_id in pending_ids if n_id not in removed_ids]
            self._pending_confirm_notification_ids = remaining or None
```

Without this, a mark or a live `(y/n)` dismiss confirmation could survive pointing at a
row the panel no longer shows. Those paths already fail safe (they re-resolve ids
against `self._notifications`), but leaving the stale state around means `m`-marks
silently survive into an unrelated tab.

Do not call this from the dismiss paths in this change; `_bulk_dismiss_marked_ids`
already clears marks wholesale and rewiring it is out of scope.

### 3. Update `docs/notifications.md`

The `R` paragraph (starting "`R` is scoped to the tab you are on") documents the current
"wider write than it looks" behavior but says nothing about what the panel does
afterwards. Add to that paragraph that on confirmation the marked-read rows leave the
visible list immediately, the emptied tab disappears from the tab strip, the modal moves
to the nearest surviving tab (or shows `No unread notifications` when that was the last
tab), and a toast reports how many notifications the store marked read.

Leave the keybinding table row for `R` as is — "Mark every unread notification in the
**active tab** read (confirms first)" is still accurate.

Do not touch `CHANGELOG.md`; it is generated by release-please from commit subjects
(`tools/validate_changelog` enforces this).

## Tests

All in `tests/test_notification_modal_read_tab.py` (315 lines today; the additions keep
it well inside the `toobig` thresholds). Existing helpers live in
`tests/_notification_modal_helpers.py`.

Two of the current tests encode the old behavior and must be updated, not deleted:

1. `test_complete_read_tab_success_marks_only_captured_ids_and_refreshes` — rename to
   something like `test_complete_read_tab_success_drops_acted_rows_and_refreshes` and
   assert the acted row is gone from `modal._notifications` while the untouched row
   remains and stays unread, keeping the existing
   `_schedule_notification_poll(source="mutation")` and `_rebuild_list` assertions.
2. `test_complete_read_tab_after_tab_switch_keeps_new_tab_and_leaves_it_unread` — the
   in-flight tab switch must still leave `_active_notification_tag == "beta"` and `b1`
   unread, but `a1` is now removed from `modal._notifications` rather than left in place
   with `read is True`.

Also extend `test_complete_read_tab_error_leaves_read_flags_unchanged_and_notifies` with
an assertion that a failed write removes nothing.

New tests to add:

3. **Emptied tab disappears and the modal moves to a neighbor.** Rows `a1` (tag `alpha`)
   and `b1` (tag `beta`), active tag `alpha`, result ids `("a1",)`. After completion:
   `modal._notifications` holds only `b1`, `modal._active_notification_tag == "beta"`,
   and `"alpha"` is not in `[tab.tag for tab in modal._tag_tabs()]`.
4. **Reading the last tab empties the modal.** One row, active tag `alpha`, result ids
   `("a1",)`. After completion: `modal._notifications == []`,
   `modal._active_notification_tag is None`, `modal._tag_tabs() == []`. Mock
   `_rebuild_list` for this one — the real empty branch of `_rebuild_list` mounts the
   `No unread notifications` `Static` via `query_one("#notification-left", ...)`, which
   the fake option list helper does not stub.
5. **The visible option list actually loses the rows.** Use
   `_wire_fake_option_list(modal)` with the real `_rebuild_list` (mock `_display_file`
   and `notify`, as `tests/test_notification_modal_section_toggle.py` does) on a modal
   whose active tab survives the read — e.g. read tab `alpha` while `beta` still has
   rows — and assert the resulting option ids contain no acted id. This is the test that
   would have caught the reported bug end to end.
6. **Stale modal state is dropped.** Two `alpha` rows, both in
   `_marked_notification_ids`, with `_pending_confirm_notification_id` set to one of
   them; after a successful read of both, `_marked_notification_ids` is empty and
   `_pending_confirm_notification_id is None`.
7. **Toast copy.** `matched_count=5` with ids `("n1",)` notifies
   `"Marked 5 notifications read"`; `matched_count=1` notifies
   `"Marked 1 notification read"` (singular). Assert no toast when the result reports
   nothing acted and no local rows matched.

Keep the existing dispatch/confirmation tests (target freezing, cancellation,
inactive-modal callback, no-prompt cases) unchanged — none of that behavior moves.

## Verification

Run from a workspace clone of the sase repo:

```bash
just install   # only if this workspace's venv is stale
just check
```

`just check` covers the whole-repo lint gates (ruff, mypy, symvision, toobig,
keep-sorted, changelog, docs) plus a diff-scoped test lane that will select the
notification-modal tests. If the scoped selection escalates or reports anything unusual,
run `just check-full` through the `/sase_monitor` skill with the `TESTING` / `TESTED`
status pair — never inline.

No PNG snapshot updates are expected: the ACE visual goldens under
`tests/ace/tui/visual/snapshots/png/` capture statically rendered modal states, and this
change only alters what happens after a confirmed `R`. If `just test-visual` does flag a
diff, treat it as a real regression rather than accepting it.

## Out of scope

- Any change under `../sase-core`. `mark_tab_read` and tab classification already do the
  right thing; this is a TUI presentation fix.
- Re-pulling the unread page into an open modal (see the rejected alternative above).
- Changing the `R` binding, the confirmation dialog copy, or the hint-footer text in
  `notification_modal_constants.py` — `R: read tab` stays accurate.
- The dismiss paths' own handling of marks and pending confirmations.

## Known limitation to accept, not fix

The removed set is the id set frozen when the confirmation opened, and the store write
is keyed by tab, so the two can disagree in a narrow window: a row that changed tabs
between the prompt and completion (for example, muted by another action) may be removed
from the panel without the store having marked it read, and a row that entered the tab
after capture is marked read by the store while staying visible until the panel is
reopened. Both writes serialize on the `notification-state` concurrency key, so this
requires a deliberate mutation inside the in-flight window, and removing exactly what
the user confirmed is the more predictable behavior. Do not add reconciliation machinery
for it.
