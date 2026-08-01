---
tier: tale
title: Finish and land epic sase-9y
goal: Correct the stale contention evidence, validate the integrated CI fixes, and
  close out epic sase-9y in the required order.
bead: sase-9y
create_time: 2026-07-27 11:55:48
status: done
---

- **PROMPT:** [prompts/202607/land_sase_9y.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/land_sase_9y.md)
- **PARENT:** [202607/fix_ci_bead_isolation_and_visual_flakes.md](https://github.com/sase-org/sase--plans/blob/main/202607/fix_ci_bead_isolation_and_visual_flakes.md)
- **BEAD:** [sase-9y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9y/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9y.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9y.land/README.md)
  - [bbugyi200.athena.sase-9y.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9y.land.md#member-code)
- **COMMITS:**
  - [a947469](https://github.com/sase-org/sase/commit/a947469eece2988bdfff48bd6ee40b5a9701172f) — docs: record final visual contention baseline (sase-9y)

# Finish and land epic `sase-9y`

## Goal

Finish the land-agent audit for `sase-9y` by correcting the one stale source comment found during integration,
refreshing local and real-CI evidence against the current `master`, and then performing the epic's required closeout in
the prescribed order.

The epic fixed two independent CI defects:

- Three bead-work CLI tests were missing the `project_dir` isolation fixture.
- ACE PNG snapshots could accept a frame as converged under CPU starvation and then rasterize a different frame.

The implementation commits are `3e0dbc723` (`sase-9y.1`), `a0636fcbb` (`sase-9y.2`), and `57e3acb3a` (`sase-9y.3`). All
four child beads are closed. The audit already established that the source implements the requested fixture isolation,
scheduler-progress convergence, deterministic cursor handling, and an exact converged-frame assertion at synchronous
capture time.

## Constraints

- Do not regenerate PNG goldens, relax PNG comparison tolerances, or weaken the pytest state-write guard.
- Do not edit SASE memory or generated instruction files. The epic plan notes a documentation discrepancy about CI PNG
  tolerance, but the user has not requested a memory edit.
- Treat failures introduced by the later `sase-9z` plan-reference work as separate unless evidence shows they interact
  with `sase-9y`. Current known examples are the published-core 0.11.2 binding check and CI's inability to resolve
  `sase-9z(...)` Symvision exemptions from its plans checkout.
- Use `sase repo open plans` before reading or editing the plans sidecar. Refer to the epic plan as
  `plans:202607/fix_ci_bead_isolation_and_visual_flakes.md`.
- Run `just install` before repository checks in a fresh workspace, and run `just check` after changing the `Justfile`.

## Implementation

1. Refresh the contention-harness evidence in `Justfile`.
   - Preserve the useful pre-fix result: 116 failed, 246 passed, 1 skipped.
   - Preserve the convergence-only result from `sase-9y.2`: 15 failed, 347 passed, 1 skipped, with no convergence
     timeout.
   - Replace the stale statement that those failures are still assigned to the "next epic phase" with the final
     `sase-9y.3` result: 363 passed, 1 skipped in 9m37s under 26 workers pinned to two CPUs.
   - Make clear that the final result retained exact PNG equality and did not regenerate goldens.

2. Revalidate the integrated tree.
   - Run the three bead-isolation regression cases: `test_work_missing_bead_json_error_is_one_envelope`,
     `test_bead_id_mode_rejects_parent_override`, and
     `test_bead_id_mode_rejects_plan_file_only_linking_options_as_json`.
   - Run `tests/ace/tui/visual/test_visual_idle.py` to cover scheduler-progress convergence and exact-frame rejection.
   - Run `just test-visual` so the post-epic AXE PNG test split is exercised through the shared convergence/capture
     guard. The land-agent audit's baseline before this plan was 363 passed, 1 skipped.
   - Run `just check`. Fix any regression caused by this epic. If a check fails only because of independently landed
     work, preserve the exact diagnostic and keep it out of this epic.

3. Refresh real-CI evidence on current `master`.
   - Inspect GitHub Actions run `30280994759` (or the newest replacement run if it is superseded).
   - Confirm `bead-backend` passes; it already passed on run `30280994759`.
   - Confirm `visual-test` passes after the later AXE PNG split and remains under its 45-minute budget. If it fails,
     inspect artifacts/logs and repair any `sase-9y` regression before continuing.
   - Preserve the prior durability evidence from phase `sase-9y.4`: green `visual-test` jobs on master run `30274179282`
     (27m21s) and release-PR run `30274392371` (24m05s), plus green bead-backend jobs on both.
   - Record, but do not absorb, unrelated failures from later commits. At audit time the current run was red only in
     `published-core-minimum-smoke` for five `sase-9z` plan-reference bindings and in `lint` for four unresolved
     `sase-9z(...)` Symvision exemptions.

4. Land the epic last.
   - Close the parent with `sase bead close sase-9y`, then verify all four children and the parent are closed.
   - Only after closing, check whether `just symvision` is available and run it if so. Before fixing any Symvision
     diagnostic, follow the repository's required `sase_memory_read` workflow for `sase/memory/symvision.md`.
   - Remove any expired `sase-9y` epic-symbol whitelist entries and any unused code that Symvision attributes to this
     epic. Do not remove active exemptions belonging to another open epic.
   - If post-close Symvision cleanup changes repository files, rerun the relevant focused tests and `just check`.
   - Open the plans sidecar through `sase repo open plans`, then change only the epic plan frontmatter from
     `status: wip` to `status: done` in `202607/fix_ci_bead_isolation_and_visual_flakes.md`.
   - Finish by showing `sase-9y`, confirming the plan frontmatter is `done`, and reporting source and plans-sidecar
     worktree status.
