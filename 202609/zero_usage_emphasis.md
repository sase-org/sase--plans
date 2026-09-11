---
tier: tale
title: Simplify stale usage labels and emphasize exhausted windows
goal: Usage indicators omit the stale tilde and make exact zero remaining capacity
  unmistakable with an accessible token-scoped red background.
size: small
proposed_by: bbugyi200.athena.0j8.f0
status: done
---

# Plan: Simplify stale usage labels and emphasize exhausted windows

## Outcome and scope

Make the ACE top-right usage indicator quieter in ordinary stale states and unmistakable
when a selected usage window is exhausted:

- Remove the visible `~` suffix from every numeric remaining-percentage label.
- Render the background of the exact `0%` token red, while leaving the rest of that
  window on the normal badge surface.

This is a focused follow-up to `plan:202609/grouped_usage_window_polish.md`. It is pure
Python TUI presentation work: keep the existing Rust-produced usage projection,
selection policies, configurable thresholds, provider grouping, compact window names,
ordering, width budget, tooltips, and interaction behavior intact. Do not add a feature
flag, configuration key, migration, or compatibility branch.

## Current behavior and boundaries

- `src/sase/ace/tui/widgets/_provider_usage_indicator.py::_entry_fragment` formats each
  selected window. For stale or unknown-age numeric observations it currently appends
  `~` to `format_usage_percent_text(...)`, assigns the neutral value color, and explains
  the marker in `_entry_tooltip_lines`.
- The same renderer applies one `usage_value_style` to a window's optional name,
  percentage, spaces, and countdown. The fragment retains a separate `value_color` used
  to color all `|` separators owned by the provider.
- `src/sase/ace/tui/widgets/_usage_indicator_palette.py` owns the theme-specific
  ten-bucket foreground palette and explicit badge/gap surfaces. Its zero-capacity
  bucket is already red in each theme: `#FF5F6D` on dark and `#A22534` on light.
- `format_usage_percent_text` deliberately clamps/floors values into `0%`, `<1%`, whole
  percentages, and `100%`; passed resets display `?%` instead. This change must consume
  those established display states rather than altering numeric formatting.
- The width packer measures rendered `Text` fragments by terminal cell width. Removing
  one glyph from a stale fragment should naturally reclaim one cell without adding a
  second width-calculation path.

Do not change the Providers · Usage modal, model-picker usage hints, collection,
freshness calculation, reset calculation, provider routing pills, or provider-disable
behavior. Do not add disk, network, subprocess, config, or store work to the render
path.

## Visual and state contract

### Stale and unknown-age readings

Render a stale numeric window with the same visible grammar as a fresh one, for example
`🎭 62% 3d4h`, never `🎭 62%~ 3d4h`. Preserve uncertainty through the existing neutral
foreground for the window name, percentage, and countdown and through explicit tooltip
prose. Replace the marker-dependent tooltip sentence with prose that stands alone, such
as `last observed capacity; it may be out of date`; retain the structured
`freshness: stale` or `freshness: unknown` line.

Do not remove unrelated tildes elsewhere in ACE or change the underlying freshness
value. Reset-passed windows remain `?% 0h0m↻`, unknown reset times remain `?`, and
vendor-rejected windows retain `!`.

### Exact zero-capacity emphasis

Give only the rendered `0%` token a red background. Use the existing theme's
zero-capacity bucket color as that background and the corresponding badge-surface color
as its bold foreground. This is an intentional inversion of an already contrast-tested
pair: dark mode becomes dark badge-surface text on bright red, and light mode becomes
light badge-surface text on dark red. Assert at least 4.5:1 foreground/background
contrast in both themes.

Keep the optional compact name, rejection marker, spaces, countdown, provider icon, and
`|` separators on their existing backgrounds. In particular, do not flood the whole
window or provider group red. Retain the window's ordinary zero-bucket `value_color`
metadata, so its name/countdown foreground and provider-owned dividers continue using
the red capacity color on the normal badge surface.

Key the emphasis to the established formatted zero state, including values clamped to
zero, while keeping `<1%` on the normal badge surface. A passed-reset `?%` must never
receive the zero treatment even if its retained last observation was zero. A stale or
unknown-age `0%` still receives the red percentage background because it visibly reports
an exhausted last observation; its name and countdown remain neutral and its tooltip
discloses uncertainty. Missing or nonnumeric values should retain their current
normalization and must not introduce crashes.

Compose the percentage as its own Rich span so this override cannot leak through
inherited styles from the surrounding window or routing pill. Preserve explicit
`not dim`/`not reverse` protection. A helper such as `usage_zero_percent_style` belongs
in `_usage_indicator_palette.py`; derive it from the existing theme-specific bucket and
surface helpers rather than duplicating red literals in the renderer.

## Implementation sequence

1. In `_usage_indicator_palette.py`, add one focused exported style helper for the
   exhausted percentage token. Build the bold foreground/background pair from
   `usage_percent_color(0, dark=...)` and the existing badge-surface color. Keep the
   ten-bucket palette and all nonzero value styles unchanged.
2. In `_provider_usage_indicator.py`, stop appending `~` for stale or unknown-age
   numeric observations. Preserve the stale-neutral `value_color`, then append the
   percentage separately with the exhausted style only when its established formatted
   text is exactly `0%`; continue appending the surrounding space and countdown with the
   ordinary window value style. Update stale tooltip prose so it no longer refers to a
   removed glyph.
3. Update `docs/ace.md` to remove the stale-marker legend and explain that stale or
   unknown-age readings use neutral text plus tooltip disclosure. Document the red
   background on exact `0%`, its token-only extent, and the unchanged meanings of `<1%`,
   `?% 0h0m↻`, `?`, and `!`. Keep the grouped example, threshold link, and
   provider-divider description intact.
4. Update focused presentation, composition, and visual tests and regenerate only the
   affected PNG goldens. Avoid touching configuration docs or defaults unless a caller
   search reveals text that actually describes the removed marker; display thresholds
   and precedence are unchanged.

## Acceptance and verification

Extend `tests/test_provider_usage_indicator_presentation.py` to prove:

- Fresh, stale, and unknown-age numeric readings never contain `~`; stale and
  unknown-age name/percent/countdown spans remain bold neutral where the zero override
  does not apply, and tooltip text communicates that the retained capacity may be out of
  date without naming a glyph.
- Exact `0%` has the theme-specific red background and badge-surface foreground in both
  dark and light themes. Its adjacent spaces, name, countdown, provider icon, rejection
  marker, and separators retain their own expected surfaces/styles.
- The exhausted style meets a 4.5:1 contrast floor in both themes.
- `0`, negative/clamped-to-zero, `<1%`, an ordinary first-bucket value such as `7%`,
  stale zero, rejected zero, reset-passed with a retained zero observation, and unknown
  reset time all follow the state contract above.
- Removing the stale glyph updates exact text and cell widths without breaking exact-fit
  or one-cell-short packing, overflow counts, complete-window boundaries, group order,
  or divider coloring.

Update `tests/test_provider_disables_indicator_usage.py` where its composed output or
style assertions cover a zero window. Assert the final composed `0%` span keeps the red
background even beside hard/soft disable and priority pills, with no inherited routing
background.

In `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py`,
keep the existing stale/status and disable-pill scenes and update their affected
goldens. Make the dark/light ten-bucket scene include an exact-zero first sample (while
retaining coverage of the other buckets), or add an equivalently focused two-theme
scene, so the token-only red treatment receives human visual review. Inspect the
rendered actual, expected, and diff PNGs for both themes: the `0%` block must be
readable, compact, and visually bounded, stale labels must be uncluttered, and
neighboring surfaces must not bleed.

Run focused nonvisual tests first, including:

```text
pytest tests/test_provider_usage_indicator_presentation.py tests/test_provider_disables_indicator_usage.py
```

Before completion, read `lint_and_test.md` through `/sase_memory_read` and follow its
required verification. Run the affected visual tests with the repository's snapshot
update workflow, inspect the generated diffs, rerun without update mode, and run
`just check` (using `just install` first only if dependencies are stale). Use
`/sase_monitor` for long-running verification and consult `symvision.md` through
`/sase_memory_read` before addressing any Symvision failure.

Completion requires exact token scoping, accessible dark/light rendering, unchanged
selection/configuration behavior, updated documentation, intentional inspected goldens,
and all required focused and repository checks passing.
