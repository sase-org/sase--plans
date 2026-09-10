---
tier: tale
title: Finish and land bob-cli-u after fixing route-cache and focus regressions
goal:
  Bob Mac Capture preserves cached route completion without an unnecessary Bob
  subprocess, restores the correct keyboard focus after every Add block ID outcome, and
  lands bob-cli-u only after cross-repository verification and post-close cleanup are
  complete.
size: medium
proposed_by: bbugyi200.athena.bob-cli-u.land--1
bead: bob-cli-u
create_time: 2026-09-09 19:59:45
status: wip
---

- **BEAD:** bob-cli-u

# Finish and Land `bob-cli-u`

## Context

Epic `bob-cli-u` adds opt-in discovery of open `@route+` tasks without block IDs, safe
assignment through `bob capture-task-id`, and the macOS **Add block ID** flow. Its two
implementation beads are closed, but the land audit found two issues in the macOS
completion state machine that must be resolved before the epic can close.

The audit verified the phase commits and the source they changed:

- `bob-cli` commit `2037307` implements `capture-complete --all-tasks` and
  `capture-task-id`. The completion path remains opt-in and plus-context-only, returns
  identified tasks before unidentified tasks, and carries nullable ID, route, stale-safe
  ref, and `requires_block_id` metadata. The mutation command shares Bob's
  route/block-ID/task parsing, rejects duplicate or stale state before writing,
  preserves LF/CRLF and unrelated bytes, and uses the existing same-directory atomic
  replacement helper. Its unit and CLI tests cover dry-run, shifted refs, terminal and
  already-identified tasks, ambiguous/stale refs, missing notes, and write-free errors.
- Linked `bob-mac-capture` commit `dff08a7` decodes the additive contract, always asks
  Bob for all tasks on process-backed completion, renders the two task groups, owns the
  prompt state, delegates the write to `capture-task-id`, and updates the draft only
  after Bob confirms success. Its fake-Bob, model, process-client, row, keyboard, and
  documentation coverage is present.
- No child bead contains a `PROPOSED FOLLOW-UP:` entry. The child note about
  `CapturePanelModelTests.testTaskBlockIDRouteSpanUsesCachedRouteCompletion` is not a
  separate backlog item: it exercises the completion state owned by this epic and must
  be fixed here.

The integration audit also covered every commit after the epic was created. In
`bob-cli`, batch-capture commit `a8c9ad8` is an ancestor of the phase-1 commit,
including the overlapping capture-completion and CLI-test changes. In `bob-mac-capture`,
actor isolation (`b55d485`), batch result integration (`4c22525`), and notification
polish (`c95ba0e`) are ancestors of the phase-2 commit. The only later commit,
stash-picker delete-all (`3d65b05`), touches the shared model/view/key-routing files but
keeps the task-ID prompt routing branch ahead of ordinary and stash commands; its model
and view changes do not conflict with the prompt state.

Current revalidation is green for `bob-cli` (`just all`, `just check-scripts`, and
`just package-list`) and for the portable linked-repo checks (`git diff --check`,
`bash -n Tests/Fixtures/fake-bob`, and `swift build --target CaptureCore`). Full app
build/tests still require an Xcode-capable macOS host. Do not treat the portable core
build as validation of the AppKit/SwiftUI fixes below.

## Root causes and required behavior

### Cached route completion

`CapturePanelModel.scheduleAnalysis` currently publishes the result of
`cachedRouteCompletion(...)` and then unconditionally calls
`BobProcessClient.captureComplete(...)`. That violates the established cache contract,
makes `testTaskBlockIDRouteSpanUsesCachedRouteCompletion` race/fail, and can overwrite a
valid cached route response with a later process response.

When the cursor is within a route span and the target cache can answer it, keep the
normal parse and live-preview work but make the cached response authoritative for that
completion request and skip `capture-complete`. When the cache cannot answer, preserve
the process fallback. Section, task (`@route+query` after the plus), Pomodoro,
interactive-placeholder, and wikilink completion must remain process-backed. In
particular, task completion must still pass `--all-tasks` so missing-ID candidates are
available.

### Deterministic prompt/editor focus

`TaskIDPromptCard` focuses its block-ID `TextField` on appearance, but removing that
view after success or cancellation does not explicitly return focus to the capture
editor. A server/transport failure also re-enables a field that was disabled while
saving without explicitly returning focus to it. The model restores the draft and caret
value but has no focus contract, so the next keystroke can go to an arbitrary control
even though keyboard-complete behavior is part of the epic.

Give the panel a single explicit focus state (or an equivalently deterministic request
mechanism) covering the capture editor and block-ID field. Opening the prompt focuses
the block-ID field; a failed assignment re-enables and refocuses that field while
retaining the authored ID and error; successful assignment focuses the editor after the
canonical ID and caret are installed; cancel/Escape returns to the chooser with the
prior selection and focuses the editor. Late canceled responses must not steal focus.
Initial panel presentation and all non-prompt completion, stash, capture, and plain-text
editing behavior must remain unchanged.

## Phase 1: Repair completion routing and pin the cache boundary

1. Open the linked repository with
   `sase repo open bob-mac-capture -r "Fix bob-cli-u landing regressions"` and use only
   the path it returns.
2. Refactor the completion branch in `Sources/BobMacCapture/CapturePanelModel.swift` so
   one request chooses exactly one candidate source: an available cached route response,
   otherwise `BobProcessClient.captureComplete`. Preserve generation/cancellation
   guards, selected-index reset, parse/live-preview concurrency, warning handling for
   process responses, and dismissal when neither source produces candidates.
3. Strengthen `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` so route spans for
   ordinary routes, `@route+...`, and `@route^...` prove the cache response remains
   visible after the analysis settles and prove no `capture-complete` argv is emitted.
   Also pin the fallback case with no usable cache and a true task-side completion case
   after `+`, proving the subprocess still runs with `--all-tasks`. Make the assertions
   deterministic rather than relying on inspecting the invocation log before a late
   subprocess has had time to start.

## Phase 2: Make prompt focus explicit and revalidate integration

1. Add the smallest explicit focus contract to
   `Sources/BobMacCapture/CapturePanelView.swift` and, if a request token or target is
   needed, `CapturePanelModel.swift`. Do not use timing delays or responder-chain luck.
   Coordinate the root `CapturePanelView`, `AutosizingCaptureEditor`, and
   `TaskIDPromptCard` so appearance, saving, failure, retry, cancel, successful
   replacement, and stale-response suppression produce the focus behavior described
   above.
2. Extend the model/view/controller tests to cover focus requests and their ordering
   with state mutation: success installs the returned ID and caret before requesting
   editor focus; cancel restores the same completion selection and requests editor focus
   without a write; failure retains the prompt/input/error and requests prompt focus; a
   response invalidated by cancel does nothing. Where possible, host the view and assert
   the actual focused control/first responder so the tests verify behavior, not only an
   incremented token. Keep the existing key-router tests for Return, Escape, Control-C,
   Command-Return, Tab, and arrows green.
3. Recheck the post-epic stash delete-all behavior around the touched focus/view paths:
   opening or clearing the stash must not coexist with the task-ID prompt, stash
   shortcuts must retain their documented precedence, and ordinary editor focus must
   still recover when the stash is dismissed.
4. Run focused macOS tests for `CapturePanelModelTests`, process-client completion argv,
   key routing/controller behavior, and the hosted prompt/view behavior. Then run the
   full linked-repository acceptance suite on an Xcode-capable macOS host:
   `just format-lint`, `just build`, `just test`, and `just bundle`. Also rerun
   `git diff --check`, `bash -n Tests/Fixtures/fake-bob`, its direct completion and
   assignment fixture checks, and `swift build --target CaptureCore` as portable guards.
5. Rebuild/revalidate the primary repository with `just all`, `just check-scripts`, and
   `just package-list`. Exercise the real Bob binary against a temporary vault with
   identified, unidentified, nested, terminal, and duplicate-description tasks. Verify
   existing-ID selection, Add-ID success, cancellation, local invalid input, duplicate
   ID, and stale/edited task failure; confirm only the selected task line changes and
   the draft expands only after the note write succeeds. Complete the plan's required
   light/dark, increased-contrast, keyboard, and VoiceOver inspection on macOS. Do not
   close the epic while any required validation is unavailable or failing.

## Phase 3: Close and clean up the epic

This is the final phase. Perform it only after phases 1-2 and all required validation
are complete.

1. Re-run `sase bead show bob-cli-u`, both child `show` commands, and the relevant
   histories before closing. Confirm both children remain closed, every note is now
   addressed, and there are still no unprocessed `PROPOSED FOLLOW-UP:` entries.
2. Close without force:

   ```bash
   sase bead close bob-cli-u --note "<concise audit, integration, regression-fix, follow-up, and validation summary>"
   ```

   The close note must name both verified phase commits, record that no proposed
   follow-ups were present, summarize the post-start integration commits and why they
   are compatible, state how the cached-route and focus issues were fixed, and list the
   successful primary and macOS validation. If close is rejected, finish or reopen the
   named phase; never use `--force` merely to make the command succeed.

3. After the close, check whether the primary repository exposes `just symvision`. If
   available, run it, remove every now-stale `bob-cli-u` whitelist entry and any unused
   code it reports, and rerun it until clean. Re-run the proportionate format/lint/test
   checks after cleanup. If no recipe is available, explicitly record that fact in the
   landing handoff rather than inventing a substitute.
4. Only after close and post-close cleanup succeed, edit the durable plan referenced by
   the bead, `plan:202608/file_plus_any_task.md`, to add `status: done` to its YAML
   frontmatter. Preserve the rest of the plan. Verify the plan and both repository
   worktrees contain only the intended landing changes, then hand off the completed
   epic.
