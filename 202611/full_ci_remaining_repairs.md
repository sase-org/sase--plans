---
tier: epic
title: Finish the remaining Full CI repairs
goal: Repair the remaining Files and Main deck races and CI-only snapshot differences,
  integrate the separately owned bulk golden repair, and prove a first-attempt green
  Full CI run on master containing every fix.
parent_bead: sase-1rq.2.1.5
phases:
- id: files-reserve
  title: Make the Files reserve regression deterministic
  depends_on: []
  size: medium
  description: 'files-reserve: resolve sase-1qd by diagnosing the initial short-extent
    setup timeout in test_trailing_reserve_remeasures_when_flag_already_enabled; preserve
    the real remeasurement regression and verify under coverage/load without raising
    timeouts. Coordinate with contention phase sase-1rq.2.1.4.'
- id: reply-preservation
  title: Preserve Reply selection across same-subject delivery
  depends_on: []
  size: medium
  description: 'reply-preservation: fix the bounded Main-deck scroll ownership defect
    tracked by sase-1ln, add a real same-subject delivery regression, and stabilize
    agents_renamed_generic_session_root_120x40 without hiding user scroll changes
    or accepting the clamped frame as the golden.'
- id: visual-fixtures
  title: Stabilize the stranded modal and empty Beads captures
  depends_on: []
  size: medium
  description: 'visual-fixtures: identify and fix the state leaking into agents_stranded_resolve_modal_120x40
    and artifacts_beads_empty_120x40; compare CI actuals with local captures, control
    the specific focus/layout or inbox seam, and prove exact matching with the normal
    and UTC environments.'
- id: verify-full-ci
  title: Integrate the repairs and prove Full CI green
  depends_on:
  - files-reserve
  - reply-preservation
  - visual-fixtures
  size: medium
  description: 'verify-full-ci: confirm all child fixes and sase-1r2.6''s bulk golden
    repair landed, reuse or dispatch one qualifying Full CI run, monitor it across
    turns, repair remaining in-scope reds, and publish exact first-attempt job evidence
    for the land agent. Never close the parent phase from this worker.'
proposed_by: bbugyi200.athena.sase-1rq.2.1.5
proposed_with: codex/gpt-6-astra
create_time: 2026-11-07 11:09:13
status: done
bead_id: sase-1rq.2.1.5.1
---

- **PROMPT:** [prompts/202611/full_ci_remaining_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/full_ci_remaining_repairs.md)
- **PARENT:** [202611/sase_ci_green.md](https://github.com/sase-org/sase--plans/blob/main/202611/sase_ci_green.md)
- **BEAD:** [sase-1rq.2.1.5.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1rq/sase-1rq.2.1.5.1.md)

# Plan: Finish the remaining Full CI repairs

This child epic implements phase `sase-1rq.2.1.5` of `plan:202611/sase_ci_green.md`,
under `plan:202611/red_ci_repair.md`. The three repair phases have separate failure
mechanisms and can proceed independently; integration depends on all three. One medium
coder can handle each repair slice. The multi-hour GitHub run belongs to a verification
phase with monitored continuation.

## Evidence and remaining work

The assigned phase was `in_progress` with no earlier implementation notes when read. No
status changes or reopen operation are needed. On 2026-11-07, this checkout and
`origin/master` were at `b7ca9f34503a4e348d9f5538c527ebb4b7e1d768`. Re-read the current
state before implementing; other agents are landing work.

The latest Full CI run was
[42370668331](https://github.com/sase-org/sase/actions/runs/42370668331), attempt 1, on
`d2aa3ef832f8c8b620dbb9856eb016d0d79e0e09`. GitHub's job API confirms two failures:
`full / test (3.12)` and `full / visual-test`. Eight other jobs succeeded; `docs-build`
and `release-core-floor-smoke` were skipped by design on master.

- The Python failure is job `128401500987`. Its traceback points to line 364 of
  `tests/ace/tui/widgets/decks/test_files_anchor_ownership.py`: the **first** wait after
  disabling reserve and requesting layout, before `panel.cycle_card(1)`. Its predicate
  requires a live anchor and `max_scroll_y < anchor`. The later remeasurement wait did
  not run. Task `sase-1qd` is READY and explicitly forbids raising the wait timeout.
- `sase-1r2.6` notes 5–6 describe a verified repair of 109 AGENT PROMPT heading goldens,
  a capacity hostname pin, and a `turns_gate_90x40` footer wait. The notes say it
  submitted host completion, but that commit was absent from master at planning time.
  Its round-two continuation was obstructed by staged monitor `es67j8`. No live
  `sase-1r2.6` agent appeared in the agent list, but its last progress was less than two
  hours old: this does not yet authorize taking over its goldens.
- That phase explicitly left the three visual cases in this plan and `sase-1qd`. A
  coordination note naming this split has been appended to `sase-1r2.6`.
- Task `sase-1ln` documents Reply selection stolen by a same-subject Main delivery: the
  spread scroll clamps to zero, and the scroll watcher stores Context while the view
  still identifies Reply. The current regression test asserts this buggy state and then
  tests a helper that reselects Reply. It is not a product fix.
- The stranded-modal test already waits for the three-member jump map and its fitted
  border title. Repeating those existing waits is insufficient without identifying which
  remaining focus, badge, border, or refresh transition explains the CI frame. The
  empty-Beads test already waits for its snapshot, project name, and detail text; its
  reported 389-pixel difference near the inbox label is still unidentified.

The run's actual artifact names were checked. Use `visual-failure-report.html` (artifact
`13573974664`) for the report and `ace-visual-first-attempt-diffs` (`13574573683`) for
initial differences; `ace-visual-artifacts` (`13574698322`) contains the larger raw
bundle. Do not assume the old plan's `ace-visual-failure-report` name exists. Download
only the needed artifacts into the current workspace's ignored scratch/cache directory.
For example, use
`gh run download 42370668331 --repo sase-org/sase --name ace-visual-first-attempt-diffs --dir .pytest_cache/full-ci-evidence/first-attempt`.
Inspect the report/manifest and images, not merely the filename or failure count.

## Inherited contract and ownership

The final decisions remain `test_quarantine = quarantine_flakes` and
`release_merge = gate`. This plan introduces no new reviewer decisions and no memory
edits. Read the two parent plans through `sase artifact read`, and relevant memories
through `/sase_memory_read`: `ace.md`, `tui.md`, `tui_perf.md`, `tui_screenshot.md`,
`lint_and_test.md`, `decisions:ci-two-speed-split`, and
`decisions:check-full-is-explicit`.

- Prefer root-cause fixes. Quarantine only after a failure and pass on an unchanged tree
  under the same conditions, with the exact node, SHA, commands, and logs recorded. A
  loaded failure followed by a focused pass alone does not satisfy that condition. Cite
  existing task IDs in skip reasons where available. Never skip a deterministic failure,
  increase sleeps/timeouts, or accept a transient/broken frame as a golden.
- Preserve workflow job lists, timeouts, permissions, recapture caps, exact pixel
  comparison, contention width/repeats/CPU pinning, lint levels, and all other gates. No
  repository settings or release operations are needed. Do not dispatch `release.yml` or
  merge release PRs.
- Work in the current sase checkout. These fixes concern Textual state and fixtures. If
  a fix actually requires shared backend behavior, first open `sase-core` through
  `/sase_repo`, put that behavior there, and consume it through the binding. Do not move
  the core pin past published-release coverage and recreate CORE_LAG.
- Each phase first checks the latest notes and relevant history for already-landed
  fixes. Record remaining discoveries as `PROPOSED FOLLOW-UP:` notes on its own phase;
  create no task beads. Existing tasks `sase-1qd` and `sase-1ln` remain for the outer
  land agent to settle using the fix evidence; child workers close only their own phase.
- The Files worker reads `sase-1rq.2.1.4` before editing: the contention phase may
  encounter the same node. Record the intended ownership split in the assigned phase
  and, as the approved parent coordination contract requires, on the overlapping phase.
  Whichever fix lands first is reused by the other worker.
- Re-read `sase-1r2.6` and `sase agent list` before any golden update. Preserve its bulk
  heading family, capacity pin, and gate-footer work while it owns them. Never run a
  whole-suite screenshot update while that work is pending. Only the integration worker
  may take over the remainder after that phase closes or has no live agent and no
  progress for over two hours; record that takeover before proceeding. Do not reset,
  cancel, or alter the other phase's monitor as a shortcut.

For code changes, run `just fix`, relevant targeted tests, and `sase tool run check`.
Use `/sase_monitor` for commands that cannot finish in the current turn. Whole-suite
local lanes require realistic capacity weight and at most one per phase; prefer targeted
tests and let Full CI provide exhaustive validation. The parent explicitly authorizes
`just check-full`, but it is optional, must be monitored, and its screenshot update can
conflict with the separately owned bulk repair.

Any visual update requires inspection of every creation/removal and each update group.
Then run the targeted `--check` selectors twice, once normally and once with `TZ=UTC`,
with no unstable nodes. Pin hostname, clock, fixture data, or focus only at the seam
shown by evidence; retain the canonical renderer/fonts. A `partial` update is not proof
that all selected goldens are current. Do not broadly change shared readiness helpers to
conceal a case-specific failure. If a shared helper needs a change, coordinate ownership
and verify its other callers.

## Phase: files-reserve

1. Read `sase-1qd`, `sase-1rq.2.1.4`, and the latest failing job log. Check commits
   `2e68053dab`, `aaa88b0026`, and `691ef46153` and their current descendants. The
   production remeasurement trigger already exists in
   `src/sase/ace/tui/widgets/decks/_panel_spread_files_scroll.py`.
2. Reproduce the node and its whole module with `tools/run_pytest fast` and
   `tools/run_pytest cov`, then with a bounded loaded subset at the CI coverage leg's
   three-worker width. Use supported worker settings and two allowed CPUs when
   reproducing contention; retain exact conditions in the evidence. Do not run the full
   26-worker contention suite as this phase's default.
3. Add useful failure diagnostics to the existing setup wait: anchor, actual content
   height, scroll extent, reserve flag/value, render generation, and pending restore.
   Determine whether delayed initial rendering changes the content/geometry after the
   test replaces `get_content_height`, or the intended cache/layout invalidation is
   missing. These are hypotheses, not established root causes.
4. Fix the setup ordering or the actual layout trigger, whichever the reproduction
   proves. Retain a real widget test whose sequence holds reserve out, enables its flag,
   requests navigation, releases the hold, and proves the anchor is reached without a
   filler widget. Control callback ordering or await the specific ready state; do not
   wait for arbitrary extra time. Ensure the regression still fails if the production
   remeasurement trigger is temporarily removed, then restore that probe.
5. Verify the entire anchor-ownership module and nearby navigation coverage, including
   manual scroll, superseding navigation, hidden paged restores, and late extent growth.
   Repeat the formerly failing conditions sufficiently to exercise the race, reporting
   counts rather than claiming certainty from one pass. Run targeted Files snapshots if
   production rendering/layout changed, then the standard check.

Acceptance: the initial setup reliably reaches its controlled state, and the real
remeasurement behavior remains covered without a larger timeout or weakened assertion.
Record the landed SHA and verification commands for the integration worker.

## Phase: reply-preservation

1. Read `sase-1ln` and `tests/ace/tui/test_main_deck_reply_steal_recovery.py`. Follow
   `show_main_document()` in `widgets/decks/panel_navigation.py`, `_on_main_scroll_y()`
   in `panel_interaction.py`, and the Main view's document/layout callbacks. Compare the
   CI image for `agents_renamed_generic_session_root_120x40` against the committed
   golden and a local capture.
2. Extend/rewrite the regression to exercise an actual same-subject document re-delivery
   while parked on Reply. The old test manually assigns `scroll_y`; that alone cannot
   distinguish a user scroll from a layout clamp. Reproduce the delivery and ordering
   through the production path, then assert selected card and reader position survive
   all delayed callbacks without calling the reselecting helper.
3. Preserve the reader's anchor across that delivery and prevent its temporary layout
   clamp from becoming an intentional card selection. Bound any guard to the specific
   delivery/navigation generation, release it when layout settles, and invalidate it for
   a new subject or explicit navigation. Do not globally ignore scroll watchers.
   Preserve intentional mouse/wheel scroll, card navigation, follow-newest behavior,
   paged/spread transitions, and subject switching. Keep the event loop free of I/O.
4. Verify those interactions in focused regressions, including a second delivery and
   navigation superseding a pending restore. Remove the readiness helper's Reply
   re-drive only when its users no longer need it; otherwise document the remaining
   reason and keep the new regression independent of that workaround.
5. Run the renamed-session visual node and affected session/deck-block selectors,
   including host-independence checks, inspect any legitimate golden difference, and run
   the standard check. A current same-subject delivery must not reproduce the stolen
   selection; don't fix the screenshot by waiting until Context wins.

Acceptance: a parked Reply reader remains parked through real background re-delivery,
explicit navigation still works, and the visual case is deterministic. If investigation
shows an additional unbounded product redesign is needed, retain the parent plan's
bounded fallback: stabilize only a demonstrated correct final frame, record exact
remaining product behavior against `sase-1ln`, and expose that limitation to
integration.

## Phase: visual-fixtures

This worker owns only these two visual tests and their necessary fixture/UI seams:

- `tests/ace/tui/visual/test_ace_png_snapshots_agents_stranded.py::test_agents_stranded_resolve_modal_png_snapshot`
- `tests/ace/tui/visual/test_ace_png_snapshots_artifacts_beads_empty.py::test_artifacts_beads_empty_png_snapshot`

For each, compare the first-attempt CI actual, expected image, and a local capture.
Retain the changed region and relevant state as evidence. Reproduce with controlled
callback ordering or the specific CI environment difference before choosing a fix.

For the stranded modal, inspect focus and background Agents updates after the existing
complete-map and fitted-title waits. The fixture calls `_refresh_agents_display` and
later disables `_refresh_agents_display_debounced`, but a previously queued delivery or
resize may still alter the covered screen. Settle the actual asynchronous state,
drain/invalidate the relevant delivery, or fix focus ownership if production is wrong.
Keep the modal interaction and three-member jump map represented in the screenshot.

For empty Beads, identify the inbox-label difference rather than assuming a font bug.
Check the actual text/glyph and notification/inbox data supplied by startup fixtures.
Pin the count/status source if it leaks real state, or wait for its explicit final
publication if it races. Modify fonts/renderer only if evidence proves a renderer
defect; never add pixel tolerance or mask the region.

Verify each selector independently, then together and in UTC with no unstable nodes. Use
a few repeated targeted captures to expose the identified ordering. Run existing
modal/Beads nonvisual tests for any production change, and `sase tool run check`. If the
intended final image is unchanged, retain its golden. If the intended output changes,
inspect and update only the affected golden after the seam is deterministic.

Acceptance: both CI-only differences have identified causes and targeted exact-match
evidence, with no changes to the bulk heading goldens owned by `sase-1r2.6`.

## Phase: verify-full-ci

1. Read all child notes, overlapping phase notes, and current workflow state. Fetch
   `origin/master` within this checkout and verify ancestry for every repair SHA; a
   declaration or close note alone does not prove a commit landed. Establish where the
   109-golden repair actually landed before running an integrated check. If the
   stale-owner takeover condition applies, recover/reimplement its necessary remainder
   using audited artifact/repository access, inspect it, and land through host
   completion.
2. Verify the focused Files and Main regressions plus the three visual selectors on the
   combined tree. Run `sase tool run check` for integration edits. Inspect all generated
   golden changes before declaring them. Close independent child phases on their
   verified scope even when an identical clean-base failure exists: record that evidence
   as `PROPOSED FOLLOW-UP:` and carry in-scope CI failures into this phase.
3. Reuse a Full CI run only if its SHA contains all relevant fixes. Otherwise dispatch
   `gh workflow run full.yml --repo sase-org/sase --ref master` after host completion
   has landed them. Correlate the new run by workflow, event, creation time, and SHA; do
   not accidentally watch an older scheduled run. The workflow does not publish a
   release and its concurrency group does not cancel an active run.
4. Use `/sase_monitor` at weight zero with `WATCHING CI` / `WATCHED CI` labels and an
   explicit follow-up to inspect the selected run. Stage the monitor, wait for the
   staging command to exit, then use `/sase_final` with `keep` for unfinished work.
   Never leave a foreground watch running at turn end or poll the staged successor.
5. On failure, retain job logs and visual reports. Fix the remaining in-scope cause,
   land, and dispatch a fresh first attempt on the containing SHA. Do not treat a green
   rerun as first-attempt evidence. A qualifying run from `sase-1r2.6` counts; avoid
   duplicate dispatches. Quick unrelated regressions may be fixed; larger new unrelated
   failures become explicit follow-up evidence under the parent scope rule.
6. On success, publish `ci_evidence` through `/sase_var`: `repo`, `default_branch`,
   `tip_sha`, `runs` with workflow/run ID/URL/head
   SHA/event/attempt/conclusion/completion time and job outcomes, `fixes` with landed
   SHAs, `quarantined`, and `notes`. Record any initial visual mismatches handled by the
   workflow's built-in recapture separately from GitHub's run attempt. Include a command
   to reproduce the result:
   `gh run list --repo sase-org/sase --workflow full.yml --branch master -L 5`.

Acceptance is a completed, successful **attempt 1** on a SHA containing all fixes. The
ten executing jobs must succeed: `prepare`, `build-core`, `lint`, `perf-floors`, the
three Python test legs, `visual-test`, `ace-page-group-isolation`, and `tutorial-tests`.
Only `docs-build` and `release-core-floor-smoke` may be skipped as designed for this
master run. Validate the current workflow if names have changed; no upstream-failure
skips or cancelled jobs count as green. A newer red in-scope run must be addressed or
explicitly resolved under the inherited exceptional fallback.

## Land agent and closeout

Only this child epic's land agent may finish parent phase `sase-1rq.2.1.5`. Workers
close only their own assigned child phases and run `sase bead epic-symbols <id>` before
doing so, resolving symbols or re-keying them to a still-open owner. No worker closes
the parent phase or any ancestor, including from instructions quoted in another plan.

The land agent confirms all four phases are closed, refreshes the Full CI evidence,
checks fix ancestry and the latest in-scope run, and republishes `ci_evidence` for the
outer verifier. It consolidates `PROPOSED FOLLOW-UP:` entries onto `sase-1rq.2.1.5` for
the outer epic land agent to triage; it does not create task beads or close `sase-1qd`,
`sase-1ln`, or `sase-1r2.6`. For truly exhausted repair attempts, only the inherited
research-report fallback applies, after substantial hypotheses tested against real CI;
do not claim green or completion merely because a monitor failed or a turn ended.

Before closing `sase-1rq.2.1.5`, run `sase bead epic-symbols sase-1rq.2.1.5` and resolve
or re-key every leftover. Planning found none, but recheck at completion. Close only
that parent phase with
`sase bead close sase-1rq.2.1.5 --note "<verified SHA, run URL, attempt, job results, evidence or authorized report reference>"`.
Do not close `sase-1rq.2.1`, `sase-1rq.2`, `sase-1rq`, or any part of `sase-1r2`.
