---
tier: epic
title: Finish the sase-jx.5 landing audit and closeout
goal: The last stale selected-run contract text is corrected, the combined AXE chop-overrun
  feature is verified against every child note and intervening commit, follow-ups
  are dispositioned accurately, and epic sase-jx.5 is closed without force with its
  linked plan marked done.
phases:
- id: align_selected_run_contract
  title: Align the chop-detail API documentation with selected-run rendering
  depends_on: []
  size: xsmall
  description: 'align_selected_run_contract: update the stale update_chop_display
    argument documentation that still says the header only describes the newest sampled
    run and run_idx zero; document the actual schema-v2 behavior in which run_ratios
    is aligned to raw history and the displayed run index controls the overrun segment,
    then run the focused status-section tests and the repository check required for
    a source change.'
- id: close_epic
  title: Complete verification, integration, follow-up disposition, and closeout
  depends_on:
  - align_selected_run_contract
  size: medium
  description: 'close_epic: re-audit the final combined tree and any newly landed
    base commits, run the full cross-repository verification lane, record the obsolete
    sase-js follow-up as already fixed by c30bcb012 rather than falsely corroborating
    it, close phase sase-jx.5.4 and epic sase-jx.5 without force, run post-close Symvision
    cleanup, and set status done in the linked land_axe_chop_overrun plan.'
proposed_by: bbugyi200.athena.sase-jx.5.land
parent_bead: sase-jx.5
create_time: 2026-08-12 14:02:19
status: done
bead_id: sase-jx.5.5
---

- **PROMPT:** [prompts/202608/finish_jx5_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_jx5_landing.md)
- **PARENT:** [202608/land_axe_chop_overrun.md](https://github.com/sase-org/sase--plans/blob/main/202608/land_axe_chop_overrun.md)
- **BEAD:** [sase-jx.5.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-jx/sase-jx.5.5.md)

# Plan: Finish the sase-jx.5 landing audit and closeout

## Verified starting point

The landing audit has already read `sase-jx.5`, all four children, every note and full
history event, the linked `plans:202608/land_axe_chop_overrun.md` plan, the original
feature plan, and the implementation commits in both repositories:

- `sase-core` `46ce1fe9` validates every otherwise-sampleable run timestamp, adds
  schema-v2 `run_ratios` aligned to raw request order, and updates Rust/PyO3 tests.
- `sase` `888453d3` consumes schema v2, validates aligned ratios, keys the detail mark
  to the selected raw run, repaints cached overview data across first layout and resize,
  and removes the narrow-PNG manual refresh workaround.
- `sase` `688eec2b` ratchets `sase-core-rs` from `>=0.24.0,<0.25.0` to
  `>=0.26.4,<0.27.0` in both `pyproject.toml` and `uv.lock`.

The actual Rust classifier, wire records, PyO3 binding, Python facade, collector,
dashboard render/status/output widgets, focused tests, and dependency files were read.
The runtime and test behavior reported by phases `sase-jx.5.1` through `.3` is present.

The audit also reviewed every primary-repository commit since the first cross-repo
landing commit at 2026-08-12 12:29 EDT and every core commit after `46ce1fe9`.
Intervening AXE/ACE changes for task-gate sweeping, startup telemetry, shared cached
Patch runner counts, visible-tab stopwatch completion, and cached Tier-1 artifact-index
loads are orthogonal or already compose through the same collector/refresh paths; no
duplicate classifier or render path remains. The 0.17.0 release includes the ratcheted
core floor. Re-fetch and repeat this audit for any commit that lands before close.

The sole `PROPOSED FOLLOW-UP:` entry on `sase-jx.5.2` asked to remove five stale
`sase-js` Symvision exemptions. It is an exact match for task `sase-kc`, but the current
tree proves the proposal is obsolete: post-start commit `c30bcb012` already removed all
five exemptions, deleted the truly unused symbols, and made the real uses visible.
`just symvision` passes on current master, and a concurrent landing agent closed
`sase-kc` as done with the same commit evidence during this audit. Do not add a
reproduction `+1` unless the defect actually recurs. Record the already-completed
disposition in the close note, including the proposing bead.

One epic-caused documentation defect remains. In
`src/sase/ace/tui/widgets/_axe_dashboard_status.py`, `update_chop_display()` correctly
indexes `overrun.run_ratios` by the displayed `run_idx`, but the `overrun` argument text
still claims the header only describes the newest sampled run and renders only at index
zero.

## Phase 1: Align the chop-detail API documentation with selected-run rendering

1. Update only the stale `overrun` argument documentation in
   `src/sase/ace/tui/widgets/_axe_dashboard_status.py`. State that the verdict carries
   ratios aligned to raw run history and that the header selects the displayed run's
   ratio by `run_idx`. Preserve the implemented degradation for missing or out-of-range
   ratio data.
2. Re-scan AXE docs and source for another newest-only claim about the detail header.
   Keep legitimate `latest_ratio` descriptions for the overview PACE column and
   window-level summary unchanged.
3. Run the focused status-section tests, then `just check` because this phase changes a
   primary-repository source file. Confirm the existing skipped-newest and older-overrun
   paging cases remain green. Do not change runtime behavior or snapshots.

## Phase 2: Complete verification, integration, follow-up disposition, and closeout

1. Refresh the primary and linked core repositories and inspect every newly landed
   commit since the audit boundary. Integrate any change that should now consume the
   schema-v2 classifier or conflicts with collection/rendering. Review the final diff,
   working-tree state, linked-plan link, and `sase bead show` output for `sase-jx.5` and
   all children.
2. Verify `sase-core` with `just check`. In `sase`, run `just install`, the focused
   facade/collector/dashboard/status tests, both chop-overrun PNG nodes, the published
   core floor/version/probe gates, `just check-full`, and `just test-visual -k axe`. The
   visual subset may fail only on the 11 unrelated nodes already recorded on `sase-dl`;
   compare exact signatures and do not accept unrelated goldens. Any new or
   feature-related failure remains epic work.
3. Perform practical live `sase ace` checks where the terminal environment permits:
   sidebar and collapsed-parent chips, overview PACE/advisory, selected-run detail
   paging, compact first paint and resize response, a fast agent-launching chop without
   a false mark, and the Guide legend. Record environmental limitations honestly.
4. Prepare a comprehensive close note covering phase-by-phase source evidence, command
   results, dependency publication/floor verification, every post-start integration
   commit, the selected-run documentation correction, and follow-up disposition. State
   that the proposal from `sase-jx.5.2` was already completed by `c30bcb012`, verified
   by passing current Symvision, and closed on duplicate task `sase-kc` without a false
   reproduction `+1`. Preserve the original landing-plan disposition of unrelated PNG
   drift as `sase-dl` with artifact `file:explicit:8668b25f99aed578e9b544a7`.
5. Close `sase-jx.5.4` with its verification note, then close `sase-jx.5` using
   `sase bead close sase-jx.5 --note "<verified close note>"`. Do not use `--force`. If
   closing names another unfinished descendant, finish or deliberately reopen it; forced
   canceled/superseded resolution is only for an actual lifecycle decision.
6. Only after the close, read `sase/memory/symvision.md` through `/sase_memory_read`,
   run `just symvision`, remove any entries made stale specifically by closing
   `sase-jx.5` and any unused code it exposes, and rerun the relevant checks. Do not
   absorb unrelated ready task work.
7. Open the plans sidecar through `/sase_repo`, change only the linked epic plan
   `202608/land_axe_chop_overrun.md` frontmatter from `status: wip` to `status: done`,
   and verify the sidecar is clean apart from that intentional durable-plan update.

## Non-goals

- Do not close parent epic `sase-jx`; this landing request names epic `sase-jx.5`.
- Do not accept or rebaseline the 11 unrelated AXE-editor PNG mismatches tracked by
  `sase-dl`.
- Do not add new AXE surfaces, scheduling policy, or render-time classification.
