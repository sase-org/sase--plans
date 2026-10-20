---
tier: tale
title: Keep deck-panel focus when the split keys open a new panel
goal:
  On the Agents tab, pressing `\` or `|` never moves deck focus into a newly created
  deck panel; focus stays on the panel the key was pressed from.
size: medium
proposed_by: bbugyi200.apollo.99
proposed_with: claude/opus
create_time: 2026-10-20 07:48:42
status: wip
---

# Plan: Keep Deck-Panel Focus When `\` / `|` Open A New Panel

## Goal

On the Agents tab, `\` (`toggle_deck_split_below`) and `|` (`toggle_deck_split_right`)
must leave logical deck focus on the panel that was focused when the key was pressed.
Today, any press that creates a deck panel moves focus into the new panel. That covers
both single → two panels and two → three panels (the nested T layout).

## Root Cause

- `AgentDetailDeckLayoutMixin.toggle_deck_split` in
  `src/sase/ace/tui/widgets/_agent_detail_deck_layout.py` has two panel-creating paths,
  and both use the "focus the new panel" default:
  - Single → split: `self._open_deck_split(target)`, whose `focus_new` defaults to
    `True`.
  - Two → three (nest): `toggle_split(state, target, new_panel)`, whose `focus_new` also
    defaults to `True`.
- `toggle_split` in `src/sase/ace/tui/widgets/decks/layout.py` can already keep focus.
  Both creating branches run `focus_pane(pressed, state.grid.focused)` when
  `focus_new=False`. The pure `pane_grid.press_split`
  (`src/sase/ace/tui/util/pane_grid.py`) always focuses the new pane, by design, because
  the pager shares it. The deck layer overrides that.
- The only caller that passes `focus_new=False` today is the picker's capital-letter
  path, `show_deck_in_other_panel` → `_open_deck_split(..., focus_new=False)`.

## Behavior After The Change

- **Single → two panels.** The new panel opens with the same deck choice as today
  (`choose_new_panel`). Focus stays on the original panel. The MRU order (`grid.recent`)
  becomes `(original, new)`, so the picker's capital letters and the `Ctrl+X` fallback
  target the new panel.
- **Two → three panels (nest).** The divider is still drawn through the focused panel.
  Focus stays on that panel, and the MRU order becomes `(focused, new, other)`.
- **Unchanged paths.** These already keep focus: same-key unsplit, three-pane
  erase/turn, the zoomed-state restore, refused three-pane keys (toast), and the
  picker's capital letters.
- **Consequences of the unchanged split-key rule** (intended; document them):
  - `\` then `|` now splits the _original_ panel. Press `Ctrl+F` between the two keys to
    split the new panel instead.
  - `\` `\` (or `|` `|`) is now a true round trip: the original panel survives, not the
    new one.
- **Out of scope:** the pager's own split keys (`src/sase/pager/`) are a separate
  feature that clones the focused pane. Leave the pager and `pane_grid.press_split`
  unchanged.

## Implementation

1. **`src/sase/ace/tui/widgets/decks/layout.py` › `toggle_split`.**
   - Remove the `focus_new` keyword. Both panel-creating branches (single open and two →
     three nest) always apply `focus_pane(pressed, state.grid.focused)`.
   - Rewrite the docstring to say that focus stays on the focused pane and the new pane
     becomes the MRU-next pane. Keep the rest of the docstring.
2. **`src/sase/ace/tui/widgets/_agent_detail_deck_layout.py`.**
   - `_open_deck_split`: remove the `focus_new` parameter. Pass nothing extra to
     `toggle_split`. Compute the new pane as `new_id = opened[0]`, which also feeds the
     duplicate-Files `next_file()` logic. Update the docstring: the new panel never
     takes focus.
   - `show_deck_in_other_panel`: drop the `focus_new=False` argument. Its docstring
     already says focus stays on the source panel.
   - `toggle_deck_split`: the nest path needs no logic change because it inherits the
     new `toggle_split`. Update its one-line docstring to state that focus stays on the
     focused panel.
3. **No changes** to `pane_grid.py`, the pager, keymaps, `default_config.yml`, the
   help-modal text, or the command-palette metadata. The help row ("…draw one through
   the focused pane…") is still accurate. Leaving it alone keeps the help-modal goldens
   unchanged.
4. **Docs (`docs/reference/tui/agent-statuses.md`, Reference tier; behavior of an
   existing feature):**
   - In the deck-picker paragraph (~line 263), change "Focus always stays on the panel
     the picker was opened from, unlike `\`, which moves focus into the new panel;
     `Ctrl+F` moves it if you want." to say that focus stays put here just as it does
     for `\` and `|`, and that `Ctrl+F` moves it.
   - In the split-key paragraph (~line 442), replace "the new panel takes focus and
     shows the next deck…" with: focus stays on the panel you split from, and the new
     panel shows the next deck with content, or a duplicate deck. Add one sentence after
     the split-key rule: because focus stays put, `\` then `|` splits the original
     panel; press `Ctrl+F` in between to split the new one.
   - Do not edit `CHANGELOG.md` (release-please generates it) or the glossary
     (`sase/memory/glossary/deck-panel.md` makes no focus-on-new claim).

## Tests

### Pure Layout Tests

- `tests/ace/tui/widgets/decks/test_deck_layout.py`:
  - Update the single-open tests (`test_single_backslash_opens_top_bottom` and
    `test_single_pipe_opens_left_right`) to assert `focused == 0` and
    `grid.recent == (0, 1)`.
  - Update the nest test `test_other_key_nests_third_panel_keeping_outer_shape` to
    assert that focus stays on the previously focused pane and that the new pane is
    second in `recent`.
  - Replace `test_toggle_split_focus_new_false_keeps_focus_on_first_panel` and
    `test_toggle_split_focus_new_ignored_by_unsplit_and_nest` with keep-focus tests.
    Cover the single open on both axes and the nest from both focus positions (focused
    pane at reading index 0 and at index 1).
  - Strengthen `test_unsplit_keeps_focused_panel_involution`. Pressing the same key
    twice from single must now return exactly the original single state (pane `0` with
    its original panel state).
- `tests/ace/tui/widgets/decks/test_deck_collapse_zoom.py` › `_split_state`: keep its
  "focus right" premise by applying `toggle_focus` after `toggle_split`, or by building
  the grid directly.

### Pilot / Widget Tests

- `tests/ace/tui/widgets/decks/test_deck_split_pilot.py`:
  - `test_backslash_opens_top_bottom_with_focus`: rename it (for example
    `…_keeps_focus`). Assert `focused == 0`, `panel(0)` has `-focused`, and `panel(1)`
    has `-unfocused`. Drop the vacuous `or True`.
  - `test_pipe_opens_left_right_and_unsplit_keeps_focused`: assert focus 0 after the
    split, then `toggle_deck_focus()` to panel 1 so the test still proves that unsplit
    keeps the focused panel rather than panel 0.
  - `test_unsplit_keeps_first_when_first_focused`: drop the now-redundant focus flip.
  - Fix the premises of `test_ratio_steps_and_clamp`,
    `test_ctrl_n_p_act_on_focused_panel` and `test_ctrl_f_flips_focus_and_single_noop`.
  - Add a real-key regression test on `_DeckKeyApp`. Press `backslash`: focus stays 0
    and `area.focused_panel()` is the same widget as before. Press `vertical_line`:
    three panes, and focus is still on that widget. Repeat starting from
    `vertical_line`.
- `tests/ace/tui/widgets/decks/test_deck_other_panel.py`:
  - The `_open_split` helper documents "focus lands on 1". Add
    `detail.toggle_deck_focus()` after the split to keep that premise.
  - Invert and rename `test_toggle_deck_split_still_focuses_the_new_panel` to
    `test_toggle_deck_split_keeps_focus_on_source_panel`. It should assert focus 0 and
    the matching `-focused` / `-unfocused` classes.
- Run and fix every other module that splits and then relies on the new panel being
  focused: `tests/ace/tui/widgets/decks/` (`test_deck_split_keys.py`,
  `test_deck_action_targets.py`, `test_deck_collapse_zoom.py`, `test_deck_panels.py`,
  `test_deck_search_overlay.py`, `test_deck_view_main_pilot.py`,
  `test_deck_block_rail.py`), `tests/ace/tui/test_agents_deck_picker.py`,
  `tests/ace/tui/test_agents_node_rail_wiring.py`,
  `tests/ace/tui/test_agent_archive_remaining_proof.py`,
  `tests/ace/tui/widgets/test_agent_header_panel_basic.py`,
  `tests/ace/tui/widgets/test_agent_header_panel_scroll.py`,
  `tests/ace/tui/widgets/test_agent_jump_panel_expansion.py`,
  `tests/ace/tui/widgets/test_agent_jump_panel_visibility.py`,
  `tests/ace/tui/actions/test_view_files_pager_split_keys.py`. Where a test only used
  focus-on-new as _setup_, add one `toggle_deck_focus()` / `ctrl+f` immediately after
  the split. Do not weaken its assertions. Benchmarks (`tests/ace/tui/bench_tui_jk*.py`)
  need no change.

### PNG Visual Snapshots: Goldens Must Stay Byte-Identical

Equivalence rule: one forward `ctrl+f` right after a panel-creating split key rebuilds
exactly the old grid, with the same panes, focus, and MRU order. In both creating paths
the new pane sits immediately after the focused pane in reading order:

- Single open: `(focused, new)`.
- Nest from index 0: `(focused, new, unfocused)`.
- Nest from index 1: `(unfocused, focused, new)`.

`_refocus` then produces `recent == (new, focused, …)`, which equals the old
`press_split` result. The inverse also holds: a `ctrl+f` / `ctrl+b` that used to follow
a two-pane split to get back to the original panel is now simply dropped.

Apply the rule to every visual test that presses `backslash` / `vertical_line` or calls
`action_toggle_deck_split_*`, keeping each test's named intent:

- `tests/ace/tui/visual/test_ace_png_snapshots_agents_decks.py`:
  - Add `ctrl+f` after the split in `top_bottom_focus_bottom`, the
    `live_reply_partial_and_growth` split step, `context_reply_no_files`, and
    `collapsed_split`.
  - Drop the post-split `ctrl+f` in `left_right_ratio_70_focus_left`, `zoomed`, and
    `left_right_search_committed`.
  - `_goto_three_pane_main_top`: add `ctrl+f` after each key.
  - `three_pane_main_bottom` and `three_pane_main_right`: drop the `ctrl+b` and add
    `ctrl+f` after the second key.
  - `three_pane_main_left`: add `ctrl+f` after each key.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_picker.py`: add `ctrl+f` in
  `split_bottom` and `left_right` (their titles say the new panel is focused).
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_views_main.py`: drop the
  "move focus back left" `ctrl+f` and its comment in `split_narrow` and
  `split_spread_paged`. Keep the focus-0 `wait_for_state`.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_blocks.py`: drop the
  return-left `ctrl+f` in `split_rails` and `micro`.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_final_session.py`: add `ctrl+f`
  after the split in `reply_split` and `narrow_tiers`. In `narrow_tiers`, the `P` press
  must still land on the FINAL panel.
- `tests/ace/tui/visual/test_ace_png_snapshots_tool_runs_session.py`: after
  `action_toggle_deck_split_right()`, call `action_toggle_deck_focus()` before the
  `tool_runs_split` capture.

The expected outcome is **zero golden changes**. Verify with a targeted check-only run
over the six modules above (`just fix-tui-screenshots --check -- <module paths>`,
through `/sase_monitor` if it is long). If a golden still differs, find out why before
accepting anything. Only accept a difference that is a genuine, explained consequence of
the new behavior. Accept it through the update form
(`just fix-tui-screenshots -- <selectors>`) and inspect its report, as the TUI
screenshot memory requires.

## Verification

1. Run the deck and Agents-tab pytest modules listed above directly, until they are
   green.
2. Run the targeted visual `--check` described above. It must report no drift.
3. Run `just fix` (formatting), then `sase tool run check`. Do not run `just check-full`
   unless explicitly told to.
4. Optional live sanity check: run `sase screenshot` on the Agents tab with presses `\`
   and then `|`. The originally focused panel must keep the focused chrome.
