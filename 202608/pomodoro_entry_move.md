---
tier: tale
title: Extend Ctrl+Shift+M to Pomodoro entry lines with move-and-delete or rename
goal:
  Pressing Ctrl+Shift+M on a Pomodoro ledger entry line opens the same destination
  picker, where choosing another open Pomodoro moves every sub-bullet there and deletes
  the source entry, and typing a novel name renames the current entry in place, without
  changing the existing sub-bullet or cross-note task-move behavior.
size: medium
proposed_by: bbugyi200.athena.0el
create_time: 2026-09-09 20:00:19
status: wip
---

# Ctrl+Shift+M on a Pomodoro entry line

## Goal

Today `Ctrl+Shift+M` has two routes: a Pomodoro **sub-bullet** route (move selected
bullets into another open Pomodoro or into a new named one) and an ordinary cross-note
`#task` move route. Placing the cursor on the Pomodoro ledger **entry** line itself
(`- [ ] () — BODY`) falls through to the task route and fails with
`Move tasks must start on a real #task checkbox`.

Add a third route for that case, reusing the existing picker:

- **Choosing a different open Pomodoro** moves every sub-bullet the source entry owns
  into that destination and deletes the source entry.
- **Typing a name that is not an existing open Pomodoro's name** renames the current
  entry in place. It creates nothing and moves nothing.

This is a `tale`: the work lives in one plugin file plus its focused test suite, and one
follow-up coding agent can implement, test, release, and deploy it. It is `medium`
because it adds a planner mode, a second pure planner, a picker mode, a routing branch,
and a commit path, all of which must not perturb the freshly shipped sub-bullet flow.

## Repository and current behavior

All implementation work belongs in the `bob-plugins` linked repository. Open it through
`/sase_repo` and use the printed checkout path for every read and write:

```bash
sase repo open bob-plugins -r "Add Ctrl+Shift+M support on Pomodoro entry lines"
```

Do not edit the deployed vault plugin under `~/bob/` directly; `bob-plugins/AGENTS.md`
requires a `bob plugins sync` after source changes.

Everything below refers to `plugins/bob-navigation-hotkeys/main.js` at
`bob-navigation-hotkeys` version `1.28.1`, where the relevant pieces are:

| Symbol                                                                                      |               Line | Role                                                                                                                                         |
| ------------------------------------------------------------------------------------------- | -----------------: | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `parsePomodoroEntryLine`                                                                    |               6451 | Entry grammar: `status`, `open`, `rangeStart/rangeEnd`, `placeholder`, `name`, `nameStart/nameEnd`                                           |
| `normalizePomodoroName`                                                                     |               6510 | Strip em dashes, collapse whitespace, trim, uppercase, reject empty / `> 48` chars                                                           |
| `formatPomodoroEntryLine`                                                                   |               6533 | `- [ ] () — NAME`                                                                                                                            |
| `collectPomodoroEntries`                                                                    |               6544 | Every entry in `## Pomodoros` with `entryLine`, `position`, `open`, `childStartLine`, `childEndLineExclusive`, `childIndent`, preview fields |
| `findPomodoroBulletContext`                                                                 |               6629 | Sub-bullet route predicate; returns `null` on an entry line                                                                                  |
| `discoverMovablePomodoroBulletTargets`                                                      |               6663 | Sub-bullet targets with count clamping                                                                                                       |
| `capturePomodoroBulletSubtree` / `removePomodoroBulletRanges` / `rebasePomodoroBulletBlock` | 6733 / 6760 / 6785 | Pure move primitives                                                                                                                         |
| `planPomodoroBulletMove`                                                                    |               6800 | The move planner: capture, remove, delete emptied source, re-resolve destination, merge duplicates, insert                                   |
| `createPomodoroBulletMovePickerRows`                                                        |               7135 | `new` / `existing` / `invalid` rows                                                                                                          |
| `buildPomodoroBulletMoveNotice`                                                             |               7195 | Success notice                                                                                                                               |
| `renderPomodoroBulletMovePickerRow`                                                         |              10257 | Row DOM: icon, title, meta, badge                                                                                                            |
| `PomodoroBulletMovePickerModal`                                                             |              10410 | Picker copy and `openItem` wiring                                                                                                            |
| `openTaskMoveOrPomodoroBulletPicker`                                                        |              21659 | The `Ctrl+Shift+M` router                                                                                                                    |
| `openPomodoroBulletMovePicker`                                                              |              21670 | Session construction and picker open                                                                                                         |
| `commitPomodoroBulletMoveSession`                                                           |              21997 | Guarded single-transaction write                                                                                                             |
| helper export block                                                                         |             ~24189 | Test surface                                                                                                                                 |

The keymap itself is registered once (command `move-tasks-to-note`, hotkey
`Ctrl+Shift+M`) and additionally intercepted in Vim normal mode by
`handleCountedTaskMovePhysicalKeydown` (19800) for the counted `N<Ctrl+Shift+M>` form.
Both call `openTaskMoveOrPomodoroBulletPicker`. Neither needs to change.

Baseline before this change: `node --test scripts/test-navigation-hotkeys.cjs` is green
at 277 tests, and `npm test` / `npm run validate` are green.

## Required behavior

### Routing

Extend `openTaskMoveOrPomodoroBulletPicker` with a middle branch:

1. `findPomodoroBulletContext(content, cursor.line)` → existing sub-bullet picker.
2. otherwise a new `findPomodoroEntryContext(content, cursor.line)` → new entry picker.
3. otherwise → existing `openTaskMoveDestinationPicker`.

`findPomodoroEntryContext(content, line)` mirrors `findPomodoroBulletContext`: it uses
`collectPomodoroEntries` and returns
`{ entries, section, entry, entryIndex, entryLine }` when `line` is exactly some entry's
`entryLine`, else `null`. It therefore only fires inside an unfenced `## Pomodoros`
section, and only on a column-0 line that `parsePomodoroEntryLine` accepts. The two
Pomodoro contexts are disjoint: an entry's child block starts at `entryLine + 1`, so an
entry line never resolves as a sub-bullet.

Consequences to accept deliberately:

- Any entry status routes here — open, completed `[x]`, and cancelled `[-]` — matching
  the sub-bullet route, which already lets corrections start from any status.
- A column-0 `- [ ] ()`-shaped line outside `## Pomodoros` keeps falling through to the
  task route unchanged.
- A Pomodoro entry line that also carried a `#task` tag would now take the Pomodoro
  route. This mirrors the already-documented rule that nested `#task` lines under a
  Pomodoro route to the Pomodoro move.

### Count semantics

`N<Ctrl+Shift+M>` on an entry line performs the same whole-entry operation as the bare
key; the count has no meaning because an entry move is inherently all-or-nothing and a
rename cannot apply to several entries. Do not reject the keypress. Instead:

- accept and ignore `options.additionalTaskCount`;
- keep honoring `options.countExplicit` for the existing
  `activeTaskMoveDestinationPicker` supersede guard, exactly as the sub-bullet picker
  does, so a counted press can still replace an already-open bare picker; and
- when the count was explicit and greater than zero, append
  ` · count ignored on a Pomodoro entry` to the picker subtitle so the input is not
  silently swallowed.

### Discovery: what the entry owns

Add `discoverPomodoroEntryMoveTargets(content, line)`, shaped like the existing
discovery helpers (frozen, `{ valid, error, ... }`, never throwing):

- resolve the entry through `findPomodoroEntryContext`; invalid →
  `"Place the cursor on a Pomodoro entry"`;
- record `entryLine` and `rawEntryLine` (the exact line text) for the staleness guard;
- walk `[entry.childStartLine, entry.childEndLineExclusive)` and collect every list item
  whose indent display width equals `entry.childIndent`'s width — the entry's
  **top-level child bullets**, in source order, each as `{ line, rawLine }`. Descendants
  are not listed; they travel with their parent's subtree;
- classify each top-level child bullet as **movable** or **droppable**. A bullet is
  droppable when `pomodoroBulletBodyBounds(rawLine)` is `null` (an empty placeholder
  bullet such as `\t- `) **and** its captured subtree is exactly one line. Droppable
  bullets are deleted with the entry rather than pushed into the destination, so a
  destination never inherits blank filler. Everything else is movable;
- expose `targets` (movable), `droppedLines` (droppable), `bulletCount`
  (`targets.length`), and `entry`/`entries`/`context` for the picker.

Zero movable bullets is **valid**, not an error: it is the "delete this empty Pomodoro"
case (see below).

### Move and delete

Extend `planPomodoroBulletMove` with `options.scope`, defaulting to `"bullets"`. Every
change is guarded by `scope === "entry"`, so the sub-bullet planner's behavior and
frozen result shape stay byte-identical and its existing tests keep passing unedited.

In `"entry"` scope:

1. Require `destination.kind === "existing"` and a destination other than the source;
   `"new"` is not reachable here because a typed name means rename, not create. Reject
   any other kind with the existing `"Choose a Pomodoro destination"` style error.
2. Allow `targets.length === 0` instead of returning
   `"No Pomodoro bullets were selected"`.
3. Add a source staleness guard alongside the per-target one: when
   `options.sourceRawLine` is provided, require
   `originalLines[sourceEntryLine] === options.sourceRawLine`, else
   `"The Pomodoro entry changed before it could be moved"`.
4. Capture and remove target subtrees exactly as today.
5. Replace the "is the remaining source child block blank?" test with an unconditional
   source deletion, plus a **no-silent-loss guard**: every non-blank line in the source
   entry's original child block must be covered by a captured target subtree or be one
   of the discovery's dropped placeholder lines. If any other non-blank owned line
   survives — an indented continuation, a stray note, a nested list under no bullet —
   return `invalid("Pomodoro has content that cannot be moved; nothing was moved")`
   rather than deleting it. In practice this guard never fires on real ledgers; it
   exists so the force-delete can never destroy unmoved content.
6. Delete the source with the same `removePomodoroBulletRanges` seam handling already
   used for the emptied-source case, over
   `[entry.entryLine, findCurrentBulletChildBlock(afterRemovalLines, entry.entryLine).endLineExclusive)`.
   For a childless entry this collapses to the single entry line.
7. Re-resolve the destination against the repaired content with the existing `shiftLine`
   rule. It stays correct: `removePomodoroBulletRanges` splices forward from
   `startLine`, so no line before the source entry can move, and entries after it shift
   by the net `lineDelta`.
8. Reuse the existing insertion path verbatim — append after the destination's last
   non-blank child, replace a destination's lone empty placeholder, merge
   exact-duplicate single-line blocks, preserve source order and descendant relative
   indentation, rebase onto the destination's `childIndent`.
9. Report `sourcePomodoroDeleted: true`, `createdPomodoro: false`,
   `createdPomodoroName: null`, plus the usual `movedCount`, `skippedDuplicateCount`,
   `destinationEntryLine`, and `firstMovedLine`. When nothing was inserted (zero
   targets, or every block merged away), `firstMovedLine` must fall back to
   `destinationEntryLine`, as it already does, so the commit path's cursor clamp stays
   valid.

Preserve the `## Pomodoros` heading, neighboring entries, the following section, one
intended blank separator at the deletion seam, trailing-newline state, and CRLF vs LF.

### Rename

Add a separate pure planner `planPomodoroEntryRename(content, options)` taking
`{ sourceEntryLine, sourceRawLine, name }` and returning a frozen
`{ valid, error, after, entryLine, name, previousName, unchanged }`:

- validate `name` through `normalizePomodoroName` and surface its error verbatim (empty,
  or longer than `POMODORO_NAME_MAX_LENGTH`);
- guard staleness against `sourceRawLine`;
- parse the entry with `parsePomodoroEntryLine`; a line it rejects is
  `"Source Pomodoro entry could not be found"`;
- when the entry already has a name (`parsed.name !== null`), the whole tail after
  `parsed.rangeEnd` is that name, so rewrite the line as
  `line.slice(0, parsed.rangeEnd) + " — " + NAME`;
- when the entry has no name and the tail after `parsed.rangeEnd` is blank, produce the
  same line;
- when the entry has no name but the tail is non-blank and does not start with the em
  dash separator — unsupported trailing content the grammar cannot round-trip — return
  `invalid("Pomodoro entry has unsupported trailing content; rename it by hand")`. Do
  not guess where the name belongs;
- when the normalized name equals the entry's existing normalized name, return
  `valid: true, unchanged: true` with `after` equal to the input, and let the commit
  path skip the write and report it;
- never touch the checkbox status, the parenthetical body, or any other line. Rename
  works on open and closed entries alike; only the name suffix changes.

### Picker

Reuse the existing picker machinery with an explicit mode.

`createPomodoroBulletMovePickerRows(entries, sourceEntryLine, rawQuery, options)` gains
a fourth `options` argument with `mode: "bullets" | "entry"`, defaulting to `"bullets"`
(existing three-argument calls and tests keep working):

- destination rows stay identical: open entries only, source excluded, same
  title/meta/status labels and same query matching;
- the typed-name row in `"entry"` mode becomes `{ kind: "rename", name, title: \`Rename
  to ${name}\`, meta: "Renames the current Pomodoro", badge: "Rename" }`instead of`kind:
  "new"`. Use a distinct kind rather than overloading `"new"`, so the commit path
  switches on intent;
- in `"entry"` mode build the "already exists" name set from open entries **excluding
  the source**, so typing the current Pomodoro's own name still yields a visible
  `rename` row (which commits as the reported no-op) instead of an empty result list. In
  `"bullets"` mode keep the existing set unchanged;
- typing a name that matches another open Pomodoro still suppresses the typed row and
  shows that Pomodoro's destination row, so "type an existing name" means move-into-it
  and "type a novel name" means rename — the exact rule the user asked for;
- the `invalid` row for a name that fails normalization is unchanged.

`renderPomodoroBulletMovePickerRow` handles `kind === "rename"` like `"new"` but with a
`pencil` icon and the row's `badge` text (default `"New"` when absent) in the
`is-create` badge, so the bullets mode renders exactly as before. Add
`.bob-cnp-pomodoro-row.is-rename` next to the existing `.is-new` accent-color rule in
`plugins/bob-navigation-hotkeys/styles.css:195`.

Add `PomodoroEntryMovePickerModal`, a sibling of `PomodoroBulletMovePickerModal`, with:

- title `Move or rename Pomodoro`, header icon `timer`;
- input label `Filter Pomodoro destinations`, placeholder
  `Filter open Pomodoros or type a new name`, results label `Pomodoro destinations`;
- empty text `Type a name to rename this Pomodoro`;
- subtitle `Pomodoro #<position> (<N> bullets) · <M> destinations`, plus the
  ignored-count suffix described above;
- `getFilteredItems` calling the row builder in `"entry"` mode;
- `openItem` rejecting `invalid` rows and otherwise delegating to
  `plugin.commitPomodoroEntryMoveSession(session, row)`;
- the same `onClose` release of `plugin.activeTaskMoveDestinationPicker`.

### Session and commit

Add `openPomodoroEntryMovePicker(editor, view, options)`, modeled on
`openPomodoroBulletMovePicker` (21670): same `activeTaskMoveDestinationPicker` supersede
guard, same Markdown-file and cursor preconditions (notice
`Open a Markdown Pomodoro note before moving bullets` stays appropriate; use
`Place the cursor on a Pomodoro entry` for a missing cursor), same scroll capture, and a
frozen session carrying `sourceFile`, `sourcePath`, `sourceView`, `editor`,
`sourceContent`, `cursor`, `scroll`, `countExplicit`, `discovery`, `entries`, and
`sourceEntry`.

Add `commitPomodoroEntryMoveSession(session, row)`, modeled on
`commitPomodoroBulletMoveSession` (21997):

- reuse the identical staleness gate — active view, same path, same editor instance, and
  `editor.getValue() === session.sourceContent` — with the notice
  `Source note is no longer active; nothing was moved`;
- `row.kind === "existing"` →
  `planPomodoroBulletMove(session.sourceContent, { scope: "entry", targets: session.discovery.targets, sourceEntryLine, sourceRawLine, destination: { kind: "existing", entryLine: row.entry.entryLine } })`,
  destination label from `getPomodoroBulletMoveDestinationLabel(row.entry)`;
- `row.kind === "rename"` → `planPomodoroEntryRename(...)`; when `unchanged`, emit
  `Pomodoro #<position> is already named <NAME>` and return `false` without writing;
- any other row kind → `false`;
- an invalid plan emits `` `${plan.error}; nothing was moved` `` and returns `false`;
- apply exactly one `applyEditorContentTransaction`, i.e. one undo step, and verify
  `editor.getValue() === plan.after` afterwards, with the failure notice
  `Pomodoro entry move failed; nothing was moved` (rename:
  `Pomodoro rename failed; nothing was changed`);
- cursor: for a move, `firstMovedLine` with `ch` clamped to that line's length, exactly
  as the sub-bullet commit does; for a rename, stay on the entry line with `ch` clamped;
- call `restoreTaskMoveSourceContext(session)` and emit the notice.

Notices, via a small `buildPomodoroEntryMoveNotice(plan, discovery, destinationLabel)`
next to the existing builder:

- `Moved 3 bullets from Pomodoro #2 to DEEP WORK`
- with ` (merged 1 duplicate)` appended when `skippedDuplicateCount > 0`, reusing the
  existing pluralization wording;
- `Deleted empty Pomodoro #2` when nothing moved and nothing merged;
- rename: `Renamed Pomodoro #2 to DEEP WORK`.

### The empty-source case

Invoking this on a Pomodoro with no movable sub-bullets and choosing a destination
deletes the entry and moves nothing. That is the natural reading of "delete the current
Pomodoro and move all of its sub-bullets" when the set of sub-bullets is empty, it is an
explicit user action, it is announced by its own notice, and it is one undo step. Keep
it.

## Implementation order

1. Pure engine in `plugins/bob-navigation-hotkeys/main.js`: `findPomodoroEntryContext`,
   `discoverPomodoroEntryMoveTargets`, the `scope` option on `planPomodoroBulletMove`,
   `planPomodoroEntryRename`, `buildPomodoroEntryMoveNotice`, and the `options.mode`
   argument on `createPomodoroBulletMovePickerRows`. Export every new helper from the
   test-surface block near line 24189.
2. UI: `renderPomodoroBulletMovePickerRow`'s `rename` kind, the `styles.css` rule, and
   `PomodoroEntryMovePickerModal`.
3. Wiring: the routing branch in `openTaskMoveOrPomodoroBulletPicker`,
   `openPomodoroEntryMovePicker`, and `commitPomodoroEntryMoveSession`.
4. Release: bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.28.1` to
   `1.29.0` (new feature) and update the `README.md` version cell plus the
   `Bob Navigation Hotkeys` description so it states the entry-line behavior alongside
   the existing sub-bullet sentence. Add the new coverage areas to the README's
   test-coverage paragraph.

## Tests

Extend `scripts/test-navigation-hotkeys.cjs`. Reuse `pomodoroFixtureLines()` (8525) and
`createPomodoroMovePickerHarness()` (3773); add fixtures only for shapes the existing
one lacks. All 277 existing tests must stay green **without edits** — that is the
regression signal that `scope: "bullets"` is untouched.

Routing and discovery:

1. `findPomodoroEntryContext` resolves each entry line to the right entry and returns
   `null` for sub-bullets, the `## Pomodoros` heading, blank lines, lines outside the
   section, and a column-0 checkbox that is not a valid entry.
2. `openTaskMoveOrPomodoroBulletPicker` dispatches entry lines to
   `openPomodoroEntryMovePicker` while sub-bullet and `#task` dispatch stay unchanged
   (extend the existing three-way dispatcher test at 5320).
3. `discoverPomodoroEntryMoveTargets` collects only top-level child bullets in source
   order, excludes descendants from `targets`, drops a lone `\t- ` placeholder into
   `droppedLines`, returns `valid: true` with zero targets for a childless entry, and
   rejects a non-entry line.

Move planner (`scope: "entry"`):

4. Moving a multi-bullet entry into a later open Pomodoro appends all bullets in source
   order after the destination's last child, deletes the source, and reports
   `sourcePomodoroDeleted: true` with `createdPomodoro: false`.
5. The same move into an **earlier** destination computes `destinationEntryLine` and
   `firstMovedLine` correctly after the source's removal.
6. A bullet with descendants carries its subtree and relative indentation, rebased onto
   the destination's `childIndent`.
7. Exact-duplicate single-line bullets merge away and still delete the source; a
   near-match stays distinct; a destination's lone empty placeholder is replaced, and is
   left intact when everything merged away.
8. A childless entry and a placeholder-only entry both plan a pure deletion:
   `movedCount: 0`, source deleted, destination byte-identical,
   `firstMovedLine === destinationEntryLine`.
9. The no-silent-loss guard: an entry whose child block holds a non-blank line that no
   target subtree covers returns `valid: false` and `after` equal to the input.
10. Stale `sourceRawLine`, a stale target `rawLine`, a destination equal to the source,
    an unresolvable destination, and `destination.kind === "new"` each return the right
    error and change nothing.
11. Deleting the first, a middle, and the last entry preserves the following `##`
    boundary and one blank separator, for both LF and CRLF input and with and without a
    trailing newline.
12. Closed (`[x]`) and cancelled (`[-]`) source entries follow the same rule as open
    ones.

Rename planner:

13. Renames a named entry, names an unnamed placeholder entry, and names an unnamed
    timespan entry, each producing exactly `... ) — NAME` and leaving status, body, and
    all other lines untouched.
14. Uppercases, collapses whitespace, and strips em dashes from the typed name; rejects
    an empty name and one over `POMODORO_NAME_MAX_LENGTH` with `normalizePomodoroName`'s
    errors.
15. Reports `unchanged: true` when the normalized name matches the current name.
16. Rejects an entry with unsupported non-name trailing content, and rejects a stale
    `sourceRawLine`.
17. Renames closed entries the same way as open ones.

Picker rows:

18. `"entry"` mode emits a `rename` row for a novel name with the rename
    title/meta/badge, emits a destination row (and no `rename` row) for a name matching
    another open Pomodoro, and still emits a `rename` row when the typed name equals the
    source's own name.
19. `"bullets"` mode output is unchanged, including the three-argument call form.
20. Both modes exclude the source entry and closed entries from destination rows and
    share the `invalid` row for an unusable name.

Commit path:

21. `commitPomodoroEntryMoveSession` on an `existing` row applies one transaction and
    one undo group, matches the planner's `after`, clamps the cursor to
    `firstMovedLine`, and emits `Moved N bullets from Pomodoro #P to LABEL`.
22. The same on a `rename` row applies one transaction, keeps the cursor on the entry
    line, and emits `Renamed Pomodoro #P to NAME`; the `unchanged` case writes nothing,
    emits the already-named notice, and returns `false`.
23. The empty-source move emits `Deleted empty Pomodoro #P`.
24. A session whose editor content drifted, or whose active view changed, is rejected
    with the existing notice and writes nothing.

## Verification and deployment

```bash
node --test scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
```

Confirm the focused suite is green and strictly above the 277-test baseline with no
existing test edited. Review the final diff to confirm only the navigation-hotkeys
plugin, its styles, its tests, and release metadata changed.

Then preview and perform the targeted sync from the linked checkout, using the real path
printed by `/sase_repo`:

```bash
bob plugins sync --no-pull --dry-run -p bob-navigation-hotkeys -r "<bob-plugins checkout path>"
bob plugins sync --no-pull -p bob-navigation-hotkeys -r "<bob-plugins checkout path>"
```

Do not use `--force` to bypass a dirty deployed copy. Confirm the deployed `main.js` and
`manifest.json` match source at `1.29.0`, and finish with clean or fully accounted-for
`git status` output in both the linked repo and the primary workspace.

## Memory follow-up

The `bob-cli` Pomodoro glossary strand (`sase/memory/glossary/pomodoro.md`) currently
documents only the sub-bullet form: "On a sub-bullet, `N<ctrl+shift+m>` moves it and the
next N siblings into another open Pomodoro, or into a new named one below." Shipping
this change makes that description incomplete.

Do **not** edit that memory file from this plan — plan-file authorization does not count
as user approval for SASE memory edits. Instead, after the code ships, use
`/sase_new_task` to file a `task(memory)` bead proposing the glossary sentence that adds
the entry-line behavior, so the project owner can approve it.

## Out of scope

- Making closed or cancelled Pomodoros selectable **destinations**; the picker keeps
  listing open entries only.
- Any meaning for `N<Ctrl+Shift+M>` on an entry line beyond "ignored and reported",
  including a future "merge this Pomodoro and the next N-1 into the destination".
- Moving Pomodoro entries across notes, reordering entries within `## Pomodoros`, or
  changing the named-Pomodoro grammar, `normalizePomodoroName` rules, or
  `POMODORO_NAME_MAX_LENGTH`.
- The existing sub-bullet route, its count clamping, its create-new-Pomodoro branch, and
  ordinary cross-note `#task` moves.
- `bob-ledger-tools` and `task-status-cycler` Pomodoro behavior, and any `bob-cli`
  Rust-side Pomodoro logic.
