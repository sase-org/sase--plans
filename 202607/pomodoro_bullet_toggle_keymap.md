---
tier: tale
title: Ctrl+Alt+] toggles an empty Pomodoro sub-bullet into a Pomodoro placeholder
goal:
  In an Obsidian daily note, Ctrl+Alt+] converts an empty sub-bullet into a `- [ ] ()`
  Pomodoro placeholder and converts an empty, open Pomodoro placeholder back into an
  empty sub-bullet, with no effect anywhere else.
create_time: 2026-09-09 19:53:18
status: wip
---

# Ctrl+Alt+] Pomodoro bullet toggle

## Goal

Add a single reversible keymap to the `task-status-cycler` Obsidian plugin. Inside the
`## Pomodoros` section of a daily note:

- `Ctrl+Alt+]` on an **empty sub-bullet** (`\t- `, ` -`, …) rewrites that line to the
  Pomodoro placeholder `- [ ] ()` and parks the cursor between the parentheses.
- `Ctrl+Alt+]` on an **empty, open Pomodoro placeholder** (`- [ ] ()`) rewrites that
  line back to an empty sub-bullet (`\t- `) and parks the cursor at end of line.

Everywhere else the chord is inert: the command is not offered and the document is
untouched. This closes the vault task `~/bob/bob.md ^bullet-to-pom` ("Add `<ctrl+opt+]>`
keymap to convert bullet to pomodoro bullet!").

## Where the work happens

All source changes land in the **`bob-plugins` linked repository**, not in the vault.
Open it first:

```bash
sase repo open bob-plugins -r "<reason>"
```

Use the printed checkout path for every read and write. Do **not** edit
`~/bob/.obsidian/plugins/task-status-cycler/` directly — that directory is a deploy
target and is overwritten by `bob plugins sync`.

Files touched:

- `plugins/task-status-cycler/main.js` — behavior, registration, helper exports.
- `plugins/task-status-cycler/manifest.json` — version + description.
- `scripts/test-task-status-cycler.cjs` — coverage.
- `README.md` — plugin table row.

## Findings

Everything this feature needs already exists in `plugins/task-status-cycler/main.js`;
the work is composition plus a new daily-note guard.

- **The two literal forms are already named constants** (`main.js:65-67`):
  `POMODORO_PLACEHOLDER_RE = /\(\s*\)/`, `POMODORO_PLACEHOLDER_LINE = "- [ ] ()"`, and
  `EMPTY_POMODORO_SUB_BULLET_LINE = "\t- "`. The recursive Ctrl+Enter completion flow
  already emits exactly this pair when it creates the next Pomodoro (`main.js:~1886`),
  so the new keymap must produce byte-identical output.
- **Section and block detection exist**: `findPomodorosSectionInLines()`,
  `lineIsInPomodorosSection()`, `isTopLevelTaskLine()` (`TOP_LEVEL_TASK_LINE_RE`),
  `INDENTED_LIST_LINE_RE`, `getSubBulletBlockRange()`,
  `getOwningPomodoroContextForLine()`, `TASK_LINE_RE`, `TRAILING_BLOCK_ID_RE`, and
  `CHILD_BULLET_INDENT_UNIT` (a literal tab, the repo's canonical child-bullet indent).
- **Cursor placement inside `()` exists**: `getPomodoroCursorTargetCh()` returns the
  offset just before the closing paren and is already used after Pomodoro creation.
- **No key conflict.** `getOptionBracketTaskCycleDirection()` (`main.js:226`) — the
  handler behind the existing `Alt+[` / `Alt+]` status cycle — returns `null` whenever
  `event.ctrlKey` is set, so `Ctrl+Alt+]` cannot reach it. The vault's
  `~/bob/.obsidian/hotkeys.json` binds `Alt+]`, `Alt+[`, `Alt+T`, `Alt+P`,
  `Ctrl+Alt+Shift+N`, and `Mod+Alt+Arrow{Left,Right}`; nothing claims `Ctrl+Alt+]`.
- **Two registration patterns are established.** `addCommand({ editorCheckCallback })`
  gives a command that greys itself out when ineligible;
  `registerChildBulletInputListeners()` / `getChildBulletKeydownDirection()`
  (`main.js:5559-5648`) is the narrow capture-phase `keydown` fallback the plugin uses
  so a chord still fires while CodeMirror Vim owns the keyboard. Both are needed here.
- **There is no daily-note detector in any plugin yet.** `bob-ledger-tools` knows the
  layout only as `DEFAULT_DAILY_NOTES_FORMAT = "YYYY/YYYYMMDD"` (`main.js:8`).
  `bob-cli`'s `canonical_daily_date()` (`src/native/task_status_hooks.rs:523`) is the
  authoritative rule: relative path is exactly `<4-digit-year>/<YYYYMMDD>.md`, the
  filename is 11 characters, the date parses, and the directory year equals the parsed
  year. The vault confirms it (`~/bob/2026/20260727.md`), and the rule naturally
  excludes the `_done`, `_habit`, `_poms` sidecars that live beside each daily.

## Behavior specification

### Forward: empty sub-bullet → Pomodoro placeholder

Eligible when **all** hold:

1. The active file is a daily note (see the detector below).
2. The cursor line lies inside the `## Pomodoros` section
   (`findPomodorosSectionInLines` + `lineIsInPomodorosSection`).
3. The cursor line matches `INDENTED_LIST_LINE_RE` — indented, any of `-` `+` `*` or an
   ordered marker.
4. The body after the list marker is empty or whitespace only.
5. The line is not itself a checkbox (`\t- [ ] …` is rejected).

Effect: replace the entire line with `POMODORO_PLACEHOLDER_LINE` (`- [ ] ()`). No
neighbouring line is inserted, removed, or re-indented. Move the cursor to
`getPomodoroCursorTargetCh()` of the new line, i.e. between the parentheses, ready to
type or to receive a ledger expansion.

### Reverse: empty open Pomodoro → empty sub-bullet

Eligible when **all** hold:

1. The active file is a daily note.
2. The cursor line lies inside the `## Pomodoros` section.
3. The line is a top-level task line (`isTopLevelTaskLine`) whose checkbox symbol is
   exactly `" "` (open).
4. The body after the checkbox, trimmed, is exactly an empty placeholder — it matches
   `POMODORO_PLACEHOLDER_RE` and nothing else remains: no time range, no `[t:: …]`, no
   tomato marker, no link, no trailing block id, no prose.
5. The Pomodoro owns no sub-bullets — `getSubBulletBlockRange()` for that line is empty.

Effect: replace the entire line with `${CHILD_BULLET_INDENT_UNIT}- ` (identical to
`EMPTY_POMODORO_SUB_BULLET_LINE`). Place the cursor at end of line.

### Otherwise

No document change, and `editorCheckCallback` returns `false` while `checking` so the
command is greyed out in the palette and the chord falls through untouched.

## Decisions worth your veto

These are the judgment calls the request left open. Each takes the conservative reading;
say the word at the approval gate and I will flip any of them.

1. **Scoped to the `## Pomodoros` section, not the whole daily note.** You said "only
   active in daily files"; this is narrower. Rationale: `- [ ] ()` is meaningless
   outside `## Pomodoros`, empty sub-bullets are common in a daily's other sections, and
   an unscoped binding would silently clobber them. Relaxing this later is a one-line
   change to the guard.
2. **"empty/open" is read as empty _and_ open.** A Pomodoro carrying a logged range
   (`- [x] (**0630-0645** [t:: 15m])`) or any non-`[ ]` status is refused, so the chord
   can never destroy recorded time.
3. **A Pomodoro that owns sub-bullets is refused.** Collapsing it to `\t- ` would
   silently re-parent its children into the Pomodoro above. Refusing keeps the toggle an
   exact inverse.
4. **The reverse always emits one tab.** ` -` → `- [ ] ()` → `\t- `: the round trip
   normalizes indentation to the repo's canonical `CHILD_BULLET_INDENT_UNIT` rather than
   remembering the original whitespace. This matches `EMPTY_POMODORO_SUB_BULLET_LINE`
   and the vault's dominant nested-list style.
5. **Modifiers are registered as `["Ctrl", "Alt"]`,** matching the literal `Ctrl+Alt`
   request and the vault task's `<ctrl+opt+]>` phrasing (Control+Option on macOS),
   rather than `Mod+Alt` which would become Command+Option there.

## Implementation

1. **Daily-note detector.** Add a pure `getDailyNoteDateFromPath(path)` to
   `plugins/task-status-cycler/main.js` that ports `bob-cli`'s `canonical_daily_date()`
   rule: split the vault-relative path into exactly two components, require a 4-digit
   numeric directory and an `YYYYMMDD.md` filename of length 11, require the date to be
   a real calendar date (reject `20261332`, `20260231`), and require the directory year
   to equal the parsed year. Return the date or `null`, and wrap it in
   `isDailyNotePath(path)`. Reading the internal `daily-notes` plugin's configured
   format is explicitly out of scope — `bob-ledger-tools` already carries
   `DEFAULT_DAILY_NOTES_FORMAT` if that ever needs generalizing.

2. **Pure toggle planner.** Add `getPomodoroBulletToggle(lines, activeLine)` returning
   either `null` or `{ line, direction, sourceLineText, lineText, cursorCh }`, where
   `direction` is `"to-pomodoro"` or `"to-bullet"`. It implements the eligibility rules
   above using the existing section/range helpers, classifies before it rewrites, and
   never mutates its input. Keeping the whole decision in one pure function is what
   makes the behavior testable without an editor.

3. **Command registration.** Add
   `addCommand({ id: "toggle-pomodoro-bullet", name: "Toggle Pomodoro placeholder/sub-bullet", hotkeys: [{ modifiers: ["Ctrl", "Alt"], key: "]" }], editorCheckCallback })`.
   The check callback reads the editor lines plus the active file path, returns `false`
   unless `isDailyNotePath()` and `getPomodoroBulletToggle()` both pass, and on
   execution applies the single-line replacement and sets the cursor. Route the write
   through the plugin's existing single-line replacement path so undo stays one step.

4. **Vim-mode capture fallback.** Mirror `registerChildBulletInputListeners()` with a
   `registerPomodoroBulletToggleInputListeners()` plus a
   `getPomodoroBulletToggleKeydown(event)` gate that accepts **exactly**
   `ctrlKey && altKey && !shiftKey && !metaKey` with `event.code === "BracketRight"`,
   dedupes via a `WeakSet` so the `window` and `document` capture listeners cannot
   double-fire, and resolves the focused Markdown view the same way the child-bullet
   path does. Only call `preventDefault()`/`stopPropagation()` once the toggle is
   confirmed eligible, so an ineligible press still reaches Obsidian. Deregister on
   unload through `this.register(...)`.

5. **Exports.** Add `getDailyNoteDateFromPath`, `isDailyNotePath`,
   `getPomodoroBulletToggle`, and `getPomodoroBulletToggleKeydown` to the
   `module.exports.helpers` object (`main.js:8032`), keeping its alphabetical grouping.

6. **Release metadata.** Bump `plugins/task-status-cycler/manifest.json` from `1.6.2` to
   `1.7.0` (new feature), extend its `description` to mention the Ctrl+Alt+]
   Pomodoro/sub-bullet toggle, and update the matching version and description cells in
   the `README.md` plugin table.

## Tests

Extend `scripts/test-task-status-cycler.cjs` (Node's built-in `node --test`, driven
through the exported `helpers`):

- **Forward accepts**: `\t- `, ` -` (two spaces), `\t\t- ` (deeper), `\t* `, `\t- ` with
  trailing spaces, and `\t-` with no trailing space. Each yields exactly `- [ ] ()` and
  a cursor offset inside the parens.
- **Forward rejects**: a sub-bullet with content (`\t- [[sase#^read-sase-beads]]`), an
  indented checkbox (`\t- [ ] `), a top-level `- `, an empty sub-bullet in another `##`
  section of the same daily, and an empty sub-bullet under `## Pomodoros` in a non-daily
  file.
- **Reverse accepts**: `- [ ] ()` and `- [ ] ( )` with no following sub-bullet, each
  yielding exactly `\t- ` with the cursor at end of line.
- **Reverse rejects**: `- [ ] (**0630-0645** [t:: 15m])`, `- [x] ()`, `- [/] ()`,
  `- [-] ()`, `- [ ] () ^blockid`, `- [ ] () 🍅 [[x#^y]]`, a `- [ ] ()` that owns
  sub-bullets, and a `- [ ] ()` outside the Pomodoros section.
- **Round trip**: `\t- ` → `- [ ] ()` → `\t- ` is stable, and applying the toggle twice
  from ` -` normalizes the two-space indent to a tab (documents decision 4).
- **Realistic fixture**: the shape from a real daily — a `[x]` Pomodoro with `~~…~~`
  sub-bullets, an open Pomodoro with link sub-bullets, then a trailing `- [ ] ()` with
  an empty sub-bullet — asserting the toggle fires only on the empty sub-bullet and
  leaves every other line byte-identical.
- **Key gating**: `getPomodoroBulletToggleKeydown` accepts `Ctrl+Alt+BracketRight` and
  rejects bare `Alt+]`, `Ctrl+]`, `Ctrl+Alt+Shift+]`, `Ctrl+Alt+Meta+]`, and
  `Ctrl+Alt+BracketLeft`.
- **Non-regression**: `getOptionBracketTaskCycleDirection` still returns `1` for bare
  `Alt+]`, `-1` for `Alt+[`, and `null` for `Ctrl+Alt+]`, proving the new chord does not
  shadow the existing status cycle.
- **Daily-path detector**: accepts `2026/20260727.md`; rejects `2026/20260727_poms.md`,
  `2026/20260727_done.md`, `2025/20260727.md` (year mismatch), `20260727.md` (no year
  directory), `2026/07/20260727.md` (too deep), `2026/20261332.md` and
  `2026/20260231.md` (invalid dates), and `projects/foo.md`.

## Verification and deployment

1. From the `bob-plugins` checkout, run the focused file first, then the full suite and
   the manifest/syntax gate:

   ```bash
   node --test scripts/test-task-status-cycler.cjs
   npm test
   npm run validate
   ```

2. Preview, then perform, the deploy to the vault. In a SASE workspace the linked-repo
   checkout is not the default `BOB_PLUGINS_DIR`, so `-r` is required; `--no-pull` is
   right because `sase repo open` already refreshed the checkout:

   ```bash
   bob plugins sync --no-pull --dry-run -p task-status-cycler -r "<bob-plugins checkout path>"
   bob plugins sync --no-pull -p task-status-cycler -r "<bob-plugins checkout path>"
   ```

   If the vault copy is dirty in the vault's git repo, `sync` refuses rather than
   clobbering; inspect before reaching for `--force`.

3. Manually confirm in Obsidian against today's daily note: on the trailing empty
   sub-bullet under `## Pomodoros`, `Ctrl+Alt+]` produces `- [ ] ()` with the cursor
   between the parens; pressing it again returns `\t- `; pressing it on a completed
   Pomodoro, on a link sub-bullet, in `## Notes`, and in a non-daily note does nothing.
   Verify in both Vim normal mode and insert mode, and confirm bare `Alt+]` still cycles
   task status.

4. Report whether `~/bob/bob.md ^bullet-to-pom` can be marked done; leave the vault task
   edit to the user unless asked.
