---
tier: epic
title: Prompt for a reason when Ctrl+Shift+P sets a scheduled date
goal: "After choosing a `scheduled` date in the `Ctrl+Shift+P` bullet-property picker, a
  reason stage prompts for one line of free text and records it as `- **YYYY-MM-DD:**
  <reason>` under a managed `- **SCHEDULED:**` child bullet on the task, written
  atomically with the date itself, across the single-task, project-note, counted, and
  priority-roll write paths.

  "
phases:
  - id: ledger
    title: Managed SCHEDULED ledger Markdown model
    depends_on: []
    size: medium
    description: "ledger: add the pure parse/format/plan functions for the managed `-
      **SCHEDULED:**` child bullet and its `- **YYYY-MM-DD:** <reason>` entries,
      exported and unit tested, with no UI or editor writes yet.

      "
  - id: prompt
    title: Reason stage and single-task atomic write
    depends_on:
      - ledger
    size: medium
    description: "prompt: add the free-text reason stage to the bullet-property picker
      after a `scheduled` date is chosen, and commit the date plus the ledger entry to
      an ordinary task in one whole-note editor transaction.

      "
  - id: project
    title: Reason ledger on `^prj` lifecycle tasks
    depends_on:
      - prompt
    size: small
    description: "project: record the reason on the `^prj` task line inside the same
      guarded transaction that writes the project note's `scheduled` frontmatter.

      "
  - id: counted
    title: Counted-session reason ledger
    depends_on:
      - project
    size: medium
    description: "counted: apply one reason to every task in an `N<Ctrl+Shift+P>`
      session, inserting each task's ledger entry into the single whole-note transaction
      the counted planner already builds.

      "
  - id: priority
    title: Reason prompt for priority-level rolls
    depends_on:
      - counted
    size: medium
    description: "priority: prompt for a reason when a P-level is chosen from the
      `priority` property, recording it against the rolled date on single, project, and
      counted paths, and surface it in the priority toast.

      "
  - id: release
    title: Docs, styles polish, version bump, and vault deploy
    depends_on:
      - priority
    size: small
    description:
      "release: document the ledger and the reason stage, finish the picker/toast CSS,
      bump the plugin manifest and README, and deploy the plugin to the vault."
proposed_by: bbugyi200.athena.ts
create_time: 2026-09-09 20:00:29
status: wip
---

- **PROMPT:**
  [prompts/202608/scheduled_reason_ledger.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/scheduled_reason_ledger.md)

# Plan: Prompt for a reason when Ctrl+Shift+P sets a scheduled date

## Repositories

This epic spans two repositories. Phase agents MUST open the non-primary repository
through the `/sase_repo` skill and use the printed path for every read and write:

- `sase repo open bob-plugins -r "<reason>"` — owns
  `plugins/bob-navigation-hotkeys/main.js`, `plugins/bob-navigation-hotkeys/styles.css`,
  `plugins/bob-navigation-hotkeys/manifest.json`, `scripts/test-navigation-hotkeys.cjs`,
  and `README.md`. Every code phase does its work here.
- The primary `bob-cli` repo owns `docs/projects.md`, which is where the `Ctrl+Shift+P`
  picker is documented.

Run the plugin test suite from the `bob-plugins` checkout with `npm test` (it shells out
to `node --test scripts/*.cjs`); `npm run validate` checks manifests. No phase needs the
`chezmoi` repo: this feature adds no new `~/.config/bob/config.yml` keys.

## Background

`Ctrl+Shift+P` is bound in the vault's `.obsidian/hotkeys.json` to the
`set-bullet-property` command in `plugins/bob-navigation-hotkeys/main.js`, which calls
`openBulletPropertyPicker()` and opens `BulletPropertyPickerModal` (a
`FilteredPickerModal` subclass). The modal is a staged picker driven by `this.stage`
plus
`applyOptions({items, title, headerIcon, inputLabel, placeholder, resultsLabel, emptyText, footerHints, getSubtitle, filterItem, renderItem, openItem})`:

- `stage === "properties"` — one row per entry in `~/.config/bob/config.yml`'s
  `properties` list.
- `stage === "value"` — for `values: date` this is `createBulletPropertyDateItems()`
  (ten relative presets), plus a live "typed date" row synthesized in
  `getFilteredItems()` by `createBulletPropertyTypedDateItem()` for input like
  `2026-09-01`, `9/1`, or `+3d`, plus a re-rollable `P<N> roll` row from
  `createPriorityRollDateItem()` when the task already has a priority.
- `stage === "blockid"` — a **free-text** stage. It sets `items: []` and overrides
  `getFilteredItems()` to synthesize a single preview row from `this.getRawQuery()` via
  `validateBlockIdCandidate()`. This is the exact pattern the new reason stage copies.

Choosing a value calls `applySelectedValue()`, which routes to one of four writers on
the plugin object:

| Path                      | Writer                                                                 | Mechanism                                       |
| ------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------- |
| Ordinary task, single     | `setBulletPropertyValue()` → `setInlineBulletPropertyValues()`         | `replaceEditorLine()` on one line               |
| `^prj` lifecycle task     | `setProjectNoteScheduledValue()`                                       | `applyEditorContentTransaction()` on whole note |
| `N<Ctrl+Shift+P>` counted | `setCountedBulletPropertyValue()` → `planCountedBulletPropertyBatch()` | `applyEditorContentTransaction()` on whole note |
| `priority` P-level        | `setBulletPriorityValue()` / `setCountedBulletPriorityValue()`         | delegates to the three above                    |

Existing machinery this plan reuses rather than reinvents:

- `findCurrentBulletChildBlock(lines, parentLine)` → `{startLine, endLineExclusive}` for
  a bullet's whole descendant block.
- `getDependencyChildIndent(lines, parentLine)` → reuse an existing direct-child indent,
  else `parentIndent + "\t"`.
- `getBulletIndent()` / `getBulletIndentWidth()` (tab = next multiple of 4; each `>`
  quote level = 4).
- `findNearestParentListItem(lines, childLine)` → the direct-parent list item, used to
  prove a line is a _direct_ child.
- `collectDependencyNavigationBullets(content, parentLine)` → the managed `![[#^id]]`
  dependency bullets under a task.
- `BULLET_PROPERTY_LIST_ITEM_RE` → `^\s*(?:>\s*)*(?:[-*+]|\d+[.)])\s`.
- `applyEditorContentTransaction(cm, oldContent, newContent, cursor)` → atomic
  whole-note replace with cursor restore.
- `splitMarkdownContent(content)` → `{lines, ...}` used by every whole-note planner.
- `formatBulletPropertyDate(date)`, `getBulletPropertyDateWeekday(date)`,
  `getLocalDayOffset()`, `formatRelativeDayOffset()` → the exact date/weekday/distance
  vocabulary the priority toast already speaks.
- `buildPriorityNoticeModel()` / `renderPriorityNoticeFragment()` /
  `showPriorityNotice()` → the rich HTML toast.

The vault's established convention for a generated child bullet is a bold uppercase
label: `- **CLIP:**`, `- **PHOTO:**`, `- **DEPENDS ON:**`. When `bob capture` has more
than one item it writes the label alone on the header line and nests the items one level
deeper (`src/native/capture_clip.rs::rendered_lines`). The new ledger follows that same
shape. A vault-wide grep confirms **no `- **SCHEDULED:**` bullets exist today**, so
there is nothing to migrate.

## Design

### The Markdown shape

```markdown
- [ ] #task Renew the domain [priority:: medium] [scheduled:: 2026-09-01] ^renew-domain
  - **SCHEDULED:**
    - **2026-09-01:** registrar support says to retry after the transfer lock clears
    - **2026-08-20:** first guess, before I read the transfer policy
  - **CLIP:** https://registrar.example/tickets/123
```

- **Header bullet** — `- **SCHEDULED:**`, a direct child of the task, exactly as
  requested. No emoji: `- **CLIP:**` and the canonical dependency bullets carry none
  either, and an emoji would compete with the date keys below it.
- **Entry bullet** — `- **YYYY-MM-DD:** <reason>`, a direct child of the header. The
  bold-label-plus-colon form is the vault's own `**LABEL:** value` idiom, so the date
  reads unambiguously as the key that the reason belongs to, in both source and preview.

### Decisions, and why

1. **Nothing is written until the reason is settled.** The date write and the ledger
   write happen in a single `applyEditorContentTransaction()`. Prompting _after_ writing
   the date would leave a scheduled date with no reason whenever the user hits `Esc`;
   prompting first and committing once makes cancel a true no-op. This is the
   reliability keystone of the feature and every phase must preserve it.
2. **Skipping writes no entry at all.** Empty input plus `Enter` sets the date exactly
   as the picker does today and touches no child bullets — it does not create the
   header, and does not leave a dangling `- **2026-09-01:**` with nothing after the
   colon. Every row in a ledger therefore explains something, and the feature costs
   nothing on the quick "nudge it to tomorrow" edits that make up most `scheduled`
   changes.
3. **Newest first.** A new entry is inserted directly beneath the header. The first row
   under `- **SCHEDULED:**` then always answers "why is this task sitting on the date it
   is sitting on?", which is the question the ledger exists to answer, and a long
   history never pushes the live answer out of view.
4. **One entry per date.** Choosing a date that already has an entry rewrites that
   entry's reason and moves it to the top rather than appending a duplicate key. The
   ledger stays a date-to-reason map, which is what "clearly show the schedule date it
   is associated with" implies.
5. **The plugin never deletes text it did not write.** Child bullets of the header that
   do not match the entry pattern are left exactly where they are, below the managed
   entries. Same for a `- **SCHEDULED:**` bullet the user typed by hand: it is adopted,
   not replaced.
6. **Ambiguity refuses rather than guesses.** Two `- **SCHEDULED:**` headers under one
   task is unresolvable, so the reason stage is never offered (the date write proceeds
   exactly as today, with a notice explaining why), and if the ambiguity appears between
   opening the stage and committing, the whole edit aborts with nothing written —
   matching the `"Current task changed; bullet property was not updated"` guards already
   in `getInlinePropertyWriteContext()`.
7. **Deleting `scheduled` leaves the ledger alone.** `Ctrl+D` on the `scheduled` row
   does not prompt and does not prune entries. The ledger is a record of decisions, not
   a mirror of the current field.
8. **Entries are self-describing, so drift is harmless.** Because a skipped reason
   writes nothing, the top entry's date can fall out of sync with the live
   `[scheduled:: ...]` field. That is by design and costs nothing, because each row
   names its own date.

### Considered and rejected

- **A decision timestamp per row** (`- **2026-09-01** (set 2026-08-06): ...`) —
  newest-first ordering already encodes the sequence, and the extra date halves the room
  the reason gets.
- **Wiki-linking the date** (`- **[[2026-09-01]]:**`) — would mint unresolved links for
  every non-daily-note date.
- **Appending oldest-first** — matches `bob capture`'s log shape, but buries the live
  reason at the bottom.
- **A separate `[reason:: ...]` inline field** — inline fields are Tasks-parser
  territory (see the `priority` comment in `config.yml`); a long prose value on the task
  line would be read right-to-left by Tasks parsers and would push the task text off
  screen.

### The reason stage

A new `stage === "reason"`, built exactly like the existing `blockid` stage:
`items: []`, plus a branch at the top of `getFilteredItems()` that synthesizes one
preview row from `this.getRawQuery()`.

| Element         | Content                                                                                                                                                                                                                                     |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `headerIcon`    | `message-square-quote` (`applyIcon()` already swallows unknown-icon failures, so a Lucide rename degrades to a blank slot, not a crash)                                                                                                     |
| `title`         | `Why this date?`                                                                                                                                                                                                                            |
| `getSubtitle()` | `<counted scope · >2026-09-01 · Tuesday · in 26 days` — built from `formatBulletPropertyDate()`, `getBulletPropertyDateWeekday()`, and `formatRelativeDayOffset(getLocalDayOffset(today, date))`, the same vocabulary as the priority toast |
| `inputLabel`    | `Reason for this scheduled date`                                                                                                                                                                                                            |
| `placeholder`   | `Why is this scheduled for 2026-09-01?`                                                                                                                                                                                                     |
| `resultsLabel`  | `Reason preview`                                                                                                                                                                                                                            |
| `footerHints`   | `↵ save` · `esc cancel`                                                                                                                                                                                                                     |

The single preview row is the whole point of the stage: it renders the **literal
Markdown line that will be written**, so the user can never be surprised by what lands
in the note.

| Input state                         | Row icon                       | Row title                              | Row detail                                                       |
| ----------------------------------- | ------------------------------ | -------------------------------------- | ---------------------------------------------------------------- |
| Non-empty, header absent            | `message-square-quote`         | `- **2026-09-01:** <reason>`           | `Creates a SCHEDULED bullet on this task`                        |
| Non-empty, header present, date new | `message-square-quote`         | `- **2026-09-01:** <reason>`           | `Added to the existing SCHEDULED bullet`                         |
| Non-empty, date already logged      | `refresh-cw`                   | `- **2026-09-01:** <reason>`           | `Replaces the current 2026-09-01 reason: <old reason truncated>` |
| Empty                               | `circle-slash` (muted row)     | `Skip — set 2026-09-01 with no reason` | `No SCHEDULED entry is written`                                  |
| Over the length cap                 | `alert-triangle` (invalid row) | `<reason>`                             | `Reason is too long (max 500 characters)` — `Enter` is refused   |

Row classes mirror the block-ID stage so the CSS stays one family:
`bob-cnp-reason-preview-row` with `is-valid` / `is-skip` / `is-replace` / `is-invalid`,
and a `bob-cnp-reason-preview` code-ish span for the rendered Markdown, alongside the
existing `.bob-cnp-blockid-preview*` rules in `styles.css`.

`Esc` cancels the entire picker with nothing written, matching the block-ID stage.
Because no write has happened yet, that is always a clean cancel.

### Reason normalization

`normalizeScheduledReasonText(raw)`:

1. Replace every `\r?\n` and `\t` with a single space (pasted multi-line text must never
   break the bullet).
2. Collapse runs of whitespace to one space; trim.
3. Empty result ⇒ the skip path.
4. Longer than 500 characters ⇒ invalid; `Enter` is refused with the inline invalid row
   (no notice spam, the row is already on screen).

No other escaping: `[[`, `**`, backticks and the rest are legitimate note content and
land verbatim.

### Recognition rules

- **Header** — a direct child list item of the task
  (`findNearestParentListItem() === taskLine`) whose text after the list marker matches
  `^(?:<emoji or symbol run>[ \t]+)?\*\*SCHEDULED:\*\*[ \t]*$`, case-insensitive on the
  label. A hand-added leading emoji is tolerated and normalized away on the next managed
  write, mirroring how `DEPENDENCY_NAVIGATION_BULLET_RE` accepts and normalizes its
  legacy labels.
- **Entry** — a direct child list item of the header matching
  `^\*\*(?<date>\d{4}-\d{2}-\d{2}):\*\*[ \t]*(?<reason>.*?)[ \t]*$` whose `date` is a
  real calendar date (reuse the existing date validator used by
  `validateProjectScheduledDate()`). Anything else under the header is unmanaged.
- **Insertion point for a new header** — immediately after the last managed
  dependency-transclusion bullet in the task's direct-child block
  (`Math.max(...collectDependencyNavigationBullets(content, taskLine).lineIndices) + 1`),
  or `taskLine + 1` when there are none. Generated task metadata stays grouped directly
  under the task, above free-form children like `- **CLIP:**`.
- **Header indent / marker** — `getDependencyChildIndent(lines, taskLine)` and the
  marker of the first existing direct child, defaulting to `-`.
- **Entry indent / marker** — reuse the first existing managed entry's exact indent and
  marker when one exists; otherwise `headerIndent + (headerIndent − taskIndent)`
  reapplied as literal text, falling back to `headerIndent + "\t"`. This is the same
  "match the target note's own indentation" rule the capture writer follows and the
  literal-tab default `task-status-cycler` uses for generated child bullets.

## Phases

### 1. `ledger` — Managed SCHEDULED ledger Markdown model

Pure functions in `plugins/bob-navigation-hotkeys/main.js`, added near the
dependency-navigation helpers (roughly lines 1180–1900) so the two managed-child-bullet
subsystems sit together. No editor writes, no UI.

Add and export from `module.exports`:

- `SCHEDULED_REASON_LABEL = "SCHEDULED"` and the header/entry regexes.
- `normalizeScheduledReasonText(raw)` — the normalization above; returns
  `{text, valid, reason}` where `reason` is `"empty"` or `"too-long"` when invalid.
- `formatScheduledReasonHeaderBullet(indent, marker)` →
  `` `${indent}${marker} **SCHEDULED:**` ``.
- `formatScheduledReasonEntryBullet(date, reason, indent, marker)` →
  `` `${indent}${marker} **${date}:** ${reason}` `` (no trailing space when `reason` is
  empty — though the callers never write an empty entry).
- `parseScheduledReasonEntry(line)` → `{date, reason, indent, marker}` or `null`.
- `collectScheduledReasonLedger(content, taskLine)` → frozen
  `{headerLine, headerIndent, headerMarker, entries: [{line, date, reason, indent, marker}], unmanagedLines,  startLine, endLineExclusive, reason}`
  where `reason` is `null` on success or one of `"parent-out-of-range"`, `"not-task"`,
  `"ambiguous-header"` (more than one header under the task). Mirror
  `collectDependencyNavigationBullets()`'s shape and its
  `createDependencyNavigationCollection()` freezing style.
- `planScheduledReasonUpsert(content, taskLine, date, reason)` → a frozen plan
  `{operation, changed, reason, content, insertedLineCount, replacedDate, cursorLineDelta}`
  where `operation` is one of `"noop"` (empty reason), `"insert-header"` (header plus
  first entry), `"insert-entry"`, `"replace-entry"` (same date already logged: drop the
  old line, insert the new one at the top), or `"guard"` (with a `reason` code). The
  plan returns **whole new note content**, so callers hand it straight to
  `applyEditorContentTransaction()`.
- `planScheduledReasonUpsertsInContent(content, edits)` — the same, applied to many
  `{taskLine, date, reason}` edits in one pass, processed **bottom-up by task line** so
  earlier insertions never invalidate later line numbers. Returns
  `{valid, error, content, changed, insertedLineCount, taskLineDeltas: Map<originalLine, delta>}`.
  `counted` and `priority` depend on this; building it here keeps every line-shifting
  rule in one tested place.

Tests in `scripts/test-navigation-hotkeys.cjs` (`node --test`, the helpers are pulled
from the module's exports):

- Creating the header on a task with no children; with tab-indented children; with
  two-space-indented children; with a quoted (`> - [ ] #task`) task.
- Header placement above a `- **CLIP:**` child and below existing `![[#^dep]]`
  dependency bullets.
- Adopting a hand-written `* **scheduled:**` header (alternate marker, lowercase) and a
  `- 📅 **SCHEDULED:**` header.
- Newest-first insertion ordering across three successive dates.
- Same-date replacement moving the entry to the top and preserving the other entries'
  relative order.
- Unmanaged children of the header surviving untouched.
- `"ambiguous-header"` guard when two headers exist.
- Reason normalization: multi-line paste, tab, whitespace collapse, trim, empty,
  500-character boundary and 501 rejection, and a reason containing `[[`, `**`, and
  backticks landing verbatim.
- `planScheduledReasonUpsertsInContent()` over three tasks in one note, asserting both
  the resulting content and each `taskLineDeltas` entry.

### 2. `prompt` — Reason stage and single-task atomic write

Wire the stage into `BulletPropertyPickerModal` and commit ordinary single-task edits.

Modal work:

- `showReasonStage({date, commit})` — sets `this.stage = "reason"`, stashes
  `this.pendingScheduledDate = date` and `this.pendingScheduledCommit = commit` (a
  callback the caller supplies so `project`, `counted`, and `priority` can reuse the
  stage untouched), then `applyOptions({...})` per the table in **The reason stage**,
  and `renderAll({clearQuery: true})`.
- A `this.stage === "reason"` branch at the top of `getFilteredItems()` synthesizing the
  single preview item from `normalizeScheduledReasonText(this.getRawQuery())` plus
  `collectScheduledReasonLedger()` on live editor content, so the "replaces the current
  reason" state is accurate as the user types.
- `renderReasonPreviewItem(item, rowEl, query)` following `renderBlockIdPreviewItem()`.
- `openItem` refuses the invalid (too-long) row and otherwise calls
  `this.pendingScheduledCommit(item.reasonText)`.
- Clear the pending reason state in `clearPendingBatch()` and `showPropertyStage()` so a
  re-entered picker never inherits a stale date.
- Route into the stage from `applySelectedValue()` **only** when the selected property
  is `values: date` **and** its normalized name is `scheduled` **and**
  `collectScheduledReasonLedger()` does not report `"ambiguous-header"`. On ambiguity,
  fall through to today's direct write and add
  `; task has multiple SCHEDULED bullets, reason not recorded` to the notice.
  Non-`scheduled` date properties are untouched, so a future `due`-style property does
  not silently acquire a prompt.

Writer work — new
`setInlineScheduledValueWithReason(cm, cursor, filePath, lineText, date, reason, options)`:

1. `getInlinePropertyWriteContext()` for the usual stale guards.
2. Compute the new task line exactly as `setInlineBulletPropertyValues()` does today —
   including `isFutureInlineScheduledValue()` ⇒ `blockObsidianTaskCheckboxStatus()`, and
   the `isDueInlineScheduledValue()` ⇒ `buildTargetScheduledRecoveryByLine()` /
   `reconcileBlockedScheduledTaskLine()` recovery path with its post-await re-guard. Do
   not fork this logic: factor the line computation out of
   `setInlineBulletPropertyValues()` and call it from both, so Blocked/recovery behavior
   can never drift between the reason and no-reason paths.
3. Splice the new task line into the content, run `planScheduledReasonUpsert()` on the
   result, and apply the whole thing with one `applyEditorContentTransaction()`,
   restoring the cursor to the task line at `min(cursor.ch, nextLine.length)`.
4. Any guard failure ⇒ `Notice` and return `false` with **nothing written**.

When the reason is empty (skip), keep calling the existing `setBulletPropertyValue()`
path verbatim so current behavior and its tests are untouched.

Notice: append `; logged reason` or `; updated 2026-09-01 reason` to the existing
`` `${name} → ${value}` `` text, after the Blocked clause and before
`scheduledRecoveryNoticeSuffix()`.

Styles: add the `.bob-cnp-reason-preview-row` / `.bob-cnp-reason-preview` rules next to
the existing `.bob-cnp-blockid-preview*` block in `styles.css`, reusing the same
Obsidian CSS variables and the existing `@media` breakpoints.

Tests: drive the real modal through the existing `createBulletPropertyPickerHarness()` /
`TransactionEditor` helpers — open the picker, `openItemAtIndex()` the `scheduled`
property, `openItemAtIndex()` a date row, assert `picker.stage === "reason"` and the
subtitle text, set `picker.inputEl.value`, `renderResults()`, assert the preview row's
title and detail, then `openSelectedItem()` and assert the final note content. Cover:
creating the header; appending to an existing one; same-date replacement; skip-on-empty
writing no child bullets; refusal of the too-long reason; `Esc`-equivalent
(`picker.close()`) leaving the note byte-identical; the future-date Blocked marker and
the due-date recovery still applying with a reason attached; and the ambiguous-header
fallthrough writing the date with the explanatory notice.

### 3. `project` — Reason ledger on `^prj` lifecycle tasks

On a `^prj` lifecycle task the picker's `scheduled` row targets project frontmatter
(`PROJECT_NOTE_PROPERTY_TARGETS`), but the reason still belongs on the `^prj` task line,
which is where the user is standing and which already carries `[priority:: ...]` and
dependency bullets.

- Route `applySelectedValue()`'s `target.kind === "project-frontmatter"` branch through
  the reason stage as well.
- Extend `setProjectNoteScheduledValue()` with an optional `scheduledReason`. It already
  builds whole-note content via `planProjectScheduledUpdate()` and applies it with
  `applyEditorContentTransaction()`, so run `planScheduledReasonUpsert()` on
  `plan.content` at `plan.cursorLine` before the single apply. The frontmatter edit, the
  propagated task schedules, the derived Blocked/recovery statuses, and the ledger entry
  all land together or not at all.
- Notice: same `; logged reason` clause.

Tests: a project note whose `^prj` task gains a ledger entry alongside the frontmatter
write; a second date on the same `^prj` task appending newest-first; skip-on-empty
leaving today's project behavior byte-identical.

### 4. `counted` — Counted-session reason ledger

`N<Ctrl+Shift+P>` applies one date to N tasks. One prompt, one reason, one entry per
task.

- Reason-stage subtitle gains the counted scope prefix already produced by
  `getTaskSessionSubtitle()`, e.g. `3 tasks · 2026-09-01 · Tuesday · in 26 days`, and
  the preview row's detail becomes `Added to 3 tasks` /
  `Creates a SCHEDULED bullet on 2 of 3 tasks`.
- The ambiguity guard runs over **every** target: if any task has two headers, the
  reason stage is not offered and the notice explains it, exactly as in the single-task
  case.
- `setCountedBulletPropertyValue()` gains an optional `scheduledReason`.
  `planCountedBulletPropertyBatch()` already produces whole-note content plus a
  `taskLineDelta`; feed its `plan.content` and the per-target lines (mapped through that
  delta) into `planScheduledReasonUpsertsInContent()` and apply the combined content in
  the single existing `applyEditorContentTransaction()` call.
  `planScheduledReasonUpsertsInContent()`'s bottom-up ordering is what makes the
  multi-task insertion safe; do not hand-roll a second ordering here.
- The counted `cursorLine` must be corrected by the ledger's `taskLineDeltas` so the
  cursor still lands on the original task after insertions above it.
- Notice: `; logged reason on 3 tasks`.

Tests: three consecutive tasks, one reason, asserting all three ledgers and the final
cursor line; a mixed session where one task already has the header and another does not;
a session including a `^prj` task; skip-on-empty leaving current counted behavior
byte-identical; a stale-session guard between opening the reason stage and committing
aborting with nothing written.

### 5. `priority` — Reason prompt for priority-level rolls

Choosing `P1`–`P4` from the `priority` property rolls a `scheduled` date, so without
this phase the ledger would silently develop holes and could no longer be trusted to
answer why a task sits where it does.

- After a level is chosen in the `priority` value stage, roll the date first (single
  task: one roll; counted: one roll per task via the existing `scheduledValueByLine`
  map), then open the reason stage.
- Single-task and `^prj` subtitle: `P2 · 2026-09-01 · Tuesday · in 26 days`. Counted
  subtitle, where the dates differ per task: `P2 · 3 tasks · 2026-09-01 → 2026-09-24`,
  reusing `getPriorityNoticeScheduleSummary()`'s span vocabulary so the stage and the
  toast describe the roll identically. The counted preview row shows the first task's
  line followed by `· and 2 more tasks, each with its own rolled date`.
- `setBulletPriorityValue()`, `setCountedBulletPriorityValue()`, and the project branch
  each take the reason through to their existing single transaction, per-task date for
  per-task entry.
- The `P<N> roll` suggestion row **inside the `scheduled` date stage** already flows
  through the `prompt` phase's path and needs no extra work — verify it with a test
  rather than new code.
- Toast: add a reason chip to `buildPriorityNoticeModel()` /
  `renderPriorityNoticeFragment()` — the reason text truncated to roughly 60 characters,
  with the full text as the element's `title`. Keep the plain-text fallback in
  `model.text` in sync for the `showPriorityNotice()` catch path.
- `Ctrl+D` on `priority` and on `scheduled` still never prompt and never touch the
  ledger.

Tests: single-task P2 with a seeded `random` asserting the exact rolled date in both the
ledger entry and the toast model; counted P3 across three tasks asserting three distinct
rolled dates each with the same reason text; the `P2 roll` row in the date stage;
`Ctrl+D` leaving the ledger intact.

### 6. `release` — Docs, styles polish, version bump, and vault deploy

- `bob-cli` `docs/projects.md`: a new subsection after **Scheduling from the `^prj`
  task** documenting the ledger Markdown shape, newest-first ordering,
  one-entry-per-date replacement, the skip path writing nothing, the ambiguous-header
  fallthrough, and that `Ctrl+D` leaves the ledger alone. Extend the **Priority property
  and scheduled rolls** subsection with the roll-time prompt. State plainly that
  `bob capture <text> p:<N>` writes no ledger entry.
- `bob-plugins` `README.md`: extend the Bob Navigation Hotkeys row.
- `plugins/bob-navigation-hotkeys/manifest.json`: `1.17.0` → `1.18.0` (new user-facing
  capability, no breaking change), and refresh its `description`. Run
  `npm run validate`.
- Final `styles.css` pass: check the reason stage and the toast reason chip in both
  light and dark themes and at the existing narrow `@media` breakpoint, and confirm a
  500-character reason wraps instead of overflowing the row.
- `npm test` green.
- Deploy with
  `bob plugins sync -p bob-navigation-hotkeys -r "<bob-plugins checkout path>"`. Run
  `-d` first and review the diff. Passing `-r` explicitly is required: the default repo
  path is not the SASE workspace checkout.

## Non-goals

- `bob capture <text> p:<N>` keeps rolling dates from the CLI without writing ledger
  entries.
- No query, dashboard, or report surface reads the ledger.
- No migration or backfill: there are no existing `- **SCHEDULED:**` bullets in the
  vault.
- `scheduled` values written by `bob projects sync`, `bob task-status-hooks`, or
  project-schedule propagation are not ledger events — only an explicit `Ctrl+Shift+P`
  choice is.

## Risks

- **`main.js` is ~19.8k lines and every phase edits it.** The phase chain is
  deliberately linear so two agents never hold overlapping edits to the picker's commit
  routing.
- **Line-shifting bugs are the main correctness hazard.** They are confined to
  `planScheduledReasonUpsertsInContent()` in phase `ledger` and covered there by tests
  before any writer uses it.
- **Prompt fatigue.** Mitigated by the skip path being a single `Enter` on an empty
  input, with the preview row saying in words what that `Enter` will do.
