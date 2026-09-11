---
tier: tale
title: Group and polish the TUI usage window indicators
goal:
  Make ACE's top-right usage display compact and readable with one icon per provider,
  default-first windows, colored names and pipe separators, and no caution triangles,
  while preserving configurable selection and useful overflow.
size: medium
proposed_by: bbugyi200.athena.0j8
create_time: 2026-09-11 07:50:56
status: wip
---

# Group and polish the TUI usage window indicators

## Outcome and scope

Show each provider's selected usage windows together, with its default first and
additional windows named and colored as cohesive units. Remove the top-bar caution
triangle. Preserve independent readings, existing configurable thresholds, and access to
the full details when the terminal cannot show every window.

This is one bounded presentation change for a single follow-up implementation agent: a
`tale` of `medium` size. Grouping, Rich styling, layout, and the resulting tests belong
in the Python TUI. Selection, window classification, freshness, attention, and
configuration policy already come from the Rust core; consume that projection without
implementing a second policy engine or changing its wire API.

Implement the entire visual replacement together. It requires no configuration
migration, feature flag, or compatibility branch. Provider routing and disable-pill
behavior, collection, the Usage modal, and model-picker indicators are outside this
change. Do not suppress a provider's readings merely because it is disabled.

## Current implementation

- `src/sase/ace/tui/widgets/_provider_usage_indicator.py` builds one `UsageBadge` per
  selected window, repeating provider icons and sorting windows by attention. It also
  creates standalone collector-failure badges. Its packer drops whole badges, then falls
  back to `usage N`, `N`, or `…`.
- `_usage_indicator_format.py` in that directory formats names such as `wk/fable` and
  `5h/all`, percentages, and reset countdowns. `_usage_indicator_palette.py` supplies
  the existing ten capacity colors and explicit dark/light surfaces.
- `provider_disables_indicator.py` composes routing pills and usage, budgets usage to
  available space capped at half the top bar, handles resize/theme changes, and supplies
  a combined tooltip and click action.
- `src/sase/llm_provider/usage/peek.py::cached_usage_indicator_projection` projects the
  cached observation through `provider_usage_project_indicator`. Entries carry
  `provider`, `window_key`, `weekly_all`, `scope`, `period`, numeric/reset state,
  `display_attention`, `collector_problem`, and effective policy details.
- `src/sase/default_config.yml` already declares the needed indicator settings;
  `docs/configuration.md` documents their precedence and strict threshold boundary.

## Visual contract

### Grouping and names

The normal wide layout is:

```text
🎭 62% 3d4h | fable 7% 1d8h | 5h 18% 2h9m | 🤖 81% 5d2h
```

The provider icon appears once. The default is the existing positively classified
`weekly_all` entry, not a guessed window key or the first item received. It renders as
`<N>% <duration>`; every other selected window renders as `<name> <N>% <duration>`. Here
`duration` always means time until reset, not the window's period. Every percentage
remains the independent percentage remaining.

Order provider groups by their highest selected `display_attention` rank, then provider
ID, retaining the current attention-rank table. Inside each group put the default first,
then extras by descending attention and window key. Reordering the input must not change
the display. This preserves attention priority while giving each provider a predictable
default-first structure.

Derive compact names from the existing structured period/scope formatter and model alias
mapping; omit the redundant weekly period and all-model scope components:

| Existing identity                     | New name         |
| ------------------------------------- | ---------------- |
| Weekly Fable, `wk/fable`              | `fable`          |
| All-model session, `5h/all`           | `5h`             |
| All-model monthly, `mo/all`           | `mo`             |
| Five-hour Fable                       | `5h/fable`       |
| Unknown Fast bucket, `Fast/5h/scope?` | `Fast/5h/scope?` |

Retain useful family/model distinctions, unknown-scope markers, and the current alias
collision protection. Generate names by components, not by blindly removing substrings
from vendor labels. If simplification yields nothing, use the full specifier, then
`window_key`, then `?`. Distinct selected windows with identical compact names must
receive stable key suffixes, e.g. `fable [<window_key>]`; compute that disambiguation
before width packing so labels stay stable across resize. Render provider-supplied text
literally with Rich `Text`, normalize embedded whitespace/control characters, and
replace literal `|` in names with `/` so names cannot introduce extra structural
dividers.

If policy hides the default, show the first selected extra immediately after the icon
with its name: `🎭 fable 7% 1d8h`. Do not resurrect a hidden default, invent a
percentage, omit that extra's name, or put a divider before the first window. Providers
with only monthly/session/unknown-scope windows follow this same rule. If more than one
selected record is classified `weekly_all`, choose the lowest window key as the unnamed
anchor and render the others with explicit names/keys; retain every reading rather than
collapsing allowances into one value.

### Colors, surfaces, and pipes

Treat each window's name, percentage, and countdown as one bold value-colored unit. Use
the existing ten-bucket palette for fresh numeric readings. Stale or unknown-age
observations and passed resets use the existing neutral value color, including the name.
Keep the provider icon at its existing neutral base style.

Use exactly one ASCII `|` between adjacent visible windows, including at provider
boundaries, with one space on either side. Thus K visible windows have K-1 pipes; there
is no leading or terminal pipe and no pipe before a count-only disclosure. This is the
design interpretation of one separator per window boundary.

A divider belongs to the provider on its left. Its foreground color is that provider's
**last actually visible window's value color**. Apply that rule to every divider
belonging to the provider, including dividers inside its group and the divider before
the next provider. It is deliberately a final-window rule, not the next window's color
or the provider's most severe color. For the wide example above, all three pipes are
Claude's 18% coral color. If only Claude's default and Fable remain visible, its
internal pipe is Fable's 7% red. If the last visible window is stale or reset-passed,
the dividers are neutral. Use normal weight for pipes so values keep visual emphasis.

Keep the current explicit badge surfaces (`#242830` dark, `#E0E0E0` light). Internal `|`
separators share the provider's continuous badge surface. At a provider boundary, paint
` |` on the preceding group's surface and the following single space on the app's
existing gap surface before the next icon. This keeps the divider's capacity color on
the surface for which contrast was tested and still separates provider rectangles.
Leading space and the gap before overflow use the app gap style. Preserve explicit
`not dim`/`not reverse` styling and prevent routing-pill colors from bleeding into
usage.

### Status and removal of triangles

Remove `⚠` from rendered usage content, including collector-only badges. A provider with
no selected windows contributes no empty icon, placeholder, divider, or overflow count.
Remove obsolete collector-only rendering helpers and warning palette symbols when the
caller search confirms they have no remaining use.

Preserve collector-failure prose in selected windows' tooltips; detailed collector
health remains available through Providers · Usage. There is no new top-bar replacement
glyph. Keep the vendor rejection `!`, placed immediately before the affected window's
percentage (`fable ! 7% 1d8h`), using its current rejection style. A simultaneous
collector failure must no longer mask rejection.

Preserve exact existing percent/countdown behavior: `0%`, `<1%`, floored/clamped
percentages, `100%`, stale `~`, reset-passed `?% 0h0m↻`, and unknown-reset `?`.

## Selection and configuration contract

Only group entries already selected by the core projection. Never add, average,
deduplicate by display name, or independently threshold the usage percentages.

Keep `llm_provider.usage_metrics.indicator` unchanged:

- `weekly_all: always` keeps the default visible by default.
- `default: {below_remaining_percent: 20}` selects extras strictly below 20% remaining,
  using the unrounded number. Exactly 20% remains hidden by default.
- `always`, `never`, and custom thresholds continue working with precedence: exact
  provider/window key, provider default, classified `weekly_all`, global default.
- Indicator/collection opt-outs and config-token refresh behavior remain intact. Config
  keys remain exact window keys, never the shortened display names.

Retain shipped config values and examples. Update comments only if needed to explain the
new presentation; do not introduce a second threshold or rename keys.

## Width packing and interaction

Store ordered complete window fragments within each provider group. Do not flatten the
group into an indivisible rendered badge or recover fragments by parsing text. Each
fragment needs its rendered value style, width, identity, and tooltip detail.

Use this deterministic packing rule over the provider-group/window order above:

1. Show all windows if they fit the existing local terminal-cell budget.
2. Otherwise show the longest prefix of complete windows that fits with a neutral `  +N`
   disclosure; N counts **selected windows hidden by width**, including extras hidden
   within the final visible provider. The group may end after any complete window. Emit
   each visible provider icon once and never skip its selected default in order to fit
   an extra. Do not skip an earlier oversized window to backfill later providers.
3. If no complete-window prefix plus disclosure fits, use the existing fallbacks
   `usage N`, `N`, `…`, then empty text, choosing the richest one that fits. Preserve
   `leading_space` behavior and handle zero budgets and wide emoji by cell width.

The longest-prefix approach is intentional: higher-attention provider groups and their
extras stay ahead of later groups. It also permits a provider with many extras to keep
its default and some extras instead of disappearing wholesale. Threshold-hidden windows
and collector-only failures are not counted. The neutral overflow token is not a usage
window and does not participate in pipe coloring.

For the four-window wide example, a budget that fits only the first two windows and
disclosure produces:

```text
🎭 62% 3d4h | fable 7% 1d8h  +2
```

The pipe is now Fable red; `+2` is neutral and counts Claude's hidden session plus
Codex's hidden default. Both remain in the tooltip. A wider resize restores both windows
and changes Claude's pipes back to the session's coral color. Group order is based on
all selected entries, so width changes alone never reorder providers.

Resolve each visible group's final-window divider style after packing. Widths do not
depend on color; use cumulative cell widths to choose the prefix without repeated
sorting or rebuilding every candidate. Resizing wider restores omitted windows and
correct divider colors from the retained full group data.

Keep full selected-window tooltips, including overflowed windows, in group/window order.
Preserve identity, exact key, precise remaining value, scope, effective policy/source,
freshness, and reset information. Update the notation legend for compact names and
pipes, retaining `+N counts hidden windows`. Clicking usage or overflow continues
opening Providers · Usage; the current action is for the whole widget, so no new
per-window hit testing is needed. With no selected windows, retain the existing
routing-widget action. Keep command-palette access at zero display space.

## Implementation sequence

1. Refactor `_usage_indicator_format.py` to provide the compact name construction above,
   reusing existing period/scope helpers. Keep percentage and countdown formatting
   unchanged. Remove old formatter exports only when orphaned.
2. Refactor `_provider_usage_indicator.py` into provider grouping plus complete window
   fragments and the deterministic packing/rendering above. Keep this pure presentation;
   use a focused sibling helper module if needed for file-size limits. Remove standalone
   collector badges, and remove the `providers` input if it no longer has a consumer.
   Retain per-entry tooltip generation.
3. Adjust `_usage_indicator_palette.py` for divider/base-style needs and remove newly
   unused warning/secondary helpers and exports. Reuse both existing capacity palettes
   rather than inventing new colors.
4. Update `provider_disables_indicator.py` call sites, cached types, and tooltip
   notation for the new group representation. Preserve the existing budget cap, routing
   composition, click behavior, unchanged-content signature shortcut, theme repaint,
   coalesced resize reflow, and off-thread usage-cache load.
5. Update rendering and widget tests plus their constructed fixtures to use the
   resulting representation. Add behavioral coverage below; do not retain mock
   collector-only triangle badges that production no longer produces.
6. Update the ACE usage section in `docs/ace.md` with the preview, ordering, compact
   names, final-visible-window divider coloring, and partial-group overflow. Correct
   `docs/configuration.md`'s `indicator.enabled` description that mentions
   collector-health usage marks. Keep its policy semantics and `docs/llms.md`'s strict
   threshold/config documentation aligned. Regenerate relevant PNG goldens.

All new rendering work operates on the existing memory-only projection. Add no disk
reads, network calls, subprocesses, usage-store locks, timers, or refresh paths on the
UI thread. Sorting is O(W log W), grouping/width accounting O(W), and only the chosen
visible prefix needs composition. Reuse cached model aliases; do not introduce
configuration reads per fragment or width candidate.

## Acceptance and verification

Extend `tests/test_provider_usage_indicator_presentation.py` and, where useful, split
focused name/packing coverage into a sibling test module. Cover:

- One provider with default, Fable, and session, plus a second provider: exact
  default-first text, one icon per provider, deterministic attention/key order, K-1
  pipes, no triangles, and no missing or duplicated readings.
- Every visible name matches its own percent/countdown foreground and weight in both
  themes. Assert resolved styles by character offsets/ranges rather than depending on
  Rich's coalescing of equal-style runs.
- Three differently colored windows expose the final-window divider rule, including the
  boundary to another provider; repeat with a neutral last window and after overflow
  hides the former last window. Check separator backgrounds as well.
- Hidden/absent defaults, a provider with no selected entries, nonweekly and unknown
  providers, unknown scope, multiple default-classified entries, colliding names, long
  names, and literal/control characters in labels.
- Numeric/countdown edge cases and stale/reset/rejected states, including rejection plus
  collector failure. Collector-only state produces no badge or overflow count; selected
  failing entries retain failure details in their tooltip.
- Budget sweeps from zero through full width, exact fit and one cell short, wide emoji,
  leading-space variants, count digit boundaries, a partial final group, and
  narrow-to-wide restoration. No output exceeds its cell budget, truncates a window,
  leaves an orphan icon/divider, or miscounts hidden windows.

Extend `tests/test_provider_disables_indicator_usage.py` and
`tests/test_provider_disables_indicator_widget.py` for routing composition, overflow
clicks, complete tooltip disclosure, resize/theme repaint, and unchanged content
avoiding redundant updates. Update `tests/_provider_disables_indicator_helpers.py` as
needed. Preserve routing style assertions and default/off/soft/priority behavior.

Add an integration regression using synthetic snapshots through the real
`provider_usage_project_indicator` binding and then the renderer. Prove an extra at
19.99% appears, one at 20% does not, and an exact-window threshold override changes that
result. Cover `always`/`never`, hidden defaults, and disabled indicator output without
mocking the selection algorithm. Reuse current usage fixture builders, not live provider
probes. Keep the cache/config regressions in `tests/llm_provider/test_usage_peek.py` and
`test_usage_config.py` passing.

Use the existing visual scenes in
`tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py`,
`test_ace_png_snapshots_provider_usage_indicator_states.py`, and
`_provider_usage_indicator_fixtures.py`. Update affected goldens and add a grouped
multi-provider scene in both themes plus a partial-group overflow scene. Include
140/160-column readable layouts, crowded 60/80-column layouts, and retained status
states. Adapt the ten-decile palette scene so grouping does not silently hide the colors
it is intended to verify; keep all ten visible in both themes.

Before implementation ends, read `lint_and_test.md` through `/sase_memory_read` and run
the required `just check` (use `just install` first if dependencies are stale). Run
focused tests while developing, then `just test-visual` for the dedicated snapshot
suite. Accept intended changes with `--sase-update-visual-snapshots`, inspect
actual/expected/diff PNGs under `.pytest_cache/sase-visual/`, and rerun the affected
visual tests without the update option. Visually confirm readable names, continuous
provider surfaces, uncluttered dividers, and routing coexistence.

Use `/sase_monitor` for long verification runs; run `just check-full` only when the
project's verification guidance calls for it. Consult `symvision.md` through
`/sase_memory_read` before fixing any Symvision failures. Completion requires the
requested grammar and configuration integration checks, clean required validation, and
inspected intentional visual changes.
