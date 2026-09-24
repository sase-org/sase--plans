---
tier: epic
title: Cut over the Agents tab to decks and delete the legacy detail UI
goal: Deck panels become the only Agents-tab detail UI. The agent_decks beta flag
  and every Off branch are removed. The p view picker, the Z zoom modal, the detail
  panel and layout enums, the legacy panel ids and CSS, the view chip, the metadata
  section-stop actions and the retired redirect handlers are deleted. Keymap ids are
  retired or renamed with compatibility aliases, tests use the deck API, and every
  affected PNG golden is regenerated and inspected.
phases:
- id: deck-flag-removal
  title: Remove the agent_decks flag and its Off branches
  depends_on: []
  size: large
  description: 'deck-flag-removal: delete the agent_decks beta flag, make every deck
    On branch unconditional, delete each Off-branch body and the legacy compose branch,
    close flag bead sase-17k, remove the flag override wrappers in tests, and migrate
    or delete non-visual tests that exercised the flag-off UI. Legacy modules left
    unreachable stay in place for legacy-ui-deletion.'
- id: legacy-ui-deletion
  title: Delete the legacy detail UI and retire its keymap ids
  depends_on:
  - deck-flag-removal
  size: large
  description: 'legacy-ui-deletion: delete the p picker, the view modal, the zoom
    modal and its seed and CSS, the panel/layout enums and cycle, the hidden compat
    host ids and legacy CSS, the view chip, the section-stop actions and the redirect
    handlers. Retire choose_agent_view and the section-stop ids, rename next/prev_agent_file
    to next/prev_chop_run with aliases, and delete or migrate the tests of the deleted
    code, including visual test modules.'
- id: cutover-goldens
  title: Regenerate and inspect every affected PNG golden
  depends_on:
  - legacy-ui-deletion
  size: medium
  description: 'cutover-goldens: run the full just fix-tui-screenshots through /sase_monitor,
    inspect every creation, removal and update group, fix any rendering regression
    it reveals, capture live deck screenshots, and run the final j/k perf bench.'
proposed_by: bbugyi200.athena.sase-17d.10
parent_bead: sase-17d.10
create_time: 2026-09-24 10:13:41
status: wip
bead_id: sase-17d.10.1
---

- **PROMPT:** [prompts/202609/deck_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/deck_cutover.md)
- **BEAD:** [sase-17d.10.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.10.1.md)

# Plan: Cut over the Agents tab to decks (phase `deck-cutover` of epic sase-17d)

## 1. Context

This is the `deck-cutover` phase of the epic "Agents tab agent data decks and cards"
(bead `sase-17d`, plan `plan:202609/agents_tab_decks_and_cards.md`). Read that plan's
§3.4, §3.9, §4.7, §14 and §16 before starting any phase here. **That plan's §3.9 is the
source of truth for what is retired, renamed and deleted.** This plan splits its §14
into three phases, because the work is too broad for one agent:

1. `deck-flag-removal`: remove the flag, so decks become the only runtime path.
2. `legacy-ui-deletion`: delete the code that path no longer reaches, and retire its
   keymap ids.
3. `cutover-goldens`: regenerate and inspect goldens, then run live checks and the perf
   bench.

Every earlier epic phase (sase-17d.1 through sase-17d.9) has landed behind the `beta`
flag `agent_decks` (flag bead **sase-17k**). The flag defaults to off, so today's
shipped UI and every existing PNG golden are still the legacy one. The sibling phase
`sase-17d.11` (docs, glossary strands and the post-update key notice) depends on this
work and owns every `docs/` change. **Phases here do not edit `docs/` or `sase/memory/`
files.**

All paths below are relative to `src/sase/ace/tui/` unless they start with `src/`,
`tests/`, `docs/` or `tools/`.

Rust core boundary: all of this is presentation-only Textual state. **No `sase-core`
change is needed.**

### 1.1 Handoff notes from earlier phases (bead notes on sase-17d.3 and sase-17d.6)

- **Temporary flag-on guards (sase-17d.3 "D12").** Each one is a deck-mode
  short-circuit:
  - `p` is blocked in `actions/agents/_agent_view_picker.py`
    (`_agent_view_picker_block_reason`).
  - The palette's `app.choose_agent_view` is forced unavailable in
    `commands/_availability_agents.py` (via `ctx.agent_decks_active`, plumbed through
    `commands/context.py` and `commands/types.py`).
  - The `view:` chip is forced to `""` in `actions/agents/_display_detail_info.py`.
  - The legacy metadata search short-circuits in `actions/agents/_metadata_search.py`.
- **Hidden compat host.** In deck mode, `#agent-deck-source-host` (`display: none`)
  holds the hidden Main source `AgentPromptPanel` inside `#agent-prompt-scroll`, plus
  `#agent-search-scroll`, `#agent-search-panel` and `#agent-search-command`. It exists
  only so legacy queries keep resolving, and this phase's deletion step removes it.
  `_clear/_show/_hide_metadata_scrolls` still query the `#agent-search-*` widgets.
- **Items marked CUTOVER in sase-17d.6's inventory** (flag-off-only paths):
  - `widgets/_agent_detail_panels.py`
  - the picker and its modal
  - `scroll_prompt_down/up` and the section-stop actions
  - the zoom path in `actions/agents/_panel_detail.py`
  - the app-wide file/LLM scroll fallbacks
  - the legacy ids in `styles.tcss`
- **Pre-existing red gates.** Several phase notes report the `just _lint-symvision` gate
  failing on a pristine tree, with private-import errors in unrelated
  usage/doctor/plugins-browser files. Compare against a clean tree before blaming this
  work, and never "fix" those unrelated files here.

## 2. Phase `deck-flag-removal`

Follow the removal rule in the `sase_flags` reference memory: delete the Off branch,
make the On branch unconditional, remove the registry entry, and close the flag bead in
the same change.

- **Flag and registry.**
  - Delete `FeatureFlag.agent_decks` and its definition in
    `src/sase/feature_flags/registry.py`.
  - Re-sync the generated schema block in `src/sase/config/sase.schema.json` with
    `tools/sync_feature_flags_schema`.
  - Close flag bead `sase-17k` with `sase flag` or `sase bead close`, whichever the
    `sase_flags` memory prescribes.
  - `ace.agent_decks.*` config (`agent_decks_settings.py`, `spread_max_screens`) is a
    config field, not the flag. **Keep it.**
- **Flag helpers.**
  - Delete `widgets/decks/flag.py` (`agent_decks_enabled`, `agent_decks_active`).
  - Delete the `AgentDetail._decks_enabled` / `decks_enabled` state and every
    `_decks_active()` / `decks_enabled` / `agent_decks_active` check.
  - Delete `ctx.agent_decks_active` from `commands/context.py` and `commands/types.py`.
  - In every consumer, keep the deck branch and delete the other one. About 40 source
    files call these helpers. Find them all with:

    ```bash
    grep -rn "agent_decks_enabled\|agent_decks_active\|decks_enabled\|_decks_active\|FeatureFlag.agent_decks" src
    ```

    Examples include `_app_action_availability.py`, `commands/_availability_agents.py`,
    `actions/agents/{_panel_detail,_folding,_metadata_search,_display_detail_info,_display_detail_footer,_deck_layout_actions,_deck_persistence,_deck_search_host}.py`,
    `actions/navigation/{_basic,_fold}.py`, `actions/hints/_files.py`,
    `actions/clipboard/{_agents,_palette_helpers}.py`, `actions/startup.py`,
    `actions/_state_init_late.py`, `actions/_startup_prompt_catalog.py`,
    `widgets/{agent_detail,_agent_detail_state,_agent_detail_panels,_agent_detail_display,_agent_detail_jump,_agent_detail_decks,_agent_detail_deck_targets}.py`,
    `widgets/decks/panel_spread.py` and `widgets/prompt_panel/_agent_slow_tools.py`.

- **`AgentDetail.compose`.**
  - Always compose the deck layout. Delete the legacy branch, which composed
    `#agent-prompt-scroll.expanded`, `#agent-file-scroll`/`AgentFilePanel` and
    `#agent-llm-calls-scroll`/`AgentLLMCallsPanel`.
  - `on_mount` always attaches the main-document sink.
  - `toggle_header_expanded` always calls `reapply_main_view_pins()`.
  - Keep `#agent-deck-source-host` and its ids for now; `legacy-ui-deletion` removes
    them.
- **Off-branch bodies.** Delete them inline. Examples:
  - `action_next_agent_file`'s Agents branch (Services chop-run stepping stays)
  - `scroll_prompt_down/up`'s Agents branch (Services and Artifacts stay)
  - the modal path of `action_zoom_panel` (the in-place zoom stays)
  - `_expand_prompt_only` and the panel-mode visibility code whose only callers were Off
    branches
  - the app-wide file/LLM scroll fallbacks

  Whole modules and symbols that become unreachable stay in place for
  `legacy-ui-deletion`. That includes the picker, the view modal, the zoom modal, the
  enums, the section-stop actions and the redirect handlers. Leave each call-free, and
  do not rewrite it.

- **Symvision.** If a now-unreferenced public symbol fails `just _lint-symvision`,
  whitelist it with `--epic-symbol <bead>(<symbol>)` in the Justfile. Key it to the
  `legacy-ui-deletion` phase bead, which will delete it. Never key it to this phase's
  bead: `sase bead close` refuses while this phase has leftovers. Check with
  `sase bead epic-symbols <this phase bead>` before closing.
- **Keymap registry, interim state.**
  - The four flag-disjoint `_CONTEXTUAL_APP_DUPLICATES` pairs in `keymaps/registry.py`
    stay until `legacy-ui-deletion` retires their partner ids, because those ids still
    exist. They are `next_deck_card`/`next_agent_metadata_section`,
    `prev_deck_card`/`prev_agent_metadata_section`, `next_deck`/`next_agent_file` and
    `prev_deck`/`prev_agent_file`.
  - Update their comment to say the legacy ids are now dead Agents bindings awaiting
    retirement.
  - Re-word the `toggle_deck_focus`/`scroll_prompt_down` comment from flag-disjoint to
    tab-disjoint: Agents vs Services/Artifacts.
- **Tests (non-visual).**
  - Remove every `override_flags(agent_decks=True)` wrapper and `SASE_FEATURE_FLAGS`
    `agent_decks` environment setup (about 53 sites under `tests/`, mostly
    `tests/ace/tui/widgets/decks/`), since deck mode is now the default.
  - Delete the "flag off" halves of both-state tests.
  - Run the Agents-tab test directories and fix every failure. For each one:
    - A test that set up visibility through the enums, `_panel_mode`,
      `_detail_layout_mode` or the legacy `#agent-file-scroll`/`#agent-llm-calls-scroll`
      ids migrates to the deck API: `show_deck(panel_index, deck)`,
      `set_deck_preferred_card`, `deck_area` and the focused-panel accessors in
      `widgets/_agent_detail_deck_targets.py`.
    - A test whose subject was flag-off-only behavior is deleted.

    Candidates include `tests/ace/tui/test_agent_detail_two_phase.py`,
    `tests/ace/tui/widgets/test_agent_header_panel.py`,
    `tests/ace/tui/widgets/test_agent_jump_panel_*.py`,
    `tests/ace/tui/widgets/test_agent_tribe_summary.py`,
    `tests/ace/tui/widgets/test_agent_display_workflow_async.py`,
    `tests/ace/tui/widgets/test_prompt_panel_bottom_pin.py`,
    `tests/ace/tui/widgets/file_panel/test_scroll_anchor.py`, `tests/test_file_panel.py`
    and `tests/ace/tui/test_agent_metadata_search.py`.

  - Tests of the picker, view modal and zoom modal whose subject still exists stay until
    `legacy-ui-deletion` deletes that subject. If one fails only because its entry point
    is now unreachable, delete it here and say so in the bead note.

- **Visual tests.**
  - Do not regenerate goldens here. Nearly every Agents golden changes once decks are
    the default, and `cutover-goldens` regenerates and inspects all of them at once.
  - The visual lane runs only on the scheduled Full CI, not on the master gate, so the
    short stale window is acceptable. Record it in the closing note.
  - Visual modules must still import and collect. `mypy` covers only `src`, but pytest
    collection must not break.
- **Verify.**
  - Run `just fix`, then `sase tool run check`, via `/sase_monitor` if long.
  - Run `sase bead epic-symbols` on this phase.
  - Record in the closing note the list of symbols and modules left for
    `legacy-ui-deletion`.

## 3. Phase `legacy-ui-deletion`

Delete everything in the epic plan's §3.9 list, which is now unreachable, and apply its
keymap retirements and renames.

- **Delete these modules and their wiring:**
  - `modals/agent_view_modal.py`
  - `actions/agents/_agent_view_picker.py` and its mixin in the app class
  - `modals/zoom_panel_{modal,content,events,navigation,rendering,search,types,widgets}.py`
  - the `ZoomPanelModal`/`ZoomPanelSeed`/`ZoomPanelTarget` rows in
    `modals/_export_table.py`, `modals/__init__.py` and `modals/__init__.pyi`
  - the `ZoomPanelModal { … }` block in `styles.tcss` (about lines 7579-7705)
  - the zoom seed and target helpers in `actions/agents/_panel_detail.py`.
    `action_zoom_panel` keeps its id and its in-place deck zoom behavior.
  - Fix the stale `ZoomPanelModal` mention in the `src/sase/pager/_screen_body.py`
    docstring.
- **Delete the panel enums.**
  - Delete `DetailPanelMode`, `DetailLayoutMode`, `DETAIL_LAYOUT_CYCLE`,
    `next_detail_layout_mode` and `_MODE_LABELS` from `widgets/_agent_detail_panels.py`.
  - Delete the `_panel_mode`/`_detail_layout_mode` state and the layout and visibility
    methods only they fed, in `agent_detail.py`, `_agent_detail_state.py`,
    `_agent_detail_panels.py` and `_agent_detail_display.py`.
    `AgentDetail.toggle_layout` is one of them.
  - Keep the file and LLM Calls event handlers the deck views still use. Shrink or
    rename `_agent_detail_panels.py` rather than leave an empty shell.
- **Delete the hidden compat host.**
  - Remove `#agent-deck-source-host`'s legacy ids: `#agent-prompt-scroll`,
    `#agent-search-scroll`, `#agent-search-panel` and `#agent-search-command`.
  - The hidden Main source `AgentPromptPanel` must stay mounted, because it builds the
    Main documents. Mount it in a hidden container with a deck-named id.
  - Retarget or delete every remaining query of those ids, including:
    - `_active_metadata_scroll`
    - `_clear/_show/_hide_metadata_scrolls` and `actions/agents/_metadata_search.py`
      (per-panel deck search overlays replaced the legacy search)
    - `actions/navigation/{_basic,_fold}.py`
    - `widgets/file_panel/_content.py` and `widgets/llm_calls_panel.py`
  - Delete the legacy CSS for `#agent-prompt-scroll`, `#agent-file-scroll`,
    `#agent-llm-calls-scroll` and `#agent-search-*`, and the layout-mode classes, from
    `styles.tcss`.
  - Delete any view code on the hidden Main source that no deck path uses, such as
    bottom-pin or section-stop plumbing that only the legacy scroll drove. **Keep the
    section-anchor machinery**, because folds and spread mode use it.
- **Delete the view chip.** Remove the `view:` chip and `_VIEW_MODE_STYLES`
  (`widgets/agent_info_panel.py`, `actions/agents/_display_detail_info.py`), and the
  temporary guards listed in §1.1.
- **Section stops and redirects.**
  - Delete `action_next/prev_agent_metadata_section` (`actions/navigation/_basic.py`).
  - Delete the redirect handlers `action_toggle_layout`, `action_toggle_thinking` and
    `action_toggle_thinking_reverse` (`actions/agents/_panel_detail.py`). Their ids are
    already in `_RETIRED_APP_KEYS`.
- **Keymap retirements and renames** (epic §3.9):
  - Add `choose_agent_view`, `next_agent_metadata_section` and
    `prev_agent_metadata_section` to `_RETIRED_APP_KEYS` (`keymaps/registry.py`).
  - Rename `next_agent_file`/`prev_agent_file` to `next_chop_run`/`prev_chop_run`, which
    are Services-only. Add `LEGACY_APP_KEY_ALIASES` entries
    `next_agent_file → next_chop_run` and `prev_agent_file → prev_chop_run`, with a
    comment naming this epic.
  - Rename the action handlers in `actions/agents/_panel_detail.py`, which keep only
    their Services branch. Consider moving them next to the other Services chop-run
    actions.
  - Update every reference to the renamed ids:
    - `keymaps/app_keymaps.py` and `keymaps/metadata.py` (`_BINDING_META`)
    - `bindings.py`
    - `src/sase/default_config.yml` (the core keymap gotcha)
    - `commands/_app_metadata_nav.py` (Services-only applicability; title "Next job run"
      or similar)
    - `commands/_availability_agents.py` and `_app_action_availability.py`
    - `modals/help_modal/axe_bindings.py` and `widgets/_keybinding_bindings_axe.py`
    - `widgets/axe_onboarding.py`
  - Remove the retired ids everywhere:
    - `app_keymaps.py`, `metadata.py` and `bindings.py`
    - `default_config.yml`
    - `commands/_app_metadata_display.py` and `commands/_app_metadata_nav.py`
    - `commands/_availability_agents.py` and `_app_action_availability.py`
    - `modals/help_modal/agents_bindings.py`: the picker rows and the section-stop row.
      Keep the 57-char boxes and descriptions of 32 chars or fewer.
    - the `("choose_agent_view", "view_picker")` hint in
      `widgets/prompt_panel/_agent_slow_tools.py`, which must no longer suggest `p`
  - In `_CONTEXTUAL_APP_DUPLICATES`, remove the
    `choose_agent_view`/`pick_artifacts_project` pair and the four flag-disjoint legacy
    pairs. Add tab-disjoint pairs `next_deck`/`next_chop_run` and
    `prev_deck`/`prev_chop_run`.
  - In `default_config.yml`, fix the stale "Z zooms agent detail…" comment to describe
    the in-place deck zoom.
  - `scroll_prompt_down/up` have already lost their Agents branch. Update their
    docstrings, palette metadata and help text so none of them mentions the Agents
    prompt.
  - Keep `src/sase/doctor/checks_config_keymap_actions.py` and
    `src/sase/main/config_handler.py` working with the new aliases, and extend their
    tests if they enumerate aliases.
- **Tests.**
  - Delete the tests of deleted code:
    - `tests/ace/tui/test_agent_view_picker.py`
    - `tests/ace/tui/modals/test_agent_view_modal.py`
    - `tests/ace/tui/test_agents_zoom_panel_{action,files,modal,search,tribe}.py`
    - `tests/ace/tui/_agents_zoom_panel_helpers.py`
    - `tests/ace/tui/test_panel_mode_cycle_refresh.py`
    - section-stop action tests
  - Migrate files that only used deleted code for setup, such as
    `tests/test_command_availability_agents_panels.py`,
    `tests/ace/tui/test_agents_jump_to_patches_subtab.py`,
    `tests/ace/tui/test_agents_panel_fold_mounted.py`,
    `tests/ace/tui/widgets/test_agent_info_panel_badges.py` and
    `tests/ace/tui/widgets/_agent_info_panel_helpers.py`.
  - The enforcement tests must pass: `tests/test_command_catalog.py`,
    `tests/test_keymaps_defaults.py`, `tests/test_keymaps_app_bindings.py` and
    `tests/test_command_catalog_guards.py`.
  - Add a test that a user override for `next_agent_file` resolves to `next_chop_run`,
    and that one for `choose_agent_view` is dropped quietly.
  - Remove stale test ids from `tests/shard_timings.json` only if a tool or test
    requires it.
- **Visual test modules.**
  - Delete the modules for deleted features:
    - `tests/ace/tui/visual/test_ace_png_snapshots_agents_zoom.py`
    - `tests/ace/tui/visual/test_ace_png_snapshots_agents_zoom_context.py`
    - `tests/ace/tui/visual/_ace_agents_png_snapshot_zoom_fixtures.py`
    - any picker-modal snapshot tests
  - Migrate modules that import deleted symbols to the deck API:
    `test_ace_png_snapshots_agents_panel_layout.py`, `..._jump_panel.py`,
    `..._llm_calls.py`, `..._proc_shells.py`, `..._sase_context.py`, `..._waiting.py`
    and `tests/ace/tui/visual/_ace_agents_png_snapshot_helpers.py`. A legacy layout
    scenario maps to the matching deck layout, for example metadata+file to a Main|Files
    split. Drop a scenario that has no deck equivalent.
  - Do not regenerate goldens; `cutover-goldens` does. Every visual module must still
    collect: `pytest --collect-only tests/ace/tui/visual`.
- **Verify.**
  - Grep `src/` and `tests/` for every deleted name. Zero hits is required, except
    `_RETIRED_APP_KEYS`/`LEGACY_APP_KEY_ALIASES` entries and their tests:

    ```bash
    grep -rn "DetailPanelMode\|DetailLayoutMode\|ZoomPanel\|choose_agent_view\|agent_view_modal\|next_agent_metadata_section\|next_agent_file\|agent-prompt-scroll\|agent-file-scroll\|agent-llm-calls-scroll\|agent-search-scroll\|agent-deck-source-host"
    ```

  - Run `just fix`, then `sase tool run check`, via `/sase_monitor` if long.
  - Resolve every `--epic-symbol` entry that `deck-flag-removal` keyed to this phase,
    then confirm with `sase bead epic-symbols`.
  - Run a live check with `sase screenshot` of the Agents tab, including a split and a
    zoom, and inspect the PNGs.

## 4. Phase `cutover-goldens`

- **Run the goldens.** Run the full `just fix-tui-screenshots` through `/sase_monitor`.
  It takes about 10 minutes and must not block the turn.
- **Inspect.**
  - Open the report and inspect every creation, removal and update group before
    finalizing. Look at each changed Agents PNG, not a sample.
  - Expected changes:
    - Agents-tab goldens now show the DeckArea: a MAIN tab-strip title, the deck
      subtitle and empty states.
    - Help-modal goldens lose the picker and section-stop rows.
    - Services help and onboarding goldens show the renamed chop-run keys.
    - Zoom-modal and picker goldens are removed.
  - Check each for layout, focus styling, tab strips, spread separators, empty states,
    the collapsed spine and chips, and that no `view:` chip or `(p)` hint remains.
- **Fix regressions.** If a golden shows a real regression, fix the source and
  regenerate only that scope with `just fix-tui-screenshots -- <selector>`. Examples are
  a clipped card, a missing Reply, or a wrong empty state where content should exist.
- **Add coverage.** Add the goldens that earlier phases proposed as follow-ups, if they
  are still missing:
  - spread Main, spread Files and paged-after-threshold (sase-17d.8 notes)
  - a LEFT_RIGHT committed-search overlay (sase-17d.6 notes)
- **Live screenshots.** Capture live `sase screenshot` PNGs of: single Main, a
  Main|Files split, Context|Reply, a collapsed node panel, and a zoomed panel. Inspect
  them.
- **Perf.**
  - Run `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` via `/sase_monitor`.
  - Record p50/p95 for SINGLE and LEFT_RIGHT in the closing note. Compare them with the
    sase-17d.3 notes (#2 flag off, #3 flag on).
  - Treat a p95 over 16 ms as a regression only if it is worse than those baselines on
    the same scenario. Otherwise note host noise.
- **Verify.** Run `just fix-tui-screenshots --check` clean, then `sase tool run check`.

## 5. Risks and decisions

- **Why three phases.** Deleting the Off branches first makes decks the runtime path for
  users at once. That is the intended end state, and it avoids an interim where flag-off
  users lose `p` or the section stops before decks are on. Leaving the now-dead legacy
  modules for the next phase keeps each diff reviewable. Symvision epic-symbol
  whitelisting keyed to `legacy-ui-deletion` is the sanctioned bridge.
- **Stale goldens between phases.** They are accepted because the master gate does not
  run the visual lane. `cutover-goldens` must land before the parent epic's land agent
  runs `just check-full`.
- **Hidden Main source.** The prompt panel stays mounted and hidden. Removing it would
  mean re-implementing Main document building, which is out of scope.
- **User configs.**
  - Stale overrides for retired ids are dropped quietly (`_RETIRED_APP_KEYS`), and
    renamed ids resolve through `LEGACY_APP_KEY_ALIASES`.
  - A stale `agent_decks` flag value in user state or config is handled by the existing
    unknown-flag diagnostics and cleanup (`src/sase/feature_flags/resolver.py`,
    `snapshot.py`). Do not add a special case.
- **Out of scope.**
  - `docs/` rewrites, glossary strands and the post-update key notice all belong to
    sase-17d.11.
  - Closing the parent epic sase-17d belongs to its land agent. Record discovered
    follow-ups as `PROPOSED FOLLOW-UP:` bead notes.
