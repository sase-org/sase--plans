---
tier: tale
title: Keep the prompt stash panel open after a partial delete
goal:
  Confirming delete marks with Enter in the prompt stash panel deletes those rows and
  keeps the panel open on the remaining entries, closing only when nothing remains or
  when rows are also being restored.
size: small
proposed_by: bbugyi200.athena.0pz
create_time: 2026-09-23 10:14:08
status: wip
---

# Keep the prompt stash panel open after a partial delete

## Problem

In the unified prompt stash picker (`StashedPromptsModal`, opened with `Ctrl+G p` in the
prompt bar or `@` from the TUI tabs), `d` / `D` mark rows for deletion and `Enter`
confirms. Today `action_confirm` always calls `self.dismiss(...)`, so the panel closes
after every confirm, even when only some rows were deleted and others are still stashed.
The app then deletes the rows in `_on_prompt_stash_restore_confirmed` →
`_apply_stash_restore`.

Closing is only right when the delete would leave the panel empty. After a partial,
delete-only confirm, the panel should stay open and show the remaining entries.

## Desired behavior

`Enter` in `action_confirm` handles these cases:

1. **Restore marks present** (`tab` / `a` marked at least one row), with or without
   delete marks: no change. Dismiss with the full `StashRestoreResult`. Restoring loads
   drafts into the prompt bar, so the panel has to close.
2. **Delete-only, deleting every remaining row** (e.g. `D` then `Enter`, or `d` on the
   only row): no change. Dismiss with `StashRestoreResult(delete_ids=...)` so the app
   deletes the rows and shows the usual "Deleted N stashed prompts" message.
3. **Delete-only, some rows remain**: new behavior. The panel stays open, the marked
   rows are deleted from the store, and they disappear from the list. The title count,
   highlight, preview, and number keys all reflect the remaining rows.
4. **No marks**: no change. Restore the highlighted row, or dismiss with `None`.

## Design

Keep the modal's existing rule that it never touches the store. Pin toggles already work
this way: the modal posts `PinToggled` and the app persists it off the event loop.
Deletes will follow the same pattern.

### Modal: `src/sase/ace/tui/modals/stashed_prompts_modal.py`

- Add a nested message `DeleteRequested(Message)` next to `PinToggled`. It carries
  `entry_ids: list[str]` in display order. Its docstring says it is posted when `Enter`
  confirms a delete-only selection that leaves at least one row, so the app should
  delete those ids immediately while the panel stays open.
- In `action_confirm`, after computing `marked` / `delete_ids`: if `not marked` and
  `delete_ids` is non-empty and `len(delete_ids) < len(self._entries)`, call a new
  helper `self._apply_deletions_in_place(delete_ids)` and `return` without dismissing.
  All other paths stay as they are.
- `_apply_deletions_in_place(delete_ids)` should:
  1. Record which entry is highlighted before the change
     (`_highlighted_index_and_entry()`).
  2. Post `self.DeleteRequested(list(delete_ids))`.
  3. Remove the deleted ids from `self._entries`, `self._prompt_counts`,
     `self._highlight_cache`, `self._pinned`, and `self._pop`, then clear
     `self._deleted`.
  4. Update the title `Label` (`#stashed-prompts-title`) with `self._title_text()`,
     which reads `len(self._entries)`, so the count drops.
  5. Rebuild the rows with `_refresh_rows()` (it already clamps the highlight). Then set
     the highlight: if the previously highlighted entry survived, highlight its new
     index; otherwise highlight the entry that now sits at the old index, or the last
     row if the old index is past the end. Set the highlight inside the
     `_refreshing_options` guard, or set it after `_refresh_rows()` and paint the
     preview directly. Either way, the preview must show the newly highlighted entry and
     never a deleted one.
  6. Paint the preview for the new highlighted entry with `_paint_preview(entry_id)`,
     directly and without the debouncer, because the panel just changed shape.
- Update the module docstring and the `_hint_text` wording only if they now say
  something wrong. The hint is two lines, so don't make it longer.

### App layer: `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore.py`

- Add `on_stashed_prompts_modal_delete_requested(self, event: object)`, modeled on
  `on_stashed_prompts_modal_pin_toggled`. It checks
  `isinstance(event, StashedPromptsModal.DeleteRequested)`, then
  `self._spawn_prompt_stash_task(self._delete_prompt_stash_entries_async(event.entry_ids))`.
- Add `async def _delete_prompt_stash_entries_async(self, entry_ids: list[str])`. It:
  - takes `self._prompt_stash_write_lock()`, the same lock pin writes use, so a pin
    toggle and a delete made from the same open panel apply in order;
  - runs `pop_prompt_stash(prompt_stash_path(), entry_ids)` in `asyncio.to_thread`;
  - on an exception, calls
    `self.notify(self._prompt_stash_error_message("Failed to delete stashed prompt", exc), severity="error")`
    and returns;
  - on success, counts the ids actually removed (`{e.id for e in outcome.removed}`
    intersected with `entry_ids`), calls `self._notify_restore_outcome(0, deleted)` so
    the message matches the existing wording ("Deleted stashed prompt" / "Deleted N
    stashed prompts"), and calls
    `self._apply_prompt_stash_snapshot_counts(outcome.snapshot)` to refresh the top-bar
    badge.
- `_apply_stash_restore` and `StashRestoreResult` do not change. They still handle cases
  1 and 2 when the panel is dismissed.

### Docs: `docs/ace.md`

In the "stashed-prompt picker" paragraph (search for "`D` marks every row for
deletion"), add one sentence: confirming delete marks with `Enter` deletes those rows
and keeps the picker open on the remaining entries, and the picker closes only when
nothing remains or when rows are also being restored. Keep the paragraph's style. No
config or keymap changes are needed, so `src/sase/default_config.yml` stays as is.

## Tests

Modal tests belong in `tests/ace/tui/modals/`. Use `ModalHost` / `make_entry` from
`stashed_prompts_modal_test_helpers.py`. Add a `delete_events` list to `ModalHost` and
an `on_stashed_prompts_modal_delete_requested` handler that appends to it, the same way
`pin_events` works. Put the new modal tests in a new focused module,
`tests/ace/tui/modals/test_stashed_prompts_modal_delete.py`, so
`test_stashed_prompts_modal_selection.py` doesn't grow:

- **Update** `test_delete_mark_returns_delete_ids_not_restore` in
  `test_stashed_prompts_modal_selection.py`, since it deletes 1 of 2 rows and now
  expects the panel to stay open. Either rename and move it, or rewrite it to assert:
  `app.result == "UNSET"` (not dismissed), `app.screen` is still the modal, exactly one
  `DeleteRequested` with `entry_ids == ["a"]`,
  `[e.id for e in modal._entries] == ["b"]`, `modal._deleted == set()`, and the title
  label shows `Stashed prompts (1)`.
- New: after a partial delete, the remaining row can still be restored. Press `enter`
  again (no marks) or `1`, and it dismisses with `pop_ids == ["b"]`.
- New: several delete marks (e.g. `d` on rows 1 and 3 of 4) leave the panel open with
  the 2 remaining rows in their original order, and the highlight/preview is on a
  surviving row.
- New: `D` followed by `d` on one row, then `enter`, deletes all but that row and keeps
  the panel open with only that row.
- New: pins survive a partial delete. A pinned remaining row is still in
  `modal._pinned`, and restoring it after the delete returns `keep_ids`.
- Existing tests that delete every row stay as they are and must still pass, because
  they still dismiss with `delete_ids`: `test_pin_is_orthogonal_to_delete_selection`,
  `test_delete_wins_over_prior_pop_selection`,
  `test_delete_all_replaces_restore_marks_and_preserves_pins`, and
  `test_delete_mark_can_delete_bundle_row`.
- New: a mixed restore + delete confirm (`tab` on one row, `d` on another, with a third
  unmarked row left) still dismisses with both `pop_ids` and `delete_ids`, and posts no
  `DeleteRequested`.

App-layer tests go in `tests/ace/tui/actions/test_prompt_stash_restore_confirm.py`,
using the existing `_RestoreHarness`, `_seed`, `_point_store_at`,
`_wait_prompt_stash_tasks`, and `_skip_without_prompt_stash_bindings` helpers:

- New:
  `harness.on_stashed_prompts_modal_delete_requested(StashedPromptsModal.DeleteRequested(["a"]))`
  with seeded entries `a`, `b` removes `a` from the store, loads nothing into the bar
  (`bar.restored is None`, `home_mounts == []`), shows `"Deleted stashed prompt"`, and
  records the badge count (`applied_counts == [1]`).
- New: deleting two ids shows `"Deleted 2 stashed prompts"`.
- New: a non-`DeleteRequested` event passed to the handler does nothing.

## Verification

Read the `lint_and_test` SASE memory note before finishing and run the verification
recipe it prescribes (`just check`, using the guarded tool-run flow it describes).
Everything must pass, including the updated and new tests above.
