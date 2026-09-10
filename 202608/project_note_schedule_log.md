---
tier: tale
title: Nest a source task's schedule log under ^prj in the new project note
goal:
  The `Ctrl+Shift+Alt+N` "Create project note from task" command moves the source task's
  managed `🗓️ **SCHEDULE LOG**` child bullet (and everything nested under it) to the new
  project note's `^prj` task as a direct child, instead of converting it into a bogus
  `#task` line in the `## Tasks` section.
size: medium
proposed_by: bbugyi200.athena.z4
create_time: 2026-09-09 20:00:22
status: wip
---

# Plan: the schedule log follows the task onto `^prj`

## Context

`Ctrl+Shift+Alt+N` is bound in the vault (`.obsidian/hotkeys.json`) to
`bob-navigation-hotkeys:create-project-note-from-task`. Everything below lives in the
**`bob-plugins` linked repo**, except the one documentation edit in step 7, which lives
in **this `bob-cli` checkout**. Open the linked repo with the `/sase_repo` skill before
touching it:

```bash
sase repo open bob-plugins -r "Nest the source task's schedule log under ^prj"
```

Use only the path that command prints. These are plain CommonJS Obsidian plugins:
`plugins/bob-navigation-hotkeys/main.js` is the source, there is no build step.

### The bug

A task that has been rescheduled through `Ctrl+Shift+P` keeps a managed schedule log as
a direct child bullet:

```markdown
- [ ] #task Ship the widget [created::2026-07-01] [scheduled:: 2026-08-20] ^widget
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-05 → 2026-08-20_ — 🎲 P2 · in **15** (8–30) days
    - _2026-08-05_ — waiting on the vendor quote
  - Draft the API
  - REQUIREMENTS
    - Must work offline
```

`buildProjectSeedFromChildBullets()` treats every direct child that is not an ALL-CAPS
section bullet as a task candidate, so the marker becomes a task in the new project
note:

```js
// verified against plugins/bob-navigation-hotkeys/main.js @ 07250cf
taskLines: [
  "- [ ] #task 🗓️ **SCHEDULE LOG** [created::2026-08-13]",
  "\t- *2026-08-05 → 2026-08-20* — 🎲 P2 · in **15** (8–30) days",
  "\t- *2026-08-05* — waiting on the vendor quote",
  "- [ ] #task Draft the API [created::2026-08-13]",
];
```

That line is a fake open task in `## Tasks`, it inflates the project's task counts, and
the log stops being a log — `findScheduleLogParent()` only recognizes a marker that is a
**direct child** of the task it belongs to, so the next `Ctrl+Shift+P` on the project
would start a second, unrelated log.

### How the command works today

`BobNavigationHotkeysPlugin.createProjectNoteFromTask(editor, view)` (around line
20302):

1. `getProjectSourceTaskBlock()` (line 3453) captures the cursor's task line plus its
   contiguous indented child block:
   `{ startLine, endLineExclusive, lines, childLines }`.
2. `buildProjectSeedFromChildBullets(childLines, createdDate)` (line 3660) splits that
   child block into `{ taskLines, sections, lossless }`. Direct children (the shallowest
   child indent) become `- [s] #task ... [created::DATE]` lines, unless they qualify as
   ALL-CAPS section bullets, whose descendants are copied in verbatim as section notes.
   `lossless: false` means some nonblank child line could not be represented.
3. The caller (lines 20363-20377) only accepts the conversion when
   `conversion.lossless && (conversion.taskLines.length > 0 || conversion.sections.length > 0)`;
   otherwise it sets `childConversionLossy`, which later shows a notice and **keeps the
   source task** so nothing is lost.
4. `createProjectNoteFile()` instantiates `_templates/new_project.md` through Templater
   and `applyProjectCreationFrontmatter()` writes `parent`/`type`/`status`/`scheduled`.
5. `buildProjectContentFromTask(content, parsedTask, { childTaskLines, sections })`
   (line 4172) replaces the `(REPLACE WITH PROJECT COMPLETION CRITERIA)` placeholder on
   the `^prj` line, applies the priority, calls `replaceProjectTasksPlaceholder()`, then
   `insertProjectSectionNotes()`. It returns
   `{ content, seeded, tasksInserted, tasksSectionMissing, sectionsInserted, sectionsCreated }`.
6. Each partial failure gets its own "…; source task was kept" notice (lines
   20440-20469).
7. On full success `removeTaskBlockFromContent()` deletes the source block and
   `getProjectFromTaskNoticeText()` (line 4316) reports the result.

The project template (`_templates/new_project.md` in the vault) is:

```markdown
- [ ] #task #prj (REPLACE WITH PROJECT COMPLETION CRITERIA) #hide ^prj

## Tasks

- [ ] #task (REPLACE WITH TASK DESCRIPTION) [created::<date>]

## Future Work

## Requirements
```

### Why `^prj` is the right destination

The `^prj` task _is_ the source task after the move: it inherits the description, the
priority, and — through frontmatter — the schedule. `Ctrl+Shift+P` on a project note's
`^prj` task already writes its schedule log as a **direct child of the `^prj` bullet**
(`planScheduleLogEntry()` at line 1601, reached from the project-scheduled writer around
line 16794). Nesting the moved log there means the project note simply continues the
history the task started, and `findScheduleLogParent()` will append to it.

The relevant existing pieces:

- `SCHEDULE_LOG_PARENT_RE` / `parseScheduleLogParentBullet()` (lines 288, 1500)
  recognize the marker in all three accepted spellings: `🗓️ **SCHEDULE LOG**`, the
  emoji-less `**SCHEDULE LOG**`, and the legacy `**Schedule log:**`. Both are already
  exported in `helpers`.
- `SCHEDULE_LOG_INDENT_UNIT` is `"\t"` (line 264), and `getDependencyChildIndent()` /
  `getScheduleLogEntryIndent()` already treat one tab as one nesting level.
- `isProjectLifecycleTaskLine(lineText)` (line 9237) is true exactly for a `#task` line
  whose trailing block ID is `prj`. Already exported.

## Desired behavior

For the source task above, `Ctrl+Shift+Alt+N` produces:

```markdown
---
parent: "[[Area]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: 2026-08-13T09:00:00-04:00
scheduled: 2026-08-20
---

- [ ] #task #prj Ship the widget [created::2026-07-01] #hide ^prj
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-05 → 2026-08-20_ — 🎲 P2 · in **15** (8–30) days
    - _2026-08-05_ — waiting on the vendor quote

## Tasks

- [ ] #task Draft the API [created::2026-08-13]

## Future Work

## Requirements

- Must work offline
```

### Recognition rules

A direct child bullet is a **schedule-log bullet** when `parseScheduleLogParentBullet()`
matches it. That check runs **before** the section-bullet check and before the task
conversion, and it wins over both:

- The three accepted marker spellings all qualify; a bullet is never rewritten to the
  canonical spelling on the way over. The plugin's stated rule (comment at line 253) is
  that an existing log is never orphaned **or silently rewritten**, and the move is not
  the moment to change that.
- A checkbox rules the bullet out, because `SCHEDULE_LOG_PARENT_RE` does not match
  `- [ ] 🗓️ **SCHEDULE LOG**` at all. Such a bullet keeps today's task conversion.
- A plain `- SCHEDULE LOG` bullet (no `**`) is _not_ a marker. It stays on the existing
  path, which makes it an ALL-CAPS section bullet titled `Schedule Log` when it has
  nested list items. Pin this boundary with a test; do not special-case it.
- Only bullets at the **direct-child indent** are moved. A marker nested deeper belongs
  to the child bullet above it and is copied verbatim under that converted task, exactly
  as today.
- An empty marker with no entries still moves; the marker itself is the task's statement
  that it keeps a log.
- If the source task somehow has more than one direct-child marker, all of them move, in
  source order, each with its own subtree. The project note then mirrors the source task
  exactly, and `findScheduleLogParent()` picks the first one just as it did before —
  transfer, do not normalize.

### Rendering rules

The moved lines are copied verbatim apart from re-indentation:

- The marker bullet is re-indented to exactly one tab (`SCHEDULE_LOG_INDENT_UNIT`),
  which is what the plugin's own writers use for a direct child of `^prj`, and what
  `bob projects sync` expects for `^prj` sub-block lines (`render_subprojects_line()` in
  `src/native/projects.rs` renders at `prj_indent + "\t"`).
- Every descendant keeps its indentation **relative** to the marker, prefixed by that
  tab. With the vault's tab-indented notes this yields the canonical `\t` / `\t\t`
  shape. A space-indented source keeps its own relative widths (`"\t  - *…* — …"`); that
  is cosmetically mixed but self-consistent, and `getScheduleLogEntryIndent()` reuses an
  existing entry's indent for later writes, so the log stays coherent. Mirroring the two
  existing sibling helpers is worth more than a depth-renormalizing pass here.
- A descendant whose leading whitespace does not start with the marker's own indent
  (mixed tabs and spaces) falls back to one extra tab, the way
  `normalizeNestedChildLine()` already does.
- Blank lines are trimmed from both ends of the moved block and preserved in the middle,
  matching `flushTask()` / `flushSection()`.

### Placement rules

The moved lines are spliced in immediately after the `^prj` task line, before
`## Tasks`. They are the `^prj` bullet's own child block, so no blank line is added
around them.

`bob projects sync` later inserts its `🧩 **Sub-projects:**` line at the same level,
directly after the `^prj` line (`prj_sub_block_layout()` + the else branch around line
1966 of `src/native/projects.rs`), which pushes the marker down one line. That is
harmless: both stay direct children of `^prj`, so `findScheduleLogParent()` still
resolves the marker through `findNearestParentListItem()`.

## Implementation steps

### 1. Add the re-indent helper

In `plugins/bob-navigation-hotkeys/main.js`, next to `normalizeProjectSectionNoteLine()`
(around line 3594), add a third sibling:

```js
// Re-indent a line from the source task's schedule-log subtree so the marker
// bullet sits one Obsidian Tab level under the new note's `^prj` task and every
// descendant keeps its depth relative to that marker. Blank lines collapse to
// "". A descendant whose indentation does not extend the marker's (mixed tabs
// and spaces) falls back to one level deeper, like normalizeNestedChildLine().
function normalizeProjectScheduleLogLine(lineText, markerIndent) { … }
```

For the marker line itself (`leading === markerIndent`) the relative indent is `""`, so
the result is `"\t- 🗓️ **SCHEDULE LOG**"`. Note this is deliberately _not_
`normalizeNestedChildLine()`, which forces an empty relative indent to `"\t"` and would
flatten entries onto the marker's level.

### 2. Route schedule-log bullets in `buildProjectSeedFromChildBullets()`

Same file, lines 3660-3859. Extend the return shape:

```js
Object.freeze({
  taskLines, // unchanged
  sections, // unchanged
  scheduleLogLines, // frozen, already re-indented for insertion under `^prj`
  lossless, // unchanged semantics
});
```

- Add a `currentScheduleLog` accumulator alongside `current` and `currentSection`, and a
  `flushScheduleLog()` that end-trims blank lines and appends the block to
  `scheduleLogLines` (multiple markers concatenate in source order).
- In the direct-child branch (line 3794), flush all three accumulators, then test
  `parseScheduleLogParentBullet(line)` **first**. On a match, push the marker line
  through `normalizeProjectScheduleLogLine()` with the bullet's own leading indent as
  `markerIndent`, open `currentScheduleLog`, and `continue` — before the section check
  and before `buildProjectTaskLineFromChildBullet()`.
- In the nested branch (lines 3833-3841), add a `currentScheduleLog` arm that pushes
  `normalizeProjectScheduleLogLine(line, currentScheduleLog.markerIndent)`.
- Blank lines (line 3780) push `""` into the open schedule-log accumulator, like the
  other two.
- Lines consumed this way are represented, so they must **not** set `lossless = false`.
- Flush all three at the end (lines 3844-3845).
- Update the function's doc comment; it is the contract other readers rely on.

### 3. Insert the lines under `^prj`

Add two helpers near `findProjectTasksHeaderIndex()` (line 3873):

- `findProjectLifecycleTaskIndex(lines)` → the index of the first
  `isProjectLifecycleTaskLine()` line, or `-1`. Skip frontmatter and fenced code by
  mirroring `findProjectTasksHeaderIndex()`'s scan exactly (same
  `startsWithFrontmatter()` / `FRONTMATTER_DELIMITER_RE` / `getFenceOpening()` /
  `isClosingFence()` handling).
- `insertProjectScheduleLogLines(content, scheduleLogLines)` → `{ content, inserted }`.
  Returns the content unchanged with `inserted: false` when there are no lines or no
  `^prj` line; otherwise splices the lines in at `lifecycleIndex + 1` and rejoins with
  the content's detected line ending, like `replaceProjectTasksPlaceholder()` does.

No dedupe or merge logic: the note was created from the template moments earlier and has
no log of its own. This helper is not a general-purpose merger.

### 4. Thread it through `buildProjectContentFromTask()`

Lines 4172-4232. Accept `options.scheduleLogLines`. Run
`insertProjectScheduleLogLines()` **after** the placeholder/priority replacement and
**before** the `replaceProjectTasksPlaceholder()` call — the `^prj` child block is part
of seeding the lifecycle task, and both later helpers re-derive their own indices from
the content string they are handed, so the ordering is safe either way. Add
`scheduleLogInserted` to both returned objects (the early `!seeded` return gets
`false`). Leave `content`, `seeded`, `tasksInserted`, `tasksSectionMissing`,
`sectionsInserted`, and `sectionsCreated` unchanged.

### 5. Wire up `createProjectNoteFromTask()`

Around lines 20357-20469:

- Capture `conversion.scheduleLogLines` next to `taskLines` and `sections`.
- Widen the acceptance gate so a task whose _only_ child is a schedule log still seeds
  it instead of falling into `childConversionLossy` (which would drop the log and keep
  the source task):

  ```js
  conversion.lossless &&
    (conversion.taskLines.length > 0 ||
      conversion.sections.length > 0 ||
      conversion.scheduleLogLines.length > 0);
  ```

- Pass `scheduleLogLines` into `buildProjectContentFromTask()`.
- Add a guard mirroring the `tasksInserted` one: when lines were requested but
  `seedResult.scheduleLogInserted` is false, show
  `"Created project, but the schedule log could not be added; source task was kept"` and
  `return true` without removing the source task. Place it next to the existing
  tasks/sections guards.
- Extend `getProjectFromTaskNoticeText()` (line 4316) with a trailing `scheduleLogMoved`
  boolean parameter that appends a `schedule log moved` detail after the section chip.
  Keep it optional so the existing five-argument call sites and tests are unaffected,
  and pass it from the success notice at line 20507.

### 6. Tests

Add `node:test` cases to `scripts/test-navigation-hotkeys.cjs` in the existing
`helpers.<fn>(...)` + `assert.deepEqual` style, near the project-seed tests around lines
8634-8955. Export `normalizeProjectScheduleLogLine`, `findProjectLifecycleTaskIndex`,
and `insertProjectScheduleLogLines` from the `helpers` object (around line 21824) so the
tests can reach them; `parseScheduleLogParentBullet` and `isProjectLifecycleTaskLine`
are already exported.

Cover at least:

- A direct-child `🗓️ **SCHEDULE LOG**` bullet with two entries lands in
  `scheduleLogLines` as `["\t- 🗓️ **SCHEDULE LOG**", "\t\t- *…* — …", "\t\t- *…* — …"]`,
  is absent from `taskLines`, and leaves `lossless: true`.
- The emoji-less `**SCHEDULE LOG**` and legacy `**Schedule log:**` markers move too,
  with their spelling preserved byte-for-byte.
- `- [ ] 🗓️ **SCHEDULE LOG**` (checkbox) still converts to a task.
- A plain `- SCHEDULE LOG` bullet with sub-bullets still becomes a `Schedule Log`
  section.
- A marker nested under a converted child task stays under that task, unchanged.
- A marker with no entries moves on its own.
- A deeper note under an entry keeps its relative depth (`\t\t\t`).
- A source task whose only child is the schedule log: `taskLines` and `sections` empty,
  `scheduleLogLines` populated, `lossless: true`.
- Two direct-child markers both move, in source order.
- A mixed child block routes tasks, sections, and the log to their three destinations in
  one pass.
- `buildProjectContentFromTask()` on the real template content inserts the lines
  directly under the `^prj` line, above `## Tasks`, with `scheduleLogInserted: true`;
  assert the whole rendered note, combined with `childTaskLines` and a section.
- `buildProjectContentFromTask()` on a customized note whose completion placeholder is
  not on a `^prj` line returns `scheduleLogInserted: false` and content that is
  otherwise seeded normally.
- CRLF content round-trips through `insertProjectScheduleLogLines()`.
- `findProjectLifecycleTaskIndex()` skips a `^prj` line inside frontmatter or a fenced
  code block.
- `getProjectFromTaskNoticeText()` renders the `schedule log moved` detail, and omits it
  when the flag is falsy.

Run from the `bob-plugins` checkout:

```bash
npm test
npm run validate
```

### 7. Docs, version, and deploy

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.24.0` to `1.25.0`.
  Versions are per-plugin; touch no other manifest.
- Update the `Bob Navigation Hotkeys` row in the `bob-plugins` `README.md` plugin table
  (version column and description) to mention that a source task's schedule log moves to
  the new project's `^prj` task, matching the row's existing run-on style.
- In **this `bob-cli` checkout**, extend `docs/projects.md`. The ALL-CAPS section-bullet
  paragraph ends around line 215; add a short paragraph after it covering the
  schedule-log move: which marker spellings are recognized, that it lands as a direct
  child of `^prj` rather than a `## Tasks` line, and that later `Ctrl+Shift+P` entries
  continue the same log. Keep the file's existing line-wrapping width. This is a
  documentation-only change to the Rust project; no `just` recipe is required for it.
- Deploy to the vault, as `bob-plugins/AGENTS.md` requires. Run from the `bob-plugins`
  checkout the `/sase_repo` skill printed. `-r "$PWD"` is required because the default
  source path does not exist in a SASE workspace, and `--no-pull` avoids a pull against
  the dirty checkout:

  ```bash
  bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull --dry-run
  bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull
  ```

  If sync reports the vault file as dirty and suggests `-F/--force`, do **not** force
  blindly: compare the deployed file against the vault repo's committed baseline first
  and report the discrepancy.

- After deploying, tell the user to reload the plugin in Obsidian; a running Obsidian
  will not pick up the new `main.js` on its own.

## Verification

1. `npm test` and `npm run validate` pass in the `bob-plugins` checkout.
2. `bob plugins sync … --dry-run` shows only `bob-navigation-hotkeys` `main.js` +
   `manifest.json` changing.
3. Manual check in Obsidian after a plugin reload, from an area note, on a source task
   shaped like the "The bug" example: the new project note's `^prj` task carries the
   `🗓️ **SCHEDULE LOG**` bullet with both entries intact, `## Tasks` holds only
   `Draft the API`, `## Requirements` gains `Must work offline`, and the source block is
   gone from the area note.
4. In that new project note, `Ctrl+Shift+P` on the `^prj` task with a reason appends the
   new entry to the moved log rather than creating a second marker.

## Out of scope

- The managed `🔗 **DEPENDS ON:**` bullet, which has the same problem for the same
  reason. Moving it correctly also means deciding what its `[[#^block-id]]` links should
  point at once the source block ID is rewritten to `^prj`, which is a separate design
  question. Leave it on today's task-conversion path and mention nothing about it in the
  docs.
- Normalizing a legacy or emoji-less marker to the canonical `🗓️ **SCHEDULE LOG**`
  spelling during the move.
- Any change to `bob projects sync` (Rust, `src/native/projects.rs`). A schedule log
  under `^prj` is already a state that command tolerates: it is not a `#task` line, so
  it does not affect counts, and the `🧩 **Sub-projects:**` writer only inspects marker
  lines.
- Editing `_templates/new_project.md`.
- The plain `Ctrl+Shift+N` "Create project note" command, which has no source task.
