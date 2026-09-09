---
tier: tale
title: Activate future-scheduled tasks when Pomodoro task links are added with `^^`
goal:
  A sole-content Pomodoro `^^` task link safely removes a selected task's future
  schedule, records the move in an existing Schedule Log without creating one, and makes
  a Blocked task Next while preserving guarded link completion.
size: medium
proposed_by: bbugyi200.athena.023
create_time: 2026-09-09 19:53:28
status: wip
---

# Plan: activate future-scheduled tasks when `^^` links them to a Pomodoro

## Goal

When the Block ID Prompt plugin completes a `^^` task-picker link that is the sole
content of a Pomodoro sub-bullet, selecting a task with a strictly future `scheduled`
date makes that task actionable now:

- remove the task's future `scheduled` field;
- change `[?]` Blocked (and the existing `[ ]` Ready case) to `[*]` Next without
  downgrading an already-Next or In-Progress task;
- if the task already owns a Schedule Log marker, prepend a deterministic entry for the
  move from the old scheduled date to today; and
- never create a Schedule Log marker for a task that does not already keep one.

The link completion, optional block-ID creation, existing duplicate-link cleanup, and
concurrency guards must continue to work for both cross-note and same-note targets.

## Repository and ownership

All implementation work belongs in the linked `bob-plugins` repository. Before any read
or write, open it with:

```bash
sase repo open bob-plugins -r "Implement future-schedule activation for ^^ Pomodoro task links"
```

Use only the path printed by that command. Never edit the deployed copies under the Bob
vault: `bob plugins sync` overwrites them.

Files in scope:

- `plugins/block-id-prompt/main.js` — behavior and pure edit-planning helpers.
- `scripts/test-block-id-prompt.cjs` — helper and plugin-flow regressions.
- `plugins/block-id-prompt/manifest.json` — feature release version.
- `README.md` — matching version and user-visible behavior summary.

No `bob-cli` Rust or documentation change is needed. The `^^` picker, its task write,
and its notices are owned by `block-id-prompt`; `bob-cli` only provides adjacent
capture-completion and plugin-deployment commands.

## Current behavior to preserve

`plugins/block-id-prompt/main.js` already has the relevant control flow:

1. `findTaskPickerMarkerNearCursor` recognizes the `^^` forms and opens
   `TaskLinkPickerModal`.
2. `collectTaskPickerItems` includes `[ ]`, `[/]`, `[*]`, and `[?]` `#task` lines, skips
   terminal/hidden/non-task lines, and records the selected line and existing block ID.
3. `shouldPromoteTaskToNext` returns true only when a `[ ]` target is chosen from a link
   that is the sole content of a Pomodoro descendant bullet. Although
   `findPomodoroSourceContext` already reports `isOpen`, this predicate currently does
   not consult it.
4. `completeTaskLinkWithExistingId` and `submitLinkTaskBlockId` separately handle tasks
   that already have a block ID and tasks that need one.
5. `applyTaskLineEdit` preimage-checks the task note before changing it;
   `completeTaskSourceLink` then canonicalizes the source link and prunes duplicate
   copies from later open Pomodoros.

The current explicit regression that Blocked tasks are never promoted is too broad for
the requested feature. Narrow it so an unscheduled (or non-future-scheduled) `[?]` task
remains Blocked, while a uniquely and validly future-scheduled `[?]` task follows the
activation path below.

## Behavioral contract

### Trigger and status rules

Retain the existing sole-content and ledger-ancestry checks, and require the owner
reported by `findPomodoroSourceContext` to be open. The `^^` marker must become a task
block link that is the sole body of an indented list item owned by an open entry in the
`## Pomodoros` section. Use this qualification for both the existing Ready promotion and
the new future-schedule path, so adding a link to completed/canceled Pomodoro history
never changes the target's current state. Prose containing a link, ordinary tasks with a
nested link, top-level bullets, links outside the section, and fenced examples must not
activate a target.

For a qualified source, inspect the selected target's live line from the guarded
destination snapshot:

| Target state                                                                      | Result                                                                           |
| --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Exactly one valid `scheduled` field later than local today; status `[?]` or `[ ]` | Remove the field and set `[*]`.                                                  |
| Exactly one valid future field; status already `[*]` or `[/]`                     | Remove the field and preserve the stronger/current status.                       |
| No future field; status `[ ]`                                                     | Preserve the existing promotion to `[*]`.                                        |
| No future field; status `[?]`, `[*]`, or `[/]`                                    | Preserve the current status.                                                     |
| Invalid or ambiguous scheduled metadata                                           | Do not guess or delete anything; preserve the existing link-completion behavior. |

Recognize the same task-level forms as `bob task-status-hooks`:
`[scheduled:: YYYY-MM-DD]` and `(scheduled:: YYYY-MM-DD)`, with optional whitespace
after `::`. Require an exact four-digit year/two-digit month/two-digit day and validate
the actual calendar date. Compare date components against the runtime's local calendar
day; do not use `toISOString()`, which can select the wrong day near a timezone
boundary. Make the date/clock input injectable into the pure planner for deterministic
tests.

Treat more than one recognized `scheduled` field as ambiguous even if only one is
future. A malformed field is not proof that the task was date-blocked. These cases still
complete the link as they do today and retain their scheduled metadata; the pre-existing
`[ ]`-to-`[*]` promotion may still apply, but no `[?]` status is changed on the strength
of ambiguous data.

This gesture intentionally changes a future-scheduled `[?]` task to `[*]` as requested;
it does not reinterpret `dependsOn` metadata. If another blocking reason remains,
`bob task-status-hooks` may derive Blocked again on its next run.

### Scheduled-field removal

Remove only the one validated future `scheduled` field, including its adjacent spacing
without disturbing other Dataview fields, tags, prose, quote/list prefixes, or the
trailing `^block-id`. Collapse only the whitespace exposed by that removal so tokens
remain separated by one space and trailing whitespace is not introduced. For a task that
needs a new block ID, append it after every remaining field so the ID stays the final
task token.

Today and earlier dates remain present. Linking a mature scheduled task is not an
unschedule gesture, and the existing `[ ]`-to-`[*]` promotion continues independently.

### Existing-only Schedule Log update

Mirror the Schedule Log compatibility rules already implemented in
`plugins/bob-navigation-hotkeys/main.js`, but keep the Block ID Prompt plugin
self-contained (plugins are deployed independently and do not import each other's
`main.js`). Recognize a direct child marker in all currently supported forms:

- `- 🗓️ **SCHEDULE LOG**`
- `- **SCHEDULE LOG**`
- legacy `- **Schedule log:**`

Ignore a marker belonging to a nested grandchild. If a direct marker exists, prepend one
newest-first entry under it, reusing its list marker and an existing entry's indentation
when available (otherwise one tab deeper than the marker). The exact entry is:

```markdown
- _<old-future-date> → <local-today>_ — 🍅 pulled into today's Pomodoro
```

For example, on 2026-08-15:

```markdown
- [?] #task Ship it [scheduled:: 2026-08-20] ^ship
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-01 → 2026-08-20_ — waiting on review
```

becomes:

```markdown
- [*] #task Ship it ^ship
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-20 → 2026-08-15_ — 🍅 pulled into today's Pomodoro
    - _2026-08-01 → 2026-08-20_ — waiting on review
```

If no recognized direct marker exists, remove the date and update the status but add no
marker and no entry. A marker with no existing entries still counts as an existing log
and receives the new entry. Preserve LF versus CRLF and whether the file ends in a
newline.

### Transaction and stale-data behavior

Replace the status-only/single-span edit construction with one pure target-note planner
that accepts the preimage, selected task line, optional new block ID, source context,
and clock. It should return the complete postimage plus explicit outcome metadata, for
example:

- whether a future schedule was removed;
- whether the status changed;
- whether a Schedule Log entry was added;
- whether a block ID was appended; and
- how many lines were inserted before the source marker when source and target are the
  same note.

Compose status replacement, scheduled-field removal, optional block-ID append, and log
insertion before performing any write. Apply the target postimage only after re-reading
and matching the expected preimage, preserving the existing "selected task changed" and
"note changed before update" guards. One target-note write must contain all target
mutations; never leave only the status or only the log changed.

The source link remains the second step, matching current failure policy. If it changed
after the target write, report the partial outcome accurately (for example, task
unscheduled/set Next/logged, but link completion stopped) rather than claiming only
"Task set Next".

Same-note completion needs special treatment because prepending a log entry can shift
the Pomodoro marker down by one line. Use the planner's line-delta metadata to update a
copy of the source coordinates before `sourceMarkerStillPresent` and
`completeTaskSourceLink` run. Cover both orderings (target/log before the Pomodoro
source and after it), and keep later-Pomodoro duplicate cleanup plus final cursor
placement correct. Do not relocate the source by fuzzy text search.

### Notices

Extend both existing-ID and new-ID success/partial-failure notices from structured
planner outcomes. Keep the current concise phrases for ordinary links and add chips only
when they happened:

- `removed future schedule`
- `set Next`
- `logged schedule change`
- the existing `removed N future link(s)` cleanup suffix

A task with no existing Schedule Log must not claim it logged anything. An already
Next/In-Progress task whose date is removed must not claim `set Next`.

## Implementation steps

1. Add small date and scheduled-field helpers near the existing task-line parsing
   helpers. Return spans and validated values so removal is positional and does not rely
   on broad string replacement. Expose only helpers useful to focused tests.
2. Add the minimal Schedule Log constants, direct-child discovery, indentation, and
   entry-format helpers beside the Pomodoro helpers. Comment that their marker/entry
   grammar mirrors `bob-navigation-hotkeys/main.js` and must remain compatible.
3. Introduce the pure target-note mutation planner described above. Preserve all current
   no-future outcomes while combining future unscheduling, status choice, optional ID
   creation, and existing-only log insertion into one postimage.
4. Replace the existing `shouldPromoteTaskToNext`/`setCheckboxStatusEdit` branches in
   `completeTaskLinkWithExistingId` and `submitLinkTaskBlockId` with the planner. Keep
   duplicate-ID validation and selected-line identity checks ahead of the write.
5. Generalize `applyTaskLineEdit` into a guarded content-plan writer. Thread the outcome
   and same-file source-line delta through source-link completion and notices.
6. Export the new pure helpers needed by `scripts/test-block-id-prompt.cjs`; do not
   expose plugin runtime internals unnecessarily.

## Tests

Add focused `node:test` cases in `scripts/test-block-id-prompt.cjs` for:

1. Calendar parsing and strict-future boundaries (yesterday/today/tomorrow, leap day,
   impossible date, malformed width) using an injected local date.
2. Both bracketed and parenthesized scheduled fields, arbitrary field order, quoted and
   numbered task lines, and preservation of unrelated fields/tags/trailing IDs.
3. A future `[?]` task becoming `[*]`, a future `[ ]` task becoming `[*]`, and future
   `[*]`/`[/]` tasks retaining status while losing the date.
4. Existing behavior for unscheduled `[ ]` promotion and unscheduled `[?]`
   non-promotion; today/past schedules remain untouched.
5. Invalid and duplicate scheduled fields remaining byte-identical and never causing a
   log entry.
6. Exact newest-first log output for canonical, emoji-less, and legacy markers;
   marker-with-no-entry fallback indentation; nested-marker rejection; and the rule that
   no marker is ever created.
7. LF, CRLF, final-newline, mixed spaces/tabs, and schedule-field whitespace
   preservation.
8. New-ID planning combines the ID, unschedule, status, and log changes while keeping
   the ID final; existing-ID planning never duplicates it.
9. Runtime existing-ID and new-ID flows against a separate task note, asserting the
   guarded postimage, canonical source link, outcome notice, and unchanged source when
   the target preimage goes stale.
10. Same-note flows where log insertion occurs before and after the source marker,
    asserting corrected line coordinates, cursor placement, and duplicate cleanup from
    later open Pomodoros.
11. A source marker changed after a successful target write, asserting the accurate
    partial-success notice and no unrelated edits.
12. Sole-content/Pomodoro qualification regressions so prose links, ordinary task
    children, top-level bullets, completed/canceled Pomodoro contexts, and fenced
    examples do not gain activation behavior; assert that open placeholder and timed
    Pomodoro owners still qualify.

Run the focused test first, then the complete plugin suite and manifest validation from
the opened `bob-plugins` root:

```bash
node --test scripts/test-block-id-prompt.cjs
npm test
npm run validate
```

## Release, documentation, and deployment

1. Bump `plugins/block-id-prompt/manifest.json` from `1.5.0` to `1.6.0` because this is
   a user-visible feature.
2. Update the Block ID Prompt row in `README.md` to `1.6.0` and state that a sole
   Pomodoro `^^` link pulls a future-scheduled task into today, updates an existing
   Schedule Log, and does not create one.
3. After tests and validation pass, preview and deploy only this plugin from the opened
   linked-repository root, as required by that repository's `AGENTS.md`:

   ```bash
   bob plugins sync -p block-id-prompt -r "$PWD" --dry-run
   bob plugins sync -p block-id-prompt -r "$PWD"
   ```

4. Report that Obsidian must reload the Block ID Prompt plugin (or restart) before the
   new runtime code is active.

## Out of scope

- Changing `bob task-status-hooks`, `bob capture`, the navigation-hotkeys property
  picker, or the Schedule Log schema globally.
- Creating a Schedule Log for tasks that opted out by not having a marker.
- Removing today/past schedules or repairing malformed/duplicate schedule metadata.
- Persisting a pre-Blocked status or suppressing a later derived Blocked result from
  remaining dependencies.
- Making cross-note target mutation and source-link insertion atomic across two vault
  files; retain the existing guarded ordering and improve only the accuracy of its
  partial-success notice.
