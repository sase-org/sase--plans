---
tier: tale
title: Content-aware sizing for the K preview panel
goal:
  The K preview panel keeps its current size when content fits and grows toward
  near-full-screen when it does not.
size: medium
proposed_by: bbugyi200.athena.0ol
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0ol](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0ol.md)
- **COMMITS:**
  - [f330e2b](https://github.com/sase-org/sase/commit/f330e2b949b5e9744213dd48d7c785cab29b3de1)
    — feat(preview): content-aware sizing for the prompt preview panel

# Content-Aware Sizing for the Prompt Preview Panel (`K`)

## Goal

Pressing `K` in normal mode in the prompt input opens `PreviewPanelModal`
(`src/sase/ace/tui/modals/preview_panel_modal.py`, pushed from
`src/sase/ace/tui/widgets/_prompt_preview.py`). Today its size is fixed in
`src/sase/ace/tui/styles.tcss` (`PreviewPanelModal > Container`: `width: 96%`,
`height: 85%`, `max-width: 150`, `max-height: 42`). A long xprompt like
`#research_swarm` (~300 lines) therefore opens in a 42-row box even on a 70-row
terminal.

Make the panel size itself to its content:

1. **Content fits the current size** → keep exactly the current size (the "baseline").
   Small previews look the same as today; existing visual snapshots stay unchanged.
2. **Content does not fit** → grow just enough to show it all, up to a **maximum** that
   is nearly the whole screen but leaves a thin frame of dimmed backdrop so it still
   reads as a floating modal.
3. **Content is very long** (`#research_swarm`) → open at the maximum immediately.

The panel never shrinks below the baseline, and it never jitters while the user is
reading.

## Design

### Geometry model (pure, unit-testable)

Add a new pure module `src/sase/ace/tui/modals/preview_panel_sizing.py`. It has no
Textual imports, so it is cheap to unit-test exhaustively.

```python
@dataclass(frozen=True)
class PanelGeometry:
    width: int
    height: int

def baseline_geometry(screen_w: int, screen_h: int) -> PanelGeometry
def max_geometry(screen_w: int, screen_h: int) -> PanelGeometry
def compute_panel_geometry(
    *, screen_w: int, screen_h: int,
    content_rows: int,      # rows the body needs at the chosen width
    content_cols: int,      # columns the body wants (see width rule)
    chrome_rows: int,       # every non-body row inside the container
    chrome_cols: int,       # every non-body column inside the container
) -> PanelGeometry
```

- **Baseline** mirrors today's TCSS exactly: `width = min(int(screen_w * 0.96), 150)`,
  `height = min(int(screen_h * 0.85), 42)`. (Check that Textual's percentage rounding
  matches; if it doesn't, copy Textual's rounding so the baseline is pixel-identical to
  today.)
- **Maximum** = `width = screen_w - 4`, `height = screen_h - 2`: two backdrop columns on
  each side and one backdrop row top and bottom. Terminal cells are about twice as tall
  as they are wide, so this looks like an even gutter. Keep the margins as named module
  constants (`_MAX_MARGIN_COLS = 2`, `_MAX_MARGIN_ROWS = 1`).
- Guard tiny screens: `max` is clamped to at least `baseline`. The baseline is also
  clamped to the screen, so the result is always `baseline <= result <= max` and never
  exceeds the screen.
- `height = clamp(chrome_rows + content_rows, baseline.height, max.height)`.
- `width = clamp(chrome_cols + content_cols, baseline.width, max.width)`.
- Keep the constants (`0.96`, `0.85`, `150`, `42`) next to a comment pointing at the
  TCSS block, and keep the TCSS as the pre-mount fallback. Add a unit test that parses
  `styles.tcss` and asserts the two copies agree, so they cannot drift apart.

### Width rule

Height is the main axis. Width grows only when the source genuinely has wide lines, and
it uses the **95th-percentile** display width of source lines (after expanding tabs,
with `rich.cells.cell_len`) plus the line-number gutter. That way one pathological long
line, such as a URL or a minified JSON blob, cannot blow the panel out to full width.
The width is picked first. The wrapped row count is then computed at the resulting body
width, so a wider panel correctly needs fewer rows.

### Measuring content rows

Reuse the wrap model the search feature already trusts (`modals/preview_search.py`,
`_wrapped_row_offsets` plus the plain-render notice row and the `PLAIN_RENDER_MAX_LINES`
truncation). Add a public helper there:

```python
def count_wrapped_rows(content: str, available_width: int, lexer: str, *, limit: int) -> int
```

It shares the row logic with `build_search_result` (refactor so neither copy can drift),
and it **stops early** once the running total exceeds `limit`. Pass
`limit = max.height - chrome_rows`, which keeps the cost bounded to about one screen of
lines even for 5,000-line files. Add a fast path: if the logical line count alone is
`>= limit`, return `limit + 1` without wrapping anything.

### Chrome rows/cols (analytical, no flicker)

The geometry must be known **before the first paint** so the panel opens at its final
size instead of popping from baseline to large. Compute chrome from the modal's own
known structure in one helper on the modal, `_chrome_rows(inner_width)`, with named
constants that mirror the TCSS:

- container: thick border 2 + vertical padding 2
- title: rendered line count of `_build_title()` at the inner width (1 or 2 lines, and
  more if a long path wraps; measure with a Rich `Console(width=...)`), plus
  `margin-bottom: 1`
- properties band (only when displayed): its Rich-measured height at the inner width,
  capped at `max-height: 12`, plus `margin-bottom: 1` (its top and bottom borders are
  part of the widget's box, so check whether `max-height` includes them and match
  Textual)
- scroll box: border 2
- footer: `height: 3` + `margin-top: 1`

Chrome columns: container border 2 + horizontal padding 4 + scroll border 2 + scroll
padding 2 + the stable scrollbar gutter (`scrollbar-size-vertical`, default 2).

These constants duplicate TCSS numbers, so the integration test below is the guard: it
opens a mid-sized payload and asserts that the scroll viewport height equals the content
row count exactly (no scrollbar, no blank slack). If someone edits the TCSS without
updating the constants, that test fails.

### Applying geometry in the modal

In `PreviewPanelModal`:

- `_apply_geometry()` reads `self.app.size`, computes chrome and content dimensions,
  calls `compute_panel_geometry`, and sets on `#preview-modal-container`:
  `styles.width`, `styles.height` (ints, in cells) and `styles.max_width = None`,
  `styles.max_height = None`. The container stays centred by the existing
  `align: center middle`.
- Call it from `on_mount` (before the first compositor frame, so there is no flash) and
  from `on_resize` (screen resize recomputes from scratch; see ratchet below).
- Skip the work entirely when the logical line count plus chrome is clearly under the
  baseline height and the widest line is under the baseline width. That fast path keeps
  the common small preview free.

### Rendered-Markdown view: grow-only ratchet

Markdown xprompts (including `#research_swarm`) open in **rendered** view by default
(`payload.default_view == "rendered"`). Rendered Markdown adds heading spacing and
code-block frames, so its height can exceed the source row count. The source estimate is
used for the initial, flicker-free size. Then:

- When the rendered view becomes visible (`_set_view_mode("rendered")` after
  `_rendered_ready`), schedule `call_after_refresh` to measure the rendered widget's
  real height (`#preview-rendered` `outer_size.height`, since it is `height: auto`
  inside the scroll). If it needs more rows than the current viewport, grow (still
  capped at max).
- **Grow-only**: keep `self._geometry_floor`, the largest geometry applied since the
  last screen resize. Toggling `R` (source/rendered) or `p` (properties) or opening
  search never shrinks the panel, so the frame does not jump while the user flips views.
  Only a real screen resize resets the floor and recomputes.
- Properties view: size it from the same measurement path (a Rich render of
  `_properties_view` at the body width) through the same ratchet.

For very long content the source estimate already hits max, so the ratchet is a no-op
and the panel opens at max with no second step.

### Scope

- The sizing lives in `PreviewPanelModal`, so every caller of this panel gets it: `K` in
  the prompt input, plus the plans pane, artifact files, beads browse/issue, and alias
  history previews. This keeps them consistent, since they are the same panel. Small
  payloads keep the exact baseline, so those callers look unchanged unless their content
  is long.
- Not in scope: `GlossaryPreviewModal`, `RepoPreviewModal`, `SpellcheckPanelModal`, and
  `WordDefinitionModal`, the other panels `K` can open for glossary terms, repo names,
  and plain words. Their content is short. Note them in the final summary as possible
  follow-ups if the user wants the same behaviour there.
- No Rust core change: this is presentation-only Textual layout.
- No new keybinding or config key, so `default_config.yml` is unchanged.

## Implementation Steps

1. Read the `tui` reference memory and its children (`tui_perf`, `tui_screenshot`) with
   `/sase_memory_read` before touching TUI code, and `lint_and_test` before finishing.
2. `modals/preview_search.py`: factor the wrapped-row walk into a shared internal
   generator and add `count_wrapped_rows(..., limit=...)` with the early exit and the
   line-count fast path. `build_search_result` behaviour is unchanged.
3. New `modals/preview_panel_sizing.py`: `PanelGeometry`, `baseline_geometry`,
   `max_geometry`, `compute_panel_geometry`, and a `p95_line_width(content)` helper.
4. `modals/preview_panel_modal.py`: add chrome measurement, `_apply_geometry()`, the
   `on_mount` and `on_resize` hooks, the grow-only floor, and the rendered and
   properties post-layout measurement via `call_after_refresh`. Guard every step with
   `self.is_attached`. Keep the modal file readable. If it grows past about 650 lines,
   move the geometry glue into a small `_preview_panel_geometry.py` mixin, matching the
   existing `_source_file_actions.py` mixin pattern.
5. `styles.tcss`: leave the baseline rules in place, and add a comment above
   `PreviewPanelModal > Container` saying the values are the pre-mount baseline and are
   mirrored in `preview_panel_sizing.py`.

## Tests

- `tests/ace/tui/modals/test_preview_panel_sizing.py` (new, pure):
  - Content that fits returns exactly the baseline (for example 120x40 → 115x34, and
    200x60 → 150x42).
  - Moderate overflow grows height to exactly `chrome + rows`.
  - Huge content clamps to `screen - margins`.
  - Tiny screens (for example 40x12) never exceed the screen, and max is at least
    baseline.
  - The p95 width rule ignores a single very long outlier line but widens for
    consistently wide content.
  - A TCSS/constant agreement test.
- `tests/ace/tui/modals/test_preview_search.py` (or an existing search test file):
  `count_wrapped_rows` agrees with `build_search_result` row offsets, early-exits at the
  limit, and handles the plain-render notice row.
- `tests/ace/tui/modals/test_preview_panel_modal.py` (extend):
  - A short payload at `size=(100, 30)` keeps baseline geometry.
  - A 300-line source payload at `size=(160, 60)` opens at `(156, 58)`.
  - A payload sized between baseline and max opens with scroll viewport height equal to
    its wrapped row count and `max_scroll_y == 0` (the chrome guard).
  - Toggling `R` or `p` never shrinks the container. A rendered Markdown payload whose
    rendered height exceeds the source estimate grows after rendering.
  - Resizing the app (`pilot.resize_terminal`) recomputes geometry.
- `tests/ace/tui/widgets/test_prompt_normal_mode_preview.py`: one end-to-end `K` test
  with a long xprompt payload, asserting the pushed modal's container is taller than the
  baseline.
- Visual: add one PNG snapshot to
  `tests/ace/tui/visual/test_ace_png_snapshots_preview_panel.py` for a long markdown
  xprompt (about 300 lines, like `#research_swarm`) at 120x40, showing the maxed panel.
  Existing preview snapshots must remain byte-identical because their content fits the
  baseline. Generate and inspect the PNG per the `tui_screenshot` memory, confirm the
  one-row and two-column backdrop gutter looks balanced, and adjust the margin constants
  if it does not.

## Verification

- `just check` (per the `lint_and_test` memory), including symvision for the new public
  helpers.
- Manual or screenshot check: `K` on `#research_swarm` in a large terminal opens near
  full-screen immediately with no flash. `K` on a short xprompt looks identical to
  today. Resizing the terminal while the panel is open re-fits it.
