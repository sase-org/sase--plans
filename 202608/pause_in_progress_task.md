---
tier: tale
title: Pause In Progress tasks with an optional work summary
goal:
  Ctrl+Shift+Enter reliably pauses an In Progress Obsidian task, clears its live
  Pomodoro plans, and records a user-supplied work summary in a managed Work Log.
size: medium
proposed_by: bbugyi200.athena.028.f0.f0
create_time: 2026-09-09 20:00:16
status: wip
---

# Plan: Pause In Progress tasks with an optional work summary

## Goal

Extend the source-controlled `block-id-prompt` Obsidian plugin so the existing
`Ctrl+Shift+Enter` command treats an In Progress task (`[/]`) as a second reverse
operation. Instead of linking that task to a Pomodoro and promoting it to Next, open a
polished optional-summary prompt. Confirming the prompt must change the task to Open
(`[ ]`), remove every resolvable link to that task from today's current and future open
Pomodoros, and—only when the user entered non-whitespace text—record the summary under a
managed `WORK LOG` direct child of the task. Preserve the existing forward behavior for
Ready and Blocked tasks and the existing immediate reverse behavior for Next tasks.

This is a `medium` tale: the work is substantial because it combines a new modal, a
structured Markdown planner, and guarded same-note/cross-note writes, but it remains a
bounded change to one deployable plugin and its focused test suite.

## User-facing behavior

### Status-aware command dispatch

Keep the command ID, display name, and `Ctrl+Shift+Enter` keymap unchanged. After the
existing single-cursor, Markdown task, code-fence, open-status, and duplicate-block-ID
validation:

- Ready (`[ ]`) and Blocked (`[?]`) tasks retain the current forward flow: ensure a
  unique block ID, become Next, and link to today's current/next Pomodoro.
- Next (`[*]`) tasks retain the 1.8.0 reverse flow without a prompt: become Open and
  lose resolvable links from all of today's open Pomodoros.
- In Progress (`[/]`) tasks open the new work-summary prompt. Nothing—not even a daily
  note read—is changed before the user confirms.
- Any other status remains ineligible under the existing task-resolution rules.

An In Progress task does not need a block ID to be paused. Without one there can be no
safe block-link cleanup, so confirmation still changes the status and optionally writes
the work log without opening the block-ID prompt or requiring today's daily note.

### Work-summary prompt

Add a small, purpose-built modal whose language reflects the actual action rather than
the plugin's internal terminology:

- Header: a pause/work icon, `Pause task`, and a concise subtitle explaining that the
  task will become Open and be removed from current/future Pomodoros.
- Context card: the selected task's cleaned display text and source note, so the user
  can verify what will change before committing.
- A focused, single-line `Work summary (optional)` input with a human placeholder such
  as `What did you get done?`.
- A live outcome/preview card. Nonblank input previews the exact summary that will be
  nested beneath `🛠️ **WORK LOG**`; blank input clearly says that no work-log entry will
  be added. If the normalized text contains `::`, show the same kind of non-blocking
  Dataview-inline-field warning used by the schedule-reason experience.
- Theme-native, namespaced styling in `styles.css`, using Obsidian color, radius,
  spacing, and typography variables. Render user text via `textContent`/Obsidian element
  helpers, never `innerHTML`, and include useful labels/roles for keyboard and screen
  reader users.
- The primary button label changes between `Set Open` and `Set Open & log`; Enter
  confirms and Escape or Cancel dismisses the modal.

Cancellation is a clean abort: it changes no file and shows no misleading success
notice. Confirming an empty or whitespace-only value is intentionally different from
cancelling: it performs the reverse operation but creates no `WORK LOG`. Close the modal
only after a successful operation so the typed summary remains available across a
retryable guarded-write failure; always release `promptOpen`/prompt-key state on close.

Normalize a submitted summary by trimming it and collapsing runs of whitespace to one
space. Preserve ordinary Markdown, wikilinks, and backticks verbatim otherwise. The
stored entry is always one Markdown list-item line, which keeps the note structure
stable even if pasted input contained line breaks.

### Managed Work Log structure

The first nonblank summary creates this shape as the task's final direct-child subtree:

```markdown
- [ ] #task Ship the feature ^ship
  - 🛠️ **WORK LOG**
    - Added guarded cleanup and regression coverage
```

Model the structural rules on Schedule Log maintenance, without importing another
plugin's `main.js`:

- Canonically write `🛠️ **WORK LOG**`.
- Recognize the canonical marker plus hand-written emoji-less `**WORK LOG**` and legacy
  `**Work log:**` spellings, so an existing log is maintained rather than duplicated.
- Only a marker that is a direct child of the selected task qualifies; ignore a marker
  belonging to a nested child task or another subtree.
- When no marker exists, reuse the task's established direct-child indentation when
  possible, otherwise use the existing canonical child indentation; create the parent
  and one sub-sub-bullet as the task's final direct-child subtree.
- When a marker exists, preserve its indentation and list marker, reuse an existing
  direct entry's indentation when available, and insert the new summary immediately
  below the marker. Entries are therefore newest-first, matching Schedule Log history.
- Preserve unrelated child bullets, blank lines, CRLF/LF style, existing marker text,
  and all older entries exactly. If multiple direct-child markers already exist, use the
  first and leave the others untouched, matching the existing Schedule Log rule.
- A blank normalized summary produces no marker, no entry, and no whitespace-only
  structural edit—even if a Work Log already exists.

Implement the normalization, marker parsing, direct-child lookup, indentation choice,
entry formatting, and insertion as pure helpers. Fold the optional work-log insertion
into the task-side `[/] → [ ]` mutation plan so status and history can never diverge
within one task-note write. Keep the existing `[*] → [ ]` planner behavior unchanged;
either generalize it with an explicit expected source status and optional summary or
compose it from status and work-log sub-plans without weakening its status guard.

## Reliable mutation ordering

Reuse `planAllOpenPomodoroLinkCleanup` for the daily-note side. It must continue to
remove only exact, resolvable links for the task's note path and block ID from every
open Pomodoro entry (the current timed entry and future placeholders), prune a dedicated
link bullet with its descendants, remove only the token when it shares a bullet with
other content, and preserve closed Pomodoro history, fenced examples, nested
pseudo-entries, unrelated links, and unresolved/ambiguous references.

On summary confirmation, revalidate the selected task line and unique block ID against
the live editor before planning. Then follow the existing reverse-flow safety model:

- If the task and today's daily note are the same file, build the task/status/work-log
  plan and Pomodoro cleanup plan from one snapshot, validate that all edits are
  non-overlapping, and apply them as one descending-offset editor transaction. Restore
  the user's cursor afterward.
- If they are different files and cleanup has changes, guard and write the daily-note
  cleanup first, then guard and apply the combined task/status/work-log plan. This
  avoids claiming that work was paused/logged while its live Pomodoro plans remain in
  place.
- If the task has no block ID, today's daily note does not exist, or no matching open
  links exist, skip cleanup and still apply the task plan.
- If the daily note cannot be read, changed before its guarded write, or cannot be
  modified, stop with the task still In Progress and no work entry added.
- If daily cleanup succeeds but the task snapshot then becomes stale or cannot be
  written, report the exact retryable partial result: how many links were removed, that
  the task remains In Progress, and—when a summary was supplied—that the work summary
  was not logged. A retry must not duplicate anything.
- Suppress the plugin's automatic block-ID scanner around editor mutations and preserve
  existing behavior when the editor/vault APIs throw.

Success notices should concisely compose the actual outcome, for example
`Task set Open · logged work · removed 2 current/future Pomodoro links`; omit
`logged work` for blank input and retain the explicit no-links-removed suffix. Keep the
existing Next-task notices and partial-failure wording status-specific (`Next` versus
`In Progress`).

## Implementation scope

Work in the linked `bob-plugins` source-of-truth repository opened through `sase repo`.
Expected files are:

- `plugins/block-id-prompt/main.js`: Work Log constants/helpers/planner, the modal,
  status dispatch, guarded orchestration, and outcome reporting.
- `plugins/block-id-prompt/styles.css`: namespaced modal/context/preview/warning styles.
- `scripts/test-block-id-prompt.cjs`: pure planner, modal contract, dispatch, runtime,
  concurrency, and regression coverage.
- `plugins/block-id-prompt/manifest.json`: minor version bump from `1.8.0` to `1.9.0`.
- `README.md`: describe the In Progress pause/summary branch alongside the existing
  Ready/Blocked forward and Next reverse behavior.

Do not change the command ID/keymap, create a Work Log for Next-task reversals, add
timestamps or synthetic text to work summaries, touch closed Pomodoro history, or move
this behavior into another plugin.

## Test plan

Extend the existing Node test harness with focused coverage for:

1. Summary normalization: whitespace-only emptiness, collapsed pasted newlines/spaces,
   preserved Markdown/wikilinks/backticks, and the `::` warning signal.
2. Work Log planning: canonical first creation, append position after existing task
   children, newest-first insertion into canonical/emoji-less/legacy markers, reused
   indentation/list markers, nested-marker rejection, first-of-multiple behavior,
   blank-summary no-op, LF/CRLF preservation, and no unrelated text changes.
3. Status planner guards: In Progress changes only to Open, Next behavior remains
   unchanged, other statuses are rejected, and a nonblank summary produces one combined
   status-plus-log plan.
4. Command dispatch: `[/]` opens only the work-summary modal and performs no early I/O;
   Cancel is inert; blank confirmation pauses without a log; nonblank confirmation logs;
   `[*]` still reverses immediately; Ready/Blocked still use the link/block-ID path; the
   command registration and hotkey remain unchanged.
5. Runtime behavior without cleanup: no block ID, missing daily note, no Pomodoros
   section, and zero matching links all still set Open and log only when requested.
6. Cross-note success: cleanup is written before the task changes; exact open links are
   removed; closed history remains; status and log are applied together; cursor and
   success notice are correct.
7. Same-note success: cleanup, status, and new/existing Work Log edits merge into one
   guarded editor transaction without overlap or duplicate markers/entries.
8. Failure paths: changed task while the modal is open, duplicated block ID discovered
   before confirmation, unreadable/stale/unwritable daily note, stale/unwritable task
   after successful cleanup, resolver exceptions, and accurate partial-result notices
   with and without a supplied summary.
9. Regressions for the complete pre-existing `Ctrl+Shift+Enter` forward and Next reverse
   matrices, including Schedule Log pull-forward behavior and all-open cleanup parsing.

## Verification and deployment

From the opened `bob-plugins` checkout:

1. Run `node --test scripts/test-block-id-prompt.cjs` while iterating.
2. Run the full `npm test` suite.
3. Run `npm run validate` for all plugin manifests.
4. Preview deployment with
   `bob plugins sync --dry-run --no-pull --repo . --plugin block-id-prompt`.
5. Deploy with `bob plugins sync --no-pull --repo . --plugin block-id-prompt` as
   required by that repository's instructions.
6. Re-run the dry-run and `bob plugins list --no-pull --repo . --format json` to verify
   `block-id-prompt` 1.9.0 is enabled and synced with zero pending copies.

The implementation is complete when every status branch has the behavior above, a
cancelled or failed prompt cannot leave a false Work Log entry, cross-file failures are
reported truthfully and remain safely retryable, all focused/full tests and validation
pass, documentation matches the shipped interaction, and the source-controlled plugin is
synced to the vault.
