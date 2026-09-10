---
tier: tale
title: Reverse Ctrl+Shift+Alt+N to convert a project note back into a parent task
goal:
  Pressing `Ctrl+Shift+Alt+N` while the cursor sits on a project note's `^prj` lifecycle
  task converts that note back into a single Obsidian task in the parent note's `##
  Tasks` section — carrying its tasks, sections, and managed logs back as child bullets,
  repointing `#^prj` block links, and trashing the note — or fails with a specific
  notice when the project note is not in the expected format.
size: medium
proposed_by: bbugyi200.athena.04s
create_time: 2026-09-09 20:00:23
status: wip
---

# Plan: Convert a project note back into a parent task

## Context

`Ctrl+Shift+Alt+N` is bound in the vault (`.obsidian/hotkeys.json`) to
`bob-navigation-hotkeys:create-project-note-from-task`. That binding already exists and
needs **no vault change** — this plan adds the reverse direction inside the same
command.

All code changes except the one documentation edit in step 8 live in the **`bob-plugins`
linked repo**. Open it with the `/sase_repo` skill before touching it:

```bash
sase repo open bob-plugins -r "Implement the reverse project-note-to-task conversion"
```

Use only the path that command prints. These are plain CommonJS Obsidian plugins:
`plugins/bob-navigation-hotkeys/main.js` is the source, there is no build step. Note
that `main.js` contains emoji, so `grep` reports it as a binary file — use `grep -a`.

### How the forward command works today

`BobNavigationHotkeysPlugin.createProjectNoteFromTask(editor, view)` (around line
20489):

1. Requires a markdown area-or-project note and a cursor line that
   `parseProjectSourceTaskLine()` accepts (open status, `#task`, nonempty description).
   That parser strips a trailing `^blockId`, a `[p::N]` field, and a `[scheduled::DATE]`
   field out of the description.
2. `getProjectSourceTaskBlock()` captures the task line plus its contiguous indented
   child block.
3. `buildProjectSeedFromChildBullets()` splits that child block into `taskLines` (for
   `## Tasks`), `sections` (`[{ title, noteLines }]` for `##` headers), and
   `scheduleLogLines` (re-indented for insertion under `^prj`).
4. `createProjectNoteFile()` instantiates `_templates/new_project.md` via Templater and
   `applyProjectCreationFrontmatter()` writes `parent`, `type`, `status: wip`, and
   `scheduled`.
5. `buildProjectContentFromTask()` fills the `^prj` completion placeholder, inserts the
   schedule log under `^prj`, replaces the `## Tasks` placeholder, and inserts sections.
6. When the source task had a block ID, the new note's basename is
   `getProjectBasenameFromTaskBlockId(sourceBasename, blockId)` =
   `<parent>_<id with - replaced by _>`, and `applyBlockIdLinkRewrites()` repoints every
   inbound `[[<parent>#^<id>]]` link at `[[<project>#^prj]]` via
   `rewriteBlockIdLinkOriginal()`.
7. The source block is removed with `removeTaskBlockFromContent()` and
   `getProjectFromTaskNoticeText()` reports the result.

The vault template produces this shape (`_templates/new_project.md`):

```markdown
---
parent: "[[Health]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: 2026-08-01T09:12:00-04:00
---

- [ ] #task #prj (REPLACE WITH PROJECT COMPLETION CRITERIA) #hide ^prj

## Tasks

- [ ] #task (REPLACE WITH TASK DESCRIPTION) [created::2026-08-01]

## Requirements
```

### What is missing

There is no way to undo the conversion. Worse, pressing the hotkey on a project note's
`^prj` line today _succeeds_ at the wrong thing: the `^prj` line parses as a valid open
`#task`, so the command creates a nested child project from the lifecycle task. The
reverse operation replaces that behavior.

## Desired behavior

### Worked example

`Health_gym_habit.md`:

```markdown
---
parent: "[[Health]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: 2026-08-01T09:12:00-04:00
scheduled: 2026-09-01
---

- [ ] #task #prj Build a gym habit [p::1] #hide ^prj
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-05 — 2026-08-01 → 2026-09-01 — 🎲 auto_

## Tasks

- [ ] #task Buy shoes [created::2026-08-01]
  - Check the outlet store
- [x] #task Sign up [created::2026-08-01]

## Requirements

- Must be within walking distance

## Open Questions

- Morning or evening?
```

With the cursor on the `^prj` line, `Ctrl+Shift+Alt+N` appends this block to the
`## Tasks` section of `Health.md` and trashes `Health_gym_habit.md`:

```markdown
- [ ] #task Build a gym habit [p::1] [scheduled::2026-09-01] [created::2026-08-01]
      ^gym-habit
  - [ ] #task Buy shoes [created::2026-08-01]
    - Check the outlet store
  - [x] #task Sign up [created::2026-08-01]
  - REQUIREMENTS
    - Must be within walking distance
  - OPEN QUESTIONS
    - Morning or evening?
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-05 — 2026-08-01 → 2026-09-01 — 🎲 auto_
```

Pressing `Ctrl+Shift+Alt+N` on that restored task re-creates a project note, so the two
directions compose.

### Dispatch

Inside `createProjectNoteFromTask()`, immediately after the existing `lineText === null`
guard (around line 20515, where `cursor` and `lineText` are already resolved) and
**before** `parseProjectSourceTaskLine()`:

```js
if (
  isProjectLifecycleTaskLine(lineText) &&
  isProjectLifecycleTaskAtLine(editor.getValue(), cursor.line)
) {
  return this.convertProjectNoteToTask(editor, view, cursor, lineText);
}
```

Placing the check after the existing `isMarkdownFile` / `isAreaOrProjectNote` / cursor
guards leaves every forward-path notice and its ordering untouched.
`isProjectLifecycleTaskAtLine()` (line 9566) rejects a `^prj` example inside frontmatter
or a fenced block. Dispatch deliberately keys on the `^prj` line **alone**, not on
`type: [[project]]`: a `^prj` line in a note that is not a project note is a mistake,
and the reverse path reports it precisely instead of silently creating a nested project.

### Expected format and failure notices

`convertProjectNoteToTask()` validates in this order and shows exactly one notice, then
returns `false` without touching any file. Every check has a distinct message:

| Condition                                                                                                                                                           | Notice                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `vault.process`/`vault.read` unavailable                                                                                                                            | `Vault content updates are unavailable` (existing string)                                                      |
| no `view.save`, or the save throws                                                                                                                                  | `Could not save project note`                                                                                  |
| `parseProjectNoteFrontmatter()` rejects the note (no frontmatter, unclosed, malformed, not a mapping, `type` is not `[[project]]`, multiple or invalid `scheduled`) | that helper's own `error` text, which is already notice-shaped (e.g. `The ^prj task is not in a project note`) |
| more than one `^prj` line outside frontmatter and fences                                                                                                            | `Project note has multiple ^prj tasks`                                                                         |
| `^prj` status not in `PROJECT_OPEN_TASK_STATUSES` (`" "`, `"/"`, `"*"`)                                                                                             | `Only open projects can be converted back to a task`                                                           |
| completion criteria still contains `PROJECT_COMPLETION_PLACEHOLDER`                                                                                                 | `Project completion criteria is still the template placeholder`                                                |
| completion criteria empty after stripping tags                                                                                                                      | `Project completion criteria is empty`                                                                         |
| frontmatter `parent` missing or not a link                                                                                                                          | `Project note has no parent link`                                                                              |
| `parent` does not resolve to a markdown file                                                                                                                        | `Parent note "X" not found`                                                                                    |
| `parent` resolves to the project note itself                                                                                                                        | `Project note parent points at itself`                                                                         |
| a nonblank line outside the `^prj` block and outside every `##` section                                                                                             | `Project note has content outside the ^prj task and its sections: "SNIPPET"`                                   |
| a nonblank, non-list line inside a `##` section (paragraph, `###` header, fenced block)                                                                             | `Section "X" has content that is not a list item: "SNIPPET"`                                                   |
| a nonempty section whose title cannot become an ALL-CAPS bullet                                                                                                     | `Section "X" cannot be converted into a task bullet`                                                           |
| `collectChildNotes()` finds notes whose `parent` resolves to this note                                                                                              | `Project has N child notes; move them before converting`                                                       |
| inbound `#^prj` links exist but no valid block ID can be derived                                                                                                    | `Could not derive a task block ID for the links that point at ^prj`                                            |
| derived block ID already appears in the parent note                                                                                                                 | `Parent note already contains block ID: X`                                                                     |
| any other inbound reference from a note other than the parent                                                                                                       | `N notes link to "X"; update them before converting (first: PATH)`                                             |
| `parseTaskMoveDestinationFrontmatter()` rejects the parent (not an area, or a project that is not `wip`/`waiting`)                                                  | `Parent note "X" must be an area or open project`                                                              |
| `insertTaskMoveBlocks()` cannot place the block                                                                                                                     | `Parent note "X" has no ## Tasks section`                                                                      |

`SNIPPET` uses `truncateProjectTaskDescription()` on the trimmed line.

Failures **after** the first write report what happened and stop, leaving a recoverable
state rather than attempting a rollback (the vault is a git repo):

| Condition                                     | Notice                                                                                                                            | Return  |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------- |
| parent content changed between read and write | `Parent note changed; nothing was converted`                                                                                      | `false` |
| `vault.process` on the parent throws          | `Could not update the parent note`                                                                                                | `false` |
| some `#^prj` link rewrites failed             | `Restored the task in PARENT, but N links could not be updated; "NOTE" was kept` (skip trashing)                                  | `true`  |
| `fileManager.trashFile` throws                | `Restored the task in PARENT, but could not delete "NOTE"`                                                                        | `true`  |
| success                                       | `getProjectNoteToTaskNoticeText()`, e.g. `Converted Health_gym_habit into a task in Health (2 tasks; 2 sections; 1 link updated)` | `true`  |

### Reconstruction rules

**The task line.** Take the `^prj` line's status character and body. Strip the trailing
`^prj` block ID, then remove every whole-token `#task`, `#prj`, and `#hide` tag, then
collapse whitespace with `collapseProjectTaskDescription()`. Everything else in the body
survives in place, including `[p::N]` and any `[id::...]` field. Render
`- [<status>] #task <description>`, then:

- Append `[scheduled::<frontmatter scheduled>]` when frontmatter has a valid `scheduled`
  and the description has no `scheduled` field of its own. This is the exact inverse of
  the forward transfer, which moved the field into frontmatter.
- Append `[created::<date>]` when the description has no `created` field, using the
  `YYYY-MM-DD` prefix of the raw frontmatter `created:` line, falling back to
  `formatProjectTaskCreatedDate(new Date())` when that value is missing or malformed.
  Read the raw line text rather than the parsed YAML value, which Obsidian's `parseYaml`
  may turn into a `Date` and shift across time zones.
- Append fields with `getBulletPropertyAppendIndex()` exactly as
  `buildProjectTaskLineFromChildBullet()` does, then append ` ^<blockId>` last when a
  block ID was derived.

Preserving `[id::...]` verbatim is deliberate: the Obsidian Tasks plugin matches
`[dependsOn::]` values against `[id::]` values as opaque strings, so every existing
dependency on the project's lifecycle task keeps resolving without a vault-wide rewrite.

**The block ID.** Let `suffix` be the note basename with a leading `<parentBasename>_`
removed when present, otherwise the whole basename. Replace `_` with `-`. Accept the
result only when it matches `PROJECT_BLOCK_ID_RE` (`/^[A-Za-z0-9-]+$/`); otherwise there
is no block ID. This inverts `getProjectBasenameFromTaskBlockId()` exactly, because
block IDs cannot contain `_`. A renamed note (`Ship It Project`) yields no block ID and
still converts — the restored task simply has none — unless inbound `#^prj` links need a
target, which is the one case that fails.

**Child bullets**, in this order:

1. Every `## Tasks` list item (skipping a line containing `PROJECT_TASKS_PLACEHOLDER`),
   re-indented one Tab deeper.
2. Every other nonempty `##` section, in document order: one `- <TITLE>` bullet where
   `TITLE` is the header text trimmed, whitespace-collapsed, and uppercased, followed by
   that section's list items re-indented two Tabs deeper.
3. The `^prj` line's own child subtree, re-indented one Tab deeper.

The `^prj` subtree goes **last** so managed `🗓️ **SCHEDULE LOG**` and `🛠️ **WORK LOG**`
children stay the final sub-bullets of the task, matching the vault convention that
`bob capture` inserts new sub-bullets before managed logs. Everything in that subtree
moves verbatim, including user bullets and any stale `- 🧩 **Sub-projects:**` line;
`bob projects sync` only manages that line under a `^prj` task, so it becomes inert user
content under an ordinary task.

Re-indentation preserves relative depth: for each block, the base indent is the
shallowest nonblank indent in it, and each line becomes `"\t".repeat(depth)` plus its
indent relative to that base. Blank lines are dropped from the restored block so it is
one contiguous subtree; blank lines are formatting, not content.

An uppercased title must still satisfy `PROJECT_SECTION_TITLE_RE` and contain a letter,
so `Future Work` → `FUTURE WORK` and `Api Design` → `API DESIGN` round-trip through the
forward converter, while `Q&A: notes!` is rejected rather than silently producing a
bullet the forward direction would treat as a task. Empty sections (including the
template's bare `## Requirements`) produce no bullet at all. A `##` header whose
normalized title is `tasks` is the task section, not a note section.

Frontmatter is discarded apart from `parent`, `scheduled`, and `created`; `type`,
`status`, `template`, and any user-added keys are dropped with the note.

### Parent insertion, links, and deletion

Reuse the `Ctrl+Shift+M` "move tasks to note" machinery, which already solves this exact
insertion problem:

- `parseTaskMoveDestinationFrontmatter(parentContent)` decides `kind` (`area` or open
  `project`) and rejects closed projects.
- `collectTaskMoveBlockIds(parentContent)` powers the block-ID collision check.
- `insertTaskMoveBlocks(parentContent, [taskLines], kind)` replaces a lone `## Tasks`
  placeholder, otherwise appends after the section's last nonblank line, and creates a
  `## Tasks` section at EOF for an area note that has none. It returns `insertedLine`.
- `focusTaskMoveDestination(parentFile, { line, text, blockId })` opens the parent and
  puts the cursor on the restored task, retrying across frames.

Inbound references come from `metadataCache.getBacklinksForFile(projectFile)`:

- Originals that target `#^prj` are rewritten to `[[<parentBasename>#^<blockId>]]` (or
  `(<parentBasename>.md#^<blockId>)` for markdown links) — the exact inverse of the
  forward rewrite, through the same `rewriteBlockIdLinkOriginal()` and
  `applyBlockIdLinkRewrites()` code paths.
- Any other reference (plain wikilink, embed, heading link) **from a note other than the
  parent** blocks the conversion, because trashing the note would break it.
- References from the parent note are ignored: the machine-owned
  `- 🧩 **Sub-projects:**` ledger line always links to its children and is pruned by the
  next `bob projects sync`, and a hand-written parent link sits right where the restored
  task lands.

Write order, with no rollback: parent note, then link rewrites, then
`focusTaskMoveDestination()`, then `fileManager.trashFile(projectFile)` (trash, not
permanent delete, matching `deleteCurrentFile()`). Focusing before trashing means the
leaf has already moved off the project note when it disappears.

## Implementation steps

All steps are in `plugins/bob-navigation-hotkeys/main.js` unless stated otherwise.
Export every new pure helper from the `helpers` object (around line 21961) so tests can
reach it.

### 1. Add constants and small helpers

Near the other `PROJECT_*` constants (lines 72-101):

- `PROJECT_LIFECYCLE_TAG_GLOBAL_RE = /(^|[\s([{])#(?:prj|hide)(?=$|[\s)\]},.;:!?])/g` —
  whole-token `#prj`/`#hide` stripping, mirroring `PROJECT_TASK_TAG_GLOBAL_RE`.

Near the project helpers (lines 3288-3700):

- `indentProjectReversalLine(lineText, baseIndent, depth)` — returns `""` for a blank
  line, otherwise `"\t".repeat(depth)` plus the line's indent relative to `baseIndent`
  (falling back to `"\t"` when the indent does not extend the base) plus its content.
  Model it on `normalizeProjectScheduleLogLine()`; leave that helper alone so the
  forward path is untouched.
- `getProjectReversalBlockId(noteBasename, parentBasename)` — the block-ID derivation
  above, or `null`.
- `formatProjectReversalSectionTitle(headerText)` — trimmed, whitespace-collapsed,
  uppercased title, or `null` when it fails `PROJECT_SECTION_TITLE_RE` / has no letter.
- `parseProjectLifecycleTaskBody(lineText)` — returns `{ status, description }` for a
  `^prj` line, or `null` when the line is not one. Strips the trailing block ID and the
  `#task`/`#prj`/`#hide` tokens, then applies `collapseProjectTaskDescription()`.
- `getProjectFrontmatterCreatedDate(lines, closingLine)` — the `YYYY-MM-DD` prefix of
  the raw `created:` frontmatter line, or `""`.
- `buildTaskLineFromProjectNote({ status, description, scheduled, created, blockId })` —
  renders the task line per the rules above.

### 2. Add the note walker

`splitProjectNoteForReversal(content)` → frozen
`{ valid, error, status, description, lifecycleChildLines, taskLines, sections }` where
`sections` is `[{ title, noteLines }]` in document order and `error` is one of the
structure messages from the table.

Walk `splitMarkdownContent(content).lines` with `getMarkdownLineContexts(content)` so
frontmatter and fenced regions are classified rather than parsed as structure:

- Skip frontmatter lines.
- The `^prj` line is found with `isProjectLifecycleTaskLine()` on non-frontmatter,
  non-fence lines; a second match is the multiple-`^prj` error.
- Its child block is the following run of lines that are blank or indented deeper than
  it, trimmed of trailing blanks — the same rule `getProjectSourceTaskBlock()` uses,
  applied to an array instead of an editor.
- `PROJECT_SECTION_HEADER_RE` (level-2 only, non-fence) opens a section. Its body runs
  until the next `PROJECT_SECTION_BOUNDARY_HEADER_RE` match or EOF. Inside a body, blank
  lines are ignored, `PROJECT_LIST_ITEM_RE` lines are collected, and anything else is
  the non-list error — including fenced lines, which the context array marks.
- A nonblank line that is neither the `^prj` block nor inside a section is the
  content-outside error. A header that is not level 2 hits the same error.
- Bodies whose normalized title is `tasks` collect into `taskLines` with placeholder
  lines dropped; other nonempty bodies become sections via
  `formatProjectReversalSectionTitle()`.

Return the raw collected lines; step 3 does the re-indentation.

### 3. Build the restored block

`buildTaskBlockFromProjectNote(content, options)` → frozen
`{ valid, error, lines, taskCount, sectionCount, blockId, scheduled, created }` where
`options` carries `noteBasename`, `parentBasename`, and `now`.

It calls `parseProjectNoteFrontmatter()` (reusing its error text), enforces the
open-status / placeholder / empty-description rules, calls
`splitProjectNoteForReversal()`, derives the block ID, and assembles `lines`: the task
line, then tasks at depth 1, then each section as `- TITLE` at depth 1 with its notes at
depth 2, then the `^prj` children at depth 1. `taskCount` counts top-level restored task
bullets (those at depth 1 that `parseProjectChildListItem()` reports as list items),
`sectionCount` counts emitted sections.

Keep `parent` resolution out of this function — it needs the vault. Return
`parentLink: data.parent` so the caller can resolve it.

### 4. Parameterize the link rewriter

- `rewriteBlockIdLinkOriginal(original, newBasename, blockId = "prj")` — replace the two
  hardcoded `#^prj` occurrences with the argument. Existing forward callers keep working
  unchanged.
- `applyBlockIdLinkRewrites(rewrites, newBasename, blockId = "prj")` — pass the argument
  through.
- `collectProjectNoteBacklinkClassification(backlinksData, excludePath)` → frozen
  `{ blockRewrites, otherPaths }`. `blockRewrites` is
  `collectBlockIdBacklinkRewrites( backlinksData, "prj")`. `otherPaths` lists every
  other path with at least one original that is not in that file's `#^prj` set,
  excluding `excludePath` (the parent) and the project note itself. Walk the nested
  backlink structure the same way `collectBlockIdBacklinkOriginals()` does, with the
  same depth cap.

### 5. Add the plugin method

`async convertProjectNoteToTask(editor, view, cursor, lineText)`:

1. Guard `vault.process`, `vault.read`, and `view.save`; save the view.
2. `const content = editor.getValue()`; run `buildTaskBlockFromProjectNote()`; on
   failure show its `error` and return `false`.
3. Resolve `parentLink` with `resolveLinkTargetFile(link, sourceFile.path)` (after
   `extractLinkTarget()`); reject a missing target and a self-reference.
4. `collectChildNotes(sourceFile)` must be empty.
5. Read the parent with `vault.read`; run `parseTaskMoveDestinationFrontmatter()`, the
   block-ID collision check against `collectTaskMoveBlockIds()`, and
   `insertTaskMoveBlocks()`.
6. Classify backlinks through a thin
   `getProjectNoteBacklinkClassification(file, parentPath)` method that wraps
   `metadataCache.getBacklinksForFile()` in try/catch exactly like
   `getProjectTaskBlockIdBacklinkRewrites()`; reject `otherPaths.length > 0`; reject
   `blockRewrites.length > 0` with no derived block ID.
7. `vault.process(parentFile, (current) => current === parentContent ? insertion.content : current)`,
   detecting the unchanged case to report `Parent note changed; nothing was converted`.
8. `applyBlockIdLinkRewrites(blockRewrites, parentBasename, blockId)`; on `failed`,
   notice and return `true` **without** trashing.
9. `focusTaskMoveDestination(parentFile, { line: insertion.insertedLine, text: lines[0], blockId })`,
   then `fileManager.trashFile(sourceFile)`.
10. Notice via
    `getProjectNoteToTaskNoticeText(noteBasename, parentBasename, taskCount, sectionCount, updatedLinkCount)`,
    built like `getProjectFromTaskNoticeText()` with `;`-joined detail chips and
    singular/plural words. Return `true`.

### 6. Add the notice helper

`getProjectNoteToTaskNoticeText(noteBasename, parentBasename, taskCount, sectionCount, updatedLinkCount)`
→ `Converted NOTE into a task in PARENT (2 tasks; 2 sections; 1 link updated)`. Omit
each chip at zero; render `(no child content)` when all three are zero.

### 7. Tests

Add `node:test` cases to `scripts/test-navigation-hotkeys.cjs` in the existing
`helpers.<fn>(...)` + `assert.deepEqual` style, plus command-level tests that build a
`NavigationHotkeysPlugin` with a stub `app` (the `setProjectNoteScheduledValue` tests
around line 2609 are the model; `notices` collects toasts).

Pure-helper coverage:

- The worked example above round-trips: assert the exact restored block, then feed it
  back through `buildProjectSeedFromChildBullets()` and assert the tasks, sections, and
  schedule log come back.
- Task line: status preserved (`[ ]`, `[/]`, `[*]`); `#task`/`#prj`/`#hide` stripped
  while `#hidden` survives; `[p::1]` and `[id::Health_gym_habit__prj]` preserved
  verbatim; `[scheduled::]` added from frontmatter and skipped when the body already has
  one; `[created::]` taken from the raw frontmatter line, and today's date when it is
  missing or malformed.
- Block ID: `Health_gym_habit` under `Health` → `gym-habit`; a basename with no parent
  prefix falls back to the whole basename; a basename with a space yields `null`.
- Sections: `Future Work` → `FUTURE WORK`; `Api Design` → `API DESIGN`; empty sections
  emit nothing; `## TASKS` is treated as the task section; two sections keep document
  order; `Q&A: notes!` is rejected.
- Re-indentation: nested depth preserved relative to each block's shallowest line; blank
  lines dropped; tab and two-space source indents both normalize.
- Structure errors: prose between `^prj` and `## Tasks`; a `###` header inside a
  section; a fenced block inside a section; a `#` level-1 header; two `^prj` lines; a
  `[x]` `^prj` task; the unreplaced completion placeholder; frontmatter without
  `type: [[project]]`.
- CRLF content round-trips (`splitMarkdownContent` detects `\r\n`; the joined output
  keeps it).
- `rewriteBlockIdLinkOriginal("[[Health_gym_habit#^prj]]", "Health", "gym-habit")` →
  `[[Health#^gym-habit]]`; the alias, embed, and markdown-link forms; the default
  `blockId` argument keeps the forward result.
- `collectProjectNoteBacklinkClassification()` separates `#^prj` originals from other
  references and excludes the parent path.

Command-level coverage:

- Happy path: the parent's `## Tasks` placeholder is replaced by the restored block, the
  `#^prj` backlink is rewritten, `trashFile` is called once, and the notice matches
  `/Converted .* into a task in Health/`.
- Dispatch: with the cursor on an ordinary `#task` line the forward path still runs;
  with the cursor on `^prj` the reverse path runs; a `^prj` example inside a fenced
  block falls through to the forward path.
- Refusals leave every file untouched and `trashFile` uncalled: missing parent link,
  unresolvable parent, closed parent project, parent with no `## Tasks` section,
  existing child note, colliding block ID, extra inbound link from a third note.
- Post-write failures: a throwing `trashFile` reports `/could not delete/` and still
  returns `true`; a failed link rewrite reports `/links could not be updated/` and skips
  `trashFile`.

Run from the `bob-plugins` checkout:

```bash
npm test
npm run validate
```

### 8. Docs, version, and deploy

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.26.0` to `1.27.0`.
  Versions are per-plugin; touch no other manifest.
- Extend the `Bob Navigation Hotkeys` row in the `bob-plugins` `README.md` plugin table
  (line 16) with the reverse direction, matching that row's run-on style, and update the
  version cell.
- In the **`bob-cli` checkout**, add a paragraph to `docs/projects.md` after the
  section-bullet paragraph that ends near line 214. Cover: the same hotkey on a project
  note's `^prj` task restores the task in the parent's `## Tasks` section; what carries
  back (tasks, uppercased section bullets, `^prj` sub-bullets including managed logs,
  `scheduled` and `created` as inline fields, a block ID derived from the note name);
  that `#^prj` links are repointed and the note is trashed; and that anything outside
  the expected format fails with a notice and changes nothing. Keep the file's line
  wrapping.
- Deploy to the vault, as `bob-plugins/AGENTS.md` requires. Run from the `bob-plugins`
  checkout that `/sase_repo` printed. `-r "$PWD"` is required because the default source
  path does not exist in a SASE workspace, and `--no-pull` avoids a pull against the
  dirty checkout:

  ```bash
  bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull --dry-run
  bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull
  ```

  If sync reports the vault file as dirty and suggests `-F/--force`, do **not** force
  blindly: compare the deployed file against the vault repo's committed baseline first
  and report the discrepancy.

- After deploying, tell the user to reload the plugin in Obsidian; a running Obsidian
  will not pick up the new `main.js` on its own. No `.obsidian/hotkeys.json` change is
  needed.

## Verification

1. `npm test` and `npm run validate` pass in the `bob-plugins` checkout.
2. `bob plugins sync --dry-run` shows only `bob-navigation-hotkeys` `main.js` and
   `manifest.json` changing.
3. Manual check in Obsidian after a plugin reload:
   - Build the worked example: from an area note, run `Ctrl+Shift+Alt+N` on a task with
     child tasks, an ALL-CAPS section bullet, and a schedule log. Then run
     `Ctrl+Shift+Alt+N` on the new note's `^prj` line and confirm the task returns to
     the area's `## Tasks` section with the block shown above, the cursor lands on it,
     and the project note is in the trash.
   - Repeat with a parent that is an open project note and confirm the restored task
     lands in that note's `## Tasks` section.
   - Add a stray paragraph to a project note and confirm the notice names it and nothing
     changes.
   - Confirm `bob projects sync` afterwards reports no errors for the parent and prunes
     the converted child from any `- 🧩 **Sub-projects:**` line.

## Out of scope

- Rewriting plain inbound `[[project]]` links, embeds, and heading links. The reverse
  operation refuses instead; relaxing that into a rewrite is a follow-up.
- Vault-wide `[dependsOn::]`/`[id::]` canonicalization. Preserving the `^prj` line's
  `[id::]` value verbatim keeps existing dependencies resolving, and a later
  `Ctrl+Shift+M` move canonicalizes it.
- Rollback of partial writes. The three post-validation failure modes each report a
  specific, recoverable state.
- A separate command ID for the reverse direction. The user asked for the existing
  hotkey to reverse itself, and one entry point keeps the two directions from drifting.
- Converting closed (`[x]`/`[-]`) projects, and any change to `bob projects sync`
  (`src/native/projects.rs`) or to `_templates/new_project.md`.
- The plain `Ctrl+Shift+N` "Create project note" command.
