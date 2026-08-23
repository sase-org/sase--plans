---
tier: tale
title: Color-code Agents tree guides by hierarchy depth
goal:
  Make Agents-tab child connectors immediately legible at every practical hierarchy
  depth, including on the selected row, without changing tree layout or navigation
  behavior.
size: medium
proposed_by: bbugyi200.athena.0bk
create_time: 2026-08-23 08:01:22
status: wip
---

# Plan: Color-code Agents tree guides by hierarchy depth

## Goal

Replace the Agents tab's single dim-grey tree-indent style with a curated, depth-aware
structural palette. Each ancestor guide should retain the color of the level it
represents, while the terminal `└─` branch should use the current row's depth color.
Selected rows should strengthen the same color with bold weight so the connector remains
clear over the highlight background.

## Design contract

- Preserve the exact plain-text footprint and alignment for every row: depth 1 remains
  ` └─`, depth 2 remains ` │  └─`, and deeper rows continue by inserting one `│  `
  segment per ancestor. Root rows remain unprefixed.
- Use a fixed, high-contrast six-color palette ordered for adjacent distinction and a
  calm progression through the current dark theme: sky blue (`#5FAFFF`), mint
  (`#5FD7AF`), gold (`#FFD75F`), rose (`#FF87AF`), lavender (`#D7AFFF`), and cyan
  (`#5FD7FF`). The common first three depths therefore remain especially distinct on
  both the normal surface and the translucent purple selection background.
- Color each visible connector segment independently. On a depth-3 row, for example, the
  first `│` uses level 1 sky blue, the second `│` uses level 2 mint, and `└─` uses level
  3 gold. This lets a user's eye trace a level vertically instead of turning the entire
  indent into a row-level rainbow block.
- Remove the current `dim` treatment from tree connectors. Render the palette at normal
  weight on ordinary rows and bold the same per-level colors on the selected row. Keep
  the existing glyph shape and indentation as a non-color cue for color-vision and
  low-color-terminal resilience.
- Resolve depths beyond the palette with deterministic modulo cycling. This keeps the
  hot render path total for malformed or unusually deep trees, guarantees adjacent
  levels use different colors, and avoids introducing data-dependent theme generation.
- Keep the implementation presentation-only: do not change tree projection, folding,
  navigation, row identity, CSS highlight geometry, configuration, or the Rust core. The
  render cache already keys on `tree_depth` and `is_selected`, so static palette
  constants require no cache-key expansion or invalidation path.
- Do not add a feature flag. This is an atomic visual-polish change with no behavioral
  branch or migration, and it will land only with unit and exact visual coverage.

## Implementation

1. Define the depth palette alongside the existing Agents-list tree glyph constants in
   `src/sase/ace/tui/widgets/_agent_list_styling.py`, with comments documenting the
   ordering, selected-row contrast, accessibility fallback, and cycling behavior.
2. Refactor `_append_tree_indent` in
   `src/sase/ace/tui/widgets/_agent_list_render_agent.py` to append the leading spacing,
   each ancestor `_TREE_GUIDE`, and the final branch as separate Rich `Text` spans.
   Select each span's color from its one-based logical depth, accept the existing
   `is_selected` state, and add bold weight only for selected rows. Pass selection from
   `format_agent_option` without altering other row-formatting branches.
3. Add focused rendering tests under `tests/ace/tui/widgets/` that inspect both plain
   text and Rich styles for roots and depths 1 through 3, verify a depth-3 row carries
   three correctly ordered colors, verify selected connectors are bold and never dim,
   and exercise a depth beyond the palette to lock down safe deterministic cycling.
   Include a cache-path assertion if useful to prove selected and unselected renders do
   not alias; `is_selected` is already an input to `agent_render_key`.
4. Refresh only the affected Agents clan-tree PNG goldens produced by
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_clans.py`. Inspect the generated
   actual/expected/diff artifacts before accepting them, paying particular attention to
   the selected direct child and the fully expanded multi-depth state; unrelated pixels
   must remain unchanged.

## Verification

1. Run `just install` before repository checks so the workspace environment matches the
   current lock state.
2. Run the new focused Rich-span tests plus the existing Agents tree-title, status-icon,
   tree-rendering, and render-cache tests that protect connector placement and cached
   selection behavior.
3. Run the focused clan-tree visual test once against the old goldens to inspect its
   diffs, update intentional snapshots with `--sase-update-visual-snapshots`, inspect
   the resulting PNGs at normal and selected depths, and rerun the same test without the
   update flag to require exact equality.
4. Run `just check` for the mandatory whole-repository lint gates and diff-scoped test
   lane. If its selector escalates or reports unusual selection, follow the repository
   guidance for broader verification.

## Acceptance criteria

- Child prefixes are vivid and readable on ordinary and highlighted Agents-tab rows; no
  tree connector uses the old `dim #808080` style.
- Adjacent hierarchy levels have visibly distinct colors, and a connector's color is
  stable wherever that level appears vertically in the tree.
- Selected connectors preserve their depth hue and gain bold weight rather than being
  washed out by the selection background.
- Root rows, text content, cell widths, alignment, folding, navigation, status/type
  glyphs, and runtime suffix layout remain unchanged.
- Arbitrary positive depths render deterministically without exceptions, including
  depths beyond the curated palette.
- Focused semantic tests, exact PNG regression tests, and `just check` all pass.
