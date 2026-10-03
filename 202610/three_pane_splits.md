---
tier: epic
title: Three-pane splits for the Agents deck and the pager
goal: 'The Agents-tab deck and the sase pager share one closed split model with seven
  geometries: single, two two-pane splits, and four three-pane T shapes. The existing
  `\` / `|` keys follow one rule: erase a full-span divider, draw one through the
  focused pane, or turn a three-pane layout. New keys add a focus ring (`ctrl+f` /
  `ctrl+b`), content swap (`ctrl+shift+f` / `ctrl+shift+b`, aliases `>` / `<`), close-focused
  (`ctrl+shift+d`, alias `ctrl+x`) and turn (`ctrl+t`). Rendering is spatially stable,
  shows exactly one full-strength frame, and never remounts a pane. The work lands
  without colliding with the sase-1es pager performance epic.

  '
phases:
- id: terminal-chain
  title: Deliver the ctrl+shift chords through kitty and tmux
  depends_on: []
  size: small
  description: 'terminal-chain: in the linked chezmoi repo, unmap kitty''s ctrl+shift+f/b/o
    window maps and enable tmux CSI-u extended keys so ctrl+shift chords reach Textual,
    then record a manual verification checklist.'
- id: pane-grid-model
  title: Shared pure PaneGrid model and golden transition table
  depends_on: []
  size: medium
  description: 'pane-grid-model: add a stdlib-only PaneGrid algebra (split key rule,
    close, focus ring, swap, turn, resize, MRU target, fit, positions, grid spec).
    Pin it with a golden transition table, Hypothesis invariants, and an import-weight
    test.'
- id: deck-grid-adapter
  title: Agents deck on PaneGrid with flat grid rendering
  depends_on:
  - pane-grid-model
  size: medium
  description: 'deck-grid-adapter: rebuild DeckAreaState and DeckArea on PaneGrid
    with pane-ID-keyed panels and a flat CSS grid that never remounts. Remove every
    two-pane index assumption, keep the focused panel on same-key unsplit, and make
    split keys while zoomed only restore. Two-pane goldens stay byte-identical.'
- id: deck-pane-keys
  title: Agents deck reverse focus, swap, close, and turn keys
  depends_on:
  - deck-grid-adapter
  size: medium
  description: 'deck-pane-keys: add ctrl+b reverse focus, ctrl+shift+f/b swap (aliases
    > and <), ctrl+shift+d close (alias ctrl+x) and ctrl+t turn on the Agents tab.
    Wire keymaps, registry pairs, availability, palette, help and docs, and move the
    debug leak chord to f12.'
- id: deck-three-panels
  title: Agents deck three panels behind the three_pane_splits beta flag
  depends_on:
  - deck-pane-keys
  size: medium
  description: 'deck-three-panels: create the three_pane_splits beta flag and, behind
    it, enable the third deck panel: nest, turn and erase, fit refusal toasts, MRU
    picker target with position glyphs, 3-of-3 zoom chrome, additive persistence,
    a three-panel footer, a j/k perf check and T-shape goldens.'
- id: pager-grid-adapter
  title: Pager on PaneGrid with grid panes and the new pane keys
  depends_on:
  - pane-grid-model
  size: medium
  description: 'pager-grid-adapter: wait until sase-1es.6 has landed, then port the
    pager split onto PaneGrid. Use a flat grid #pager-panes with pane-ID-keyed views,
    close paths that remove only the discarded view (the sase-1er fix), and the ctrl+b,
    swap, close and turn keys on two panes. All pager goldens stay unchanged.'
- id: pager-three-panes
  title: Pager three panes with MRU ctrl+w and a target preview
  depends_on:
  - pager-grid-adapter
  - deck-three-panels
  size: medium
  description: 'pager-three-panes: behind three_pane_splits, enable pager nest, turn
    and erase, with transactional third-view mounts and fit refusal. Retarget ctrl+w
    to the MRU pane with a lifted target frame, show a three-pane footer and help,
    and add T-shape goldens.'
- id: unflag-docs
  title: Remove the flag and finish docs, help, glossary, and release note
  depends_on:
  - deck-three-panels
  - pager-three-panes
  size: small
  description: 'unflag-docs: delete the three_pane_splits Off branch and close its
    flag bead, then clear leftover epic-symbol entries. Finish docs/ace.md, docs/pager.md,
    configuration docs and help sheets, update the Deck Panel glossary strand, and
    write the changelog-facing commit message.'
proposed_by: bbugyi200.athena.0ve
create_time: 2026-10-02 11:33:47
status: done
bead_id: sase-1eu
---

- **PROMPT:** [prompts/202610/three_pane_splits.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/three_pane_splits.md)
- **BEAD:** [sase-1eu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1eu/README.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                                  | Why                                                                                  |
| ------------ | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| derives-from | [research:202610/deck_and_pager_three_pane_splits/deck_and_pager_three_pane_splits.md][1] | consolidated three-pane research whose recommendations the user adopted in full      |
| derives-from | [research:202610/three_pane_splits_before_sase_1es.md][2]                                 | sequencing analysis that orders Agents work first and gates pager work on sase-1es.6 |
| related      | [bead:sase-1er][3]                                                                        | pager close-path survivor bug that the pager adapter phase must leave fixed          |
| related      | [bead:sase-1es][4]                                                                        | pager performance epic; pager phases here wait for its ScrollView body phase         |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202610/deck_and_pager_three_pane_splits/deck_and_pager_three_pane_splits.md
[2]:
  https://github.com/sase-org/sase--research/blob/main/202610/three_pane_splits_before_sase_1es.md
[3]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1er/README.md
[4]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-1es/README.md

<!-- sase:links:end -->

# Plan: Three-pane splits for the Agents deck and the pager

## Summary

Today both surfaces support three layouts: one pane, or two panes stacked (`\`) or side
by side (`|`). The Agents deck lives in `src/sase/ace/tui/widgets/decks/`; the pager
split lives in `src/sase/pager/split.py` and `_screen_split.py` and came from epic
sase-1eg. Each surface has its own copy of the two-pane algebra, and both assume
`other = 1 - focused`.

This epic replaces both copies with **one shared, pure, closed model** (`PaneGrid`) with
seven geometries and at most three panes. Each surface gets a thin adapter that renders
the model through **one flat Textual CSS grid**, so no layout change ever remounts a
pane. Behavior follows the consolidated research report
(`research:202610/deck_and_pager_three_pane_splits/deck_and_pager_three_pane_splits.md`)
and its sequencing companion (`research:202610/three_pane_splits_before_sase_1es.md`).
The user adopted every recommendation in both, including the four open decisions: A2,
A3, A6, and the terminal-chain change.

Delivery is **Agents first**. The pager half waits for `sase-1es.6`, which is rewriting
the pager body and `_screen_split.py` right now. A default-off beta flag keeps the
opposite split key meaning "rotate" on both surfaces until both three-pane halves have
landed. The final phase then removes the flag in one change.

## Design decisions

### Adopted from the research (user-approved)

| #   | Decision                                                                                                                                                                                                                                                                                                |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | "The larger pane" means the structural **main pane**, the one that spans the full width or height. It is not decided by area.                                                                                                                                                                           |
| A2  | Erasing the full-span divider keeps **the side you are on**. With focus in the pair, the main pane closes. With the main pane focused, the main pane survives and the layout becomes single. A split key never closes the pane you are reading.                                                         |
| A3  | Same-key unsplit keeps the **focused** pane on both surfaces. This changes the Agents deck, which keeps panel 0 today. The old `\` `\` peek becomes `\` … `ctrl+x`.                                                                                                                                     |
| A4  | Splitting the focused pane opens the new pane right of or below it, and the new pane takes focus. The unfocused pane becomes main and neither moves nor resizes. This yields four T geometries, not two.                                                                                                |
| A5  | Switching between the three-pane views is a **turn**: a transpose that flips the axis, so main-top ↔ main-left and main-bottom ↔ main-right. Pair order, focus and both ratios are kept. A turn is self-inverse and pins the top-left pane.                                                             |
| A6  | `ctrl+t` turns any split (R2 ↔ C2, R3 ↔ C3), restoring the one-key rotation that the opposite split key no longer provides.                                                                                                                                                                             |
| A7  | The chords stay as defaults, plus single-key aliases: `>` / `<` swap and `ctrl+x` closes. The kitty → tmux chain is fixed so the chords arrive. `debug_leak_snapshot` moves off `ctrl+shift+d`.                                                                                                         |
| A8  | "Other pane" means the MRU pane, previewed while armed. Focus after a close goes to the MRU pane. Resize steps the focused pane's parent split, and the outer and inner splits keep separate ratios. New three-pane geometries are refused when they do not fit. While zoomed, split keys only restore. |
| A9  | The sase-1er pager close bug is fixed before or as part of any new pager close path.                                                                                                                                                                                                                    |

### Resolutions made while designing this plan

- **Doubled `ctrl+w` focuses the armed target**, which is the MRU other pane. With two
  panes this equals `ctrl+f`, as today. The research said both "alias of `ctrl+f`" and
  "targets MRU", and those differ with three panes. The armed preview frame makes "where
  `^W` points" visible, so `^W ^W` must go to the pane the frame lights up.
- **Restore-only split keys while zoomed land in `deck-grid-adapter`, unflagged.**
  Today, a split key while zoomed drops the snapshot and splits the zoomed panel, which
  silently discards the hidden panel's session. Today's code also needs an index-0 remap
  hack for this case, and the pane-ID adapter removes it.
- **`debug_leak_snapshot` moves to `f12`**, still gated by `SASE_ACE_DEBUG_LEAKS=1`. No
  function key is bound anywhere in ACE or the pager, and kitty does not map a bare
  `f12`.
- **Measured: grid `fr` tracks round exactly like today's box layout.** A Textual 8.0.1
  prototype compared `Vertical` / `Horizontal` with `{r}fr {100-r}fr` children against a
  1×2 or 2×1 `layout: grid` with the same `fr` tracks. It covered both axes, ratios 30,
  50 and 70, and extents 13–89, and found identical regions in all 462 cases. So single
  and two-pane PNG goldens **must stay byte-identical**. A diff there is a bug, not
  noise.

## UX contract

### Vocabulary

- In code, docs and help, say **stacked** (`\`) and **side by side** (`|`). Keep
  "horizontal" and "vertical" only as aliases, because vim and tmux use them in opposite
  senses.
- **main pane:** the pane that spans the full width or height in a three-pane layout.
- **pair:** the other two panes.
- **reading order:** the outer first region, then the second; within the pair, the first
  member, then the second.
- **turn:** transpose.
- The pager says "pane" and Agents says "panel".

```
 single     R2 stacked    C2 side by side
┌──────┐    ┌──────┐       ┌───┬───┐
│  A   │    │  A   │       │ A │ B │
│      │    ├──────┤       │   │   │
└──────┘    │  B   │       └───┴───┘
            └──────┘
 R3 main-top  R3 main-bottom  C3 main-left   C3 main-right
┌──────┐     ┌───┬───┐       ┌───┬───┐      ┌───┬───┐
│  A   │     │ B │ C │       │   │ B │      │ B │   │
├───┬──┤     ├───┴───┤       │ A ├───┤      ├───┤ A │
│ B │C │     │   A   │       │   │ C │      │ C │   │
└───┴──┘     └───────┘       └───┴───┘      └───┴───┘
```

### The one rule

> `\` draws a stacked divider and `|` a side-by-side divider. **If that kind of divider
> already spans the whole area, the key erases it, and the side you are on grows to fill
> the space. Otherwise, with fewer than three panes, the key draws the divider through
> the focused pane. With three panes, the key turns the layout.**

| From   | `\` (stacked divider)                                                                                                            | `\|` (side-by-side divider)                                                                                                                        |
| ------ | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| single | **R2**: new pane below, focused, 50/50                                                                                           | **C2**: new pane right, focused, 50/50                                                                                                             |
| R2     | **single**, keeping the focused pane                                                                                             | **R3**: the focused pane splits, the new pane opens to its right and takes focus, and the unfocused pane becomes main (it does not move or resize) |
| C2     | **C3**: the focused pane splits, the new pane opens below it and takes focus, and the unfocused pane becomes main                | **single**, keeping the focused pane                                                                                                               |
| R3     | Erase the full-width divider. Focus in the pair → **C2** of the pair, keeping the pair ratio. Focus on main → **single** (main). | **Turn → C3** (main-top ↔ main-left, main-bottom ↔ main-right)                                                                                     |
| C3     | **Turn → R3**                                                                                                                    | Erase the full-height divider. Focus in the pair → **R2** of the pair, keeping the pair ratio. Focus on main → **single** (main).                  |

Examples: R2 with the top pane focused, then `|`, gives main-bottom. R2 with the bottom
pane focused, then `|`, gives main-top.

`\` and `|` never create a fourth pane. `ctrl+w` (pager) and the deck picker capitals
never create a third pane. From a single pane they still open a two-pane split, as
today.

**While `three_pane_splits` is off** (only until `unflag-docs`), the opposite key on a
two-pane split rotates it exactly as today, and no surface ever shows a third pane.

### Focus, swap, close, turn, resize

- **Focus.**
  - `ctrl+f` / `ctrl+b` move to the next / previous pane in reading order, and wrap.
    Both are inert on a single pane; with two panes they coincide.
  - A click focuses the clicked pane by pane ID.
  - Exactly one pane has logical focus, and Textual focus always agrees with it. That
    includes after a close: never leave Textual focus on a hidden widget.
- **Swap.**
  - `ctrl+shift+f` / `ctrl+shift+b` (aliases `>` / `<`) exchange the focused pane's
    whole session with the next / previous pane in reading order, and wrap. A session is
    the document or deck, card, scroll anchor, search, folds, history and in-flight
    work.
  - Geometry and both ratios belong to slots and do not change. Focus follows the
    content.
  - From any pair pane, one press in the right direction puts that pane in the main
    slot. With two panes both directions coincide. Inert on a single pane.
- **Close.**
  - `ctrl+shift+d` (alias `ctrl+x`) closes the focused pane, and so do the pager's
    existing `q` / `Esc` / exhausted `Backspace` when split. Active only with two or
    more panes. `ctrl+shift+d` never exits a single-pane pager; `q` still does.
  - A closed pair member's sibling takes over its space. The result is a two-pane split
    on the outer axis, with the outer ratio kept.
  - A closed main pane leaves the pair to fill the area. The result is a two-pane split
    on the inner axis, with the pair ratio kept.
  - In a two-pane split, close leaves a single pane.
  - Focus goes to the most recently focused survivor, falling back to the first pane in
    reading order.
  - Close is the exact inverse of split: positions, ratios, focus and MRU order all
    return.
- **Turn.** `ctrl+t` transposes any split: R2 ↔ C2 and the R3 ↔ C3 mapping above. Inert
  on a single pane.
- **Resize.**
  - Agents `}` / `{` and pager `+` / `-` step the split that directly separates the
    focused pane from its sibling: the inner split for a pair pane, the outer split for
    the main pane or for either pane of a two-pane split.
  - Grow means the focused pane gets bigger. Steps stay at 30/50/70 and clamp silently.
  - A new inner split starts at 50.
- **"Other pane".** Pager `ctrl+w <label>`, doubled `ctrl+w`, and the deck picker
  capitals (`p` then `M` / `F` / `T` / `N`) target the **most recently focused other
  pane**, falling back to the next pane in reading order.
  - The target's pane ID is captured when the action is armed.
  - If a structural change removes the target before an async result lands, the action
    is canceled with a short message. The result is never redirected.
  - History is pushed only in the destination. Keep the sase-1eg fix that leaves the
    source pane's history generation untouched.
- **New-pane content.**
  - Pager: a clone of the focused pane, with independent history caches (unchanged).
  - Agents: `choose_new_panel()` over every visible deck. With Main and Files shown, the
    third panel opens on Tools, or on FINAL when FINAL has positive content. The
    existing duplicate fallback stays.

### Fit

- Every **new three-pane geometry** (nest or turn) must give each pane the surface
  minimum. The pager minimum is the existing 7 rows × 32 columns. Agents gets new
  constants of about 8 rows × 40 columns, tuned on real screens.
- Otherwise the key is refused with a toast that names what would fit. Examples: "Not
  enough room for a third panel — ctrl+s collapses the node list", "Not enough room to
  turn the panels".
- A key keeps one meaning per state: it never falls back to a different action when
  space is short.
- Never collapse the node list automatically. The Agents two-panel path stays unguarded,
  and the pager keeps its existing two-pane guard.
- Three-pane resize steps that would break a minimum clamp silently, as pager two-pane
  resizing does today.
- Shrinking the terminal never closes a pane.

### Zoom (Agents)

- `Z` snapshots the whole state.
- The chip names the position, for example `◲ 3 of 3 · Z restore`.
- Focus, swap, close, resize and turn are disabled while zoomed.
- A split key while zoomed **only restores** the layout, keeping deck and card edits
  made while zoomed (`exit_zoom_keeping_panels`).
- Persistence saves the restored geometry with the edits made while zoomed, not a stale
  snapshot.

### Keys

| Action                       | Agents default (in `default_config.yml`, configurable)           | Pager (hard-coded)                                      | Active   |
| ---------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------- | -------- |
| Split stacked / side by side | `backslash` / `vertical_line` (existing)                         | same (existing)                                         | always   |
| Focus next / previous        | `ctrl+f` (existing) / `ctrl+b`                                   | `ctrl+f` (existing) / `ctrl+b`                          | 2+ panes |
| Swap with next / previous    | `ctrl+shift+f,greater_than_sign` / `ctrl+shift+b,less_than_sign` | same                                                    | 2+ panes |
| Close focused                | `ctrl+shift+d,ctrl+x`                                            | same, plus existing `q` / `Esc` / exhausted `Backspace` | 2+ panes |
| Turn                         | `ctrl+t`                                                         | `ctrl+t`                                                | 2+ panes |
| Grow / shrink                | `}` / `{` (existing)                                             | `+` / `-` (existing)                                    | 2+ panes |

A zoomed deck counts as one visible pane, so every "2+ panes" key is off while zoomed.

Agents action ids:

- Keep `toggle_deck_focus` as the id for user-config compatibility, relabeled "Focus
  Next Deck Panel".
- Add `toggle_deck_focus_reverse`, `swap_deck_panel_next`, `swap_deck_panel_prev`,
  `close_deck_panel` and `turn_deck_layout`.

### Beauty spec

1. **Spatial stability is the aesthetic.** A split subdivides only the focused pane. A
   turn pins the top-left pane. A close puts back exactly what was there. Swap, turn,
   resize, focus and close never remount a surviving widget, so text never jumps.
2. **Exactly one full-strength frame.**
   - Each surface keeps its frame language: the pager's round section-accent frames and
     the deck's solid deck-accent frames, which drop to 35% when unfocused.
   - Do not frame the pair as a group, and draw no custom T-junction box art in v1. The
     doubled edge where frames meet already exists in two-pane layouts.
   - Only the focused pager pane paints link and time-band labels, as today.
3. **Name positions only when naming a target.**
   - Extend the deck's zoom glyphs `◧ ◨ ⬒ ⬓` with `◰ ◱ ◲ ◳`; all eight are one cell
     wide.
   - Main-top reads `⬒ ◱ ◲`, main-bottom `◰ ◳ ⬓`, main-left `◧ ◳ ◲`, and main-right
     `◰ ◱ ◨`.
   - Position words are `top`, `bottom`, `left`, `right`, `top-left`, `top-right`,
     `bottom-left` and `bottom-right`.
   - Use glyphs in the zoom chip, the picker hint ("show in the ◲ bottom-right panel")
     and the armed `^W` footer. Steady-state titles stay quiet.
   - `◰`–`◳` draw outlines rather than fills. Judge their weight in the live look pass,
     and record any substitution in the phase notes.
4. **Preview the target while `ctrl+w` is armed** (pager). The target pane's frame lifts
   to about 70% strength until the label lands or the arm is canceled.
5. **The footer does not grow.**
   - Two-pane footers stay byte-identical.
   - With three panes, `^F/^B pane` (pager) and `^F/^B panel` (Agents) replace the
     two-pane focus entry.
   - Swap, close and turn live in `?` help and the palette.
6. **Narrow pair panes truncate gracefully.** The deck or document name is always kept,
   and optional title detail drops first. Each pane budgets its chrome from its own
   width and height.
7. **No animation.** Toasts appear only on refusal, never on success; the layout change
   is its own feedback.

## Architecture

### `PaneGrid`: one shared, pure, presentation-only model

The model lives in `src/sase/ace/tui/util/pane_grid.py`.

- It is stdlib only: no Textual, Rich or ACE widget/action imports, and no deck or pager
  concepts.
- The pager already imports `sase.ace.tui.util` (`pump_tasks`, `trace`), so this adds no
  new dependency direction. Importing a module there costs about 0.1 s and loads no
  `textual`.
- This is presentation layout, which no other frontend has to match, so it stays out of
  `sase_core` under the Rust-core boundary litmus.
- If the module would pass the `toobig` 700-line tier, split the geometry helpers into a
  sibling module (`pane_grid_geometry.py`).

Target shape (names may be refined, but not the semantics):

```python
RATIO_STEPS = (30, 50, 70)
MAX_PANES = 3

class Axis(StrEnum):
    ROWS = "rows"   # stacked, drawn by `\`
    COLS = "cols"   # side by side, drawn by `|`

@dataclass(frozen=True, slots=True)
class Pair:
    region: int       # outer region holding the pair: 0 = top/left, 1 = bottom/right
    ratio: int = 50   # first pair member's share

@dataclass(frozen=True, slots=True)
class PaneGrid:
    panes: tuple[int, ...] = (0,)   # visible stable pane IDs in reading order, 1..3, unique
    focused: int = 0                # a member of panes
    axis: Axis | None = None        # outer split axis; None iff one pane
    ratio: int = 50                 # outer first region's share
    pair: Pair | None = None        # set iff three panes
    recent: tuple[int, ...] = (0,)  # MRU permutation of panes, focused first

class Geometry(StrEnum): SINGLE, R2, C2, R3_MAIN_TOP, R3_MAIN_BOTTOM, C3_MAIN_LEFT, C3_MAIN_RIGHT

def geometry(g) -> Geometry
def main_pane(g) -> int | None
def free_pane_id(g) -> int | None                    # lowest unused ID in range(MAX_PANES)
def press_split(g, axis, new_id, *, nest=True) -> PaneGrid   # the one rule; nest=False is the flag-off rotate
def close_focused(g) -> PaneGrid
def focus_pane(g, pane_id) -> PaneGrid
def cycle_focus(g, step) -> PaneGrid
def swap_focused(g, step) -> PaneGrid
def turn(g) -> PaneGrid
def step_ratio(g, grow: bool) -> PaneGrid
def other_target(g) -> int | None                    # MRU other pane, else next in reading order
def pane_rects(g, width, height) -> dict[int, tuple[int, int, int, int]]  # floor rule of split_fits
def fits(g, width, height, *, min_width, min_height) -> bool
def position_name(g, pane_id) -> str
def position_glyph(g, pane_id) -> str
def grid_spec(g) -> GridSpec   # column/row fr weights, per-pane cell + spans, DOM order
```

Rules:

- **Every transition is total.** Invalid input, or a key that is inert in the current
  state, returns an equal grid. Nothing raises.
- **`press_split` implements the transition table exactly.** With `nest=False`, a
  two-pane grid plus the other axis key turns instead of nesting, and no result ever has
  three panes. That parameter is the flag-off branch; `unflag-docs` deletes it.
- **MRU bookkeeping:**
  - Focusing pane X moves X to the front of `recent`.
  - Closing a pane removes it from `recent`, and the new focus is the head of what
    remains.
  - Swap and turn leave `recent` unchanged, because focus follows content.
- **`grid_spec` maps directly onto a Textual grid:**
  - R2 is 1×2 and C2 is 2×1.
  - A T shape is 2×2. The main pane spans two cells: column-span in R3, row-span in C3.
  - In R3 the outer ratio sets the row tracks and the pair ratio sets the column tracks.
    In C3 it is the reverse.
  - `dom_order` is the row-major order of each pane's top-left cell. It is **not**
    reading order: main-right is `B A C`.

### Rendering: one flat CSS grid, no reparenting

Each surface uses one container with `layout: grid`. Every pane widget is a direct
child.

- `grid_size_columns` / `grid_size_rows`, the `fr` track lists and per-child
  `column_span` / `row_span` come from `grid_spec`.
- Hidden panes are `display: none`.
- Order is fixed with `move_child`, called only when `dom_order` changes.
- Nested `Vertical` / `Horizontal` containers are rejected. R2 → R3 would require
  `remove()` + `mount()`, which cancels a `PagerView`'s pump-free tasks and recomposes
  every deck for a `DeckPanel`.

The research prototype on Textual 8.0.1 showed exact regions for every T shape and zero
mount/unmount events across six transitions. Tests on both surfaces must assert widget
identity across every structural key. Surfaces hold explicit pane-ID → widget maps and
never resolve panes by DOM query order or by querying repeated IDs from the host.

### Agents deck adapter

- `DeckAreaState` holds:
  - `grid: PaneGrid`;
  - `panels: Mapping[int, DeckPanelState]`, keyed by pane ID;
  - `nodes_collapsed` and `zoom_snapshot`.
- `layout` (`DeckLayout.SINGLE` / `TOP_BOTTOM` / `LEFT_RIGHT`, naming the outer axis),
  `focused` (a pane ID) and `ratio` (outer) remain as derived read-only properties.
  Existing comparisons across ACE keep working, while two-pane-only logic is rewritten.
- `DeckPanel` widgets keep fixed pane IDs: `panel_index` equals the pane ID equals the
  DOM suffix `agent-deck-panel-<id>`, which inner scroll IDs already embed.
- `DeckArea.panel(pane_id)` resolves through an explicit map.
- Swap permutes `grid.panes`; the widgets move cells and keep their content.
- `panels[pane_id]` is the session state for that widget.

### Pager adapter

- `PagerScreen` keeps a `PaneGrid` plus a `dict[int, PagerView]` keyed by pane ID.
- `#pager-panes` becomes the grid container.
- `sase/pager/split.py` keeps only pager policy: the existing minimums `7` / `32` and
  pager fit wrappers. Its duplicated algebra is deleted, not extended, so no third copy
  of the algebra ever exists.
- Structural changes are transactions: validate fit, mount, revalidate after every
  `await`, then publish state. On failure, remove the orphan.

### Persistence (Agents, additive schema v1)

- Keep `schema_version: 1` and raise `MAX_PANELS` to 3.
- `panels` are written in **reading order**. `focused` stays a reading-order index, as
  before.
- A three-panel state also writes an optional
  `"pair": {"region": 0|1, "ratio": 30|50|70}`. `layout` keeps its existing values:
  `top-bottom` / `left-right` name the outer axis.
- On load, pane IDs `0..n-1` are assigned in reading order.
- Validate uniqueness, focus range and ratios. Recover to a smaller valid layout with a
  warning, and never crash.
- An older reader ignores `pair` and truncates to two panels, which is still a valid
  R2/C2. A file without `pair` loads exactly as today.
- While `three_pane_splits` is off, the loader truncates exactly as an old reader would.

### Feature flag

- `three_pane_splits` (`beta`, default off) is created by `deck-three-panels` with
  `sase flag new` and removed by `unflag-docs`.
- Read it only at use sites (split key handling and the persistence load), never at
  module import time. This keeps the pager's cold-path import-weight tests from
  sase-1es.3 green.
- No other flag is added.

## Sequencing with sase-1es and sase-1er

As of master `6cca547014`, `sase-1es` is in progress. Its phase `.6` will replace the
pager body with a Line-API `ScrollView` and rewrite `_screen_split.py`, `view.py`
(`set_pane_role`, split seed) and about 12 `_screen_*` call sites. It forbids pager
golden and split-behavior changes while in flight. Nothing in sase-1es touches
`ace/tui/widgets/decks/`, `bindings.py`, keymaps, `default_config.yml` or `styles.tcss`.

- **Phases that may start immediately:** `terminal-chain`, `pane-grid-model`, and the
  three deck phases. None of them may edit `src/sase/pager/` or `tests/pager/`.
- **External gate for `pager-grid-adapter`, and therefore `pager-three-panes`:** before
  editing anything, confirm that `sase-1es.6` is closed and its commit is on
  `origin/master`. Epic `depends_on` cannot name another epic's phase, so the worker
  enforces the gate itself:
  - Check with `sase bead read sase-1es.6 -f json -r "<why>"`.
  - If the phase is not closed, use `/sase_monitor` to wait. One workable poll is:
    `until sase bead list -t phase -s closed -n 0 -f json | python3 -c 'import json,sys; sys.exit(0 if any(b["id"] == "sase-1es.6" for b in json.load(sys.stdin)["results"]) else 1)'; do sleep 600; done`
  - Resume in the follow-up turn. Never start pager edits early, and never end the turn
    promising to resume.
  - After the gate, write against the new body API (`_invalidate_body_paint` /
    `_invalidate_body_layout`), not the deleted `_ensure_body` idiom.
  - If `sase-1es.7` or `.8` are still open, stay out of `VimSearchController` and the
    search row source, and rebase `docs/pager.md` edits onto theirs.
- **sase-1er** (pager close and unsplit detach the surviving view) is a ready standalone
  task bead. The user should launch it now, from its TaskTriage gate, so it lands before
  `sase-1es.6` starts.
  - If it is closed by the time `pager-grid-adapter` runs, keep its regression tests
    green.
  - If it is still open, this phase's close-path rewrite is the fix. Add the Pilot
    assertions the bead names, and record
    `PROPOSED FOLLOW-UP: close sase-1er as fixed by pager-grid-adapter` on this phase
    bead.

## Guardrails for every phase

- **Boundary.**
  - The model and adapters are presentation code and stay in Python. Nothing moves to
    `sase-core`.
  - No pager file changes before the gate above.
  - Do not edit files owned by in-flight sase-1es phases.
- **Model API stability.**
  - After `pane-grid-model` lands, later phases may add functions to `pane_grid.py` but
    may not change existing semantics without updating the golden transition table in
    the same change and saying why in their notes.
  - Deck and pager phases can run concurrently, so rebase onto the latest master before
    touching the model.
- **No remounts.** Layout changes never remove or mount a surviving pane widget. Every
  structural key's Pilot test asserts widget identity, and asserts scroll position where
  relevant.
- **Goldens.**
  - Single and two-pane PNG goldens on both surfaces stay byte-identical; grid rounding
    was measured identical, so a diff is a bug.
  - New goldens are added only where a phase says so.
  - Run the targeted visual suites with `just fix-tui-screenshots -- <selectors>`,
    through `/sase_monitor` when long. Inspect the report and every golden change before
    finishing.
- **Event-loop safety.** Follow the `tui_perf.md` reference memory:
  - No I/O in key handlers; persistence keeps the existing coalesced off-thread save.
  - Keep the TUI perf rule that only visible panels receive document updates.
  - Keep Textual pump callbacks thin.
- **Default keymap config.** Every Agents keymap change updates
  `src/sase/default_config.yml`, `keymaps/app_keymaps.py`, keymap metadata, the
  command-palette rows and the help modal together.
- **File size.** Stay under the `toobig` 700-line tier, splitting new logic into sibling
  modules or mixins. `_agent_detail_deck_layout.py` is already 580 lines.
- **Symvision.**
  - `pane-grid-model` adds Justfile `--epic-symbol <this epic's bead id>(<symbol>)`
    entries for public symbols that only later phases consume.
  - Each consuming phase removes the entries it satisfies, and `unflag-docs` verifies
    none remain.
  - Read `symvision.md` before fixing any Symvision failure.
- **Verification.**
  - Read the `lint_and_test.md`, `symvision.md`, `tui.md`, `tui_perf.md` and
    `tui_screenshot.md` reference memories before finishing.
  - Run `sase tool run check`.
  - Do not run `just check-full` unless the phase bead says so.
- **Follow-ups.** Phase workers record discovered work as `PROPOSED FOLLOW-UP:` notes on
  their own bead and create no beads. The one exception is the flag bead that
  `sase flag new` creates in `deck-three-panels`.

## Phase terminal-chain: Deliver the ctrl+shift chords through kitty and tmux

On this host, kitty 0.41.1 keeps `kitty_mod = ctrl+shift`, and its defaults send
`kitty_mod+f` / `kitty_mod+b` to `move_window_forward` / `move_window_backward`. tmux
3.5a has `extended-keys off`. Textual parses CSI-u (`ESC[100;6u` → `ctrl+shift+d`) but
not tmux's xterm form. As a result, `ctrl+shift+d` arrives as `ctrl+d` (half-page
scroll) and `ctrl+shift+f/b` never arrive at all.

1. Open the linked repo with `sase repo open chezmoi -r "<why>"`, use only the printed
   path, and read its `AGENTS.md`.
2. In `home/dot_config/kitty/kitty.conf`, unmap `kitty_mod+f`, `kitty_mod+b` and
   `kitty_mod+o` so they pass through to the program. `kitty_mod+o` is
   `pass_selection_to_program` today, which also kills ACE's `ctrl+shift+o` forward
   jump.
   - Use the pass-through form kitty documents: a `map` with no action. Confirm it
     against the installed definitions (`/usr/lib/kitty/kitty/options/definition.py` and
     the kitty docs).
   - Do not use `no_op`, which swallows the key.
3. In `home/dot_config/tmux/tmux.conf`, next to the existing `terminal-features` lines,
   add:
   - `set -s extended-keys on`
   - `set -s extended-keys-format csi-u`
   - `set -as terminal-features 'xterm-kitty:extkeys'`

   Prefer `on`. Use `always` only if a Textual app inside tmux never receives CSI-u
   under `on`, and record why.

4. Verify what can be automated without touching the live session:
   - Start a private tmux server (`tmux -L <tmpname> -f <conf> ...`) and confirm the
     options parse and read back.
   - Inside that server, run a ten-line Textual app that prints `event.key`. Where this
     tmux version can inject a modified chord with `send-keys`, use it to confirm the
     tmux → Textual leg. Kill only the private server.
5. Do not run `chezmoi apply`; the user applies it. Record a user checklist in this
   phase bead's notes:
   - `chezmoi apply` (its run-onchange scripts reload kitty and tmux).
   - Inside tmux inside kitty, run a key logger (`textual keys` from `textual-dev`, or
     the ten-line app). Press `ctrl+shift+f`, `ctrl+shift+b`, `ctrl+shift+d` and
     `ctrl+shift+o`, and expect those exact names.
   - Repeat over SSH from the Mac, where nested layers need the same settings.

**Acceptance:** the chezmoi change is committed in the linked repo, the automated checks
pass, and the manual checklist is recorded. Nothing in the sase repo changes.

## Phase pane-grid-model: Shared pure PaneGrid model and golden transition table

1. Implement `src/sase/ace/tui/util/pane_grid.py` per [Architecture](#architecture) and
   the full [UX contract](#ux-contract), including:
   - every three-pane rule;
   - the `nest` policy argument;
   - MRU bookkeeping;
   - `pane_rects` / `fits` using the same floor arithmetic as today's pager
     `split_fits`;
   - `position_name` / `position_glyph` with the glyph table from the
     [beauty spec](#beauty-spec);
   - `grid_spec`.
2. **Golden transition table** (`tests/ace/tui/util/`): render every (state, key) pair
   into a checked-in plain-text table that reviewers can read as the spec.
   - States are the seven geometries × each focus position: 17 states.
   - Keys are `\`, `|`, focus ±, swap ±, close, turn, grow and shrink: 10 keys.
   - That is about 170 rows. Each row uses a compact notation: geometry, panes in
     reading order with the focused one marked, outer and pair ratios, and MRU order.
   - Include representative non-50 ratios.
   - Regeneration is an explicit, opt-in test flag; the default test run compares.
   - Add a second short table for `nest=False`.
3. **Hypothesis invariants:**
   - one to three unique panes;
   - `focused ∈ panes`, and `recent` is a permutation with `focused` first;
   - `axis is None` exactly when single, and `pair` is set exactly when there are three
     panes;
   - ratios stay in `RATIO_STEPS`;
   - `turn∘turn = id`;
   - `swap(+1)∘swap(−1) = id`;
   - `close∘split = id` from every one- and two-pane grid, with ratios, focus and MRU
     included;
   - `n` focus steps visit every pane and return to the start;
   - `grid_spec` cells tile the track grid with no gap or overlap;
   - `pane_rects` tile any `width × height` exactly;
   - no transition ever raises or yields four panes;
   - `nest=False` never yields three panes.
4. **Import-weight test:** in a subprocess, `import sase.ace.tui.util.pane_grid` must
   not load `textual`, `rich`, `sase.ace.tui.widgets` or `sase.ace.tui.actions`. Follow
   the pattern in `tests/memory/test_history_import_cost.py`.
5. Add Symvision `--epic-symbol` entries for every public symbol that has no non-test
   consumer yet.

**Acceptance:** the table and invariants pass under `sase tool run check`, and the
module has no non-stdlib imports.

## Phase deck-grid-adapter: Agents deck on PaneGrid with flat grid rendering

This phase still shows at most two panels and adds no keys. Every existing key keeps its
meaning, except the A3 survivor and the restore-only zoom split keys.

1. **State.**
   - Rebuild `DeckAreaState` (`decks/model.py`) on `PaneGrid`, as in
     [Agents deck adapter](#agents-deck-adapter).
   - Rewrite `decks/layout.py` transitions (`toggle_split`, `toggle_focus`,
     `step_ratio`, zoom helpers) as thin wrappers over the model. Pass `nest=False` for
     now; `deck-three-panels` switches it behind the flag.
   - Same-key unsplit keeps the **focused** panel (A3).
   - Rewrite the ~49 `DeckAreaState(...)` constructions in `src/` and `tests/`. Use a
     small test helper for legacy-shaped fixtures if it cuts churn, but keep each test's
     intent.
2. **Widgets.**
   - `DeckArea` (`decks/area.py`) becomes a `layout: grid` container driven by
     `grid_spec`, and `panel(pane_id)` resolves through an explicit map.
   - Delete the ID-keyed ratio rules and the `-left-right` horizontal rule from
     `styles.tcss` (the `#agent-deck-area.-top-bottom.-ratio-*` /
     `.-left-right.-ratio-*` block). Keep geometry classes on the area only where other
     CSS or tests still read them.
   - `on_deck_panel_focus_requested` focuses the clicked pane by ID instead of calling
     `toggle_focus`.
3. **Generalize every two-pane site:**
   - `visible_panels` (reading order);
   - `focused_panel`;
   - `_sync_panel_views` and `set_preferred_cards` (keyed by pane ID);
   - `picker.other_panel_target` (MRU via `other_target`) and `panel_position_label`
     (`position_name`);
   - `titles._ZOOM_HALF_GLYPHS` → `position_glyph`, with the zoom index and count in
     reading order;
   - `_open_deck_split`, which takes the new ID from `free_pane_id` and shows the deck
     on that pane;
   - `show_deck_in_other_panel`;
   - `toggle_deck_split`, which currently passes `state.panels[-1]`;
   - the `_agent_detail_deck_*` and `_panel_detail.py` call sites that index panels;
   - `panel_transitions.py` (`state.panels[self._panel_index]`).

   No `1 - x`, `panel(0)` or `panel(1)` assumptions may remain in deck code.

4. **Zoom.**
   - The zoomed state shows only the focused pane; hidden panels keep their widgets and
     sessions.
   - A split key while zoomed only restores (`exit_zoom_keeping_panels`). This deletes
     the "zoom on panel 1 → remap widget 0" hack in `_open_deck_split`.
   - The node-rail `ctrl+s` and `Z` keep their restore semantics.
5. **Persistence** (`models/agent_deck_persistence.py`, format unchanged in this phase):
   - Write panels in reading order and load them into pane IDs `0..n-1`.
   - `snapshot_from_area_state` persists the restored geometry plus the edits made while
     zoomed.
6. **Tests:**
   - Update the deck layout, model, picker, zoom-chrome, persistence, split-key and
     split-pilot tests. Flip only the assertions that encoded the panel-0 survivor.
   - Add Pilot assertions that the same `DeckPanel` objects persist, with no
     mount/unmount, across split, unsplit, rotate, focus, resize and zoom, and that the
     scroll offset is kept.
   - Remove the `--epic-symbol` entries this phase satisfies.
7. **Goldens.** Run the Agents deck visual selectors (`agents_deck*`, `agents_decks_*`,
   `agents_final_reply_split`). They must be byte-identical.

**Acceptance:** all existing deck behavior except A3 and zoom-restore is unchanged,
goldens are identical, and `sase tool run check` passes.

## Phase deck-pane-keys: Agents deck reverse focus, swap, close, and turn keys

1. **Actions** (`actions/agents/_deck_layout_actions.py` plus `AgentDetail` mixin
   methods; put new code in a sibling mixin if `_agent_detail_deck_layout.py` would pass
   700 lines):
   - `toggle_deck_focus` cycles forward and `toggle_deck_focus_reverse` cycles back.
   - `swap_deck_panel_next` / `swap_deck_panel_prev` permute `grid.panes`, with no
     re-show and no remount, and focus follows the content.
   - `close_deck_panel` hides the closed panel's widget, drops its session, and moves
     Textual focus to the new focused panel's visible scroll.
   - `turn_deck_layout` turns the layout.
   - Every action notifies the coalesced deck-state save and refreshes the footer.
2. **Keymaps and plumbing.** Use the defaults from the [keys table](#keys). Wire them
   through:
   - `default_config.yml`, `keymaps/app_keymaps.py` fields and `keymaps/metadata.py`
     labels;
   - `bindings.py`;
   - `_app_action_availability.py`: add every new action to `_DECK_LAYOUT_ACTIONS` and
     `_DECK_SPLIT_ONLY_ACTIONS`. Zoomed state is single, so these keys are already off
     while zoomed. Keep it that way.
   - `commands/_app_metadata_nav.py` palette rows and
     `commands/_availability_agents.py`;
   - the help modal (`modals/help_modal/agents_bindings.py`), whose deck rows lead with
     the one-sentence grammar;
   - the `_deck_search_host.py` key list.

   Add these `_CONTEXTUAL_APP_DUPLICATES` pairs in `keymaps/registry.py`:
   - `{toggle_deck_focus_reverse, scroll_prompt_up}`
   - `{swap_deck_panel_prev, start_ancestor_mode}`
   - `{swap_deck_panel_next, start_child_mode}`
   - `{turn_deck_layout, beads_toggle_note_audience}`

   Verify that each partner's availability really is tab-disjoint. The two-panel footer
   stays byte-identical.

3. **Debug chord.** Move `debug_leak_snapshot` from `ctrl+shift+d` to `f12` in
   `bindings.py`, still gated by `SASE_ACE_DEBUG_LEAKS=1`. Update the
   `actions/_debug_leaks.py` docstring.
4. **Docs** (`docs/ace.md`, `docs/configuration.md`):
   - Replace "`Ctrl+B` has no Agents behavior" and "the same split key again closes the
     second panel".
   - Document the new keys, the focused-panel survivor, and zoom restore.
   - Add a terminal note modeled on the existing `Ctrl+Shift+J/K` note: the chords
     arrive only through a CSI-u-capable chain, and `>` / `<` / `ctrl+x` always work.
5. **Tests:**
   - Unit tests for each action and availability rule.
   - Registry duplicate validation.
   - Pilot tests for each key with two panels, including swap identity and scroll, click
     focus after a swap, close of either panel, and `ctrl+b` / `ctrl+f` parity.
   - Pilot presses of `ctrl+shift+f/b/d` must reach the actions.
   - The Artifacts tab keeps `<` / `>` and the Beads pane keeps `ctrl+t`.

**Acceptance:** goldens are unchanged, `sase tool run check` passes, and every new key
works on a two-panel deck.

## Phase deck-three-panels: Agents deck three panels behind the three_pane_splits beta flag

1. **Flag.** Read `sase_flags.md`, then run `sase flag new three_pane_splits -k beta`
   with:
   - when enabled: "From a two-pane split, the other split key splits the focused pane
     into a three-pane T layout on the Agents deck and in the pager; with three panes
     the split keys erase or turn the layout."
   - when disabled: "The other split key rotates a two-pane split and no surface ever
     shows a third pane."
   - remove when: "Both the Agents and pager three-pane phases of this epic have landed
     with green transition tables, goldens and live look passes; the epic's unflag-docs
     phase deletes the Off branch."

   Paste the printed registry entry and follow its both-states test checklist.

2. **Third panel.**
   - Add `DeckPanel(2)` (`agent-deck-panel-2`), composed hidden or mounted lazily on
     first use. Decide by measuring startup and first split under `SASE_TUI_TRACE=1`,
     and record the numbers.
   - With the flag on, split keys call `press_split(..., nest=True)`.
3. **Fit guard.**
   - Add Agents constants `MIN_DECK_PANEL_HEIGHT` / `MIN_DECK_PANEL_WIDTH`, starting
     near 8 × 40, and tune them in the live look pass.
   - Before committing a nest or three-panel turn, check `fits` against the deck area's
     size. On failure, toast what would fit, never acting differently.
   - When collapsing the node list would make it fit, say so: "Not enough room for a
     third panel — ctrl+s collapses the node list". Compute this from the rail width.
   - Three-panel resize steps clamp silently.
4. **Targets and chrome.**
   - Picker capitals and their hint use MRU with a position glyph: "show in the ◲
     bottom-right panel".
   - Picker row labels use position names.
   - The zoom chip shows `◲ 3 of 3 · Z restore`.
   - With three panels, the footer shows `^F/^B panel` in place of the two-panel focus
     entry. Thread a panel count, or a "three panels" bool, through
     `_display_detail_footer.py` / `_keybinding_modes.py` alongside `deck_split`.
   - Truncation tiers in `titles.py` must keep the deck glyph and name in narrow pair
     panels.
5. **Persistence.**
   - Implement the [additive v1](#persistence-agents-additive-schema-v1) format with
     `MAX_PANELS = 3` and `pair`.
   - With the flag off, load truncates to two panels.
   - Tests: v1 files without `pair` load unchanged; a three-panel state round-trips; an
     old-reader simulation (two-panel truncation, `pair` ignored) yields a valid R2/C2;
     malformed `pair` values recover with a warning.
6. **Performance.**
   - Only visible panels receive document updates, which now means up to three.
   - Benchmark `j`/`k` key-to-paint p95 with three visible panels against two, using the
     existing deck and `j`/`k` benches (`tests/ace/tui/bench_tui_jk.py`,
     `tests/ace/tui/bench_tui_deck_view.py`) and `SASE_TUI_PERF=1`. The target is p95 <
     16 ms.
   - Record both numbers. If the target is missed, fix it within the existing debounce
     and coalescing patterns, or record a `PROPOSED FOLLOW-UP`.
7. **Tests** (flag on and flag off):
   - Nest from each two-panel focus position into all four T shapes.
   - Erase with main focused and with a pair panel focused.
   - Turn both ways, and close each of the three panels.
   - Swap preserves widget identity and scroll; click focus works after a swap.
   - A split key while zoomed restores.
   - Rapid repeated keys.
   - The fit refusal toast fires and the state is unchanged.
   - Flag off: the opposite key still rotates and no third panel is ever shown.
   - `choose_new_panel` picks Tools for Main + Files.
8. **Goldens** (flag on, new): the four T shapes with a pair panel focused, a
   three-panel zoom chip, and a three-panel picker hint.
   - Use 120x40 with the node rail collapsed. If the tuned minimums do not fit there,
     use 160x40, and say which in the notes.
   - Existing goldens stay identical.
9. **Live look pass.**
   - Run `sase screenshot` with real content (long replies, diffs, FINAL, streaming) at
     80x24, 120x40 and 205x65.
   - Judge frame strength, the `◰`–`◳` glyph weight and truncation.
   - Attach the PNGs to this phase bead's notes with findings.

**Acceptance:** with the flag off, behavior and goldens match `deck-pane-keys`. With the
flag on, the full three-panel contract works on Agents. `sase tool run check` passes.

## Phase pager-grid-adapter: Pager on PaneGrid with grid panes and the new pane keys

**Gate first:** satisfy the [external gate](#sequencing-with-sase-1es-and-sase-1er) on
`sase-1es.6`, and handle `sase-1er` as described there. This phase still shows at most
two panes.

1. **Port.**
   - Replace `PagerSplitState` / `_views` / `_focused_index` with a `PaneGrid` plus a
     `dict[int, PagerView]`.
   - Keep `PagerSplitLayout` only as a derived property if remaining callers need it.
   - Strip `sase/pager/split.py` to pager policy (minimums, fit wrappers) and port
     `tests/pager/test_split_model.py` to the shared model's semantics.
   - Use `nest=False` until `pager-three-panes`.
2. **Grid.**
   - `#pager-panes` becomes the `layout: grid` container, and `_apply_split_state`
     applies `grid_spec`.
   - Keep `set_pane_role` per pane, with the focused frame and label scoping exactly as
     today.
   - Use the post-sase-1es.6 body invalidation API.
3. **Close paths.**
   - `q`, `Esc`, exhausted `Backspace`, same-key unsplit, `ctrl+shift+d` and `ctrl+x`
     remove **only** the discarded view.
   - The survivor stays a child of `#pager-panes`, keeps its workers, reading anchor and
     labels, and renders and scrolls.
   - Add Pilot assertions for each path. Today's `test_q_closes_focused_pane_then_pager`
     only checks counts.
4. **Keys.**
   - Add `ctrl+b` (focus previous), `ctrl+shift+f` / `greater_than_sign` (swap next),
     `ctrl+shift+b` / `less_than_sign` (swap previous), `ctrl+shift+d` / `ctrl+x` (close
     focused; inert and never exiting when single) and `ctrl+t` (turn) to
     `PagerScreen.BINDINGS`.
   - Add them to the `_screen_host.on_key` passthrough list, so they win over an armed
     label prefix the way `\`, `|`, `ctrl+f`, `+` and `-` do.
   - Click focus works by pane ID.
   - Label arms, goto and search cancel per pane ID, never by index.
   - `ctrl+w` targets come from `other_target` (with two panes, the other pane). The
     target pane ID is captured at arm time, and the action cancels with a short message
     if that pane vanished.
5. **Help and docs.**
   - Pager help rows (`_trail_chrome_help.py`) for the new keys, led by the one-sentence
     grammar.
   - `docs/pager.md` "Split panes" and "Keys".
   - The two-pane footer stays byte-identical.
6. **Tests:**
   - Swap and turn keep widget identity and scroll.
   - Pilot tests for each new key.
   - Extend `tests/ace/tui/actions/test_view_files_pager_split_keys.py`: the ACE modal
     pager never splits the Agents deck, including with the new keys.
   - Run `tests/pager/visual` in check mode: zero golden changes.
   - Rerun the sase-1es pager bench split/open cases if present, and record that nothing
     regressed.

**Acceptance:** all pager goldens are unchanged, every close path keeps a live survivor,
and `sase tool run check` passes.

## Phase pager-three-panes: Pager three panes with MRU ctrl+w and a target preview

1. Behind `three_pane_splits`, split keys use `press_split(..., nest=True)`.
2. **Third view.**
   - Mount exactly as the second view: `split_seed()` clone, the same `PagerView`
     construction, and the `_split_in_flight` guard.
   - Structural changes are transactions: validate fit, mount, revalidate after the
     await, then publish state. On failure, remove the orphan.
3. **Fit.**
   - New three-pane geometries need 7 rows × 32 columns per pane.
   - Refusal toasts say why ("Not enough room for a third pane") and name an alternative
     only when one would actually fit.
   - Three-pane resize clamps silently.
4. **`ctrl+w`.**
   - Arms target the MRU other pane, and that pane's frame lifts to about 70% strength
     while armed (extend `set_pane_role` with a preview state).
   - Doubled `ctrl+w` focuses the armed target.
   - Label landing, `Esc`, and any focus or structure change clear the preview.
   - The armed footer names the target with its position glyph.
   - Source and destination histories stay independent.
5. **Chrome.**
   - With three panes, the footer shows `^F/^B pane` in place of `^F pane`.
   - Help and the `docs/pager.md` "Split panes" section gain the seven-geometry diagram.
6. **Tests** (flag on and flag off):
   - Delete each of the three panes.
   - Erase with main focused and with a pair pane focused.
   - Turn both ways.
   - Rapid repeated keys during an async mount.
   - `ctrl+w` targets MRU, with the preview appearing and clearing.
   - Histories stay independent.
   - Click focus after a swap.
   - Fit refusal.
   - Flag off: rotate is preserved.
7. **Goldens** (flag on, new, 120x40): four T shapes with a pair pane focused, and one
   three-pane armed `ctrl+w` preview. Existing goldens stay unchanged. Add no three-pane
   goldens at 60 columns.
8. **Live look pass:** run real documents (long markdown, diffs, time bands) at 80x24,
   120x40 and 205x65, and attach the PNGs with findings.

**Acceptance:** with the flag on, the full contract works in the pager. With the flag
off, behavior matches `pager-grid-adapter`. `sase tool run check` passes.

## Phase unflag-docs: Remove the flag and finish docs, help, glossary, and release note

1. **Remove `three_pane_splits`.**
   - Delete the Off branch on both surfaces (the `nest=False` rotate path and the
     flag-off persistence truncation), and remove the `nest` parameter from
     `press_split` along with its `nest=False` golden table.
   - Make the On branch unconditional and remove the registry entry.
   - Close the flag bead with
     `sase bead close <flag-bead> --note "<what was verified>"`.
   - Update the both-states tests to the single remaining behavior.
2. Confirm no `--epic-symbol` entries for this epic remain.
3. **Docs and help:**
   - `docs/ace.md` and `docs/pager.md`: the one rule, the seven-geometry diagram, the
     keys table, fit and zoom behavior, and the terminal note.
   - `docs/configuration.md` keymap entries.
   - Both help sheets.
   - `default_config.yml` comments for the new Agents keys, including that `<` / `>`
     share keys with Artifacts relation modes through tab availability.
4. **Glossary** (approved as part of this plan; use `/sase_memory_write`, then
   `sase memory init`): update the `glossary:deck-panel` strand.
   - The Agents tab shows one to three deck panels.
   - State the one split-key rule (erase, draw through focused, turn).
   - Cover `ctrl+f` / `ctrl+b` focus, swap, close, `ctrl+t` turn and `Z` zoom.
   - Drop "the other split key rotates the layout".
5. **Release note.** The phase's commit message must call out, for release-please, that:
   - the other split key now nests a third pane;
   - rotation moved to `ctrl+t`;
   - the deck's same-key unsplit keeps the focused panel;
   - `debug_leak_snapshot` moved to `f12`.
6. Do a final short live look pass on both surfaces, and run the targeted visual suites
   for both surfaces in check mode.

**Acceptance:** no flag remains, docs and help match behavior, and `sase tool run check`
passes.

## Epic acceptance bar

- The golden transition table (17 states × 10 keys) and all Hypothesis invariants pass.
- Pilot tests on both surfaces cover:
  - every close path keeps a mounted, rendering, scrolling survivor;
  - swap and turn keep widget identity and scroll;
  - each of the three panes can be deleted;
  - collapse with main focused;
  - rapid repeated keys during async mounts;
  - click focus after a swap;
  - MRU `ctrl+w` with independent histories;
  - the ACE modal pager never splits the Agents deck.
- Persistence: old files load unchanged, three panels round-trip, and an old-reader
  simulation degrades to a valid two-panel layout.
- Goldens: single and two-pane goldens are byte-identical on both surfaces, and the new
  T-shape goldens exist for both.
- Agents `j`/`k` p95 with three panels is recorded against the 16 ms budget.
- Live look passes are recorded at 80x24, 120x40 and 205x65.

## Non-goals

- More than three panes, nested or recursive splits, tiled 2×2 layouts, or
  three-equal-stripe layouts.
- Synchronized scrolling, drag resizing, persisted pager layouts, or animation.
- Custom T-junction border art, or framing the pair as a group.
- Moving any of this into `sase-core`.
- Changing any sase-1es-owned behavior or its performance targets.
