---
tier: tale
title: Convert ALL-CAPS child bullets into project note sections
goal:
  The `Ctrl+Shift+Alt+N` "Create project note from task" command routes each ALL-CAPS
  direct child bullet that has its own sub-bullets into a title-cased `##` section of
  the new project note — reusing a section the template already has, or appending a new
  one — with that bullet's descendants copied in verbatim as notes instead of converted
  into `#task` lines.
size: medium
proposed_by: bbugyi200.athena.yu
create_time: 2026-09-09 20:00:24
status: wip
---

# Plan: ALL-CAPS child bullets become project note sections

## Context

`Ctrl+Shift+Alt+N` is bound in the vault to
`bob-navigation-hotkeys:create-project-note-from-task`. Everything below lives in the
**`bob-plugins` linked repo**, except the one documentation edit called out in step 7,
which lives in **this `bob-cli` checkout**. Open the linked repo with the `/sase_repo`
skill before touching it:

```bash
sase repo open bob-plugins -r "Implement ALL-CAPS section bullets for create-project-note-from-task"
```

Use only the path that command prints. These are plain CommonJS Obsidian plugins:
`main.js` is the source, there is no build step.

### How the command works today

`BobNavigationHotkeysPlugin.createProjectNoteFromTask(editor, view)`
(`plugins/bob-navigation-hotkeys/main.js`) does the following:

1. `getProjectSourceTaskBlock()` captures the cursor's task line plus its contiguous
   indented child block, returning `{ startLine, endLineExclusive, lines, childLines }`.
2. `buildProjectTasksFromChildBullets(childLines, createdDate)` converts that child
   block into rendered `## Tasks` lines. Direct children (the shallowest child indent)
   become top-level `- [s] #task ... [created::DATE]` lines via
   `parseProjectChildListItem()` + `buildProjectTaskLineFromChildBullet()`; deeper lines
   are re-indented under their task by `normalizeNestedChildLine()`. It returns
   `{ taskLines, lossless }`. `lossless: false` means some nonblank child line could not
   be represented.
3. If the conversion was lossy, the caller sets `childConversionLossy` and later shows a
   notice **and keeps the source task**, so nothing is lost.
4. `createProjectNoteFile()` instantiates `_templates/new_project.md` through Templater,
   then `applyProjectCreationFrontmatter()` writes `parent`/`type`/`status`/`scheduled`.
5. `buildProjectContentFromTask(content, parsedTask, { childTaskLines })` replaces the
   `(REPLACE WITH PROJECT COMPLETION CRITERIA)` placeholder on the `^prj` line and, when
   there are child tasks, calls `replaceProjectTasksPlaceholder()` to swap them in for
   the `(REPLACE WITH TASK DESCRIPTION)` placeholder under `## Tasks`. It returns
   `{ content, seeded, tasksInserted, tasksSectionMissing }`.
6. On full success the source block is removed with `removeTaskBlockFromContent()` and
   `getProjectFromTaskNoticeText()` reports the result.

The project template (`_templates/new_project.md` in the vault) currently ends with
three level-2 sections:

```markdown
- [ ] #task #prj (REPLACE WITH PROJECT COMPLETION CRITERIA) #hide ^prj

## Tasks

- [ ] #task (REPLACE WITH TASK DESCRIPTION) [created::<date>]

## Future Work

## Requirements
```

### What is missing

Every direct child bullet is forced through the task converter. There is no way to say
"these bullets are reference material for the `## Requirements` section", so notes
either become bogus `#task` lines or have to be moved by hand after creation.

## Desired behavior

Given a source task like:

```markdown
- [ ] #task Ship the widget ^widget
  - Draft the API
  - REQUIREMENTS
    - Must work offline
    - p95 under 200ms
      - measured at the edge
  - OPEN QUESTIONS
    - Who owns rollout?
  - Write the migration
```

`Ctrl+Shift+Alt+N` produces a project note whose `## Tasks` section holds only
`Draft the API` and `Write the migration` as converted tasks, whose existing
`## Requirements` section gains the two requirement bullets (with the nested
`measured at the edge` preserved one level under `p95 under 200ms`), and which grows a
new `## Open Questions` section at the end of the note holding `Who owns rollout?`.

### Section-bullet recognition rules

A direct child bullet is a **section bullet** when **all** of these hold:

- It sits at the direct-child indent level (the same level
  `buildProjectTasksFromChildBullets` already treats as task candidates).
- It has **no checkbox**. `parseProjectChildListItem()` returns `status: null` for a
  bare `- FOO` and a status character for `- [ ] FOO`. A checkbox child stays a task
  even when its body is ALL CAPS, because the user explicitly wrote a task.
- Its trimmed body matches the ALL-CAPS title shape (below).
- At least one **nonblank list item** is nested deeper than it. A bare ALL-CAPS bullet
  with no children is not a section and falls through to the existing task conversion,
  exactly as it does today.

The ALL-CAPS title shape is a deliberate whitelist rather than "has no lowercase", so
Markdown constructs are never mistaken for section titles:

```js
const PROJECT_SECTION_TITLE_RE = /^[A-Z0-9][A-Z0-9 \t&'(),./-]*$/;
```

plus a separate requirement that the body contain at least one `A-Z`. This accepts
`REQUIREMENTS`, `FUTURE WORK`, `NON-GOALS`, `Q&A`, and `OPEN QUESTIONS (V2)`. It rejects
`[[SOME_NOTE]]`, `#TODO`, `REQUIREMENTS ^abc`, `` `CODE` ``, `SNAKE_CASE`, and anything
containing a lowercase letter.

### Title-casing rules

Lowercase the whole body, then uppercase the first character of every maximal run of
letters/digits, and collapse internal whitespace runs to a single space.

| Bullet body      | Section title    |
| ---------------- | ---------------- |
| `REQUIREMENTS`   | `Requirements`   |
| `FUTURE WORK`    | `Future Work`    |
| `NON-GOALS`      | `Non-Goals`      |
| `OPEN QUESTIONS` | `Open Questions` |

Acronyms are not special-cased: `API DESIGN` title-cases to `Api Design`. That is the
literal reading of "converted to title-case" and keeps the rule predictable. The escape
hatch is the reuse rule below — matching is case-insensitive and **an existing header
keeps its own casing**, so adding `## API Design` to `_templates/new_project.md` makes
an `API DESIGN` bullet land there with the acronym intact.

### Section reuse rules

- Compare the title against the note's existing **level-2 (`##`) headers**, fence-aware
  and skipping frontmatter, the same way `findProjectTasksHeaderIndex()` already scans.
  Comparison normalizes both sides: trim, collapse internal whitespace, casefold.
- On a match, the existing header line is left byte-identical and the notes are appended
  to the end of that section's body. The section body ends at the next header of level
  `#` or `##` (fence-aware) or at EOF. Bounding on level ≤ 2 rather than "any header"
  means notes land after a `### Subsection` that is part of the section, not before it.
  - Body empty: insert one blank line, then the notes (`## Requirements` →
    `## Requirements`, ``, `- Must work offline`).
  - Body non-empty: insert directly after the section's last nonblank line, so the notes
    continue the existing list. Trailing blank lines that separated the section from the
    next header stay where they are.
- With no match, a new section is appended at the **end of the note**, in the source
  order of the section bullets, as one blank line, the `## Title` header, one blank
  line, then the notes. Reuse the EOF-append blank-line and trailing-newline idiom
  already in `insertTaskMoveBlocks()`.
- Two section bullets whose titles normalize equally merge into one section, in source
  order.
- `## Tasks` is not special-cased. A `TASKS` section bullet resolves to the template's
  `## Tasks` header and its notes are appended **after** the converted child tasks,
  because task insertion runs first (see step 4). This is a deliberate decision: the
  user opted in by writing `TASKS`, and appending verbatim is more predictable than
  silently rejecting it.

### Note rendering rules

Notes are copied **verbatim** apart from re-indentation. No `#task` token, no
`[created::...]` field, no checkbox rewriting. A `- [ ] foo` written under a section
bullet stays `- [ ] foo`; without `#task` the vault's Tasks queries ignore it, which is
the correct outcome for note content.

- The section bullet's own direct children (the shallowest indent nested under it)
  become top-level bullets at column 0 in the section.
- Deeper descendants keep their indentation **relative** to those direct children.
- Blank lines are trimmed from both ends of a section's note block and preserved in the
  middle, matching what `flush()` does for nested task lines today.

Concretely, this is `normalizeNestedChildLine()` semantics with one difference: that
helper forces an empty relative indent to `"\t"` because its lines nest under a task.
Section notes become top-level, so an empty relative indent must stay empty. Add a
sibling helper rather than changing `normalizeNestedChildLine()`, whose current behavior
the task path depends on.

## Implementation steps

### 1. Add the recognition and formatting helpers

In `plugins/bob-navigation-hotkeys/main.js`, near the other `PROJECT_*` constants
(around line 68-90) and the project helpers (around line 3498-3700):

- `PROJECT_SECTION_TITLE_RE` — the whitelist above.
- `parseProjectSectionBulletTitle(body)` → the title-cased string, or `null` when the
  body is not an ALL-CAPS title.
- `formatProjectSectionTitle(body)` → the lowercase-then-capitalize-runs transform, with
  whitespace collapsed.
- `normalizeProjectSectionTitle(title)` → the casefolded/whitespace-collapsed comparison
  key.
- `normalizeProjectSectionNoteLine(lineText, baseIndent)` → the
  `normalizeNestedChildLine()` variant that allows an empty relative indent and returns
  `""` for blank lines.

### 2. Split the child block into tasks and sections

Rename `buildProjectTasksFromChildBullets` to `buildProjectSeedFromChildBullets` (same
file; update the single call site in `createProjectNoteFromTask` and the `helpers`
export list around line 21436). The old name now describes only half of what the
function does. Nothing in `scripts/test-navigation-hotkeys.cjs` references it today, so
the rename costs no test churn.

New return shape:

```js
Object.freeze({
  taskLines, // unchanged
  sections, // frozen [{ title, noteLines }] in source order, titles merged
  lossless, // unchanged semantics
});
```

Inside the existing single pass over `lines`:

- When a direct-child list item is reached, first test it for section-bullet
  eligibility. Eligibility needs lookahead for "has at least one nonblank nested list
  item", so precompute, for each direct-child index, whether a deeper nonblank list item
  follows before the next direct child. A cheap way is a small pre-pass that records
  each direct child's line index and the span of lines belonging to it; the section/task
  decision then reads that span.
- A qualifying section bullet starts a section accumulator instead of a task
  accumulator. Its nested lines are pushed through `normalizeProjectSectionNoteLine()`
  using the shallowest nested indent as the base, then end-trimmed on flush and merged
  into the entry for that normalized title.
- A non-qualifying bullet keeps today's exact task behavior.
- `lossless` keeps its meaning: any nonblank child line that lands in neither a task nor
  a section still sets it to `false`.

### 3. Insert sections into the project note

Add to the same file:

- `findProjectSectionRange(lines, title)` → `{ headerIndex, bodyEndExclusive }` or
  `null`. Frontmatter- and fence-aware; matches `##` headers only; bounds the body at
  the next `#`/`##` header or EOF; `bodyEndExclusive` is one past the section's last
  nonblank line (or `headerIndex + 1` for an empty body).
- `insertProjectSectionNotes(content, sections)` →
  `{ content, insertedCount, createdCount }`. Applies the reuse rules above: existing
  sections first (highest line index first, so earlier insert points stay valid), then
  new sections appended at EOF in source order.

Note that `replaceProjectTasksPlaceholder()` bounds the `## Tasks` section with
`SECTION_HEADER_RE` (any level). Leave that alone — this step adds a new level-aware
helper and does not change the existing task path.

### 4. Thread sections through `buildProjectContentFromTask`

Accept `options.sections`. Run the existing `replaceProjectTasksPlaceholder()` call
first, then `insertProjectSectionNotes()` on the result, so `## Tasks` notes land after
converted tasks. Extend the returned object with `sectionsInserted` and
`sectionsCreated` counts; keep `content`, `seeded`, `tasksInserted`, and
`tasksSectionMissing` unchanged.

### 5. Wire up `createProjectNoteFromTask`

- Capture `conversion.sections` alongside `conversion.taskLines`.
- The `conversion.lossless && conversion.taskLines.length > 0` guard currently discards
  a lossless conversion that produced no task lines. Widen it to
  `conversion.lossless && (conversion.taskLines.length > 0 || conversion.sections.length > 0)`
  so a source task whose children are _all_ section bullets still seeds sections instead
  of falling into the `childConversionLossy` branch.
- Pass `sections` into `buildProjectContentFromTask`.
- Add a guard mirroring the existing `tasksInserted` one: when sections were requested
  but `seedResult.sectionsInserted` is short, show
  `"Created project, but a section could not be added; source task was kept"` and return
  `true` without removing the source task.
- Extend `getProjectFromTaskNoticeText()` with a section count, so the notice reads e.g.
  `Created project X from task "Ship the widget" (task removed from Y; 2 sections seeded)`.
  Use `section`/`sections` singular/plural like the existing link chip.

### 6. Tests

Add `node:test` cases to `scripts/test-navigation-hotkeys.cjs`, following the existing
`helpers.<fn>(...)` + `assert.deepEqual` style. Export every new helper from the
`helpers` object so tests can reach it. The child-bullet converter has **no** tests
today, so add coverage for the existing behavior at the same time — the rename in step 2
makes that a natural moment.

Cover at least:

- ALL-CAPS bullet **with** nested list items becomes a section; the same bullet
  **without** them stays a task.
- `- [ ] REQUIREMENTS` with children stays a task (checkbox wins).
- Rejected title shapes: `[[SOME_NOTE]]`, `#TODO`, `REQUIREMENTS ^abc`, `Mixed Case`,
  `SNAKE_CASE`.
- Title-casing: `FUTURE WORK` → `Future Work`, `NON-GOALS` → `Non-Goals`, collapsed
  internal whitespace.
- Reuse of an existing `## Requirements` (empty body → blank line then notes; non-empty
  body → appended after the last nonblank line, header line byte-identical).
- Case-insensitive match keeping the existing header's casing (`API DESIGN` bullet +
  `## API Design` header).
- New section appended at EOF with correct blank lines and preserved trailing newline;
  multiple new sections keep source order.
- Two bullets with the same normalized title merge in source order.
- Note verbatim-ness: no `#task`, no `[created::]`; nested depth preserved relative to
  the section's direct children.
- Mixed child block: tasks and sections in one source task, tasks still land in
  `## Tasks`.
- `TASKS` section bullet appends after the converted tasks.
- CRLF content round-trips (the existing helpers detect `\r\n` and rejoin with it — keep
  that).
- `lossless: false` still keeps the source task.

Run from the `bob-plugins` checkout:

```bash
npm test
npm run validate
```

### 7. Docs, version, and deploy

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.23.0` to `1.24.0`.
  Versions are per-plugin; touch no other manifest.
- Update the `Bob Navigation Hotkeys` row in the `bob-plugins` `README.md` plugin table
  to mention the new section-bullet behavior, matching the row's existing run-on style.
- In **this `bob-cli` checkout**, extend `docs/projects.md`. The paragraph at roughly
  line 190 already documents the "Create project note from task" command's
  `[scheduled::]` transfer; add a short paragraph after it covering ALL-CAPS section
  bullets, title-casing, existing-section reuse, and the verbatim note rendering. Keep
  the file's line-wrapping style.
- Deploy to the vault, as `bob-plugins/AGENTS.md` requires. Run from the `bob-plugins`
  checkout the `/sase_repo` skill printed, previewing first. `-r "$PWD"` is required
  because the default source path does not exist in a SASE workspace, and `--no-pull`
  avoids a pull against the dirty checkout:

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
2. `bob plugins sync --dry-run` shows only `bob-navigation-hotkeys` `main.js` +
   `manifest.json` changing.
3. Manual check in Obsidian after a plugin reload, from an area or project note, on a
   source task shaped like the "Desired behavior" example: tasks land in `## Tasks`,
   `REQUIREMENTS` notes join the template's existing `## Requirements`, a new
   `## Open Questions` section appears at the end, and the source task block is removed
   from the area note.

## Out of scope

- Any change to how `bob projects sync` (Rust, `src/native/projects.rs`) reads project
  notes. Section notes are plain Markdown outside `^prj` and outside `## Tasks`; that
  command already ignores them.
- Editing `_templates/new_project.md` to add more default sections. The feature works
  with whatever sections the template happens to have.
- The plain `Ctrl+Shift+N` "Create project note" command, which has no source task and
  therefore no child bullets.
- Acronym-preserving title-casing beyond the existing-header reuse rule.
