---
tier: tale
title: Fit the Glossary panel's term rail to its widest row
goal:
  The Glossary panel is wide enough that its left-hand term rail renders every
  term-plus-alias row on a single line for realistic glossaries, and the definition card
  is wider than it is today rather than narrower.
size: medium
proposed_by: bbugyi200.athena.05p
create_time: 2026-08-18 06:58:58
status: wip
---

# Plan

## Problem

The Glossary panel's left pane (the term rail) wraps rows across multiple lines. On the
reporting terminal (182x48) four of the `sase` project's 29 rows wrap:

| Row text                                 | Cells |
| ---------------------------------------- | ----: |
| `Agent Instruction File  agents.md file` |    38 |
| `Agent Hood  hood · agent neighborhood`  |    37 |
| `Required Plugin  required plugin`       |    33 |
| `Xprompt Memory  memory file`            |    27 |

Wrapping costs four extra lines in a list that otherwise fits without scrolling, and it
makes the dim alias summary read as a separate row.

## Root cause (measured, not inferred)

Two fixed sizes in `src/sase/ace/tui/styles.tcss` combine to leave the rail 26 usable
cells:

- `GlossaryPanel #glossary-panel-container` is `max-width: 130`. At 182 columns the
  `90%` width resolves to 163, so the cap binds and the container's content box is
  `130 - 2` (double border) `- 4` (`padding: 1 2`) `= 124` cells.
- `GlossaryPanel #glossary-panel-terms` is a hard `width: 32`. Its non-text chrome is 6
  cells: the `solid` border (2), `OptionList`'s inherited default `padding: 0 1` (2),
  and the vertical scrollbar reserved by `scrollbar-gutter: stable` (2). That leaves
  **26** cells of text.

A Textual harness reproducing exactly these rules at 182x48 confirms the numbers:
container content 124, `#glossary-panel-terms` `scrollable_content_region.width == 26`,
`#glossary-panel-detail` `scrollable_content_region.width == 85`, and 4 of the 29 rows
over 26 cells -- the same four the screenshot shows. The model is validated; the changes
below were sized against it.

## Design

Two changes, both of which make the panel larger:

**1. Raise the container cap `130 -> 150`.** `150` is this file's house maximum for
content modals (`PreviewPanelModal`, `ReportModal`, `WordDefinitionModal`, and
`CommitViewModal` all use `max-width: 150`), so this is an existing house value rather
than a new one. `width: 90%` is unchanged, so the panel still shrinks with the terminal
and only reaches 150 above ~167 columns.

**2. Size the term rail to its widest row instead of pinning it at 32.** A fixed number
cannot be right for every project's glossary: too small wraps, too large steals cells
from the definition card for a project whose terms are all short. This repo already
solves exactly this problem four times over -- the AXE sidebar (`BgCmdList.WidthChanged`
-> `src/sase/ace/tui/actions/_event_widgets.py`), the agent list, the Patch list, and
the Artifacts `#list-container`
(`width: 43; min-width: 43; max-width: 80; /* set programmatically */`) all publish a
content-fit width that the container clamps to a `[MIN, MAX]` range, reserving space for
the pane beside them. Follow that pattern.

Together these give, at 182x48 with the `sase` glossary:

| Measurement           | Today | After |
| --------------------- | ----: | ----: |
| Container content     |   124 |   144 |
| Term rail usable text |    26 |    38 |
| Detail pane usable    |    85 |    93 |
| Wrapped rows (of 29)  |     4 |     0 |

The definition card gets **wider**, not narrower -- the rail's 12 extra cells come out
of the container's 20 new ones.

### Rejected alternative: `width: auto` on the `OptionList`

Pure CSS (`width: auto; min-width: 32; max-width: 52`) looks simpler and needs no
Python, but it is wrong here. In `textual/widget.py::_get_box_model`, the scrollbar
allowance is only added to an auto width when
`styles.overflow_x == "auto" and styles.scrollbar_gutter == "stable"`, or when the
vertical scrollbar is already shown. `OptionList`'s default CSS sets
`overflow-x: hidden`, so the first branch never fires, and the second depends on whether
the list happens to overflow vertically -- which itself depends on whether rows wrapped.
Measured against the fixture: `get_content_width()` returns 38, auto width resolves to
42 outer, and usable text lands at 36 -- two cells short, so the widest row still wraps,
and the shortfall appears and disappears with the list's length. Compute the width
explicitly instead.

### Rejected alternative: ellipsize instead of wrap

`text-overflow: ellipsis` would guarantee one line per row but silently drops alias
text. The ask is that rows _rarely_ wrap, not never; leaving wrap as the fallback for a
pathological term keeps all information visible. No change to wrapping behavior.

## Changes

### 1. `src/sase/ace/tui/styles.tcss`

In `GlossaryPanel #glossary-panel-container`, change `max-width: 130;` to
`max-width: 150;`. Leave `width`, `height`, and `max-height` alone -- see "Height is
deliberately unchanged" below.

In `GlossaryPanel #glossary-panel-terms`, keep `width: 32` as the pre-layout default and
add `min-width` / `max-width` backstops matching the Python constants, in the style of
`#list-container`:

```
GlossaryPanel #glossary-panel-terms {
    width: 32;      /* default; fitted at runtime by term_rail_width() */
    min-width: 32;  /* keep in sync with _TERM_RAIL_MIN_WIDTH */
    max-width: 52;  /* keep in sync with _TERM_RAIL_MAX_WIDTH */
    height: 100%;
    border: solid $secondary;
    scrollbar-gutter: stable;
}
```

### 2. `src/sase/ace/tui/modals/glossary_panel_rendering.py`

Add module-private tunables and one public pure helper. Keep the tunables private
(`_`-prefixed) so tests exercise them through `term_rail_width()` rather than importing
them -- per `sase/memory/symvision.md`, a test-only consumer cannot keep a public symbol
alive.

```python
# The term rail is sized to fit its widest row. Chrome is everything in
# ``#glossary-panel-terms`` that is not text: the 1-cell ``solid`` border on
# each side, ``OptionList``'s default ``padding: 0 1``, and the 2-cell
# vertical scrollbar held open by ``scrollbar-gutter: stable``.
_TERM_RAIL_CHROME = 6
# Never narrower than the historical fixed width, and never wide enough to
# crowd the definition card. Mirrored by the ``min-width`` / ``max-width``
# backstops on ``#glossary-panel-terms`` in ``styles.tcss``.
_TERM_RAIL_MIN_WIDTH = 32
_TERM_RAIL_MAX_WIDTH = 52
# Cells the definition card keeps for itself, chrome included; 56 leaves it
# ~50 columns of prose, the narrowest comfortable measure for a definition.
_TERM_RAIL_DETAIL_RESERVED = 56


def term_rail_width(
    entries: tuple[GlossaryEntry, ...], *, available_width: int
) -> int:
    """Return the width ``#glossary-panel-terms`` should take.

    Wide enough for the widest row of *entries* to render on one line, clamped
    so the rail is never narrower than its historical fixed width and never
    takes the definition card's share of *available_width* -- the panel body's
    content width, or ``0`` before the first layout has settled.
    """
    if not entries:
        return _TERM_RAIL_MIN_WIDTH
    widest = max(build_term_row_text(entry).cell_len for entry in entries)
    desired = widest + _TERM_RAIL_CHROME
    cap = _TERM_RAIL_MAX_WIDTH
    if available_width > 0:
        # The ``- 1`` is ``#glossary-panel-detail``'s ``margin-left``.
        room = available_width - _TERM_RAIL_DETAIL_RESERVED - 1
        cap = min(cap, max(_TERM_RAIL_MIN_WIDTH, room))
    return max(_TERM_RAIL_MIN_WIDTH, min(cap, desired))
```

Reusing `build_term_row_text` is what keeps the measurement honest: the rail is measured
with the exact renderable the rows are built from, so the two can never drift. Add
`term_rail_width` to `__all__`.

### 3. `src/sase/ace/tui/modals/glossary_panel_view.py`

Add `_resize_term_rail`, which is a widget update and so belongs with the other widget
updates. Add `def _term_list(self) -> OptionList: ...` to the mixin's `TYPE_CHECKING`
protocol block (`_all_entries` is already declared there), and import `Horizontal`,
`NoMatches`, and `term_rail_width`.

```python
def _resize_term_rail(self) -> None:
    """Fit the term rail to its widest row within the panel's width."""
    try:
        body = self.query_one("#glossary-panel-body", Horizontal)
        term_list = self._term_list()
    except NoMatches:
        return
    width = term_rail_width(self._all_entries, available_width=body.size.width)
    current = term_list.styles.width
    if current is not None and current.is_cells and int(current.value) == width:
        return
    term_list.styles.width = width
```

The early return on an unchanged width matters: it keeps a resize or a reload that does
not move the rail from triggering a relayout, mirroring
`AgentList._refresh_requested_width`.

### 4. `src/sase/ace/tui/modals/glossary_panel_state.py`

In `_apply_snapshot`, call `self._resize_term_rail()` immediately after
`self._all_entries` is assigned and before `self._apply_filter(...)`. Declare
`def _resize_term_rail(self) -> None: ...` in the mixin's `TYPE_CHECKING` block
alongside the other cross-mixin declarations.

`_apply_snapshot` is the single funnel for initial load, `p`/`P` project cycling, and
refresh, so one call site covers every way the term set changes.

**Deliberately not hooked into `_set_entries` or `on_input_changed`.** Sizing per filter
keystroke would make the rail jitter as the user types and would relayout the whole body
on every character. Measuring `_all_entries` (the unfiltered set) rather than `_entries`
is what keeps the rail stable while filtering. This follows `sase/memory/tui_perf.md`
rule 6 (prefer selective updates over rebuilds) and rule 7's principle that chrome
should not chase the cursor.

### 5. `src/sase/ace/tui/modals/glossary_panel.py`

Add a resize handler to `GlossaryPanel` so the clamp is recomputed when the terminal
changes size (the container is a percentage of the terminal, so the room available to
the rail moves with it). `_apply_snapshot` also runs before the first layout has settled
in some paths, where `body.size.width` is `0` and `term_rail_width` falls back to the
`MAX`-only cap; this handler is what corrects that. `_event_widgets.on_resize` is the
same pattern.

```python
def on_resize(self, _event: events.Resize) -> None:
    """Re-fit the term rail when the terminal changes size."""
    self._resize_term_rail()
```

Import `events` from `textual`.

## Height is deliberately unchanged

`height: 90%; max-height: 44` stays as-is. It was checked, not skipped: the body gets
roughly 35 rows at `max-height: 44`, giving the rail 33 usable lines. Removing the four
wrapped lines takes the `sase` glossary from 33 rendered lines to 29, so the list stops
needing to scroll at all. Growing the height would not serve the reported problem.

## Tests

### `tests/ace/tui/modals/test_glossary_panel_rendering.py`

Unit-test `term_rail_width` against the real row builder:

- The widest row drives the result: entries whose widest row is 38 cells produce 44 with
  generous `available_width`.
- Clamped below: an empty tuple, and entries whose widest row is tiny, both return 32.
- Clamped above: an entry with a very long term returns 52.
- Room-constrained: `available_width=84` (a 100-column terminal) returns 32 even though
  the content wants 44.
- Pre-layout: `available_width=0` ignores the room clamp and returns 44.

### `tests/ace/tui/modals/test_glossary_panel.py`

Behavior tests on a mounted panel, following the existing fixtures there:

- After the initial load, `panel._term_list().styles.width` is the fitted cell width,
  not 32, for a snapshot containing a long term-plus-alias row.
- Typing in the filter so that only short terms match leaves the rail width unchanged
  (the anti-jitter guarantee).
- Cycling with `p` to a project whose terms are all short shrinks the rail back toward
  the minimum.

## Verification

1. `just install` first -- this is an ephemeral workspace.
2. `just check` (whole-repo lint gates plus the diff-scoped test lane). Hand it to
   `/sase_monitor` with a `--next` action if it runs long.
3. Regenerate the two PNG goldens the rail width moves. At the visual suite's 120x40
   terminal the container cap does not bind (`90%` resolves to 108), so the container is
   unchanged; only the rail moves, from 32 to 43, for the populated fixture (its widest
   row, `Agent Hood  hood · agent neighborhood`, is 37 cells). The empty fixture has no
   entries, so its goldens should not change -- if they do, something is wrong.

   ```bash
   just install-visual
   just test-visual tests/ace/tui/visual/test_ace_png_snapshots_glossary_panel.py
   # inspect .pytest_cache/sase-visual/ actual/expected/diff artifacts, then:
   just test-visual tests/ace/tui/visual/test_ace_png_snapshots_glossary_panel.py \
       --sase-update-visual-snapshots
   ```

   Expected to change: `glossary_panel_populated_dark_120x40.png` and
   `glossary_panel_populated_light_120x40.png`. Expected unchanged:
   `glossary_panel_empty_{dark,light}_120x40.png`.

4. Confirm in the real TUI, not only in tests: open the panel in `sase ace` (`g` prefix
   -> glossary panel) on a wide terminal and check that no `sase` term row wraps and
   that the definition body is wider than before.
5. `just check-full` before landing, through `/sase_monitor` with a `--next` action --
   it routinely outruns a single turn, and the visual suite is not in `just check`.
