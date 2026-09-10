---
tier: epic
status: done
title: Copy Pomodoro task-link notes into task Work Logs, and fix Work Log indentation
goal:
  Closing a Pomodoro with Ctrl+Enter copies each task-link sub-bullet's sub-sub-bullets
  into that task's `🛠️ **WORK LOG**` (creating the log when absent), and newly generated
  Work Log markers/entries are indented with tabs the way Obsidian indents list items.
proposed_by: bbugyi200.athena.0ey.f2
phases:
  - id: worklog-indent
    title: Tab-indent generated Work Log markers and entries
    depends_on: []
    size: small
    description:
      "worklog-indent: replace block-id-prompt's fixed two-space
      CANONICAL_POMODORO_CHILD_INDENT with a derived indent unit so new Work Log markers
      and entries are tab-indented like Obsidian's Tab key, while legacy space-indented
      notes keep inheriting their own prefix; update the affected test expectations, add
      mixed-indent coverage, bump to 1.11.0, and deploy."
  - id: pomodoro-worklog-notes
    title: Copy Pomodoro Task Link sub-sub-bullets into the linked task's Work Log
    depends_on:
      - worklog-indent
    size: medium
    description:
      "pomodoro-worklog-notes: add Work Log grammar and a note-group scan to
      task-status-cycler so Ctrl+Enter completion copies each Task Link sub-bullet's
      sub-sub-bullets into the linked task's Work Log, creating the log when absent; run
      the writes after the completion plan is applied, account for active-note
      insertions when restoring the cursor, cover it with pure and end-to-end tests,
      bump to 1.15.0, and deploy."
create_time: 2026-09-09 20:00:20
---

- **PROMPT:**
  [prompts/202608/pomodoro_work_log_notes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/pomodoro_work_log_notes.md)

# Pomodoro sub-sub-bullets become Work Log entries

## Goal

Two related changes to how work recorded under a Pomodoro reaches the task it belongs
to.

1. **Feature.** When `Ctrl+Enter` closes a Pomodoro, every sub-bullet that is a Task
   Link and has sub-sub-bullets beneath it has those sub-sub-bullets copied into the
   linked task's Work Log, creating the `🛠️ **WORK LOG**` child when the task does not
   have one yet.

2. **Bug fix.** The Work Log marker and entry lines that `block-id-prompt` generates are
   indented with two/four spaces instead of tabs, which does not match how Obsidian
   indents list items when you press Tab, and does not match the rest of the toolchain.

The bug is confirmed below with vault evidence, not assumed.

## Why this is an epic

The two changes land in **two different plugins**, with **two different focused test
suites**, **two manifests**, and **two `bob plugins sync` deploys**. Phase 1 is a small,
self-contained correction to already-shipping output. Phase 2 is a substantially larger
feature that adds a new grammar, a new cross-file write path, and new cursor accounting
to the largest plugin in the repo. They are worth reviewing and releasing separately.

Phase 2 depends on Phase 1 only for serialization: both phases edit a row in the same
`README.md` plugin table, and Phase 2 must emit the tab convention that Phase 1
establishes. There is no code dependency — the plugins do not import each other.

## Repository

All implementation work is in the `bob-plugins` linked repository. Open it through
`/sase_repo` and use the printed checkout path for every read and write:

```bash
sase repo open bob-plugins -r "Copy Pomodoro task-link notes into task Work Logs"
```

Do not edit the deployed vault copies under `~/bob/.obsidian/plugins/`. `bob-plugins`
requires `bob plugins sync` after source changes, and that command defaults to the
canonical `~/projects/github/bobs-org/bob-plugins` checkout rather than a SASE workspace
clone. Every phase must therefore deploy with an explicit repo path:

```bash
bob plugins sync --repo <linked-checkout> --no-pull --plugin <plugin-id>
bob plugins sync --repo <linked-checkout> --no-pull --plugin <plugin-id> --dry-run
```

Read
`sase memory read glossary:Pomodoro "glossary:Work Log" "glossary:Task Link" -r "..."`
before changing behavior around these terms.

## Bug confirmation

`block-id-prompt` is the only plugin that writes Work Logs. Its fallback indentation is
two spaces:

```js
// plugins/block-id-prompt/main.js:95
const CANONICAL_POMODORO_CHILD_INDENT = "  ";
```

Used at four sites: `canonicalTaskChildIndent` (1674, the new marker's indent), the
no-existing-entry fallback under a found marker (1764), the entry indent under a freshly
created marker (1782), and the Pomodoro child fallback (2782).

Five independent pieces of evidence that two spaces is wrong:

1. **Obsidian's own default.** The vault's `.obsidian/app.json` sets only
   `promptDelete`, `showLineNumber`, `vimMode`, and `alwaysUpdateLinks`. It does not set
   `useTab` or `tabSize`, so Obsidian's defaults apply — `useTab: true`. Pressing Tab in
   a list inserts a literal tab.

2. **The same complaint was already filed and fixed once, elsewhere.** `bob-cli`'s
   `sdd/prompts/202606/obsidian_child_bullet_tab_indent.md` is Bryan reporting that
   generated bullets "don't render as proper Obsidian bullets" and asking that they look
   "just like I had created a new blank line and typed `-<tab>`". The fix changed the
   child-bullet unit from two spaces to a tab.

3. **`task-status-cycler` already migrated.** It defines
   `CHILD_BULLET_INDENT_UNIT = "\t"` with the comment "A literal tab matches how
   Obsidian indents list items via Tab and the vault's dominant nested-list source
   style, unlike the prior two-space indent", and
   `EMPTY_POMODORO_SUB_BULLET_LINE = "\t- "`.

4. **`bob-cli` already migrated, so the constant's comment is stale.** The comment on
   `CANONICAL_POMODORO_CHILD_INDENT` says it "Mirrors bob-cli's `bob capture` default",
   but `bob-cli`'s `src/native/capture.rs` now uses
   `child_indent.as_deref() .unwrap_or("\t")` and its Work Log fixtures are written
   `"\t- 🛠️ **WORK LOG**"`.

5. **`block-id-prompt` is internally inconsistent.** Its sibling Schedule Log path
   already uses a tab: `findScheduleLogEntryIndent` returns `` `${markerIndent}\t` ``
   (1826). Only the Work Log path was left behind.

Vault evidence — plugin-generated logs are space-indented, a hand-authored one is
tab-indented, and the surrounding nested content in the same file uses tabs:

```text
sase_art.md:47:  - 🛠️ **WORK LOG**              <- 2 spaces (generated)
sase_art.md:48:    - *2026-08-18* — ...          <- 4 spaces (generated)
sase_art.md:44:\t\t- *2026-08-16 → 2026-08-18* … <- tabs (hand-authored, same file)
sase_clean_release.md:11:\t- 🛠️ **WORK LOG**     <- tabs
sase_clean_release.md:12:\t\t- *2026-08-26* — …  <- tabs
```

The bug is confirmed. Note that raw vault-wide bullet counts still favor spaces (~41k vs
~2.9k) because of years of migrated pre-Obsidian content; that is history, not the
current convention, and must not be read as a counter-argument.

## The feature, in real vault terms

From `2026/20260826.md`, tabs shown as `\t`:

```text
- [ ] (**1235-1340** [t:: 65m])
\t- 🍅 ~~[[sase_monitor_gate#^epic]]~~
\t\t- Read `research.15`!
\t\t- Launched `0eg` agent to create epic!
\t- 🍅 [[sase_clean_release#^prj]]
\t\t- Launched `research.18`!
\t- ~~[[sase_monitor_gate#^epic]]~~
```

Closing this Pomodoro must prepend to `sase_monitor_gate.md`'s `^epic` task Work Log:

```text
\t- 🛠️ **WORK LOG**
\t\t- *2026-08-26* — Read `research.15`!
\t\t- *2026-08-26* — Launched `0eg` agent to create epic!
```

and to `sase_clean_release.md`'s `^prj` task Work Log:

```text
\t\t- *2026-08-26* — Launched `research.18`!
```

The sub-sub-bullets stay where they are in the daily note. This is a copy, not a move:
the closed Pomodoro must remain a faithful record of the session.

## Phases

### Phase 1 — `worklog-indent`

**Tab-indent generated Work Log markers and entries.** No dependencies.

Scope is `plugins/block-id-prompt/main.js` plus `scripts/test-block-id-prompt.cjs`,
`plugins/block-id-prompt/manifest.json`, and the Block ID Prompt row of `README.md`.

#### Required behavior

Replace the fixed two-space constant with a derived indent **unit**, so that a new child
level is one Obsidian Tab deeper than its parent, while legacy space-indented notes keep
their own style instead of growing mixed indentation:

```js
// One Obsidian Tab-indent level below `parentIndent`. A literal tab is what
// Obsidian inserts for Tab (the vault leaves `useTab` at its default), and it
// matches bob-cli's `bob capture` child-indent default and this file's own
// Schedule Log path. A space-only parent indent is a legacy note style: reuse
// it as the unit so the child stays space-indented instead of becoming a
// tab/space mix.
const CANONICAL_CHILD_INDENT_UNIT = "\t";

function childIndentUnitForIndent(parentIndent) {
  const indent = String(parentIndent || "");
  return indent && !indent.includes("\t") ? indent : CANONICAL_CHILD_INDENT_UNIT;
}
```

Apply it at all four current `CANONICAL_POMODORO_CHILD_INDENT` sites:

| Line | Current                                               | Required                                                          |
| ---: | ----------------------------------------------------- | ----------------------------------------------------------------- |
| 1674 | `canonicalTaskChildIndent` → `taskIndent + "  "`      | `taskIndent + childIndentUnitForIndent(taskIndent)`               |
| 1764 | fallback entry indent → `marker.indent + "  "`        | `marker.indent + childIndentUnitForIndent(marker.indent)`         |
| 1782 | entry under a new marker → `directChildIndent + "  "` | `directChildIndent + childIndentUnitForIndent(directChildIndent)` |
| 2782 | Pomodoro child fallback → `"  "`                      | `CANONICAL_CHILD_INDENT_UNIT`                                     |

Remove the stale "Mirrors bob-cli's `bob capture` default" claim; `bob capture` no
longer defaults to two spaces.

Nothing else changes. In particular `findWorkLogMarker`, `findWorkLogEntryPrefix`, and
`findTaskDirectChildPrefix` must keep **inheriting** an existing sibling's exact prefix,
so notes that already have a space-indented Work Log keep getting space-indented entries
and no existing note is reflowed. Only content with nothing to inherit from changes.

For a column-0 task — the overwhelmingly common case — the new output is:

```text
- [ ] #task Ship it ^ship
\t- 🛠️ **WORK LOG**
\t\t- *2026-08-15* — Added guarded cleanup
```

#### Hazard to respect

`lineIndentWidth` counts raw characters, so a tab counts as 1 column. Nesting detection
therefore breaks if a tab-indented line and a space-indented line are made siblings.
`childIndentUnitForIndent` exists precisely to avoid producing that mix; do not
"simplify" it back to an unconditional tab.

#### Tests

Update the existing expectations in `scripts/test-block-id-prompt.cjs` that assert the
two-space output — they are at roughly lines 2252, 2262, 2269, 2290, 2299, 2307, 2309,
2314, 2321, 2372, 2767, 2901, 2983. These are output-format assertions for behavior this
phase deliberately changes, so editing them is correct.

Add new coverage for:

- a column-0 task with no children gets `"\t- 🛠️ **WORK LOG**"` and
  `"\t\t- *<date>* — <summary>"`;
- a task that already has a **tab**-indented direct child inherits the tab prefix;
- a task that already has a **two-space**-indented direct child still inherits two
  spaces, and its new entry is four spaces — no `"  \t"` mix anywhere in the output;
- an existing tab-indented Work Log marker with no entries yet gets a `"\t\t"` entry;
- CRLF content is preserved.

#### Release

Bump `plugins/block-id-prompt/manifest.json` to `1.11.0` (generated output format
changes, so this is a minor bump, not a patch) and update the Block ID Prompt row in
`README.md` to say Work Log entries are recorded under a tab-indented managed
`🛠️ **WORK LOG**` child bullet.

#### Verification

```bash
node --test scripts/test-block-id-prompt.cjs
npm run validate
git diff --check
bob plugins sync --repo <checkout> --no-pull --plugin block-id-prompt
bob plugins sync --repo <checkout> --no-pull --plugin block-id-prompt --dry-run
```

---

### Phase 2 — `pomodoro-worklog-notes`

**On Ctrl+Enter, copy each Task Link sub-bullet's sub-sub-bullets into that task's Work
Log.** Depends on `worklog-indent`.

Scope is `plugins/task-status-cycler/main.js` plus
`scripts/test-task-status-cycler.cjs`, `plugins/task-status-cycler/manifest.json`, and
the Task Status Cycler row of `README.md`.

#### Where this lives, and why it duplicates grammar

`Ctrl+Enter` Pomodoro completion is owned by `task-status-cycler`
(`completeActivePomodoroTask`, ~8527). Work Log grammar lives in `block-id-prompt`.
Deployed plugins must not import one another, so `task-status-cycler` duplicates the
parts it needs — the same convention it already follows for `POMODOROS_HEADING_RE`,
`POMODORO_PLACEHOLDER_RE`, and the named-Pomodoro grammar copied from
`bob-navigation-hotkeys`.

The duplicated grammar must stay byte-compatible with `block-id-prompt`'s, because both
plugins (and `bob-cli`'s `src/native/capture.rs`, a third implementation) read and write
the same logs:

```js
const WORK_LOG_EMOJI = "🛠️";
const WORK_LOG_LABEL = "WORK LOG";
const LEGACY_WORK_LOG_LABEL = "Work log";
const WORK_LOG_PARENT_RE = new RegExp(
  `^([ \\t]*)([-*+]|\\d+[.)])[ \\t]+(?:${WORK_LOG_EMOJI}[ \\t]+)?\\*\\*(?:${WORK_LOG_LABEL}|${LEGACY_WORK_LOG_LABEL}):?\\*\\*[ \\t]*$`,
);
const WORK_LOG_ENTRY_EMPHASIS = "*";
const WORK_LOG_ENTRY_SEPARATOR = " — ";
```

An entry line body is `` `*<YYYY-MM-DD>* — <summary>` ``. Copy Phase 1's
`childIndentUnitForIndent` rule too — new markers and entries must be tab-indented under
column-0 tasks, and must not create a tab/space mix under a legacy space-indented
parent.

Helpers `task-status-cycler` already has and must reuse rather than re-add:
`parseListItemPrefix` (453), `findNearestParentListItemLine` (461),
`findTaskChildBlockEndLine` (~482), `getLineIndentation` (~88), `formatLocalDate` (117),
`getDailyNoteDateFromPath` (124), `getStrikethroughSpans`,
`stripPomodoroMarkersFromLine`, `parseEmbeddedBlockTransclusions`,
`parseNonEmbeddedBlockLinks`, `isProperObsidianTaskLine` (896),
`getLineTextFromSourceText` (4216), `splitTextByLineEndings` (3636),
`insertEditorLines`, `getEditorLineTexts`, `fileMatchesPath`,
`resolveTranscludedBlockTarget` (9830).

#### 1. Which sub-bullets qualify

A **note group** is a direct child of the completed Pomodoro entry that is a Task Link
and has at least one nonblank descendant line.

- **Direct child**: a line `L` inside
  `getSubBulletBlockRange(lines, pomodoroLine, section)` with
  `findNearestParentListItemLine(lines, L) === pomodoroLine`.
- **Task Link**: after `stripPomodoroMarkersFromLine` and after removing one enclosing
  `~~ ~~` pair, the bullet's body is **exactly one** block link and nothing else —
  embedded (`![[X#^y]]`) or not (`[[X#^y]]`), and including a trailing `#` deferral
  marker. Aliased links (`[[X#^y|label]]`) qualify.
- **Descendants**: lines after `L`, up to `findTaskChildBlockEndLine(lines, L)`, that
  are indented deeper than `L`.

This is deliberately **not** `classifyPomodoroSubBullets`. That function classifies for
carry-forward, where struck (`~~…~~`) links are filtered out by `rangeIsStruck` and land
in `noteBullets`. For Work Log purposes a struck link is the _most_ important case — it
is a task you finished this session — as the `20260826.md` sample above shows. Reusing
the carry-forward classification would silently skip the common case. Do not change
`classifyPomodoroSubBullets`; add a separate pass.

Skipped, with no write and no error:

- bullets with zero or two-or-more block links (ambiguous — which task owns the notes?);
- bullets whose link is not the sole body content (`- Mix ![[A#^a]] with [[B#^b]]`);
- bullets with no nonblank descendants;
- links that do not resolve, or resolve to a line that is not a proper Obsidian task
  (`isProperObsidianTaskLine`) — a link to a plain block must not grow a Work Log.

Two different sub-bullets may resolve to the **same** task. Both note sets are logged;
do **not** deduplicate by `path#^blockId`. Deduplication here would silently drop real
notes.

#### 2. What gets written

Each depth-1 descendant of the Task Link becomes one Work Log entry:

```text
<entryIndent>- *<date>* — <descendant body, trimmed>
```

Deeper descendants (depth 2+) are carried along beneath their entry, keeping their
relative depth, re-indented onto `entryIndent`. Blank descendant lines are dropped. A
descendant whose body is empty after trimming is dropped, and if that leaves a subtree
with no dated parent, drop the subtree.

`<date>` is the **daily note's own date**, not today:
`getDailyNoteDateFromPath(activePath)` falling back to `formatLocalDate()` when the
active note is not a daily note. Closing yesterday's Pomodoro must not stamp today's
date. Add `getPomodoroWorkLogDateString(activePath)` next to the existing
`getScheduleLogDateString()` so tests can stub it, and stub it in every new test.

Insertion follows `block-id-prompt`'s Work Log semantics exactly:

- marker found among the task's **direct children** (a marker belonging to a nested
  child task is ignored) → insert the entry block at `markerLine + 1`, newest first,
  entries in source order within the block;
- no marker → append `<childIndent>- 🛠️ **WORK LOG**` plus the entries at
  `findTaskChildBlockEndLine(lines, taskLine) + 1`;
- indentation is inherited from an existing sibling entry when one exists, otherwise
  derived with the Phase 1 unit rule;
- the target file's line ending (`\r\n` vs `\n`) is preserved.

#### 3. Where it runs in `completeActivePomodoroTask`

Run the copy **after** `applyPomodoroCompletionPlan` succeeds, using note text captured
from the **pre-plan** snapshot.

This ordering is required, not stylistic. The existing pre-plan cross-file phase
(`completePomodoroTranscludedTaskBullets`, `startPomodoroNonTranscludedTaskBullets`)
only _replaces_ lines, so line numbers are stable. Work Log writes _insert_ lines, and a
target task can live in the active note itself — `2026/20260826.md` has
`- [x] #task [[gtd_daily]] … ^gtd` above `## Pomodoros`, linked from Pomodoro
sub-bullets as `- [[#^gtd]]`, and `scripts/test-task-status-cycler.cjs` already
exercises that same-note shape. Inserting above the Pomodoros section before the plan is
built would leave the captured `context.pomodoroLine` and `cursor` stale and corrupt the
note. Running after avoids the problem entirely, because `resolveTranscludedBlockTarget`
resolves by block ID against freshly read content and self-heals any line shift.

Capture the note payload (text only, no line numbers) from the same `lines` snapshot the
sub-bullet classification uses, before any edits. The plan's marker rewriting can alter
a descendant line that happens to contain a link, so the payload must be taken before
it.

#### 4. Cursor accounting

`applyPomodoroCompletionPlan` sets and centers the cursor on the created Pomodoro. A
subsequent insertion into the **active** note above that line shifts it.

Record every active-note insertion as `{ line, count }` in the order applied. Then:

```js
let cursorLine = plan.cursorTargetLine;
for (const { line, count } of activeNoteInsertions) {
  if (line <= cursorLine) {
    cursorLine += count;
  }
}
```

Processing in application order is what makes this correct when insertions shift each
other. Re-set and re-center the cursor only when `cursorLine` actually moved.

#### 5. Writing

Mirror `applyBlockedStatusRetirementToTranscludedTarget` (~8402):

- target in the active note (`fileMatchesPath(file, activePath)`) → write through the
  editor with the existing `insertEditorLines`, keeping the live buffer and undo history
  coherent;
- otherwise → `vault.process` when available, else the `vault.read` + compare +
  `vault.modify` fallback that raises on concurrent change.

Two sub-bullets targeting the same file mean two sequential writes; each re-reads live
content and re-locates the marker, so both are independently correct — the same property
the existing double-write path documents at ~8396.

Best-effort per group: a broken link, an unreadable file, or a failed write must not
block the other groups and must never undo the completion, which has already been
applied. Log with `console.error`; do not add a `Notice`.

#### 6. Explicitly out of scope

Deferred `#`-marked (move-only) sub-bullets **do** get their notes copied — they are
Task Links you worked on. But `buildPomodoroCompletionPlan` removes only the `#`-marked
link line and leaves its descendants behind, orphaned one level too deep under the
closed Pomodoro. That is a pre-existing carry-forward defect, not something this phase
introduces or fixes. Do not change carry-forward semantics here. File it with
`/sase_new_task` as a `bug` task bead, referencing `buildPomodoroCompletionPlan`'s
`movedSourceLines` removal at ~3373.

Likewise, a _descendant_ line containing a block link is currently picked up by the flat
`classifyPomodoroSubBullets` scan and carried forward with its deep indentation. Also
pre-existing, also out of scope, also worth a bead.

#### Tests

Add to `scripts/test-task-status-cycler.cjs`. Do not weaken existing assertions; the
existing tests describe behavior that must not change.

Pure-helper coverage:

- Task Link recognition: plain, embedded, aliased, struck, `🍅`-marked, struck **and**
  marked, trailing `#`; rejects two links, a link plus prose, and a bare note.
- Note-group collection: depth-1 descendants become entries; depth-2 descendants nest
  under their entry with relative depth preserved; blank lines dropped; a Task Link with
  no descendants yields no group; a sub-bullet under a _different_ Pomodoro is not
  collected.
- Entry formatting: `*2026-08-26* — Read \`research.15\`!`.
- Insertion planning: new marker on a task with no children; prepend under an existing
  canonical marker; prepend under the legacy `**Work log**` spelling; a marker belonging
  to a nested child task is ignored; CRLF preserved; inherited indentation preserved.

End-to-end coverage through `completeActivePomodoroTask` with
`createInMemoryObsidianApp` + `createTextEditor` + `attachActiveMarkdownView`:

- the `20260826.md` shape above: two targets in two other notes, each getting the right
  entries, with the daily note's own date, and the daily note's sub-sub-bullets left in
  place;
- a target task **in the active note above `## Pomodoros`** (`- [[#^gtd]]`): the Work
  Log is inserted, the Pomodoro is still correctly completed, the created Pomodoro is
  correct, and the final cursor lands on the created Pomodoro's `()` — this is the
  regression test for §3 and §4;
- two sub-bullets linking the **same** task: both note sets appear, in source order,
  neither dropped;
- a task that already has a Work Log: entries are prepended above the existing ones;
- unresolvable link, and a link resolving to a non-task block: no write, completion
  still succeeds;
- a Task Link with no sub-sub-bullets: no Work Log is created (regression — closing an
  ordinary Pomodoro must not start writing Work Logs);
- a struck link with notes: notes still copied (the case carry-forward classification
  would have missed).

#### Release

Bump `plugins/task-status-cycler/manifest.json` to `1.15.0` and update the manifest
description and the Task Status Cycler row in `README.md` to mention copying a Task Link
sub-bullet's sub-sub-bullets into the linked task's Work Log on completion.

#### Verification

```bash
node --test scripts/test-task-status-cycler.cjs
npm test
npm run validate
git diff --check
bob plugins sync --repo <checkout> --no-pull --plugin task-status-cycler
bob plugins sync --repo <checkout> --no-pull --plugin task-status-cycler --dry-run
```

## Files expected to change

- `plugins/block-id-prompt/main.js`, `plugins/block-id-prompt/manifest.json`,
  `scripts/test-block-id-prompt.cjs` (Phase 1)
- `plugins/task-status-cycler/main.js`, `plugins/task-status-cycler/manifest.json`,
  `scripts/test-task-status-cycler.cjs` (Phase 2)
- `README.md` (one plugin-table row per phase)

## Files expected not to change

- `plugins/bob-navigation-hotkeys/`, `plugins/bob-ledger-tools/`,
  `plugins/bob-project-tasks/`, `plugins/bob-vim-surround/`
- Any Markdown note under `~/bob/`, other than the vault plugin files `bob plugins sync`
  deploys
- `bob-cli` source
- SASE memory files
