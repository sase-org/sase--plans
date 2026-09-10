---
tier: tale
title: Add an explicit create-future-Pomodoro completion action
goal:
  The Mac capture panel makes Bob's named future Pomodoro creation path a clear, safe
  completion choice without changing existing selection or naming flows.
size: medium
proposed_by: bbugyi200.athena.0fw.f0
create_time: 2026-09-09 20:00:12
status: wip
---

# Add an explicit create-future-Pomodoro completion action

## Goal

Make the native Mac capture panel expose the new create-on-miss behavior for an
explicitly named Pomodoro-linked capture. A user typing `@<route>:<block-id>#<pomodoro>`
should be able to choose a clear **Create future Pomodoro** completion action when Bob
would create a new named placeholder, while retaining the existing actions that select a
named open Pomodoro or assign a name to a specific unnamed open Pomodoro.

## Scope and contract decisions

- The Pomodoro-linked spelling is `@<route>:<block-id>#<pomodoro>`. The `+` spelling is
  the distinct `@<route>+<block-id>#<task-section>` family and must remain unchanged in
  both repositories.
- Raw editor text, live dry-run preview, and final capture already pass through to
  `bob capture`, so they inherit create-on-miss without an app-side vault write. The
  missing behavior is the completion interaction: a nameable existing Pomodoro can
  currently keep the list visible and make Return/Tab open **Name Pomodoro**, obscuring
  the new create-on-submit path.
- Keep `bob` authoritative for Pomodoro grammar, canonical names, open-name
  whole-slug/prefix selection, ambiguous-current detection, and candidate ordering. Do
  not reproduce those decisions from Swift query parsing.
- Extend schema-version-1 `pomodoro_name` completion candidates additively with a
  boolean `creates_pomodoro` field. It defaults to false/absent for every existing
  candidate. A creation candidate carries the canonical selector in `replacement`, the
  canonical visible name in `name`, `creates_pomodoro: true`, and no Pomodoro `ref`;
  accepting completion does not mutate the daily note.
- Emit the creation candidate only for a nonempty valid name for which execution would
  find no matching named open Pomodoro and could compute an unambiguous creation anchor.
  Do not emit it when an exact/prefix open-name match would win, the daily note or
  `## Pomodoros` section is missing, or multiple open timed Pomodoros make creation
  ambiguous. Preserve the existing warnings and live-preview error reporting in those
  cases.
- When creation is eligible, put the creation action before substring-only named
  suggestions and before `requires_name` rows. This makes the default action agree with
  what submitting the authored selector would do. Existing exact/prefix named matches
  remain the default and do not receive a competing creation action. Empty-query
  completion remains the existing discovery list with no speculative create row.
- Keep the existing **Name Pomodoro** flow. Selecting a `requires_name` row still calls
  `bob capture-pomodoro-name` immediately for that specific existing entry; selecting
  the new creation row only preserves/canonicalizes the marker text and lets the later
  `bob capture` transaction create the placeholder and task link together.
- Maintain compatibility in both directions: older app builds can safely treat the
  creation row as an ordinary replacement, while the updated app decodes an absent
  `creates_pomodoro` as false when connected to an older Bob build.

## Implementation plan

1. Extend Pomodoro-name completion in `bob-cli` with a server-authored creation action.
   - Reuse the shared Pomodoro name canonicalizer and existing named selector resolution
     from `capture_pomodoros`; do not add a second name grammar or matching algorithm.
   - Make the completion candidate's stale Pomodoro ref optional so a non-mutating
     creation action cannot masquerade as an existing ledger entry, and serialize the
     additive `creates_pomodoro` discriminator.
   - Build the creation candidate only after scanning the actual daily ledger and
     checking the execution-relevant section and unique-current preconditions. Give it a
     canonical visible name and selector replacement, but leave all placement and
     mutation to `bob capture`.
   - Preserve named/open discovery, duplicate collapsing, nameable-row refs, warnings,
     human output, and schema version. Document the new row and its non-mutating
     acceptance contract in `bob capture-complete --help`.

2. Teach `bob-mac-capture` to decode and present the new action distinctly.
   - Add `createsPomodoro` to `CaptureCompletionCandidate`, decoding a missing field as
     false just like the other additive flags.
   - In Pomodoro completion row content, render creation as a distinct timer/plus action
     with the canonical name, **Create** / **New future Pomodoro** affordance, and an
     accessibility label and hint that explain creation happens on capture. Keep named
     and `requiresName` presentations unchanged.
   - In `CapturePanelModel.acceptSelectedCompletion`, handle creation before the
     `requiresName` branch: apply Bob's replacement range/cursor rules, dismiss the
     completion list, keep both inline prompts closed, rerun parse/live preview without
     immediately reopening completion, and announce that the named future Pomodoro will
     be created when the draft is captured. Never call `capture-pomodoro-name` for this
     action.
   - Continue to let keyboard and pointer acceptance share the same model path. The next
     ordinary Return submits the draft through the existing aggregate transaction;
     Escape and later edits retain their current behavior.

3. Add focused contract and interaction coverage in both repositories.
   - In `bob-cli`, cover novel, completed-only, and substring-only queries producing a
     first creation row; exact and prefix open matches suppressing it; empty or invalid
     queries; missing ledger structure; and multiple timed-open ambiguity. Assert JSON
     fields/order, optional ref behavior, existing nameable rows, human badges, and
     unchanged schema version.
   - In `bob-mac-capture` model decoding and row-content tests, cover absent/false and
     true `creates_pomodoro`, the new visual/accessibility contract, and stable
     rendering of existing named/nameable rows.
   - In panel-model tests, prove accepting creation preserves the canonical marker,
     closes completion without opening **Name Pomodoro**, restores the editor caret and
     focus, requests a fresh live preview, and never launches `capture-pomodoro-name`.
     Keep regression coverage showing a user can navigate to a nameable row and use the
     existing immediate naming flow.
   - Exercise a later item in a blank-line-separated draft so the server-provided global
     UTF-8 replacement range remains correct, including non-ASCII text before the
     marker.

4. Update the Mac app's public contract.
   - In `README.md`, describe named open selection versus future creation, the explicit
     create row, and the continued **Name Pomodoro** alternative.
   - Correct the syntax description to emphasize colon-Pomodoro versus plus-task-section
     behavior, and note that creation remains transactional at final capture rather than
     an eager ledger mutation.
   - Update the minimum/recommended Bob capability text so users know an older binary
     may omit the explicit create action even though additive decoding remains safe.

## Acceptance criteria

- With a valid name that has no open whole-slug or prefix match, completion selects a
  clearly labeled creation row first even when unnamed open Pomodoros also exist.
- Accepting that row does not change the daily file or invoke `capture-pomodoro-name`;
  the eventual `bob capture` dry-run/final call receives the canonical `@route:id#name`
  marker and owns the atomic creation/link transaction.
- A matching named open Pomodoro remains the destination without offering a misleading
  create action, while a completed-only name can offer creation of a new future entry.
- Existing unnamed-row naming, named-row selection, task-section completion for the plus
  family, batch/global byte ranges, keyboard behavior, and older-response decoding
  remain intact.
- Impossible creation states are not presented as actionable completions and remain
  write-free through Bob's existing preview/final validation.

## Validation

Run focused completion tests in `bob-cli`, focused Swift model/process tests in
`bob-mac-capture`, then each repository's full checks:

```bash
cargo test native::capture_complete
cargo test --test cli capture_complete_pomodoro_name
just all

./Scripts/xcode-swift.sh test --filter CaptureModelTests
./Scripts/xcode-swift.sh test --filter CompletionRowContentTests
./Scripts/xcode-swift.sh test --filter CapturePanelModelTests
just all
```
