---
tier: tale
title: Deck picker capital letters show a deck in the other panel
goal:
  In the Agents deck picker, M/F/T show that deck in the non-focused deck panel, opening
  a new top-bottom panel when the deck area is single and keeping an existing left-right
  or top-bottom layout as is, with focus staying on the panel the picker was opened
  from.
size: medium
proposed_by: bbugyi200.apollo.1p.f0
create_time: 2026-09-25 11:47:36
status: wip
---

# Plan: Capital letters in the deck picker show a deck in the other panel

## Why

The Agents deck picker (`p`, then `m` / `f` / `t`) changes the deck in the **focused**
panel. The next most common wish is "show me X beside what I'm reading". Today that
takes `\` (which picks the new panel's deck by itself), then `Ctrl+N` / `Ctrl+P`
cycling, or `Ctrl+F` plus the picker. Capital letters make it two keys: `p` then `F`
shows Files in the other panel, and opens one below if there isn't one yet.

## Design

### The rule

Inside the picker, a lowercase deck letter still means "this panel". The matching
**capital letter** (`M` / `F` / `T`) means "the other panel":

| Deck area when `p` is pressed | Capital letter does                                                                                                          |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Single panel                  | Opens a new **top-bottom** split. The new bottom panel shows the picked deck.                                                |
| Top-bottom split              | Shows the deck in the other panel (top ↔ bottom). The layout is unchanged.                                                  |
| Left-right split              | Shows the deck in the other panel (left ↔ right). The layout stays left-right; it is never rotated to top-bottom.           |
| Zoomed (from single)          | Ends the zoom the way `Z` does, then opens a new bottom panel (as in the single case).                                       |
| Zoomed (from a split)         | Ends the zoom the way `Z` does, bringing the split back in its original orientation, then shows the deck in the other panel. |

"Horizontal panel" here means the top-bottom split that `\` opens, the same as vim's
`:split`. A left-right split is the "vertical" layout. That layout is only ever reused,
never converted.

### Focus stays put

Focus stays on the panel the picker was opened from, in every case. This applies even
when a new split is created, which is where it differs from `\`: `\` moves focus into
the new panel. Reasons:

- Uppercase consistently means "the companion panel". `p F`, then `p T`, keeps changing
  the **same** other panel. If focus moved, the second pick would land on the panel the
  user was just reading and replace it.
- The user's reading position, card, scroll, and the focus-dependent keys (search,
  cards, `E`) don't jump.
- `Ctrl+F` is one key away if the user does want to move into the new panel.

### No-ops and duplicates

- A capital letter for the deck the other panel already shows closes the picker and
  changes nothing: no toast, and no deck-state save. The row's `○ in <pos> panel` badge
  already tells the user it's there. The exception is the zoomed case. There, the zoom
  still ends and the target panel is re-shown, so any content that went stale while it
  was hidden gets refreshed.
- A capital letter for the focused panel's own deck is allowed: the other panel shows a
  duplicate, as split panels already can. When this opens a new split, it reuses the
  existing duplicate niceties from `\`:
  - a duplicate **Main** opens on the card after the source panel's active card (for
    example, Context on top and Reply below)
  - a duplicate **Files** steps to the next file
- In an existing split, a capital pick is exactly `show_deck(other_index, deck)`, the
  same path `Ctrl+N` / `Ctrl+P` use on that panel.

### What the picker shows

Add one muted hint line at the bottom of the modal body that tells the user exactly
where a capital letter goes. It reads as a sentence, and each capital letter is tinted
in its deck's accent (the same accents as the row keycaps):

```
╭───────────────────── Switch deck ──────────────────────╮
│ Choose what the deck panel shows                       │
│                                                        │
│▌› m  ◆ MAIN    2 cards      ● showing                  │
│▌     Context, prompt, and reply                        │
│   f  ▤ FILES   3 files                                 │
│      Diffs and files the agent touched                 │
│   t  λ TOOLS   empty                                   │
│      LLM tool-call timeline                            │
│                                                        │
│   M/F/T  open in a new bottom panel                    │
╰──── m/f/t pick · j/k move · enter select · esc close ──╯
```

In a left-right split, with the right panel focused:

```
│ Choose what the right panel shows                      │
│   m  ◆ MAIN    2 cards      ○ in left panel            │
│   …                                                    │
│   M/F/T  show in the left panel                        │
```

The hint phrases:

| State                | Hint phrase                                 |
| -------------------- | ------------------------------------------- |
| Single               | `open in a new bottom panel`                |
| Split                | `show in the <top/bottom/left/right> panel` |
| Zoomed, from single  | `open in a new bottom panel · ends zoom`    |
| Zoomed, from a split | `show in the <pos> panel · ends zoom`       |

Layout details:

- Indent the hint so the capital `M` sits right under the rows' lowercase keycap
  letters. The lowercase and capital columns then read as a pair.
- The border-subtitle legend stays unchanged. Lowercase lives in the legend, capitals
  live in the hint, and nothing gets squeezed.
- Use no new glyphs. Everything above (`◆ ▤ λ ● ○ ›`, `·`) is already rendered by ACE,
  so there is no tofu risk.
- The modal grows by two rows (a blank line plus the hint), to about 12 rows.

### Keys and mouse

These details hold:

- Detect capitals from `event.character`, case-sensitively, so behavior doesn't depend
  on whether the terminal reports the key as `F` or `shift+f`.
- `Shift+J` / `Shift+K` / `Shift+Q` and other stray capitals stay swallowed.
- `Enter` and mouse clicks keep their "this panel" meaning. Don't add shift-click:
  terminals commonly use shift-click to bypass mouse reporting, so it is unreliable.
- If the modal is built without an other-panel hint (`other_hint=None`), capitals are
  swallowed like any other stray printable. This keeps the modal generic.
- Close-key collision handling already lowercases before comparing. If `pick_deck` is
  rebound to a capital deck letter (such as `F`), it is still dropped from the close
  keys. Add a test.

## Implementation

### 1. Pure layout helpers: `src/sase/ace/tui/widgets/decks/layout.py`

- `toggle_split(state, target, new_panel, *, focus_new: bool = True)`. From SINGLE,
  focus becomes `1 if focus_new else 0`. Unsplit and rotate ignore the flag. Existing
  callers are unchanged.
- `new_panel_for_deck(deck, current_deck, current_active_card, card_ids) -> DeckPanelState`:
  - when `deck is current_deck is DeckId.MAIN`, return
    `DeckPanelState(MAIN, cycle_card_id(card_ids, current_active_card, 1))`
  - otherwise return `DeckPanelState(deck)`

  Make `choose_new_panel`'s duplicate fallback call it, so both paths share one rule.

- `exit_zoom_keeping_panels(state) -> DeckAreaState`:
  - If not zoomed, return `state` unchanged.
  - Otherwise return the snapshot's layout, ratio, and `nodes_collapsed`, combined with
    the **current** `panels` and `focused`.
  - Use `dataclasses.replace(snapshot, panels=state.panels, focused=state.focused)`.
  - Keeping the current panels matters: a deck changed with `Ctrl+N` while zoomed must
    survive. `Z`'s exact-snapshot restore stays as it is.

### 2. Pure picker model: `src/sase/ace/tui/widgets/decks/picker.py`

- `@dataclass(frozen=True) OtherPanelTarget` with these fields:
  - `panel_index: int`: the logical index that will show the deck
  - `label: str`: `top` / `bottom` / `left` / `right`
  - `opens_split: bool`
  - `ends_zoom: bool`
- `other_panel_target(state: DeckAreaState, source_index: int) -> OtherPanelTarget`:
  - `base` is `state.zoom_snapshot` when zoomed, else `state`
  - `ends_zoom` is `is_zoomed(state)`
  - when `base.layout is SINGLE`: `panel_index=1`, `label="bottom"`, `opens_split=True`
  - otherwise: `panel_index = 1 - source_index` (clamp the source to 0/1),
    `label = panel_position_label(base, panel_index)`, `opens_split=False`
- `deck_picker_other_hint(target) -> str` returns the phrases from the table above.
- `DeckPickerState` gains `other_target: OtherPanelTarget | None = None`. The default
  keeps existing constructors and tests valid.
- `@dataclass(frozen=True) DeckPick` with fields `deck: DeckId` and `other_panel: bool`.
  This becomes the modal's result type.
- Leave the existing "no other-panel badge while zoomed" row behavior as it is.

### 3. Modal: `src/sase/ace/tui/modals/deck_picker_modal.py`

- Change the base class to `ModalScreen[DeckPick | None]`.
- Add a constructor parameter `other_hint: str | None = None`.
- Lowercase letters, `Enter`, and clicks dismiss with `DeckPick(deck, False)`.
- A capital letter `c`, where `c.lower()` is a deck letter and `other_hint` is set,
  dismisses with `DeckPick(deck, True)`.
- Build the capital-letter map from the rows (`row.key.upper()`), never from hard-coded
  letters.
- When `other_hint` is set, compose a `Static#deck-picker-other-hint` after the rows:
  - build it with Rich `Text`
  - start with a 3-space indent
  - each capital is bold in its row's accent, joined by dim `/`
  - then two spaces and the dim phrase

  The plain text is `"   M/F/T  <phrase>"`.

### 4. Styles: `src/sase/ace/tui/styles.tcss`

In the `DeckPickerModal` block, add `#deck-picker-other-hint` with:

- `height: 1;`
- `margin-top: 1;`
- `padding: 0 1;`, matching the rows
- `color: $text-muted;`

### 5. AgentDetail: `src/sase/ace/tui/widgets/_agent_detail_deck_layout.py` and `_agent_detail_decks.py`

- Refactor `toggle_deck_split`'s SINGLE branch into
  `_open_deck_split(target, deck: DeckId | None = None, *, focus_new: bool = True) -> bool`:
  - `deck=None` is today's `choose_new_panel` behavior. `toggle_deck_split` calls it
    unchanged, and `\` / `|` must behave exactly as before.
  - An explicit `deck` uses `new_panel_for_deck(...)`.
  - Keep `focus_new` threaded into `toggle_split` and keep the `was_zoomed_from == 1`
    repaint.
  - Keep the duplicate-Files `next_file()` step, the availability refresh, and
    `_notify_deck_state_changed()`.
- New `show_deck_in_other_panel(panel_index: int | None, deck: DeckId) -> bool` on
  `AgentDetailDeckLayoutMixin`:
  1. Resolve the source panel the same way `apply_picked_deck` does: `panel_index` if it
     is still visible, else the focused panel.
  2. `target = other_panel_target(area.state, source)`.
  3. If `target.ends_zoom`:
     - `area.apply_state(exit_zoom_keeping_panels(state))`
     - `self._sync_nodes_collapsed_chrome()`
     - refresh the focused panel's chrome
  4. If `target.opens_split`, return
     `self._open_deck_split(DeckLayout.TOP_BOTTOM, deck, focus_new=False)`. Return
     `True` if the zoom ended, even when the split fails.
  5. Otherwise, if the zoom did not end and
     `area.panel(target.panel_index).deck is deck`, return `False`.
  6. Otherwise call `self.show_deck(target.panel_index, deck)` (it persists) and return
     `True`.

  Wrap each step in the same defensive `try` / `except` style as its neighbors.

- `deck_picker_state()` fills `other_target=other_panel_target(state, panel_index)`.

### 6. App actions: `src/sase/ace/tui/actions/agents/_panel_detail.py`

- `action_pick_deck`:
  - Pass `other_hint = deck_picker_other_hint(state.other_target)` when `other_target`
    is set.
  - `_on_choice` accepts a `DeckPick`. It routes `other_panel=True` to
    `detail.show_deck_in_other_panel(state.panel_index, pick.deck)` and everything else
    to `apply_picked_deck` as today.
  - It refreshes the footer (`_refresh_agent_footer_bindings_only`) whenever something
    changed. Split-only keys such as `Ctrl+F` become available after a new split.
  - Keep the post-await re-checks: the tab is still Agents, and query the detail again.
- New palette-only `action_show_deck_other_at(index: int)`. It mirrors
  `action_show_deck_at`, but calls `show_deck_in_other_panel(None, DECK_CYCLE[index])`.

### 7. Discoverability

- **Palette**: in `commands/catalog.py`, extend `_iter_deck_picker_commands` to also
  yield `agents.show_deck_other.<value>`:
  - label `Show <Name> deck in other panel`
  - key sequence `(pick_deck, letter.upper())`, displayed `p F`; empty when `pick_deck`
    is unbound
  - category Navigation, `AGENTS_ONLY`
  - executor `app_action` / `show_deck_other_at` / `digit=index`
  - aliases: `deck`, `<value>`, `<value> deck`, `other panel`, `split`
  - In `commands/_availability_agents.py`, report `agents.show_deck_other.` ids as
    available. The existing `agents.show_deck.` prefix check does **not** match them,
    because of the dot.
- **Help**: in `modals/help_modal/agents_bindings.py`, add a Navigation row right after
  the `Pick deck for focused panel (decks)` row:

  ```python
  (f"{d(a.pick_deck)} {capitals}", "Show deck in other panel (decks)")
  ```

  Here `capitals` is `"/".join(DECK_PICKER_KEYS[deck].upper() for deck in DECK_CYCLE)`,
  so it renders `p M/F/T`. Handle an unbound `pick_deck` the same way the existing
  picker row does.

- **No keymap or `default_config.yml` change.** The capitals are fixed in-picker keys,
  like the lowercase letters, so no new app action binding is added.

### 8. Tests

- **`tests/ace/tui/widgets/decks/test_deck_layout.py`**:
  - `toggle_split(..., focus_new=False)` from SINGLE keeps focus 0, and the default
    still focuses 1
  - unsplit and rotate ignore the flag
  - `new_panel_for_deck` gives a duplicate Main the next card, wrapping, and gives a
    non-Main or non-duplicate panel no preferred card
  - `exit_zoom_keeping_panels`:
    - restores the layout, ratio, and `nodes_collapsed` from the snapshot, while keeping
      the current panels (including a deck changed while zoomed) and focus
    - returns an unzoomed state unchanged
- **`tests/ace/tui/widgets/decks/test_deck_picker.py`**:
  - `other_panel_target` for:
    - single → index 1 / bottom / opens split
    - top-bottom with focus 0 → 1 / bottom, and with focus 1 → 0 / top
    - left-right with focus 0 → right, and with focus 1 → left (`opens_split` False)
    - zoomed from single → opens split + ends zoom
    - zoomed from left-right with focus 1 → index 0 / left / ends zoom
  - the four `deck_picker_other_hint` phrases
- **`tests/ace/tui/modals/test_deck_picker_modal.py`**:
  - Replace the "case-insensitive" test: `m` / `f` / `t` → `DeckPick(deck, False)`, and
    `M` / `F` / `T` → `DeckPick(deck, True)`.
  - Update every result assertion to the `DeckPick` type.
  - Capitals are swallowed and the modal stays open when `other_hint=None`.
  - The hint's plain text is `"   M/F/T  open in a new bottom panel"`.
  - `Enter` and clicks return `other_panel=False`.
  - A `pick_deck` rebound to `F` is dropped from the close keys, so `F` still picks
    Files for the other panel.
- **New `tests/ace/tui/widgets/decks/test_deck_other_panel.py`** (widget pilot, using
  the `_DetailApp` / `make_artifact_agent` / `pin_paged` pattern from
  `test_deck_split_pilot.py`):
  - Single: `show_deck_in_other_panel(None, FILES)` produces TOP_BOTTOM, panel 1 FILES,
    panel 0 unchanged, `focused == 0`, returns True.
  - Top-bottom with focus 1: TOOLS lands on panel 0, the layout and focus are unchanged.
  - Left-right with focus 1: the chosen deck lands on panel 0 and the layout is still
    `LEFT_RIGHT`.
  - The other panel already shows the deck: returns False and state is unchanged.
  - A duplicate Main from single: panel 1 prefers the card after the active one.
  - Zoomed from single: not zoomed, TOP_BOTTOM, `nodes_collapsed` back to its pre-zoom
    value, focus 0.
  - Zoomed from left-right with focus 1: `LEFT_RIGHT` is restored, panel 0 shows the
    deck, focus is 1, returns True. When panel 0 already showed it, the result is still
    True because the zoom ended.
  - `toggle_deck_split` still focuses the new panel. This is a regression guard for the
    refactor.
- **`tests/ace/tui/test_agents_deck_picker.py`** (mounted `AcePage`):
  - `p` then `F` on a single Main deck gives a top-bottom split with Files below, focus
    on top, the modal closed, and a deck-state save scheduled.
  - After `vertical_line` (right focused), `p` shows the heading `right panel` and the
    hint `show in the left panel`. `T` puts Tools on the left, and the layout stays
    left-right.
  - A capital letter for the other panel's current deck changes nothing.
  - Palette `action_show_deck_other_at(1)` from single opens Files below.
- **Keymap, help, and palette tests**:
  - `tests/test_keymaps_display_help_agents.py` asserts the `p M/F/T` row.
  - `tests/test_command_catalog_build.py`: `agents.show_deck_other.files` shows `p F`,
    follows a rebound `pick_deck`, and is empty when `pick_deck` is unbound.
  - `tests/test_command_availability_scope.py`: the new ids are available only on
    Agents.
  - `tests/test_command_execution.py`: executing `agents.show_deck_other.tools` routes
    to `show_deck_other_at(2)`.
- **Visual** (`tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_picker.py`):
  - The existing `agents_deck_picker_single_120x40` and
    `agents_deck_picker_split_bottom_120x40` goldens change. They gain the hint line
    (`open in a new bottom panel` / `show in the top panel`).
  - Add `agents_deck_picker_left_right_120x40`: after `vertical_line` with the right
    panel focused, open the picker. It shows the `right panel` heading, the
    `○ in left panel` badge, and the `show in the left panel` hint, covering the
    vertical case explicitly.
  - No deck empty-state golden should change. The `_deck_switch_hint` text is untouched.

### 9. Docs

- **`docs/ace.md`**:
  - In "Agents Deck Picker", add a short paragraph for the capital-letter rule:
    - single → new bottom panel
    - split → the other panel, orientation kept
    - zoomed → the zoom ends as with `Z` first
    - focus stays on the panel the picker was opened from
    - a capital for the deck the other panel already shows is a no-op
  - Mention the new palette commands.
  - Update both Agents key-table `p` rows, for example: "Pick the focused panel's deck:
    `m` Main, `f` Files, `t` Tools; `M`/`F`/`T` show it in the other panel (opening one
    below if needed); `pp`/`Esc` close".
  - In the "Agent Data Decks and Cards" split paragraph, add one sentence noting that
    `p` plus a capital deck letter opens or fills the other panel with a chosen deck.
- **`docs/configuration.md`**: in the deck-picker paragraph, say that the in-picker
  letters and their capitals are fixed and not configurable.
- `CHANGELOG.md` is generated; don't edit it.

## Verification

- First read the `tui`, `tui_perf`, `tui_screenshot`, and `lint_and_test` memory notes
  with `/sase_memory_read`.
- Run `just fix` (or `just fmt`), then `sase tool run check`. Symvision rejects public
  symbols used only in their own file, so make any such helper private.
- Run `just fix-tui-screenshots` through `/sase_monitor`, targeted at
  `test_ace_png_snapshots_agents_deck_picker.py`. Inspect every created or updated
  golden before accepting it:
  - the two existing picker goldens differ only by the new hint line
  - the new left-right golden shows the right heading, badge, and hint
  - no glyph renders as tofu
- Capture live `sase screenshot`s:
  - the picker in single, top-bottom, and left-right layouts
  - the result of `p F` from single, where focus chrome stays on the top panel

  Check the hint alignment and the accent tints in the default theme.

- Do not run `just check-full` unless explicitly asked.

## Out of scope

- Changing `\` / `|` focus behavior, or `Z`'s exact-snapshot restore.
- Shift-click and `Shift+Enter` variants.
- Showing the hidden panel's badge while zoomed.
- Configurable in-picker letters.
- Any Rust core change. This is presentation-only TUI state, so it stays in this repo.
- A latent issue noticed during planning. `with_panel_deck` updates the zoomed state's
  panels but not `zoom_snapshot`, so `Ctrl+N` while zoomed followed by `Z` can leave the
  restored state's deck out of sync with the widget. Don't fix it here. If you confirm
  it, file a bug task bead through `/sase_new_task`.
