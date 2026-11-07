---
tier: epic
title: Turn bob-mac-capture master CI green
goal: 'The `CI` workflow (job `macOS 26 SwiftPM`) on `bobs-org/bob-mac-capture` master
  is `completed` / `success` at a head SHA at or after the last fix commit of this
  epic, with `ci_evidence` published. Or, after a very large, well-documented effort,
  a research-sidecar report tells Bryan exactly what is still needed. When either
  is done, the land agent closes phase bead sase-1rq.4 of epic sase-1rq.

  '
phases:
- id: alias-lane
  title: Fix the eleven sigil-alias rewrite-lane tests that have never passed on CI
  depends_on: []
  size: large
  description: 'alias-lane: fix the 11 deterministic CaptureSeparatorToggleTests /
    CapturePanelModelTests sigil-alias failures (`.`, `;`, `,`, `\`, `/` aliases)
    that fail on every macOS 26 CI run. Find the shared root cause in the Swift rewrite
    lane or the test harness, fix it, and prove it with Scripts/macos-ci-check before
    landing.'
- id: word-motion
  title: Fix the Ctrl-H/L word-motion undo test
  depends_on: []
  size: small
  description: 'word-motion: fix CaptureEditorWordMotionTests.testMotionsLeaveTextAndUndoUntouched
    (XCTAssertFalse at line 67: the undo manager can undo after Ctrl-L then Ctrl-H).
    It has failed on every CI run since edb2f48.'
- id: flakes
  title: Harden or quarantine the intermittent alias and card tests
  depends_on:
  - alias-lane
  size: medium
  description: 'flakes: after alias-lane lands, classify each intermittently failing
    test: still failing, timing-sensitive (harden it), or a proven flake on an unchanged
    tree (quarantine it under the parent test_quarantine decision). Leave the suite
    with no known intermittent failures.'
- id: master-green
  title: Drive master CI green and publish evidence
  depends_on:
  - alias-lane
  - word-motion
  - flakes
  size: medium
  description: 'master-green: watch bob-mac-capture master CI after every fix phase
    has landed. Repair any quick new regression that concurrent bob-cli agents landed,
    get a green first-attempt run at or after the last fix commit, and publish `ci_evidence`
    (or the fallback research report) for the parent phase sase-1rq.4.'
proposed_by: bbugyi200.athena.sase-1rq.4
proposed_with: claude/opus
parent_bead: sase-1rq.4
create_time: 2026-11-07 08:03:35
status: wip
bead_id: sase-1rq.4.1
---

- **PROMPT:** [prompts/202611/bob_mac_capture_ci_green.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/bob_mac_capture_ci_green.md)
- **PARENT:** [202611/red_ci_repair.md](https://github.com/sase-org/sase--plans/blob/main/202611/red_ci_repair.md)
- **BEAD:** [sase-1rq.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.4.1.md)

# Plan: Turn bob-mac-capture master CI green

This epic is the child plan for phase **sase-1rq.4** ("Turn bob-mac-capture CI green")
of epic **sase-1rq** ("Repair red GitHub Actions CI across five projects"). The parent's
DECISIONS are final and bind every phase here:

- `test_quarantine = quarantine_flakes`: a test proven flaky on an unchanged tree may be
  skipped, each with a `flake` bead. CI timeouts, the job list, and lint levels stay
  exactly as they are.
- `release_merge = gate`: bob-mac-capture has no release PR in scope, so this decision
  does not apply here.

## What Is Actually Wrong (Corrects The Parent Plan)

The parent plan describes "a hang that trips the 15 minute swift test watchdog". **That
is wrong.** The watchdog has not fired in any of the 11 recent runs checked (8 master, 3
`ci/precommit/*`). The text `swift test ran longer than 15 minutes; sampling xctest` in
those logs is the workflow's own script source, which GitHub echoes at the top of the
Test step. It is not an emitted `::error::`. Every run's Test step finishes in about 9
minutes. The latest master run, `1e0a092` (run `42337582432`), shows:

```
Executed 4307 tests, with 1 test skipped and 29 failures (0 unexpected) in 517.442 seconds
##[error]Process completed with exit code 1.
```

The failing step is Test, so Bundle, Verify plist and signature, Launch smoke test, and
Install and reinstall are all skipped.

Master has been red on every push since `fde8211` (2026-11-04 18:40 UTC). The last green
master run is `4906d94` (run `41888215951`). Failures piled up as features landed:
almost every failing test was red in the CI run of the commit that introduced it, so
**these tests have never passed on CI**.

The cause of the pile-up is structural. The tailnet `mac` (macOS 26.5) has only Command
Line Tools. That install ships Swift Testing but no `XCTest.framework`, so no agent can
run this XCTest suite locally. Authors only type-checked their tests
(`just typecheck-tests`), and CI is the first place the tests ever run.

### Deterministic failures (red on all 11 of the last 11 runs)

| Test (`BobMacCaptureTests.`)                                                                 | Introduced by                                           | First symptom in `1e0a092` log                                        |
| -------------------------------------------------------------------------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------- |
| `CaptureSeparatorToggleTests.testPeriodAliasAfterRouteOpensNewIDBlockPicker`                 | `3cf0c6d` feat(capture): add sigil aliases to picker    | `:999` Condition not met before timeout (`pickerVisible`)             |
| `CaptureSeparatorToggleTests.testNativeUndoRestoresLiteralSigilAliasWithoutRetriggering`     | `3cf0c6d`                                               | `:1071` Condition not met before timeout                              |
| `CaptureSeparatorToggleTests.testDelayedAcceptedAliasSupersedesGatedCompletion`              | `3cf0c6d`                                               | `:1223` Condition not met before timeout                              |
| `CaptureSeparatorToggleTests.testPeriodAliasOnMoveBulletOpensMovePicker`                     | `5ef0224` (move-bullet aliases)                         | `:1115` timeout, then "expected a Move picker"                        |
| `CaptureSeparatorToggleTests.testSemicolonAliasOnOfferMoveBulletOpensMoveAndLinkPicker`      | `5ef0224`                                               | `:1142` timeout, then "expected a Move picker"                        |
| `CaptureSeparatorToggleTests.testUndoAfterMoveAliasRestoresLiteralWithoutRetriggering`       | `5ef0224`                                               | `:1170` Condition not met before timeout                              |
| `CaptureSeparatorToggleTests.testCommaAfterCompleteTargetRunOpensNextParentPicker`           | `c95094d` feat(capture): comma parent-task continuation | `:1366` timeout; `:1368` `nil` != `" +"`                              |
| `CaptureSeparatorToggleTests.testBackslashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses` | `8e2ecd7` backslash hash alias                          | `:1709` picker kind `"route"` != `"task_type"`                        |
| `CaptureSeparatorToggleTests.testSlashAliasAfterTaskTypeOpensTypeMenuAndDoubleTapCloses`     | `d206803` `/` through the hash-alias lane               | `:1851` picker kind `"route"` != `"task_type"`                        |
| `CapturePanelModelTests.testCommaOnExactRouteUsesBobRewriteLane`                             | `c95094d`                                               | `:6062` timeout; `:6091` `"Call bank @cash,"` != `"Call bank @cash+"` |
| `CapturePanelModelTests.testCommaCommitsPartialRouteThenBobOpensParentTaskPicker`            | `c95094d`                                               | `:5998` timeout; `:6027` `"Call bank @ca,"` != `"Call bank @cash+"`   |
| `CaptureEditorWordMotionTests.testMotionsLeaveTextAndUndoUntouched`                          | `edb2f48` vim-style Ctrl-H/L word motions               | `:67` XCTAssertFalse failed (`undoManager.canUndo` is true)           |

The first 11 share a shape. A sigil-alias key is noted through
`CapturePanelController.noteSeparatorToggleIntentIfEligible` (with a `timestamp: 0`
`NSEvent`), inserted natively into an `NSTextView`, and handed to
`CapturePanelModel.editorTextDidChange(cursorUTF8Offset:)`. Bob's `capture-rewrite`
reply (rule `expand_sigil_alias`, served by `Tests/Fixtures/fake-bob` from fixtures such
as `capture-rewrite-sigil-alias-task-block-id.json`) is then never applied: the draft
keeps the literal alias character, and the follow-on picker never opens.

Sibling tests on the same helpers pass, for example
`testTypedColonAfterRouteOpensLinkPickerThroughKeyIntent` (rule
`switch_block_id_separator`). So the break is specific to the `expand_sigil_alias`
branch of the reply handling, around `CapturePanelModel.swift` lines 15980–16560 and
`applyNativeSingleEditRewrite`. The fake-bob fixtures and the `capture-complete` cases
for the post-rewrite drafts (for example `@file^`) do exist.

### Intermittent failures (red on 1–2 of the last 11 runs)

`CaptureRelationFlipAliasTests.testLabeledRelationTildeFlipsAndEqualsStaysLiteral` (2×;
throws NSCocoaErrorDomain 260 reading the fake-bob record file, so the rewrite was never
called),
`CaptureSeparatorToggleTests.testTypedCommaInCloseTaskListStaysLiteralForCloseGrammar`
(red on master `1e0a092`, green on its own precommit snapshot `41b5e596`),
`CaptureSeparatorToggleTests.testRelationEqualsIntentForwardsAtOpenerPositionsAndRelationEnds`,
`CaptureRelationEqualsAliasTests.{testTripleEqualsGivesDoubleEquals, testFastSecondEqualsUndoesOpenerAfterReply, testEqualsGestureGuardsRepeatMarkedTextAndZeroTimestamps, testEqualsAliasOpensLinksToAndCallsRewriteBeforeCompletion}`,
`CapturePullInChoiceTests.testFooterContractPinsHintsAndAccessibilityText`,
`CaptureMoveChoiceModelTests.testMoveThenPullInBackAndInboxCarryOneAnswerAndSeed`, and
`CaptureHashAliasDoubleTapTests.testSlowDoubleTapClosesMenuWithOneLiteralBackslash`.

Some of these were red only on the commit that introduced them, and the next commit
fixed them (`1e0a092` "repair fast == escape and record race"). Others appear on a
single precommit snapshot that may have held unlanded work. Treat them as **unclassified
until re-observed**.

## Contract Shared By Every Phase

### Repository and rules

- Open the repo with `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"` and work
  only in the printed path. Read its `AGENTS.md` first and follow it: compile before
  landing, format only the lines you change (`.swift-format`: 4-space indent, 100
  columns), never re-indent whole files, and no async test may await an untimed
  continuation or gate.
- Never push to master yourself, never force-push, and never change repository settings,
  variables, secrets, or `.github/workflows/ci.yml` timeouts or jobs. Host completion
  lands your changes after you submit a `commit` decision for this repo through
  `/sase_final`.
- Phase workers do not create beads. Record discovered work as
  `sase bead note <your-phase-bead> 'PROPOSED FOLLOW-UP: <summary — detail>'`. Proposed
  `flake` beads for quarantined tests go there too; the land agent files them.
- Prefer root-cause fixes. A test that encodes the intended behavior of its feature
  (read the introducing commit and the matching `docs/` page) is the spec: fix the
  product, not the assertion. Change a test only when its setup is unrealistic compared
  with the real panel. For example, a windowless `NSTextView` has no undo manager or
  window, so move that test to the hosted helpers. Then say why in the commit message.
  Never delete an assertion just to go green.
- Replacing a fixed `Task.sleep` in test code with condition polling, or with an
  explicit fake-bob barrier, is a root-cause fix and is allowed. Raising the workflow's
  15-minute watchdog, the 25-minute Test step limit, or the 60-minute job limit is not.

### Verifying without local XCTest

No host available to agents can run XCTest. Verification has three layers:

1. **Compile.** On athena, run `just portable-check` (Linux Swift 6.0.3; it covers
   `CaptureCore`/`RefsCore`). For changes under `Sources/BobMacCapture*` or
   `Tests/BobMacCapture*`, also compile a scratch copy on the `mac` tailnet host (best
   effort; read the `tailnet` reference memory first). Rsync without `.build`/`.git`
   into a temporary directory, run `./Scripts/bundle.sh` and `just typecheck-tests`
   there, then delete the copy. Never use Bryan's checkout or `just install` on the mac.
   If the mac is offline, say in the final report that the macOS targets are unverified
   locally.
2. **Run on CI before landing.** Run `Scripts/macos-ci-check` from the repo checkout. It
   snapshots the working tree to a throwaway `ci/precommit/<id>` branch, waits for that
   SHA's run, prints every failed test name, and deletes the branch. It takes about
   15–20 minutes, so start it through `/sase_monitor` with a `--next` that tells the
   follow-up turn what to inspect. Never run it inline.

   Until every phase has landed, the snapshot run stays red overall. Judge it by the
   printed failure list: your phase's tests must be absent, and no test outside the
   tables above may newly fail. Exit code 2 means the run is unverified: rerun it, don't
   land on it.

3. **Watch after landing.** After your fix lands, watch the exact-SHA master run with
   `/sase_monitor` (the `AGENTS.md` rule). Confirm that your tests stay absent from its
   failure list.

Diagnostic snapshots are encouraged. You may add temporary instrumentation (for example
`FileHandle.standardError.write` lines with a unique prefix at each early `return` in
the alias reply path) and run `Scripts/macos-ci-check` on that tree. Then read the
retained run log with `gh run view <run_id> -R bobs-org/bob-mac-capture --log-failed`.
For a diagnostic-only snapshot you may also narrow the Test step to
`./Scripts/xcode-swift.sh test --filter <Class>` to save about 8 minutes. Before
declaring, revert every instrumentation line and every `.github/` change:
`git diff --stat` must show only the intended fix. The final pre-land snapshot must be
unfiltered.

### Concurrency with other agents

bob-cli project agents land features to this repo every few hours, and they also add
tests that have never run. Before each CI round, `git fetch` and rebase onto
`origin/master` (no history rewriting of landed commits). Re-read the newest master
run's failure list too. A test that newly fails because of someone else's commit is
outside a fix phase's scope: record it as a `PROPOSED FOLLOW-UP:` note and leave it to
`master-green`.

## Phase: alias-lane

Scope: the 11 sigil-alias tests in the deterministic table (every row except
`CaptureEditorWordMotionTests`).

Approach:

1. Read the introducing commits (`3cf0c6d`, `5ef0224`, `c95094d`, `8e2ecd7`, `d206803`)
   and the keyboard/capture docs to learn the intended user-visible behavior of each
   alias.
2. Trace one test end to end before changing anything. Start with
   `testPeriodAliasAfterRouteOpensNewIDBlockPicker`: typed `.` after `@file` should
   rewrite to `@file^` through `capture-rewrite`, then open a new-ID Block ID picker.
   Trace `noteSeparatorToggleIntentIfEligible`, then `noteSeparatorToggleIntent`, then
   the `editorTextDidChange` consumption of `pendingSeparatorToggleIntent`, then the
   rewrite request and reply handling. The reply handling includes the
   `collapsedSelectionUTF8Offset()` early return, the `expand_sigil_alias` branches, and
   `applyNativeSingleEditRewrite` with `applyNativeRewriteEdit` /
   `applySeparatorToggleInEditableTextView`.
3. List every early `return` or guard that would drop an `expand_sigil_alias` reply and
   not a `switch_block_id_separator` one. Leads to check, not conclusions:
   - Gating on `NSEvent.timestamp` (the tests send `0`; the equals-gesture code has an
     explicit zero-timestamp guard).
   - Selection or caret bookkeeping that the hosted panel updates but these windowless
     tests never feed.
   - A `deferredRewriteInputDraft` or completion gate that was never released.
   - A grammar-version check (fake-bob injects `grammar: 15` into `capture-parse` unless
     told otherwise).
4. If static reading does not settle it, run one diagnostic snapshot with
   instrumentation across all 11 tests (filtered to `CaptureSeparatorToggleTests` and
   `CapturePanelModelTests`) and read which guard fires.
5. Fix the shared root cause. Then fix any test-specific remainder. The `\` and `/`
   type-menu tests also assert that a double-tap closes the menu, and
   `testCommaCommitsPartialRouteThenBobOpensParentTaskPicker` first commits a partial
   route.
6. Compile (layer 1), then run an unfiltered `Scripts/macos-ci-check`. All 11 tests must
   be absent from its failure list with no new failures. Land, then watch master (layer
   3).

Done when: the 11 tests pass on an unfiltered snapshot run and on the master run of the
landed fix. If one of them truly cannot be fixed, it stays red. Document it in a
`PROPOSED FOLLOW-UP:` note with evidence and leave the decision to `master-green`. A
deterministic failure is not a flake and must not be quarantined.

## Phase: word-motion

Scope: `CaptureEditorWordMotionTests.testMotionsLeaveTextAndUndoUntouched`. After
`routeKeyDown(controlL())` and then `routeKeyDown(controlH())` on a hosted `"foo bar"`
editor, `textView.undoManager?.canUndo` is true on CI. The sibling motion tests in the
same file pass.

Approach: read `edb2f48` and the Ctrl-H/L handling in `CaptureKeyCommandRouter` /
`CapturePanelController.routeKeyDown`. Find what registers an undo action. Candidates
include:

- Ctrl-H reaching AppKit's default `deleteBackward:` binding.
- A selection or attribute change registered as undoable.
- The panel's window-level undo manager already holding an action from
  `makePanelIfNeeded()` or from `textView.string =` before the motions run.

If the motion itself is undoable, fix the product. A word motion must never edit text or
leave an undo step. If the undo step predates the motions, fix the test setup: clear the
undo stack (`removeAllActions()`) after building the editor and before the motions, and
assert `canUndo` is false at that point too. Compile, run an unfiltered
`Scripts/macos-ci-check`, land, and watch master as in the shared contract.

## Phase: flakes

Runs after `alias-lane` lands, because the equals, dash, and hash alias tests share its
rewrite lane, and its fix may change their behavior.

For each test in the intermittent list, plus any test that fails intermittently on a
post-`alias-lane` master run:

1. Gather evidence. Use the failure lists of the last several master and
   `ci/precommit/*` runs (`gh run list -R bobs-org/bob-mac-capture -L 40`, then
   `gh run view <id> --log-failed`).
2. Classify it as **still failing**, **timing-sensitive**, or **flaky**:
   - **Still failing**: deterministic since `alias-lane`. Fix it like `alias-lane`.
   - **Timing-sensitive**: a fixed `Task.sleep` (for example the 160 ms sleep in
     `relationDashWasForwarded`, or the 400 ms sleeps before negative assertions), a
     5-second `waitUntil` that the slow runner overshoots, or a read of the fake-bob
     record file before the fake has written it (NSCocoaErrorDomain 260). Harden it:
     poll for positive conditions; for negative ones, wait on a later sentinel fake-bob
     call or another deterministic barrier instead of wall-clock time.
   - **Flaky**: it fails and passes on the same unchanged tree, and no harness cause can
     be found after a real attempt. Quarantine it. Skip with
     `throw XCTSkip("flaky on macOS 26 CI; see PROPOSED FOLLOW-UP on <phase bead>")` and
     record one `PROPOSED FOLLOW-UP:` per test proposing a `flake` bead. Include the run
     ids that show both outcomes.
3. Prove stability with at least two unfiltered `Scripts/macos-ci-check` runs on the
   same tree. Neither may show an intermittent failure from this phase's list. Land,
   then watch master.

## Phase: master-green

Runs after every fix phase has landed. This phase's final turn carries the parent
phase's evidence.

1. Read the newest master `CI` run. Any remaining failure gets one of three treatments:
   - It belongs to an earlier phase's scope and that phase recorded it as unfixable: try
     once more with fresh eyes, otherwise keep it for the report.
   - It is new and comes from a concurrent bob-cli commit: fix it when the fix is quick
     (the parent contract). Otherwise record a `PROPOSED FOLLOW-UP:` for a `ci` bead and
     cite it in the evidence notes.
   - It is intermittent: treat it under the `flakes` rules.
2. Iterate (compile, `Scripts/macos-ci-check`, land, watch) until a master `CI` run at
   or after the last fix commit of this epic is `completed` / `success`, with Bundle,
   Verify plist and signature, Launch smoke test, and Install and reinstall all
   succeeding.

   A first-attempt green is the goal. If green arrives only on a rerun, rerun at most
   once, say so in the evidence, and handle each test involved under the `flakes` rules.
   Never use `gh run rerun` to paper over a deterministic failure.

3. Before ending the green-confirming turn, publish `ci_evidence` through `/sase_var`
   (JSON) with:
   - `repo` = `bobs-org/bob-mac-capture` and `default_branch` = `master`
   - `tip_sha`
   - `runs`: one entry for workflow `CI`, with `run_id`, `url`, `head_sha`, `event`,
     `attempt`, `conclusion`, and `completed_at`
   - `fixes`: the landed SHAs from all four phases, each with a one-line summary
   - `quarantined`: tests plus the phase bead holding each flake proposal, or empty
   - `notes`: include that the parent's "watchdog hang" premise was wrong, and any
     remaining follow-ups

   Repeat the evidence as a short Markdown table in the final response, with the command
   that reproduces it:
   `gh run list -R bobs-org/bob-mac-capture -b master -w CI -L 1 --json databaseId,headSha,event,attempt,conclusion,updatedAt,url`.

4. **Fallback.** Use this only after several distinct, evidence-backed hypotheses have
   each been tried against real CI runs. Write the report described in the parent plan
   (`plan:202611/red_ci_repair.md`, "Fallback: Research Report") in the sase project's
   `research` sidecar (`sase repo open sase--research`, following its conventions). Land
   every strict improvement first, and publish `ci_report` through `/sase_var`. Likely
   items for Bryan in such a report: install full Xcode on the mac so XCTest can run
   locally, or provide another macOS host with Xcode.

## Land Agent

When all four phases are closed, confirm two things: `master-green` published
`ci_evidence` showing a green master `CI` run at or after the last fix commit (or a
`ci_report`), and every phase's code has landed on `bobs-org/bob-mac-capture` master.

Triage every `PROPOSED FOLLOW-UP:` note on this epic's phase beads into task beads
through `/sase_new_task`: `flake` beads for quarantined tests and `ci` beads for others'
regressions. Record the created bead ids against the quarantined tests in a note.

Then close the parent phase with
`sase bead close sase-1rq.4 --note "<green run id, tip SHA, fixes, quarantined beads>"`.
Run `sase bead epic-symbols sase-1rq.4` first and resolve any leftovers. Do not close
epic `sase-1rq` or any other ancestor.
