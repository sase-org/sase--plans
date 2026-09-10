---
tier: tale
title: Add beautiful date prefixes to Work Log entries
goal:
  Newly recorded Work Log summaries carry a clear local ISO-date prefix that is
  previewed faithfully and preserves all existing task and Pomodoro safety behavior.
size: small
proposed_by: bbugyi200.athena.028.f0.f0.f1
create_time: 2026-09-09 20:00:00
status: wip
---

# Plan: Add beautiful date prefixes to Work Log entries

## Goal

Polish the `block-id-prompt` plugin's existing `Ctrl+Shift+Enter` pause flow so every
nonblank Work Log summary created for an In Progress (`[/]`) task is prefixed with the
local calendar date. A newly written entry should use this canonical Markdown shape:

```markdown
- [ ] #task Ship the feature ^ship
  - 🛠️ **WORK LOG**
    - _2026-08-15_ — Added guarded cleanup and regression coverage
```

The italic ISO date and em-dash separator deliberately mirror the established Schedule
Log visual language. ISO `YYYY-MM-DD` is compact, locale-independent, chronologically
sortable, and unambiguous; keeping it as plain emphasized text avoids fragile
assumptions about Daily Notes folders or custom filename formats. The change must apply
only to new entries. Existing undated or hand-written entries remain byte-for-byte
untouched.

This is a `small` tale: the feature is a focused refinement of the pure Work Log planner
and its already-polished prompt, with bounded tests, documentation, versioning, and
deployment work in one plugin.

## User-facing behavior

- Keep the existing command ID and `Ctrl+Shift+Enter` keymap unchanged.
- Ready/Blocked forward linking, Next-task reverse behavior, and every Pomodoro cleanup
  rule remain unchanged.
- When an In Progress task is paused with a nonblank summary, create or maintain its
  managed `🛠️ **WORK LOG**` exactly as today, but write the new sub-sub-bullet as
  `*YYYY-MM-DD* — <normalized summary>`.
- Resolve the date from local calendar components, never `toISOString()`, so pausing
  near midnight cannot shift the entry to the previous or next day because of UTC
  conversion.
- Capture the local date once for the prompt interaction and use that same value in both
  the preview and the eventual write. This makes the promise on screen match the stored
  entry exactly and gives a prompt left open across midnight a stable, unsurprising
  date.
- A blank or whitespace-only summary still sets the task Open and performs eligible
  Pomodoro cleanup without creating a Work Log marker or entry; no standalone date or
  separator is ever emitted.
- Repeated pauses on one day create distinct newest-first entries bearing the same date.
  Do not merge, deduplicate, or overwrite work summaries.
- Do not migrate or rewrite existing Work Log entries, including legacy entries without
  dates, entries with hand-written date styles, or content beneath legacy marker
  spellings.

## Formatting and preview design

Centralize the output contract in a pure formatter rather than assembling the prefix in
the writer and modal independently. Reuse the plugin's existing
`localTodayParts`/`formatCalendarDate` helpers and define Work Log-specific emphasis and
separator constants consistent with the canonical Schedule Log form. The formatter
should accept normalized summary text and an explicit date value and produce exactly:

```text
*2026-08-15* — Added guarded cleanup
```

Thread an injected/captured date through `planTargetTaskOpenUpdate` into
`planWorkLogInsertion`; avoid hidden wall-clock reads inside the lowest-level text
planner. Runtime callers that do not come through the modal may fall back to one
`plugin.now()` snapshot, preserving the existing test seam and ensuring the planner is
always deterministic when a date is supplied.

Enhance the existing live preview so nonblank input shows the dated entry users will
receive, not only the raw summary. Visually render the date in italic, the em dash in a
muted tone, and the summary as normal text within the existing theme-native card. Keep
wrapping graceful on narrow screens and preserve the existing marker preview,
blank-summary message, Dataview `::` warning, keyboard behavior, focus handling, and
button labels. Build preview spans with Obsidian DOM helpers/text content rather than
`innerHTML`; expose a coherent accessible label/text equivalent to the complete entry.

## Markdown mutation contract

Retain all structural behavior from version 1.9.0:

- Create the canonical Work Log marker only for a nonblank first entry.
- Recognize canonical, emoji-less, and legacy markers without rewriting their text.
- Insert new entries immediately below the marker so history remains newest-first.
- Preserve the marker's list character and reuse existing direct-entry indentation; use
  the canonical indentation only when creating a new structure.
- Ignore nested child-task markers, preserve unrelated descendants, and preserve LF or
  CRLF line endings.
- Keep the status edit and Work Log insertion in the same task-note mutation plan, and
  keep same-note/cross-note Pomodoro cleanup ordering, stale-write guards, cursor
  restoration, notices, and retry behavior unchanged.

The result metadata should continue to expose the normalized summary and whether an
entry was added; add formatted-entry/date metadata only if it makes the modal/runtime
contract clearer. No parsing or validation of historical entry text is needed because
this feature writes new entries but never interprets old ones.

## Implementation scope

Work in the source-of-truth `bob-plugins` repository opened through `sase repo`. The
planning checkout may be behind its fetched `origin/master`; implementation must begin
from the latest repository state containing `block-id-prompt` 1.9.0 and commit
`634a931`.

Expected files:

- `plugins/block-id-prompt/main.js`: Work Log entry constants/formatter, explicit local
  date threading, exact preview rendering, and the unchanged guarded pause
  orchestration.
- `plugins/block-id-prompt/styles.css`: narrowly scoped date/separator/summary preview
  styling using Obsidian theme variables.
- `scripts/test-block-id-prompt.cjs`: formatter, planner, preview-state/runtime, local
  date, and regression coverage.
- `plugins/block-id-prompt/manifest.json`: minor version bump from `1.9.0` to `1.10.0`.
- `README.md`: document that nonblank pause summaries are recorded as locally dated Work
  Log entries.

Do not change Daily Notes resolution, turn dates into wikilinks, add a time of day,
offer a date picker, rewrite historical entries, create logs for blank summaries or Next
tasks, or alter the command's status/Pomodoro semantics.

## Test plan

Extend the focused Node harness with coverage for:

1. Pure formatting: zero-padded local ISO dates, exact italic/em-dash output, preserved
   normalized Markdown in the summary, and deterministic results with an injected date.
2. Work Log creation: a first entry creates the canonical marker and dated entry with
   correct indentation and placement after existing task children.
3. Existing logs: dated entries prepend newest-first beneath canonical, emoji-less, and
   legacy markers while preserving list markers, indentation, old undated entries, and
   hand-written content byte-for-byte.
4. Structural edge cases: nested-marker rejection, first-of-multiple-marker behavior,
   LF/CRLF preservation, repeated same-day entries, and no unrelated edits.
5. Blank summaries: remain a structural no-op with no date prefix, marker, or separator,
   while the enclosing pause status/cleanup operation still succeeds.
6. Prompt state/presentation: blank messaging is unchanged; nonblank preview contains
   the exact captured date, em dash, and normalized summary; the `::` warning and button
   label still respond correctly; user text is rendered as text rather than HTML.
7. Runtime threading: direct/cross-note and same-note pause paths use one captured local
   date, keep status plus Work Log atomic on the task side, and preserve cleanup order,
   cursor, notices, and guarded failure behavior.
8. Date boundaries: the injected plugin clock proves local calendar components are used
   even when the corresponding UTC day differs; the preview date and written date cannot
   diverge within one prompt interaction.
9. Regressions: all Ready/Blocked forward, Next reverse, blank In Progress pause,
   Schedule Log activation, block-ID, Pomodoro cleanup, and modal cancellation tests
   continue to pass.

## Verification and deployment

From the latest opened `bob-plugins` checkout:

1. Run `node --test scripts/test-block-id-prompt.cjs` while iterating.
2. Run the full `npm test` suite.
3. Run `npm run validate` for all plugin manifests.
4. Preview deployment with
   `bob plugins sync --dry-run --no-pull --repo . --plugin block-id-prompt`.
5. Deploy with `bob plugins sync --no-pull --repo . --plugin block-id-prompt`, as
   required by that repository's instructions.
6. Re-run the dry-run and `bob plugins list --no-pull --repo . --format json`; verify
   version `1.10.0`, enabled and synced state, and zero pending copies.

The implementation is complete when every newly generated nonblank Work Log entry uses
the canonical local date prefix, the modal faithfully previews it, blank and historical
entries remain untouched, all guarded task/Pomodoro behavior is preserved, focused/full
tests and manifest validation pass, documentation matches the shipped format, and the
source-controlled plugin is synced to the vault.
