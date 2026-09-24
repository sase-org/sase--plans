---
tier: tale
title: Delete the legacy Agents detail UI and retire its keymap ids
goal: The Agents tab has one deck-only detail model, with legacy UI code removed and
  stale user keymaps handled compatibly.
size: medium
proposed_by: bbugyi200.athena.sase-17d.10.1.2
bead: sase-17d.10.1.2
status: done
---

- **BEAD:**
  [sase-17d.10.1.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.10.1.2.md)

# Delete the legacy Agents detail UI and retire its keymap ids

## Objective

Complete phase bead `sase-17d.10.1.2` by removing the now-unreachable pre-deck Agents
detail UI, simplifying the surviving deck-only paths, and preserving user keymap
compatibility through explicit retirement and rename handling. The result must have one
Agents detail model: the deck area, with a hidden Main-document source and no picker,
zoom modal, panel/layout modes, legacy metadata-search host, view chip, or section-stop
actions.

This is presentation-only Textual work. Do not change `sase-core`. Do not edit `docs/`
or `sase/memory/`; those belong to `sase-17d.11`. Do not regenerate PNG goldens; phase
`sase-17d.10.1.3` owns the complete regeneration and inspection. Delete or migrate
visual test modules so they still collect, but leave the expected stale-golden window
for that next phase. Close only `sase-17d.10.1.2`, never its parent or ancestors, and do
not create follow-up beads from this phase.

## Current state and invariants

- The `agent_decks` feature flag and every live Off branch have already been removed by
  `sase-17d.10.1.1`; decks are unconditional. Remaining legacy branches are dead code,
  not alternate behavior to preserve.
- Keep `AgentPromptPanel#agent-prompt-panel` mounted and hidden as the single Main
  document builder. It still owns builders, workers, timers, generation guards, and the
  identity, jump, and Main-document sinks. Rename its host to a deck-specific id, but do
  not remove this source.
- Keep section-anchor and section-tracking machinery because folds and spread mode use
  it. Remove only section-stop actions and hidden legacy-scroll plumbing.
- Keep `action_zoom_panel`; its only behavior is the existing in-place deck zoom.
- Keep file and LLM Calls events or data that current deck views consume. Remove the
  panel-mode/layout state and old scroll visibility code only after checking current
  consumers.
- The default keymap remains the source of truth. Every action-id removal or rename must
  stay consistent across the dataclass, YAML defaults, binding metadata, fallback
  bindings, palette catalog/availability, contextual-duplicate registry, help rows,
  onboarding/footer displays, and enforcement tests.

## 1. Remove legacy picker and modal implementations

1. Delete `actions/agents/_agent_view_picker.py`, `modals/agent_view_modal.py`, and
   their app-mixin/import/export wiring.
2. Delete all `modals/zoom_panel_*.py` modules and remove `ZoomPanelModal`,
   `ZoomPanelSeed`, and `ZoomPanelTarget` from `modals/_export_table.py`,
   `modals/__init__.py`, and `modals/__init__.pyi`.
3. Delete the legacy zoom seed/target helpers from `actions/agents/_panel_detail.py`,
   while retaining the deck-only `action_zoom_panel` implementation. Remove the retired
   redirect handlers `action_toggle_layout`, `action_toggle_thinking`, and
   `action_toggle_thinking_reverse` rather than forwarding them.
4. Remove the complete `ZoomPanelModal` TCSS block and fix the stale modal mention in
   `src/sase/pager/_screen_body.py`.
5. Delete the corresponding unit-test helpers and test modules whose subjects no longer
   exist: picker/modal tests, zoom modal/action/files/search/tribe tests, and
   panel-mode-cycle tests. Preserve a test only when it exercises a surviving deck
   behavior, and migrate that test to the deck API rather than importing deleted
   symbols.

## 2. Collapse AgentDetail to its deck-only composition and state

1. Delete `DetailPanelMode`, `DetailLayoutMode`, `DETAIL_LAYOUT_CYCLE`,
   `next_detail_layout_mode`, `_MODE_LABELS`, `_panel_mode`, and `_detail_layout_mode`.
   Remove their layout cycling, selected-secondary, visibility, compatibility, and
   class-application methods from `agent_detail.py`, `_agent_detail_state.py`,
   `_agent_detail_panels.py`, and `_agent_detail_display.py`, including
   `AgentDetail.toggle_layout`.
2. Shrink or rename `_agent_detail_panels.py` around the event handlers and indicator
   updates that deck views still use. Retain no empty compatibility shell, and do not
   remove file/LLM Calls event handling merely because its old scroll-based rendering is
   gone.
3. Replace `#agent-deck-source-host` and its nested legacy scroll/search widgets with
   one hidden, deck-named Main-source container holding
   `AgentPromptPanel#agent-prompt-panel`. Remove `#agent-prompt-scroll`,
   `#agent-search-scroll`, `#agent-search-panel`, and `#agent-search-command`.
4. Simplify `actions/agents/_metadata_search.py` to the unconditional per-panel deck
   search overlay. Remove `_deck_search_active` forks, native metadata overlay
   fallbacks, legacy layout-class copying, and every fallback query of the deleted ids.
   Apply the same deck-only simplification to navigation and fold actions and to any
   remaining fallback in `widgets/file_panel/_content.py` and
   `widgets/llm_calls_panel.py`.
5. Delete legacy TCSS for the removed prompt/file/LLM/search scroll ids and the old
   layout-mode classes. Preserve deck panel styling and the hidden Main source.
6. Remove the `view:` chip and `_VIEW_MODE_STYLES` from `widgets/agent_info_panel.py`
   and `actions/agents/_display_detail_info.py`, along with now-dead temporary guards.
   Keep the node-collapse/zoom info chip supplied by the deck UI.

## 3. Retire and rename keymap actions compatibly

1. Add `choose_agent_view`, `next_agent_metadata_section`, and
   `prev_agent_metadata_section` to `_RETIRED_APP_KEYS`. Remove their `AppKeymaps`
   fields, default-config entries, fallback bindings, binding metadata, command-catalog
   rows, availability branches, help rows, and slow-tool `view_picker` hint. A stale
   user override for any retired id must be discarded quietly by the existing retirement
   loader.
2. Rename `next_agent_file` and `prev_agent_file` to `next_chop_run` and `prev_chop_run`
   everywhere. Rename the handlers to `action_next_chop_run`/`action_prev_chop_run`,
   keep only the Services job-run stepping behavior, and update titles/descriptions to
   say “job run” rather than “agent file”.
3. Add `LEGACY_APP_KEY_ALIASES` entries from `next_agent_file` to `next_chop_run` and
   from `prev_agent_file` to `prev_chop_run`, with a comment naming this epic. Ensure
   config/doctor loading translates old overrides before constructing the current
   registry.
4. Update all consumers of the renamed ids: `keymaps/app_keymaps.py`,
   `keymaps/metadata.py`, `bindings.py`, `src/sase/default_config.yml`,
   `commands/_app_metadata_nav.py`, command availability and app action availability,
   Services help, `_keybinding_bindings_axe.py`, and `widgets/axe_onboarding.py`.
5. In `_CONTEXTUAL_APP_DUPLICATES`, remove the picker/project pair and the four legacy
   deck/metadata/file pairs. Add only the surviving tab-disjoint
   `next_deck`/`next_chop_run` and `prev_deck`/`prev_chop_run` pairs. Retain the
   existing deck-card/Artifacts and deck-focus/other-tab pairs.
6. Update `scroll_prompt_down`/`scroll_prompt_up` docstrings, palette scope/titles, and
   help text so they describe only their surviving Services/Artifacts behavior. Fix the
   default-config comment for `Z` to describe in-place deck zoom.
7. Add regression tests showing that old `next_agent_file` and `prev_agent_file` user
   overrides populate the new chop-run actions, while old picker and section-stop
   overrides are dropped quietly. Update command-catalog, default/fallback parity,
   registry loading, validation, action availability, Services navigation, help, and
   onboarding expectations to use only current ids.

## 4. Migrate surviving tests and visual test modules

1. Migrate tests that used the removed modes or legacy widget ids only as setup to
   `show_deck`, `set_deck_preferred_card`, `deck_area`, and focused-panel accessors. In
   particular, review command availability, jump-to-patches, mounted fold,
   `AgentInfoPanel`, two-phase display, metadata search, file-scroll-anchor, header,
   jump-panel, tribe/workflow, and bottom-pin tests identified by the retirement grep.
2. Delete section-stop action tests while preserving lower-level section-anchor/fold
   tests. Delete stale `tests/shard_timings.json` entries only if collection or its
   validating tool requires it.
3. Delete the legacy zoom/picker visual test modules and fixtures. Migrate visual
   modules that import legacy enums/ids to equivalent deck layouts, including panel
   layout, jump panel, LLM Calls/Tools, proc shells, SASE context, waiting, and their
   shared helper. Map a legacy metadata+file split to Main|Files; drop cases with no
   deck equivalent.
4. Run visual collection only in this phase. Do not update, add, delete, or approve PNG
   golden files; `sase-17d.10.1.3` handles the full golden report after this phase
   lands.

## 5. Verification and close protocol

1. Sweep `src/` and `tests/` for all removed names and ids. Require zero runtime hits
   for `DetailPanelMode`, `DetailLayoutMode`, `ZoomPanel`, the view-picker modules,
   section-stop actions, legacy widget ids, and `agent-deck-source-host`. The only
   allowed old action-id occurrences are the three `_RETIRED_APP_KEYS`, the two
   `LEGACY_APP_KEY_ALIASES` keys, and tests explicitly proving those compatibility
   behaviors.
2. Run focused tests while iterating, then at minimum run the command/keymap enforcement
   tests, affected Agents and Services TUI tests, and
   `pytest --collect-only tests/ace/tui/visual`.
3. Read the project `lint_and_test.md` reference memory before final verification. Run
   `just fix`, then the normal guarded verification with `sase tool run check`. Use
   `/sase_monitor` for commands that may outlive a provider turn. Do not run
   `just check-full`.
4. Capture the live Agents tab with `sase screenshot`, including a split and in-place
   zoom, and inspect the PNGs for the Main source/deck layout, focus, tab strips,
   search/scroll behavior, and absence of the `view:` chip or `(p)` picker hint.
5. Run `sase bead epic-symbols sase-17d.10.1.2`. Resolve every entry owned by this
   phase; if a symbol must survive for later work, re-key its Justfile line to the
   still-open parent epic or a later phase before continuing. Re-run until no entries
   remain for this bead.
6. If out-of-scope work is discovered, record it only as
   `PROPOSED FOLLOW-UP: <one-line summary — detail>` on this bead. Then close only
   `sase-17d.10.1.2` with
   `sase bead close sase-17d.10.1.2 --note "<what was removed, migrated, and verified; include check, visual collection/live capture, and epic-symbol results>"`.
