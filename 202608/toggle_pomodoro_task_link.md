---
tier: tale
title: Toggle Next tasks out of current and future Pomodoros
goal:
  Ctrl+Shift+Enter toggles a Next task back to Open and removes its links from today's
  current and future Pomodoros without disturbing closed history.
size: medium
proposed_by: bbugyi200.athena.028.f0
create_time: 2026-09-09 20:00:33
status: wip
---

# Toggle Next tasks out of current and future Pomodoros

## Goal

Extend the source-controlled `block-id-prompt` Obsidian plugin so the existing
`Ctrl+Shift+Enter` command becomes status-aware. Its current behavior remains unchanged
for Ready, In Progress, and Blocked tasks: ensure a block ID, set the task to Next, and
link it to today's current/next Pomodoro. When the task under the cursor is already Next
(`[*]`), the same keymap instead changes it to the unchecked Open/Ready status (`[ ]`)
and removes every resolvable link to that task from today's current and future (open)
Pomodoros, while preserving closed Pomodoro history and unrelated content.

This is a single `medium` tale because the behavior stays within one deployable plugin
and its existing test suite, but requires coordinated changes to pure task/link
planners, the modal-free Next branch, same-note transactions, cross-note write ordering,
outcome reporting, documentation, versioning, and vault deployment.

## Existing behavior to preserve and reuse

- Work in the linked `bob-plugins` source-of-truth repository (opened through
  `/sase_repo`), never in the deployed copy under `~/bob/.obsidian/plugins/`.
- Preserve the command ID `link-task-to-pomodoro` and its exact default hotkey (`Ctrl` +
  `Shift` + `Enter`) so existing Obsidian hotkey overrides continue to work. Rename the
  user-facing command label if needed to accurately describe the new toggle behavior.
- Keep the current task-under-cursor eligibility contract: a single cursor in an active
  Markdown file, outside frontmatter/fences, on an open `#task` line (including
  `#hide`), with a unique existing trailing block ID when one is present.
  Closed/canceled tasks, stale task text, and duplicate block IDs remain hard failures
  with no unsafe edits.
- Keep the activation path unchanged for non-Next statuses, including the missing-ID
  modal, unique-ID validation, current/next Pomodoro selection, canonical link
  insertion, force-to-Next behavior, future `scheduled` removal, existing Schedule Log
  convention, idempotence, later-link pruning, and current notices.
- Reuse the existing wiki-block-reference parsing, metadata-cache target resolution,
  fenced-line exclusion, dedicated-bullet subtree removal, mixed-content token removal,
  alias/embed/same-note handling, newline preservation, and guarded editor/vault write
  primitives. The related all-open cleanup in `bob-navigation-hotkeys` is a
  compatibility reference, but deployed plugins must not import one another.

## Implementation

### 1. Make the command dispatch an explicit Next-to-Open branch

- After the existing editor/task eligibility and duplicate-ID checks, branch on the
  captured task status. A task whose status is exactly Next (`*`) enters a new unlink
  continuation; all other eligible open statuses retain the existing link continuation.
- Do not open the block-ID modal on the Next-to-Open branch. A Next task without a block
  ID cannot be the target of a valid task block link, so change it to `[ ]` directly and
  treat Pomodoro cleanup as an empty plan. Do not invent or remove a block ID.
- Retain the captured source path, line, exact raw task line, status, and existing ID so
  asynchronous unlink work revalidates the same task identity before every mutation. The
  command's reentrancy guard must cover both link and unlink continuations.

### 2. Add a pure all-open Pomodoro cleanup plan

- Generalize the existing future-link cleanup machinery, or add a narrowly named
  wrapper, that can select every open top-level Pomodoro entry in today's `## Pomodoros`
  section rather than only entries after a chosen owner. Here “current/future” means the
  current open timed entry plus all open placeholders/entries; completed and canceled
  entries and their descendants are immutable history.
- Match candidates by both the task's unique block ID and metadata-resolved destination
  path, including canonical same-note `[[#^id]]`, explicit paths, aliases, and embeds.
  Never guess unresolved or ambiguous destinations, never touch matching links outside
  the Pomodoros section or inside fenced examples, and preserve nonmatching task links.
- Preserve the established cleanup semantics: remove the whole list-item subtree when a
  matched link is the dedicated bullet body; otherwise remove only matching link tokens
  (and wrappers/markers already consumed by this plugin's cleanup primitive) without
  clobbering unrelated bullet text. Deduplicate/merge overlapping edits, preserve
  LF/CRLF and trailing-newline state, expose the edit list and exact removed-link count,
  and make missing section/no open entries/no matches a successful no-op rather than an
  insertion- style ledger error.
- Keep the existing later-only cleanup API and behavior intact for the `^^` picker and
  non-Next link path; share lower-level range/match/edit construction to prevent the two
  cleanup modes from drifting.

### 3. Plan and apply the inverse status mutation safely

- Extend the pure target-task planner with an explicit inverse mode (or an equally clear
  dedicated planner) that changes only an original Next checkbox from `[*]` to `[ ]`.
  This inverse must not remove or synthesize `scheduled` metadata, append a Schedule Log
  entry, alter the block ID, or affect unrelated task text. Return discrete offset
  edits, a complete postimage, and outcome metadata suitable for guarded writes and
  notices.
- For a task and today's daily note in the same file, merge the status edit and all-open
  cleanup edits against one preimage, verify that they do not overlap, recheck the live
  editor snapshot, and apply them in one descending-offset editor transaction/undo
  group.
- For different files, read and plan both snapshots before writing. If cleanup has
  edits, guard and write the daily note first, then guard and change the task to Open.
  This order is intentional: if the second step fails, the task remains Next and
  pressing the keymap again safely retries the same inverse branch; changing the task
  first would make a retry incorrectly enter the link direction. If the daily note has
  no cleanup changes, update only the task.
- A missing today's daily note is equivalent to having no current/future Pomodoro links
  for this inverse operation, so it must not prevent a Next task from becoming Open.
  Likewise, a missing section, no open entries, or zero matching links is a clean
  status- only success. Actual daily-note read/write failures when a resolvable daily
  file exists must stop before changing the task; a task write/freshness failure after
  successful cleanup must report that links were removed but the task remains Next.
- Emit accurate, direction-specific notices based on actual plan metadata: task set
  Open, zero/one/many current-or-future Pomodoro links removed, no-op cleanup, and
  partial failure. Never reuse activation text such as “set Next,” “linked,” or Schedule
  Log chips for the inverse branch.

### 4. Document, version, test, and deploy

- Add focused pure and runtime coverage in `scripts/test-block-id-prompt.cjs` for:
  - unchanged command ID/default hotkey and status-aware dispatch;
  - Next with and without an existing block ID, including proof that the missing-ID case
    does not open a modal or create an ID;
  - exact `[*]` to `[ ]` mutation with task metadata, schedule fields, Schedule Log,
    block ID, whitespace, CRLF/LF, trailing-newline state, and cursor position
    preserved;
  - removal from a current timed Pomodoro and every future open Pomodoro, including
    multiple duplicates, aliases, embeds, same-note links, dedicated bullets with nested
    children, and mixed-content bullets;
  - preservation of closed/canceled history, earlier completed Pomodoros, links outside
    the section, fenced examples, unrelated IDs/paths, and unresolved/ambiguous links;
  - missing daily note, missing section, no open entries, and no matching links as
    status-only successes;
  - stale task/daily preimages, duplicate task IDs, daily-read/write failure before the
    status edit, same-note one-transaction behavior, cross-note cleanup-before-status
    ordering, and the retryable partial failure where cleanup succeeds but the task
    write does not;
  - regression coverage proving Ready, In Progress, and Blocked tasks still use the
    existing link/prompt/force-Next/schedule-log path and the `^^` later-only cleanup is
    unchanged.
- Bump `plugins/block-id-prompt/manifest.json` by one minor version and update the
  plugin table in `README.md` to describe `Ctrl+Shift+Enter` as a toggle: non-Next tasks
  link and become Next, while Next tasks become Open and are removed from current/future
  Pomodoros.
- Run the focused block-ID prompt tests, then `npm test` and `npm run validate` from the
  opened `bob-plugins` repository. Follow the repository deployment rule by dry-running
  and performing `bob plugins sync --no-pull --repo . --plugin block-id-prompt`, then
  verify the deployed plugin reports `synced` without editing the vault copy directly.

## Acceptance criteria

- Pressing `Ctrl+Shift+Enter` on a Next task changes only its checkbox from `[*]` to
  `[ ]`, leaves its ID/scheduling/history metadata intact, and removes every resolvable
  matching task block link beneath today's open Pomodoro entries.
- The inverse branch never prompts for or creates a block ID. A Next task without an ID,
  or with no current/future link to remove, still becomes Open successfully.
- Closed/canceled Pomodoro history, unrelated links/content, fenced examples, and
  unresolved/ambiguous destinations are not modified.
- Same-note changes form one guarded transaction. Cross-note cleanup happens before the
  task status change, and any partial failure is reported accurately and remains safe to
  retry with `Ctrl+Shift+Enter`.
- Ready, In Progress, and Blocked tasks retain the existing link-to-Pomodoro behavior,
  including missing-ID prompting, force-to-Next, future-schedule removal, and Schedule
  Log handling.
- Focused tests, the full plugin test suite, manifest validation, and vault sync all
  pass.
