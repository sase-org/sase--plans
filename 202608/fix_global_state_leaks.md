---
tier: tale
title: Finish the sase-j7.4 global-state leak phase
goal:
  Genuine cross-test state poisoning is fixed and prevented by a precise blocking
  full-suite gate, with every residual named flake deterministically investigated.
size: medium
proposed_by: bbugyi200.athena.sase-j7.4
bead: sase-j7.4
create_time: 2026-08-10 17:34:00
status: wip
---

- **PARENT:**
  [202608/fix_sase_ct_flake_class.md](https://github.com/sase-org/sase--plans/blob/main/202608/fix_sase_ct_flake_class.md)
- **BEAD:**
  [sase-j7.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-j7/sase-j7.4.md)

# Finish the `sase-j7.4` global-state leak phase

## Objective

Turn the reporting-only global-state detector from `sase-j7.2` into a precise blocking
gate, fix every genuine cross-test poisoning mechanism it identifies, and investigate
the six residual bead/snooze/plan-approval flakes with deterministic same-process
ordering evidence. Preserve the epic's retirement boundary: do not edit
`tests/reproducible_flake_baseline.txt`, do not close the parent epic, and record any
out-of-scope follow-up only as a `PROPOSED FOLLOW-UP:` note on `sase-j7.4`.

## Evidence and constraints

- The durable `sase-j7.2` inventory reports 4,934 poisoning changes across 2,118 tests,
  but its dominant records are teardown normalization rather than state capable of
  poisoning a successor: 2,130 config globals change from a populated cache to `None`,
  hundreds of `functools` caches are cleared, and the named `sase-j6` node only clears
  linked-repository resolution caches. The snooze-gate victim similarly leaves the
  notification load cache empty. These cold/reset states rebuild from current inputs and
  must be reported separately from stale non-cold substitutions.
- The current detector sorts records by node ID and omits worker/order provenance, so it
  cannot identify the predecessor that poisoned a failing test under work-stealing.
- The detector's options are registered through `tests/conftest.py`; the prior phase
  verified that an invocation without an explicit test path cannot reliably enable the
  option. The blocking lane therefore needs an early-loadable pytest plugin path.
- The confirmed VCS metadata leak was already fixed by `c0520947d`; retain its
  production invalidation entry point, helper teardown, and autouse snapshot/restore
  backstop as the model for any newly confirmed leaks.
- A pass in isolation is not evidence. A residual node is fixed only when a command that
  runs its poisoner first and victim second in one process fails before the fix and
  passes after it. If honest investigation cannot reproduce a node, record exactly what
  was attempted and leave it for the retirement phase rather than guessing.

## Implementation

1. Make `tests/_global_state_leak_detector.py` a self-contained, early-loadable pytest
   plugin. Keep report-only mode for diagnostics, add an explicit fail-on-poison mode,
   fail closed on report/worker aggregation errors when blocking is requested, and
   retain deterministic JSON output. Include worker ID and per-worker execution order in
   each record so a full-lane failure can be paired with its immediate predecessors.
2. Refine state classification around whether the successor can observe stale data:
   additions from a cold state remain warming; resets to canonical cold/empty state and
   opaque cache clears become a separately counted cooling/invalidation class; changes
   between two distinct non-cold values, non-prefix collection rewrites, leaked
   environment keys, `sys.path` changes, and working-directory changes remain blocking
   poisoning. Report enough sanitized delta metadata to identify changed environment
   keys without serializing secret values. Add focused unit tests for every transition,
   aggregation, worker ordering, error path, and blocking exit-status behavior.
3. Load the plugin before normal conftest discovery from `tools/run_pytest` for the
   exhaustive cost lane, remove duplicate option/plugin registration from
   `tests/conftest.py`, and pass both detection and blocking switches in that lane.
   Because `just check-full` and the Python 3.13 CI leg both run `just test-cost`, this
   makes one existing full-suite execution carry the gate without adding a duplicate
   suite run. Add command-construction tests proving the plugin and flags are present
   only in the intended exhaustive lane and work when no explicit test path is given.
4. Regenerate the inventory with the corrected classifier. Group remaining poison by
   state name/reason, inspect every group, and fix each mechanism at its production
   invalidation boundary plus the responsible test helper teardown. Use snapshot/restore
   only as a backstop when the state is not reconstructible through a public reset API.
   Add a regression that fails on the parent commit for every confirmed leak, then run
   the detector again until the blocking report contains zero genuine poisoning changes.
   Do not suppress node IDs or add broad allowlists to obtain a green report.
5. Investigate these nodes explicitly:
   - `tests/test_bead/test_snooze_gate.py::test_bead_snooze_gate_preview_carries_the_real_snooze_note`
   - `tests/test_bead/test_snooze_lifecycle.py::test_cancel_snooze_returns_the_bead_to_ready`
   - `tests/test_bead/test_snooze_lifecycle.py::test_plus_one_target_wakes_the_bead_with_the_preset_note`
   - `tests/test_bead/test_snooze_lifecycle.py::test_snooze_round_trips_through_every_persistence_surface`
   - `tests/test_bead/test_plus_one_presentation.py::test_post_close_plus_one_badge_marker_search_and_json_agree`
   - `tests/test_plan_approval_actions.py::test_headless_epic_approval_submits_while_inflight_launch_holds_anchor`

   Use full-lane worker/order records, targeted randomized serial runs, and
   delta-debugged predecessor sets to derive the smallest poisoner-before-victim
   command. Fix genuine global-state leaks by mechanism. If a failure is instead an
   assertion timeout or another non-global defect, fix it on those terms and add a
   deterministic regression. For any node still unreproducible after the corrected
   full-lane and targeted searches, append a bead note listing the exact commands,
   repeat counts, and observations; do not alter its flake-baseline entry in this phase.

## Validation

1. Run `just install` before verification because this is an ephemeral workspace.
2. Run focused detector/plugin/runner tests and every new poisoner-before-victim
   regression. For each reproduced residual node, demonstrate the minimized ordering
   fails on the pre-fix commit and passes on the implemented tree.
3. Run the corrected detector over focused calibration cases and then the full nonvisual
   suite in blocking mode; require a zero-poison report while reporting warming and
   cooling counts separately.
4. Run `just check`. Because the detector and root conftest are selection-broadening
   files, expect or manually perform exhaustive verification.
5. Run `just check-full` and confirm its `test-cost` execution visibly ran the blocking
   detector. Confirm the CI workflow still exercises that same recipe.
6. Re-run the full blocking detector after all fixes to guard against changes made
   during residual-flake work, inspect `git diff` for forbidden baseline changes, add
   any required `PROPOSED FOLLOW-UP:` note to `sase-j7.4`, and close only `sase-j7.4`
   with a note naming the zero-poison gate, deterministic reproductions or honest
   unreproduced-node record, and `just check-full` result.
