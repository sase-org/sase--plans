---
tier: tale
title: A calm, persistent usage-window header
goal:
  Make provider usage easier to scan in the existing header, with independent controls,
  honest overflow, and preserved usage semantics.
size: medium
proposed_by: bbugyi200.athena.0jo
create_time: 2026-09-11 14:31:55
status: wip
---

# A calm, persistent usage-window header

## Outcome and scope

Place usage in the existing application title row, with the title fixed on the left and
usage anchored to the right. Keep navigation, routing, project, and notification
controls on the second row. Within a provider, replace parentheses and pipes with quiet
middle dots. Retain the provider surfaces, icons, values, colors, warning treatments,
selection policies, and attention ordering.

This is one coherent presentation change for one coding agent: a `tale` of size
`medium`. The widget extraction, header layout, interaction migration, and visual
verification should land together. Shared backend behavior stays in Rust; this work uses
the existing core projection without changing its schema or logic. No feature flag or
new configuration is needed for the complete replacement.

The intended appearance is:

```text
[icon] sase ace (version)         🎭 62% 3d4h · fable 100% 1d8h  🤖 45% 2d5h
[tabs]                           [model/routing controls] [project] [alerts]
```

The square-bracketed items above describe existing content, not new literal labels. The
application retains its existing two rows of chrome in its ordinary collapsed header
state. The title's left edge and usage's right edge stay fixed when only usage text
changes. Individual values within usage can move as lengths or attention order change;
this design does not promise fixed numeric columns.

## Research and current-code evidence

Read the report through the audited artifact command before implementation:

```bash
sase artifact read file:explicit:f7c2085d0f7af40d82811b47 "Implement the usage-window header design of record"
```

This is the immutable published copy of
`research:202609/usage_window_header_design/usage_window_header_design.md`. The
canonical research reference did not resolve in the planning environment; the published
copy was successfully read and is linked above. The accompanying infographic was also
read as `file:default:56c578dfc5830a8e2f70b7dc`. It illustrates placement, not
authoritative colors or pixel geometry. All implementation paths below are relative to
the SASE checkout.

Relevant existing behavior:

- `src/sase/ace/tui/_app_layout.py` composes a standard `Header()` above `#top-bar`.
  `ProviderDisablesIndicator` currently owns both routing and usage on the latter.
- `widgets/_provider_usage_indicator.py` already constructs independent window
  fragments, provider groups, attention order, complete-prefix packing, and full tooltip
  facts. `+N` counts hidden **windows**, not providers.
- `widgets/provider_disables_indicator.py` owns the memory-only projection read,
  off-thread cache reload, 30-second refresh, theme repaint, coalesced resize reflow,
  and content-signature short circuit. Its click handler currently opens Usage whenever
  selected usage exists, even when the click was on routing.
- `actions/base.py::action_open_provider_usage` gets its default provider from
  `#provider-disables-indicator`. `_refresh_launch_indicators` in
  `actions/agent_workflow/_leader_mode.py` also refreshes the combined widget.
- The installed Textual Header uses an eight-cell `HeaderIcon`, a centered
  `HeaderTitle`, and a ten-cell `HeaderClockSpace` even when no clock is shown. Its
  title watches app and screen titles/subtitles; clicking unused header space toggles
  the existing tall-header class. Preserve these behaviors deliberately while replacing
  the unused clock slot and title alignment.
- `tests/ace/tui/test_app_title.py` covers the asynchronous switch from a short release
  title to a longer development version. Header budgeting must respond to that switch,
  not just terminal resize.

An in-memory planning check using the current real fragments reproduced the report:

| Visible windows in the report's four-window example | Current cells | New cells |
| --------------------------------------------------- | ------------: | --------: |
| All four                                            |            61 |        57 |
| Three plus `+1`                                     |            52 |        48 |
| Two plus `+2`                                       |            39 |        35 |
| One plus `+3`                                       |            19 |        17 |

These are usage-segment budgets, including margins, not terminal widths. The report's
illustrative three-cell icon reservation is not the current eight-cell HeaderIcon.
Measure actual geometry; do not promise four windows at every 80-column terminal.

## Design contract

### Placement and title priority

Introduce an ACE-specific Header subclass in `widgets/usage_header.py`, composed in
place of `Header()` by `_app_layout.py`. Reuse Textual's existing icon/title behavior,
including command-palette access, formatted title/subtitle, reactive updates, and
ordinary header expansion. Keep any interaction with Textual's internal header
components isolated in this adapter and covered by integration tests.

Keep the current icon area and hit target. Left-align the title immediately after it.
Replace the unused clock placeholder with a separate `ProviderUsageIndicator`
(`widgets/provider_usage_indicator.py`, id `provider-usage-indicator`). ACE does not
currently enable a clock; do not leave its ten-cell spacer in the new composition. Scope
header CSS to this ACE header so other headers and modal titles are unaffected.

Use the header's inner terminal-cell width, the icon's actual outer width, and the
natural cell width of the fully formatted title/subtitle to calculate the budget:

```text
usage_budget = max(0, header_inner_width - icon_outer_width - title_natural_width)
```

Account for any real margins/borders once in the geometry. Usage's own leading and
trailing cells are included in this budget. The complete title has priority: if it fits,
never truncate it merely to show more usage. If the title alone exceeds the available
row, give usage zero cells and let the existing single-line title ellipsis handle the
title. Keep the full title available through a title tooltip when clipped.

Use a flexible gap and a tight, right-aligned usage region; do not make the unused space
between title and usage a giant Usage click target. The usage widget's width comes from
its final rendered cell width, while its budget comes from the independent header
measurements. Do not measure a previously clipped title or use the usage widget's
current width to compute its next budget. This avoids resize feedback loops.

Reflow after mount, terminal resize, title/subtitle changes (including app/screen
overrides and the late version update), and usage content changes. Coalesce requests and
apply only changed widths/content. An initially unknown or zero geometry must not paint
an unlimited usage string across the title; keep usage empty until a valid budget is
available. The widget remains mounted and can recover as space returns. There is no
half-row cap, wrapping, extra row, or usage-based title recentering.

### A single, quiet text grammar

For each provider:

```text
<icon> <existing-window-fragment> · <existing-window-fragment> ...
```

- Render the provider icon once, followed by one space.
- Use `·` between that provider's visible windows, including a neutral, normal-weight
  middle dot on the existing provider badge surface.
- Remove structural parentheses. Separate providers with the existing two spaces on the
  existing gap surface; do not add a separator between providers.
- Reduce outer padding from two cells to one on each side, owned by the segment exactly
  once. Add no duplicate widget padding.
- Preserve existing `Text` fragments and styles, including compact names and collision
  disambiguation. Do not rebuild styled values from plain strings.
- Use natural token widths. `10h10m` and `23h59m` require six cells, and larger
  durations can require more. Use Rich/Textual cell measurement for icons and Unicode
  labels, not Python `len()` or a variation-selector width override.

Examples, omitting the one-cell outer margins for readability:

```text
🎭 62% 3d4h
🎭 62% 3d4h · fable 0% 1d8h · 5h 18% 2h9m
🛰️ ! 4% 3d4h  🤖 ?% 0h0m↻  🎭 88% ?
🎭 62% 3d4h · fable 100% 1d8h  +2
```

Preserve the following semantic distinctions as they work today:

| State                                     | Required treatment                                                                                |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Fresh positive capacity                   | Existing ten-bucket color and bold value/name styling                                             |
| Exact zero                                | Existing inverted `0% <countdown>` run, with surrounding names/markers on their existing surfaces |
| Positive below one percent                | `<1%`, never rounded to exhausted zero                                                            |
| Stale or unknown-age observation          | Existing neutral numeric text and explicit freshness explanation in the tooltip                   |
| Reset passed                              | `?% 0h0m↻`; last observation remains in the tooltip                                               |
| Unknown reset                             | `?` countdown; do not invent a reset time                                                         |
| Vendor rejection                          | Existing `!` marker and its styling                                                               |
| Collector problem                         | Existing tooltip prose; no new header warning icon or fabricated window                           |
| No selected windows or indicator disabled | Empty usage region; routing remains available                                                     |

Do not combine overlapping percentages, infer routing from zero capacity, suppress a
provider's windows when disabled, add stale glyphs, or change collection thresholds. The
old `sase_plan_usage_indicator_merged_badges.md` in the checkout describes earlier work
and is not the design contract for this change.

### Honest overflow

Keep the existing provider ordering by highest attention, then provider name. Within a
provider, retain the weekly-all anchor followed by extras in attention/key order. Keep
the longest complete prefix that fits, reserving the actual `+N` width before admitting
a prefix. The final provider can be partially visible. Never leave a dangling dot,
orphan icon, partial name, split percentage/countdown, or clipped `+N`. `N` equals
selected windows minus rendered windows.

Update both rendering and prefix-width calculation together: remove the conditional
two-cell parentheses overhead and use one-cell outer margins. Preserve the two-space gap
before overflow. When no window plus disclosure fits, use the existing fallback order
with reduced normal margins: `usage N`, `N`, `…`, then empty at zero budget. Retain the
existing relaxation of padding for tiny counts; do not reserve margins that make the
one-cell fallback unreachable. Use exact hidden-count digit widths, including the
transition from 9 to 10. Every nonempty result must fit its budget.

### Interactions and information access

The usage region is one click target. It opens Providers · Usage at the current
attention-leading provider, including when only a count or ellipsis is visible. Stop the
actual Textual click event from bubbling into Header's expansion handler. Clicking the
routing pill always opens Config > Launch, independently of usage. Clicking the header
icon continues to open the command palette. Unused header space keeps the existing
expansion behavior; clicking usage must not add header rows.

Move the usage tooltip and notation legend to the usage widget. Preserve the full facts
for every selected window, including hidden windows, exact key, scope, precise remaining
percentage, reset timestamp, freshness, effective policy, and collector problem prose.
Update the legend to describe dots and remove its parentheses/pipe explanation. Keep
routing's tooltip exclusively about routing and its launch action.

Update `action_open_provider_usage` to read the new widget when no provider argument was
supplied. Explicit arguments continue to win. The command palette and Launch Control's
existing `u` route must work when usage is empty, hidden by config, or has zero layout
space. Do not add a global binding to the conditional footer.

## Implementation sequence

1. Extract the usage lifecycle from `widgets/provider_disables_indicator.py` into
   `widgets/provider_usage_indicator.py`. Move projection/group state, provider
   selection, usage tooltip assembly, theme handling, usage-cache worker tracking,
   30-second display refresh, and usage content signature/reflow logic. Keep one owner
   of usage reloads, not two polling widgets. Preserve routing expiration updates and
   routing tooltip/rendering in `ProviderDisablesIndicator`. Remove its usage
   arguments/imports and obsolete top-bar sibling-budget helpers. Share a small
   presentation helper only if both widgets still need it.
2. Change the grammar and matching width accounting in
   `widgets/_provider_usage_indicator.py`. Keep `_usage_indicator_format.py`'s
   value/name rules and `_usage_indicator_palette.py`'s color assignments intact. Remove
   only structural code made obsolete by removing parentheses.
3. Add the scoped header adapter and layout rules in `widgets/usage_header.py` and
   `styles.tcss`; export widgets through `widgets/__init__.py` where needed. Mount it in
   `_app_layout.py`. Keep the existing order of `#top-bar` children and the routing
   widget id; the new usage id is a descendant of the header only.
4. Update `actions/base.py` and audit every use of `usage_open_provider`,
   `ProviderDisablesIndicator`, and `provider-disables-indicator` for assumptions about
   usage. Extend `_refresh_launch_indicators` to refresh the new usage widget as well,
   preserving prompt feedback after settings/eligibility changes. Retain the existing
   memory-cache/worker path for that refresh. Migrate fixtures and mocks that currently
   plant usage state on the routing widget.
5. Update `docs/ace.md` with header placement, sample text, independent click actions,
   overflow meaning, title priority, and preserved exceptional states. Review
   `docs/configuration.md` and relevant help text for stale placement or action
   descriptions. No config/keymap defaults change; if implementation finds that a
   configuration or keymap change is necessary, update `default_config.yml` and the help
   popup together instead of silently inventing a new option.
6. Complete the behavioral and visual verification below, inspect the resulting images,
   and revise any layout defect before accepting goldens. Ship the entire change
   together, with no retained alternate rendering mode.

## Performance constraints

Read `tui_perf.md` and `lint_and_test.md` through `sase memory read` before coding. All
new layout/render callbacks operate on in-memory state and geometry. Preserve
`cached_usage_indicator_projection`, the existing time-gated change-token check, and
off-thread `refresh_usage_peek_cache`; do not add disk reads, network probes,
subprocesses, store locks, or asynchronous waits to the serial UI message pump.

Keep the 30-second usage cadence and one in-flight reload guard, released on success,
error, or cancellation. Use a distinct worker group for the new widget and guard against
updates after unmount. Reflow must not schedule collection or rebuild agent lists. Keep
style-aware content signatures so an unchanged tick performs no Static update, while a
theme change repaints even when text is identical. Coalesce geometry changes and verify
that they settle rather than continually scheduling callbacks. Formatting/packing should
remain linear in selected windows aside from the existing sort; do not repeatedly render
every candidate prefix just to measure it.

## Verification and acceptance

### Behavioral tests

Adapt the existing presentation suites, keeping their real-core projection coverage:

- `tests/test_provider_usage_indicator_presentation.py`,
  `tests/test_provider_usage_indicator_presentation_layout.py`, and
  `tests/test_provider_usage_indicator_presentation_style.py`: new separators and
  margins; unchanged fragment styles in both themes; all state rows above; missing
  default window, duplicate names, long labels/durations, Unicode and fallback provider
  names. Check each budget from zero through the full rendered width, exact-fit
  boundaries and one-cell neighbors, and multi-digit overflow. Retain the existing
  threshold/override integration with the Rust projection.
- Turn usage-specific cases from `tests/test_provider_disables_indicator_usage.py` and
  `tests/test_provider_disables_indicator_widget.py` into tests of the new
  widget/header. Keep routing-only coverage in the existing routing suites. Add mounted
  tests for initial cache rendering, late worker results, failed/cancelled reload
  recovery, unchanged ticks, theme-only repaint, and unmount cleanup.
- Add focused header integration coverage, e.g. `tests/ace/tui/test_usage_header.py`:
  real measured widths at 60, 80, 120, 140, 160, and 240 columns, short and long titles,
  app/screen subtitles, late version changes, no usage, zero space, and narrow-to-wide
  restoration. Assert no overlap or horizontal scroll, one-line content, stable title
  origin and right edge, and no retained clock placeholder. Test an extremely narrow row
  for graceful title clipping without a layout loop.
- Using the same mounted app, freeze all non-usage state and change usage from `100%` to
  `99%`, `10h10m` to `10h9m`, one to several windows, empty to nonempty, and different
  attention order. Assert every control-row sibling's region is unchanged after layout
  settles. Assert the normal header/top-bar height stays two rows through these changes.
- Send real pilot clicks to a window, `+N`, `usage N`, bare count, and ellipsis; assert
  one Usage opening, correct initial provider, and no header expansion. Also cover
  routing with concurrent usage, palette icon, explicit provider arguments, and
  command-palette entry at zero space. Do not rely solely on direct `on_click()` calls,
  which miss event bubbling.
- Update `tests/test_models_panel_leader_mode.py`,
  `tests/ace/tui/test_top_bar_order.py`, and `tests/ace/tui/test_app_title.py` as needed
  to pin the new refresh owner, unchanged control ordering, and reactive title behavior.
  Keep tests deterministic with frozen clocks and cache fixtures.

### Visual review

Migrate `tests/ace/tui/visual/_provider_usage_indicator_fixtures.py` and both
`test_ace_png_snapshots_provider_usage_indicator*.py` modules to the new widget. Avoid
fixture shortcuts that inject combined routing/usage text or bypass the real header
budget. Render representative cases at all six widths above in both themes. Cover a
three-window provider, multiple providers, partial overflow, count-only and zero-space
layouts, long/changed titles, each exceptional state, and usage beside active routing
controls on the separate row.

Compare the current committed screenshots with the proposed layout using matching data.
During visual review, also make a temporary whitespace-only separator preview from the
same fragments; keep the dot design unless the comparison reveals a concrete attribution
or legibility defect. This preview is review material, not a runtime option. Verify
provider ownership, independent window boundaries, percentage and reset readability,
light/dark contrast, and the exact extent of the zero alarm. Do not copy the
infographic's illustrative provider colors.

Header alignment changes will intentionally affect other full-app PNG goldens too.
Review actual/expected/diff images under `.pytest_cache/sase-visual/`; accept only
intended header/control-row changes and explain any change below those rows. Use the
pinned font/rendering setup. Inspect actual terminal behavior for provider emoji, middle
dots, and countdown transitions on available supported terminals; record any terminal
unavailable for manual checking rather than claiming coverage.

Run `just check` for the implementation. Run the dedicated `just test-visual` lane,
using `--sase-update-visual-snapshots` only for reviewed intentional changes, and rerun
it clean after accepting goldens. Use `sase_monitor` for long checks, as the
verification memory requires. Run `just check-full` through that skill if scoped
verification escalates or the landing policy requires it. Do not bypass suite gates.

The change is complete when the plan's grammar and geometry hold across the tested
matrix, the control row stays stationary during usage-only changes, every exceptional
state and hidden window remains inspectable, interactions reach the correct view,
refresh stays responsive, and required behavioral/visual checks pass. No claim of faster
human glance time is required without a user study; the concrete acceptance criteria are
readable grouping, preserved meaning, correct fit, and stable controls.
