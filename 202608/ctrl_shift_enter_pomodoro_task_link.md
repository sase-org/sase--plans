---
tier: tale
title: Add Ctrl+Shift+Enter Pomodoro task linking
goal:
  Ctrl+Shift+Enter safely links the task under the cursor to today's current or next
  Pomodoro while ensuring task identity, Next status, and schedule history.
size: medium
proposed_by: bbugyi200.athena.028
create_time: 2026-09-09 19:59:59
status: wip
---

# Add Ctrl+Shift+Enter Pomodoro task linking

## Goal

Extend the source-controlled `block-id-prompt` Obsidian plugin so `Ctrl+Shift+Enter`
associates the open Obsidian task under the single cursor with today's current/next
Pomodoro. The command must create a canonical sole-content task block-link sub-bullet,
ensure the task has a unique block ID, activate the task as Next, preserve the
scheduling history conventions already used by the special `^^` Pomodoro task-picker
flow, and fail without clobbering concurrent editor or vault changes.

This is a single `medium` tale because the behavior belongs to one deployable plugin and
one test suite, but it needs coordinated planning and application of two note mutations
plus a modal continuation when the task lacks a block ID. No `bob-cli` feature change is
expected; its Pomodoro-linked capture behavior is the compatibility reference and its
existing plugin-sync command is the deployment mechanism.

## Existing behavior to preserve and reuse

- Work in the linked `bob-plugins` source-of-truth repository (opened through
  `/sase_repo`), not in the deployed copies under `~/bob/.obsidian/plugins/`.
- `plugins/block-id-prompt/main.js` already recognizes open Obsidian task statuses,
  validates note-wide block-ID uniqueness, prompts for a missing ID, resolves block
  links, prunes duplicate links from later open Pomodoros, and applies guarded task-note
  writes.
- Its special `^^` flow already has the scheduling activation primitive: remove exactly
  one strictly-future valid `scheduled` field, choose the Next status, and prepend an
  entry such as `_2026-08-20 → 2026-08-15_ — 🍅 pulled into today's Pomodoro` when the
  task already owns a direct-child `SCHEDULE LOG`. It deliberately does not create a log
  for a task that keeps none and leaves ambiguous, malformed, today, or past scheduling
  metadata intact.
- Match `bob capture` when choosing the destination ledger entry: find today's
  `## Pomodoros` section, ignore fenced/nested/closed entries, prefer the single open
  top-level entry with a recognized time range, otherwise use the first open top-level
  entry (the next placeholder), and reject multiple open timed entries. Insert after the
  selected Pomodoro's existing child block, reusing its child indentation when possible
  and the section's established child indentation before falling back to the canonical
  indentation.
- Resolve today's daily-note path from Obsidian's enabled Daily Notes plugin options
  (`folder` and `format`) using local calendar components, consistent with
  `bob-ledger-tools`; do not hard-code the currently configured `YYYY/YYYYMMDD.md`
  layout.
- Register the mapping as the new command's exact default Obsidian hotkey (`Ctrl` +
  `Shift` + `Enter`). Do not hand-edit the live vault's `.obsidian/hotkeys.json`; a new
  command with this default has no existing user override and remains source-controlled
  with the plugin.

## Implementation

### 1. Discover and validate the task and destination before writing

- Add a clearly named editor command to `block-id-prompt` with the `Ctrl+Shift+Enter`
  default hotkey. Its check/dispatch path must require one cursor in an active Markdown
  file and an unfenced open Obsidian task line under that cursor. Keep this
  direct-cursor workflow separate from task-picker filtering so it can link the task the
  user explicitly selected (including a `#hide` project task); reject non-task,
  closed/canceled, frontmatter, fenced, and multi-selection contexts with focused
  notices and no edits.
- Capture a stable source identity: file path, line number, exact original task line,
  original status, existing trailing block ID if any, and enough block context to
  revalidate after asynchronous work. Reject an existing block ID that is duplicated
  elsewhere in the source note rather than creating an ambiguous link.
- Resolve and read today's daily note and produce a pure destination plan using the
  `bob capture` target-selection rules above. Missing daily note, missing `Pomodoros`
  section, no eligible open entry, or multiple open timed entries must stop the command
  before any task mutation.
- Construct a canonical block link from the source file and block ID: use a same-note
  `[[#^id]]` link when the task lives in today's daily note and an unambiguous vault
  path without the `.md` suffix otherwise. Treat an already resolved matching link
  beneath the selected Pomodoro as idempotent rather than appending a duplicate, and
  plan removal of matching occurrences from later open Pomodoros just as `^^` does.

### 2. Reuse the block-ID prompt as a continuation of the whole operation

- When the task already has one unique valid trailing block ID, proceed directly with
  the guarded activation/link operation.
- When it has none, open the existing Block ID modal with the task preview and a new
  source/continuation kind dedicated to this command. Canceling the modal must leave
  both notes byte-for-byte unchanged. Submitting must re-read and revalidate the task,
  proposed ID uniqueness, daily-note target, and both preimages before applying
  anything; stale task text, a newly duplicated ID, or a changed/invalid Pomodoro ledger
  must stop cleanly.
- Keep the modal's existing validation contract (nonblank letters, digits, and hyphens),
  and append a newly accepted `^id` as the final token after all surviving task
  metadata.

### 3. Plan task activation and Pomodoro insertion together

- Refactor/extend the existing pure target-task planner with an explicit mode for this
  command. Preserve the `^^` behavior for its current callers, while the new mode forces
  every eligible original open status (`[ ]`, `[/]`, `[*]`, or `[?]`) to Next (`[*]`),
  satisfying the new command even for an unscheduled Blocked or In Progress task.
- Reuse the `^^` future-schedule behavior rather than duplicating it: if there is
  exactly one strictly-future valid bracketed or parenthesized `scheduled` field, remove
  it. If the original task was Blocked and already has a direct child Schedule Log, this
  produces the required old-date-to-today entry with the existing Pomodoro reason; the
  same established behavior should remain available for a Ready task with a future
  schedule. Never create a Schedule Log solely because this command ran, and do not
  fabricate a date transition when there is no unique future date to transition from.
- Build pure, offset-based edits for the source task (status, optional schedule removal,
  optional block-ID append, optional log entry) and daily note (selected-Pomodoro
  insertion/idempotence plus later-link cleanup). Preserve CRLF/LF style,
  trailing-newline state, unrelated whitespace, child content, aliases/embeds outside
  managed matching occurrences, and cursor position.
- For the same-note case, merge non-overlapping edits and apply them in one editor
  transaction/undo group with line/offset shifts derived from the original snapshot. For
  different notes, validate both snapshots first, use the live editor for any open
  buffer and guarded vault writes otherwise, and serialize the mutations so stale
  content is never overwritten. If the second write fails after the first succeeds,
  report the exact partial result instead of claiming the link and activation both
  completed.
- Emit concise outcome notices whose chips are based on actual plan metadata
  (linked/already linked, added block ID, set Next, removed future schedule, logged
  schedule change, and removed later duplicate links).

### 4. Document, version, test, and deploy

- Add focused pure and runtime coverage in `scripts/test-block-id-prompt.cjs`, expanding
  its Obsidian stubs only as needed. Cover:
  - command registration and the exact `Ctrl+Shift+Enter` default;
  - task-under-cursor eligibility, including `#hide`, fence, closed-task,
    multi-selection, and stale-line guards;
  - daily path formatting and current-versus-next selection, plus missing
    daily/section/target and multiple-timed-entry failures;
  - insertion placement/indentation, canonical same-note and cross-note links,
    idempotence, and cleanup of later duplicates;
  - existing-ID and prompted-ID success, invalid/duplicate ID retry, modal cancellation,
    and stale state while the modal is open;
  - force-to-Next behavior for Ready, In Progress, Next, and Blocked tasks;
  - Blocked plus future `scheduled` plus existing Schedule Log, absence of a Schedule
    Log, and ambiguous/malformed/nonfuture schedule preservation;
  - same-note one-transaction edits, cross-note guarded edits and partial failure
    notices, LF/CRLF/trailing-newline preservation, and cursor stability.
- Bump `plugins/block-id-prompt/manifest.json` by one minor version and update the
  plugin table/description in `README.md` to document the command, hotkey, current/next
  selection, missing-ID prompt, Next transition, and Schedule Log behavior.
- Run `npm test` and `npm run validate` from the `bob-plugins` repository. Then follow
  that repository's deployment rule: dry-run and perform
  `bob plugins sync --no-pull --repo . --plugin block-id-prompt` from the opened source
  repository, and verify the deployed plugin reports `synced` without editing its vault
  copy directly.

## Acceptance criteria

- Pressing `Ctrl+Shift+Enter` on an eligible task creates exactly one canonical task
  block-link child under today's current timed Pomodoro, or under the first next open
  Pomodoro when no timed one exists.
- A task without an ID prompts before any mutation; cancel is a complete no-op, and a
  valid unique submitted ID becomes the task's final token and the link's target.
- The task ends in Next status. A uniquely future schedule is removed, and a previously
  Blocked task with an existing Schedule Log gets the same dated pull-into-today entry
  used by the `^^` flow; a task without a Schedule Log does not gain one.
- Invalid ledger state, invalid/ambiguous task identity, duplicate IDs, stale preimages,
  and write failures never silently overwrite user work and produce accurate notices.
- Repeating the command does not duplicate the selected Pomodoro link, later
  open-Pomodoro duplicates are removed, all plugin tests and manifest validation pass,
  and the source changes are synced to the Bob vault through `bob plugins sync`.
