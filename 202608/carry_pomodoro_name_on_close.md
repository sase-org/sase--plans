---
tier: tale
title: Carry a closed Pomodoro's name onto the Pomodoro Ctrl+Enter creates
goal:
  When Ctrl+Enter completes a named Pomodoro and the Task Status Cycler creates a fresh
  Pomodoro below it, the new entry is created with the same name (`- [ ] () — NAME`)
  instead of an unnamed `- [ ] ()` placeholder.
size: small
proposed_by: bbugyi200.athena.0ey
create_time: 2026-09-09 19:59:56
status: wip
---

# Carry the closed Pomodoro's name onto the created Pomodoro

## Goal

Pressing `Ctrl+Enter` on an open Pomodoro entry (or on one of its sub-bullets) closes
that Pomodoro and, when it has something to carry forward or is the last entry in the
section, inserts a fresh Pomodoro directly below it. That fresh entry is always the
unnamed placeholder `- [ ] ()`, even when the Pomodoro just closed was a **named**
Pomodoro such as `- [x] (**1815-1905** [t:: 50m]) — RELEASE`.

Make the created entry inherit the closed entry's name, so the same keypress yields
`- [ ] () — RELEASE` with its carried-forward links beneath it. Nothing else about
completion changes: status flip, marker rewriting, worked-on/deferred carry order,
insertion point, cursor placement, and the "do not create, jump to the existing next
Pomodoro" branch all keep their current behavior.

This is a `tale` because the work lives in one plugin file plus its focused test suite,
and one follow-up coding agent can implement, test, release, and deploy it. It is
`small` because there is exactly one construction site for the new entry line, one new
pure parser, and a well-understood release path.

## Repository and current behavior

All implementation work belongs in the `bob-plugins` linked repository. Open it through
`/sase_repo` and use the printed checkout path for every read and write:

```bash
sase repo open bob-plugins -r "Carry a closed Pomodoro's name onto the created Pomodoro"
```

Do not edit the deployed vault plugin under `~/bob/.obsidian/plugins/`;
`bob-plugins/AGENTS.md` requires a `bob plugins sync` after source changes.

Everything below refers to `plugins/task-status-cycler/main.js` at `task-status-cycler`
version `1.13.0`:

| Symbol                           |  Line | Role                                                                                 |
| -------------------------------- | ----: | ------------------------------------------------------------------------------------ |
| `POMODORO_PLACEHOLDER_RE`        |    65 | `/\(\s*\)/` — the empty `()` body                                                    |
| `POMODORO_PLACEHOLDER_LINE`      |    66 | `"- [ ] ()"` — the created entry line                                                |
| `EMPTY_POMODORO_SUB_BULLET_LINE` |    67 | `"\t- "` — the empty child written when nothing is carried                           |
| `isPomodoroTaskLine`             |  2885 | Column-0 checkbox line inside `## Pomodoros` (deliberately body-agnostic)            |
| `getSubBulletBlockRange`         |  2893 | `{ startLine, endLine }` with an **exclusive** `endLine` used as the insertion point |
| `getPomodoroBulletToggle`        |  2916 | `Ctrl+Alt+]` placeholder/bullet toggle                                               |
| `getPomodoroCursorTargetCh`      |  3257 | Cursor column inside the created entry's `()`                                        |
| `buildPomodoroCompletionPlan`    |  3284 | The only completion planner; builds the `insertLines` edit at 3365–3378              |
| `completeActivePomodoroTask`     |  8463 | Both `Ctrl+Enter` dispatch paths (entry line and sub-bullet) funnel through here     |
| `applyPomodoroCompletionPlan`    |  9530 | Applies edits, then sets and centers the cursor                                      |
| `module.exports.helpers`         | 10392 | Test surface; new pure helpers must be exported here                                 |

Two facts make this a one-site change:

- `POMODORO_PLACEHOLDER_LINE` is written in exactly two places — the completion plan
  (3371) and the `Ctrl+Alt+]` toggle (2976). Only the first is `Ctrl+Enter`.
- The cycler has **no** named-Pomodoro grammar today. The grammar already exists in
  `plugins/bob-navigation-hotkeys/main.js` (`POMODORO_NAME_SEPARATOR` 64,
  `POMODORO_NAME_TAIL_RE` 66, `parsePomodoroEntryLine` 6451, `formatPomodoroEntryLine`
  6533), which writes `- [ ] () — NAME` for its `Ctrl+Shift+M` picker. Deployed plugins
  must not import one another, so the cycler duplicates the parts it needs — the same
  convention it already follows for `POMODOROS_HEADING_RE` and
  `POMODORO_PLACEHOLDER_RE`.

Real vault shapes to honor (from `2026/20260826.md`):

```text
- [x] (**1540-1615** [t:: 35m])  — PAGER
- [x] (**1815-1905** [t:: 50m])   — RELEASE
- [ ] () — ARTIFACTS
```

Note the **two or three spaces** before the em dash on closed entries: the ledger time
range expands in place and leaves its own trailing space ahead of the pre-existing
` — NAME`. Parsing must tolerate any run of spaces/tabs there; the entry this plan
creates must always be written canonically with single spaces.

## Required behavior

### 1. A named Pomodoro, for this plan

A column-0 Pomodoro entry line is **named** when, after its checkbox, its trimmed body
begins with either the empty placeholder `()` (or `( )`) or a colon/compact ledger time
range, and the text following that leading parenthetical matches `/^[ \t]*—[ \t]*(.*)$/`
with a nonblank capture. The name is that capture, trimmed.

Parsing the tail only after the leading parenthetical is what keeps an em dash inside
`[t:: ...]` metadata from being read as the separator.

Anything else yields **no name**, and therefore today's exact behavior:

- a body with no leading parenthetical (`- [ ] Focus`, `- [ ] Focus — X`) — the cycler
  accepts such lines as Pomodoros and its existing tests use them;
- a leading parenthetical whose trailing text is not an em-dash tail
  (`- [ ] () note — X`);
- an em-dash tail that is empty or whitespace only (`- [ ] () —`).

Degrading to the unnamed placeholder is always the safe fallback: it is exactly what
ships today.

### 2. Carry the name verbatim

The created entry uses the source name **as written**, trimmed, with no case folding,
whitespace collapsing, em-dash stripping, or length cap — deliberately _not_
`normalizePomodoroName`'s rules:

- names created by the `Ctrl+Shift+M` picker are already normalized, so for every
  picker-created Pomodoro verbatim and normalized carry are identical;
- for a hand-typed name, echoing the user's own text is the least surprising reading of
  "copy that name", and rewriting it would make the two entries disagree in the file;
- nothing downstream needs canonical text: `bob-navigation-hotkeys` normalizes only for
  display and picker matching (`getNormalizedPomodoroEntryName` 7298) and falls back to
  the raw trimmed name when normalization fails, and `bob-cli`'s Rust ledger parsers
  (`open_ledger_task`, `task_time_range` in `src/native/pomodoro.rs`) treat the whole
  body as opaque text after the time range.

Only the separator spacing is canonicalized: the created line is always
`- [ ] () — NAME` with one space on each side of the em dash, regardless of how many
spaces the source line had.

### 3. Where the name is and is not applied

- **Applies** whenever `buildPomodoroCompletionPlan` decides to create an entry — i.e.
  when the closed Pomodoro carries at least one worked-on or deferred link, or when it
  is the last Pomodoro in the section. Both the carried-links and the empty-child
  (`EMPTY_POMODORO_SUB_BULLET_LINE`) variants get the name.
- **Does not apply** when no entry is created (nothing to carry and a later Pomodoro
  already exists). That branch must keep completing in place and jumping the cursor to
  the existing next Pomodoro; it must never rename, move, or otherwise touch that
  existing entry.
- **Does not apply** to reopening (`reopenActivePomodoroTask`), to the `#`-deferral
  move-only toggle, or to any other keymap.
- A name is carried even when a later open Pomodoro already has the same name. Two open
  entries sharing a name are legal today, the `Ctrl+Shift+M` picker lists entries as
  distinct rows, and suppressing the carry would contradict the feature.

### 4. Cursor and downstream behavior are unchanged

`getPomodoroCursorTargetCh` locates the first `()` in the line, so on
`- [ ] () — RELEASE` it still returns column 7, leaving the cursor between the
parentheses ready for the time range, and `applyPomodoroCompletionPlan` still centers
that line. Carried sub-bullet text, marker rewriting on the closed entry, removal of
move-only source bullets, the closed entry's own retained name, and
`finalizeClosedTasks` retirement are all untouched.

## Implementation

### 1. Add the grammar to `plugins/task-status-cycler/main.js`

Beside `POMODORO_PLACEHOLDER_LINE` (line 66), add constants mirroring the
`bob-navigation-hotkeys` grammar, with a comment stating that the duplication is
intentional because deployed plugins must not import one another:

```js
const POMODORO_NAME_SEPARATOR = "—";
const POMODORO_NAME_TAIL_RE = /^[ \t]*—[ \t]*(.*)$/;
const POMODORO_COLON_TIME_RANGE_RE =
  /\((\*\*)?(\d\d):(\d\d)\s*-\s*(\d\d):(\d\d)(\*\*)?(\s+[^)]*)?\)/;
const POMODORO_COMPACT_TIME_RANGE_RE =
  /\((\*\*)?(\d\d)(\d\d)\s*-\s*(\d\d)(\d\d)(\*\*)?(\s+[^)]*)?\)/;
```

### 2. Add two pure helpers

`parsePomodoroEntryLineParts(lineText)` → frozen
`{ placeholder, rangeText, name, trailingText }` or `null`:

- return `null` unless the line is a top-level task line (reuse the existing
  `isTopLevelTaskLine` / `TASK_LINE_RE` pair rather than a new marker regex, so the
  cycler keeps accepting `-`, `+`, `*`, and `1.`/`1)` markers consistently);
- take the body exactly as `getPomodoroBulletToggle` does today —
  `lineText.slice(taskMatch[0].length).trim()`;
- match `POMODORO_PLACEHOLDER_RE` at index 0; if it does not match there, try
  `POMODORO_COLON_TIME_RANGE_RE` then `POMODORO_COMPACT_TIME_RANGE_RE` at index 0;
  return `null` when none matches at index 0;
- `placeholder` is true only for the `()` match; `rangeText` is the matched text;
- apply `POMODORO_NAME_TAIL_RE` to the remaining body. On a match with a nonblank
  trimmed capture set `name` to that trimmed capture and `trailingText` to `""`;
  otherwise set `name` to `null` and `trailingText` to the remaining body verbatim (`""`
  when there is none). A blank em-dash tail is `name: null` with `trailingText: ""`, so
  it reads as an unnamed entry rather than a malformed one.

`formatPomodoroPlaceholderLine(name)` → `POMODORO_PLACEHOLDER_LINE` when `name` is
null/blank, otherwise
`` `${POMODORO_PLACEHOLDER_LINE} ${POMODORO_NAME_SEPARATOR} ${String(name).trim()}` ``.
Keep `POMODORO_PLACEHOLDER_LINE` as the single source of the unnamed form.

Export both from `module.exports.helpers`, alphabetically placed.

### 3. Use them in `buildPomodoroCompletionPlan`

- After the existing status guard, parse the source Pomodoro line once and keep
  `const createdPomodoroName = parsed && parsed.name ? parsed.name : null;`.
- In the `shouldCreatePomodoro` branch, replace the literal `POMODORO_PLACEHOLDER_LINE`
  in the `insertLines` edit with `formatPomodoroPlaceholderLine(createdPomodoroName)`.
- Return `createdPomodoroName` on the plan object (`null` when nothing was created, so
  set it only inside the `shouldCreatePomodoro` branch), next to `createdPomodoro` and
  `copiedBulletLines`, so tests and any future notice can read it without re-parsing.
- Change nothing else in the planner: edit order, `removedLineCountBefore`,
  `cursorTargetLine`, and the carried-link grouping stay byte-for-byte as they are.

### 4. Companion: keep `Ctrl+Alt+]` usable on the entries this feature creates

`getPomodoroBulletToggle`'s "to-bullet" direction currently requires the body to be
_exactly_ `()` (`placeholderMatch[0].length === body.length`), so it already refuses
named placeholders. Once `Ctrl+Enter` starts producing them, the "I do not want another
Pomodoro after all" escape hatch would stop working right where it is most needed.

Relax that one condition to: `parsePomodoroEntryLineParts` returns a parse with
`placeholder === true` and `trailingText === ""` (name present or not). Every other
guard stays — open status, childless
(`subBulletRange.startLine === subBulletRange.endLine`), inside `## Pomodoros`. Toggling
a named placeholder to `EMPTY_POMODORO_SUB_BULLET_LINE` discards the name, and the
reverse direction keeps creating the unnamed `POMODORO_PLACEHOLDER_LINE`; that asymmetry
is intended, since the reverse gesture starts from a blank bullet with no name to
recover.

This step is separable: dropping it leaves the `Ctrl+Enter` feature fully working, at
the cost of that escape hatch on named entries.

### 5. Release metadata

- Bump `plugins/task-status-cycler/manifest.json` from `1.13.0` to `1.14.0` and extend
  its `description` so the Pomodoro clause reads as carrying the closed Pomodoro's name
  onto the created one.
- Update the `Task Status Cycler` row in `README.md`: the version cell and the "Complete
  Pomodoros (carrying worked-on links above deferred `#`-marked links, each in source
  order)" clause, which should also state that a closed named Pomodoro's name is copied
  onto the Pomodoro that completion creates, and that `Ctrl+Alt+]` also toggles named
  empty placeholders back to a sub-bullet.

## Tests

Add focused coverage to `scripts/test-task-status-cycler.cjs` (123 tests pass today;
finish strictly above that baseline without editing an existing test):

- `parsePomodoroEntryLineParts` units: `- [ ] ()`, `- [ ] ( )`,
  `- [x] (**1540-1615** [t:: 35m])  — PAGER` (multi-space separator),
  `- [ ] (**09:20-09:50**) — DEEP WORK`, `- [ ] () — deep work` (verbatim, not
  uppercased), `- [ ] () —` (blank tail → `name: null`, `trailingText: ""`),
  `- [ ] () note — X` (→ `name: null`, nonempty `trailingText`),
  `- [x] (**0920-0950** [t:: 30m — ish])` (em dash inside metadata → `name: null`),
  `- [ ] Focus — X` and a non-task line (→ `null`).
- `formatPomodoroPlaceholderLine`: `null`/`""`/whitespace → `- [ ] ()`; `"RELEASE"` →
  `- [ ] () — RELEASE`; a source name with odd spacing is emitted with single spaces
  around the separator.
- Completion with carried links from a named entry: the `insertLines` edit's first line
  is `- [ ] () — RELEASE`, `plan.createdPomodoroName === "RELEASE"`, and
  `plan.copiedBulletLines` is unchanged from the equivalent unnamed case.
- Completion of a named **last** Pomodoro with nothing to carry: inserts
  `- [ ] () — RELEASE` followed by `EMPTY_POMODORO_SUB_BULLET_LINE`.
- Regression: an unnamed source still inserts exactly `- [ ] ()` and
  `plan.createdPomodoroName` is `null`.
- Nothing to carry with a later Pomodoro present, source named: `createdPomodoro` is
  `false`, `createdPomodoroName` is `null`, no `insertLines` edit exists, and the later
  entry's line text is untouched.
- End-to-end through `applyPomodoroCompletionPlan` with `createTextEditor`: closing
  `- [ ] (**1815-1905** [t:: 50m])  — RELEASE` produces the expected buffer and leaves
  the cursor at `{ line: <created>, ch: 7 }`, i.e. between the parentheses of the named
  entry.
- `getPomodoroBulletToggle` (step 4): a childless open `- [ ] () — ARTIFACTS` toggles to
  `EMPTY_POMODORO_SUB_BULLET_LINE`; the same line **with** a sub-bullet still returns
  `null`; a named entry with a time range still returns `null`; `- [ ] () note` still
  returns `null`; the unnamed cases keep their current results.

## Verification and deployment

```bash
node --test scripts/test-task-status-cycler.cjs
npm test
npm run validate
```

Review the final diff to confirm only `plugins/task-status-cycler/main.js`,
`scripts/test-task-status-cycler.cjs`, that plugin's `manifest.json`, and `README.md`
changed. Then preview and perform the targeted sync from the linked checkout, using the
real path printed by `/sase_repo`:

```bash
bob plugins sync --no-pull --dry-run -p task-status-cycler -r "<bob-plugins checkout path>"
bob plugins sync --no-pull -p task-status-cycler -r "<bob-plugins checkout path>"
```

Do not use `--force` to bypass a dirty deployed copy. Confirm the deployed `main.js` and
`manifest.json` match source at `1.14.0`, and finish with clean or fully accounted-for
`git status` output in both the linked repo and the primary workspace.

## Memory follow-up

The `bob-cli` Pomodoro glossary strand (`sase/memory/glossary/pomodoro.md`) documents
named Pomodoros and the `N<ctrl+shift+m>` move, but says nothing about what `Ctrl+Enter`
creates. Shipping this makes a one-sentence addition worthwhile.

Do **not** edit that memory file from this plan — plan-file authorization does not count
as user approval for SASE memory edits. After the code ships, use `/sase_new_task` to
file a `task(memory)` bead proposing the sentence, so the project owner can approve it.

## Out of scope

- Renaming, reordering, merging, or deleting any pre-existing Pomodoro entry, including
  the "jump to the existing next Pomodoro" branch.
- Normalizing carried names (uppercasing, whitespace collapsing, em-dash stripping,
  length caps) or otherwise changing the named-Pomodoro grammar,
  `normalizePomodoroName`, or `POMODORO_NAME_MAX_LENGTH` in `bob-navigation-hotkeys`.
- Any `bob-navigation-hotkeys`, `bob-ledger-tools`, or `block-id-prompt` change,
  including the ledger time-range expansion that leaves the extra space before ` — NAME`
  on closed entries.
- Naming the Pomodoro created by the `Ctrl+Alt+]` "to-pomodoro" direction, or adding a
  name prompt/picker to any `task-status-cycler` gesture.
- Reopen (`Ctrl+Enter` on a completed Pomodoro), marker repair, carried-link ordering,
  deferral (`#`) semantics, and all `bob-cli` Rust-side Pomodoro logic.
