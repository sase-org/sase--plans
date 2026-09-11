---
tier: tale
title: Make usage window indicators readable at a glance
goal:
  Give each existing usage window a distinct, high-contrast presentation with a
  color-matched percentage and reset countdown, preserving all displayed content and
  layout capacity.
size: medium
proposed_by: bbugyi200.athena.0j0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.sase-zf.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-zf.land/README.md)
- **COMMITS:**
  - [89c8f4e](https://github.com/sase-org/sase--plans/commit/89c8f4ede58e4053a31e0a15cacb8f24c774ab17)
    — docs(plans): mark agents_query_unification done after landing epic sase-zf

# Make usage window indicators readable at a glance

## Outcome and scope

Make the ACE top-bar usage windows immediately legible in the dense header shown in
`~/tmp/screenshots/20260910_164611.png`. Each window should read as one visual unit:
provider and scope identify it, and a bold, consistently colored percentage/countdown
pair communicates its current state. Separate these units from the neighboring routing
control and from one another through background treatment and existing whitespace.

This is one `medium` tale: a bounded Python/Textual presentation change with focused
rendering regressions and visual review. One coding agent can implement and verify it.
There are no independent phases or shared backend changes.

Preserve the exact current plain text and terminal-cell width for identical inputs and
budgets, including spaces. Preserve provider icons, scope/specifier strings, percentage
rounding, countdown formatting, `~`, `?`, `↻`, `!`, `⚠`, provider ordering, tooltip
content, click targets, and overflow disclosure. In particular, do not change which
windows appear or interpret percentages differently. Keep the top bar one row high.
There are no new labels, separators, progress bars, padding cells, controls, settings,
keybindings, collection behavior, or changes to the Usage modal.

## Grounded diagnosis

The implementation is in:

- `src/sase/ace/tui/widgets/_provider_usage_indicator.py`: constructs `UsageBadge`
  values and packs complete badges into a cell budget.
- `src/sase/ace/tui/widgets/_usage_indicator_palette.py`: ten capacity buckets and
  neutral/warning/rejection styles for dark and light themes.
- `src/sase/ace/tui/widgets/provider_disables_indicator.py`: combines routing and usage,
  budgets the header, watches theme changes, and skips unchanged updates.
- `src/sase/ace/tui/widgets/_override_pill.py`: supplies routing-pill base styles.

Two presentation problems explain the screenshot:

1. `_entry_badge()` makes the percentage bold and capacity-colored but assigns the
   countdown the secondary neutral style. The two related values look unrelated.
2. `_append_usage_content()` copies the routing `Text` before appending usage. The
   routing text's base style includes an orange, yellow, or blue background, while usage
   specifies only foreground colors. Rich therefore paints usage on the routing
   background. A read-only rendering probe confirmed that a fresh `7%` is rendered as
   `bold #ff5f6d on #ffaf5f`, approximately 1.62:1 sRGB contrast. Icons, gaps, and
   secondary text can inherit routing attributes too.

The existing palette's contrast assumptions concern dark/light neutral surfaces, not
these routing accents. Existing tests cover labels, ordering, packing, and several
usage-only PNG scenes, but they do not pin the resolved styles of full usage badges
beside a routing pill.

## Visual specification

Use compact, flat badge backgrounds with the existing two-space gaps visible between
them. Apply background through Rich spans, without changing any characters. Keep the
current ten capacity colors and bucket boundaries; give both values in a window the same
resolved foreground color and bold weight.

| Role                                         | Dark theme                       | Light theme                      | Treatment                                                                  |
| -------------------------------------------- | -------------------------------- | -------------------------------- | -------------------------------------------------------------------------- |
| Badge surface                                | `#242830`                        | `#E0E0E0`                        | Explicit background over the complete badge, including its internal spaces |
| Existing inter-badge gaps                    | `#121212`                        | `#FAFAFA`                        | Explicit neutral background; retain the exact existing two spaces          |
| Provider text fallback and window specifier  | `#B8C0CC`                        | `#4B535F`                        | Normal weight, full opacity; retain provider emoji glyphs                  |
| Fresh percentage and countdown               | Existing dark bucket color       | Existing light bucket color      | Both bold, same foreground, on the badge surface                           |
| Stale/unknown-age or reset-passed value pair | `#B8C0CC`                        | `#4B535F`                        | Both bold and neutral; preserve all uncertainty characters                 |
| Collector/rejection marker                   | Existing warning/rejection color | Existing warning/rejection color | Bold, on the badge surface                                                 |
| Overflow disclosure                          | `#B8C0CC`                        | `#4B535F`                        | Bold text on the badge surface; existing gaps remain neutral               |

The dark background gives the existing capacity palette a minimum contrast of about
5.00:1; the light background gives about 4.66:1. The specified neutral text is about
8.06:1 on dark and 5.89:1 on light. These are calculated sRGB text/background ratios,
not a claim about emoji rendering or every terminal's color capabilities. Retain a
minimum 4.5:1 for all non-emoji text against its actual badge background.

The reading order stays exactly as it is today. For example:

```text
🎭 wk/fable 7% 1d8h  🎭 62% 3d4h  🤖 15% 2h9m
```

Here `7% 1d8h` is one bold red pair, `62% 3d4h` one bold green pair, and `15% 2h9m` one
bold coral pair. Each entire window sits on its own neutral rectangle, with the two
existing blank cells separating rectangles. Scope text stays readable but subordinate
through normal weight. This description is a style schematic; the formatter remains the
authority for actual labels and values.

For stale or unknown-age observations, match the countdown to the neutral percentage,
including the existing `~`. For a passed reset, match `?%` and `0h0m↻` in neutral. For a
fresh observation with an unknown reset time, color the existing countdown `?` with that
window's percentage color; the question mark still communicates uncertainty. Keep
collector-only badges as the existing icon and warning marker, with the same surface
isolation. Do not invent a value pair for them.

Keep routing controls in their established accent colors. The background transition at
the existing routing/usage boundary must make it clear where routing ends and usage
begins. Avoid blinking, gradients, extra borders, or animation. The unchanged numbers,
markers, weight, and grouping must remain understandable without distinguishing hues.

## Implementation

1. Extend `_usage_indicator_palette.py` with a small set of presentation helpers or
   constants for the badge surface, gap surface, and explicit base/secondary styles.
   Update its contrast documentation and the neutral colors above. Preserve the capacity
   bucket colors and warning/rejection semantics. Reuse the existing `dark` parameter
   and app theme watcher; do not introduce theme configuration.
2. In `_entry_badge()`, select the value color once using the existing freshness/reset
   decisions, then apply the same bold style to both percent and countdown tokens. Give
   each badge a complete explicit base style, including background and normal weight for
   secondary content. Prevent inherited dim/reverse attributes. Apply the same base
   treatment in `_collector_only_badge()` and preserve marker overrides.
3. In `_join_badges()` and `build_usage_indicator_segment()`, style the existing gaps
   and every fallback path explicitly. Preserve the packing algorithm, all existing
   spacing, and the `full -> whole badges plus +N -> usage N -> N -> ellipsis -> empty`
   ladder. No partial badge truncation or added width. A fallback must remain legible
   when routing is present, including tiny budgets.
4. In `_append_usage_content()`, compose routing and usage into a neutral new `Text`
   container and append both as styled segments, rather than using the routing pill as
   the composite's base style. Preserve routing's original styles exactly. Keep the
   routing-only return path unchanged. Ensure every usage cell, including icon fallbacks
   and whitespace, has the intended presentation after composition.
5. Ensure theme-only changes repaint every representation, including count-only and
   collector-only states. `_text_signature()` currently compares plain text and spans
   but omits `Text.style`; include the base style in its signature and update its type
   annotation so base-only changes cannot be skipped. Cover this with a focused
   regression. Keep unchanged refreshes coalesced and skipped.

Use the existing memory-only projection, refresh cadence, cell-budget calculation, click
routing, and tooltip building. Presentation belongs in this repo; the Rust projection
and Python formatting/collection modules are outside this change. There should be no
need to change `styles.tcss`, `_app_layout.py`, or `default_config.yml`. Do not move
usage into separate widgets or add work to timers, event handlers, or render paths
beyond constructing these small styles and text spans.

## Verification and visual review

Read `lint_and_test.md` and `tui_perf.md` through `sase memory read` before
implementing. Use the existing presentation and widget tests for content/behavior
preservation; extend them with regressions that test the actual observed defect rather
than merely checking helper return values.

- In `tests/test_provider_usage_indicator_presentation.py`, parameterize resolved
  percentage/countdown style equality across both themes, all ten capacity buckets,
  stale and unknown age, passed resets, and unknown reset times. Check bold weight and
  neutral uncertainty handling. Keep the existing exact text, ordering, and width
  expectations intact. Check the defined text colors, including neutral and markers,
  against the actual badge backgrounds with a small contrast assertion in tests; no
  runtime contrast computation is needed. Cover `<1%`, `0%`, and `100%` without altering
  the formatter's existing semantics.
- In `tests/test_provider_disables_indicator.py`, render a complete composite through
  Rich and inspect resolved segment styles. Cover hard disable, soft disable, provider
  priority, and no routing. Verify that the routing prefix retains its original styles,
  that usage/gaps/fallbacks never inherit routing backgrounds, and that each
  percentage/countdown pair agrees after composition. Include collector-only usage.
- Add focused mounted-widget coverage for dark/light theme switching with identical
  plain text, including a count-only fallback. Check that unchanged repeated refreshes
  still avoid updates. Exercise a wide/narrow/wide resize and retain whole-badge
  packing, tooltip access to hidden windows, and the existing click target. Reuse
  `tests/ace/tui/test_top_bar_order.py` for header order and bounds coverage.

Use the existing PNG fixtures and snapshot files:

- `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py`
- `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py`
- `tests/ace/tui/visual/_provider_usage_indicator_fixtures.py`
- `tests/ace/tui/visual/snapshots/png/top_bar_usage_*.png` and
  `top_bar_compact_usage_badges_120x24.png`

Add a deterministic, sufficiently wide scene with an orange disable pill and full usage
badges visible together, modeled on the supplied screenshot. Include low capacity, stale
data, and a healthy window, and capture both dark and light themes. Existing crowded
routing scenes disclose only a count and cannot catch the original full-badge problem.
Retain the current 60/80/120/140/160/240-column scenarios as applicable; do not multiply
every scene by every routing state.

Run the focused visual suite with:

```bash
just test-visual tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator.py tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py
```

Inspect actual/expected/diff images in `.pytest_cache/sase-visual/`. Evaluate the full
header at normal reading size: pair association, clearly separated windows, readable
scope text, retained warning markers, and a clean boundary after routing. Inspect dark
and light palette captures and narrow disclosure, not just a zoomed sample. Adjust only
the intended presentation if needed, preserving the contrast and content contracts.
Accept reviewed intentional goldens with the same command plus
`--sase-update-visual-snapshots`, then rerun without that flag to prove they pass.

Run `just check` after implementation. It provides the required whole-repo lint gates
and scoped nonvisual tests; confirm the affected presentation, widget, and top-bar tests
are selected, and run any missing focused tests explicitly. Keep visual testing explicit
because `just check` excludes it. Follow the memory's escalation rules if broader checks
become necessary; use `/sase_monitor` for a long-running verification command and always
for `just check-full`. Do not broaden or repeat passing checks without a new failure,
change, or unresolved concern.

## Acceptance

The change is complete when each percentage and reset countdown has identical effective
foreground color and bold emphasis; usage remains legible beside every routing style;
neighboring windows are visibly separated at the same character positions as before; and
dark/light transitions repaint full and compact disclosure. All existing contents,
widths, ordering, uncertainty markers, tooltip/click behavior, and header geometry
remain unchanged. The focused visual suite and required `just check` pass, and the
implementation report identifies the reviewed screenshots and verification results.
