---
tier: tale
title: Reorder planned Pomodoros with Ctrl+Shift+J/K
goal:
  With the cursor on an open Pomodoro entry that has no time range yet, Ctrl+Shift+J/K
  moves that Pomodoro (and everything it owns) down/up among the other planned Pomodoros
  instead of jumping, never crossing a current or closed/cancelled Pomodoro; every other
  cursor position keeps the exact navigation behavior it has today.
size: medium
proposed_by: bbugyi200.athena.0ff
create_time: 2026-09-09 20:00:20
status: wip
---

# Plan

## Repository

All code changes land in the **`bob-plugins`** linked repo, in the
`bob-navigation-hotkeys` plugin. No `bob-cli` source or docs reference these keymaps, so
`bob-cli` needs no changes.

Open the repo first with the `/sase_repo` skill:

```bash
sase repo open bob-plugins -r "Implement Ctrl+Shift+J/K planned-Pomodoro reordering"
```

Use the path it prints for every read and write below. Paths in this plan are relative
to that checkout root.

> Practical note while working in this repo: `grep` on
> `plugins/bob-navigation-hotkeys/main.js` can silently produce no output under some
> locales because of the emoji/em-dash content. Run greps with `LC_ALL=C` (e.g.
> `LC_ALL=C grep -n ...`) or pass `-a`.

## Background: what exists today

### The keymap

`Ctrl+Shift+J` / `Ctrl+Shift+K` are bound in the vault's `hotkeys.json` to the
`bob-navigation-hotkeys` commands `jump-to-next-open-task` and `jump-to-prev-open-task`.
Both call `NavigationHotkeysPlugin.jumpToOpenObsidianTask(editor, direction)` with
`direction` of `1` / `-1`.

A capture-phase keydown fallback, `handleOpenTaskJumpPhysicalKeydown` →
`getOpenTaskJumpKeydownDirection`, calls the _same_ method so the chord still works in
Vim normal mode (CodeMirror Vim swallows it before Obsidian's dispatcher). Because both
routes funnel through one method, a single physical press can reach it twice in one
dispatch turn; `isOpenTaskJumpDispatchPending` / `markOpenTaskJumpDispatch` deduplicate
that per `(editor, direction)` and clear on the next macrotask.

`jumpToOpenObsidianTask` currently:

1. reads the cursor, bailing with `"No active markdown editor"`,
2. returns early if the dispatch guard is already pending, then marks it,
3. asks `getOpenObsidianTaskJumpLine(lines, cursor.line, direction)` for a target,
4. on `null`, shows `"No next open task"` / `"No previous open task"`,
5. otherwise moves the cursor and calls `scheduleOpenTaskJumpCenter`.

Jump targets come from `getOpenTaskNavigationLines`: Ready/In Progress/Next `#task`
lines anywhere, plus open-or-done top-level Pomodoro ledger lines inside a
`## Pomodoros` section (`isPomodoroNavigationTaskLine`). Jumps wrap around the file.

### Pomodoro ledger vocabulary (project glossary)

A Pomodoro is a column-0 checkbox item under `## Pomodoros` in a daily note. Its body is
either a timespan such as `(**0920-0950** [t:: 30m])` or an empty `()` placeholder,
optionally followed by ` — NAME`. **Closed/past** Pomodoros carry a timespan and a
closed checkbox; the **current** Pomodoro is open with a timespan; **planned** Pomodoros
are open placeholders. Child bullets log the session's work.

### Reusable machinery already in `main.js`

These already exist and should be reused rather than reimplemented:

- `parsePomodoroEntryLine(lineText)` — parses a column-0 `- [c] (...)[ — NAME]` ledger
  entry; returns `null` for anything else (indented lines, non-`-` markers, bodies that
  are neither `()` nor a leading time range). Exposes `status`, `open` (checkbox not in
  `POMODORO_LEDGER_CLOSED_STATUSES` = `{"x","X","-"}`), and `placeholder` (body starts
  with an empty `()`).
- `collectPomodoroEntries(content)` — returns `{ section, entries }` for the note's
  **first** `## Pomodoros` section. Each entry is frozen with `index`, `position`
  (1-based), `entryLine`, `status`, `open`, `name`, `rangeText`, `placeholder`,
  `childStartLine`, `childEndLineExclusive` (clamped to the section end), `childIndent`,
  `bulletLines`, `previewText`, `moreCount`. Fenced lines are skipped.
- `findPomodoroEntryContext(content, line)` —
  `{ entries, section, entry, entryIndex, entryLine }` when `line` is exactly an entry
  line, else `null`.
- `findCurrentBulletChildBlock(lines, parentLine)` — the block a bullet owns: later
  lines that are blank or indented deeper, stopping at the first non-blank line at or
  shallower than the parent. Interior blank lines are included; trailing blank lines are
  excluded. (`collectPomodoroEntries` already applied this and stored the result as
  `childStartLine`/`childEndLineExclusive`.)
- `splitMarkdownContent(text)` → `{ lines, lineEnding }` — use it so CRLF notes
  round-trip.
- `getNormalizedPomodoroEntryName(entry)` and
  `getPomodoroBulletMoveDestinationLabel(entry)` → the entry's normalized name or
  `Pomodoro #<position>`.
- `applyEditorContentTransaction(cm, originalContent, nextContent, finalCursor)` — one
  undo-grouped write. When line counts match (always true for a pure reorder) it diffs
  line-by-line and rewrites only changed lines.
- `normalizePosition(position)`.

`Ctrl+Shift+M` already has a whole-entry route (`openPomodoroEntryMovePicker` /
`commitPomodoroEntryMoveSession`) built on the same helpers; mirror its structure and
its notice tone.

## Desired behavior

When `Ctrl+Shift+J` / `Ctrl+Shift+K` fires and the cursor sits on a **movable Pomodoro
entry line** — a column-0 ledger entry inside the note's `## Pomodoros` section that is
`open` **and** `placeholder` (no time range yet) — the keymap reorders that entry
instead of jumping:

- `Ctrl+Shift+J` (`direction > 0`) moves it one position **down**.
- `Ctrl+Shift+K` (`direction < 0`) moves it one position **up**.
- The entry line moves together with every line it owns
  (`childStartLine`..`childEndLineExclusive - 1`), i.e. all sub-bullets and their
  descendants.

### The ordering invariant

A planned Pomodoro must never end up above a current or closed/cancelled one. Enforce
this with a single, symmetric rule:

> A move is only allowed when the **adjacent Pomodoro entry in that direction is itself
> movable** (open **and** placeholder). Otherwise the move is refused.

Consequences:

- Moving up is refused when the entry above is the current Pomodoro, a closed
  `[x]`/`[X]` Pomodoro, a cancelled `[-]` Pomodoro, or when the entry is already the
  section's first entry.
- A planned Pomodoro can therefore only be reordered inside the contiguous run of
  planned Pomodoros it belongs to. It can never jump over a non-planned entry in either
  direction, which is strictly stronger than the stated requirement and keeps the
  operation predictable.
- Swapping two adjacent blocks _is_ "move by one position", so this handles blocks of
  different sizes correctly with no extra logic.

Anything strictly between the two blocks (blank lines, stray prose) stays where it is;
only the two owned blocks are permuted. A blank separator between the two Pomodoros
therefore remains a blank separator between them.

### Refusal

When the cursor is on a movable Pomodoro entry but the move is refused, show a notice
and **do not fall through to the jump** — the cursor must not move. This is the whole
point: on a planned Pomodoro, the chord is a move chord.

### Every other case is unchanged

The jump path must be reached, byte-for-byte as today, when the cursor is on:

- a Pomodoro entry with a time range (current or closed) — `placeholder` is false;
- a cancelled `[-] ()` entry — `open` is false;
- a Pomodoro sub-bullet, or any line that is not an entry line;
- a `#task` line or any other line in the note;
- a note with no `## Pomodoros` section;
- an entry-shaped line inside a fenced code block (`collectPomodoroEntries` skips fenced
  lines);
- a `* [ ] ()` / `1. [ ] ()` line, or an entry in a _second_ `## Pomodoros` section.
  `parsePomodoroEntryLine` only accepts a column-0 `-` marker and
  `collectPomodoroEntries` only scans the first `## Pomodoros` section, while the
  navigation scanner (`getOpenTaskNavigationLines`) is broader on both counts. That
  asymmetry is intentional and matches `Ctrl+Shift+M`; these lines keep jumping exactly
  as before.

No count prefix (`N<Ctrl+Shift+J>`) support — the jump has none today, and none is being
added.

## Implementation

### 1. Pure planner: `planPomodoroEntryReorder(content, options)`

Add next to the other Pomodoro planners in `plugins/bob-navigation-hotkeys/main.js`
(near `planPomodoroEntryRename`).

Signature:
`planPomodoroEntryReorder(content, { sourceEntryLine, sourceRawLine, direction })`.

Return a frozen object, following the shape convention of `planPomodoroEntryRename`:

```js
{
  valid,            // boolean
  error,            // string | null
  after,            // string; === content when !valid
  entryLine,        // requested source entry line
  movedEntryLine,   // entry line after the reorder, or null
  entry,            // the source entry from collectPomodoroEntries, or null
  neighborEntry,    // the swapped-with entry, or null
  direction,        // normalized to -1 or 1
}
```

Behavior:

1. Normalize `direction` to `-1` (any negative) or `1`.
2. `findPomodoroEntryContext(content, sourceEntryLine)`; invalid with
   `"Place the cursor on a Pomodoro entry"` when `null`.
3. Guard staleness the way `planPomodoroEntryRename` does: when `options.sourceRawLine`
   is a string and does not equal the current line text, invalid with
   `"The Pomodoro entry changed before it could be moved"`.
4. Invalid with `"Only an open Pomodoro without a time range can be moved"` when
   `!entry.open || !entry.placeholder`. (Callers gate on this too; the planner stays
   defensive so it is safe to unit-test directly.)
5. Find the neighbor: `entries[entryIndex + direction]`. Invalid with
   `` `${label} is already the ${direction < 0 ? "first" : "last"} planned Pomodoro` ``
   when there is no neighbor, or when the neighbor is not `open && placeholder`. `label`
   is `getPomodoroBulletMoveDestinationLabel(entry)`.
6. Build `after`: take `splitMarkdownContent(content).lines`, treat the source block as
   `[entryLine, childEndLineExclusive)` and the neighbor block the same way, then splice
   the two blocks so they trade places while every line between and around them stays
   put. Join with the original `lineEnding`.
   - Note both blocks are already clamped to the section by `collectPomodoroEntries`,
     and consecutive entries cannot overlap, so the lower block always starts at or
     after the upper block's end.
7. `movedEntryLine`: the source entry line's new index.
   - Moving down: `entryLine + (neighborBlockLength)` where
     `neighborBlockLength = neighbor.childEndLineExclusive - neighbor.entryLine`.
   - Moving up: `neighbor.entryLine`.
   - Derive both from the actual splice rather than by hand if that reads more clearly;
     just assert it in tests.
8. Total line count is unchanged by construction — a useful internal invariant and
   something the tests should assert.

Export `planPomodoroEntryReorder` from `module.exports.helpers` (alongside
`planPomodoroEntryRename`). Also export `getPomodoroBulletMoveDestinationLabel` if the
tests need it — it is currently unexported.

### 2. Predicate: `isMovablePomodoroEntryContext(context)`

A tiny frozen-shape-free helper (or an inline check) that returns `true` when a context
from `findPomodoroEntryContext` has `context.entry.open && context.entry.placeholder`.
Export it so the routing rule is directly testable.

### 3. Plugin method: `movePlannedPomodoroEntry(editor, direction)`

Add to `NavigationHotkeysPlugin`, next to `jumpToOpenObsidianTask`.

Return a tri-state so the caller can tell "not my case" from "handled":

- `false` — the cursor is not on a movable Pomodoro entry; the caller should fall
  through to the jump.
- `true` — handled (either moved, or refused with a notice). The caller must return
  without jumping.

Body:

1. Require `editor` with a `getValue` function and a cursor; if either is missing,
   return `false` (let the jump path own its `"No active markdown editor"` notice).
2. `const sourceContent = String(editor.getValue() || "")`.
3. `const context = findPomodoroEntryContext(sourceContent, cursor.line)`; return
   `false` when `null` or when the entry is not `open && placeholder`.
4. `const plan = planPomodoroEntryReorder(sourceContent, { sourceEntryLine: cursor.line, sourceRawLine: <current line text>, direction })`.
5. When `!plan.valid`: `new Notice(plan.error)` and return `true`.
6. Compute
   `finalCursor = { line: plan.movedEntryLine, ch: Math.min(cursor.ch, afterLines[plan.movedEntryLine].length) }`
   so the cursor rides along with the Pomodoro, exactly like
   `commitPomodoroEntryMoveSession` does.
7. Apply with
   `applyEditorContentTransaction(editor, sourceContent, plan.after, finalCursor)`.
   Mirror `commitPomodoroEntryMoveSession`'s verification: wrap in `try/catch`, then
   confirm `String(editor.getValue() || "") === plan.after`; on failure show
   `"Pomodoro move failed; nothing was moved"` and return `true`.
8. On success: ``new Notice(`Moved ${label} ${direction < 0 ? "up" : "down"}`)`` with
   `label = getPomodoroBulletMoveDestinationLabel(plan.entry)` — e.g.
   `Moved VERIFY down`, `Moved Pomodoro #3 up`. Return `true`.

Do **not** call `scheduleOpenTaskJumpCenter` here. The reorder is an in-place edit;
`applyEditorContentTransaction` sets the selection in the same transaction and
CodeMirror scrolls the cursor into view on its own. Forcing a `zz`-style recenter on
every move would be jarring.

### 4. Route from `jumpToOpenObsidianTask`

Insert the branch **after** the existing dispatch-guard block and before
`getOpenObsidianTaskJumpLine`:

```js
this.markOpenTaskJumpDispatch(editor, direction);

if (this.movePlannedPomodoroEntry(editor, direction)) {
  return true;
}

const targetLine = getOpenObsidianTaskJumpLine(...);
```

Placement matters:

- **After the guard** so the Obsidian-hotkey route and the Vim capture-phase route
  cannot both fire for one physical press and move the Pomodoro two positions.
- **Before the jump lookup** so a movable Pomodoro never falls through.
- Inside `jumpToOpenObsidianTask` (not in the keydown handler) so both routes are
  covered by one change. `handleOpenTaskJumpPhysicalKeydown` and
  `getOpenTaskJumpKeydownDirection` need no edits — they already `preventDefault` once
  the chord matches.

Update the comment above the dispatch guard so it describes both outcomes (a jump _or_ a
move).

### 5. Command names

Keep the command **ids** `jump-to-next-open-task` and `jump-to-prev-open-task` unchanged
— the vault's `hotkeys.json` binds by id and renaming them would break the user's
bindings. Update only the human-readable `name` fields so the command palette reflects
the dual behavior, e.g.:

- `"Jump to next open task or move a planned Pomodoro down"`
- `"Jump to previous open task or move a planned Pomodoro up"`

## Tests

All in `scripts/test-navigation-hotkeys.cjs`, run with `npm test` from the `bob-plugins`
repo root. Follow the file's existing style: pure-helper tests use `helpers.*`,
plugin-level tests use `new NavigationHotkeysPlugin()` with `TransactionEditor` and the
module-level `notices` array. `pomodoroFixtureLines()` is a ready-made fixture with a
closed entry, a current `(**0920-0950**)` entry, an unnamed placeholder with a nested
child, and `— BODY` / `— VERIFY` named placeholders.

Planner tests (`planPomodoroEntryReorder`):

1. Moving a named placeholder down swaps it with the next placeholder, carries its child
   bullets, leaves total line count unchanged, and reports the expected
   `movedEntryLine`.
2. Moving a placeholder up swaps it with the previous placeholder.
3. Blocks of unequal size (one entry with two nested levels, the neighbor with none)
   round-trip: down-then-up restores the original content exactly.
4. Moving up is refused when the entry above is the **current** open entry with a time
   range — error mentions "first planned Pomodoro", `after === content`.
5. Moving up is refused when the entry above is **closed** `[x]`, and when it is
   **cancelled** `[-]`.
6. Moving down is refused for the last placeholder in the section.
7. A lone placeholder in the section is refused in both directions.
8. A blank line between two placeholders stays between them after the swap.
9. Refused for a non-placeholder entry (has a time range) and for a cancelled
   placeholder — `"Only an open Pomodoro without a time range can be moved"`.
10. Refused when `sourceRawLine` does not match the live line.
11. CRLF content round-trips with `\r\n` preserved.
12. Content following the `## Pomodoros` section (the fixture's `## Not Pomodoros`) is
    untouched.

Routing / plugin tests (`jumpToOpenObsidianTask`):

13. Cursor on a placeholder entry, `direction: 1` → exactly one transaction, one undo
    group, resulting content equals `helpers.planPomodoroEntryReorder(...).after`,
    cursor lands on `movedEntryLine` with `ch` clamped, and the last notice is
    `Moved VERIFY down`-style.
14. Cursor on a placeholder entry with the move refused → **no** transaction, cursor
    unchanged, notice is the planner's refusal text, and the method returns `true`.
15. Cursor on the current `(**0920-0950**)` entry → no transaction; the cursor lands on
    the same line `getOpenObsidianTaskJumpLine` returns for that cursor (regression
    guard for "unchanged in every other case").
16. Cursor on a Pomodoro sub-bullet and on a plain `#task` line → jump behavior
    unchanged, same assertion style as (15).
17. Dispatch-guard regression: two calls with the same `(editor, direction)` in one
    macrotask perform exactly **one** move.

Harness note for tests 13–17: `markOpenTaskJumpDispatch` calls `this.register(...)` to
schedule guard cleanup, and the stub `Plugin` base class in the test file has no such
method — stub `plugin.register = () => {}` (and `plugin.app = {}`) the way other
plugin-level tests stub lifecycle hooks. The jump path's `scheduleOpenTaskJumpCenter` is
deferred to the next frame and is a no-op on a `TransactionEditor`, so tests 15–16 can
assert on `editor.setCursorCalls` / `editor.cursor` synchronously without stubbing it.

Also keep the existing `"open-task navigation keeps existing Pomodoro ledger targets"`
and `"next and previous jumps skip Blocked tasks and wrap to an allowed task"` tests
passing untouched — they are the contract that navigation itself did not change.

Run:

```bash
npm test
npm run validate
```

## Docs and release

1. Bump `plugins/bob-navigation-hotkeys/manifest.json` `version` to `1.30.0` (new
   feature, no breaking change). Leave the manifest `description` alone unless it is
   already inaccurate.
2. Update `README.md`:
   - the `Version` cell for `bob-navigation-hotkeys` in the plugin table (`1.29.0` →
     `1.30.0`);
   - the plugin's table description, extending the existing
     `` `Ctrl+Shift+J/K` navigates Ready, In Progress, and Next `#task` lines while skipping Blocked `[?]` tasks ``
     clause with the new behavior: on an open Pomodoro entry that has no time range yet,
     the same chord instead moves that Pomodoro and everything it owns down/up one
     position among the adjacent planned Pomodoros, refusing (with a notice, without
     jumping) to cross a current, closed, or cancelled Pomodoro.
3. `npm run validate` must pass (it checks manifest fields and semver, and syntax-checks
   `main.js`).
4. Per the `bob-plugins` repo instructions, deploy to the vault after the change lands.
   From a SASE workspace this needs an explicit repo argument:

   ```bash
   bob plugins sync -r "<path printed by sase repo open>"
   ```

## Acceptance criteria

- On a `- [ ] ()` or `- [ ] () — NAME` Pomodoro entry inside `## Pomodoros`,
  `Ctrl+Shift+J`/`Ctrl+Shift+K` reorders that entry, with all of its sub-bullets, one
  position down/up among adjacent planned Pomodoros.
- A planned Pomodoro can never be moved above a current, closed, or cancelled Pomodoro;
  the refusal shows a notice and leaves the cursor and the note untouched.
- Every other cursor position produces the same jump target, wrap-around, notice text,
  and `zz`-style centering as before.
- One physical key press performs at most one move (Obsidian hotkey route and Vim
  normal-mode capture route both covered, no double-apply).
- The reorder is a single undo group, preserves the note's line ending, and changes no
  line outside the two swapped blocks.
- `npm test` and `npm run validate` pass.
