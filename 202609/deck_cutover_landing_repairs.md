---
tier: epic
title: Finish the deck cutover's visual migration, coverage goldens and live checks
goal: Every ACE visual test that still drives the deleted legacy Agents detail UI
  is migrated to the deck API (or deleted when its subject is gone), its goldens are
  regenerated and inspected, the coverage goldens and live screenshot checks that
  the cutover-goldens phase skipped are done, and `just fix-tui-screenshots --check`
  is clean apart from failures that reproduce without the cutover.
parent_bead: sase-17d.10.1
phases:
- id: family-visual-migration
  title: Migrate the family, monitor and collapsed-panel visual tests to decks
  depends_on: []
  size: medium
  description: 'family-visual-migration: confirm the land-agent repairs from section
    2 are on master (reapply any that are missing), then migrate the family-panel,
    monitor, gate-shell and collapsed-panel ACE PNG tests listed in section 3.1 off
    the deleted legacy ids and section-stop APIs onto the deck API, regenerate only
    their goldens with a scoped just fix-tui-screenshots, and inspect every changed
    PNG.'
- id: tribe-files-visual-migration
  title: Migrate the tribe, clan, files, LLM Calls, search and waiting visual tests
  depends_on: []
  size: medium
  description: 'tribe-files-visual-migration: migrate the tribe, clan, slow-tools,
    linked/external repo, LLM Calls, metadata-search and waiting ACE PNG tests listed
    in section 3.2 onto the deck API, delete the zoom-modal-only waiting scenario,
    regenerate only their goldens with a scoped just fix-tui-screenshots, and inspect
    every changed PNG.'
- id: cutover-coverage-and-live
  title: Add the missing deck coverage goldens and run the live and full checks
  depends_on:
  - family-visual-migration
  - tribe-files-visual-migration
  size: medium
  description: 'cutover-coverage-and-live: add the spread Files, paged-after-threshold
    and LEFT_RIGHT committed-search-overlay goldens, capture and inspect live sase
    screenshot PNGs of the deck layouts, check the deck border-title clipping, run
    the full just fix-tui-screenshots --check clean except for failures proven to
    reproduce without the cutover, and record j/k bench numbers.'
proposed_by: bbugyi200.athena.sase-17d.10.1.land
create_time: 2026-09-24 17:30:34
status: wip
bead_id: sase-17d.10.1.4
---

- **PROMPT:** [prompts/202609/deck_cutover_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/deck_cutover_landing_repairs.md)
- **PARENT:** [202609/deck_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202609/deck_cutover.md)
- **BEAD:** [sase-17d.10.1.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.10.1.4.md)

# Plan: finish landing the deck cutover (child of epic sase-17d.10.1)

## 1. Context

Epic `sase-17d.10.1` ("Cut over the Agents tab to decks and delete the legacy detail
UI", plan `plan:202609/deck_cutover.md`) made deck panels the only Agents-tab detail UI.
Its three phases are closed: `deck-flag-removal` (`sase-17d.10.1.1`),
`legacy-ui-deletion` (`sase-17d.10.1.2`) and `cutover-goldens` (`sase-17d.10.1.3`). Its
land agent found that the last phase was closed without finishing its goal. Read the
parent plan's section 4 ("Phase cutover-goldens") and the bead notes on
`sase-17d.10.1.3` before starting any phase here.

The legacy detail UI no longer exists. The following are all gone:
`#agent-prompt-scroll`, `#agent-file-scroll`, `#agent-llm-calls-scroll`,
`#agent-search-scroll`/`-panel`/`-command`, `#agent-deck-source-host`,
`AgentInfoPanel._view_mode`, the view chip, `DetailPanelMode`/`DetailLayoutMode`, the
zoom modal (`modals/zoom_panel_*`), the view picker, and the metadata section-stop
actions. The hidden Main source `AgentPromptPanel` (`#agent-prompt-panel`, class
`-deck-source`) is still mounted, because it builds the Main documents. However, it no
longer scrolls, so its `active_section_identity` stays `None`.

The deck API that replaces the legacy detail UI:

- `AgentDetail.show_deck(panel_index, deck)` and `set_deck_preferred_card`
- `AgentDetail.toggle_deck_split(DeckLayout.LEFT_RIGHT)` (and other layouts)
- `AgentDetail.deck_area`, with `focused_panel()` and `panel(i)`
- `DeckPanel.active_scroll()`, `show_search_overlay()`/`hide_search_overlay()` and
  `search_command()`
- `focused_tools_view()` and the other accessors in
  `src/sase/ace/tui/widgets/_agent_detail_deck_targets.py`

Existing deck tests are working examples:

- `tests/ace/tui/widgets/decks/` (for example `test_deck_search_overlay.py` and
  `test_deck_panels.py`)
- goldens named `agents_decks_*`, for example `agents_decks_single_main_reply_120x40`

Before changing visual tests or goldens, read the `tui` reference memory
(`sase memory read sase/memory/tui.md`). Run every full or broad
`just fix-tui-screenshots` through `/sase_monitor`. Scoped runs
(`just fix-tui-screenshots -- <selector>`) are fine inline.

**Scope rules.** Presentation-only work; no `sase-core` change. Do not edit `docs/` or
`sase/memory/`; sibling phase `sase-17d.11` owns those. Do not "fix" unrelated failing
tests. Compare against a tree without the cutover first. The cutover commits are
`bda83083b`, `742c1df38` and `37c8bb264`; `git revert --no-commit` of those three in a
scratch worktree gives a usable baseline. Resolve the two trivial conflicts in
`src/sase/feature_flags/registry.py` and
`tests/ace/tui/widgets/test_agent_slow_tools.py`. Then run pytest with
`PYTHONPATH=<worktree>/src` and `-m visual`.

## 2. Land-agent repairs (verify first in `family-visual-migration`)

The land agent made these repairs while verifying the epic. Confirm each one is on
master. If any is missing, reapply it exactly as described.

1. **mypy (10 errors).** `AgentDetailDisplayMixin` (`widgets/_agent_detail_display.py`)
   and `AgentDetailStateMixin` (`widgets/_agent_detail_state.py`) now inherit
   `textual.widgets.Static`, as `AgentDetailJumpMixin` does, so that `query_one`
   type-checks. Both declare a `_sync_header_visibility` stub that raises
   `NotImplementedError`. The state mixin also declares an
   `update_display(agent, stale_threshold_seconds=10, attempt_number=None)` stub.
   `AgentDetail` defines the real header sync. The display mixin, which is earlier in
   the MRO, defines the real `update_display`.
2. **Symvision unused-public symbols.** The deleted code was the only cross-file
   consumer of these symbols, so each was made private in its own file:
   - `_FileSourceLabel` (`widgets/file_panel/_file_list.py`)
   - `_invert_search_direction`, `_offset_for_row` and `_wrap_feedback_message`
     (`widgets/vim_search_controller.py`, removed from `__all__`; the unit test imports
     `_offset_for_row`)
   - `_status_text` in `src/sase/main/monitor_render.py` and
     `src/sase/main/proc_render.py` (removed from `__all__`;
     `tests/main/test_monitor_render_status.py` imports `_status_text`)
3. **Dead legacy-id fallbacks.**
   - `AgentLLMCallsPanel._get_scroll_container` (`widgets/llm_calls_panel.py`) and
     `AgentFilePanel._get_scroll_container` (`widgets/file_panel/_content.py`) no longer
     fall back to querying `#agent-llm-calls-scroll`/`#agent-file-scroll`.
   - Each returns its `VerticalScroll` parent or `None`.
   - Each keeps a `try/except` around `self.parent`, because unmounted test panels raise
     there.
4. **Non-visual tests.**
   - `tests/ace/tui/test_agent_fold_transitions_llm_calls.py` now asserts the deck
     semantics: a focused Tools view takes `l`/`h`/`L`/`H`, and an unfocused one leaves
     the keys to the tree.
   - `tests/ace/tui/test_agents_tab_current_project_seed.py` drops the deleted
     `view_mode` kwarg.
   - `tests/ace/tui/widgets/test_agent_jump_panel_visibility.py`'s search-overlay test
     uses `deck_area.focused_panel().show_search_overlay()`.
   - `tests/test_file_panel.py::test_zoom_file_cap_subtitle_points_to_editor`, which
     tested the deleted zoom modal, is deleted.

## 3. Visual tests to migrate

Each test below fails deterministically on master because it drives a deleted legacy
API. Selectors are under `tests/ace/tui/visual/`. For each test:

- A legacy scenario maps to the matching deck state. For example:
  - scrolling `#agent-prompt-scroll` to a section becomes selecting that card or section
    in the Main deck
  - the file panel becomes the Files deck
  - LLM Calls becomes the Tools deck
  - metadata search becomes the per-panel deck search overlay
  - a metadata+file layout becomes a Main|Files split
- Replace `active_section_identity` waits on the hidden source panel with a deck-visible
  predicate, such as the focused deck's active card or scroll offset.
- Drop a scenario that has no deck equivalent, and say so in the bead note.
- Keep `wait_for_visual_idle(page)` as the final await before capture. The external repo
  test currently fails the convergence guard.
- Regenerate only the migrated scope. Open each changed PNG and check it for:
  - the deck tab strip and subtitle
  - the empty states
  - no `view:` chip and no `(p)` hint
  - the intended card or section in view

### 3.1 Phase `family-visual-migration`

- `test_ace_png_snapshots_agents_family_panel_monitor.py`: all 7
  `test_monitor_state_detail_png_snapshots[...]` cases (`assert None == 'monitor'`),
  plus `test_family_panel_shells_monitor_metadata_png_snapshot` and
  `test_family_conversation_monitor_phase_png_snapshot` (`assert None == 'agent-reply'`)
- `test_ace_png_snapshots_agents_family_panel_gate.py::test_selected_gate_shell_output_png_snapshot`
  (`#agent-prompt-scroll` NoMatches)
- `test_ace_png_snapshots_agents_family_panel.py::test_family_panel_fold_levels_and_member_override_png_snapshots`
  (`assert None == 'agent-xprompt'`)
- `test_ace_png_snapshots_agents_panels.py::test_agents_collapsed_panel_png_snapshot`
  (`AgentInfoPanel._view_mode` AttributeError)

### 3.2 Phase `tribe-files-visual-migration`

- `test_ace_png_snapshots_agents_tribe_clan_summaries.py`,
  `..._agents_tribe_panel.py::test_tribe_panel_four_level_png_snapshots`,
  `..._agents_tribe_prompts.py` (section-identity and `#agent-prompt-scroll` failures)
- `test_ace_png_snapshots_agents_clan_panel.py::test_swarm_clan_panel_png_snapshots` (it
  still queries `#agent-prompt-scroll` near line 320)
- `test_ace_png_snapshots_agents_slow_tools.py::test_agents_slow_tool_calls_fold_levels_png_snapshots`
  ("Timed out focusing slow-tool calls section"; it queries `#agent-prompt-scroll`)
- `test_ace_png_snapshots_agents_linked_repos.py`: the diff-file and commit-messages
  tests (`#agent-file-scroll`/`#agent-prompt-scroll`)
- `test_ace_png_snapshots_agents_external_repos.py::test_agents_external_repo_diff_file_panel_png_snapshot`
  (capture frame changed after convergence)
- `test_ace_png_snapshots_llm_calls.py`, the `agents_llm_calls_panel_full_120x40` case
  (it queries `#agent-llm-calls-scroll` near line 438)
- `test_ace_png_snapshots_agents_metadata_search.py::test_agents_metadata_search_typing_and_committed_png_snapshots`
  (`#agent-search-command`). Migrate it to the deck search overlay.
- `test_ace_png_snapshots_agents_waiting.py::test_agents_waiting_unknown_zoom_modal_png_snapshot`.
  The zoom modal is deleted, so delete this test and its golden. If the waiting-badge
  coverage it gave is not already covered by another test, move that coverage into a
  deck scenario.

The following phase-2 leftovers only affect test harnesses. Update them if the phase
touches those modules anyway:

- `tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel_gate.py:149`
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py:50`
- the `modals/zoom_panel_modal.py` path string in the LLM Calls visual fixture

## 4. Phase `cutover-coverage-and-live`

- **Coverage goldens.** Add each of these that is still missing. The spread Main case is
  already covered by `agents_decks_single_main_reply_120x40`.
  - spread Files
  - paged-after-threshold (see the sase-17d.8 notes)
  - a LEFT_RIGHT committed-search overlay (see the sase-17d.6 notes)
- **Live screenshots.** Capture live `sase screenshot` PNGs of each layout below and
  inspect them. Also check whether the deck detail border title clips its label near the
  tab strip. The `sase-17d.10.1.3` notes saw this in the regenerated deck goldens. If it
  clips, fix the source and regenerate only that scope.
  - single Main
  - a Main|Files split
  - Context|Reply
  - a collapsed node panel
  - a zoomed panel
- **Full check.** Run `just fix-tui-screenshots --check` through `/sase_monitor`. It
  must be clean except for failures that also reproduce on the no-cutover baseline. At
  the land agent's check, the two failures below timed out in `wait_for_startup` on both
  trees, so they are not cutover work:
  - `test_ace_png_snapshots_provider_usage_indicator.py::test_top_bar_usage_attention_narrow_png_snapshot`
  - `test_ace_png_snapshots_provider_usage_indicator_states.py::test_top_bar_usage_badges_crowded_narrow_png_snapshot`

  List every such baseline failure in the bead note.

- **Perf.**
  - Run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` via `/sase_monitor`.
  - Record the p50/p95 figures in the closing note, and compare them with the sase-17d.3
    notes.
  - The land agent's run under load average ~36 missed the p95 budget for these tests:
    - clan fold level 1
    - tribe fold level 1
    - fleet `hung_host`, `reconnect_churn` and `event_burst`
    - the link-rail delta
  - The no-cutover baseline failed more under the same load, and `test_bench_axe_jk`
    fails on both trees.
  - Treat an overrun as a cutover regression only if it is worse than the baseline on
    the same scenario.
- **Verify.** Run `just fix`, then `sase tool run check`.

## 5. Verification for every phase

- Run `just fix`, then `sase tool run check`. At planning time `check` stopped at
  `_lint-mypy` and `_lint-symvision` on errors from the command-line and launch-prompt
  work, outside this epic:
  - `command_line/input.py`, `command_line/screen.py`
  - `actions/agent_workflow/_launch_prompt_inputs.py`
  - unused `command_line` symbols

  Confirm that any remaining failure also reproduces without your diff before ignoring
  it.

- Run `pytest -m visual` on every module you touched, and a scoped
  `just fix-tui-screenshots -- <modules>`. Commit the regenerated goldens.
- Run `pytest --collect-only tests/ace/tui/visual`. It must still collect.
- Record `PROPOSED FOLLOW-UP:` notes on your own phase bead. Do not create beads.
