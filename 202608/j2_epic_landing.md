---
tier: tale
title: Complete and land epic sase-j2
goal:
  Epic sase-j2 is fully integrated, verified, closed, Symvision-clean, and marked done
  in its durable plan.
size: medium
proposed_by: bbugyi200.athena.sase-j2.land
bead: sase-j2
create_time: 2026-08-10 15:41:29
status: done
---

- **PROMPT:**
  [prompts/202608/j2_epic_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/j2_epic_landing.md)
- **PARENT:**
  [202608/tribe_zoom_and_panel_isolation_keymap.md](https://github.com/sase-org/sase--plans/blob/main/202608/tribe_zoom_and_panel_isolation_keymap.md)
- **BEAD:**
  [sase-j2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-j2/README.md)

# Complete and land epic `sase-j2`

## Objective

Finish the epic-owned cleanup found by the land audit, revalidate the combined tree,
then close `sase-j2`, run the required post-close Symvision cleanup, and mark its
durable plan done.

## Verified baseline

- Epic `sase-j2` has two closed children: `sase-j2.1` (panel isolation on `=`) and
  `sase-j2.2` (whole-tribe metadata zoom on `Z`). Neither child nor the epic contains a
  `PROPOSED FOLLOW-UP:` note.
- The implementation commits are `5f6d8ea64` and `63f9f15d6`. Their current source,
  tests, docs, keymaps, command metadata, footer plumbing, modal refresh/enrichment, and
  isolation state handling match the intended behavior.
- Non-epic commits after the first epic commit are `c8e4016c7`, `83bb8a6f7`, and
  `e01584098`. The first two are test-infrastructure changes. The last adds snippet
  configuration and moves snippet-target code; its edits to `default_config.yml` and
  modal exports are separate from the epic's keymap and zoom code, so no integration
  change is needed there.
- The focused behavior/keymap/command/footer suite passes: 200 tests.
- A full `just test-visual` run deterministically fails 23 snapshots. Representative
  expected/actual inspection shows the intended new `= only panel` footer entry and the
  resulting footer/panel reflow. Rerunning exactly those 23 nodes reproduces all 23
  failures, so the committed goldens are stale rather than flaky.
- A few identifiers/comments still call isolation a `Z` or zoom action even though the
  invoked action is now `action_isolate_panels`.

## Phase 1: Finish terminology cleanup

Update only stale internal descriptions; do not alter behavior:

- In `src/sase/ace/tui/actions/agents/_folding_panels.py`, describe isolation markers as
  applying to the next isolation action, not a whole-panel zoom action.
- In `tests/ace/tui/test_agent_panel_isolation_revert.py`, retitle the module and its
  restore-ownership comment from zoom-action terminology to isolation-action
  terminology.
- In `tests/ace/tui/test_agent_panel_collapse_isolation.py`, rename the three
  `test_capital_z_*` tests to `test_equals_*` (or an equivalently explicit isolation
  name).
- In `tests/test_keymaps_display_help.py`, make the test name explicitly say
  `zoom_and_isolation` so it no longer reads as a combined zoom-isolation action.

Search again for stale `Z`/zoom-isolation wording. Preserve legitimate zoom-search and
unrelated `Z` bindings.

## Phase 2: Refresh and review the stale visual goldens

Regenerate only the 23 reproducible failures with
`just test-visual -- --sase-update-visual-snapshots <node ids>`, using these nodes:

1. `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py::test_agents_neighbor_jump_expands_target_panel_png_snapshot`
2. `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py::test_agent_neighbor_modal_folded_clan_and_tribe_png_snapshot`
3. `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py::test_agent_neighbor_modal_narrow_png_snapshot`
4. `tests/ace/tui/visual/test_ace_png_snapshots_agents.py::test_agent_list_png_snapshot`
5. `tests/ace/tui/visual/test_ace_png_snapshots_agents.py::test_agent_reverted_indicator_png_snapshot`
6. `tests/ace/tui/visual/test_ace_png_snapshots_agents.py::test_agent_stopped_status_png_snapshot`
7. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_clan_collapse.py::test_selected_panel_clan_collapse_precedes_status_group_png_snapshot`
8. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_cleanup.py::test_agent_lane_cleanup_confirmation_png_snapshot`
9. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_layout.py::test_agents_overflowing_panel_uses_full_height_png_snapshot`
10. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_layout.py::test_agents_unread_highlight_png_snapshot`
11. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py::test_agents_sole_selected_panel_png_snapshot`
12. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py::test_agents_collapsed_panel_png_snapshot`
13. `tests/ace/tui/visual/test_ace_png_snapshots_agents.py::test_agents_selected_row_png_snapshot`
14. `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_display_config_png_snapshot`
15. `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py::test_agents_leader_jump_auto_expands_panel_png_snapshot`
16. `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py::test_tribe_panel_four_level_png_snapshots`
17. `tests/ace/tui/visual/test_ace_png_snapshots_agents_group_clan_collapse.py::test_selected_clan_collapses_before_open_sibling_png_snapshot`
18. `tests/ace/tui/visual/test_ace_png_snapshots_agents_group_clan_collapse.py::test_group_clan_collapse_precedes_status_banner_png_snapshot`
19. `tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py::test_auto_approve_modal_png_snapshot`
20. `tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py::test_agent_workspace_tmux_modal_png_snapshot`
21. `tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py::test_wait_modal_png_snapshot`
22. `tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py::test_agent_cleanup_clan_modal_png_snapshot`
23. `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py::test_agents_neighbor_badge_png_snapshot`

Review the changed PNGs before accepting them. Every change should be explained by the
new isolation footer affordance and its deterministic layout reflow; investigate rather
than accept any unrelated content change.

## Phase 3: Revalidate the combined epic tree

- Rerun the focused 200-test behavior/keymap/command/footer suite from the land audit.
- Run `just test-visual` without the update flag and require the full visual suite to
  pass.
- Run `just check-full` as required by the epic's original plan. If any unrelated
  failure appears, follow the repository's bead workflow rather than silently ignoring
  it; epic-caused failures remain part of this work.
- Inspect `git diff --check`, the final diff, and the current source paths named in the
  epic plan. Confirm again that the later commits need no integration edits.

## Phase 4: Close and finalize the epic

This is the final phase and must run only after all prior phases are green.

1. Re-show `sase-j2` and both children and confirm every descendant is closed. Record
   that no `PROPOSED FOLLOW-UP:` entries existed; therefore no `/sase_new_task` outcome
   is required.
2. Close with `sase bead close sase-j2 --note "..."`. The note must summarize the
   source/commit verification, the three later non-epic commits and integration result,
   the stale terminology and 23-golden remediation, all verification commands/results,
   and the absence of follow-up proposals. If close is rejected, resolve or reopen the
   named incomplete phases; do not force a successful close.
3. After the close succeeds, run `just symvision`. Remove only stale `sase-j2` whitelist
   entries and unused code it reports, then rerun the appropriate repository checks for
   any resulting file changes.
4. In the linked plans repository, change the frontmatter of
   `202608/tribe_zoom_and_panel_isolation_keymap.md` from `status: wip` to
   `status: done`, preserving all other plan content.
5. Finish with clean status checks for both repositories and report the close,
   verification results, Symvision outcome, plan status update, and any residual
   unrelated work explicitly.
