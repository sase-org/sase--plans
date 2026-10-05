---
tier: tale
title: Skip the Stash to Trash confirmation
goal: Sending a stashed prompt to Trash no longer shows a y/n dialog, because the
  draft stays recoverable from the Trash view.
size: small
proposed_by: bbugyi200.athena.0x0
status: done
---

# Plan: Skip the Stash to Trash confirmation

## Outcome

In the Prompts overlay, `d` or `D` followed by `Enter` moves the marked Stash rows to
Trash immediately. No `ConfirmActionModal` appears, and the user does not press `y` or
`n`. The existing success toast still reports the move. When the batch evicts older
Trash rows, that same toast still names the actual evictions and
`sase prompt stash-archive`.

The discarded draft is recoverable from the Trash view: `t` or the trash chip opens it,
and `Enter` there puts the row back in Stash. The stored entry keeps its pin,
frontmatter, project, cursor, and bundled panes across that round trip. That recovery is
why the blocking prompt goes away.

## Why this prompt exists today

Both Stash → Trash paths share `_confirm_trash_commit_async` in
`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash_restore_trash.py`:

- A partial discard posts `TrashRequested` and `_trash_staged_in_place_async` confirms
  before writing, leaving the overlay open.
- A discard of every row, or a discard combined with restores, dismisses the overlay
  with `StashRestoreResult.trash_ids`. `_apply_prompts_stash_result` confirms before
  writing.

The dialog is `ConfirmActionModal` titled `Move to Trash`, kind `DANGER`, so the default
button is No. `y` confirms and `n`, `q`, and `Esc` cancel. The body comes from
`trash_commit_confirm_text`: always `Move N draft(s) to Trash?`, plus a pin line when
any marked row is pinned, plus an expected-eviction line when `trash_count + marked`
exceeds `trash_limit`.

`d` / `D` is already the explicit discard mark. `Enter` applies that mark. The y/n is a
second confirmation of a move into the recovery bin.

The 2026-09 epic `plan:202609/prompt_recall_tabs_and_stash_trash.md` required an
explicit confirm before a pinned row moved, and an upfront expected-loss count before a
batch that would evict Trash rows. This tale supersedes that presentation. Do not keep a
conditional y/n for pins or for expected evictions.

## Decision

Remove the dialog for every Stash → Trash commit, including pinned rows and batches
whose preview `expected_evictions` is greater than zero.

Pinned rows are still recoverable.
`tests/test_core_facade/test_prompt_stash_lifecycle.py` shows a pinned entry stays
pinned inside the trash record. Restoring it returns that same entry to Stash.

Evicted rows leave the Trash view. They remain recoverable with
`sase prompt stash-archive` (reason `evicted`). `trash_outcome_text` already appends the
actual eviction count, the evicted ids, and that command after a successful write.
Report the loss there. Do not block the move to preview it.

A row the user just discarded is the newest trash record, so an ordinary overflow evicts
older Trash rows, not the row they just sent. A batch larger than `trash_limit` can
evict some of the newly trashed rows in the same transaction. The toast covers that case
too.

## Keep unchanged

- Trash-view permanent purge. `d` / `D` then `Enter` in the Trash view still goes
  through `_confirm_purge_async` and `purge_confirm_text`
  (`Permanently delete N drafts from Trash?`). `_confirm_async` stays for that path.
- `ace.prompt_stash.trash_limit: 0`. Delete marks are permanent deletions, not Trash
  moves, and a partial discard already applies with no confirmation. Do not add a prompt
  there.
- Successful unpinned Stash restores (`Enter`, digits, `@`). Those leave Stash and do
  not enter Trash.
- Prompt-history deletion and `sase prompt delete`.
- The Rust stash lifecycle, `trash_prompt_stash`, and the wire types. This is a TUI
  presentation change.
- Authoritative repaint from the store outcome, the write lock, and the failure toasts
  `Failed to move drafts to Trash` and `Failed to read stashed prompts`.

## Implementation

1. Stop pushing a confirmation from the Stash → Trash preflight. Keep the preflight's
   other behavior:
   - Re-read the overlay snapshot off the UI thread, as today.
   - On read failure, toast `Failed to read stashed prompts` and do not write.
   - If `preview_trash_commit` yields no live marked ids, return without writing and
     without a `Moved 0 drafts to Trash` toast. Unknown ids are stale no-ops.
   - When at least one marked id is still active, call `_trash_entries_async` with the
     original id list. Rust already ignores unknown ids.
2. Preserve the caller split in `_apply_prompts_stash_result`. A failed or empty trash
   preflight must not skip a mixed dismiss's `pop_ids`, `keep_ids`, or `delete_ids`.
3. Rename or inline the preflight so it no longer reads as a confirmation.
   `_confirm_async` remains the purge dialog helper.
4. Remove `trash_commit_confirm_text` and its re-export from
   `src/sase/ace/tui/modals/stash_pane.py` (`import` and `__all__`). Nothing else should
   call it. Keep `preview_trash_commit`, `TrashCommitPreview`, `trash_outcome_text`, and
   `trash_preview_for_marks`.
5. Update the `TrashCommitPreview` docstring. `pinned_ids` no longer means "needs
   explicit confirmation before the row may move." The field can stay so preview tests
   and `trash_preview_for_marks` keep working.
6. Update comments that say the host confirms a Stash → Trash move, in:
   - `_prompt_bar_stash_restore_trash.py` (class and method docs for the stash result
     and the in-place trash request)
   - `stash_controller.py` `action_confirm`
   - `stash_messages.py` `TrashRequested`
   - `_stash_controller_state.py` only if a comment claims the host confirms the move

   Leave purge comments that require an explicit confirmation.

7. In `docs/ace.md`, under the Prompts overlay Stash/Trash section:
   - State that `d` / `D` then `Enter` moves the marked rows to Trash immediately, with
     no y/n.
   - Replace the sentence that says an overflowing discard batch names the expected
     permanent-loss count up front. The success toast names the actual evictions.
   - In the compact demo, step 1 is highlight, `d`, `Enter`. The toast is
     `Moved 1 draft to Trash`. Remove the instruction to confirm
     `Move 1 draft to Trash?`.

   `docs/prompt.md` and `docs/configuration.md` do not describe this y/n. Leave them
   unless a sentence you touch claims an upfront Stash → Trash confirm.

## Tests

- Rewrite `test_stash_dismiss_with_trash_marks_confirms_then_moves` in
  `tests/ace/tui/actions/test_prompts_overlay_entry_points.py`. The dismiss moves the
  row without pushing `ConfirmActionModal` and without invoking a confirm callback. The
  store and the Trash toast still match.
- Cover the in-place path (`on_stashed_prompts_modal_trash_requested` /
  `_trash_staged_in_place_async`): a marked row moves with no dialog.
- Cover a pinned id and a batch that would evict: both move with no dialog, and the
  eviction toast still includes the permanently-deleted count and
  `sase prompt stash-archive`.
- Cover a stale id: no write and no move toast. Cover a snapshot read failure: the
  existing read-error toast, and no write.
- In `tests/ace/tui/modals/test_prompts_modal_trash.py`, drop assertions on
  `trash_commit_confirm_text` and `Move 2 drafts to Trash?`. Keep the
  `preview_trash_commit` count assertions and the `trash_outcome_text` assertions.
- Do not change purge confirmation tests in `tests/ace/tui/modals/test_trash_pane.py`.
  They must still expect `Permanently delete N drafts from Trash?`.

Run:

```bash
pytest tests/ace/tui/actions/test_prompts_overlay_entry_points.py tests/ace/tui/modals/test_prompts_modal_trash.py tests/ace/tui/modals/test_trash_pane.py
```

## Out of scope

- No new setting to bring the y/n back.
- No change to the Trash purge dialog, the limit-0 permanent delete path, or archive
  retention.
- No deletion of the unused `trash_preview_for_marks` helper.
- No Rust or `sase-core` edit.
