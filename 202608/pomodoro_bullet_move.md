---
status: done
tier: epic
title: Pomodoro sub-bullet moves and named Pomodoros
goal: "`<ctrl+shift+m>` on a Pomodoro sub-bullet moves that bullet — and, with a count,
  the next N sibling bullets — into another open Pomodoro in the same note, or into a
  new named Pomodoro created just below the current one, and the Pomodoro glossary entry
  documents named Pomodoros and this keymap.

  "
phases:
  - id: pomodoro-move-engine
    title: Pure named-Pomodoro model, discovery, and planner
    depends_on: []
    size: medium
    description: "pomodoro-move-engine: add the named-Pomodoro grammar,
      entry/bullet-context model, sibling target discovery with count and clamping, and
      the pure same-file move planner to bob-navigation-hotkeys, all exported and unit
      tested with no UI or routing change.

      "
  - id: pomodoro-move-ui
    title: Picker, routing, and commit
    depends_on:
      - pomodoro-move-engine
    size: medium
    description: "pomodoro-move-ui: add the destination picker rows and modal with typed
      create-new-Pomodoro support, route `Ctrl+Shift+M` to the Pomodoro move when the
      cursor is on a sub-bullet, and commit the planned move as one guarded editor
      transaction with a reporting notice.

      "
  - id: pomodoro-move-docs
    title: Docs, version, sync, and glossary
    depends_on:
      - pomodoro-move-ui
    size: small
    description:
      "pomodoro-move-docs: bump the plugin manifest, extend the bob-plugins README,
      deploy with `bob plugins sync`, rewrite the Pomodoro glossary strand to cover
      named Pomodoros and the keymap, and regenerate sase memory."
proposed_by: bbugyi200.athena.0eb
create_time: 2026-09-09 20:00:19
---

- **PROMPT:**
  [prompts/202608/pomodoro_bullet_move.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/pomodoro_bullet_move.md)

# Pomodoro sub-bullet moves and named Pomodoros (`<ctrl+shift+m>`)

## Goal

Teach the `<ctrl+shift+m>` keymap a second mode. Today it moves `#task` lines to another
note. When the cursor sits on a **Pomodoro sub-bullet** it must instead move that bullet
(and, with a count, the sibling bullets below it) into another **open Pomodoro in the
same file**, chosen from a picker that also accepts a typed name for a brand-new
Pomodoro created directly below the current one.

Alongside that, formalize **named Pomodoros**: a Pomodoro whose ledger body is followed
by an em dash and an ALL-CAPS name.

```
- [x] (**0855-0920** [t:: 25m])
	- ~~[[#^gtd]]~~
	- 🍅 [[dev#^lower-athena-disk-use]]
- [ ] (**0920-0950** [t:: 30m])
	- [[bob#^move-pomodoros]]
- [ ] ()
	- [[sase#^pager]]
- [ ] () — BODY
	- [[body#^email-jenika]]
- [ ] () — VERIFY
	- [[sase_better_config#^no-focus-xprompts]]
```

The em dash is U+2014 with one space on each side. `BODY` and `VERIFY` above are named
Pomodoros; the others are unnamed.

## Repositories

Two repositories are touched. Agents MUST open the linked repo through `/sase_repo`
before reading or writing it:

```bash
sase repo open bob-plugins -r "<why>"
```

- **`bob-plugins`** (linked repo) — all plugin code, tests, and plugin docs. Phases 1–3.
- **bob-cli** (this workspace) — only `sase/memory/glossary/pomodoro.md` plus the
  regenerated memory output. Phase 3.

`bob-plugins/AGENTS.md` requires `bob plugins sync` after changing files in that repo,
so the deployed vault copy under `~/bob/.obsidian/plugins/` matches source. Phase 3 owns
that single sync for the whole epic; Phases 1 and 2 must NOT sync.

## Why this shape

Everything lands in `plugins/bob-navigation-hotkeys/main.js`, which already owns:

- the `Ctrl+Shift+M` binding (`move-tasks-to-note`, `main.js:15860`) and its counted Vim
  path (`registerCountedTaskMoveInputListeners`, `main.js:18842`);
- the Pomodoro vocabulary it needs — `POMODOROS_HEADING_RE`, `isPomodorosHeading`,
  `hasPomodoroTimeRange`, `POMODORO_LEDGER_TOP_LEVEL_LINE_RE`,
  `isOpenPomodoroLedgerEntryLine`, `findPomodorosSectionRange`,
  `collectOpenPomodoroRanges`, `pomodoroBulletBodyBounds`,
  `collectPomodoroBlockLinkOccurrences`, `isDedicatedPomodoroLinkLine`
  (`main.js:6243`–`6440`);
- the subtree/rebase/transaction machinery — `captureTaskMoveSubtree`,
  `removeTaskMoveRanges`, `rebaseTaskMoveBlock`, `findCurrentBulletChildBlock`,
  `getBulletIndentWidth`, `applyEditorContentTransaction`;
- the picker chrome — `FilteredPickerModal` (`main.js:9094`) with its `bob-cnp-*`
  styling, keyboard navigation, and footer hints.

So this is one plugin, additive, with no new dependency between plugins.

The file's house style is a large body of **pure, exported helpers** near the top and a
thin `Plugin` class near the bottom that wires them to Obsidian.
`scripts/test-navigation-hotkeys.cjs` tests the pure helpers directly through
`NavigationHotkeysPlugin.helpers`. The epic is split along that existing seam: Phase 1
builds the pure engine and its tests, Phase 2 wires the UI onto it, Phase 3 documents
it.

## Design decisions (binding for all phases)

### Named Pomodoro grammar

A ledger entry is a column-0 `- [<status>] <body>` line inside `## Pomodoros`
(`POMODORO_LEDGER_TOP_LEVEL_LINE_RE`). Its body is a parenthetical — an empty `()`
placeholder or a time range — optionally followed by ` — NAME`.

The name suffix is parsed **only from the text after the closing `)` of that
parenthetical**, so an em dash inside `[t:: ...]` metadata is never mistaken for a name
separator.

Name normalization (`normalizePomodoroName`): strip every U+2014, collapse internal
whitespace runs to one space, trim, then `toUpperCase()`. The result is invalid when it
is empty or longer than `POMODORO_NAME_MAX_LENGTH = 48`. Stripping em dashes (rather
than rejecting them) means a user who types `— body` still gets `BODY`, and every
accepted name round-trips through the parser.

### Routing

`Ctrl+Shift+M` dispatches on the cursor line:

- `findPomodoroBulletContext(content, cursor.line)` non-null → **Pomodoro bullet move**.
- otherwise → the existing cross-note task move, unchanged.

This is unambiguous in practice: Pomodoro sub-bullets are indented under a column-0
ledger entry and are not `#task` checkboxes, so `Ctrl+Shift+M` there is an error today
("Move tasks must start on a real #task checkbox"). The one overlap — a real `#task`
line nested under a Pomodoro entry — resolves to the Pomodoro move, because the cursor
is literally on a Pomodoro sub-bullet. Document that in the README.

### Source and destination eligibility

- **Source**: a sub-bullet of _any_ Pomodoro entry, open or closed. Moving a link out of
  a finished session is a legitimate correction, and it changes no `[t:: ]` duration.
- **Destinations**: every _other_ **open** entry in the section (`open` per
  `isOpenPomodoroLedgerEntryLine`: any checkbox other than `x`, `X`, `-`), named or
  unnamed, plus the typed-name "create new" row.

### Count

`N<Ctrl+Shift+M>` reuses the existing convention verbatim: `getPendingVimRepeat` returns
the typed count, which is passed as `additionalTaskCount`, so `2<Ctrl+Shift+M>` moves
the bullet under the cursor **plus the 2 below it** — 3 bullets.

"The N below it" means the next N **siblings at the start bullet's indent depth**,
inside the same Pomodoro's child block. A selected bullet's own deeper descendants ride
along with it and are never counted as separate targets. Running out of siblings clamps
and is reported, mirroring `discoverMovableObsidianTaskTargets`.

The start bullet may itself be a nested grandchild (e.g. the `` `sase-tj` `` note under
a task link). It is then promoted to the destination's child depth, carrying its own
descendants. This is intentional and must be covered by a test.

### Insertion

- **Existing destination**: appended after the destination entry's last non-blank child
  line, in source order. If the destination's only child is an empty placeholder bullet
  (a list item with an empty body, e.g. `\t- `), the moved blocks replace it.
- **New destination**: a `- [ ] () — NAME` entry is inserted immediately after the
  **source** Pomodoro's block (after removal and any placeholder repair), with the moved
  blocks as its children.

Each moved block is rebased so its root sits at the destination's child indent and every
descendant keeps its relative depth — the same prefix-stripping approach as
`rebaseTaskMoveBlock`, but re-prefixed with the destination child indent instead of
column 0. The destination child indent is the indent of its first existing child bullet,
falling back to a single tab (Obsidian's default, and what
`EMPTY_POMODORO_SUB_BULLET_LINE` uses in `task-status-cycler`).

### Source repair

If the move empties the source Pomodoro's child block, insert `<sourceChildIndent>- ` so
the entry keeps a sub-bullet. This matches `buildPomodoroCompletionPlan` in
`task-status-cycler`, which writes `EMPTY_POMODORO_SUB_BULLET_LINE` for the same reason.

### Duplicate merging

A moved block that is a **single line** whose trimmed text **exactly equals** an
existing trimmed child line of the destination is dropped instead of inserted, and
counted in `skippedDuplicateCount`. Exact-text equality (not link-key equality) is
deliberate: it keeps `~~[[x]]~~`, `🍅 [[x]]`, and `[[x]]` distinct, so no state is ever
lost, and the notice reports the merge. Blocks with descendants are never merged away.

### Atomicity

The whole move is a same-file, content-to-content plan applied through one
`applyEditorContentTransaction` — a single undo step, and no cross-file rollback path is
needed.

## Phases

### Phase `pomodoro-move-engine` — pure named-Pomodoro model, discovery, and planner

**Size:** medium **Depends on:** nothing

Repo: `bob-plugins`. File: `plugins/bob-navigation-hotkeys/main.js` (helpers near the
existing Pomodoro helpers around `main.js:6243`–`6440`, exported from the `helpers`
object at the bottom of the file) and `scripts/test-navigation-hotkeys.cjs`.

No UI, no routing, no behavior change. Ship pure helpers plus tests only.

Add these constants:

- `POMODORO_NAME_SEPARATOR = "—"`
- `POMODORO_NAME_MAX_LENGTH = 48`
- a regex matching the optional ` — NAME` tail of a ledger body.

Add these helpers, all returning frozen objects:

1. **`parsePomodoroEntryLine(lineText)`** → `null`, or
   `{ indent, status, open, bodyStart, rangeText, rangeStart, rangeEnd, placeholder, name, nameStart, nameEnd }`.
   - `null` unless the line matches `POMODORO_LEDGER_TOP_LEVEL_LINE_RE` and its body
     carries an empty `()` placeholder (`POMODORO_PLACEHOLDER_RE`) or a time range
     (`hasPomodoroTimeRange`).
   - `open` is `false` for `x`, `X`, `-` (reuse `POMODORO_LEDGER_CLOSED_STATUSES`).
   - `name` is `null` when there is no em-dash suffix; otherwise the raw (not
     normalized) trimmed name, parsed only from text after the parenthetical's `)`.
2. **`normalizePomodoroName(raw)`** → `{ valid, name, error }` per the normalization
   rule above.
3. **`formatPomodoroEntryLine(name)`** → `"- [ ] ()"` when `name` is empty, otherwise
   `"- [ ] () — NAME"`.
4. **`collectPomodoroEntries(content)`** → `{ section, entries }`.
   - `section` from `findPomodorosSectionRange`; `null` (with `entries: []`) when the
     note has no `## Pomodoros` section.
   - One entry per parsed ledger line in the section, skipping fenced lines via
     `getMarkdownLineContexts`, each:
     `{ index, position, entryLine, status, open, name, rangeText, placeholder, childStartLine, childEndLineExclusive, childIndent, bulletLines, previewText, moreCount }`.
   - `position` is the 1-based ordinal of the entry within the section — the `#k` shown
     in the picker.
   - Child range from `findCurrentBulletChildBlock`, clamped to `section.endLine`.
   - `childIndent` is the indent of the first child list item, else `"\t"`.
   - `bulletLines` lists the child lines that are list items (`PROJECT_LIST_ITEM_RE`) at
     exactly `childIndent`'s display width; `previewText` is the first such line's
     trimmed body and `moreCount` the number of remaining ones — both feed the picker
     rows.
5. **`findPomodoroBulletContext(content, line)`** → `null`, or
   `{ entries, section, entry, entryIndex, line, depth }`.
   - Non-null only when `line` is a list item and
     `entry.childStartLine <= line < entry.childEndLineExclusive` for some entry.
   - `depth` is the line's indent display width, used by discovery.
6. **`discoverMovablePomodoroBulletTargets(content, startLine, additionalBulletCount)`**
   → the same frozen shape `discoverMovableObsidianTaskTargets` returns —
   `{ valid, error, explicit, startLine, requestedAdditionalCount, requestedCount, actualCount, clamped, targets }`
   — plus `entryLine` and `context`.
   - `targets` are `{ line, rawLine }`, matching the existing target shape so the
     staleness guard is identical.
   - Walk forward from `startLine` within the entry's child block collecting list items
     at exactly `depth`; skip deeper descendants; stop at `childEndLineExclusive`.
   - Errors: `"Place the cursor on a Pomodoro sub-bullet"` when context is `null`.
7. **`planPomodoroBulletMove(content, options)`** where `options` is
   `{ targets, sourceEntryLine, destination }` and `destination` is
   `{ kind: "existing", entryLine }` or `{ kind: "new", name }` (name already
   normalized). Returns
   `{ valid, error, after, destinationEntryLine, firstMovedLine, movedCount, skippedDuplicateCount, createdPomodoro, createdPomodoroName, sourcePlaceholderInserted }`.

   Algorithm, in order:
   1. Re-verify every target's `rawLine` still matches its line; on mismatch return
      `{ valid: false, error: "A selected bullet changed before it could be moved" }`.
   2. Capture each target's subtree: following lines whose indent display width exceeds
      the target's, bounded by the source entry's `childEndLineExclusive`. Retain blank
      lines only when deeper content follows (the `pendingBlankEnd` rule from
      `captureTaskMoveSubtree`).
   3. Remove the captured ranges (mirror `removeTaskMoveRanges`).
   4. If the source entry's child block is now free of non-blank lines, insert
      `<sourceChildIndent>- ` and set `sourcePlaceholderInserted`.
   5. Rebase each block onto the destination child indent (see _Insertion_ above).
   6. Drop exact-duplicate single-line blocks (see _Duplicate merging_), counting them.
   7. Insert: append into the existing destination's child block (replacing a lone empty
      placeholder bullet), or emit `formatPomodoroEntryLine(name)` right after the
      source entry's block and put the blocks beneath it.
   8. Compute `firstMovedLine` against the **resulting** content — the first inserted
      block's root line, or `destinationEntryLine` when every block was merged away.

   Reject with `valid: false` when the destination equals the source entry, when an
   `"existing"` destination line no longer parses as an entry, or when a `"new"` name is
   invalid.

**Tests** (`scripts/test-navigation-hotkeys.cjs`, `helpers`-driven, matching the file's
existing style). Build one shared realistic fixture modeled on the example at the top of
this plan — closed entries with ranges, open entries with ranges, open placeholders,
named entries, a nested grandchild bullet, and a trailing `##` heading.

- `parsePomodoroEntryLine`: named and unnamed; placeholder and both range styles; closed
  vs open status; an em dash inside `[t:: ]` metadata does **not** become a name;
  indented and non-ledger lines return `null`.
- `normalizePomodoroName`: lowercase → uppercase; whitespace collapse; leading `—`
  stripped; empty and over-length rejected with an error string.
- `formatPomodoroEntryLine`: with and without a name.
- `collectPomodoroEntries`: correct entries, `position`, `open`, `name`, child ranges,
  `childIndent`, `previewText`/`moreCount`; section ends at the next `##`; a
  ledger-shaped line inside a fence is skipped; a note with no section returns
  `{ section: null, entries: [] }`.
- `findPomodoroBulletContext`: entry line → `null`; sub-bullet → owning entry; nested
  grandchild → owning entry with the deeper `depth`; blank line → `null`; a line outside
  the section → `null`.
- `discoverMovablePomodoroBulletTargets`: bare count; `N` count; clamping at the entry's
  last bullet sets `clamped` and reports `requestedCount`; a subtree child is not
  counted as a sibling; starting on a grandchild collects grandchild siblings only.
- `planPomodoroBulletMove`: one bullet appended last in an existing open Pomodoro; three
  bullets with a nested child carried along; a move into a **new** named Pomodoro
  inserted directly below the source; source emptied → placeholder inserted; destination
  whose only child is an empty placeholder → replaced; exact-duplicate merged with
  `skippedDuplicateCount === 1` while a `🍅`-prefixed or `~~`-struck near-match is _not_
  merged; a stale `rawLine` → `valid: false`; `firstMovedLine` points at the first moved
  root; moving a bullet out and back restores the original content apart from
  placeholder repair.

**Done when:** `npm test` and `npm run validate` pass in `bob-plugins`, every new helper
is exported from `helpers`, and no existing behavior changed.

---

### Phase `pomodoro-move-ui` — picker, routing, and commit

**Size:** medium **Depends on:** `pomodoro-move-engine`

Repo: `bob-plugins`. Files: `plugins/bob-navigation-hotkeys/main.js`,
`plugins/bob-navigation-hotkeys/styles.css`, `scripts/test-navigation-hotkeys.cjs`.

**1. Picker row model (pure, exported, tested).**

`createPomodoroBulletMovePickerRows(entries, sourceEntryLine, rawQuery)` → an ordered
array of frozen rows:

- A **create row** `{ kind: "new", name, title, meta }` first, when the trimmed query is
  non-empty, normalizes to a valid name, and that name does not equal an existing open
  entry's normalized name (in which case the matching existing row is what the user
  wants, so no create row is shown).
- An **invalid row** `{ kind: "invalid", statusText }` instead, when the trimmed query
  is non-empty but normalization fails (empty after stripping, or over
  `POMODORO_NAME_MAX_LENGTH`). Not submittable.
- **Existing rows** `{ kind: "existing", entry, title, meta, statusEmoji, statusLabel }`
  for every open entry except `sourceEntryLine`, in file order, filtered by the query.

Row presentation:

| entry                | `title`                | `statusLabel`                | `meta`  |
| -------------------- | ---------------------- | ---------------------------- | ------- |
| named                | `NAME`                 | `0920-0950` or `Unscheduled` | preview |
| unnamed, ranged      | `0920-0950`            | `Unscheduled` omitted        | preview |
| unnamed, placeholder | `Pomodoro #<position>` | `Unscheduled`                | preview |

`meta` is the entry's `previewText`, suffixed with ` +<moreCount> more` when
`moreCount > 0`, or `No sub-bullets yet` when the entry has none. The create row's
`meta` is `Created below the current Pomodoro`.

Query matching is case-insensitive against the name, the range text, `#<position>` and
the bare position number, and the preview text.

**2. `PomodoroBulletMovePickerModal extends FilteredPickerModal`.**

Override `getFilteredItems()` to return `createPomodoroBulletMovePickerRows(...)` from
the live raw query — that is the clean seam for the synthetic create/invalid rows, and
it inherits the existing `bob-cnp` chrome, arrow/`^N`/`^P` navigation, scroll-into-view,
and footer hints (`KEYBOARD_HINTS`).

- `title`: `Move Pomodoro bullets`; `headerIcon`: `timer` (deliberately distinct from
  the note picker's `move-right`).
- `placeholder`: `Filter open Pomodoros or type a new name`.
- `emptyText`: `Type a name to create a new Pomodoro` — so a file whose only open
  Pomodoro is the source still opens usefully instead of erroring.
- `getSubtitle`: `<N> bullet(s) from Pomodoro #<k> · <M> destination(s)`, plus
  ` · requested <R>; reached end of Pomodoro` when `discovery.clamped`.
- `renderItem`: reuse the `bob-cnp-row-icon` / `-row-text` / `-row-title` / `-row-path`
  / `-row-badges` / `-row-status` structure and `appendHighlighted`, exactly as
  `renderTypedNotePickerRow` does. Create row icon `plus`, existing rows `timer`.
- `closeBeforeOpenItem: true`; `openItem` calls
  `plugin.commitPomodoroBulletMoveSession(session, row)` and returns `false` for the
  invalid row.
- `onClose` clears the shared active-picker slot.

`styles.css`: add only what the new rows need on top of the existing `bob-cnp-*` rules —
an `is-new` accent for the create row's icon and a status variant for `Unscheduled`.
Reuse existing custom properties; do not fork the picker's layout rules.

**3. Routing and session.**

- Rename nothing. Reuse the existing `this.activeTaskMoveDestinationPicker` field as the
  shared guard for **both** `Ctrl+Shift+M` pickers, so the two can never be open at
  once; its count-explicit upgrade rule (`openTaskMoveDestinationPicker`,
  `main.js:20729`) and `onClose` identity check work unchanged for the new modal. Add a
  comment saying it guards both.
- Add `openPomodoroBulletMovePicker(editor, view, options)` mirroring
  `openTaskMoveDestinationPicker`'s guards: active-picker/count-upgrade check, Markdown
  view check, cursor check, discovery, then open the modal with a frozen session
  `{ sourceFile, sourcePath, sourceView, editor, sourceContent, cursor, scroll, countExplicit, discovery, entries, sourceEntry }`.
- Add `openTaskMoveOrPomodoroBulletPicker(editor, view, options)` — the new single entry
  point that runs `findPomodoroBulletContext` on the cursor line and delegates to
  `openPomodoroBulletMovePicker` or the untouched `openTaskMoveDestinationPicker`. Point
  both callers at it: the `move-tasks-to-note` command's `editorCallback`
  (`main.js:15864`) and `handleCountedTaskMovePhysicalKeydown` (`main.js:18889`). The
  registered hotkey, command id, and command name stay exactly as they are.

**4. `commitPomodoroBulletMoveSession(session, row)`.**

- Guard first, mirroring `commitTaskMoveSession`: the active Markdown view is still the
  source file, `activeView.editor === session.editor`, and
  `session.editor.getValue() === session.sourceContent`. Otherwise notice
  `"Source note is no longer active; nothing was moved"` and return `false`.
- Build `destination` from the row (`{ kind: "existing", entryLine }` or
  `{ kind: "new", name }`) and call `planPomodoroBulletMove`. On `!plan.valid`, notice
  `` `${plan.error}; nothing was moved` `` and return `false`.
- Apply `plan.after` with a single `applyEditorContentTransaction` and a final cursor of
  `{ line: plan.firstMovedLine, ch: <clamped session.cursor.ch> }`. On failure, notice
  `"Pomodoro bullet move failed; nothing was moved"` and return `false`.
- Restore focus and scroll via the existing `restoreTaskMoveSourceContext(session)`.
- Success notice, built by an exported pure helper
  `buildPomodoroBulletMoveNotice(plan, discovery, destinationLabel)`:
  - `Moved 3 bullets to VERIFY`
  - `Moved 1 bullet to Pomodoro #6`
  - `Moved 3 bullets to new Pomodoro BODY`
  - append ` (merged 1 duplicate)` when `skippedDuplicateCount > 0`
  - append ` (requested 5; reached end of Pomodoro)` when `discovery.clamped`.

**Tests** (`scripts/test-navigation-hotkeys.cjs`):

- `createPomodoroBulletMovePickerRows`: source entry excluded; closed entries excluded;
  create row appears for a novel name and is suppressed on a name collision (including a
  case-insensitive collision); invalid row for empty-after-stripping and over-length
  queries; titles and `meta` for all four entry shapes in the table above; query
  matching by name, range, `#<position>`, bare position, and preview text; a file whose
  only open Pomodoro is the source yields only the create row.
- `buildPomodoroBulletMoveNotice`: singular/plural, new vs existing destination, merged
  duplicates, clamped, and both suffixes together.
- Routing: the existing registration test still asserts `move-tasks-to-note` is bound to
  `{ modifiers: ["Ctrl", "Shift"], key: "M" }`; add a test that
  `findPomodoroBulletContext` returns `null` for a plain `#task` line outside the
  Pomodoros section (so cross-note moves are untouched) and non-null for a sub-bullet.

**Done when:** `npm test` and `npm run validate` pass, cross-note `Ctrl+Shift+M`
behavior is byte-for-byte unchanged for `#task` lines, and the new picker path is fully
covered. Do **not** run `bob plugins sync` in this phase.

---

### Phase `pomodoro-move-docs` — docs, version, sync, and glossary

**Size:** small **Depends on:** `pomodoro-move-ui`

**1. `bob-plugins` docs and version.**

- `plugins/bob-navigation-hotkeys/manifest.json`: `1.27.1` → `1.28.0` (new feature, no
  breaking change).
- `README.md` line ~22: update the sentence naming `bob-navigation-hotkeys` as "ahead of
  the others at `1.27.1`" to `1.28.0`.
- `README.md` plugin table row for Bob Navigation Hotkeys: extend the existing
  `Ctrl+Shift+M` clause. It currently reads "bare `Ctrl+Shift+M` moves the current task
  and focuses the destination note on the moved task, `N<Ctrl+Shift+M>` moves it plus
  the next N movable tasks and focuses the destination note on the first of them". Add
  that on a **Pomodoro sub-bullet** the same keymap instead moves that bullet — and,
  with a count, the next N sibling bullets, each carrying its own descendants — into
  another open Pomodoro in the same note, picked from a filter that also accepts a typed
  name to create a new `- [ ] () — NAME` Pomodoro directly below the current one; that
  names are uppercased and em dashes stripped; that a bullet identical to one already in
  the destination is merged away; that emptying the source Pomodoro leaves a blank
  sub-bullet; and that a `#task` line nested under a Pomodoro entry routes to the
  Pomodoro move rather than the cross-note move. Match the row's existing dense,
  single-cell prose style.
- `README.md` "Validation" section: extend the `npm test` sentence to mention the new
  named-Pomodoro parsing and Pomodoro bullet-move planner coverage.

**2. Sync.** Run `bob plugins sync` from this workspace — the one sync for the whole
epic, per `bob-plugins/AGENTS.md`.

**3. Glossary (bob-cli).** Rewrite `sase/memory/glossary/pomodoro.md`. Keep the
`keyword: Pomodoro` frontmatter. The body must stay a single tight paragraph — this
strand is pulled into context by `sase memory read`, and the glossary web uses
`closure: mentions`, so naming _task link_ in prose is enough to link the related strand
(no wiki-link syntax; see `glossary.md` and the sibling strands). Cover, and no more:

- what a Pomodoro is (column-0 checkbox under `## Pomodoros` in a daily file, one work
  session);
- the body grammar — a timespan such as `(**0920-0950** [t:: 30m])` or an empty `()`
  placeholder;
- **named Pomodoros**: an optional ` — NAME` suffix (em dash + ALL-CAPS name);
- past (closed) / current (open + timespan) / planned (open + `()`) — preserving the
  distinction the current entry already makes;
- child bullets log that session's work (mentioning _task link_);
- `N<ctrl+shift+m>` on a sub-bullet moves it and the next N siblings into another open
  Pomodoro, or into a new named one created just below — matching how the Schedule Log
  strand cites `<ctrl+shift+p>`.

Target roughly 90–110 words. Every token in this strand is paid for on demand, so cut
any sentence that does not change what an agent would do.

**4. Regenerate memory.** The user explicitly asked for the glossary update, which
carries approval for the full workflow, so run:

```bash
sase memory init
```

That regenerates `AGENTS.md`, the provider instruction shims, and the memory README.
Both repos' changes belong in this turn's final declaration.

**Done when:** `npm test` and `npm run validate` pass in `bob-plugins`,
`bob plugins sync` has run, `sase memory init` has run, and `git status` in both repos
shows only intended changes.

## Out of scope

Leave these alone; file them as task beads via `/sase_new_task` if they still look worth
doing at the end.

- **bob-cli.** No Rust change is needed. `pomodoro::open_ledger_task` only inspects the
  checkbox, `task_time_range` scans parentheticals, and `clean_task` strips the range
  and bracket fields — so a named Pomodoro simply renders its name in `bob pomodoro` /
  tmux status output (`0920-0950 — BODY`), which is a benign improvement.
  `bob-ledger-tools` splices time ranges by `startCh`/`endCh`, so a trailing ` — NAME`
  survives its increment and offset operations. Verify this holds; do not "fix" it.
- **Inheriting names on completion.** `buildPomodoroCompletionPlan` in
  `task-status-cycler` inserts a bare `- [ ] ()` when `Ctrl+Enter` completes a Pomodoro.
  Whether a follow-on Pomodoro should inherit the completed one's name is a real
  question, but it belongs to `task-status-cycler`, not this keymap.
- **A "Pomodoro sub-bullet" glossary strand.** `~/bob/bob.md` already tracks "Add
  'pomodoro bullet' to glossary!" as its own open task with its own type taxonomy. This
  epic only extends the existing Pomodoro strand.
- **Migrating `<ctrl+shift+]>` features to `<ctrl+shift+m>`.** Tracked separately in
  `~/bob/bob.md` and scheduled for 2026-09-01.
- **Moving bullets across notes**, reordering Pomodoros, and the
  `<ctrl+shift+option+j/k>` up/down keymaps.
