---
tier: tale
title: Swap current and future Pomodoros while preserving the current time range
goal: Ctrl+Shift+J/K can exchange current and future Pomodoros while keeping the exact
  time range in the current slot and preserving past history.
size: medium
proposed_by: bbugyi200.athena.0h0
status: done
---

# Swap the current Pomodoro with future Pomodoros using Ctrl+Shift+J/K

## Outcome

Allow `Ctrl+Shift+J` on the current Pomodoro to move its entry and complete subtree
down, promoting the next future Pomodoro into the current slot. Allow `Ctrl+Shift+K` on
the adjacent future Pomodoro to perform the same swap from below. Preserve the current
slot's exact parenthetical time range, including bold markup, spacing, and duration
metadata, and associate it with the newly current Pomodoro. The displaced entry becomes
an open placeholder. Past and cancelled Pomodoros must remain immovable and cannot be
crossed.

This is a `tale` with a `medium` implementation size: one coding agent can extend the
existing pure planner and shared editor route, add focused coverage, and deploy one
plugin. No separate implementation phases are needed.

## Repository and established behavior

The implementation belongs in the linked `bob-plugins` source repository. Before reading
or modifying it, use `/sase_repo` and run:

```sh
sase repo open bob-plugins -r "Implement current Pomodoro swaps for Ctrl+Shift+J/K"
```

Use only the checkout path returned by that command, and read its `AGENTS.md`. All
plugin paths below are relative to that returned repository. Do not edit installed
plugin files directly. The plugin is plain CommonJS; `main.js` is source, with no build
step.

Relevant files and symbols:

- `plugins/bob-navigation-hotkeys/main.js`: `parsePomodoroEntryLine`,
  `collectPomodoroEntries`, `findPomodoroEntryContext`, `isMovablePomodoroEntryContext`,
  `planPomodoroEntryReorder`, `movePlannedPomodoroEntry`, and `jumpToOpenObsidianTask`.
- `scripts/test-navigation-hotkeys.cjs`: `pomodoroFixtureLines`,
  `countedPomodoroReorderLines`, planner coverage, `TransactionEditor`, counted command
  routing, physical-key routing, and duplicate-dispatch coverage.
- `plugins/bob-navigation-hotkeys/manifest.json` and the Bob Navigation Hotkeys row in
  `README.md` (currently version `1.33.0`).

The planner currently requires the source and every crossed entry to be open
placeholders. The dispatcher independently applies the same restriction, so a current
timed entry currently falls through to task navigation. Merely relaxing these guards
would incorrectly carry the time range with the old current entry.

The parser already provides exact `rangeStart`, `rangeEnd`, and `rangeText` spans. It
recognizes compact and colon-separated ranges, optional bold markup, metadata, and named
or unnamed entries. Reuse this grammar. Its existing open classification excludes `x`,
`X`, and `-`; preserve that classification rather than creating a new checkbox or
clock-based definition of current/past.

The current planner rotates entire entry blocks for an exact Vim count, keeps
inter-entry gaps in their slots, preserves line endings and surrounding text, and
reports the moved source's resulting line. The editor route applies one guarded
transaction, follows the source with the cursor, and deduplicates the Obsidian command
and Vim fallback. Preserve these properties.

Planning verification: 35 targeted existing planner, dispatcher, and key-routing tests
passed, with 327 unrelated tests skipped and no failures. These include old
current-entry rejection/navigation expectations that must change below.

## User-visible contract

For example, either `J` on ALPHA or `K` on BETA, with Ctrl+Shift held, changes:

```markdown
## Pomodoros

- [x] (**0850-0920** [t:: 30m]) — FINISHED
  - completed work
- [ ] (**0920-0950** [t:: 30m]) — ALPHA
  - [[Tasks#^alpha]]
- [ ] () — BETA
  - [[Tasks#^beta]]
```

into:

```markdown
## Pomodoros

- [x] (**0850-0920** [t:: 30m]) — FINISHED
  - completed work
- [ ] (**0920-0950** [t:: 30m]) — BETA
  - [[Tasks#^beta]]
- [ ] () — ALPHA
  - [[Tasks#^alpha]]
```

Names, checkbox prefixes, suffix text, links, and all owned descendants travel with
their entry. Only the two parenthetical range fields exchange ownership. Do not
close/start a session, recalculate times, move task statuses, merge children, or edit
linked tasks as part of the swap.

Counts keep their existing meaning: `N<Ctrl+Shift+J/K>` moves the selected entry exactly
N positions by rotation, atomically, with no clamping or wrapping. For
`[A(time), B(), C()]`, moving A down two yields `[B(time), C(), A()]`; moving C up two
from the original yields `[C(time), A(), B()]`. The time belongs to the original current
ordinal slot, not a fixed character offset or a task identity. The cursor follows the
source: it follows A into the future for the first operation and C into current for the
second, with its column clamped to the rewritten header length.

An attempted current-up move is handled with a notice and no mutation or task jump.
Moving beyond the last available future entry, above the current slot, or across any
closed/cancelled entry refuses the entire request. A current entry without a following
future entry cannot move. A note with only planned entries retains its existing reorder
behavior. On past/cancelled entries, sub-bullets, ordinary tasks, and other contexts,
retain the existing navigation route and never reorder past/cancelled entries.

## Implementation

1. Extend the reorder eligibility predicate to recognize parsed open timed entries as
   well as open placeholders. Keep this predicate about routing; enforce direction and
   span legality in `planPomodoroEntryReorder`. Ensure invalid requests on a current
   entry are consumed by the reorder route.

2. Validate the whole requested span before rendering any edits. Preserve stale-source
   validation and repeat normalization. A legal span is either entirely open
   placeholders, or exactly one open timed entry at the earliest span position followed
   solely by open placeholders. In the timed case, that entry must be the source moving
   down or the destination of a future source moving up. Reject a timed entry in the
   interior, multiple timed entries in the span, future-down moves into a timed entry,
   and current-up moves, including irregular input with a placeholder above current. Do
   not skip closed/cancelled entries or cross section boundaries. This is a local span
   rule; unrelated entries outside the span need no new global validation.

3. Preserve the existing block rotation and gap assembly. When the span includes
   current, determine which rotated block will occupy its first slot. Exchange that
   incoming entry's exact placeholder range text with the old current entry's exact
   time-range text using each original header's parsed range offsets and string slices.
   Perform this on copied block/header data, without mutating frozen parser results. For
   current-down counted moves the incoming entry is the first successor, not the final
   destination; for future-up it is the source. Leave intermediate placeholder headers
   unchanged. Do not reconstruct entire lines or normalize names, statuses, spacing, or
   metadata. Preserve placeholder whitespace such as `(  )` by exchanging the original
   strings, so inverse moves can restore the original bytes.

4. Keep the existing planner return fields and calculate `movedEntryLine` from the
   rendered blocks as today. Route the new cases through the same guarded
   `applyEditorContentTransaction` call and shared duplicate-dispatch guard. Preserve
   count consumption for both physical-key and command routes. Rename
   `movePlannedPomodoroEntry` to an accurate name if useful, updating its callers.
   Update stale comments, command display names, and refusal notices to cover
   current/future reordering; keep command IDs and key bindings stable. Refusal notices
   must describe the current/history boundary accurately.

5. Extend existing tests and replace obsolete current-entry expectations. Retain
   equivalent coverage for contexts that still navigate or refuse. Update the README
   behavior description and plugin version together, using the next available minor
   version (expected `1.34.0`; recheck the checkout).

## Regression coverage and acceptance criteria

Use explicit expected Markdown for the core cases rather than deriving all expectations
from the planner under test.

- Both uncounted entry points produce the example above, leave exactly one timed open
  entry in the affected span, and preserve closed history exactly. Cover named/unnamed
  entries, unequal subtree sizes, nested children, and childless entries. Include an
  existing alternate open checkbox such as `[/]` and assert prefixes travel with the
  entries without status conversion.
- Assert exact transfer of compact/colon ranges, bold/unbold forms, duration metadata
  (including embedded em dashes), spacing, and whitespace placeholders. Swap back to
  restore the original content exactly.
- Cover both counted examples above and their inverses, including intermediate
  placeholders with distinct range whitespace. Retain all planned-only cases.
- Refuse current-up, lone current, moving beyond current, insufficient future entries,
  closed/cancelled source or crossed entry (`x`, `X`, `-`), an interior timed entry,
  multiple timed entries in a requested span, and stale source headers. Failures return
  the original bytes and no moved-entry position. Include a counted request that could
  swap once but must fail as a whole.
- Preserve LF/CRLF, trailing-newline presence/absence, unequal separator gaps, all
  outside text, fences, and existing first-Pomodoros-section scoping.
- Through `jumpToOpenObsidianTask`, cover current-down and future-up, cursor
  location/column clamping, exactly one transaction/undo group, current-boundary refusal
  without a jump, and unchanged non-reorder navigation. Exercise a counted promotion via
  command-route pending Vim count and the physical-key route, including both dispatch
  orderings so one press cannot swap twice. Preserve stale-content/failed-transaction
  safeguards for the new route.

From the linked repository, run focused coverage during development:

```sh
node --test --test-name-pattern='PomodoroEntryReorder|isMovablePomodoroEntryContext|jumpToOpenObsidianTask|dispatch guard|physical Ctrl' scripts/test-navigation-hotkeys.cjs
```

Then run the required final checks:

```sh
npm test
npm run validate
git diff --check
```

The full plugin suite guards shared parser and other-plugin behavior. No Rust or
capture-contract change is expected in `bob-cli` or `bob-mac-capture`.

## Deployment and completion

After the implementation and checks pass, follow the linked repository's mandatory
`bob plugins sync` deployment instruction. From the primary checkout, set a
task-specific shell variable `pomodoro_plugins_repo` to the path returned by
`sase repo open`, then preview and deploy only the changed plugin:

```sh
bob plugins sync --no-pull --repo "$pomodoro_plugins_repo" --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo "$pomodoro_plugins_repo" --plugin bob-navigation-hotkeys
```

Inspect output and verify the Bob Navigation Hotkeys row reports synced using
`bob plugins list --no-pull --repo "$pomodoro_plugins_repo" --format json`. The list
command does not accept `--plugin`. The explicit repo argument ensures the edited
checkout is deployed; the default can resolve to a different clone. A zero exit code can
still accompany skipped dirty vault files, so do not claim deployment succeeded solely
from the exit status.

If live Obsidian interaction is available, smoke-test both hotkeys, a counted promotion,
undo, and a history-boundary refusal on a disposable note after reloading the plugin.
Otherwise report live interaction as unverified while distinguishing it from passing
automated tests and verified sync. Report the changed repository, test results, and
deployment outcome through the required SASE final declaration workflow.
