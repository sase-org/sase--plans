---
tier: tale
title: Finish and land epic sase-j9
goal:
  Restrict the Agents-tab panel sweep to lanes and clans, integrate concurrent changes,
  and close sase-j9 with post-close cleanup complete.
size: medium
proposed_by: bbugyi200.athena.sase-j9.land
bead: sase-j9
create_time: 2026-08-10 20:18:23
status: wip
---

- **PARENT:**
  [202608/agents_panel_fold_sweep.md](https://github.com/sase-org/sase--plans/blob/main/202608/agents_panel_fold_sweep.md)
- **BEAD:**
  [sase-j9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-j9/README.md)

# Plan: Finish and land epic sase-j9

## Goal

Finish the unresolved landing work for epic `sase-j9`: make the Agents-tab `-` action
collapse and restore only structural agent lanes and clans, never grouping banners such
as `Done` or `Running`; preserve `H` as the precise hinted collapse for all expanded
fold kinds; integrate the post-start keymap changes; verify every child report and
follow-up disposition; then close the epic, run the required post-close Symvision
cleanup, and mark the epic's original plan done.

## Verified starting point

- Epic `sase-j9` has two closed children: `sase-j9.1` and `sase-j9.2`. The original plan
  is `plans:202608/agents_panel_fold_sweep.md` in the `plans` sidecar.
- The epic's user notes supersede Phase 1's original group-banner behavior: `-` must
  collapse only agent clans and agent lanes, never panel grouping banners, and the old
  bottom-up group-collapse fallback can be dropped.
- Epic commits are `62a4ddeb5` (`sase-j9.1`, the panel sweep) and `9608b163e`
  (`sase-j9.2`, hinted `H`). The current implementation still imports
  `expanded_panel_level_zero_group_keys`, stores `group_keys` in `PanelFoldSweepRecord`,
  mutates `panel_fold_registry`, and advertises `-` whenever a group banner alone is
  collapsible. Those paths violate the epic notes.
- Commits after `62a4ddeb5` and outside this epic were reviewed. The only functional
  overlap is the concurrent commits-to-stitches keymap migration. Commit `9c46891c5`
  changed `test_stitches_action_override_wins_over_legacy_commits_alias` from the newly
  claimed `minus` key to unclaimed `f24`; that test now passes at HEAD. Do not revert
  that integration.
- `H`'s collapse-intent hint mode in `_panel_hint_folding.py` is complete and covered by
  `tests/ace/tui/test_agent_panel_hint_collapse.py`. It intentionally continues to hint
  expanded grouping banners as well as lanes/clans. Do not narrow `H` while narrowing
  `-`.
- Follow-up triage is already durable:
  - `sase-ji` is the new ready small task for the unrelated missing `=` → `equals_sign`
    user-override alias proposed by `sase-j9.1`.
  - Exact ready tasks `sase-jb` and `sase-j6` were corroborated for the two unrelated
    reproducible flakes proposed by `sase-j9.1`; the same evidence was routed to active
    flake epic `sase-j7`.
  - `sase-jf` captures the old stitches/minus collision and already records that
    `9c46891c5` fixed it; the targeted test passes at current HEAD.
  - Commit `9edf6807` removed the closed `sase-j3` Symvision whitelist and privatized
    `_SnippetTriggerMatch`, resolving `sase-j9.2`'s stale-whitelist proposal.
  - Phase `sase-j9.1` itself refreshed the specifically stale
    `prompt_stack_g_prefix_hints_120x40.png`; the later `sase-j3` land commit `9edf6807`
    audited and added the other missing planned snippet-save golden.
  - The footer-probe performance proposal was conditional on an observed p95 regression;
    none was reported. Narrowing the probe to lanes/clans removes its grouping-tree
    build and therefore reduces work rather than introducing a new risk.

## Phase 1: Restrict `-` to structural lane and clan folds

1. In `src/sase/ace/tui/models/agent_panels.py`, make `PanelFoldSweepRecord` remember
   only `(fold_key, FoldLevel)` structural entries. Remove `group_keys` and the now-dead
   `GroupKey` type-only import while preserving the exact-level reverse semantics.
2. In `src/sase/ace/tui/actions/agents/_folding_panel_sweep.py`:
   - remove the grouping-banner enumerator and `panel_fold_registry` dependencies;
   - make the liveness filter return only still-live, still-collapsed lane/clan levels;
   - sweep, restore, count, and availability-check only resolved lane and clan keys;
   - preserve per-panel records, row-focus re-anchoring, panel retirement, hint-mode
     teardown, isolation-revert independence, exact `FoldLevel` restoration, and one
     repaint/footer refresh per action;
   - when a panel has only open grouping banners and no open lane/clan, `-` must report
     that there is nothing to collapse or restore and must not mutate banner state.
3. Remove `expanded_panel_level_zero_group_keys` and its export from
   `src/sase/ace/tui/actions/agents/_folding_agent_groups.py` once no consumer remains.
   Keep the ordinary row-scoped group-collapse logic and `H` hint enumeration intact.
4. Rewrite `tests/ace/tui/test_agent_panel_fold_sweep.py` around the narrowed contract:
   - assert open `Done`/`Running` or project banners remain open across both sweep and
     reverse while lane/clan levels round-trip exactly;
   - retain coverage for hidden structural owners, row focus, merged layout, sibling
     isolation, malformed owners, record retirement, liveness filtering, warning paths,
     hint teardown, and no-op behavior outside Agents;
   - add an explicit group-only regression proving `-` neither collapses a banner nor
     creates a restore record.
5. Synchronize user-facing text in `docs/ace.md`, `docs/agent_families.md`, the Agents
   help modal, and command metadata so `-` is described as a lane/clan or structural
   fold sweep, not a sweep of every panel/group fold. Keep the action/config ID and
   default `-` binding stable. Keep the `H` documentation explicit that its precise
   hinted collapse can still target grouping banners.

## Phase 2: Verify implementation and integration

1. Run focused tests for the sweep, hinted `H`, footer, command catalog/help, keymap
   defaults/loading/validation, and the post-start stitches override regression. Confirm
   the `f24` fixture remains and the test passes.
2. Run `just test-visual`. Because group-only panels will no longer advertise the `-`
   sweep chip, regenerate intentional Agents-tab PNG goldens with
   `--sase-update-visual-snapshots`, inspect the actual/expected/diff artifacts, and
   accept only footer/reflow changes explained by the narrowed eligibility.
3. Run `just check`. If its scoped lane escalates or reports unusual selection, follow
   project guidance and run `just check-full`; otherwise run `just check-full` anyway
   because this is the combined epic landing tree. Distinguish actual test failures from
   the historical flake-baseline meta-gate: `sase-jb` and `sase-j6` own the two
   unrelated live nodes, while `sase-jf` records the already-fixed pre-`9c46891c5`
   stitches/minus history. Do not broaden this epic by fixing those unrelated tasks or
   by adding a currently passing node to the flake baseline.
4. Re-run the targeted stitches test and inspect `git diff` after all formatting or
   snapshot updates. Confirm no post-start commit is duplicated, reverted, or left
   unable to use the completed feature.

## Phase 3: Close, clean, and mark the original epic plan done

This is the final phase and must be completed by the implementing agent.

1. Re-run `sase bead show sase-j9`, `sase-j9.1`, and `sase-j9.2`. Confirm both children
   remain closed and the code/tests now satisfy the epic notes. Build a close note that
   names the two epic commits, the post-start integration audit, verification commands,
   and every proposed-follow-up outcome listed in this plan.
2. Close normally with
   `sase bead close sase-j9 --note "<complete verification and follow-up disposition>"`.
   Do not force. If close rejects unclosed descendants, finish or deliberately reopen
   the named work; use forced canceled/superseded closure only when the facts genuinely
   support that resolution.
3. Only after the epic is closed, run `just symvision`. If it reports expired `sase-j9`
   whitelist entries, dead exports, or unused code, first use the required Symvision
   memory workflow, remove the stale entries/code, and rerun Symvision plus `just check`
   after those file changes.
4. Open the plans sidecar with `/sase_repo` in the implementing workspace, then use
   `apply_patch` to change only the original epic plan's frontmatter status in
   `202608/agents_panel_fold_sweep.md` from `wip` to `done`.
5. Finish with `git status --short`, `sase bead show sase-j9`, and a frontmatter read of
   the original plan to prove the epic is closed, post-close cleanup is green, and the
   plan is marked done.
