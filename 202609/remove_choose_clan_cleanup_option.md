---
tier: tale
title: Remove the "Choose clan" option from the Agent Cleanup panel
goal: The Agent Cleanup panel no longer offers a clan chooser, and agent shells/families
  inside agent clans are cleaned up by the ordinary panel-scoped options exactly like
  agents outside clans, with regression tests locking that uniform treatment in.
size: medium
proposed_by: bbugyi200.athena.0ge
status: done
---

# Remove the "Choose clan" option from the Agent Cleanup panel

## Goal

Remove the `Choose clan` action from the ACE Agents-tab "Agent Cleanup" panel (`X` on
the Agents tab) and everything that exists only to serve it. Agent shells and agent
families that belong to an agent clan must be treated by the cleanup panel exactly like
ones that do not: done agents inside clans are dismissed by `Dismiss completed in panel`
/ `Dismiss completed everywhere`, and running ones are killed by
`Kill and dismiss panel` / `... everywhere`, with no clan-specific chooser in between.

## Verified current behavior (read this before changing anything)

The uniform treatment the user asked for **already holds** on the panel-scoped paths —
this was verified empirically against the current tree:

- Clan member rows are real rows in the Agents-tab list. `project_clan_tree`
  (`src/sase/ace/tui/models/_agent_tree.py:483`) emits the synthetic container followed
  by its member rows, so `_agents_in_focused_panel()` returns members alongside the
  container.
- `_dismiss_all_done_agents` / `_kill_and_dismiss_all_agents` (and their `_global`
  variants) filter those candidates by status + `raw_suffix`, which keeps members (they
  have suffixes) and drops the synthetic container (`raw_suffix is None`). A probe with
  one clan (one DONE member, one RUNNING member) plus a standalone DONE agent showed
  `Dismiss completed in panel` dismissing the DONE member and the standalone agent, and
  the same held for a DONE family (root + `--1` shell) nested inside a clan.
- The `x` (kill/clean row) action on a clan container row expands to the clan's members
  via `clan_members_for_container`
  (`src/sase/ace/tui/actions/agents/_kill_action_flow.py:89`) — a separate path from the
  cleanup panel that must keep working.

So this change is a **pure removal of the redundant clan chooser surface**, plus
regression tests that lock the uniform treatment in.

No feature flag is needed: per the flags policy a `sunset` flag exists to keep an old
branch reachable while callers migrate. Nothing programmatic consumes the `Choose clan`
panel action, every cleanup outcome it produced remains reachable through the other
panel options (and `x` on a clan row), and the project owner explicitly asked for
outright removal.

## Source changes (all in this repo)

1. `src/sase/ace/tui/modals/agent_cleanup_panel_modal.py`
   - Remove the `("C", "clan", "Clan")` entry from `BINDINGS`, the `action_clan` method,
     the `clan` `_ActionRow` in `_build_rows`, and the `_clan_detail` helper.
   - Remove `C clan` from the hints line in `compose` (new text:
     `"d/D dismiss completed  k/K kill running + dismiss completed  m marked  g group  t tribe  c custom  q close"`).
   - In `_context_block`, drop the `· N clans` segment (keep marked, group, and
     tribe-panel counts).
2. `src/sase/ace/tui/modals/agent_cleanup_types.py`
   - Remove `"clan"` from the `AgentCleanupAction` literal.
   - Delete `AgentCleanupClanKey` and `AgentCleanupClanResult`.
   - Remove the `clan_count` and `focused_clan_label` fields from
     `AgentCleanupPanelState`.
   - Update `__all__`.
3. Delete `src/sase/ace/tui/modals/agent_cleanup_clan_modal.py`
   (`AgentCleanupClanModal`).
4. Drop the clan modal + clan type exports from
   `src/sase/ace/tui/modals/agent_cleanup_modal.py`,
   `src/sase/ace/tui/modals/__init__.py`, `src/sase/ace/tui/modals/__init__.pyi`, and
   `src/sase/ace/tui/modals/_export_table.py`.
5. Delete `src/sase/ace/tui/actions/agents/_kill_cleanup_clan.py`
   (`AgentCleanupClanMixin`: `_agent_cleanup_clans_in_focused_panel`,
   `_agent_cleanup_clan_key`, `_focused_cleanup_clan_key`,
   `_focused_cleanup_clan_label`, `_plan_clan_cleanup_container`,
   `_open_clan_cleanup_selector`, `_present_clan_cleanup`) and unwire the mixin from
   `AgentKillMixin` in `src/sase/ace/tui/actions/agents/_kill_action.py`.
6. `src/sase/ace/tui/actions/agents/_kill_cleanup_panel.py`
   - In `_build_agent_cleanup_panel_state`, delete the whole
     clans/`clan_targets`/`clan_target_wires`/`cleanable_clans` block and the
     `clan_count=` / `focused_clan_label=` constructor arguments.
   - In `_run_agent_cleanup_panel_action`, delete the `action == "clan"` branch.
7. `src/sase/ace/tui/styles.tcss` — remove every `AgentCleanupClanModal ...` rule block
   (around lines 1406–1450).
8. `src/sase/ace/tui/modals/help_modal/agents_bindings.py` — change the
   `open_agent_cleanup_panel` line from `"Open cleanup panel (C: clan)"` to plain
   `"Open cleanup panel"`. Keep the `kill_agent` line
   (`"Clean row/panel/group/clan/marks"`) unchanged: `x` on a clan container row still
   cleans that clan. Per `src/sase/ace/CLAUDE.md`, keep the help modal's 57-char box
   formatting intact.

Keep untouched (they serve paths that remain):

- `src/sase/ace/tui/actions/agents/_clan_cleanup.py` (`clan_members_for_container`) —
  used by `_kill_action_flow`, `_kill_identity`,
  `_agent_cleanup_targets_from_candidates`, and the custom-selection flow.
- The clan expansion inside `_agent_cleanup_targets_from_candidates` and
  `_present_custom_cleanup` — custom selection may still land on a clan container row
  and must expand it to members.
- `src/sase/core/agent_cleanup_wire.py`, `agent_cleanup_python.py`,
  `agent_cleanup_targets.py` and their `tests/test_core_facade/` clan tests — the Python
  planner is a faithful mirror of the Rust planner in sase-core, and the `clan` scope
  must leave both sides in lockstep. That cross-repo removal is deliberately out of
  scope here and is tracked by task bead **sase-wz** ("Remove the dead clan cleanup
  scope from the Rust cleanup planner and its Python mirror"). Do not partially delete
  the mirror.
- `tests/ace/tui/test_agent_clan_dismiss_cascade.py` — covers the `x`-on- container
  path, which stays.
- The Agents-tab clan folding/summary/wait/fork features — the word "clan" appears all
  over the TUI; only the cleanup-panel chooser surface listed above is being removed.
- `src/sase/default_config.yml` — no change needed; the `C` key was a modal-local
  binding, not a configurable keymap.

## Test changes

1. Delete `tests/ace/tui/test_agent_cleanup_clan_modal.py`.
2. `tests/ace/tui/test_agent_cleanup_clan_e2e.py` — replace the clan-chooser flow with
   an end-to-end regression for the uniform treatment (keep the existing `_clan_member`
   fixtures/loaders): from the Agents tab with clans in tribe panels, `X` then `d` must
   dismiss the DONE clan members in the focused panel and leave other panels' members
   and RUNNING members alone; `X` then `k` must kill RUNNING members and dismiss DONE
   ones in the focused panel. Assert against `_dismissed_agents` and the killed set the
   way the current test does. Rename the file/test accordingly (e.g.
   `test_agent_cleanup_panel_clan_members_e2e.py`).
3. `tests/ace/tui/test_agent_cleanup_modal.py` — drop `clan_count` /
   `focused_clan_label` kwargs from `_state`, delete
   `test_agent_cleanup_modal_clan_row_context_and_availability`, remove the
   `rows["clan"]` assertion, and update the hints-string expectation to the new text.
4. `tests/ace/tui/test_panel_scoped_bulk.py` — delete the five clan-chooser tests
   (`test_clan_cleanup_panel_state_counts_cleanable_clans_and_focus`,
   `test_open_clan_cleanup_selector_is_tribe_scoped_and_pre_highlighted`,
   `test_single_clan_cleanup_uses_clan_scope_and_strips_containers`,
   `test_mixed_clan_cleanup_unions_whole_clan_and_member_selection`,
   `test_clan_cleanup_preserves_generation_and_no_tribe_panel_scope`). Add two focused
   replacements: panel-scoped dismiss and panel-scoped kill-and-dismiss candidate
   selection include clan member rows (direct members and a family shell inside a clan)
   exactly like non-clan rows, with the synthetic container itself never selected.
5. `tests/ace/tui/visual/test_ace_png_snapshots_agents_modals.py` — delete
   `test_agent_cleanup_clan_modal_png_snapshot` plus its `_cleanup_clan_member` /
   `_cleanup_clan_modal` fixtures, and delete the golden
   `tests/ace/tui/visual/snapshots/png/agent_cleanup_clan_modal_partial_100x32.png`. The
   `AgentCleanupModal` panel itself has no PNG golden, and
   `sase_agent_cleanup_confirmation_120x40.png` does not exercise the clan flow, so no
   goldens need regenerating.

## Verification

- `just install` first if this workspace's venv is stale, then `just check` (whole-repo
  lint gates + the diff-scoped test lane) before finishing — this is mandatory for any
  file change in this repo.
- `just test-visual` to prove the PNG suite is green after the snapshot test and golden
  deletion (no `--sase-update-visual-snapshots` should be needed — nothing rendered by
  remaining goldens changes).
- Watch symvision in `just check`: the deletions must not orphan
  `clan_members_for_container` (it stays used) and must remove every now-unused
  clan-chooser symbol in the same change rather than whitelisting.

## Related context

- Task bead **sase-wz** tracks the follow-up removal of the then-dead `clan` scope from
  the Rust planner in sase-core and its Python mirror here; it is blocked on this tale
  landing.
- Task bead **sase-wo** (bare uppercase modal keys dead under CSI-u key reporting) lists
  this modal's `C` binding among its findings; this change removes that key entirely,
  shrinking sase-wo's remaining scope. No action needed here beyond the removal itself.
