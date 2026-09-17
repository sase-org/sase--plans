---
tier: tale
title: Fix usage indicator color boundaries
goal:
  Give each of the ten foreground colors exactly ten displayed integer percentages while
  preserving the special exhausted style at zero.
size: small
proposed_by: bbugyi200.athena.0me
create_time: 2026-09-17 10:53:04
status: wip
---

# Plan: Fix usage indicator color boundaries

## Problem and scope

The usage-window indicator has eleven numeric styles: an inverted red-background style
for `0%`, plus ten foreground colors for positive capacity. Currently,
`_usage_percent_bucket()` in `src/sase/ace/tui/widgets/_usage_indicator_palette.py` uses
`min(9, int(clamped // 10))`. Consequently, positive integers 1–9 use red, 10–19 use
coral/orange, and 90–100 use blue. A read-only enumeration confirmed foreground bucket
populations of 9, 10, 10, 10, 10, 10, 10, 10, 10, and 11.

This is a focused presentation correction suitable for one coding agent. Keep it in the
existing Python TUI palette; no shared backend behavior or Rust API needs to change.
Preserve the palette's actual colors, capacity data, provider policies, refresh
behavior, and layout. No configuration or feature flag is needed.

## Required behavior

Choose foreground colors using the displayed whole-number percentage, following the
existing flooring behavior of `format_usage_percent_text()` in
`src/sase/ace/tui/widgets/_usage_indicator_format.py`:

| Displayed percentage | Existing palette index | Dark foreground | Light foreground |
| -------------------- | ---------------------- | --------------- | ---------------- |
| 1–10%                | 0                      | `#FF5F6D`       | `#A22534`        |
| 11–20%               | 1                      | `#FF805F`       | `#A03620`        |
| 21–30%               | 2                      | `#FFA552`       | `#8C480E`        |
| 31–40%               | 3                      | `#EBC04F`       | `#775800`        |
| 41–50%               | 4                      | `#CED44C`       | `#5F6500`        |
| 51–60%               | 5                      | `#AADC64`       | `#456C1B`        |
| 61–70%               | 6                      | `#78DB8D`       | `#206F3C`        |
| 71–80%               | 7                      | `#4CD4B0`       | `#006E56`        |
| 81–90%               | 8                      | `#48CCD0`       | `#006C6C`        |
| 91–100%              | 9                      | `#65C3ED`       | `#006381`        |

- Keep `0%` as the existing red-background exhausted style. Palette index zero must
  remain available at zero because `usage_zero_value_style()` obtains its background
  through `usage_percent_color(0, ...)`.
- Positive values below 1% continue to display `<1%` in red foreground, without the
  exhausted background.
- Fractional readings follow their displayed integer: 10.99% displays `10%` in red; 11%
  begins the second color. Apply the same rule at every boundary, including 90.99%
  versus 91%. Do not use `ceil(raw_percent / 10) - 1`, which would allow two readings
  displayed as `10%` to have different colors.
- Preserve finite-value clamping to 0–100 and the current non-finite fallback to zero,
  including for NaN and both infinities.
- Preserve stale/unknown-age neutral styling, passed-reset `?%` styling, vendor
  rejection markers, and the existing special handling of stale zero values.

## Implementation

1. Update `_usage_percent_bucket()` after its existing normalization/clamping. Floor the
   clamped percentage and map it with `max(0, min(9, (math.floor(clamped) - 1) // 10))`,
   or an equivalent expression with identical behavior. Keep this a pure constant-time
   calculation. Update the helper's docstring and palette range comments to describe
   displayed ranges 1–10 through 91–100, including the subpercent/zero fallback.

2. Correct and extend `tests/test_provider_usage_indicator_presentation_style.py`:
   - Replace the existing boundary test, which currently enforces the bug, with explicit
     expected palette/range cases for both dark and light themes. Cover all integers
     1–100 so every foreground color is assigned exactly ten positive integer
     percentages. Expected results must come from the range table, not from reproducing
     the implementation formula.
   - Cover 0, a negative value, a positive subpercent, 100, greater than 100, NaN, and
     both infinities. For each transition, cover the upper integer of the preceding
     range, a fraction still displaying that integer, and the next integer; include
     9/10/10.99/11 and 90/90.99/91 explicitly.
   - Add parameterized rendered-style regression coverage through
     `build_usage_indicator_segment()` for both themes. Verify that 10 and 10.99 render
     `10%` in red foreground on the badge surface, while 11 uses the next foreground
     color. Verify the window name, percent, and countdown share the selected color, and
     retain existing zero/subpercent/state tests.
   - Change `test_defined_colors_meet_minimum_contrast_on_badge_surface()` to sample one
     representative from each corrected bucket, such as 10, 20, ..., 100. Its current
     samples 0, 10, ..., 90 would duplicate red and omit blue after the fix. Assert ten
     distinct sampled foreground colors so the contrast test continues to cover the
     complete palette.

3. Update the existing dark/light palette snapshot fixture in
   `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py` to
   show all eleven numeric styles using 0, 10, 20, ..., 100. Currently it shows zero and
   15–95, omitting the positive red foreground entirely. Preserve the existing snapshot
   names and dimensions if all eleven windows fit; verify the scene has no hidden-window
   disclosure. Update the fixture's description to reflect eleven styles, then inspect
   and refresh the two corresponding PNG goldens under
   `tests/ace/tui/visual/snapshots/png/`. Inspect any other changed usage-indicator
   snapshots; accept only color changes caused by the corrected ranges (for example a
   visible fresh 50%).

## Verification and acceptance

Read the current `lint_and_test.md` reference memory before finishing implementation.

1. Run the targeted nonvisual tests:

   ```bash
   just test -- tests/test_provider_usage_indicator_presentation.py tests/test_provider_usage_indicator_presentation_style.py tests/test_provider_usage_indicator_presentation_layout.py tests/test_provider_usage_indicator_widget.py
   ```

2. Run the two existing usage-indicator visual test modules through the pinned renderer:

   ```bash
   just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py
   ```

   Inspect failures' actual/expected/diff images, update only intentional affected
   goldens with `--sase-update-visual-snapshots`, and rerun those modules without the
   update flag. Ensure both palette scenes visibly cover the red background at zero and
   every positive foreground color.

3. Run `just check` after the final implementation changes. Follow the `/sase_monitor`
   skill if verification becomes long-running; run `just fix` (or at least `just fmt`)
   before handing verification to a monitor.

Accept when the displayed ranges match the table in both themes, each positive integer
color has ten members, zero remains visually distinct, fractional values agree with
their displayed percentage, all ten colors retain contrast coverage, and the targeted
tests, visual tests, and `just check` pass.
