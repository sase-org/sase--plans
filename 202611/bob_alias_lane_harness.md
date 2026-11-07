---
tier: tale
title: Repair the eleven deterministic sigil-alias CI tests
goal:
  The eleven alias-lane tests pass on an unfiltered macOS snapshot and the landed master
  run, with no new failures caused by the repair.
size: medium
proposed_by: bbugyi200.athena.sase-1rq.4.1.1
proposed_with: codex/gpt-6-astra
bead: sase-1rq.4.1.1
create_time: 2026-11-07 08:16:49
status: wip
---

- **PARENT:**
  [202611/bob_mac_capture_ci_green.md](https://github.com/sase-org/sase--plans/blob/main/202611/bob_mac_capture_ci_green.md)
- **BEAD:**
  [sase-1rq.4.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.4.1.1.md)

# Repair the eleven deterministic sigil-alias CI tests

Implement assigned phase **sase-1rq.4.1.1** in `bobs-org/bob-mac-capture`. The governing
design is `plan:202611/bob_mac_capture_ci_green.md` (epic `sase-1rq.4.1`). Read it
through `sase artifact read`, and read the assigned bead for current notes and inherited
decisions. Open the repository with
`sase repo open gh:bobs-org/bob-mac-capture -r "Implement the alias-lane harness repair"`
and use only its returned checkout. Read that repository's `AGENTS.md`.

This is one bounded coding task: repair the fake protocol responses and test wiring
identified below, preserve the behavioral assertions, and verify on macOS CI. Monitor
continuations may carry the same work across CI waits. No earlier worker left a
remainder on this bead; this plan follows the initial investigation, not an unfinished
implementation attempt. No product or test files were changed while planning.

## Scope and constraints

The following eleven tests must pass; retain their coverage and names:

| Class                       | Test                                                           |
| --------------------------- | -------------------------------------------------------------- |
| CaptureSeparatorToggleTests | testPeriodAliasAfterRouteOpensNewIDBlockPicker                 |
| CaptureSeparatorToggleTests | testNativeUndoRestoresLiteralSigilAliasWithoutRetriggering     |
| CaptureSeparatorToggleTests | testDelayedAcceptedAliasSupersedesGatedCompletion              |
| CaptureSeparatorToggleTests | testPeriodAliasOnMoveBulletOpensMovePicker                     |
| CaptureSeparatorToggleTests | testSemicolonAliasOnOfferMoveBulletOpensMoveAndLinkPicker      |
| CaptureSeparatorToggleTests | testUndoAfterMoveAliasRestoresLiteralWithoutRetriggering       |
| CaptureSeparatorToggleTests | testCommaAfterCompleteTargetRunOpensNextParentPicker           |
| CaptureSeparatorToggleTests | testBackslashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses |
| CaptureSeparatorToggleTests | testSlashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses     |
| CapturePanelModelTests      | testCommaOnExactRouteUsesBobRewriteLane                        |
| CapturePanelModelTests      | testCommaCommitsPartialRouteThenBobOpensParentTaskPicker       |

The epic decisions remain final: `test_quarantine = quarantine_flakes` and
`release_merge = gate`. These eleven failures are deterministic and may not be
quarantined. Preserve CI jobs, watchdogs, timeouts, and lint levels. Do not change
memory notes. The word-motion and intermittent-test repairs belong to sibling phases.
Record unrelated discoveries only as `PROPOSED FOLLOW-UP:` notes on this bead; do not
create beads. Never close an ancestor. Never push master or create manual commits;
repository completion belongs to `/sase_final`.

## Verified findings and the corrected diagnosis

At investigation time, both local HEAD and remote master were
`1e0a092441d0a2bc0a50fc5c42ec09fd7463ce89`; the checkout was clean and no later fix had
landed. The existing macOS failure evidence is
[CI run 42337582432](https://github.com/bobs-org/bob-mac-capture/actions/runs/42337582432),
job `128297901212`. Build and lint passed; Test failed.

The parent design's claim that every alias reply is dropped is inaccurate. The log shows
no failed rewritten-draft assertions for the period, move-bullet, or next-parent cases:
the later picker/continuation assertions time out. The hash aliases likewise reach
`jot idea #` but receive `route` rather than `task_type` completion. Both route comma
tests first fail while waiting for their initial route completion, before typing the
comma.

Read `docs/capture-panel.md` (rewrite and parent-task sections), `docs/keyboard.md`, and
`docs/task-pickers.md` for the intended behavior. The relevant introducing commits are
`3cf0c6d`, `5ef0224`, `c95094d`, `8e2ecd7`, and `d206803`.

Directly executing `Tests/Fixtures/fake-bob` established these gaps:

| Request                                       | Current response                        | Consequence                                     |
| --------------------------------------------- | --------------------------------------- | ----------------------------------------------- |
| parse `@file^`                                | Generic task, empty `needs` and `spans` | No new-ID completion request                    |
| parse `@cash^goog-exit@\n- ^`                 | Generic task, empty `needs` and `spans` | No Move completion request                      |
| parse `@cash:goog-exit@\n- :`                 | Generic task, empty `needs` and `spans` | No Move & link completion request               |
| parse `@cash+goog-exit +`                     | Generic task, empty `needs` and `spans` | No next-parent completion request               |
| parse `Call bank @ca` / `Call bank @cash`     | Generic task, empty `needs` and `spans` | Initial cached route completion never starts    |
| complete `jot idea #` without fixture options | `context: route`                        | Wrong menu in both hash-alias tests             |
| complete `Call bank @cash+`                   | Earlier legacy branch without `picker`  | Later parent-task fixture branch is unreachable |

For comparison, parse `@file:` reports `pomodoro_id` plus appropriate spans and its
semicolon test passes. The route parse cases currently recognize capitalized `@Ca` and
`@Cash`, but not the lowercase strings used by the comma tests.

`CapturePanelModel.shouldRequestCompletion` intentionally requires Bob's parse needs or
a matching span, except for the leading-route special case. It must not be relaxed to
compensate for fake responses. `applyNativeRewrite` already requests analysis after an
accepted alias. The checked rewrite fixtures have matching inputs, valid edit ranges,
and valid post-rewrite cursors.

Other setup defects will remain after completing the fixtures:

- Neither comma test wires `model.applyNativeRewriteEdit`; the real controller wires it
  during initialization. The partial-route path also changes the model to
  `Call bank @cash,` before Bob replies, so a detached text view must receive that
  model-to-editor update before applying the native rewrite.
- The partial-route test asserts `Call bank @cash+` synchronously immediately after
  `editorTextDidChange`, despite the subprocess rewrite being asynchronous.
- The next-parent test and routed comma tests inspect `completionResponse` for a parent
  picker. `handleParentTaskCompletion` dismisses inline completion and presents
  `model.picker`; `CaptureParentTaskPanelTests` already verifies this representation.
  Check `.parentTask(context)` and its continuation instead of an intentionally cleared
  inline completion response.

## Implementation

1. **Refresh the baseline.** Re-read the bead, inspect status/history, fetch master and
   integrate newer changes without overwriting another agent's work. Compare the latest
   run's failures with the eleven names above. Skip repairs already present. Preserve
   sibling fixes and record unrelated regressions on this bead.

2. **Complete the fake protocol coverage.** In `Tests/Fixtures/fake-bob`, add narrowly
   matched parse cases for the six distinct draft shapes in the table. Use existing
   parse fixtures or neighboring faithful protocol responses, with correct input,
   grammar, mode, needs, and UTF-8 spans. Preserve capitalized route coverage while
   adding lowercase cases. Supply `block_id` / project-move / parent-task needs as
   appropriate; do not make the fake advertise completion for all drafts. Reuse
   `move-parse-*` and `parent-task-parse-*` data only after adjusting ranges and input
   to the actual shorter strings.

   Resolve the shadowed `Call bank @cash+` completion case so the modern comma tests
   receive `parent-task-complete-comma-routed.json` and its continuation. Preserve
   intentional legacy inline-completion coverage through an explicit fixture mode or
   distinct supported draft, checking the neighboring plus-route tests before choosing.
   Do not globally change all route completion responses or leave duplicate unreachable
   branches. Confirm the follow-up query/refetch cursor also gets consistent data.

3. **Repair the tests' environment and observations.** Enable
   `FAKE_BOB_TASK_TYPE_CHOOSER=1` only for the two task-type alias cases using the
   existing `processClient(extra:)` helper. Keep the second-key literal escape and
   no-extra-completion assertions intact.

   Wire the two route comma tests to the same native edit path as the real panel,
   following the hosted/native helpers in `CaptureSeparatorToggleTests`,
   `CaptureHashAliasDoubleTapTests`, and `CaptureParentTaskPanelTests`. Preserve both
   model-to-editor and editor-to-model synchronization around partial-route acceptance;
   do not fabricate a successful rewrite in the callback or weaken `expectedDraft`. Use
   bounded condition waits for the actual post-rewrite text and picker. Assert the
   parent picker source, scope, continuation ` +`, final text/caret, and recorded Bob
   rewrite/completion calls; assert inline completion is cleared where appropriate.
   Update the next-parent test to observe that same public picker state.

   Preserve the distinction between a partial route (accept route, then ask Bob to
   rewrite comma) and an exact route (ordinary rewrite lane). Preserve the undo tests'
   real native undo checks and their proof that undo does not trigger another rewrite.
   Explain in the commit message why the changed setup/state assertions match the real
   panel rather than weakening the intended behavior.

4. **Check the repair before broad CI.** Replay the fake parse/rewrite/complete calls
   directly, checking exact input, UTF-8 ranges, needs, completion context, and picker
   continuation. Reuse existing protocol test infrastructure for any additional
   regression needed to catch a shadowed dispatch branch. The eleven existing tests
   remain the integration regressions; do not add a large duplicative test suite. Format
   only changed Swift lines and run relevant shell/script checks.

   Start with a harness-only repair. If macOS execution still exposes a product bug,
   trace that specific failing guard and change only the proven defect while retaining
   generation, draft, caret, IME, and undo protections. Temporary diagnostics or
   filtered snapshots are allowed by the parent design, but remove them before the final
   run.

## Compilation, CI, and completion

Follow the opened repository's `AGENTS.md`; never land uncompiled Swift. Run
`just portable-check` on athena. For changed macOS tests or product sources, read the
`tailnet` reference memory, create an isolated scratch copy on the Mac via rsync
excluding `.git` and `.build`, run `Scripts/bundle.sh` and `just typecheck-tests`, then
delete that scratch copy. Never use Bryan's checkout or install there. The Mac has
Command Line Tools without XCTest: type-checking is not test execution. If it is
offline, record that local macOS validation was unavailable and rely on successful CI
Build/Test evidence for the final tree.

Before landing, run an **unfiltered** `Scripts/macos-ci-check` from the opened
repository. Use `/sase_monitor`, profile `verify`, timeout `95m`, and a `--next` prompt
carrying this bead, the eleven test names, and the requirement to inspect their actual
results and any new failures. Wait for the staging command to exit, then finish through
`/sase_final` according to its host context; preserve pre-land verification ordering. Do
not wait for CI inline or treat a staged request as a completed test. Any diagnostic
workflow edits and instrumentation must be absent from the final snapshot.

Exit 2 is unverified and requires repair/retry. Exit 1 can reflect the sibling phases'
known failures: inspect the completed Test output, confirm all eleven tests executed and
passed, and confirm there are no new failures caused by this change. A missing test name
in a failed Build or truncated log is not proof. Record run ID, snapshot SHA, test
results, and unrelated failures on this bead. An unchanged-base failure is a
`PROPOSED FOLLOW-UP:` with clean-base evidence, not grounds to leave the completed phase
open. Do not quarantine a deterministic failure to satisfy this plan.

After successful pre-land evidence, submit the external repository's commit decision
through `/sase_final`; keep the bead open until the required landed-run evidence is
collected. Stage a `/sase_monitor` continuation that discovers and watches the exact
landed master SHA, then confirms these tests still pass. Do not accidentally watch an
older green run. An overall red result from known sibling failures is acceptable for
this phase when its eleven tests pass and it introduced no new failures.

Record the landed SHA and both snapshot/master run evidence. Run
`sase bead epic-symbols sase-1rq.4.1.1`, resolve any entries or re-key them to an
appropriate still-open bead, then close **only** `sase-1rq.4.1.1` with
`sase bead close sase-1rq.4.1.1 --note "<fix summary and verification evidence>"`. Do
not close `sase-1rq.4.1`, `sase-1rq.4`, or `sase-1rq`. No manual status transition or
reopen is needed for the assigned phase. Finish every root turn with `/sase_final`.
