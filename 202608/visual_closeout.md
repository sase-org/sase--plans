---
tier: tale
title: Fix visual convergence and clear the PNG backlog
goal:
  Repair diagnosed visual fixture and settling defects, then accept only inspected
  intentional goldens until the serial ACE PNG lane is green.
size: medium
proposed_by: bbugyi200.athena.sase-rm.13
bead: sase-rm.13
create_time: 2026-08-21 06:53:37
status: wip
---

- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.13.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-rm.13](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-rm.13.md)
- **COMMITS:**
  - [72f93fb](https://github.com/sase-org/sase/commit/72f93fb1fb3917c39f1859650b87ac33b6d80847)
    — fix: stabilize visual closeout and verification gates

# Fix visual convergence and clear the PNG backlog

## Goal

Make the ACE PNG snapshot lane trustworthy and green by repairing the four diagnosed
fixture/convergence mechanisms first, preserving the selected-detail split-pane coverage
already integrated from `sase-os`, and accepting only visually inspected, intentional
rendering changes in named snapshot nodes.

## Current evidence and constraints

- Phase `sase-rm.13` owns close-ready implementation evidence for `sase-cx`, `sase-ol`,
  `sase-ph`, `sase-pu`, and finally `sase-r5`; the parent epic's land agent, not this
  phase worker, closes those task beads.
- The selected-detail variants and four corresponding goldens required by `sase-os` are
  already present on the integrated tree. They must be verified alongside the original
  empty-detail variants rather than replaced.
- `sase-cx`, `sase-ol`, and `sase-pu` describe stable-looking transient frames under
  contention. Their semantic predicates must settle the intended content before the
  generic visual-idle check runs.
- `sase-ph` is host-state leakage: every launch-default reader and change token used by
  the indicator must be pinned by the visual fixture so its async refresh worker cannot
  escape to the machine's configured provider/model.
- `sase-r5` contains both stale goldens from intentional UI changes and overlaps with
  the mechanism defects above. Never run the broad `just test-visual-update` recipe;
  update only explicitly named nodes after inspecting actual, expected, diff, and SVG
  failure artifacts.

## Implementation

1. Install the workspace dependencies, reproduce the four mechanism-owned nodes and both
   empty/selected split-pane parameter sets, and run the serial visual lane to obtain a
   fresh failure inventory for the integrated tree.
2. Strengthen the slow-tools test to settle the populated tools footer and active
   section together; strengthen AXE layout snapshots to wait for the populated status
   summary; strengthen the phase-family context snapshot to wait until ARTIFACTS and
   MEMORY are no longer resolving before convergence is accepted.
3. Pin the launch-default change token and every provider/model resolution seam used by
   the indicator in `patch_startup_loaders`, with regression assertions that prove the
   fixture is bound to the runtime call sites.
4. Run the mechanism-owned nodes repeatedly, including the contention recipe where
   applicable, and verify all eight split-mode frames continue to cover both empty and
   selected detail states.
5. For each remaining serial-lane failure, inspect the generated actual/expected/diff
   images and SVG, attribute the rendering change to integrated behavior, and invoke
   `--sase-update-visual-snapshots` only for that exact node. Re-run each updated node
   without update mode before moving on.
6. Add one evidence-rich close-ready block per assigned task to phase bead `sase-rm.13`,
   naming the mechanism or intentional rendering source and the exact verification
   performed. Record any distinct out-of-scope discovery only as a `PROPOSED FOLLOW-UP:`
   phase note.

## Verification and completion

- Run focused non-visual tests for any changed helpers and repeated focused visual tests
  for each mechanism-owned node.
- Run `just check` after all repository changes.
- Run `SASE_PYTEST_WORKERS=1 just test-visual` end to end and require a green result on
  the final tree.
- Confirm `git status` contains no unexplained golden changes.
- Run `sase bead epic-symbols sase-rm.13`; resolve every remaining symbol or re-key its
  Justfile ownership to a still-open bead.
- Close only `sase-rm.13` with a note summarizing the verified mechanism fixes, scoped
  goldens, repository gate, serial visual lane, and symbol audit. Do not close `sase-rm`
  or any assigned task bead.
