---
tier: tale
title: Finish sase-126 integration
goal:
  The current Python tree uses one complete published core release and the epic's
  selection-health dispositions are reconciled without duplicate debt.
size: medium
proposed_by: bbugyi200.athena.sase-126.land
bead: sase-126
create_time: 2026-09-18 06:43:50
status: wip
---

- **PARENT:**
  [202609/restore_actions_ci.md](https://github.com/sase-org/sase--plans/blob/main/202609/restore_actions_ci.md)
- **BEAD:**
  [sase-126](https://github.com/sase-org/sase--beads/blob/main/pages/sase-126/README.md)

# Plan: Finish sase-126 integration

## Objective

Complete only the integration work uncovered while landing `sase-126`: restore one
complete, jointly verified `sase-core-rs` source/published floor after concurrent gate,
service, and machine-setup work advanced the Rust contract, and reconcile the duplicate
selection-health baseline stanza introduced by the epic's integrated-verification
commit.

The parent epic's close, epic-symbol retirement, post-close Symvision run, and linked
plan `status: done` update are deliberately outside this child plan. Those remain the
resumed `sase-126` land agent's duties.

## Current evidence

- Phase `sase-126.1` aligned the source pin and dependency floor at complete release
  `0.34.47`, but concurrent commit `df0090f040` later advanced this repository to
  `sase-core-rs>=0.34.48,<0.35.0` and source release `0.34.50` for gate-decision
  contracts. Current `python tools/ratchet_core_window --check` exits 3 because PyPI
  `0.34.48` is incomplete: it has neither an sdist nor a Windows wheel.
- The linked core repository now contains release `0.34.51` at
  `f7bf8f00f834385e16839a5877efc63e1e1abfb4`, after `0.34.49` added the service-status
  fix, `0.34.50` added the gate-decision contract, and `0.34.51` added the machine-setup
  fix. At planning time PyPI had not yet published a complete release newer than
  `0.34.47`; do not assume release publication merely because the core release commit
  exists.
- Phase `sase-126.4` appended a 22-node `sase-126.4` block to
  `tests/reproducible_flake_baseline.txt` after concurrent baseline hygiene had already
  dispositioned the same nodes. Thirteen already have live owner entries for ready flake
  tasks `sase-12a` through `sase-12k`, plus `sase-120` and `sase-121`. Six are already
  retired by existing `fixed-at` entries (`sase-10s`'s two dismissed-save audits, the
  hold-created-at assertion, `sase-11z`, `sase-10p`, and `sase-10v`). The remaining
  three gate nodes were already routed as a discovered issue to active epic
  `sase-zr.7.1.1.5` and were fixed by `df0090f040` when their fixtures gained nonempty
  attempt IDs. The trailing `sase-126.4` block therefore duplicates 19 existing
  dispositions and turns three fixed gate-contract records into permanent live debt.
- The late `sase-126` screenshot note is already integrated by phase `sase-126.4`:
  commit `eaa1cbf4de` updated the Agents metadata-view helper and regenerated
  `agents_list_120x40.png` after the detail picker landed. Preserve that integration.

## Implementation

1. Recheck PyPI through `tools/ratchet_core_window` and the linked core release history.
   Wait for a complete non-yanked published release that contains every binding and
   behavior required by the current Python tree. Do not ratchet to incomplete `0.34.48`,
   mix a newer source pin with an older minimum, or weaken the completeness checks. If
   the newest complete release is newer than `0.34.51`, inspect its core changes and use
   the single newest release that is valid for both source and package floor.
2. Update `sase-core-revision.txt`, the `pyproject.toml` requirement, and only the
   necessary `uv.lock` package/requirement records so the source pin and exact published
   minimum name that same complete release. Use the existing ratchet workflow; preserve
   the `<0.35.0` compatibility ceiling and source-built/published provenance checks.
3. Reconcile `tests/reproducible_flake_baseline.txt` by deleting the duplicated trailing
   `sase-126.4` stanza. Keep the earlier task-owned entries and existing `fixed-at`
   retirements. Replace the three gate-node raw duplicates with one documented
   `fixed-at: 2026-09-18T02:21:29Z` line each, naming active owner epic
   `sase-zr.7.1.1.5` and fix commit `df0090f040`, unless newer post-fix evidence proves
   one is still live. The nodes are:
   - `tests/test_gate_cli_show.py::test_show_reports_an_accepted_failed_gate`
   - `tests/test_gate_decision_acceptance.py::test_cancel_is_permitted_after_current_failure`
   - `tests/test_gate_decision_acceptance.py::test_conflicting_selection_supersedes_after_current_failure`

4. Run focused gate tests for those three nodes and
   `just selection-health --fail-on-new-flake`. If the gate reports post-fix evidence,
   inspect the exact records and route it to the active gate epic instead of restoring a
   duplicate blanket baseline entry.
5. Validate the selected source build and exact published minimum independently. Run
   `tools/check_sase_core_rs_bindings`, `tools/validate_sase_core_rs`, the release-floor
   smoke/contract procedure used by CI, and focused current service-status,
   gate-decision/failure, and machine-setup contract tests. Confirm
   `python tools/ratchet_core_window --check` exits 0 on the finished tree.
6. Read `lint_and_test.md` through the audited memory workflow, run `just fix`, inspect
   its diff, then run `just check`. Run `just test-visual` to ensure the already-fixed
   Agents golden remains integrated and `just phase7-perf-check` to preserve the epic's
   performance acceptance. Use the SASE monitor workflow for long commands. Do not
   weaken thresholds, snapshot tolerances, required bindings, or release completeness
   rules to obtain green results.

## Acceptance criteria

- `sase-core-revision.txt`, `pyproject.toml`, and `uv.lock` agree on one complete
  published core release containing all bindings required by the current Python tree;
  the ratchet check exits 0 and both source-built and exact-minimum validation pass.
- The duplicate `sase-126.4` flake-baseline block is gone. Every one of its 22 nodes
  retains exactly one deliberate disposition: 13 task-owned live entries, 6 existing
  fixed retirements, and 3 gate-epic fixed retirements.
- Focused gate tests, selection health, `just check`, the visual suite, and Phase 7
  performance checks pass without weakening contracts.
- No parent epic close, epic-symbol cleanup, or parent plan-status edit is performed in
  this child plan.
