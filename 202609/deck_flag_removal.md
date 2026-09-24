---
tier: tale
size: medium
title: Remove the agent_decks flag and its Off branches
goal:
  Decks are the only Agents-tab runtime path. The agent_decks flag, its registry and
  schema entries, its helpers and every Off branch are gone, flag bead sase-17k is
  closed, and the non-visual tests pass on the deck API. Now-unreachable legacy modules
  are left call-free for legacy-ui-deletion.
proposed_by: bbugyi200.athena.sase-17d.10.1.1
bead: sase-17d.10.1.1
create_time: 2026-09-24 10:38:30
status: wip
---

- **PARENT:**
  [202609/deck_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202609/deck_cutover.md)
- **BEAD:** sase-17d.10.1.1

# Plan: Remove the `agent_decks` flag and its Off branches (phase `deck-flag-removal`, bead sase-17d.10.1.1)

## 1. Context

This is phase `deck-flag-removal` of epic sase-17d.10.1 ("Cut over the Agents tab to
decks and delete the legacy detail UI", plan `plan:202609/deck_cutover.md`, §2). Every
deck phase of epic sase-17d has landed behind the `beta` flag `agent_decks` (flag bead
**sase-17k**, default off). This phase makes decks the only runtime path. It deletes the
flag, makes every On branch unconditional, deletes every Off-branch body, closes
sase-17k, and fixes the non-visual tests.

**Scope boundaries:**

- Whole legacy modules and symbols that become unreachable **stay in place** for the
  next phase, `legacy-ui-deletion` (bead **sase-17d.10.1.2**). That includes the `p`
  picker mixin and `modals/agent_view_modal.py`, the `modals/zoom_panel_*` modules,
  `DetailPanelMode`/`DetailLayoutMode` and their cycle, the section-stop actions, the
  redirect handlers, `#agent-deck-source-host` and its legacy ids, the `view:` chip and
  the `(p)` slow-tools hint mapping. Make each one call-free. Do not rewrite it.
- Do **not** edit `docs/` or `sase/memory/`. Phase sase-17d.11 owns docs. Leave
  `docs/ace.md` and `docs/configuration.md` alone, even though they still mention the
  flag.
- Do **not** regenerate PNG goldens. Phase `cutover-goldens` regenerates all of them.
- Keep the `ace.agent_decks.*` config: `agent_decks_settings.py`, `spread_max_screens`,
  the `ace.agent_decks` schema/default_config entries and
  `test_config_schema_agent_decks_parity`. It is a config field, not the flag.
- No `sase-core` change is needed. This is all presentation-only Textual code.

All paths below are relative to `src/sase/ace/tui/` unless they start with `src/`,
`tests/` or `tools/`.

## 2. Flag, registry and flag bead

1. Delete `FeatureFlag.agent_decks` (the enum member) and its `FeatureFlagDefinition`
   entry in `src/sase/feature_flags/registry.py`.
2. Re-sync the generated block with `tools/sync_feature_flags_schema` (write mode; check
   `--help` for the flag). The `"agent_decks"` property under `properties.feature_flags`
   in `src/sase/config/sase.schema.json` disappears. The separate `ace.agent_decks`
   config property (around line 2200) must stay.
3. Delete `widgets/decks/flag.py` (`agent_decks_enabled`, `agent_decks_active`).
4. Close the flag bead in the same change, as the `sase_flags` removal rule and the epic
   plan require. `sase flag` has no close subcommand, so run:
   `sase bead close sase-17k --note "Removed by sase-17d.10.1.1: registry entry, schema block, flag helpers and every Off branch deleted; decks are unconditional."`
   sase-17k is a flag task bead, not an ancestor of this phase, so closing it is in
   scope.
5. A stale `agent_decks` value in user state/config is already handled by the
   unknown-flag diagnostics and cleanup (`feature_flags/resolver.py`, `snapshot.py`).
   Add no special case.

## 3. Source: make the On branch unconditional

The complete consumer inventory, from
`grep -rn "agent_decks_enabled\|agent_decks_active\|decks_enabled\|_decks_active\|FeatureFlag.agent_decks" src`:

`_app_action_availability.py`, `commands/{_availability_agents,context,types}.py`,
`actions/agents/{_panel_detail,_folding,_metadata_search,_display_detail_info,_display_detail_footer,_deck_layout_actions,_deck_persistence,_deck_search_host,_agent_view_picker}.py`,
`actions/navigation/{_basic,_fold}.py`, `actions/hints/_files.py`,
`actions/clipboard/{_agents,_palette_helpers}.py`,
`widgets/{agent_detail,_agent_detail_state,_agent_detail_panels,_agent_detail_display,_agent_detail_jump,_agent_detail_decks,_agent_detail_deck_targets}.py`
and `widgets/prompt_panel/_agent_slow_tools.py`.

`actions/startup.py`, `actions/_state_init_late.py` and
`actions/_startup_prompt_catalog.py` only touch `AgentDecksSettings`. Leave them alone.

**General rules for every site:**

- `if bool(getattr(x, "decks_enabled", False)): <on>; return` followed by `<off>`
  becomes `<on>`. Delete the Off body.
- `if not decks: return` / `if decks_active: return False` style guards collapse the
  same way.
- `agent_decks_active(app)` also returned False when no `AgentDetail` was mounted. Do
  not re-add a "mounted" probe. Every On branch already queries `#agent-detail-panel`
  inside `try/except` and degrades gracefully. Keep any `current_tab == "agents"` check
  that surrounds the flag check. Example: `_decks_navigation_active` in
  `actions/navigation/_basic.py` becomes `return self.current_tab == "agents"`. Inline
  that wrapper where it reads more simply.
- Delete wrappers that become constant-true, and inline them at their call sites:
  `_panel_detail._decks_active`, `_deck_persistence._decks_persistence_active`, and
  `_deck_layout_actions._decks_layout_active` (which becomes just the tab check).
- Remove every import made dead by a deleted Off body (for example `AgentFilePanel` and
  `AgentLLMCallsPanel` in `agent_detail.py`, and `VerticalScroll`/`NoMatches` where
  unused). Let `ruff` confirm.

**Specific sites:**

- **`widgets/agent_detail.py`**
  - Delete `self._decks_enabled`.
  - `compose` always composes the deck layout: `#agent-header-panel`, then
    `#agent-deck-source-host` (keep it and its `#agent-prompt-scroll` /
    `#agent-search-*` ids for now), then `DeckArea`, then `AgentJumpPanel`. Delete the
    legacy branch that composed `#agent-prompt-scroll.expanded`,
    `#agent-file-scroll`/`AgentFilePanel` and
    `#agent-llm-calls-scroll`/`AgentLLMCallsPanel`.
  - `on_mount` always attaches the main-document sink.
  - `toggle_header_expanded` always calls `reapply_main_view_pins()`.
  - `_panel_mode`/`_detail_layout_mode` state and `_active_metadata_scroll` stay for
    `legacy-ui-deletion`.
- **`widgets/_agent_detail_decks.py`**: delete the `_decks_enabled` class attribute and
  the `decks_enabled` property.
- **`widgets/_agent_detail_deck_targets.py`**: drop the six
  `if not decks_enabled: return None/…` guards.
- **`widgets/_agent_detail_state.py`** (about 13 sites): `show_empty`,
  `show_tribe_summary`, `refresh_current_file`, `cycle_next/prev_file`,
  `on_linked_deltas_refreshed`, `is_llm_calls_visible`, `_llm_calls_panel_or_none`,
  `is_file_visible`, `is_metadata_visible`, `effective_detail_scroll_id`,
  `get_editor_file_info` and `get_current_image_path`. In each, keep the deck branch and
  delete the legacy body.
- **`widgets/_agent_detail_panels.py`** (8 sites) and
  **`widgets/_agent_detail_display.py`** / **`widgets/_agent_detail_jump.py`** (1 site
  each): same collapse.
  - Delete `_expand_prompt_only` and other panel-mode visibility **methods** only if
    their sole callers were deleted Off branches. Symvision only checks top-level defs,
    so dead methods do not fail lint.
  - When in doubt, leave the method for `legacy-ui-deletion`. It deletes the enums and
    their state.
  - Keep the file and LLM Calls event handlers that deck views still use.
- **`actions/agents/_panel_detail.py`**
  - `action_next/prev_agent_file`: the Agents branch becomes a no-op; keep the Services
    chop-run stepping. Keep the method names (the rename belongs to
    `legacy-ui-deletion`).
  - `action_next/prev_deck_card` and `action_next/prev_deck`: guard only on
    `current_tab != "agents"`.
  - `action_zoom_panel`: keep only the in-place deck zoom (`toggle_deck_zoom` plus the
    footer refresh), and delete the `ZoomPanelModal` path.
  - The in-file private zoom helpers (`_zoom_seed_for_tribe`, `_zoom_target_for_detail`,
    `_zoom_seed_from_detail`, `_zoom_border_subtitle`) are now caller-free. Delete them
    here, because they were part of the deleted modal path and would otherwise be dead
    code. Leave the `modals/zoom_panel_*` modules for `legacy-ui-deletion`.
  - Keep `action_toggle_layout` (redirect handler) as is.
- **`actions/agents/_agent_view_picker.py`**
  - `_agent_view_picker_block_reason` now always returns the deck-mode block reason on
    the Agents tab.
  - Delete the flag import; keep the module and mixin wired.
- **`actions/agents/_display_detail_info.py`**
  - `view_mode` is always `""`. Delete the tribe/`panel_mode_label` branches that fed
    it.
  - The node-collapsed/zoomed chip block runs unconditionally.
  - Keep `view_picker_available` and the chip plumbing for `legacy-ui-deletion`.
- **`actions/agents/_metadata_search.py`** (5 sites),
  **`actions/agents/_display_detail_footer.py`** (2), **`actions/agents/_folding.py`**
  (3), **`actions/agents/_deck_search_host.py`**, **`actions/navigation/_fold.py`**,
  **`actions/hints/_files.py`**, **`actions/clipboard/{_agents,_palette_helpers}.py`**:
  keep the deck branch and delete the legacy branch.
  - In `_metadata_search.py`, the deck path routes to per-panel deck search overlays.
    Keep that path, and delete the legacy inline-search body only where it sat on the
    Off side of a flag check.
- **`actions/navigation/_basic.py`**
  - `_decks_navigation_active` becomes the tab check.
  - In `scroll_prompt_down/up`, delete the Agents-tab branch (Services and Artifacts
    stay).
  - Delete the app-wide file/LLM scroll fallbacks that only served the legacy panels.
  - `action_next/prev_agent_metadata_section` keep existing (now unreachable on Agents)
    for `legacy-ui-deletion`.
- **`_app_action_availability.py`**
  - Deck nav/layout actions are available on the Agents tab, with split-only actions
    still gated by `_deck_split_active`.
  - `scroll_prompt_down/up` and `_LEGACY_DECK_KEY_ACTIONS` are unavailable on the Agents
    tab.
- **`commands/types.py`** / **`commands/context.py`**: delete the
  `CommandContext.agent_decks_active` field and its computation.
- **`commands/_availability_agents.py`**
  - `app.choose_agent_view` → `False`.
  - Deck card/deck/split/node-panel entries → `True`.
  - Focus/grow/shrink → `bool(ctx.agent_deck_split)`.
  - `scroll_prompt_*` and the legacy section/file entries → `False` on this Agents
    predicate. Check first that the predicate is Agents-only, so the Services
    `next/prev_agent_file` entries keep working.
  - `app.zoom_panel` → `True`.
- **`widgets/prompt_panel/_agent_slow_tools.py`**
  - `slow_tool_overflow_hint` loses its `decks_enabled` parameter and its Off body. That
    body is the `view_picker_key` clause, so drop that parameter too.
  - The call site stops importing the flag.
  - The `("choose_agent_view", "view_picker")` key mapping in
    `resolve_slow_tool_overflow_keys` may stay for `legacy-ui-deletion`. It is harmless.
- **Comments**
  - `bindings.py:277`: "(agents tab, agent_decks flag on)" → "(agents tab)".
  - `keymaps/registry.py` `_CONTEXTUAL_APP_DUPLICATES`: keep the four pairs
    `next_deck_card`/`next_agent_metadata_section`,
    `prev_deck_card`/`prev_agent_metadata_section`, `next_deck`/`next_agent_file` and
    `prev_deck`/`prev_agent_file`. Change their comment to say the legacy ids are now
    dead Agents bindings awaiting retirement in `legacy-ui-deletion`.
  - Re-word the `toggle_deck_focus`/`scroll_prompt_down` comment from flag-disjoint to
    tab-disjoint: Agents vs Services/Artifacts.

Finally,
`grep -rn "agent_decks_enabled\|agent_decks_active\|decks_enabled\|_decks_active\|FeatureFlag.agent_decks\|decks\.flag" src tests`
must return zero hits.

## 4. Symvision

Symvision only scans top-level defs in `src/sase`.

- Deleting the modal path of `action_zoom_panel` may leave
  `ZoomPanelModal`/`ZoomPanelSeed`/`ZoomPanelTarget` (and anything else the Off branches
  alone imported) without a non-test consumer.
- For each public symbol `just _lint-symvision` reports as unused, add
  `--epic-symbol sase-17d.10.1.2(<symbol>)` to the `_lint-symvision` recipe in the
  `Justfile`. That bead is the `legacy-ui-deletion` phase, which deletes the symbol.
- **Never key an entry to sase-17d.10.1.1.** `sase bead close` refuses while this phase
  has leftovers.
- A private top-level def left dead must be deleted instead. Epic symbols cover public
  defs only.
- The phase notes report pre-existing private-import symvision errors in unrelated
  usage/doctor/plugins-browser files. Compare against a clean `git stash` tree before
  blaming this change, and never "fix" those files here.

## 5. Tests

A trial run of the whole non-visual suite with decks forced on was done while planning
(`agent_decks_enabled` monkeypatched to `True`). Beyond the tests that already fail on a
clean tree, it identified these decks-caused failures. Treat the list as a starting
point, not the full set:

- The trial did not flip the `decks_enabled` getattr default on fake detail objects, or
  the "no mounted AgentDetail" case.
- Run the full suite after the change and fix everything this change causes.

**Delete** each of these, because its subject is flag-off-only behavior or an entry
point that is now unreachable. List every deletion in the bead note.

- `tests/ace/tui/widgets/decks/test_deck_panels.py::test_flag_off_compose_tree`, and
  every other `override_flags(agent_decks=False)` half in `test_deck_panels.py` and
  `test_deck_spread_pilot.py` (`test_flag_off_suite_unchanged`), plus the "off"
  assertions in the `CommandContext` availability tests (lines ~239-243 and ~338).
- `tests/ace/tui/test_agent_view_picker.py`: the 9 tests that fail because `p` is now
  blocked. They are `test_agents_equal_layout_survives_view_changes`,
  `…file_only_layout_and_cycle_escape_to_equal`,
  `…fresh_tab_stays_metadata_only_until_picker_layout`,
  `…llm_calls_only_layout_preserves_across_file_switch`,
  `…llm_calls_panel_message_enables_picker_layouts`,
  `…picker_direct_layout_choices_apply_all_three`,
  `…p_opens_picker_and_direct_mode_choice_applies`,
  `…pp_cycles_visible_file_layout_next` and
  `…p_upper_p_cycles_visible_file_layout_previous`. Keep tests in that file that still
  pass, because `legacy-ui-deletion` deletes the file.
- `tests/ace/tui/test_agents_zoom_panel_modal.py::test_zoom_seed_uses_textual_content_and_paints_file_panel`,
  plus any other zoom-modal test whose failure is only that `Z` no longer opens the
  modal (the seed helpers were deleted).
- `tests/ace/tui/widgets/test_prompt_panel_bottom_pin.py`: 12 tests drive the legacy
  `#agent-prompt-scroll` bottom pin.
  - If a deck Main-view pin test already covers the behavior under
    `tests/ace/tui/widgets/decks/`, delete them.
  - Otherwise migrate the essential cases (growth keeps the view at the end, a relative
    scroll releases the pin) to the focused deck panel's Main scroll.
- `tests/ace/tui/widgets/test_prompt_panel_section_navigation_actions.py`: 3 mounted
  tests of the legacy section-stop actions. Delete them; the actions are unreachable on
  Agents.
- `tests/ace/tui/widgets/test_agent_slow_tools.py`: delete
  `test_slow_tool_overflow_hint_decks_off_with_key`, and drop the `decks_enabled=`
  kwargs and Off assertions from the other hint tests.

**Migrate** each of these to the deck API. Use `show_deck(panel_index, deck)`,
`set_deck_preferred_card`, `deck_area` and the focused-panel accessors in
`widgets/_agent_detail_deck_targets.py`.

- `tests/ace/tui/test_artifacts_limit_keys.py::test_agents_tab_ctrl_j_does_not_rewrite_artifacts_query`:
  it asserts `next_agent_metadata_section` is available on Agents. Assert
  `next_deck_card` instead.
- `tests/ace/tui/test_agents_panel_fold_mounted.py::test_mounted_clan_fold_chords_zoom_and_patch_isolation`:
  it reads `active_section_identity` from the hidden source panel. Read the focused Main
  deck panel's active card/section instead, or drop just that assertion if there is no
  deck equivalent.
- `tests/ace/tui/test_agent_metadata_search.py` (3 tests: commit/repeat/passthrough,
  reverse-key override, yank/frozen refresh): they assert on the legacy
  `#agent-search-command` subtitle. Retarget them to the per-panel deck search overlay
  (see `tests/ace/tui/widgets/decks/test_deck_search_overlay.py`). Delete a test only if
  the overlay already covers its subject.
- `tests/ace/tui/widgets/test_agent_tribe_summary.py::test_tribe_document_invalidates_agent_render_and_uses_prompt_scroll`:
  expect `effective_detail_scroll_id()` to be the deck Main scroll
  (`#agent-deck-panel-0-main-scroll`) and drop the `_panel_mode` assertion.
- `tests/ace/tui/widgets/test_agent_prompt_panel_steps.py::test_update_display_expands_prompt_for_done_workflow_without_diff`:
  it queries `#agent-file-scroll`. Assert the Files deck is empty or not shown instead.
- `tests/ace/tui/widgets/test_agent_header_panel.py`
  (`test_bottom_pinned_body_stays_pinned_across_toggle`,
  `test_secondary_only_keeps_header_visible_and_toggleable`) and
  `tests/ace/tui/widgets/test_agent_jump_panel_{expansion,visibility}.py`
  (`…bottom_pinned_body_stays_pinned_across_jump_toggle`,
  `…panel_sits_below_secondary_scroll_in_every_layout`):
  - Migrate them to a deck split (for example Main|Files through `show_deck`).
  - Delete the "every legacy layout" parametrization and the SECONDARY_ONLY case.

**Flag wrappers and helpers.**

- Remove every `override_flags(agent_decks=True)` wrapper (dedent the body) under
  `tests/ace/tui/widgets/decks/` (`test_deck_panels`, `test_deck_split_pilot`,
  `test_deck_spread_pilot`, `test_deck_action_targets`, `test_deck_collapse_zoom`,
  `test_deck_search_overlay`, `test_deck_split_keys`) and in
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`.
- Drop `override_flags` imports that become unused.
- `test_deck_split_keys.py` (~line 40) and `test_deck_collapse_zoom.py` (~line 125)
  monkeypatch `flag.agent_decks_active`.
  - Delete that helper and its `decks=False` parametrizations, since decks are always
    on.
  - Keep the `decks=True` assertions against a fake app on the Agents tab.
- Delete `assert detail.decks_enabled is True/False` lines.
- Remove `agent_decks_active=` from every `CommandContext(...)` construction.

**Visual modules.** Do not regenerate goldens. Record the stale window (the visual lane
runs only on scheduled Full CI) in the closing note. Every visual module must still
collect: `.venv/bin/pytest --collect-only -q -m visual tests/ace/tui/visual`.

## 6. Verification

1. Grep: the §3 grep returns nothing, and
   `grep -rn "agent_decks" src tests | grep -v "agent_decks_settings\|ace\.agent_decks\|AgentDecksSettings\|agent_decks:\|\"agent_decks\""`
   has no flag hits. Check the remaining hits are only the config field.
2. `tools/sync_feature_flags_schema` in check mode passes, and so does
   `tools/check_feature_flags`.
3. Targeted runs:
   - `tests/ace/tui/widgets/decks/`
   - `tests/ace/tui/widgets/`
   - `tests/ace/tui/test_agent_*.py` and `tests/ace/tui/test_agents_*.py`
   - `tests/test_command_catalog*.py`, `tests/test_keymaps_*.py` and
     `tests/test_command_availability_agents_panels.py`
   - `tests/feature_flags/`
4. `just fix`, then `sase tool run check` (the guarded agent verification recipe). If it
   will take long, hand it to `/sase_monitor` rather than blocking. Read the
   `lint_and_test` memory before finishing, and follow it.
   - Several failures reproduce on a clean tree while planning. Before touching any of
     them, confirm with `git stash` that they are unrelated. Examples:
     `test_app_import_budget`, `test_agent_prompt_panel_monitor`,
     `test_agent_prompt_semantic`, `test_no_ref_prefix_dispatch`,
     `test_dispatch_federation` and
     `test_timezone_display_tui::test_llm_calls_panel_fallbacks_use_configured_wall_time`.
5. Run `sase bead epic-symbols sase-17d.10.1.1`. It must report no entries.
6. Close sase-17k (§2.4).
7. Bead note, then close:
   - Record a bead note on sase-17d.10.1.1 listing:
     - deleted and migrated tests
     - the epic-symbol entries keyed to sase-17d.10.1.2
     - the legacy modules and symbols left for `legacy-ui-deletion`: picker mixin/modal,
       zoom modal modules, the enums and their state, `_active_metadata_scroll`, the
       section-stop actions, the redirect handlers, the source-host ids, the view chip
       plumbing and the `view_picker` hint mapping
     - the stale-goldens window
   - Record any discovered follow-ups as `PROPOSED FOLLOW-UP:` notes.
   - Then close only this phase bead with `sase bead close sase-17d.10.1.1 --note "…"`.
     Do not close sase-17d.10.1, sase-17d.10 or sase-17d.
