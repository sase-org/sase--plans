---
tier: tale
title: Prompt stash delete-all keymap
goal: Add a modal-scoped D shortcut that stages every stashed prompt for deletion
  while preserving the prompt stash panel's existing confirmation and selection semantics.
create_time: 2026-07-19 07:12:45
status: done
prompt: 202607/prompts/prompt_stash_delete_all.md
---

# Plan: Prompt stash delete-all keymap

## Context and intended behavior

The unified prompt-stash modal currently supports restoring all rows with `a` and toggling deletion of the highlighted
row with `d`, but clearing a large stash still requires visiting every row. Add an uppercase `D` shortcut labeled and
documented as “Delete All.” This is a panel-local key, matching the modal's existing static navigation and selection
bindings; it should not become an app-wide configurable keymap or alter `default_config.yml`.

`D` must stage every row currently represented by the modal for deletion rather than mutating the on-disk stash
immediately. The existing `Enter` confirmation path remains the persistence boundary, and `Escape`/`q` must continue to
cancel without deleting anything. Marking all rows for deletion must remove those same IDs from the restore selection so
no row is simultaneously restored and deleted, while pin state remains independent just as it is for single-row
deletion. A later `d` may unmark an individual row, and the existing restore-all action may replace delete marks with
restore marks.

## Modal behavior and discoverability

- Extend `StashedPromptsModal.BINDINGS` with `D` mapped to a dedicated delete-all action and expose “Delete All” as its
  binding description.
- Implement the action against the modal's complete entry-ID set: no-op safely when there are no entries, add every ID
  to `_deleted`, remove every ID from `_pop`, and refresh the rendered rows once so all rows display the existing red
  deletion treatment.
- Keep result construction and store mutation unchanged. On confirmation, the modal should return all staged IDs through
  `StashRestoreResult.delete_ids`; the app layer will continue removing them in its existing asynchronous,
  single-store-call flow and reporting the deletion count.
- Update the modal/module behavior documentation, the test module overview, and the on-screen hint text so `d` remains
  visibly distinct as single-row delete and `D` is advertised as delete all. Keep the hint readable in both split-pane
  and narrow layouts.

## Verification

- Add focused Textual pilot coverage that starts from mixed selection state (including an existing restore mark and
  representative pinned or bundled rows), presses `D`, and verifies that every entry is deletion-marked, conflicting
  restore marks are cleared, pins remain unchanged, and `Enter` returns only the complete delete-ID set. Exercise
  subsequent single-row `d` unmarking if needed to lock in the staged-selection semantics.
- Extend the existing title/hint assertions to require the new `D` delete-all affordance and retain the established `d`
  single-delete guidance.
- Because the visible hint changes, regenerate only the affected prompt-stash modal PNG goldens (the regular restore
  panel, bundle-preview panel, and narrow panel), inspect the produced actual/expected/diff artifacts, and run the
  dedicated visual snapshot test to ensure the new help text fits without unintended layout regressions.
- Run `just install` before repository checks, then run the focused prompt-stash modal tests, the prompt-stash visual
  snapshot tests, and the required full `just check` suite.

## Acceptance criteria

- Pressing `D` in a non-empty prompt-stash panel visually marks every row for deletion without touching persistent
  storage before confirmation.
- No entry remains selected for restore after `D`; persisted pin state is not changed.
- `Enter` deletes all staged rows through the existing result/application path, while canceling still leaves the stash
  untouched.
- The panel advertises both `d` for one-row deletion and `D` for delete all in supported terminal widths, and all
  focused, visual, and repository-wide checks pass.
