---
tier: tale
title: Re-baseline the two stale AXE layout goldens and land epic sase-jx.5.5
goal:
  The two AXE layout PNG goldens that epic commit 888453d39 left recording the pre-fix
  wide overview table are re-baselined to the corrected compact rendering, the epic's
  follow-up proposals are dispositioned against the existing ready tasks that already
  cover them, and epic sase-jx.5.5 and its parent sase-jx.5 are closed without force
  with both linked plans marked done.
size: medium
proposed_by: bbugyi200.athena.sase-jx.5.5.land
bead: sase-jx.5.5
create_time: 2026-08-12 15:29:10
status: wip
---

- **PARENT:**
  [202608/finish_jx5_landing.md](https://github.com/sase-org/sase--plans/blob/main/202608/finish_jx5_landing.md)
- **BEAD:**
  [sase-jx.5.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jx/sase-jx.5.5.md)

# Plan: Re-baseline the two stale AXE layout goldens and land epic sase-jx.5.5

## Why this plan exists

The landing audit for epic `sase-jx.5.5` verified every child note and every intervening
commit and found the epic's work complete except for one defect the epic itself caused:
two committed AXE PNG goldens still record the pre-fix wide-mode overview table and no
longer match what the fixed code renders.

Phase `sase-jx.5.5.2` recorded these two failures as an unrelated `PROPOSED FOLLOW-UP:`.
That disposition is wrong. They are epic work, and this plan finishes them and then
lands the epic.

## Verified starting point

Verified at `master` = `origin/master` = `b4c6038e6`, clean working tree.

Feature verification (all re-run independently, not taken from child notes):

- `src/sase/axe/chop_overrun.py` validates the schema-v2 wire contract, rejects a
  `run_ratios` length that is not aligned to the raw run count, and returns `None` only
  for a non-positive interval or empty history.
- Linked core `sase-core` at `d2a418d` (workspace version `0.26.5`) derives
  `level == "over"` from `samples[0].over` and `latest_ratio` from `samples.first()`,
  while `run_ratios` keeps one slot per raw run with `None` for unsampleable runs. That
  is exactly what the Python facade and both render surfaces assume.
- Phase `sase-jx.5.5.1`'s docstring fix (`c18410204`) matches the implementation at
  `src/sase/ace/tui/widgets/_axe_dashboard_status.py:345-364`, which indexes
  `overrun.run_ratios[self._chop_run_idx]`, range-guards the index, and renders at
  `>= 1.0`. The `_pace_cell` docstring in `_axe_dashboard_render.py` is also accurate
  given the core's `level` semantics.
- Phase `sase-jx.5.5.2`'s ratchet (`b4c6038e6`) sets `sase-core-rs>=0.26.5,<0.27.0`.
  `tools/probe_core_floor --json` returns
  `{"declared_floor": "0.26.5", "status": "ok", "exit_code": 0}`.
- `just install` succeeded and installed `sase-core-rs 0.26.5`.
- `just check` passed with exit 0. Its scoped lane escalated to the full suite (rules:
  `contract-set-only`, `core-identity-changed`), so every lint gate and the full
  non-visual test suite are green at `b4c6038e6`.

Integration audit: nothing has landed since the epic's own last commit — local `master`
equals `origin/master` at `b4c6038e6`. The four non-epic commits that landed after the
epic was created (`32ccc9eb7`, `14fcbc21a`, `1f388edee`, `67d846327`) touch
external-PR-mirror batching, the ACE diff-badge loader, duplicate lumberjack chop config
entries, and mirrored bead status sync. None of them reads the overrun verdict,
classifies chop timing, or renders the AXE overview/detail surfaces, so there is nothing
to rewire and no duplicate classifier or render path.

## The defect

Both goldens were re-baselined by epic commit `d4c4efda5` (phase `sase-jx.4`), which
predates `888453d39` (phase `sase-jx.5.2`). `888453d39` is the commit that cached
lumberjack overview data for width-only repaints so the dashboard picks the compact
layout after Textual layout settles. It updated the fixtures and removed a
manual-refresh workaround from `tests/ace/tui/visual/test_ace_png_snapshots_axe.py` but
never re-baselined these two goldens, so they still record the pre-fix rendering.

`just test-visual -k axe` on `b4c6038e6` gives 13 failed, 19 passed, 1 skipped:

- 11 failures in `test_ace_png_snapshots_axe_editor.py`, all sharing the exact
  `4758/4173`-pixel signature already recorded on canceled task `sase-dl`.
- 2 failures with entirely different, much larger signatures:
  - `axe_constrained_width_no_wrap_60x30`: 28306/586500 changed, 28303 material
  - `axe_long_label_widened_120x40`: 20350/1520532 changed, 20347 material

The near-equality of changed and material pixels in those two proves a content
difference rather than the antialiasing halo that marks the `sase-dl` drift. Inspecting
`.pytest_cache/sase-visual/` artifacts confirms it directly: in both cases
`expected.png` shows the wide `NAME / LAST RUN / WHEN / DURATION / PACE` overview table
wrapping its own column headers across several lines inside a right panel too narrow to
hold it, while `actual.png` shows the compact stacked lumberjack status block and chop
list. The current rendering is correct and the committed goldens are stale.

`axe_constrained_width_no_wrap_60x30`'s own test docstring says it "proves no-wrap +
ellipsis behavior", which the committed golden visibly contradicts.

## Why re-baselining on this host is sound

A full `just test-visual` on this host reports 209 failed, 447 passed, 1 skipped — the
broad renderer/font drift that canceled task `sase-dl` documents across `sase-bl`,
`sase-c5`, and `sase-c6`. That drift does not reach these two nodes: 19 of the 32 AXE
cases, including both new chop-overrun goldens, match their committed goldens exactly on
this host, and the drift inside the AXE family is confined to the 11 editor nodes.
Regenerating only the two layout goldens here therefore captures the content fix without
importing host drift.

## Steps

1. Re-read this plan's evidence, then re-run
   `just test-visual tests/ace/tui/visual/test_ace_png_snapshots_axe_layout.py` and
   confirm both nodes still fail with the signatures above. Open
   `.pytest_cache/sase-visual/**/actual.png` and `expected.png` for both and confirm the
   compact-versus-wide difference before accepting anything. If the actual rendering is
   not the compact stacked layout, stop and report instead of re-baselining.

2. Re-baseline only those two goldens:

   ```bash
   just test-visual tests/ace/tui/visual/test_ace_png_snapshots_axe_layout.py \
       -- --sase-update-visual-snapshots
   ```

   That file contains exactly the two affected tests. Confirm with `git status --short`
   that the diff touches only
   `tests/ace/tui/visual/snapshots/png/axe_constrained_width_no_wrap_60x30.png` and
   `tests/ace/tui/visual/snapshots/png/axe_long_label_widened_120x40.png`. Revert
   anything else.

3. Re-run `just test-visual -k axe`. Expect 11 failed, 21 passed, 1 skipped, with the
   remaining 11 failures being only the `sase-dl` editor nodes at the unchanged
   `4758/4173` signature. Then run `just check` and require exit 0.

4. Commit through the `/sase_git_commit` skill only. Describe the change as
   re-baselining two AXE layout goldens that `888453d39` left stale, not as a new visual
   change.

5. Disposition the follow-ups from the child beads through `/sase_new_task`, passing the
   proposing bead and the evidence below. Both are duplicates of existing ready tasks,
   so expect corroboration rather than new beads:
   - `sase-jx.5.5.2`'s flake-baseline proposal. Independently reproduced at `b4c6038e6`
     with `tools/selection_health --fail-on-new-flake`: 7 reproducible flakes exceed
     `tests/reproducible_flake_baseline.txt`.
     `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`
     plus the 5 `tests/test_core_vcs_log.py` nodes match ready task `sase-jq` exactly,
     and
     `tests/test_external_mirror_issues.py::test_creation_budget_defers_then_converges_next_pass`
     matches ready task `sase-kd` exactly. None is caused by this epic, whose only
     source change is a docstring.
   - `sase-jx.5.5.2`'s AXE PNG proposal is resolved by this plan for the two layout
     nodes. The remaining 11 editor nodes belong to `sase-dl`, which the owner closed as
     canceled on 2026-08-12 in a deliberate backlog cut. Do not re-file or reopen it;
     the drift still does not block any gate.

6. Close the epic without force:

   ```bash
   sase bead close sase-jx.5.5 --note "<close note>"
   ```

   The close note must record the verification in this plan's "Verified starting point",
   the integration audit, the stale-golden fix, and every follow-up outcome from step 5
   including that phase `sase-jx.5.5.2` mis-dispositioned the two layout goldens as
   unrelated.

7. Only after that close, read `sase/memory/symvision.md` through `/sase_memory_read`,
   run `just symvision`, and remove any entry made stale by closing `sase-jx.5.5` plus
   any code it exposes as unused. A repo-wide search for `sase-jx` currently finds no
   whitelist entry, so expect a clean no-op; report it either way.

8. Set `status: done` in the frontmatter of `plans:202608/finish_jx5_landing.md`,
   opening the plans sidecar through `/sase_repo`.

9. Then finish the parent landing that this epic exists to complete. Epic
   `sase-jx.5.5`'s own goal is that "epic sase-jx.5 is closed without force with its
   linked plan marked done", and phase `sase-jx.5.5.2` recorded that descendant
   validation rejected its attempt only because `sase-jx.5.5` was still open, explicitly
   leaving it "for the land agent". With `sase-jx.5.5` closed, run
   `sase bead close sase-jx.5` without `--force` and set `status: done` in
   `plans:202608/land_axe_chop_overrun.md`. If the close is still rejected, record the
   named descendant and stop; do not force.

## Non-goals

- Do not re-baseline the 11 `sase-dl` AXE editor goldens or any of the other ~198
  broadly drifted goldens outside the AXE family. Only the two named files may change.
- Do not close epic `sase-jx`. The `finish_jx5_landing` plan names that as an explicit
  non-goal, and sibling phase `sase-jx.5.4` owns it.
- Do not change AXE runtime behavior, the classifier, the wire schema, or the
  `sase-core-rs` floor. The rendering is already correct; only the goldens are stale.
- Do not add entries to `tests/reproducible_flake_baseline.txt`.
