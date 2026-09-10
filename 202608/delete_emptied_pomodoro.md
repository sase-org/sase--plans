---
tier: tale
title: Delete Pomodoros emptied by Ctrl+Shift+M bullet moves
goal:
  Moving the last owned sub-bullet out of a Pomodoro deletes the empty source entry
  without disturbing the destination, surrounding ledger structure, or existing
  Ctrl+Shift+M behavior.
size: small
proposed_by: bbugyi200.athena.0ei
create_time: 2026-09-09 20:00:00
status: wip
---

# Delete a Pomodoro when Ctrl+Shift+M moves out its last sub-bullet

## Goal

Change the Pomodoro-sub-bullet branch of `Ctrl+Shift+M` so a move that removes the
source Pomodoro's last owned content deletes that Pomodoro entry instead of repairing it
with an empty child bullet. Keep the existing destination picker, count semantics,
subtree moves, duplicate merging, same-file atomic write, and ordinary cross-note task
moves unchanged.

This is a `tale`: the behavior is owned by one pure planner in one plugin, and one
follow-up coding agent can implement, test, release, and deploy the bounded change. It
is `small` because the root cause and consumers are known, although the implementation
must handle line-index shifts carefully.

## Repository and current behavior

All implementation work belongs in the `bob-plugins` linked repository. Open it through
`/sase_repo` and use the printed checkout path for all reads and writes:

```bash
sase repo open bob-plugins -r "Delete Pomodoros emptied by Ctrl+Shift+M bullet moves"
```

Do not edit the deployed vault plugin directly. `bob-plugins/AGENTS.md` requires running
`bob plugins sync` after source changes.

The relevant implementation is in `plugins/bob-navigation-hotkeys/main.js:6798`, where
`planPomodoroBulletMove()` currently:

1. captures and removes the selected bullet subtrees;
2. scans the source entry's remaining child block;
3. inserts `${sourceEntry.childIndent}- ` when that block has no nonblank content; and
4. reports `sourcePlaceholderInserted: true`.

The placeholder is intentional legacy behavior from the original Pomodoro move plan. It
is asserted in `scripts/test-navigation-hotkeys.cjs` by the single, counted, create-new,
duplicate, and out-and-back planner cases, and documented in the
`bob-navigation-hotkeys` README row as "emptied sources keep a blank sub-bullet." The
existing focused suite is green (271 tests), and manifest validation is green for all
six plugins before this change.

## Required behavior

### When the source is deleted

After the selected subtrees are removed, inspect the source Pomodoro's owned child block
in the post-removal line array. If it contains no nonblank content, delete the complete
source Pomodoro entry instead of adding a placeholder. This applies uniformly to open,
completed, and cancelled source entries because the existing keymap deliberately allows
corrections from any Pomodoro status.

Delete only the source ledger entry and content owned by that entry. Preserve the
`## Pomodoros` heading, neighboring Pomodoros, the following section, the document's
trailing-newline state, and CRLF versus LF. At the deletion seam, follow the existing
task/Pomodoro range-removal convention: avoid creating a doubled blank separator while
retaining one existing separator rather than globally reformatting the section.

The emptiness decision is content-safe. If any nonblank owned line remains—even an
unusual indented continuation or note that is not another list item—keep the source
Pomodoro and that content exactly; deleting it would lose data. Moving a nested
grandchild also keeps the source whenever its ancestor or another owned line remains.

### Destination and cursor invariants

- For an existing destination before or after the source, re-resolve its final line
  after source removal and append/merge the moved blocks exactly as today. A destination
  after the source must account for the deleted entry, removed subtrees, and any
  collapsed blank seam; do not rely on stale picker line numbers when writing the
  result.
- For a new named destination when the source is deleted, insert the new
  `- [ ] () — NAME` entry at the deleted source's former position. This is the only
  meaningful continuation of the current "directly below the source" ordering rule once
  the source no longer exists, and it prevents the new entry from drifting below a later
  Pomodoro.
- Preserve source order, descendant relative indentation, destination placeholder
  replacement, and exact-single-line duplicate merging.
- If every selected block merges into an existing destination, still delete the
  now-empty source. Keep `firstMovedLine` pointed at the surviving destination entry, so
  the existing commit path's cursor clamping remains valid.
- The final editor update remains one guarded `applyEditorContentTransaction` and
  therefore one undo step. Picker/routing code and success-notice wording do not need to
  change.

## Implementation

1. Refactor the source-repair portion of `planPomodoroBulletMove()` in
   `plugins/bob-navigation-hotkeys/main.js` into source cleanup:
   - capture all selected blocks before structural deletion, as now;
   - determine whether nonblank source-owned content survives after those blocks are
     removed;
   - when content survives, keep the source and make no placeholder edit;
   - when nothing survives, remove the source entry with a small pure range/seam helper
     (or equivalently clear inline logic) that returns a stable insertion anchor for the
     create-new case;
   - collect/re-resolve Pomodoro entries against the cleaned content before locating an
     existing destination, so entries on either side of the source receive correct final
     indexes; and
   - insert a new destination at the saved source anchor when the source was deleted,
     otherwise retain the current immediately-after-source placement for the defensive
     case where nonblank source content remains.

2. Replace the obsolete planner result flag `sourcePlaceholderInserted` with
   `sourcePomodoroDeleted` in both valid and invalid result shapes. Set it only when the
   source ledger entry was actually removed. No production consumer currently reads the
   old field, but the explicit new result makes the planner contract and regression
   assertions truthful.

3. Leave `commitPomodoroBulletMoveSession()` structurally unchanged unless a small
   adjustment is required to consume the planner's corrected final line numbers. It must
   continue to reject stale sessions, apply exactly one editor transaction, restore
   focus/scroll, clamp the cursor to `firstMovedLine`, and emit the existing move
   notice.

4. Release the bug fix in `bob-plugins`:
   - bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.28.0` to `1.28.1`;
   - update the corresponding README version reference; and
   - replace the README's "emptied sources keep a blank sub-bullet" statement with the
     new deletion behavior. The current Pomodoro glossary does not mention source
     placeholders, so no `bob-cli` memory edit is needed.

## Tests

Update `scripts/test-navigation-hotkeys.cjs` around the existing Pomodoro move planner
and runtime tests. Replace assertions that codify placeholder repair and add focused
cases proving:

1. Moving the sole child into a later existing destination removes the source entry,
   reports `sourcePomodoroDeleted: true`, appends at the correct final destination line,
   and leaves no empty bullet.
2. Moving the sole child into an earlier existing destination also removes the source
   and computes `destinationEntryLine` / `firstMovedLine` correctly.
3. A counted move that consumes every source sibling (including a subtree with
   descendants) deletes the source, while moving only some siblings or a nested
   grandchild with an ancestor left behind preserves it and reports
   `sourcePomodoroDeleted: false`.
4. A source with any surviving nonblank owned content is preserved byte for byte and
   receives no placeholder.
5. Creating a named destination from the last source bullet places the new entry at the
   deleted source's former position, for both a middle entry and the only Pomodoro in
   the section.
6. When the moved single-line block is an exact duplicate and therefore nothing is
   inserted, the empty source is still deleted and the cursor target remains the
   surviving destination entry. Preserve the existing near-match and lone
   destination-placeholder protections.
7. Open and closed source entries follow the same deletion rule.
8. First, middle, and last entry deletion preserve the following `##` boundary, one
   intended blank separator, trailing-newline state, and CRLF input.
9. The `commitPomodoroBulletMoveSession()` integration test observes the same deletion
   in the editor, one transaction/undo group, correct cursor placement, and unchanged
   notice text. The Ctrl+Shift+M dispatcher and ordinary task-move registration tests
   remain green.

The former out-and-back test should be rewritten for the new invariant: once a source
Pomodoro is deleted, moving the bullet back cannot restore that deleted entry
automatically. Assert the resulting documented ordering instead of expecting placeholder
repair.

## Verification and deployment

Run the focused suite first, then the complete repository gates:

```bash
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
```

Review the final diff to confirm only the planner, its tests, release metadata, and
documentation changed. Then preview and perform the required targeted sync from the
linked checkout (use its actual path printed by `/sase_repo`):

```bash
bob plugins sync --no-pull --dry-run -p bob-navigation-hotkeys -r "<bob-plugins checkout path>"
bob plugins sync --no-pull -p bob-navigation-hotkeys -r "<bob-plugins checkout path>"
```

Do not use `--force` to bypass a dirty deployed copy. Confirm the deployed `main.js` and
`manifest.json` match source at `1.28.1`, and finish with clean or fully accounted-for
`git status` output in both the linked repo and the primary workspace.

## Out of scope

- Changing which Pomodoro statuses may serve as sources or destinations.
- Changing counted-key semantics, picker filtering, move notices, or ordinary cross-note
  task moves.
- Reordering Pomodoros, moving bullets across notes, or changing named-Pomodoro parsing.
- Updating `task-status-cycler`'s independent Pomodoro-completion placeholder behavior.
