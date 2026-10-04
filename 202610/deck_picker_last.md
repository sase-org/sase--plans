---
tier: tale
title: Deck picker last-deck return
goal: "On the Agents tab, the deck picker opened by p offers a p row that returns the
  focused deck panel to its last deck, or to the same deck Ctrl+P would select when that
  panel has no history yet, so pp toggles two decks.

  "
size: medium
proposed_by: bbugyi200.athena.0wa
create_time: 2026-10-04 07:12:55
status: wip
---

# Plan: Deck picker last-deck return

## Outcome

Pressing `p` on the Agents tab still opens the centered deck picker for the focused deck
panel. The picker gains one return row, bound to the same opener. A second press of that
opener (`pp` with the default binding) switches the focused panel to a resolved deck and
closes the picker. Pressing it again switches back. `Esc` and `q` still close without
changing anything.

The return target is the deck that panel was showing immediately before its current
deck, when that history exists. Otherwise it is the deck `Ctrl+P` (`prev_deck`) would
select: one step backward in the active cycle, wrapping. The row always names the
destination before the key is pressed.

This replaces today's "`pp` closes the picker" behavior. That change is intentional.
Cancel stays on `Esc` and `q`.

## Decisions

History is one slot per deck panel, not a stack and not a global pair.

- Every successful deck change on a panel stores the deck it just left as that panel's
  `previous_deck`. Picker letters, palette jumps, `Ctrl+N` / `Ctrl+P`, and a capital
  letter that changes the other panel all count, and each updates only the panel whose
  deck actually changed.
- A change that leaves the panel on the same deck does not touch `previous_deck`.
  Re-picking the current letter, or a capital letter for a deck the other panel already
  shows, keeps the toggle partner.
- `pp` then `pp` therefore swaps exactly two decks. `Main → Files → Tools` remembers
  Files, not Main. The next `pp` returns to Files, and the one after that returns to
  Tools.
- A new panel starts with no history, including a panel opened by `\` , `|`, or a
  capital letter. Do not copy the source panel's history into it. The source panel keeps
  its own.
- History is a property of the panel, not of the selected node or attempt. Moving the
  selection does not clear it. Swapping panels (`>` / `<`) already moves pane ids with
  their `DeckPanelState`, so history follows the session. Closing a panel drops its
  history with it.
- While zoomed, a deck change updates the live panel the same way it does today. `Z`
  still restores the zoom snapshot exactly, so both the deck and its `previous_deck`
  revert together. Do not mirror the new deck into `zoom_snapshot` from
  `with_panel_deck`; view changes already have their own snapshot write, and deck
  changes do not. Persistence of a still-zoomed session keeps using
  `exit_zoom_keeping_panels`, so a save while zoomed keeps the live deck and its
  history.

No feature flag and no new config key. `pick_deck` stays the opener. The in-picker deck
letters stay fixed. The return gesture is "press the opener again," not a second
binding.

Do not edit anything under `sase/memory/`. `docs/ace.md` is the behavior contract.

## Resolution

Add a pure helper in `src/sase/ace/tui/widgets/decks/picker.py`:

```python
def resolve_back_deck(current: DeckId, previous: DeckId | None) -> tuple[DeckId, BackOrigin]:
```

`BackOrigin` is `LAST` or `PREVIOUS`.

- `LAST` when `previous` is in the active cycle and is not `current`.
- `PREVIOUS` otherwise: `cycle_deck_id(current, -1)`, the same step as
  `action_prev_deck`.
- If `current` is not in the cycle, return Main and `PREVIOUS`. Never raise on this
  path.

The picker row, the palette actions, and the tests all call this helper. Do not
reimplement the rule in the modal.

## State

Add `previous_deck: DeckId | None = None` to `DeckPanelState` in
`src/sase/ace/tui/widgets/decks/model.py`. Default `None` so existing constructors stay
valid.

`with_panel_deck` records history only when the deck value changes:

- Different deck: set `previous_deck` to the deck being left.
- Same deck: leave `previous_deck` untouched. Prefer returning the same state object
  when both deck and history are unchanged, so a follow-up `show_deck` of the deck a
  panel already has cannot invent history.

That same-deck rule is load-bearing. `_open_deck_split` and
`_apply_agents_deck_snapshot` both call `show_deck` after the panel is already on the
target deck. Those calls must not overwrite a restored or freshly empty `previous_deck`.

`DeckPickerState` gains `previous: DeckId | None = None`. `deck_picker_state()` copies
it from the focused panel.

## Persistence

Store the slot in the existing `~/.sase/ace_agents_deck_state.json` snapshot. Do not
bump `schema_version`. Add an optional panel field `previous_deck`.

- Omit the key when the value is `None`, so a panel that has never switched decks
  serializes as it does today.
- On load, a missing key is `None`. An unknown deck, a deck outside the active cycle, or
  a value equal to that panel's current deck becomes `None` and does not fail the file.
  Log a warning for a present but unusable value, matching the fail-open style of
  `views`.
- Round-trip the field through `snapshot_from_area_state` / `area_state_from_snapshot`,
  including a still-zoomed area via `exit_zoom_keeping_panels`.
- The existing coalesced deck-state writer is the only save path. The picker key path
  does no disk I/O, no JSON work, and no toast.

## Picker row

The return row is not a fifth deck. `build_deck_picker_rows` stays one row per cycle
deck. Add `build_deck_picker_back(state) -> DeckPickerBackRow` next to it.

`DeckPickerBackRow` carries the resolved `DeckId`, the origin, the opener's display key,
and the destination's glyph, name, accent, count label, and empty flag. It has no deck
blurb.

`DeckPickerModal` takes the back row and the opener's key alternatives (`back_keys`).
`action_pick_deck` stops passing the opener as `close_keys`. Those alternatives become
`back_keys`. Drop any alternative that collides with `j`, `k`, `q`, `enter`, `escape`,
or a deck letter, using the same collision rule close keys use today. With the default
binding, `p` is free because `DECK_PICKER_RESERVED_KEYS` already reserves it.

Key handling, in order:

1. `Enter`, `Esc`, `q`, and `j` / `k` behave as they do now.
2. A `back_keys` match selects the return row for the focused panel.
3. When the other-panel hint is showing and the opener is a single letter, the capital
   of that letter (`P` by default) selects the same resolved deck for the other panel. A
   non-letter opener such as `f12` has no capital. Capitals of deck letters are
   unchanged.
4. Deck letters and their capitals are unchanged.
5. Every other printable key is still swallowed.

`Enter` and a click select the highlighted or clicked row for the focused panel. A click
never targets the other panel. The highlight still opens on the row whose deck is
current, not on the return row, so `Enter` immediately after `p` still re-affirms the
current deck and changes nothing.

`j` / `k` wrap across the return row plus the four deck rows.

Selecting the return row dismisses with a normal `DeckPick` of the resolved deck.
`apply_picked_deck` / `show_deck_in_other_panel` stay the appliers. A target that is
already showing is still a no-op and does not move history.

## Visual

The picker stays one centered round panel, title `Switch deck`, width 56. The return row
is a single line above the deck rows, with a quiet rule under it. Deck rows stay two
lines. The return row does not repeat a blurb and does not repeat the other panel's
`○ in … panel` note; the matching deck row already says that.

Unfocused return row, remembered history, Main as the destination:

```text
  p  ↩ ◆ MAIN          2 cards          last deck
```

Unfocused return row, no history yet, focused panel on Main (cycle wraps to FINAL):

```text
  p  ↩ ⊛ FINAL                              previous
```

Focused return row uses the existing `›` pointer, the existing focus wash, and the
destination deck's left-edge accent (`-deck-main`, `-deck-files`, `-deck-tools`,
`-deck-final`). The keycap is the same bold-on-accent pill as `m` / `f` / `t` / `n`. `↩`
(U+21A9), the destination glyph, and the destination name use that deck's accent. An
empty destination dims the name and shows `empty`, the way an empty deck row does.

The badge is the only word that distinguishes the two origins:

- `LAST`: `last deck`, bold, in the destination accent.
- `PREVIOUS`: `previous`, dim.

Put the badge at the right edge of the row. If the count would collide with it, drop the
count and keep the badge, the name, and the key. Meaning never depends on color alone.

Give the row the classes `deck-picker-row`, `deck-picker-back`, and the destination's
picker class. Style `deck-picker-back` with `margin-bottom: 1` and a dim `border-bottom`
so the return action sits apart from the catalog without becoming a second header. Do
not give it a second line of body text.

The border subtitle leads with the return gesture, then the deck letters, and keeps the
existing shorter tiers when the width is tight. Default shape:

```text
p back · m/f/t/n pick · j/k move · enter select · esc close
```

Use the live opener display (`F12`, not a hardcoded `p`) as the back token. The
other-panel hint gains that capital only when the capital gesture exists, colored with
the destination accent:

```text
P/M/F/T/N  open in a new bottom panel
```

`P` is the capital of the opener, not a new deck letter. When the remembered deck is
Main, `P` and `M` both send Main to the other panel. Leave that redundancy in place.

Leave the empty-deck hint (`p pick deck · Ctrl+N/Ctrl+P cycle decks`) unchanged. The
picker is the teacher; do not add a toast on the first toggle.

## Palette and help

Add two Agents-only palette commands beside the existing deck commands in
`src/sase/ace/tui/commands/catalog.py`:

- `agents.show_last_deck`, label `Show last deck in focused panel`, display `p p`
  (space-joined, same as `p f`). Executor: a new `action_show_last_deck` with no digit.
- `agents.show_last_deck_other`, label `Show last deck in other panel`, display `p P`.
  Emit this command only when the opener is a single letter with a distinct capital.
  Executor: `action_show_last_deck_other`.

Both actions resolve through `resolve_back_deck` on the focused panel, then call
`apply_picked_deck(None, deck)` or `show_deck_in_other_panel(None, deck)`. They no-op
when the target is already showing. They are available on the Agents tab under the same
tab gate as `agents.show_deck.*`, including when `pick_deck` is unbound (empty key
sequence, command still listed), and unavailable on Artifacts and Services.

An unbound or rebound opener updates both displays the way `agents.show_deck.files`
already follows `pick_deck`.

Help, in `src/sase/ace/tui/modals/help_modal/agents_bindings.py`:

- The `pick_deck` row reads `Pick a deck; press it again for the last deck`.
- The other-panel row includes the capital when it exists (`p P/M/F/T/N`) and its
  description says that capital shows the last deck in the other panel.

## Docs

Update `docs/ace.md` section `Agents Deck Picker` and the deck-cycle paragraph that
points at it:

- `pp` returns to the last deck, or to the cycle-previous deck when the focused panel
  has no history.
- The row shows the destination, `last deck` versus `previous`.
- `Esc` and `q` close.
- History is per panel, one deck deep, updated by every real deck change, and restored
  with the deck layout.
- `P` shows that same resolved deck in the other panel, with the existing split and zoom
  rules.
- Name FINAL alongside Main, Files, and Tools in any picker sentence this edit rewrites.

Update `docs/configuration.md` where it describes `pick_deck`: the opener is
configurable; pressing it again inside the picker is the return gesture and is not its
own binding; deck letters stay fixed. Mention `P` next to `M` / `F` / `T` / `N` when the
opener is a single letter.

## Tests

Pure model (`tests/ace/tui/widgets/decks/test_deck_picker.py` and `test_deck_model.py`):

- A fresh Main panel resolves to FINAL / `PREVIOUS`. After a change to Files, it
  resolves to Main / `LAST`.
- `previous` equal to current, unknown, or outside the cycle resolves as `PREVIOUS`.
- `with_panel_deck` sets `previous_deck` only when the deck changes, and a second call
  with the same deck keeps it.
- A zoomed deck change is visible on the live state and is gone after `_exit_zoom`,
  together with the history written during the zoom.

Persistence (`tests/ace/tui/models/test_agent_deck_persistence.py`):

- Omit `previous_deck` when unset, round-trip it when set, and ignore a bad value
  without dropping the rest of the panel.

Modal (`tests/ace/tui/modals/test_deck_picker_modal.py`):

- Replace the "`p` cancels" case. `p` selects the return deck for this panel. `P`
  selects it for the other panel when the hint is set, and is swallowed when the hint is
  absent.
- `Esc` and `q` still dismiss with `None`.
- Highlight opens on the current deck row. `k` from that row moves to the return row
  when the return row is above it; wrapping still reaches every row.
- The return row's plain text contains the key, `↩`, the destination name, and either
  `last deck` or `previous`, and does not contain a deck blurb.
- A back key that collides with a deck letter is dropped; the deck letter still wins.

Mounted app (`tests/ace/tui/test_agents_deck_picker.py`):

- Replace `test_agents_pp_closes_picker_without_change`. From a fresh Main panel, `pp`
  lands on FINAL and the next `pp` lands on Main.
- `p`, then `t`, then `pp` returns to Main (`LAST` wins over the cycle neighbor Files).
  Another `pp` returns to Tools.
- `Ctrl+N` after that updates history, so the next `pp` returns to the deck `Ctrl+N`
  just left.
- Re-picking the current deck letter does not change the partner.
- Two split panels keep independent history.
- `p` then `P` from a single panel opens the resolved deck below and leaves focus and
  the source panel's history unchanged. The new panel's first `pp` uses the cycle
  fallback.
- `p` then `P` when the other panel already shows that deck changes nothing and saves
  nothing.
- Palette `action_show_last_deck` and `action_show_last_deck_other` match the picker.

Catalog and help: extend `tests/test_command_catalog_build.py`,
`tests/test_command_execution.py`, `tests/test_command_availability_scope.py`, and
`tests/test_keymaps_display_help_agents.py` for the new commands, the `p p` / `p P`
display, rebound and unbound openers, and Agents-only availability.

Visual: the three existing picker goldens in
`tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_picker.py` will show the quiet
`previous` row (they open on a fresh Main panel). Add one golden that switches to Files
and opens the picker again, so the `last deck` row is in the set. Name it
`agents_deck_picker_last_deck_120x40`. Update goldens with the targeted
`just fix-tui-screenshots` invocation for that file, then read the report and look at
every changed PNG before treating them as current.

## Out of scope

- A history stack, a per-node memory, or a pair that ignores `Ctrl+N` / `Ctrl+P`.
- A configurable in-picker return letter distinct from `pick_deck`.
- Changing what `Ctrl+P` itself does outside the picker.
- A feature flag, a config field, a toast, or an empty-state hint rewrite.
- Any edit under `sase/memory/`.

## Verification

Run the focused unit tests listed above first. Then update and inspect the picker visual
goldens. Before finishing, follow the repo's agent verification recipe in the
lint-and-test memory, since this tale changes tracked TUI code, docs, and snapshots. The
picker key path must stay free of disk I/O and of any new refresh path.
