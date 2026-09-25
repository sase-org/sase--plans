---
tier: tale
title: Agents tab deck picker on p
goal:
  On the Agents tab, pressing p opens a small, polished deck picker, and one more
  keypress (m, f, or t) switches the focused deck panel to that deck. The picker is
  predictable in split and zoom layouts, never leaks keys, and is easy to discover and
  extend when new decks are added.
size: medium
proposed_by: bbugyi200.apollo.1p
create_time: 2026-09-25 10:12:40
status: wip
---

# Plan: Agents tab deck picker (`p`)

## Why

`Ctrl+N` / `Ctrl+P` cycle the focused deck panel through `DECK_CYCLE` (Main → Files →
Tools). That's fine for three decks, but it gets slower with every deck added, and the
user has to remember the order. A picker turns "show deck X" into two keys (`p`, then
the deck's letter), no matter how many decks exist. `p` has been free on the Agents tab
since the old detail-view picker (`choose_agent_view`) was retired. The Artifacts tab
keeps `p` for `pick_artifacts_project`; the two tabs never overlap.

## UX design

### Opening

On the Agents tab, `p` (new app action `pick_deck`) opens a centered `DeckPickerModal`
that acts on the **focused** deck panel. It won't open if any of these hold:

- the tab is not Agents
- a prompt bar owns keys
- another modal is already showing

It follows the same guards as `choose_agent_grouping`.

### Mock-up

This is what the single layout looks like, with Main shown and Tools known to be empty:

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
╰──────── m/f/t pick · j/k move · enter · esc close ─────╯
```

### Title, heading, and footer

- **Border title:** `Switch deck`.
- **Heading:** one muted line, `Choose what the <panel> shows`. `<panel>` is
  `deck panel` in the single layout. In a top-bottom split it is `top panel` or
  `bottom panel`. In a left-right split it is `left panel` or `right panel`. While
  zoomed it is `zoomed panel`. Logical panel 0 is always the top or left widget.
- **Border subtitle:** a muted key legend built from the live deck letters, for example
  `m/f/t pick · j/k move · enter select · esc close`. Trim the tail words if the legend
  would not fit the container.

### Rows

There is one two-line row per deck in `DECK_CYCLE` order. That order matches the
`main · files · tools` subtitle switcher on the deck panel border.

Line 1 contains, in order:

- the pointer (`›` on the highlighted row)
- the deck's letter as a keycap chip filled with the deck accent
- the deck glyph (`DECK_GLYPHS`)
- the bold deck name in its accent
- a count label
- a badge

Line 2 is a short muted blurb.

**Accents** match the deck panel borders: Main uses the live theme `secondary`, Files
uses `green`, and Tools uses `#87D7FF`. Resolve them through the panel chrome, as
`DeckPanelChromeMixin._resolve_accent` does, so theme changes are respected.

**Count label** comes from the focused panel's `DeckAvailability`:

- `1 card` / `N cards` for Main, `1 file` / `N files` for Files, `1 call` / `N calls`
  for Tools
- `empty` when `has_content is False`
- blank when the count is unknown, so the picker never shows a misleading number

**Badges:**

- `● showing` (accent, bold) marks the focused panel's current deck.
- In a split (not zoomed), the deck shown in the _other_ panel gets a muted
  `○ in <top|bottom|left|right> panel`.

**Empty decks** show their name and blurb dimmed, and their keycap is outline-styled
(accent text, no fill). They stay selectable, which matches `Ctrl+N`/`Ctrl+P`: nothing
is skipped, and the panel's empty-state card explains what's missing.

**Highlighted row** gets a subtle `$primary` tint and a thick left bar in the deck's
accent.

### Keys inside the picker

Every key is handled in the modal's `on_key`. Every printable key is consumed so nothing
leaks through to the Agents tab.

- **Deck letter** (`m` Main, `f` Files, `t` Tools; case-insensitive): pick that deck and
  close right away. This is the single keypress.
- **Enter:** pick the highlighted row. The cursor starts on the current deck.
- **`j`/`k`/`Down`/`Up`:** move the highlight. It **wraps**, the same way deck cycling
  does.
- **`Esc`, `q`, or the live `pick_deck` key** (so `pp` toggles the picker closed):
  cancel. If the `pick_deck` key collides with a deck letter or `j`/`k`, drop it from
  this set.
- **Mouse click** on any part of a row picks that deck.
- Any other printable key is swallowed, and the modal stays open.

### Applying a choice

- Picking the deck that is already showing closes the picker without changing anything
  and without a toast.
- Any other deck goes through the existing `AgentDetail.show_deck(panel_index, deck)`
  path, the same one `Ctrl+N`/`Ctrl+P` use. That path already handles:
  - loading, including duplicate-deck cache reuse across split panels
  - availability refresh
  - chrome refresh
  - deck-state persistence to `~/.sase/ace_agents_deck_state.json` via
    `_notify_deck_state_changed`

  After it, refresh the Agents footer (`_refresh_agent_footer_bindings_only`), because
  the `cards` hint depends on the deck.

- Counts are a snapshot taken when the picker opens. The modal does no refresh, polling,
  or I/O while it is open.

## Implementation

### 1. Deck catalog: `src/sase/ace/tui/widgets/decks/titles.py`

Next to `DECK_GLYPHS` / `DECK_NAMES` / `DECK_ACCENTS`, add:

- `DECK_PICKER_KEYS: dict[DeckId, str] = {MAIN: "m", FILES: "f", TOOLS: "t"}`
- `DECK_BLURBS: dict[DeckId, str]`:
  - Main: "Context, prompt, and reply"
  - Files: "Diffs and files the agent touched"
  - Tools: "LLM tool-call timeline"
- `DECK_COUNT_NOUNS: dict[DeckId, tuple[str, str]]` (singular, plural): card/cards,
  file/files, call/calls
- `DECK_PICKER_RESERVED_KEYS = frozenset({"j", "k", "q", "p"})`

This keeps every per-deck descriptor in one place. Adding a deck later means adding one
entry to each map, and the guard test in step 9 fails if one is missing. Keep the module
light: `commands/catalog.py` will import from it.

### 2. Pure picker model: new `src/sase/ace/tui/widgets/decks/picker.py`

This module does no I/O and has no Textual imports. Import `DeckAvailability` only under
`TYPE_CHECKING`, because `availability.py` pulls in the file-panel and LLM-calls
modules.

- `panel_position_label(state: DeckAreaState, panel_index: int) -> str`: returns
  `"deck"` for the single layout, `"top"` / `"bottom"` for top-bottom, `"left"` /
  `"right"` for left-right, and `"zoomed"` when `layout.is_zoomed(state)`.
- `@dataclass(frozen=True) DeckPickerState`: `panel_index`, `panel_label`,
  `current: DeckId`, `other: tuple[DeckId, str] | None` (the other visible panel's deck
  and its position label), `availability: Mapping[DeckId, DeckAvailability]`, and
  `accents: Mapping[DeckId, str]`.
- `@dataclass(frozen=True) DeckPickerRow`: `deck`, `key`, `glyph`, `name`, `accent`,
  `count_label`, `blurb`, `has_content: bool | None`, `is_current: bool`,
  `other_panel_label: str | None`.
- `build_deck_picker_rows(state) -> tuple[DeckPickerRow, ...]` in `DECK_CYCLE` order.
- `deck_count_label(deck, availability) -> str`, following the rules above.
- `deck_picker_heading(state) -> str` returns `Choose what the <label> panel shows`. The
  single layout reads `deck panel`.

### 3. Modal: new `src/sase/ace/tui/modals/deck_picker_modal.py`

`DeckPickerModal(ModalScreen[DeckId | None])` is modeled on `agent_grouping_modal.py`.
It takes `rows`, `heading`, and `close_keys: tuple[str, ...]`.

Compose:

- a `Container#deck-picker-container`, with `border_title` = "Switch deck" and
  `border_subtitle` = the legend
- a `Static#deck-picker-heading`
- one `Static#deck-picker-row-<i>` per row, with classes `deck-picker-row`,
  `-deck-<value>`, `focused`, `current`, and `empty` as they apply

Build row text with Rich `Text` in fixed-width columns so tests can assert on `.plain`.

Behavior:

- `on_key`, `on_click`, and a `_dismiss_once` guard, following the Keys section above.
- `_refresh()` updates rows in place and `scroll_visible(animate=False)` on the
  highlighted row.

Import it directly from its module, as the grouping picker does. Registering it in the
`modals` lazy-export table is optional.

### 4. Styles: `src/sase/ace/tui/styles.tcss`

Add a `DeckPickerModal` block after the `AgentGroupingModal` block:

- `align: center middle`
- container:
  `width: 56; max-width: 94%; height: auto; max-height: 90%; border: round $primary;`
  with a bold `$primary` border title, a `$text-muted` border subtitle, both centered,
  `background: $surface; padding: 1 2;`
- heading: muted, one line, bottom margin 1
- rows: `height: auto; padding: 0 1;`
- `.focused` rows: `background: $primary 15%; padding-left: 0;` with a thick left border
  per deck class (`-deck-main` → `$secondary`, `-deck-files` → `green`, `-deck-tools` →
  `#87D7FF`)

Keep the whole modal compact: about 10 terminal rows for three decks.

### 5. AgentDetail API: `src/sase/ace/tui/widgets/_agent_detail_decks.py` and `decks/panel.py`

- `DeckPanel`: add a public read-only `availability` property that returns a copy of
  `_availability`. Add a public `deck_accents()` wrapper around the chrome mixin's
  `_accent_for()`. This avoids private access from other modules and keeps Symvision
  clean.
- `AgentDetailDeckMixin.deck_picker_state() -> DeckPickerState | None` reads the deck
  area: focused panel, its deck, availability and accents, the other visible panel (none
  when single or zoomed), and `panel_position_label`. It returns `None` on any failure.
- `AgentDetailDeckMixin.apply_picked_deck(panel_index: int | None, deck: DeckId) -> bool`
  resolves the target panel index:
  - use `panel_index` if it is still among `visible_panels()`
  - otherwise use the focused panel
  - `None` means the focused panel

  It returns `False` without doing anything when the target already shows `deck`.
  Otherwise it calls `self.show_deck(index, deck)` and returns `True`.

### 6. App action and palette action: `src/sase/ace/tui/actions/agents/_panel_detail.py`

Put these next to `action_next_deck` / `action_prev_deck`.

`action_pick_deck()`:

1. Guard: the tab is Agents and the screen is not already a `ModalScreen`.
2. Get `AgentDetail` and `deck_picker_state()`, and build the rows and heading.
3. Take the close keys from
   `split_key_alternatives(self._keymap_registry.app.pick_deck)`.
4. `push_screen(DeckPickerModal(...), _on_choice)`.

`_on_choice` then:

1. Returns early if the result is `None` or the tab is no longer Agents. Re-check after
   the await, per `tui_perf` rule 4.
2. Re-queries the detail and calls `apply_picked_deck(state.panel_index, deck)`.
3. Refreshes the footer.

`action_show_deck_at(index: int)`:

- It is palette-only and has no keymap field.
- Guard: the tab is Agents and `0 <= index < len(DECK_CYCLE)`.
- Call `apply_picked_deck(None, DECK_CYCLE[index])` and refresh the footer.

### 7. Keymap plumbing (new action id `pick_deck`)

Do **not** reuse the retired `choose_agent_view` id. It stays in `_RETIRED_APP_KEYS`, so
stale overrides are still dropped quietly.

- `src/sase/default_config.yml` (`ace.keymaps.app`, after `prev_deck`): add
  `pick_deck: "p"` with this comment: "Agents-only deck picker. Shares p with
  pick_artifacts_project (Artifacts); disambiguated by tab availability."
- `keymaps/app_keymaps.py`: add a `pick_deck: str` field after `prev_deck`.
- `keymaps/metadata.py`: add `("pick_deck", "Pick Deck", False)` after `prev_deck`.
- `bindings.py`: add a fallback `Binding("p", "pick_deck", "Pick Deck", show=False)`
  next to the deck bindings.
- `keymaps/registry.py` `_CONTEXTUAL_APP_DUPLICATES`: add
  `frozenset({"pick_deck", "pick_artifacts_project"})` with a comment ("Tab-disjoint:
  Agents deck picker vs Artifacts project scope").
- `_app_action_availability.py`:
  - Add `"pick_deck"` to `_DECK_NAV_ACTIONS`. That makes it Agents-only and blocked
    while the prompt owns keys.
  - Also return `False` while a `ModalScreen` is active, mirroring
    `choose_agent_grouping`.
- `actions/agents/_deck_search_host.py` `deck_structural_exit_keys`: add `"pick_deck"`,
  so `p` during a committed deck search exits the frozen search overlay first. Without
  this, the overlay would stay pinned to the old deck.

### 8. Discoverability

- **Help:** in `modals/help_modal/agents_bindings.py` (Navigation), add
  `(d(a.pick_deck), "Pick deck for focused panel (decks)")` just before the "Cycle deck
  (decks)" row.
- **Palette `app.pick_deck`:** in `commands/_app_metadata_nav.py`, add
  `("pick_deck", "Pick deck for focused panel", "Navigation", AGENTS_ONLY, ("deck", "picker", "switch deck", "p"))`
  after `prev_deck`. In `_availability_agents.py`, add `"app.pick_deck"` to the
  always-available deck set.
- **Palette direct commands:** in `commands/catalog.py`, add
  `_iter_deck_picker_commands(registry)`. It mirrors `_iter_agents_panel_layout_command`
  and yields one spec per deck in `DECK_CYCLE`:
  - id: `agents.show_deck.<value>`
  - label: `Show <Main|Files|Tools> deck in focused panel`
  - key sequence: `(pick_deck, letter)`, displayed like `p f`; empty when `pick_deck` is
    unbound
  - category: Navigation; tabs: `AGENTS_ONLY`
  - executor: `CommandExecutor(kind="app_action", action="show_deck_at", digit=index)`
  - aliases: `deck`, `<value>`, `<value> deck`, `switch deck`

  Register it in `build_command_catalog` right after the panel-layout command. Make sure
  the Agents availability predicate reports these ids as available.

- **Empty-state hint:** in `decks/panel_chrome.py`, change `_deck_switch_hint` to
  `"<pick> pick deck · <next>/<prev> cycle decks"` when `pick_deck` is bound, and keep
  today's text as the fallback.
- Leave the slow-tool overflow hint (`_agent_slow_tools.py`) unchanged. Its `^N/^P`
  guidance is still correct.

### 9. Tests

- **New `tests/ace/tui/widgets/decks/test_deck_picker.py`** (pure):
  - catalog guard: every `DECK_CYCLE` deck has a picker key, blurb, and count noun; keys
    are unique single lowercase characters, outside `DECK_PICKER_RESERVED_KEYS`
  - `deck_count_label`: singular, plural, `empty`, and unknown-blank
  - `panel_position_label` for single, top-bottom, left-right (both indices), and zoomed
  - `build_deck_picker_rows`: order, current badge, other-panel label, and no
    other-panel label when zoomed or single
  - heading text
- **New `tests/ace/tui/modals/test_deck_picker_modal.py`:**
  - each letter dismisses with its `DeckId`, and uppercase works
  - a stray printable (such as `x`) keeps the modal open with no result
  - `Esc`, `q`, and the close key dismiss with `None`
  - `j`/`k` wrap in both directions, and `Enter` picks the highlighted row
  - the cursor starts on the current deck
  - clicking a row picks it
  - the dismiss-once guard holds
- **New `tests/ace/tui/test_agents_deck_picker.py`** (mounted
  `AcePage(initial_tab="agents")`):
  - `p` opens `DeckPickerModal`, and `f` switches panel 0 to Files and schedules the
    deck-state save
  - `pp` closes with no change
  - picking the current deck changes nothing
  - after `backslash` (bottom panel focused), `p` then `m` changes only panel 1; the
    heading says `bottom panel` and the other-panel badge names `top`
  - calling `action_pick_deck` again while the picker is open does not stack a second
    modal
  - `p` on the Artifacts tab still resolves to `pick_artifacts_project`, not the picker
  - `p` does nothing while the prompt bar owns keys
  - `p` during a committed deck search exits the search and opens the picker
- **Keymaps:**
  - extend
    `tests/test_keymaps_app_bindings.py::test_default_jump_and_deck_navigation_keys_are_unique`
    so the actions bound to `p` are exactly `pick_artifacts_project` and `pick_deck`
  - add a registry test: the pair is accepted, and a third app action on `p` is still
    rejected
- **Help and palette:**
  - `tests/test_keymaps_display_help_agents.py` asserts the new help row
  - palette tests check that `app.pick_deck` is available only on Agents
  - `agents.show_deck.files` shows `p f`, follows a rebound `pick_deck`, and shows empty
    when `pick_deck` is unbound
  - executing `agents.show_deck.tools` shows Tools in the focused panel
- **Empty-state hint:** update or add a unit test for the new `_deck_switch_hint` text,
  covering both the bound and unbound `pick_deck` cases.
- **Visual:** new `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_picker.py`
  with two goldens, using the deck fixtures and helpers from
  `test_ace_png_snapshots_agents_decks.py`:
  - `agents_deck_picker_single_120x40`: the picker open over a Main deck with Context
    and Reply cards
  - `agents_deck_picker_split_bottom_120x40`: after `backslash`, showing the heading and
    the other-panel badge

  Use only glyphs that ACE already renders (`◆ ▤ λ ● ○ ›`), and confirm in the PNG that
  none of them render as tofu.

### 10. Docs

- **`docs/ace.md`:**
  - Replace the "Agents Detail View Picker (retired)" section with "Agents Deck Picker",
    describing the design above. Keep one sentence noting that the old view picker is
    retired.
  - Add rows to both Agents key tables: `p` ("Pick the focused panel's deck: `m` Main,
    `f` Files, `t` Tools; `pp`/`Esc` close") next to the `Ctrl+N` / `Ctrl+P` rows.
  - Mention `p` in the "Agent Data Decks and Cards" intro.
- **`docs/configuration.md`:**
  - Add a `pick_deck` (`p`) row to the Agents app-key table.
  - Change the shared-key allowlist row from
    `(retired on Agents) / pick_artifacts_project` to
    `pick_deck / pick_artifacts_project`.
  - In the retired-keys paragraph, note that deck picking is now `pick_deck`.
  - State that the in-picker deck letters are fixed and not configurable, like the
    grouping picker's letters.
- `CHANGELOG.md` is generated; don't edit it.

## Verification

- Read the `lint_and_test`, `tui_screenshot`, and `tui_perf` memory notes first.
- Run `just fmt` (or `just fix`), then `sase tool run check`.
- Run `just fix-tui-screenshots` through `/sase_monitor`. Target the new deck-picker
  file, `test_ace_png_snapshots_agents_decks.py`, and any other snapshot whose deck
  empty-state hint changed. Inspect every created or updated golden before accepting it:
  the empty-state goldens change only in the hint line, and the new picker goldens look
  right.
- Capture a live `sase screenshot` with the picker open (press `p` on the Agents tab),
  both single and split. Look over the spacing, accents, and highlight in the default
  theme.
- Do not run `just check-full` unless explicitly asked.

## Out of scope

- A "previous deck" flip, or choosing a panel other than the focused one. `Ctrl+F`
  already moves focus.
- Configurable in-picker letters.
- Changing the slow-tool overflow hint.
- Any Rust core change. This is presentation-only TUI state, so it stays in this repo.
