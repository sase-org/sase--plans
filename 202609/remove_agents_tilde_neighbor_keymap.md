---
tier: tale
title: Remove the Agents-tab ~ neighbor keymap and neighbors badge
goal: 'On the Agents tab, ~ no longer opens or jumps to neighbors and the info panel
  no longer shows [neighbors: N (~)]; numbered NEIGHBORS jumps remain the only neighbor
  navigation, while Patches/Artifacts ~ family navigation is unchanged.'
size: medium
proposed_by: bbugyi200.apollo.1c
status: done
---

# Remove the Agents-tab `~` neighbor keymap and the `[neighbors: N (~)]` badge

## Goal

Agent-neighbor navigation now lives in the numbered `NEIGHBORS` section of the agent
metadata panel (`0`–`9` / `00`–`99` member jumps, including reviving dismissed
descendants). Retire the old path completely:

- On the Agents tab, pressing `~` (`start_sibling_mode`) no longer does anything: no
  direct jump and no `AgentNeighborModal` chooser.
- The `[neighbors: <N> (~)]` badge in the Agents info panel at the top of the TUI is
  removed.
- Every Agents-tab hint that points at `~` goes away: the footer binding, the help modal
  rows, command palette exposure, and the docs.

## Scope decision: `~` stays on Patches / Artifacts

`start_sibling_mode` is shared. On the Patches and Artifacts tabs, `~` drives
relation-panel **FAMILY** navigation (`~~`, `~a`, …). The Rust relation layout
(`sase_core` `artifact_relation_layout`) produces those key labels, and the numeric
NEIGHBORS keymaps did not replace them. So this plan removes only the Agents-tab
behavior and every Agents-tab surface for it. It does **not** delete the
`start_sibling_mode` action, its `~` default in `src/sase/default_config.yml`, the
`Binding("~", ...)` in `bindings.py`, `_process_sibling_key`, or the Patches/Artifacts
relation panel and help entries. No `sase-core` changes are needed.

## Keep (the numeric NEIGHBORS path still depends on these)

- `AgentNeighborMixin._agent_neighbor_index`, `_build_agent_neighbor_index`,
  `_revealable_agent_neighbor_rows`, `_visible_agent_neighbor_rows`,
  `_active_dismissed_agent_objects`, `_dismissed_descendant_agents`,
  `lane_neighbor_projection_for`, `_agent_neighbor_display_hoods`, and the
  `_agent_neighbor_index_cache` invalidation hooks. `_member_jump.py`, the NEIGHBORS
  section renderer, and the prompt panel header use them.
- `models/sase_agent_neighbors.py` (`build_sase_agent_neighbor_projection`) and
  `models/agent_hoods.py` (`AgentNeighborIndex`, including `neighbor_count`).
- `_selected_agent_neighbor_count` in `actions/agents/_display_detail_info.py`. The
  footer's `lane_neighbor_jump_available` (the `0-9 neighbor` hint) and
  `_notify_lane_fold_scope` in `actions/navigation/_fold.py` still use it.

## Changes

### 1. Agents-tab `~` behavior (`src/sase/ace/tui/actions/navigation/_tree.py`)

- In `action_start_sibling_mode`, drop the `if self.current_tab == "agents":` delegation
  branch. The action then falls through to the relation-contract check, which already
  returns early on tabs without a relation contract, so it is a no-op on Agents. Adding
  an explicit early `return` for `agents` is also fine.
- Update the docstring so it describes only Patch/Artifacts FAMILY navigation.

### 2. Delete the chooser flow (`src/sase/ace/tui/actions/agents/_neighbors.py`)

Remove `_start_agent_neighbor_navigation` and everything only it reaches:
`_AgentNeighborPayload`, `_focus_agent_neighbor_by_identity`,
`_restore_agent_neighbor_jump_history`, `_refresh_agent_neighbor_jump_views`,
`_agent_neighbor_choices`, `_agent_neighbor_panel_label`,
`_agent_neighbor_dismissed_panel_label`, and `_agent_neighbor_time_hint`. Also remove
imports that become unused (the `AgentNeighborChoice` TYPE_CHECKING import, the
`_agent_reveal` helpers if nothing else uses them, `dataclass`, etc.). Before deleting a
helper, re-grep `src/` and `tests/` to confirm it has no other caller. If a helper turns
out to be shared with the member-jump path, keep it.

### 3. Delete `AgentNeighborModal`

- Delete `src/sase/ace/tui/modals/agent_neighbor_modal.py`.
- Remove its `AgentNeighborChoice` / `AgentNeighborModal` entries from
  `src/sase/ace/tui/modals/__init__.py` (`__all__`), `__init__.pyi`, and
  `_export_table.py`.
- Remove the `AgentNeighborModal { ... }` block, and any rules scoped to its child
  widgets, from `src/sase/ace/tui/styles.tcss`.
- `tests/ace/tui/widgets/test_imported_owner_badge.py` builds an `AgentNeighborChoice`
  to check the imported-owner badge in neighbor pickers. Delete that test case. If
  another neighbor surface still renders the imported badge (for example the NEIGHBORS
  section), point the test at that surface instead.

### 4. Remove the info-panel badge

- `src/sase/ace/tui/widgets/agent_info_panel.py`: delete `_append_neighbor_badge` and
  its call in `_build_display_text`, the `_neighbor_count` attribute, and the
  `neighbor_count` keyword on `update_state`. Take `neighbor_count` out of both
  stable-state tuples (around lines 258, 283, and 317). Drop the `key_display_name`
  import if nothing else uses it.
- `src/sase/ace/tui/actions/agents/_display_detail_info.py`: stop computing
  `neighbor_count` for the info panel and stop passing `neighbor_count=` to
  `update_state`. Keep `_selected_agent_neighbor_count` itself, because the footer uses
  it.

### 5. Remove the footer `~ neighbors (N)` hint

- `src/sase/ace/tui/widgets/_keybinding_bindings.py`: delete the
  `if neighbor_count > 0:` block that appends `start_sibling_mode` with `neighbor` /
  `neighbors (N)`, and remove the now-unused `neighbor_count` parameter. Keep
  `lane_neighbor_jump_available` and its `("0-9", "neighbor")` hint.
- `src/sase/ace/tui/widgets/_keybinding_modes.py`: remove the `neighbor_count` parameter
  and pass-through (around lines 70, 157, and 191).
- `src/sase/ace/tui/actions/agents/_display_detail_footer.py`: stop passing
  `neighbor_count=`. Keep the local `neighbor_count` value that
  `lane_neighbor_jump_available` uses.

### 6. Help modal and command palette

- `src/sase/ace/tui/modals/help_modal/agents_bindings.py`: delete both
  `d(a.start_sibling_mode)` rows ("Jump ancestor/neighbor/desc" and "Neighbors modal
  (see NEIGHBORS)"). Keep the `("0-9", "Jump numbered member/neighbor")` row.
- `src/sase/ace/tui/commands/_app_metadata_display.py`: change the `start_sibling_mode`
  entry's tab scope from `CL_AGENTS` to `CL_ONLY`, matching ancestor/child.
- `src/sase/ace/tui/commands/_availability_agents.py`: remove `"app.start_sibling_mode"`
  from `_COLLAPSED_PANEL_HIDDEN_AGENT_COMMANDS` and `_REMOTE_AGENT_LOCAL_COMMANDS`.
- `src/sase/ace/tui/_app_action_availability.py`: remove `"start_sibling_mode"` from
  `_LOCAL_AGENT_ROW_ACTIONS`. Keep it in `_ARTIFACT_RELATION_ACTIONS`. Check that
  `check_app_action` no longer treats it as an agent-row action on the Agents tab.
- Leave `patches_bindings.py`, `patches_artifact_bindings.py`, and
  `widgets/artifacts/relation_panel.py` unchanged. They are Patch/Artifacts relation
  surfaces.
- `src/sase/default_config.yml` keeps `start_sibling_mode: "~"`, because Patches still
  uses it. Check whether the key's comment or label mentions agents or neighbors, and
  fix it if so. Also update the `keymaps/metadata.py` label only if it mentions agents.

### 7. Docs (`docs/ace.md`)

- Delete the Agents keymap table row
  ``| `~` | Jump among agent-node-name ancestors, descendants, and shared-hood neighbors (see `NEIGHBORS`) |``.
- Rewrite the "On the Agents tab, `~` uses dotted agent-name relationships…" paragraph
  (around line 1275) so it describes the relationships themselves (sase-agent names,
  hoods, ancestors/descendants, dismissed descendants, reveal through folds) as what the
  numbered `NEIGHBORS` rows show and what a digit jump reaches. Remove the
  chooser/direct-jump wording.
- In the member-jump paragraph, replace "exactly as `<enter>` does in the `~` chooser"
  with plain wording (a digit on a dismissed neighbor revives that agent).
- In the NEIGHBORS-section text (around lines 1636–1659), stop defining the rows as "the
  rows the `~` chooser offers". Define them directly as ancestors, descendants
  (including same-session dismissed descendants), then hood neighbors. Rewrite the last
  sentence so it no longer mentions the `~` chooser or the info panel's `neighbors:`
  badge (say the heading count still includes the suppressed rows).
- Leave the Patches `<` / `>` / `~` rows (around lines 319, 698, and 945) unchanged.

### 8. Tests

Delete or rewrite every test that exercises the removed surfaces, and keep coverage of
the behavior that survives:

- `tests/ace/tui/modals/test_agent_neighbor_modal.py`: delete.
- `tests/ace/tui/test_agent_neighbor_navigation.py` and
  `tests/ace/tui/test_agent_neighbor_navigation_targets.py`: delete tests driven by
  `action_start_sibling_mode()` or `AgentNeighborModal`. Keep, and move into a suitably
  named file if needed, the tests that check index semantics without `~`: for example
  `test_selected_agent_neighbor_count_includes_ancestors`, the
  `_selected_agent_neighbor_count(...) == 0/1` assertions, and
  stale-identity/filtered-target rejection. Where a deleted `~` test covered behavior
  that digit jumps now provide (reveal through collapsed clans or tribe panels, revive
  of dismissed descendants, stale-target cancellation), check that
  `tests/ace/tui/test_member_jump*.py` (and helpers such as
  `_member_jump_navigation_helpers.py`) already cover it. Add a digit-jump test if
  coverage is missing. Clean up `tests/ace/tui/_agent_neighbor_navigation_helpers.py` if
  it becomes unused.
- Add a regression test: on the Agents tab, with a selected agent that has neighbors,
  `action_start_sibling_mode()` does not push a screen and does not change
  `current_idx`.
- `tests/ace/tui/widgets/test_agent_info_panel_badges.py`: remove the `neighbors:` badge
  tests and the `DEFAULT_NEIGHBOR_KEY` import. Update
  `tests/ace/tui/widgets/_agent_info_panel_helpers.py` (drop `DEFAULT_NEIGHBOR_KEY` and
  any `neighbor_count` plumbing). Add one assertion that the rendered panel never
  contains `neighbors:`.
- `tests/test_command_catalog_guards.py::test_tree_navigation_command_tab_scopes`:
  expect `sibling.tabs == ("artifacts",)` and update the docstring.
- `tests/test_command_availability_agents_panels.py`: remove `"app.start_sibling_mode"`
  from the hidden-command set (it is no longer an Agents command). Check
  `tests/ace/tui/test_artifacts_relation_surfaces.py` and
  `tests/ace/tui/artifacts_contract/harness.py` still pass unchanged, since they cover
  Artifacts `~`.
- Any footer-binding tests that assert `neighbors (N)` or `neighbor` with the `~` key:
  update them to expect only the `0-9 neighbor` hint.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_neighbors.py`:
  - Delete `test_agents_neighbor_badge_png_snapshot`,
    `test_agent_neighbor_modal_folded_clan_and_tribe_png_snapshot`,
    `test_agent_neighbor_modal_narrow_png_snapshot`, and
    `test_agent_neighbor_modal_dismissed_descendant_png_snapshot`. Delete their goldens
    in `tests/ace/tui/visual/snapshots/png/`: `agents_neighbor_badge_120x40.png`,
    `agent_neighbor_folded_clan_modal_70x32.png`, `agent_neighbor_modal_60x30.png`, and
    `agent_neighbor_modal_descendants_dismissed_60x30.png`.
  - `test_agents_neighbor_jump_expands_target_panel_png_snapshot` calls
    `action_start_sibling_mode()`. Change it to reach the same target through the
    numbered NEIGHBORS digit key (press the digit shown for the target row), keeping the
    same state assertions. If the resulting screen differs, regenerate its golden
    `agents_neighbor_jump_expanded_panel_120x40.png`. If a digit jump cannot reproduce
    the scenario, delete the test and its golden instead.
  - Any remaining Agents-tab goldens that showed the badge in the info panel, or the
    `~ neighbor(s)` footer hint (for example the lane-neighbors-section snapshots), must
    be regenerated with the repo's documented PNG snapshot update workflow. Read the
    `tui` memory (`tui_screenshot`) before regenerating, and look at each updated PNG to
    confirm that only the badge or footer hint changed.
- Remove stale `neighbor_count` stubs in test helpers such as
  `tests/ace/tui/_agents_panel_fold_mode_helpers.py` only if their callers are gone.
  `_selected_agent_neighbor_count` itself stays.

## Verification

- `rg -n 'AgentNeighborModal|AgentNeighborChoice|_start_agent_neighbor_navigation|_append_neighbor_badge|DEFAULT_NEIGHBOR_KEY' src tests docs`
  returns nothing.
- `rg -n 'neighbors: ' src` returns nothing, and `rg -n '~' docs/ace.md` shows no
  Agents-tab neighbor references.
- Manual TUI check with the `/run` skill or `sase screenshot`: on the Agents tab, select
  an agent with hood neighbors. The top info panel shows no `[neighbors: …]` segment,
  the footer shows `0-9 neighbor` but no `~` entry, pressing `~` does nothing, and a
  digit from the NEIGHBORS section still jumps (or revives a dismissed descendant). On
  the Patches tab, `~` still drives relation FAMILY navigation.
- Run `just check`, following the `lint_and_test` memory, and fix any Symvision
  unused-symbol findings caused by the deletions (read the `symvision` memory if
  needed).
