---
tier: tale
title: Refine usage grouping, exhausted values, and the Fable default
goal:
  Give usage indicators clear spacing and neutral grouping, highlight exhausted values
  with their countdowns, and show observed Claude Fable windows by default.
size: medium
proposed_by: bbugyi200.athena.0j8.f0.f2
create_time: 2026-09-11 10:05:49
status: wip
---

# Refine usage grouping, exhausted values, and the Fable default

## Outcome

Give the ACE top-bar usage block room to breathe, make provider boundaries clear, and
make an exhausted window's remaining capacity and reset countdown read as one
highlighted value. Show Claude's observed weekly Fable window at any capacity by
default, with the existing configuration controls available to override it.

This is a medium tale: one coder can implement the presentation, package default,
documentation, and focused regression coverage together. The current checkout already
has provider grouping, complete-window overflow packing, no stale `~` or collector
caution triangle, and an inverted red background on exact `0%` alone.

## Display contract

The following examples show the body of the usage block. In the real output, add exactly
two ASCII spaces before the first provider icon and two after the entire block. Those
spaces belong to the usage segment and its measured cell width.

```text
🎭 (10% 1d10h | fable 0% 1d10h)  🤖 81% 5d2h
🎭 10% 1d10h
🎭 fable 100% 1d10h
🎭 (10% 1d10h | fable 0% 1d10h)  +2
🎭 10% 1d10h  +3
```

- Render one icon per provider, followed by one space. Separate providers with exactly
  two spaces. A provider boundary has no pipe.
- Wrap the windows in one pair of parentheses only when that provider has two or more
  **actually visible** windows. There is no interior padding beside the parentheses.
  Join its windows with `|`, with no trailing pipe.
- Give pipes and parentheses the explicit neutral, normal-weight badge style in both
  themes. They convey structure without adopting a window's capacity color. Explicitly
  reset bold, dim, reverse, and background so a neighboring red value or routing pill
  cannot bleed into punctuation.
- Retain the existing continuous badge surface within each provider. Outer padding and
  the gaps between providers use the existing shell/gap surface. This produces distinct
  provider sections surrounded by a quiet margin.
- Keep the existing default-first window order and provider/extra-window attention
  ordering. When the default is absent, a single named extra is unparenthesized; two
  named extras are parenthesized. Preserve compact-name disambiguation.
- When the rendered percentage token is exactly `0%`, highlight the complete
  `0% <countdown>` run, including its internal space, with the existing inverted red
  style. Reuse the dark/light palette and its contrast-safe foreground. The name, space
  preceding the value, rejection marker, parentheses, pipes, provider icons, and
  following gaps remain on their ordinary surfaces.
- Preserve the current zero eligibility rule: clamped zero and retained
  stale/unknown-age zero receive the same full-value highlight. Unknown reset time
  therefore renders highlighted `0% ?`. Positive fractional capacity stays `<1%` without
  a red background. A passed reset stays neutral `?% 0h0m↻`, including when the retained
  observation was zero. Freshness and rejection details stay available in the tooltip; a
  rejected zero still has a separate `!` marker.

## Width and composition

Keep the existing half-top-bar usage budget and longest-prefix-of-complete-windows
packing policy. Include all padding, provider gaps, parentheses, and disclosure text in
terminal-cell measurements using Rich's `cell_len`, including emoji and unknown-provider
fallback icons.

For each candidate prefix, derive parentheses from that prefix's visible count per
provider. The transition from one to two visible windows adds both parentheses as well
as the second window and its separator. Closing parentheses must be present before
measuring the candidate. A partially visible provider is never left open, and a prefix
with one visible window has no parentheses even if more were selected.

Keep `+N` outside all provider parentheses, separated by two gap-surface spaces, and
inside the block's final two-space padding. `N` counts hidden windows, including hidden
windows from a partially visible provider. Full selected details remain in the tooltip
and usage/overflow clicks keep opening Providers · Usage.

If no complete-window prefix plus disclosure fits, use this explicit fallback order,
accepting the first candidate whose cell width fits:

1. Two spaces, `usage N`, two spaces.
2. Two spaces, the bare total `N`, two spaces.
3. One space, the bare total `N`, one space.
4. The bare total `N` without padding.
5. `…`, then empty output if even one cell is unavailable.

This only relaxes padding for text-only disclosure; any rendered provider icon and
window retain the full two-space outer margin. Empty groups and zero budget yield empty
text, with no whitespace-only widget. Test digit-count transitions as well as one-cell
boundaries. Shrinking and growing the bar must recalculate balanced groups and restore
complete content without changing selection or attention ranking.

The usage renderer owns the outer margin even when appended to a routing pill. Remove
the internal `leading_space` switch and its caller/test plumbing rather than letting
routing presence suppress the new margin. Preserve the routing pill's own text and
styles; its existing colored trailing space is distinct from the usage segment's two
neutral spaces. Avoid adding a second copy via widget CSS.

## Package configuration

Change `src/sase/default_config.yml` at `llm_provider.usage_metrics.indicator.providers`
to:

```yaml
providers:
  claude:
    windows:
      "weekly:claude-fable-5": always
```

The exact key is corroborated by the current Claude collector tests in
`tests/llm_provider/test_claude_usage.py`. Use the existing policy machinery: `always`
means display an eligible, observed window at any percentage, including 100%; missing or
collection-ineligible windows are not synthesized, and selected Fable windows still
participate in width overflow.

Keep `weekly_all: always` and the general strict-below-20-percent fallback. Package
defaults are recursively merged before indicator validation/projection. An explicit user
value for this exact key, including `never` or a threshold, wins over the bundled
`always`. An empty provider map does not erase bundled entries, and broader
provider/global defaults have lower policy precedence than the exact window override.
Document overriding the exact key to hide Fable or restore
`{below_remaining_percent: 20}`. `indicator.enabled: false` still hides all entries.

Update the corresponding `indicator.providers` default metadata in
`src/sase/config/sase.schema.json` to match the package default while retaining its
open-ended provider/window schema. Update the nearby YAML examples so the newly active
default and optional overrides are clear and there is no duplicate active `providers`
key. Collection's sibling `usage_metrics.providers` is separate.

This is configuration data passed to the existing Rust-owned selection behavior.
Presentation stays in the Python TUI; keep normalization, policy precedence,
eligibility, collection, and routing in their existing owners. No new selection
algorithm, Rust wire/API, provider-key matching rule, or backend fallback is needed.

## Implementation map

1. Update `_render_visible_windows`, `_prefix_cell_widths`, `_fallback_segment`, and
   related packing helpers in `src/sase/ace/tui/widgets/_provider_usage_indicator.py`
   for the grammar above. Keep prefix sizing linear in selected windows and rendering
   memory-only; share small spacing/count rules so sizing and output agree without
   repeatedly rendering every candidate. Update `_append_usage_content` in
   `src/sase/ace/tui/widgets/provider_disables_indicator.py` to use the owned margin.
2. In `_entry_fragment`, apply the zero style across percentage, internal space, and
   countdown. In `src/sase/ace/tui/widgets/_usage_indicator_palette.py`, rename
   `usage_zero_percent_style` to a value-oriented name such as `usage_zero_value_style`,
   update its documentation/exports/callers, and remove the value-color argument from
   `usage_divider_style` so dividers always use the neutral badge style. Remove any
   now-unused presentation bookkeeping introduced solely for coloring dividers; retain
   the existing capacity palette.
3. Apply the package and schema default changes above. Extend existing config tests to
   consume the real bundled default and merged user overrides, rather than testing a
   copied mapping or only mocking the Rust selection result.
4. Update `docs/ace.md`, the usage notation tooltip in `provider_disables_indicator.py`,
   and `docs/configuration.md`. Replace old claims about inter-provider pipes,
   final-window divider coloring, percentage-only zero highlighting, and Fable
   inheriting the generic threshold. Show the new grammar, describe actual-visible-count
   parentheses and the full-value highlight, and explain the package default and
   exact-key opt-out. Follow ACE's help-popup maintenance rule if any existing help text
   describes this behavior; no new ACE CLI option or keybinding is being introduced.

## Regression and visual verification

Update existing assertions instead of preserving obsolete separator and token-only
contracts. Add focused behavior coverage where the new grouping can fail:

- In `tests/test_provider_usage_indicator_presentation.py`, assert exact unstripped
  output for one provider/window, multiple providers with one window each, two and three
  windows within a provider, a hidden default, and partial overflow dropping from three
  windows to two to one. Assert one icon per visible provider, neutral normal-weight
  structural punctuation, and no inter-provider/trailing pipes.
- Sweep budgets from zero through full width for representative mixed groups, multi-cell
  icons, unknown-provider labels, and multi-digit hidden counts. Assert
  `cell_len <= budget`, complete values, balanced structural parentheses, correct hidden
  totals, exact-fit acceptance, and the longest fitting window prefix. Use an
  independent test reference that measures actual candidate text rather than copying the
  production width arithmetic. Pin outer/gap styles and the documented fallback ladder,
  including the tiny-budget count.
- In both themes, inspect effective Rich styles at every character of `0%` plus its
  countdown and internal space, and immediately outside that run. Cover named and
  rejected zeros, stale and unknown-age zeros, unknown reset time, clamped zero,
  positive `<1%`, and passed-reset zero. Keep existing minimum 4.5:1 contrast checks.
  Check punctuation beside an exhausted value and after a partially hidden group.
- In `tests/test_provider_disables_indicator_usage.py`, cover no routing pill, hard/soft
  disables, and priority in both themes. Assert the neutral margin, unchanged routing
  prefix styles, highlighted countdown, no red/background bleed, tiny-budget disclosure,
  and shrink/grow restoration. Check effective styles by offsets rather than assuming
  Rich emits a separate segment for `0%`.
- In `tests/llm_provider/test_usage_config.py` and the existing real-projection renderer
  integration coverage, use isolated configuration with the real bundled default and
  actual Rust projector. Show Fable at 100% and above the general threshold, show the
  weekly all-model window, and keep session/other extra windows subject to their normal
  threshold. Check explicit exact-key `never`, threshold boundary values, empty maps,
  unrelated provider overrides, indicator disable, and a snapshot with no Fable
  observation. Clear config caches where necessary. Run schema/default validity coverage
  as well.
- Update affected scenes in
  `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py` and
  `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py`,
  reusing `_provider_usage_indicator_fixtures.py`. Cover normal multi-provider gaps, a
  parenthesized Claude group with exhausted Fable, a healthy always-visible Fable,
  partial-group overflow, crowded narrow bars, and composition with routing pills.
  Ensure dark/light scenes cover parentheses, neutral pipes, and zero plus countdown
  together; existing single-window-only palette images are insufficient for this
  combined contract.

Run the focused presentation/composition/config tests and `tests/test_config_schema.py`
/ `tests/test_config_schema_validity.py`. Refresh only the affected usage PNG goldens
with `just test-visual` restricted to the two usage snapshot files and
`--sase-update-visual-snapshots`; visually inspect the resulting expected/actual/diff
images and rerun those files without update mode. Verify the first and last margins
against neighboring top-bar widgets, balanced parentheses, neutral punctuation, readable
highlighted countdowns, and stable narrow overflow.

Read the current `lint_and_test.md` reference memory and run `just check` before
implementation completion. Use `sase_monitor` for verification that becomes
long-running, as the skill/memory require. Reproduce and report any unrelated gate
failure through the prescribed existing-task workflow instead of treating a prior
conversation's failure as current evidence. Finish with `git diff --check` and a review
that only intended implementation, docs/config, tests, and goldens changed.

Completion requires all four requested behavior changes, real-default override coverage,
inspected stable dark/light snapshots, and an accurate report of the required
verification outcome.
