---
tier: tale
title: Retarget Agents detail actions to the focused deck panel
goal:
  With agent_decks on, every Agents detail action (folds, v hints, ,/ search, E, l/h/L/H
  LLM Calls detail levels, clipboard file paths, footer, palette context, auto-refresh,
  d/. pin re-apply) targets the focused deck panel. A per-panel search overlay replaces
  the legacy metadata search. The SLOW TOOL CALLS overflow hint is accurate in both flag
  states, and flag-off behavior is otherwise unchanged.
size: medium
proposed_by: bbugyi200.athena.sase-17d.6
bead: sase-17d.6
create_time: 2026-09-24 08:36:43
status: wip
---

- **PARENT:**
  [202609/agents_tab_decks_and_cards.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_decks_and_cards.md)
- **BEAD:**
  [sase-17d.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.6.md)

# Plan: Retarget Agents detail actions to the focused deck panel (`deck-action-retarget`, bead sase-17d.6)

## Context

Epic `sase-17d` (plan `plan:202609/agents_tab_decks_and_cards.md`, §10 is this phase)
replaces the Agents-tab metadata panel and its Files / LLM Calls panels with one or two
**deck panels** behind the `agent_decks` beta flag (flag bead `sase-17k`). Earlier
phases landed the deck package (`src/sase/ace/tui/widgets/decks/`), the hidden Main
source (`AgentPromptPanel#agent-prompt-panel.-deck-source`, which still runs every
builder and feeds `MainDeckView`s through the main-document sink), Ctrl+J/K / Ctrl+N/P,
splits, Ctrl+F focus and the `{`/`}` ratio. Scrolling (Ctrl+D/U, g/G, bottom pin)
already targets the focused panel through `AgentDetail.effective_detail_scroll_id()`.

What is still wrong with the flag on is every remaining detail action that looks up a
legacy id (`#agent-file-panel`, `#agent-llm-calls-panel`, `#agent-prompt-scroll`,
`#agent-search-scroll`) or treats "LLM Calls visible anywhere" as "the keys act on LLM
Calls". This phase gives each one a deck-mode path. **Flag-off behavior must stay
byte-for-byte unchanged**, except the SLOW TOOL CALLS overflow text (below).

All paths below are relative to `src/sase/ace/tui/` unless they start with `src/`,
`tests/` or `docs/`. Everything here is presentation-only Textual state; **no
`sase-core` change is needed**. Other epic phases (`node-panel-collapse-zoom`,
`deck-spread-mode`) may be landing in parallel and touch the same files. Keep edits
narrow, rebase carefully, and do not change `Z`/zoom behavior; that belongs to
`node-panel-collapse-zoom`.

## Consumer inventory (the phase deliverable; record it in the bead's closing note)

Disposition key: **DONE** = already has a correct deck path; **THIS** = fixed in this
phase; **CUTOVER** = only reachable with the flag off, so it is deleted by
`deck-cutover`.

| Consumer                                                                                                                           | Disposition                                                                      |
| ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `widgets/_agent_detail_panels.py` (enums, layout/panel mode, legacy message handlers)                                              | DONE (deck-gated) / CUTOVER                                                      |
| `actions/agents/_agent_view_picker.py`, `modals/agent_view_modal.py` (`p`)                                                         | DONE (disabled in deck mode) / CUTOVER                                           |
| `widgets/_agent_detail_state.py` `is_file_visible` / `is_llm_calls_visible` / `is_metadata_visible` / `effective_detail_scroll_id` | DONE; "visible" means "any visible panel shows that deck with content"           |
| `widgets/_agent_detail_state.py` `refresh_current_file`                                                                            | THIS: refresh every panel showing Files, not the focused panel                   |
| `widgets/_agent_detail_state.py` `get_editor_file_info`, `get_current_image_path`                                                  | THIS: Main branch for `E`; image path goes through the new Files-view resolver   |
| `widgets/_agent_detail_state.py` `_llm_calls_panel_or_none`                                                                        | THIS: key actions use the focused Tools view only                                |
| `widgets/_agent_detail_display.py` (`update_display_with_hints`, header summary reads)                                             | DONE: drives the hidden source, and the sink fans out to Main views              |
| `widgets/agent_detail.py` compose / `on_mount` / `toggle_header_expanded` (`d`)                                                    | DONE; the deck-mode `#agent-search-*` widgets are removed in THIS phase (step 3) |
| `widgets/_agent_detail_jump.py` sink attach                                                                                        | DONE (hidden source)                                                             |
| `widgets/_agent_detail_jump.py` `toggle_jump_panel_expanded` (`.`)                                                                 | THIS: re-apply the pin on the visible Main views                                 |
| `widgets/_agent_detail_decks.py` `_deck_source_panel`                                                                              | DONE (hidden source)                                                             |
| `actions/navigation/_basic.py` scroll / top / bottom actions                                                                       | DONE (earlier phase)                                                             |
| `actions/navigation/_basic.py` `scroll_prompt_*`, section-stop actions                                                             | DONE (gated off in deck mode) / CUTOVER                                          |
| `actions/navigation/_fold.py` `_current_agent_metadata_section_id` (`za`/`zA`)                                                     | THIS                                                                             |
| `actions/navigation/_fold.py` `_selected_lane_has_*` (read the header-summary cache)                                               | DONE: reads the hidden source's cache, deck-safe                                 |
| `actions/agents/_metadata_search.py` (`,/` search)                                                                                 | THIS: per-panel overlay                                                          |
| `actions/agents/_panel_detail.py` zoom target/seed                                                                                 | DONE (Z disabled in deck mode; `node-panel-collapse-zoom` owns it) / CUTOVER     |
| `actions/agents/_panel_detail.py` `action_edit_panel` (`E`)                                                                        | THIS (through `get_editor_file_info`)                                            |
| `actions/agents/_folding.py` `_route_llm_calls_detail_level` + `h`/`L` actions                                                     | THIS                                                                             |
| `actions/clipboard/_agents.py` `_copy_file_path`, `actions/clipboard/_palette_helpers.py`                                          | THIS                                                                             |
| `actions/clipboard/_core.py` copy footer                                                                                           | DONE (via `is_file_visible`)                                                     |
| `actions/agents/_display_detail_footer.py` `file_visible`                                                                          | DONE                                                                             |
| `actions/agents/_display_detail_footer.py` `llm_calls_visible`                                                                     | THIS: focused deck is Tools                                                      |
| `commands/context.py` `_file_panel_visible`                                                                                        | DONE (any panel shows Files)                                                     |
| `actions/event_refresh/_auto_refresh_surfaces.py`                                                                                  | DONE for the gate; its `refresh_current_file` call is fixed by THIS              |
| `widgets/file_panel/_content.py`, `widgets/llm_calls_panel.py` app-wide scroll fallback                                            | CUTOVER (only reached when the parent is not a scroll)                           |
| `widgets/prompt_panel/_agent_slow_tools.py` overflow hint                                                                          | THIS                                                                             |
| `src/sase/ace/testing/prompt_document.py`                                                                                          | DONE (reads the hidden source)                                                   |
| `styles.tcss` legacy ids                                                                                                           | CUTOVER; new overlay CSS added in THIS                                           |

Before closing, re-run the grep from epic §10 over `src/sase/`, and confirm no consumer
is missing from this table. Add any that are.

## Implementation

### 1. Deck resolvers on `AgentDetail`

To keep `_agent_detail_state.py` under the ~500-line `toobig` limit, add a small mixin
module `widgets/_agent_detail_deck_targets.py`, mixed into `AgentDetail`. It holds pure
lookups over `deck_area`, and every method returns `None` / `False` with the flag off:

- `focused_deck() -> DeckId | None`
- `focused_file_view() -> AgentFilePanel | None`. This is the view that key actions and
  clipboard targets use. Pick the focused panel's Files view when the focused deck is
  Files. Otherwise pick the first visible panel showing Files. Otherwise `None`.
- `focused_tools_view() -> AgentLLMCallsPanel | None`. Return a view only when the
  focused deck is Tools; there is no fallback.
- `main_view_for_actions() -> tuple[DeckPanel, MainDeckView] | None`. Pick the focused
  panel when it shows Main. Otherwise pick the other visible panel that shows Main.
  Otherwise `None`.
- `ensure_main_deck_shown() -> None`. When no visible panel shows Main, call
  `show_deck(focused_index, DeckId.MAIN)`.
- `reapply_main_view_pins() -> None`. For each visible panel's `main_view` that
  `is_pinned_to_bottom`, call `_schedule_bottom_pin_reapply()`. Refactor
  `toggle_header_expanded`'s deck branch (`widgets/agent_detail.py`) to use it.

Flag-off legacy branches stay where they are.

### 2. Rewire the simple consumers

- **Clipboard.** `actions/clipboard/_agents.py::_copy_file_path` and
  `actions/clipboard/_palette_helpers.py::warm_agent_file_path`: in deck mode use
  `focused_file_view()`. `None` gives the existing "File panel is not visible" warning /
  `None`. With the flag off, keep `#agent-file-panel`.
- **Image path.** `get_current_image_path` (deck branch) uses `focused_file_view()`.
- **Auto-refresh.** Change the deck branch of `refresh_current_file` so it calls
  `refresh_file(agent)` on the `file_view` of every panel in
  `deck_area.panels_showing(DeckId.FILES)`. Duplicate views rely on the existing
  in-flight dedupe. Add a test that the diff is not fetched twice. It currently
  refreshes the focused panel even when that panel shows Main.
- **`.` jump toggle.** `widgets/_agent_detail_jump.py::toggle_jump_panel_expanded`: in
  deck mode call `reapply_main_view_pins()` instead of pinning the hidden source.
- **`E`.** In the deck branch of `get_editor_file_info`, add Main. When the focused deck
  is Main, return `(None, renderable_to_text(Group(*card.renderables)), ".md")` for the
  focused panel's active card
  (`panel._main_document.card(panel.main_view.active_card_id)`; add a public
  `DeckPanel.active_main_card()` accessor instead of reaching into privates). This
  follows `modals/zoom_panel_content.py::editor_info`'s Main behavior. Files (real path,
  else diff content) and Tools (LLM Calls text as `.md`) keep their existing deck
  branches.
- **LLM detail levels (`l`/`h`/`L`/`H`).** In `actions/agents/_folding.py`:
  - `_route_llm_calls_detail_level` gates on `focused_tools_view() is not None` in deck
    mode (`agent_decks_active(self)` / `detail.decks_enabled`). It keeps
    `is_llm_calls_visible()` with the flag off.
  - `_llm_calls_panel_or_none` (deck branch) returns `focused_tools_view()`.
  - Deck mode only: `action_hooks_or_collapse` routes `"collapse"` first, and
    `action_expand_all_folds` routes `"max"` **before** its Agents
    `toggle_selected_agent_panels` branch. This fixes today's unreachable `h`/`L`.
    Flag-off ordering stays unchanged.
- **Footer.** In `actions/agents/_display_detail_footer.py`, the deck-mode
  `llm_calls_visible` becomes `focused_tools_view() is not None`, so the "more/less
  detail" chips match what `l`/`h` actually do. `llm_calls_detail_level` reads the same
  view.

### 3. Per-panel search overlay (`,/`)

- **DeckPanel compose.** Add a hidden overlay to `DeckPanel.compose`:
  `VerticalScroll(id=f"agent-deck-panel-{i}-search-scroll", classes="deck-search-scroll")`
  holding a `Static(classes="deck-search-panel")`, plus a
  `Static(classes="deck-search-command")` below it. Both start without `-shown`. Add CSS
  in `styles.tcss` next to the existing `.deck-scroll` rules:
  - hidden unless `-shown`
  - the command line is one bordered row with a `search` border title, like
    `#agent-search-command`
- **DeckPanel API.** Add `show_search_overlay()` / `hide_search_overlay()` (hide the
  active deck scroll and the empty state, show the overlay; restore through
  `set_deck(self._deck)`) and `search_scroll()`, `search_panel()` and `search_command()`
  accessors. Showing or hiding must not remount anything.
- **Corpus.** Add a module `widgets/decks/search_corpus.py` with
  `deck_search_corpus(panel) -> str`, salvaged from
  `modals/zoom_panel_search.py::vim_search_corpus`:
  - **Main:** every card in document order. Each card is a title separator line
    (`── Context ──`) followed by `renderable_to_text(Group(*card.renderables))`, so
    search sees the text of all cards.
  - **Files:** the active card: `file_view.get_current_content()`, else its
    `_full_content` string. Add a public accessor on the file panel rather than reading
    the private attribute.
  - **Tools:** `tools_view.get_llm_calls_text() or ""`.
  - **Empty deck:** `""`, which triggers the controller's existing "no matches"
    feedback.
- **Host routing.** In `actions/agents/_metadata_search.py`, the deck path goes in a new
  sibling module (for example `actions/agents/_deck_search_host.py`) so the mixin stays
  under the size limit:
  - `_agent_metadata_search_can_start`: in deck mode, return True when the focused panel
    is visible. Drop the early `return False`.
  - On start, capture the target `DeckPanel` (`self._agent_metadata_search_panel`).
    Every host-protocol method (`vim_search_corpus`, `origin_scroll`,
    `overlay_viewport`, `show/hide_overlay`, `paint_overlay`, `command_width`,
    `paint_command_line`, `scroll_overlay`, `restore_scroll`, `focus_overlay`,
    `focus_native`, and `_scroll_agent_metadata_search`) dispatches to that panel's
    widgets when a panel is captured, and to the legacy ids otherwise.
  - Origin and restore scroll use `panel.active_scroll()`. Clear the captured panel in
    `vim_search_exited`.
  - Today's semantics are unchanged: typing / committed modes, `n`/`N`, the reverse key,
    `y`/`Y` yank, `q`/esc, frozen corpus, and identity-change exit.
- **Structural exits.** In deck mode, committed search passes `passthrough_exit_keys`
  built from the **live keymap** for `next_deck`, `prev_deck`, `next_deck_card`,
  `prev_deck_card`, `toggle_deck_split_below`, `toggle_deck_split_right`,
  `toggle_deck_focus`, `grow_deck_panel`, `shrink_deck_panel` and `edit_panel`. Use
  `split_key_alternatives` on `self._keymap_registry.app.<id>`. These keys tear the
  overlay down and still run their normal action, following the zoom modal's
  `_STRUCTURAL_SEARCH_EXIT_KEYS` idea. Flag off still passes `None`.
- **Legacy widgets.** Remove the deck-mode `#agent-search-scroll`, `#agent-search-panel`
  and `#agent-search-command` from the deck branch of `AgentDetail.compose`, after
  confirming that no deck-mode code path queries them. Grep and run the deck tests. If
  something still needs them, leave them and list them as CUTOVER instead.
- **Palette label.** Leave the `search_forward` palette label ("Search metadata
  forward") alone; `deck-cutover`/docs rename it.

### 4. Folds (`za`/`zA`)

In `actions/navigation/_fold.py::_current_agent_metadata_section_id`, deck mode uses
`main_view_for_actions()`:

- `row = max(0, int(scroll.scroll_y) - view.virtual_region.y)` with
  `scroll = view.parent`. Return
  `view.resolve_section_at_row(row, width=view.size.width)`.
- With no Main view shown, return `None` and notify once: "Show the Main deck to fold a
  section". Fold-level keys (`z1…`, `zz`, `zM`/`zR` equivalents) stay document-global
  and unchanged.
- Verify in a pilot test that the section ids published by `MainDeckView` match those of
  the source panel for the same card. They come from the same renderables. Also verify
  that the fold re-render (source → sink → view) keeps the view on that section.

### 5. Hints (`v`)

In `actions/hints/_files.py::_view_agent_files_impl`, before mounting the hint bar in
deck mode, call `agent_detail.ensure_main_deck_shown()`. Hint rendering already flows
through `update_display_with_hints` → hidden source → sink → Main views. Nothing else
changes. Hints in the Reply card show when that card is active.

### 6. SLOW TOOL CALLS overflow hint

`widgets/prompt_panel/_agent_slow_tools.py:243-250` hard-codes "press ] for the full LLM
Calls timeline". `]` no longer does that.

- Add a pure helper
  `slow_tool_overflow_hint(overflow, *, decks_enabled, next_deck_key, prev_deck_key, view_picker_key) -> str`:
  - decks on: `+ N more · ^N/^P → Tools deck for the full LLM Calls timeline`
  - decks off: `+ N more · p → LLM Calls view for the full timeline`
  - with no key known, drop the key clause: `+ N more · full timeline in LLM Calls`
- Key text uses `key_display_name` of the live keymap ids `next_deck`/`prev_deck` and
  `choose_agent_view`.
- Thread an optional `overflow_hint_keys` (or a pre-resolved callable) parameter through
  `append_slow_tool_calls_section` and `append_slow_tool_calls_section_no_fold_owner`,
  with a keymap-free default. Resolve the keys in the `AgentPromptPanel` builder callers
  (`_agent_display_header.py`, `_workflow_render.py`, and any clan-member caller) from
  `self.app._keymap_registry`, failing open. Builders must stay free of geometry and
  I/O.
- The flag is read via `agent_decks_enabled()` at render time, not at import. The text
  changes only when the keymap or flag changes, so digests stay stable otherwise.
- Update `tests/ace/tui/widgets/test_agent_slow_tools.py`. The two `press ]` assertions
  become the new strings, and add a unit test per branch of the helper.

## Tests (all flag-on tests use `override_flags(agent_decks=True)`)

New tests go under `tests/ace/tui/widgets/decks/`, split by topic, with each file under
the size limit. Reuse the `_DetailApp` / `make_artifact_agent` pattern from
`test_deck_split_pilot.py`, and use full-app pilots where an action lives on the app.

- **Resolvers:** `focused_file_view` / `focused_tools_view` / `main_view_for_actions` in
  SINGLE and in both splits, with every focus combination.
- **Clipboard:** `%E` copies the path from the focused Files view, falls back to the
  other visible Files panel, and warns when no panel shows Files.
- **Auto-refresh:** refreshes only panels showing Files; duplicate Files panels fetch
  once.
- **`E`:** Main opens the active card's text (Context vs Reply differ), Files opens the
  real path, Tools opens LLM Calls text. Monkeypatch `subprocess.run` / the editor.
- **`l`/`h`/`L`/`H`:** with Tools focused they change the focused view's detail level,
  including `h` and `L`. With Tools only in the unfocused panel they fall through to
  normal Agents behavior. The footer chip follows focus.
- **Search:**
  - `/` in the focused panel shows that panel's overlay (not the other panel's).
  - The Main corpus includes Reply text while Context is the active card.
  - Files searches the active diff; Tools searches LLM Calls text.
  - `q` restores the native scroll position.
  - A structural key (Ctrl+N) exits search and still cycles the deck.
  - An identity change exits.
- **Folds:** `za` in a Main view toggles the section at its viewport top. It targets the
  other panel's Main view when the focused panel is Files. With no Main shown it is a
  no-op plus a notice.
- **Hints:** `v` with Files focused in SINGLE switches the panel to Main and paints hint
  markers.
- **`.`/`d`:** a pinned Main view stays pinned after each toggle.
- **Flag-off regression:** run the existing suites unchanged:
  - `tests/ace/tui/test_agent_metadata_search.py`
  - `tests/ace/tui/test_agents_zoom_panel_search.py`
  - `tests/ace/tui/test_agent_fold_transitions_llm_calls.py`
  - the clipboard / auto-refresh / footer tests
  - `tests/ace/tui/widgets/decks/`

## Verification

1. Read the `lint_and_test.md` and `tui.md` → `tui_perf.md` / `tui_screenshot.md` memory
   notes (via `sase memory read`) before finishing.
2. Run `just fix`, then run `sase tool run check` (use `/sase_monitor` if it may outrun
   the turn). Do not run `just check-full`. The `symvision` gate's known pre-existing
   private-import failures on clean HEAD are not this phase's; confirm by diffing
   against a clean tree. Any new public symbol must have a non-test consumer.
3. **Goldens.** Rendered flag-off output changes only in the slow-tools overflow line.
   Run a targeted `just fix-tui-screenshots -- <selectors>` for any golden showing that
   line, plus `tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`. Inspect
   every changed PNG. Add one flag-on golden of the per-panel search overlay in a
   LEFT_RIGHT split (focused right panel, committed search with a highlighted match).
4. **Live screenshot.** Take a `sase screenshot` PNG with
   `SASE_FEATURE_FLAGS='{"agent_decks": true}'` of a split with search active, and
   inspect it for overlay placement, the command line and focus styling.
5. **Epic symbols.** Run `sase bead epic-symbols sase-17d.6` and resolve or re-key any
   leftovers.
6. **Close the bead.** Close `sase-17d.6` only, with a note listing the final consumer
   inventory table and what was verified. Record out-of-scope discoveries as
   `PROPOSED FOLLOW-UP:` bead notes. Do not close the epic.
