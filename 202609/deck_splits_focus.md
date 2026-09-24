---
tier: tale
title: Deck split layouts, focus and split ratio
goal:
  With agent_decks on, \ and | toggle top-bottom and left-right deck splits through a
  pure layout state machine, Ctrl+F moves logical focus, { and } resize the focused
  panel, duplicate decks fan out to both panels without double fetches or remounts, and
  the lost deck-navigation-keys phase work is re-landed with it.
size: medium
proposed_by: bbugyi200.athena.sase-17d.5
bead: sase-17d.5
create_time: 2026-09-24 07:36:05
status: wip
---

- **PARENT:**
  [202609/agents_tab_decks_and_cards.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_tab_decks_and_cards.md)
- **BEAD:**
  [sase-17d.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17d/sase-17d.5.md)

# Plan: Deck split layouts, focus and split ratio (bead sase-17d.5, phase `deck-splits-focus`)

Epic plan: `plan:202609/agents_tab_decks_and_cards.md`. Read §3.3 (layouts and
transitions), §3.4 (keymap), §3.8 (visual language), §4.4 (composition), §4.6 (pure
state model), §4.8 (performance) and §9 (this phase). **The epic plan wins wherever this
plan is silent.** All paths are relative to `src/sase/ace/tui/` unless they start with
`src/`, `tests/` or `docs/`.

## 0. Blocker found while planning: phase 4 (`deck-navigation-keys`) never landed

Bead `sase-17d.4` is CLOSED with a "done" note, but none of its code is on `master`
(`grep -rn next_deck_card src/` is empty). Its agent's turn ended with
`Finalizer declaration recovery failed`, so the host never committed the work. The
tracked part of its diff survived in a temp cache and is now saved at:

`~/.sase/rescue/202609/sase-17d.4-deck-navigation-keys.diff` (sha256
`7572cef9782f721f22d2b4b8b7a10df0636a8aef1e554b9440b1ebb60dcfd65d`)

This phase depends on it (the focused-panel scroll retargeting, the four deck actions
and their registration). So step 1 re-lands it:

1. `git apply --exclude='src/sase/ace/tui/widgets/file_panel/_content.py' <that diff>`.
   It applies cleanly to current master. The excluded hunk was a mypy drive-by that
   master already fixed differently; skip it.
2. What the diff contains: `next_deck_card`/`prev_deck_card` (Ctrl+J/K) and
   `next_deck`/`prev_deck` (Ctrl+N/P) with `AppKeymaps` fields, `default_config.yml`
   defaults, `_BINDING_META` rows, fallback `bindings.py` rows, `check_app_action` gates
   (deck actions need Agents + decks on; the legacy section-stop and file-cycle actions
   are unavailable on Agents while decks are on), palette metadata and availability,
   help rows, `_CONTEXTUAL_APP_DUPLICATES` pairs (tab-disjoint vs. Artifacts paging,
   flag-disjoint vs. the legacy Agents actions), pure `cycle_deck_id` and
   `cycle_card_id` in `widgets/decks/model.py`, `MainDeckView.show_card`,
   `DeckPanel.cycle_card` plus a live-keymap empty-state hint, and
   `AgentDetail.cycle_focused_deck_card` / `cycle_focused_deck`. Ctrl+D/U, g/G and the
   bottom pin target the focused panel through `effective_detail_scroll_id` and the new
   `_release_focused_deck_bottom_pin` / `_pin_focused_deck_to_bottom` helpers in
   `actions/navigation/_basic.py`. Its tests are edits to existing files
   (`tests/ace/tui/widgets/decks/test_deck_model.py`, `test_deck_panels.py`,
   `tests/test_keymaps_app_bindings.py`).
3. What was lost (untracked files): phase 4's pilot key tests and its two flag-on PNG
   goldens. It also never added the `^J/^K` footer entry. §6 and §7 below restore these
   alongside this phase's own.
4. One recovered behavior changes in this phase: the diff made Agents `Ctrl+F`/`Ctrl+B`
   (`scroll_prompt_down/up`) half-page the focused deck panel via
   `_scroll_focused_deck_panel`. Per epic §3.4 and §9, with decks on Ctrl+F becomes
   `toggle_deck_focus` and Ctrl+B does nothing. Delete `_scroll_focused_deck_panel` and
   its two call sites, and gate `scroll_prompt_down/up` off on Agents while decks are on
   (§4).

Record this recovery in the bead's closing note, so the epic's land agent knows that
phase 4's work arrived in this phase's commit.

## 1. Pure layout state machine (`widgets/decks/layout.py`, new)

Keep `model.py` small. Put the layout types and transitions in a new module, and export
nothing through `widgets/__init__` unless needed.

- `DeckLayout(StrEnum)`: `SINGLE`, `TOP_BOTTOM`, `LEFT_RIGHT`. Never use the words
  horizontal or vertical in identifiers.
- Extend `DeckAreaState` (in `model.py`) with `layout: DeckLayout = SINGLE` and
  `ratio: int = 50` (the first panel's share). `panels` has length 1 in SINGLE and 2 in
  a split. Update `with_panel_deck` / `with_preferred_card` to preserve the new fields
  (use `dataclasses.replace`). `DeckArea.__init__` currently seeds two panel states:
  change it to the one-panel SINGLE default and drop the `_ = SINGLE` hack.
- `RATIO_STEPS = (30, 50, 70)`.
- Pure functions, each with a docstring:
  - `choose_new_panel(current_deck, current_active_card, shown, has_content, card_ids) -> DeckPanelState`.
    Walk `DECK_CYCLE` forward from `current_deck`. Pick the first deck that is not in
    `shown` and whose `has_content[deck]` is not `False`. Unknown (`None`) counts as
    content, because Files/Tools probes are unknown until a panel has shown them and
    skipping them would almost always give Context | Reply. If none qualifies, duplicate
    `current_deck` with
    `preferred_card = cycle_card_id(card_ids, current_active_card, 1)` (card ids are
    only meaningful for Main; pass `()` otherwise).
  - `toggle_split(state, target, new_panel) -> DeckAreaState`, where `target` is
    `TOP_BOTTOM` for `\` or `LEFT_RIGHT` for `|`:
    - From SINGLE: open `new_panel` as panel 1, set `layout = target`, `focused = 1` and
      `ratio = 50`.
    - Same layout as `target`: unsplit. Keep only `panels[0]`, `focused = 0`,
      `layout = SINGLE`, `ratio = 50`.
    - The other split layout: rotate. Change only `layout`; decks, preferred cards,
      focus and ratio are kept.
    - `new_panel` is only consulted in the SINGLE case. Callers compute it with
      `choose_new_panel` beforehand.
  - `toggle_focus(state)`: in a split, `focused = 1 - focused`. In SINGLE, return the
    state unchanged.
  - `step_ratio(state, grow: bool)`: split only. Grow means the focused panel gets
    bigger. If panel 0 is focused, grow moves up through `RATIO_STEPS`; if panel 1 is
    focused, grow moves down. Clamp at the ends; no wrap.
- Unit tests in `tests/ace/tui/widgets/decks/test_deck_layout.py`, one per transition
  row in epic §3.3 plus:
  - the new-panel deck rule, including the no-files/no-tools agent giving Context |
    Reply
  - unknown availability counts as content
  - duplicate Main opening on the next card, with wrap
  - both focus directions for the ratio clamp
  - `toggle_focus` in SINGLE is a no-op
  - unsplit keeps panel 0 even when panel 1 is focused (involution: `|` then `|` returns
    the original state)
  - rotate keeps everything but the layout

## 2. Widgets: `DeckArea`, `DeckPanel`, CSS

- **`DeckArea.apply_state(new_state)`** is the single choke point. It stores the state
  and syncs classes only, with no mount or remove:
  - `-single` / `-top-bottom` / `-left-right` and `-ratio-30|50|70` on the area
  - `hidden` on panel 1 in SINGLE
  - `set_focused(bool)` on each panel. In SINGLE the only panel is always focused.
  - `visible_panels()` returns `(panel 0,)` in SINGLE and both panels in a split, so the
    existing `panels_showing`, `_on_main_document` and `_deck_refresh_views` loops fan
    the Main document and Files/Tools refreshes out to both panels without further
    change.
  - DOM order is always panel 0 then panel 1, so no `move_child` is needed.
- **`DeckPanel.set_focused(focused)`** toggles `-focused` / `-unfocused` and calls
  `refresh_chrome()`. Remove the unconditional `add_class("-focused")` from `set_deck`.
  `refresh_chrome` passes `focused=self._focused` to `deck_title`, which already
  supports muted unfocused titles.
- **Click to focus.** `DeckPanel.on_click` posts a small
  `DeckPanelFocusRequested(index)` message and does not stop the click. `DeckArea`
  handles it by applying `toggle_focus` only when `index` differs from the focused
  panel, and then asks the app to refresh the Agents footer
  (`_refresh_agent_footer_bindings_only`). Clicking must not take Textual widget focus
  away from the node list; j/k keep driving the selection.
- **CSS** (`styles.tcss`, next to the existing `#agent-deck-area` rules):
  - `#agent-deck-area.-left-right { layout: horizontal; }`
  - TOP_BOTTOM ratio rules:
    `#agent-deck-area.-top-bottom.-ratio-30 #agent-deck-panel-0 { height: 3fr; }`, with
    panel 1 at `7fr`, and likewise for 50/50 and 70/30.
  - LEFT_RIGHT: the same three ratios with `width`, and `height: 1fr` on both panels.
  - Unfocused panels:
    `.deck-panel.-deck-main.-unfocused { border: solid $secondary 35%; }`, and the same
    for Files (`green`) and Tools (`#87D7FF`). Geometry must be identical in both focus
    states: same border type, no padding change.
  - Avoid `:not()` selectors (the deck-panel-core phase dropped them as unreliable).
- **Image cards on resize.** File-panel images are sized once from the viewport
  (`image_preview_size_for_viewport` in `file_panel/_display.py`). Add a small public
  `AgentFilePanel.rerender_for_viewport()` that re-displays the current page only when
  it is an image, through the existing static-image path. `DeckPanel.on_resize` calls it
  after refresh when its Files deck is shown. Text and diff cards re-wrap by themselves.

## 3. AgentDetail layout API (`widgets/_agent_detail_deck_layout.py`, new mixin)

`_agent_detail_decks.py` is about 400 lines after step 0. Put the layout API in a new
mixin module and add the mixin to `AgentDetail`'s bases.

- `toggle_deck_split(target: DeckLayout) -> None`:
  - From SINGLE, gather the inputs for `choose_new_panel`: panel 0's deck and active
    card (Main: `main_view.active_card_id`; the Main card ids come from
    `_main_deck_document`), the decks shown, and `has_content` from panel 0's current
    availability map (`panel._availability`, already filled with no-I/O probes by
    `_deck_refresh_availability`).
  - Apply `toggle_split` through `deck_area.apply_state`.
  - When a panel was opened, call the existing `show_deck(1, deck)` so the new panel
    loads for the current subject at once, from caches first. For a duplicate Main,
    `show_deck` already honors the preferred card. For a duplicate Files deck, step the
    new panel's file view once with `next_file()` when its file list is already
    populated, so it opens on the card after panel 0's. A duplicate Tools deck has one
    card.
  - Then `_deck_refresh_availability()`.
  - Unsplit and rotate touch no view content: scroll positions and active cards survive
    because nothing is rebuilt.
- `toggle_deck_focus()`, `step_deck_ratio(grow: bool)` and `deck_layout` (a read-only
  property) are thin wrappers over the pure functions plus `apply_state`.
- **No double fetches for duplicate decks.** `_inflight_diff_tasks` in
  `file_panel/_fetch.py` is per-instance, so two Files views of the same agent would
  each start a diff worker. Make `_deck_refresh_views` / `show_deck` load a deck's data
  once per refresh:
  - The first visible panel showing Files or Tools fetches normally.
  - A second panel showing the same deck is fed from the shared caches (`file_cache`,
    `peek_tool_calls_cache_entry`) and re-fed when the owner's view reports completion.
    The owner's `FileVisibilityChanged` / `FileListChanged` /
    `LLMCallsVisibilityChanged` already reach its `DeckPanel`; have `DeckPanel` notify
    `AgentDetail`, which re-loads the sibling duplicate from cache.
  - Keep this in the deck mixin; do not change legacy flag-off paths.
  - A test must prove exactly one diff worker and one tool-call fetch per refresh with a
    duplicate deck in both panels, and that both panels end up showing the content.
  - If investigation shows the Tools path is already deduped by the mtime-keyed cache
    throttle, keep only the test for Tools and say so in the bead note.

## 4. Actions and full keymap registration

Add a new app mixin module `actions/agents/_deck_layout_actions.py`
(`AgentDeckLayoutActionsMixin`), next to where `AgentPanelDetailMixin` is mixed into the
app. `_panel_detail.py` is already near 500 lines. Actions:

| Action id                 | Default key           | Behavior                        |
| ------------------------- | --------------------- | ------------------------------- |
| `toggle_deck_split_below` | `backslash`           | `toggle_deck_split(TOP_BOTTOM)` |
| `toggle_deck_split_right` | `vertical_line`       | `toggle_deck_split(LEFT_RIGHT)` |
| `toggle_deck_focus`       | `ctrl+f`              | `toggle_deck_focus()`           |
| `grow_deck_panel`         | `right_curly_bracket` | `step_deck_ratio(grow=True)`    |
| `shrink_deck_panel`       | `left_curly_bracket`  | `step_deck_ratio(grow=False)`   |

Each action no-ops unless the Agents tab is current and decks are active. After each
state change, call `_refresh_agent_footer_bindings_only()`.

Registration, following exactly what step 0's diff did for the Ctrl+J/K/N/P actions:

- `AppKeymaps` fields in `keymaps/app_keymaps.py`.
- `src/sase/default_config.yml` defaults under `ace.keymaps.app`, next to the deck keys
  from step 0 (the core gotcha).
- `_BINDING_META` rows in `keymaps/metadata.py`.
- Fallback rows in `bindings.py`, placed after the existing owners of Ctrl+F and
  `{`/`}`. Textual runs the first same-key binding whose `check_action` passes, so
  gating decides the owner.
- **`check_app_action` gates** (`_app_action_availability.py`), in a
  `_DECK_LAYOUT_ACTIONS` set that respects `_prompt_input_owns_keys`:
  - the two split actions need Agents and decks active
  - `toggle_deck_focus` and the grow/shrink actions also need a split layout
  - `scroll_prompt_down` / `scroll_prompt_up` return False on Agents while decks are
    active. This lets Ctrl+F fall through to `toggle_deck_focus` and makes Agents Ctrl+B
    a no-op. Services and Artifacts are unchanged.
  - Flag off, Agents Ctrl+F/B keep their half-page scroll: nothing changes, because the
    new actions are unavailable.
- **Palette**:
  - rows in `commands/_app_metadata_nav.py`: `AGENTS_ONLY`, labels such as "Split deck
    panels top/bottom", "Split deck panels left/right", "Focus other deck panel", "Grow
    focused deck panel", "Shrink focused deck panel"
  - availability in `commands/_availability_agents.py`: decks active, and split-only for
    focus/grow/shrink
  - add `agent_deck_split: bool = False` to `CommandContext` (`commands/types.py`), set
    in `commands/context.py` next to `agent_decks_active`
  - hide `app.scroll_prompt_down/up` on Agents while decks are active
- **Help rows** in `modals/help_modal/agents_bindings.py`, beside step 0's deck rows:
  - `\ / |` "Split deck panels (decks)"
  - `^F` "Focus other deck panel (decks)"
  - `{ / }` "Shrink/grow deck panel (decks)"

  Keep the 57-char boxes and descriptions ≤ 32 characters. Build key text from the live
  keymap with `d(...)`.

- **`_CONTEXTUAL_APP_DUPLICATES`** in `keymaps/registry.py`:
  - flag-disjoint: `{toggle_deck_focus, scroll_prompt_down}`
  - tab-disjoint: `{grow_deck_panel, cycle_artifacts_split}` and
    `{shrink_deck_panel, cycle_artifacts_split_reverse}`
  - Add a pair for any other collision the registry validation reports; check each is
    genuinely disjoint first.
- **Footer** (`widgets/_keybinding_bindings_agents.py`): add kwargs `deck_split: bool`
  and `deck_card_count: int`, threaded from the three `_compute_agent_bindings` callers
  (`widgets/_keybinding_modes.py`, `actions/agents/_display_detail_footer.py`,
  `actions/agents/_panel_detail.py`):
  - `^F` "other panel" and `{/}` "resize" appear only while split. Key text uses
    `self._kd(...)`, with `{/}` joined like the other pairs.
  - `^J/^K` "cards" appears only while the focused deck has more than one card. This
    restores phase 4's lost footer item.
  - Both are appended only when decks are active.

## 5. Behavior details to honor

- Transitions follow epic §3.3 exactly. Unsplit always closes panel 1, even when it is
  focused. The new panel takes focus. Every new split starts at 50/50. Rotate keeps
  decks, cards, focus and ratio.
- Ctrl+N/P on panel 1 cycle panel 1's deck. Duplicate decks are allowed. Each panel
  keeps its own active card and scroll, because each panel owns its own views.
- Ctrl+D/U, g/G and the bottom pin act on the focused panel. They already route through
  `effective_detail_scroll_id` and step 0's helpers; verify this with panel 1 focused.
- The shared chrome (`AgentHeaderPanel`, `AgentJumpPanel`) spans both panels unchanged.
- A split layout persists across j/k selection changes. Panel 1's views refresh through
  the fan-out, and an empty deck shows its empty-state card without reflow.
- **Perf** (epic §4.8): layout and ratio changes only swap CSS classes. The j/k
  immediate paint must not gain work proportional to panel count. The Main document
  fan-out reuses one built `MainDeckDocument`.

## 6. Tests

- **Pure:** `test_deck_layout.py` (§1).
- **Pilot, flag on:** a new `tests/ace/tui/widgets/decks/test_deck_split_pilot.py`. Use
  the existing `_DetailApp` / `make_artifact_agent` helpers from `test_deck_panels.py`,
  or move them to a shared `_deck_helpers.py` if reuse needs that.
  - `\` and `|` from SINGLE open panel 1 with focus, CSS classes and a visible panel 1.
  - A second press of the same key unsplits and panel 0 keeps its card and scroll
    offset.
  - The other key rotates without remounting: assert the panel and view widget
    identities are unchanged.
  - Ctrl+F flips focus and the `-focused`/`-unfocused` classes; it is a no-op in SINGLE.
  - `}`/`{` step the ratio classes from each focused side and clamp.
  - Clicking panel 0 focuses it.
  - Context | Reply for an agent with no files or tools.
  - The Main document reaches both panels on update.
  - Ctrl+N/P act on the focused panel.
  - Ctrl+D/U scroll the focused panel's scroll.
  - The duplicate-deck single-fetch test (§3).
  - An image card re-renders after a ratio change.
  - Ctrl+B does nothing on Agents.
- **Flag off:** Agents Ctrl+F/B still half-page scroll `#agent-prompt-scroll`. `\`, `|`,
  `{`, `}` do nothing on Agents. Artifacts `{`/`}` still cycle the split. Services and
  Artifacts Ctrl+F are unchanged.
- **Restore phase 4's lost pilot key tests:** Ctrl+J/K/N/P through real key presses in
  both flag states, including the Artifacts load-more/unload and Services chop-run
  regressions (`tests/ace/tui/test_axe_chop_run_nav.py`, `test_artifacts_limit_keys.py`
  must still pass).
- **Palette and availability:** extend the `agents_available` test from step 0 with the
  new ids, split and unsplit contexts, and the `scroll_prompt_*` hiding.
- **Enforcement suites must pass:**
  - `tests/test_command_catalog.py`
  - `tests/test_command_catalog_guards.py`
  - the `tests/test_keymaps_defaults*.py` files
  - `tests/test_keymaps_app_bindings.py`: extend the same-key ordering assertions for
    ctrl+f, `{`, `}`
  - `tests/test_keymaps_validation.py`
  - the help-modal box-width tests
  - the footer tests for `_compute_agent_bindings`

## 7. PNG goldens and live screenshots

- Add `tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`. Follow
  `test_ace_png_snapshots_agents_panel_layout.py`, enable the flag with
  `override_flags(agent_decks=True)` around app startup, and use deterministic fixture
  agents. Goldens:
  - SINGLE Main on Context
  - SINGLE Main on Reply
  - SINGLE empty state (these three restore phase 4's lost goldens)
  - TOP_BOTTOM, focus on the bottom panel
  - LEFT_RIGHT at ratio 70 with focus on the left, showing the focus styling contrast
  - Context | Reply for an agent without files or tools
- Generate them with a targeted
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`,
  through `/sase_monitor` if it may be long. Inspect every created PNG and the report
  before finalizing.
- Capture live `sase screenshot` PNGs with `SASE_FEATURE_FLAGS='{"agent_decks": true}'`
  (read the `tui_screenshot.md` memory first). Inspect them for layout, borders, focus
  dimming, titles and subtitles in both splits. Also check that flag-off Agents are
  unchanged.

## 8. Verification and closing

- Read `tui_perf.md` before touching the paint paths. Run
  `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` with `SASE_TUI_PERF=1` in SINGLE and
  LEFT_RIGHT. j/k p95 should stay under 16 ms; report numbers honestly, since the host
  is shared and noisy.
- `just fix`, then `sase tool run check` (through `/sase_monitor` if needed). Known:
  symvision failed on a clean HEAD with pre-existing unrelated private-import errors
  during phases 3–4. Re-check it; if it still fails identically on a pristine tree,
  report it as pre-existing rather than fixing unrelated files. Do not run
  `just check-full`.
- `sase bead epic-symbols sase-17d.5` must report no entries. It reports none today; do
  not add `--epic-symbol` lines unless a symbol genuinely lands before its consumer.
  Then:
  - `sase bead close sase-17d.5 --note "<verified items + the phase-4 recovery>"`
  - Record out-of-scope findings as `PROPOSED FOLLOW-UP:` notes on `sase-17d.5`. One
    known finding: phase 4's host finalizer failure closed a bead whose work never
    committed.
  - Never close the epic or other beads.
