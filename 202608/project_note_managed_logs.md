---
tier: tale
title: Move managed work logs under ^prj when promoting a task to a project
goal:
  Ctrl+Shift+Alt+N moves a source task's managed work logs and schedule logs under the
  new project note's ^prj task instead of converting either into a
size: medium
proposed_by: bbugyi200.athena.0g4
create_time: 2026-09-09 20:00:22
status: wip
---

# Plan: work logs follow the task onto `^prj`, like schedule logs already do

## Context

`Ctrl+Shift+Alt+N` is bound in the vault (`.obsidian/hotkeys.json`) to
`bob-navigation-hotkeys:create-project-note-from-task`. Almost everything below lives in
the **`bob-plugins` linked repo**; only the documentation edit in step 8 lives in **this
`bob-cli` checkout**. Open the linked repo with the `/sase_repo` skill before touching
it:

```bash
sase repo open bob-plugins -r "Move managed work logs under ^prj when promoting a task to a project"
```

Use only the path that command prints. These are plain CommonJS Obsidian plugins:
`plugins/bob-navigation-hotkeys/main.js` is the source, there is no build step.

### The bug

An in-progress task that has been cycled through `Ctrl+Shift+Enter` with a work summary
keeps a managed **Work Log** as a direct child bullet, right alongside the managed
**Schedule Log** that `Ctrl+Shift+P` writes:

```markdown
- [ ] #task Ship the widget [created::2026-07-01] [scheduled:: 2026-08-20] ^widget
  - Draft the API
  - REQUIREMENTS
    - Must work offline
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-05 → 2026-08-20_ — 🎲 P2 · in **15** (8–30) days
  - 🛠️ **WORK LOG**
    - _2026-08-15_ — Added coverage for the parser
    - _2026-08-11_ — Sketched the migration
```

`ec6c3c2` taught `buildProjectSeedFromChildBullets()` to move the **Schedule Log** under
the new note's `^prj` task, but it only ever learned the schedule marker.
`bob-navigation-hotkeys/main.js` contains no Work Log grammar at all, so the work-log
marker still falls through to the task-conversion branch:

```js
// verified against plugins/bob-navigation-hotkeys/main.js @ b57e485
taskLines: [
  "- [ ] #task Draft the API [created::2026-08-29]",
  "- [ ] #task 🛠️ **WORK LOG** [created::2026-08-29]",
  "	- *2026-08-15* — Added coverage for the parser",
  "	- *2026-08-11* — Sketched the migration",
];
```

That line is a fake open task in `## Tasks`: it inflates the project's open-task counts,
it keeps `^prj` hidden off `dash.md` forever (an active project with a permanently open
task never has `#hide` removed by `bob projects sync`), and the log stops being a log.
`planWorkLogInsertion()` in `block-id-prompt` and the work-log writer in
`task-status-cycler` both require the marker to be a **direct child** of the task it
belongs to, so the next `Ctrl+Shift+Enter` on the project's `^prj` task would start a
second, unrelated work log.

### How the command works today

`BobNavigationHotkeysPlugin.createProjectNoteFromTask(editor, view)` (line 23189):

1. `getProjectSourceTaskBlock()` (line 3497) captures the cursor's task line plus its
   contiguous indented child block:
   `{ startLine, endLineExclusive, lines, childLines }`.
2. `buildProjectSeedFromChildBullets(childLines, createdDate)` (line 3950) splits that
   child block into `{ taskLines, sections, scheduleLogLines, lossless }`. In the
   direct-child branch (line 4106) it flushes the three accumulators, then tests
   `parseScheduleLogParentBullet(line)` (line 1515) **first**; on a match it opens
   `currentScheduleLog` and re-indents through `normalizeProjectScheduleLogLine()` (line
   3663). Otherwise it falls through to the ALL-CAPS section check and finally to
   `buildProjectTaskLineFromChildBullet()`. The nested branch (line 4164) has a matching
   `currentScheduleLog` arm.
3. The caller (lines 23253-23272) accepts the conversion only when
   `conversion.lossless && (taskLines.length > 0 || sections.length > 0 || scheduleLogLines.length > 0)`;
   otherwise `childConversionLossy` shows a notice and **keeps the source task**.
4. `createProjectNoteFile()` instantiates `_templates/new_project.md` through Templater
   and `applyProjectCreationFrontmatter()` writes `parent`/`type`/`status`/`scheduled`.
5. `buildProjectContentFromTask(content, parsedTask, { scheduleLogLines, childTaskLines, sections })`
   (line 4579) fills the `^prj` completion placeholder, applies the priority, calls
   `insertProjectScheduleLogLines()` (line 4296, splices at
   `findProjectLifecycleTaskIndex() + 1`), then `replaceProjectTasksPlaceholder()`, then
   `insertProjectSectionNotes()`.
6. Each partial failure gets its own "…; source task was kept" notice (lines
   23345-23383), including the schedule-log guard at line 23352.
7. On success `removeTaskBlockFromContent()` deletes the source block and
   `getProjectFromTaskNoticeText()` (line 4740) reports the result; its optional sixth
   parameter `scheduleLogMoved` appends a `schedule log moved` detail.

The reverse direction already handles both kinds: `splitProjectNoteForReversal()`
(line 11626) collects the entire `^prj` subtree as `lifecycleChildLines` and
`buildTaskBlockFromProjectNote()` re-indents it back under the restored parent task
without inspecting it, so a work log parked on `^prj` round-trips for free.

### The canonical grammar already exists — in Rust

`bob-cli` owns the shared marker grammar in `src/native/capture.rs`
(`parse_managed_task_log_marker()`, `ManagedTaskLogKind`, lines 1702-1795). It accepts
**both** kinds and — importantly — rejects a mismatched emoji/label pair: it strips a
known emoji and then filters the parsed label down to that emoji's kind, so
`🗓️ **WORK LOG**` and `🛠️ **SCHEDULE LOG**` are not markers. `task-status-cycler`
carries a comment (line 580) stating its work-log grammar must stay byte-compatible with
bob-cli's. `bob-navigation-hotkeys` is the one plugin on the promotion path that never
got the work-log half. This plan closes that gap and mirrors bob-cli's per-kind
rejection rule exactly.

## Desired behavior

For the source task in "The bug", `Ctrl+Shift+Alt+N` produces:

```markdown
---
parent: "[[Area]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: 2026-08-29T09:00:00-04:00
scheduled: 2026-08-20
---

- [ ] #task #prj Ship the widget [created::2026-07-01] #hide ^prj
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-05 → 2026-08-20_ — 🎲 P2 · in **15** (8–30) days
  - 🛠️ **WORK LOG**
    - _2026-08-15_ — Added coverage for the parser
    - _2026-08-11_ — Sketched the migration

## Tasks

- [ ] #task Draft the API [created::2026-08-29]

## Future Work

## Requirements

- Must work offline
```

### Recognition rules

A direct child bullet is a **managed task log** when the new
`parseManagedTaskLogParentBullet()` matches it. That check replaces today's
`parseScheduleLogParentBullet()` call in the direct-child branch, still runs **before**
the section-bullet check and the task conversion, and still wins over both.

- Eight spellings qualify per kind: an optional matching emoji (`🗓️` / `🛠️`), the
  canonical (`SCHEDULE LOG` / `WORK LOG`) or legacy (`Schedule log` / `Work log`) bold
  label, and an optional colon inside the bold span. A bullet is never rewritten to the
  canonical spelling on the way over — the plugin's stated rule (comment at line 268) is
  that an existing log is never orphaned **or silently rewritten**, and the move is not
  the moment to change that.
- The emoji must agree with the label. `🗓️ **WORK LOG**` and `🛠️ **SCHEDULE LOG**` are
  not markers and stay on today's task path, mirroring bob-cli's
  `parse_managed_task_log_marker()`. Pin this with a test.
- A checkbox rules the bullet out, because the marker regexes do not match
  `- [ ] 🛠️ **WORK LOG**` at all. Such a bullet keeps today's task conversion.
- A plain `- WORK LOG` bullet (no `**`) is _not_ a marker. It stays on the existing
  path, which makes it an ALL-CAPS section bullet titled `Work Log` when it has nested
  list items. Pin this boundary with a test; do not special-case it.
- Only bullets at the **direct-child indent** move. A marker nested deeper belongs to
  the child bullet above it and is copied verbatim under that converted task, exactly as
  today.
- An empty marker with no entries still moves; the marker itself is the task's statement
  that it keeps a log.
- Any number of markers of either kind move, **in source order**, each with its own
  subtree, into one flat block. Transfer, do not normalize or reorder: the project note
  then mirrors the source task exactly, and each kind's own finder still picks its first
  marker just as it did before.

### Rendering and placement rules

Unchanged from the schedule-log move, and now shared by both kinds:

- The marker bullet is re-indented to exactly one tab; every descendant keeps its
  indentation **relative** to its marker, prefixed by that tab. A descendant whose
  leading whitespace does not start with the marker's own indent falls back to one extra
  tab.
- Blank lines are trimmed from both ends of each moved block and preserved in the
  middle.
- All moved lines are spliced in as one contiguous block immediately after the `^prj`
  task line, before `## Tasks`, in source order across kinds. They are the `^prj`
  bullet's own child block, so no blank line is added around them.

## Implementation steps

All file/line references are to `plugins/bob-navigation-hotkeys/main.js` at `b57e485`
unless stated otherwise; re-derive them if the file has moved.

### 1. Add the work-log grammar next to the schedule-log grammar

Around lines 268-310, where `SCHEDULE_LOG_*` is declared:

- Extract the parent-marker regex construction into one builder so both kinds share it
  verbatim, including the existing blockquote-tolerant indent group:

  ```js
  function buildManagedTaskLogParentRe(emoji, labels) {
    return new RegExp(
      `^(?<indent>\\s*(?:>\\s*)*)(?<marker>(?:[-*+]|\\d+[.)]))[ \\t]+(?<emoji>${emoji}[ \\t]+)?\\*\\*(?<label>${labels
        .map((label) => label.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"))
        .join("|")}):?\\*\\*[ \\t]*$`,
    );
  }
  ```

  A `function` declaration is hoisted, so it may be defined lower in the file next to
  the parsers while the `const` regexes at the top still call it.

- Add `WORK_LOG_EMOJI = "🛠️"` (`U+1F6E0 U+FE0F` — keep the variation selector),
  `WORK_LOG_LABEL = "WORK LOG"`,
  `LEGACY_WORK_LOG_LABELS = Object.freeze(new Set(["Work log"]))`, and
  `WORK_LOG_PARENT_RE`. Copy the emoji and labels byte-for-byte from
  `plugins/task-status-cycler/main.js` lines 586-591 rather than retyping them.
- Add `MANAGED_TASK_LOG_INDENT_UNIT = "\t"` and redefine
  `SCHEDULE_LOG_INDENT_UNIT = MANAGED_TASK_LOG_INDENT_UNIT` so the two never drift.
- Add `MANAGED_TASK_LOG_KIND_SCHEDULE = "schedule"` and
  `MANAGED_TASK_LOG_KIND_WORK = "work"`.
- Rebuild `SCHEDULE_LOG_PARENT_RE` through the builder; its matched text must not
  change.

Write a short comment above the pair recording that this grammar mirrors
`parse_managed_task_log_marker()` in bob-cli's `src/native/capture.rs` and must stay
byte-compatible with it, matching the comment `task-status-cycler` already carries.

### 2. Add `parseManagedTaskLogParentBullet()` and narrow the schedule parser

Replace `parseScheduleLogParentBullet()` (line 1515) with two functions:

```js
// Parse a managed task-log marker bullet of either kind. Mirrors bob-cli's
// parse_managed_task_log_marker(): a present emoji must agree with the label,
// so `🗓️ **WORK LOG**` is not a marker.
function parseManagedTaskLogParentBullet(line) {
  // try SCHEDULE_LOG_PARENT_RE, then WORK_LOG_PARENT_RE
  // → Object.freeze({ indent, marker, hasEmoji, kind })  |  null
}

function parseScheduleLogParentBullet(line) {
  const parsed = parseManagedTaskLogParentBullet(line);
  return parsed && parsed.kind === MANAGED_TASK_LOG_KIND_SCHEDULE
    ? Object.freeze({
        indent: parsed.indent,
        marker: parsed.marker,
        hasEmoji: parsed.hasEmoji,
      })
    : null;
}
```

`parseScheduleLogParentBullet()` keeps its exact current return shape and stays
kind-filtered. Every schedule-log **writer** — `findScheduleLogParent()` (line 1555),
`getScheduleLogEntryIndent()`, `formatScheduleLogParentBullet()`,
`planScheduleLogEntry()`, and the `Ctrl+Shift+P` paths — must be left untouched and must
keep calling it, so `Ctrl+Shift+P` can never append a schedule entry to a work log.

### 3. Generalize the project-creation names

Confine the rename to the promotion path. Leaving `ScheduleLog` names on code that now
carries work logs is the kind of thing that gets one kind quietly dropped again.

| Rename                                                         | To                                        |
| -------------------------------------------------------------- | ----------------------------------------- |
| `normalizeProjectScheduleLogLine()` (3663)                     | `normalizeProjectManagedLogLine()`        |
| `insertProjectScheduleLogLines()` (4296)                       | `insertProjectManagedLogLines()`          |
| seed result key `scheduleLogLines`                             | `managedLogLines`                         |
| `buildProjectContentFromTask` option `scheduleLogLines`        | `managedLogLines`                         |
| `buildProjectContentFromTask` result `scheduleLogInserted`     | `managedLogsInserted`                     |
| local `convertedScheduleLogLines` (23250)                      | `convertedManagedLogLines`                |
| accumulator `currentScheduleLog` / `flushScheduleLog()` (4071) | `currentManagedLog` / `flushManagedLog()` |

Do **not** rename anything else, in particular: the `SCHEDULE_LOG_*` writer constants,
`parseScheduleLogParentBullet`, `parseScheduleLogEntryBullet`, `findScheduleLogParent`,
`getScheduleLogEntryIndent`, `format*ScheduleLog*`, `planScheduleLogEntry`, or the
`scheduleLoggedTaskCount` / `scheduleLogCreatedParentCount` /
`scheduleLogFallbackTaskCount` planner counters. Update `helpers` exports (lines 24972,
24987-24988) and every touched doc comment, including the reference to
`normalizeProjectScheduleLogLine()` at line 3689.

### 4. Route both kinds in `buildProjectSeedFromChildBullets()`

Lines 3950-4192:

- In the direct-child branch (line 4111), swap `parseScheduleLogParentBullet(line)` for
  `parseManagedTaskLogParentBullet(line)`. Nothing else about that branch changes: it
  still runs before the section check and before
  `buildProjectTaskLineFromChildBullet()`, still opens the accumulator with the bullet's
  own leading indent as `markerIndent`, and still `continue`s.
- The nested arm (line 4164) and the blank-line arm (line 4097) need only the
  accumulator rename.
- `flushManagedLog()` keeps appending each trimmed block to the single `managedLogLines`
  array, so multiple markers of either kind concatenate in source order.
- Lines consumed this way are represented and must **not** set `lossless = false`.
- Update the function's doc comment (lines 3935-3949): it is the contract other readers
  rely on, and it currently says "managed schedule-log child lines".

### 5. Thread it through `buildProjectContentFromTask()` and the caller

- `buildProjectContentFromTask()` (4579): read `options.managedLogLines`, call
  `insertProjectManagedLogLines()`, and return `managedLogsInserted` from both the early
  `!seeded` return (`false`) and the success return. Placement in the function is
  unchanged — the `^prj` child block is spliced before
  `replaceProjectTasksPlaceholder()` and `insertProjectSectionNotes()`, both of which
  re-derive their own indices.
- `createProjectNoteFromTask()` (23189): rename the local, keep the widened acceptance
  gate (`... || conversion.managedLogLines.length > 0`) so a task whose only child is a
  log still seeds instead of falling into `childConversionLossy`, and pass
  `managedLogLines` through.
- Reword the guard at line 23352 to cover both kinds:
  `"Created project, but the task logs could not be added; source task was kept"`.
- Change `getProjectFromTaskNoticeText()`'s sixth parameter (line 4746) from the boolean
  `scheduleLogMoved` to `movedLogKinds = []`, an array or Set of kind strings. Render at
  most two details in a **fixed** order — `schedule log moved`, then `work log moved` —
  regardless of source order, de-duplicating and ignoring unknown values, and appending
  after the section chip. This is a breaking signature change for one call site (line
  23422, which should pass the distinct kinds present in `convertedManagedLogLines`) and
  two existing assertions; there are no other callers.

### 6. Tests

Add `node:test` cases to `scripts/test-navigation-hotkeys.cjs` in the existing
`helpers.<fn>(...)` + `assert.deepEqual` style, beside the project-seed tests at lines
12200-12400 and the content/notice tests at 12650-12800. First rename `scheduleLogLines`
→ `managedLogLines`, `scheduleLogInserted` → `managedLogsInserted`, and the two helper
names in the existing tests; then add:

- A direct-child `🛠️ **WORK LOG**` with two entries lands in `managedLogLines` as
  `["\t- 🛠️ **WORK LOG**", "\t\t- *…* — …", "\t\t- *…* — …"]`, is absent from
  `taskLines`, and leaves `lossless: true`.
- The emoji-less `**WORK LOG**` and legacy `**Work log:**` markers move too, with their
  spelling preserved byte-for-byte.
- `- [ ] 🛠️ **WORK LOG**` (checkbox) still converts to a task.
- A plain `- WORK LOG` bullet with sub-bullets still becomes a `Work Log` section.
- A work-log marker nested under a converted child task stays under that task,
  unchanged.
- An empty work-log marker moves on its own; and a source task whose **only** child is a
  work log yields empty `taskLines`/`sections`, populated `managedLogLines`, and
  `lossless: true`.
- A source task carrying **both** a schedule log and a work log moves both, in source
  order, into one `managedLogLines` array — assert with the work log written first, so
  the test would fail if the implementation sorted by kind.
- One mixed child block routes tasks, an ALL-CAPS section, a schedule log, and a work
  log to their destinations in a single pass.
- `parseManagedTaskLogParentBullet()` returns the right `kind` for all sixteen accepted
  spellings, returns `null` for trailing text after the marker, and accepts a
  blockquote-prefixed marker.
- **Kind-mismatch pin:** `🗓️ **WORK LOG**` and `🛠️ **SCHEDULE LOG**` are not markers —
  `parseManagedTaskLogParentBullet()` returns `null` and
  `buildProjectSeedFromChildBullets()` leaves them on the task path. Reference bob-cli's
  `parse_managed_task_log_marker()` in the test name or a comment.
- **Writer-isolation pin:** `parseScheduleLogParentBullet("\t- 🛠️ **WORK LOG**")` is
  `null`, and `findScheduleLogParent()` on a task carrying both markers returns the
  schedule marker's line, not the work-log marker's.
- `buildProjectContentFromTask()` on the real template content, given both logs plus
  `childTaskLines` and a section, splices the whole log block directly under the `^prj`
  line and above `## Tasks` with `managedLogsInserted: true`; assert the entire rendered
  note.
- `buildProjectContentFromTask()` on a note whose completion placeholder is not on a
  `^prj` line returns `managedLogsInserted: false` and is otherwise seeded normally.
- `getProjectFromTaskNoticeText()` renders `schedule log moved` alone, `work log moved`
  alone, both in the fixed order from a reversed input array, and neither for `[]`.

Run from the `bob-plugins` checkout:

```bash
npm test
npm run validate
```

### 7. Version and README

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.31.1` to `1.32.0`.
  Versions are per-plugin; touch no other manifest.
- Update the `Bob Navigation Hotkeys` row in the `bob-plugins` `README.md` plugin table
  (version column and description). The row currently reads "a source task's managed
  schedule log moves under the new project's `^prj` task"; widen it to the source task's
  managed schedule **and work** logs, in source order, matching the row's existing
  run-on style.

### 8. Docs in this `bob-cli` checkout

Rewrite the `docs/projects.md` paragraph at lines 250-256 that currently covers only the
schedule log. It should say that a managed source-task log of **either** kind — schedule
(`🗓️ **SCHEDULE LOG**`, `**SCHEDULE LOG**`, legacy `**Schedule log:**`) or work
(`🛠️ **WORK LOG**`, `**WORK LOG**`, legacy `**Work log:**`) — moves with the source
task, lands as a direct child of the new project's `^prj` task rather than a `## Tasks`
task line, preserves its marker spelling and nested entries, and keeps source order when
both are present; and that later `Ctrl+Shift+P` and `Ctrl+Shift+Enter` edits on the
`^prj` task continue appending to the matching log. Keep the file's existing
line-wrapping width. The reversal paragraph at lines 243-249 already says "including
managed schedule and work logs" and needs no change.

This is a documentation-only change to the Rust project; `just all` is not required for
it, though it is cheap insurance if anything else in the checkout was touched.

### 9. Deploy to the vault

`bob-plugins/AGENTS.md` requires a sync after any change to that repo. Run from the
`bob-plugins` checkout path that `/sase_repo` printed. `-r "$PWD"` is required because
the default source path does not exist in a SASE workspace, and `--no-pull` avoids a
pull against the dirty checkout:

```bash
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull --dry-run
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull
```

If sync reports the vault file as dirty and suggests `-F/--force`, do **not** force
blindly: compare the deployed file against the vault repo's committed baseline first and
report the discrepancy.

After deploying, tell the user to reload the plugin in Obsidian; a running Obsidian will
not pick up the new `main.js` on its own.

## Verification

1. `npm test` and `npm run validate` pass in the `bob-plugins` checkout.
2. `bob plugins sync … --dry-run` shows only `bob-navigation-hotkeys` `main.js` +
   `manifest.json` changing.
3. Manual check in Obsidian after a plugin reload, from an area note, on a source task
   shaped like the "The bug" example: the new project note's `^prj` task carries both
   the `🗓️ **SCHEDULE LOG**` and `🛠️ **WORK LOG**` bullets with all entries intact and
   in source order, `## Tasks` holds only `Draft the API`, `## Requirements` gains
   `Must work offline`, and the source block is gone from the area note.
4. In that new project note, `Ctrl+Shift+P` on the `^prj` task with a reason appends to
   the moved schedule log rather than creating a second marker or writing into the work
   log. Then set `^prj` to `[/]` and `Ctrl+Shift+Enter` it back to open with a work
   summary: the entry appends to the moved work log.
5. `bob projects list` on the new project reports the expected open-task count — the
   work log no longer shows up as a task.
6. Round-trip: `Ctrl+Shift+Alt+N` on that `^prj` task restores the parent task with both
   logs back among its children.

## Out of scope

- The managed `🔗 **DEPENDS ON:**` bullet, which has the same problem for the same
  reason. Moving it correctly also means deciding what its `[[#^block-id]]` links should
  point at once the source block ID is rewritten to `^prj`, which is a separate design
  question. Leave it on today's task-conversion path and mention nothing about it in the
  docs.
- Normalizing a legacy or emoji-less marker to its canonical spelling during the move,
  and merging two markers of the same kind into one.
- `task-status-cycler`'s own promotion and demotion of bullets between a project's
  `## Tasks` section and a parent-task block. It may have the same gap, but it is a
  different keymap and a different plugin; file it as a follow-up task bead if the
  implementer confirms it.
- Any Rust change beyond the `docs/projects.md` edit. `bob capture-task-sections` and
  `bob capture` already skip both marker kinds through
  `parse_managed_task_log_marker()`, and `bob projects sync` already tolerates a log
  under `^prj`: it is not a `#task` line, so it does not affect counts, and the
  `🧩 **Sub-projects:**` writer only inspects marker lines.
- Editing `_templates/new_project.md`.
- The reversal path's ordering, which places the restored `^prj` sub-bullets after the
  restored tasks and section bullets. That is pre-existing behavior for schedule logs
  and is not a regression here.
- The plain `Ctrl+Shift+N` "Create project note" command, which has no source task.
