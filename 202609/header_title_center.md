---
tier: tale
title: Center the ACE header title on screen and emphasize exhausted window names
goal:
  The ACE header title stays centered on the terminal's horizontal center at every width
  regardless of usage-cluster content, and an exact 0% usage window renders its compact
  name inside the same inverted exhausted run as its percentage and countdown.
size: medium
proposed_by: bbugyi200.athena.0jq.f0
create_time: 2026-09-12 05:22:59
status: wip
---

# Plan: Screen-centered header title and full-run 0% emphasis

Two independent refinements to the ACE application header that landed in
`feat(ace): show usage windows in header`. Both are presentation-only Textual/Rich
changes; neither belongs behind the Rust core boundary, and neither needs a feature flag
or a new config option.

## 1. Current behavior

`UsageHeader` (`src/sase/ace/tui/widgets/usage_header.py`) composes Textual's
`HeaderIcon` (docked left, width 8), `HeaderTitle` (`width: 1fr`,
`content-align: left middle`), and `ProviderUsageIndicator`
(`#provider-usage-indicator`, docked right, `width: auto`).

- The title is left-aligned immediately after the icon, so it sits wherever the icon
  ends and never relates to the screen center.
- `_measure_usage_budget()` returns `inner - icon_width - title_width`, i.e. the usage
  cluster receives every cell the unclipped title does not need.
- `_entry_fragment()` in `src/sase/ace/tui/widgets/_provider_usage_indicator.py` styles
  an exact `0%` reading by emitting `f"{percent} {countdown}"` as one
  `usage_zero_value_style()` run, while the window's compact name (`fable`) and the
  vendor-rejection marker (`!`) stay on the normal badge surface.

## 2. Change one: the title is centered on the screen

### 2.1 Contract

The title's **content box is always symmetric about the header's horizontal center**.
Because the header is docked top at full width, header center is screen center.

Let `inner = header.content_size.width`, `icon = HeaderIcon.outer_size.width` (8), and
`title_width = header.format_title().cell_length` (the unclipped title, subtitle
included).

1. **Usage budget** — the right-hand region reserved for usage is

   ```text
   budget = max(0, min(inner - icon, max(icon, (inner - title_width) // 2)))
   ```

   The `max(icon, ...)` term grants usage a free allowance equal to the icon width,
   because the left margin is already `icon` cells wide and a usage region that narrow
   costs the title nothing. The `min(inner - icon, ...)` term keeps usage from
   overlapping the icon on tiny terminals.

2. **Reserved region** — `ProviderUsageIndicator` is given an explicit width equal to
   `budget` (instead of `width: auto`), keeping `dock: right` and
   `content-align: right middle`. The cluster still hugs the right edge; unused reserve
   renders as header background.

3. **Title padding** — `HeaderTitle` becomes `content-align: center middle` /
   `text-align: center`, and the header applies

   ```text
   padding-left  = max(0, budget - icon)
   padding-right = max(0, icon - budget)
   ```

   so the title's content box is exactly
   `[max(icon, budget), inner - max(icon, budget))`.

This yields two properties that matter:

- **Exact centering.** The content box is symmetric by construction, so the title text
  is centered within half a cell of the screen center at every width.
- **No jitter.** The box depends only on `inner`, `icon`, and `title_width` — never on
  how wide the usage text happens to render. Percent and countdown changes (`100%` →
  `99%`, `9h59m` → `10h0m`) on the 30-second cadence cannot move the title.

It also preserves the existing doctrine that the complete title has priority over usage
density: by construction the symmetric box is `inner - 2*budget ≈ title_width`, so usage
can never shrink the space the unclipped title needs. When
`(inner - title_width) // 2 < icon` the terminal is simply too narrow, the box collapses
to `inner - 2*icon`, and the title ellipsizes — still centered, with the full string in
its tooltip.

### 2.2 Verified geometry

A Textual probe of this exact arithmetic (icon 8, `text-align: center`, explicit usage
width, computed padding) produced:

| width | title cells | budget | content box | box center | screen center |
| ----- | ----------- | ------ | ----------- | ---------- | ------------- |
| 100   | 17          | 41     | x=41, w=18  | 50.0       | 50.0          |
| 60    | 8           | 26     | x=26, w=8   | 30.0       | 30.0          |
| 30    | 34          | 8      | x=8, w=14   | 15.0       | 15.0          |
| 20    | 8           | 8      | x=8, w=4    | 10.0       | 10.0          |

At width 30 the title correctly ellipsizes to `sase ace (v0.…` while staying centered;
at width 20 it becomes `sas…`.

### 2.3 Implementation notes

- Keep everything inside `UsageHeader._apply_usage_budget()`, which already coalesces
  title, mount, and resize triggers through `_schedule_usage_budget()` /
  `call_after_refresh`.
- **Assign styles only when the value actually changes** (both the indicator width and
  the title padding). Unconditional assignment dirties layout on every coalesced pass;
  see the TUI performance rules about keeping pump callbacks thin and avoiding new
  refresh paths.
- Update `_apply_title_tooltip()` to compare `format_title().cell_length` against the
  new symmetric box width (`inner - 2 * max(icon, budget)`) instead of `inner - icon`.
- CSS lives in two places and both must change: the `HeaderTitle` rule in
  `UsageHeader.DEFAULT_CSS`, and the higher-specificity
  `#ace-header HeaderTitle { content-align: left middle; text-align: left; }` rule in
  `src/sase/ace/tui/styles.tcss`. Missing the second silently keeps the title
  left-aligned.

### 2.4 Click semantics for the reserved region

A fixed-width usage region means blank reserved cells now belong to
`ProviderUsageIndicator` rather than to `HeaderTitle`. Clicking blank header space must
keep Textual's tall-header toggle rather than opening a modal, so:

- `ProviderUsageIndicator.on_click` claims the event **only** when the click lands
  inside the rendered text extent, i.e.
  `event.screen_x >= region.right - rendered_content_width`. Outside that extent it
  neither stops nor prevents the event, so it bubbles to `Header._on_click` and toggles
  `-tall` exactly as an empty-header click does today.
- `UsageHeader.on_click`'s docked-usage fallback uses the same extent predicate instead
  of `usage.region.contains(...)`.
- An empty cluster (usage disabled, or no selected windows) renders nothing, so every
  click in its reserved region bubbles.

## 3. Change two: an exact 0% emphasizes the whole window run

In `_entry_fragment()`, when `percent_text == "0%"`, render the window's entire value
run as **one contiguous** `usage_zero_value_style()` span: compact name (with any
`[key]` disambiguation suffix), the space after it, the `!` rejection marker and its
space when present, then `0% <countdown>`. Nothing else changes.

- The provider icon, the `·` window separator, the two-space provider gap, and the outer
  padding keep their current styles, so adjacent windows and adjacent providers stay
  attributable and two neighboring exhausted windows read as two distinct blocks.
- The `!` marker joins the inverted run rather than keeping `usage_rejected_style()`.
  Its red foreground would be invisible on the exhausted red background, and a striped
  run (emphasis, normal marker, emphasis) is harder to read than one solid exhausted
  block. The glyph itself still distinguishes it.
- The rule keys on the rendered `0%` text, exactly as today: `<1%` is not emphasized, a
  stale `0%` still is, and a passed window renders `?%` and is not.
- No new palette function is needed; `usage_zero_value_style()` already carries the
  documented ≥4.5:1 contrast against the exhausted background.
- **Plain text is unchanged** by this half of the plan — only style spans move.

## 4. Files

Source:

- `src/sase/ace/tui/widgets/usage_header.py` — budget formula, reserved width, title
  padding, tooltip threshold, click fallback predicate, CSS.
- `src/sase/ace/tui/widgets/provider_usage_indicator.py` — expose the rendered content
  width and apply the click extent predicate.
- `src/sase/ace/tui/widgets/_provider_usage_indicator.py` — contiguous zero run in
  `_entry_fragment()`.
- `src/sase/ace/tui/styles.tcss` — `#ace-header HeaderTitle` alignment.

Docs (`docs/ace.md`, application-header section around the usage-cluster example):

- Replace "The title stays left-aligned after the header icon." with the centered
  contract: the title is centered on the header line, usage occupies a right-docked
  reserve sized so the complete title still fits centered, and changing usage text never
  moves the title. Keep the existing complete-title-priority and tooltip-on-clip
  sentences.
- Update the `0%` sentence so the inverted red run covers the window's whole value run
  (name, rejection marker, percentage, countdown), and fix the following sentence, which
  currently claims adjacent names and rejected markers keep their normal surfaces —
  after this change that is true only for non-zero windows.

## 5. Tests

### 5.1 Behavioral (`tests/ace/tui/test_usage_header.py`)

Existing tests whose contracts this plan deliberately changes:

- `test_usage_header_keeps_title_left_and_usage_right` — rename to a centering test and,
  for each of the existing widths (60, 80, 120, 140, 160, 240), assert the title content
  region's center equals the header center within one cell, the icon still starts at the
  header origin with width 8, and the usage region's right edge is the header's right
  edge.
- `test_no_usage_keeps_title_and_empty_cluster` — an empty cluster now still reserves
  `budget` cells; assert nothing is rendered (`usage.render().plain == ""`) and the
  title stays centered, instead of asserting `usage.region.width == 0`.
- `test_pilot_clicks_open_usage_without_expanding_header` — click inside the rendered
  text extent (right-aligned) rather than at `offset=(1, 0)`, which is now reserved
  blank space.
- `test_usage_only_changes_do_not_move_control_row` — strengthen it to assert the
  title's **content region** (not only `region.x`) is byte-identical across every usage
  variant, including the empty variant. This is the anti-jitter contract.

New coverage:

- Complete-title priority: at a width where the title fits, the rendered title is not
  ellipsized and the measured budget never exceeds `(inner - title_width) // 2` once it
  is above the icon-width allowance.
- Degenerate narrow case: a title wider than `inner - 2*icon` ellipsizes, stays
  centered, and carries the full title in `HeaderTitle.tooltip`.
- Reserved-space clicks: a click in the blank part of the reserved usage region toggles
  `-tall` and opens no modal; a click on the rendered cluster opens `ProviderUsageModal`
  and leaves `-tall` unset.
- Idempotence: running the budget application twice in a row changes no widget style,
  guarding against a layout/refresh loop.

### 5.2 Presentation style (`tests/test_provider_usage_indicator_presentation_style.py`)

- `test_named_and_rejected_zero_neighbors_keep_normal_badge_surface` asserts today that
  `grok-preview` and `!` keep the badge surface at 0%. That is the contract being
  inverted: rewrite it to assert the whole `grok-preview ! 0% 3d4h` run is one
  `usage_zero_value_style()` span via `_assert_style_run`, that the run's neighbors
  (provider icon, outer padding) keep non-exhausted backgrounds, and that the plain text
  is unchanged.
- Extend `test_exact_zero_percent_uses_inverted_value_style` with a named window so the
  emphasized run starts at the name.
- Add a two-window case proving the `·` separator between an exhausted named window and
  a healthy one keeps the divider style, so the blocks remain distinguishable.
- Leave `test_zero_percent_state_contract` percent-token assertions intact; add the name
  token where the fixture has one.

### 5.3 Visual snapshots

Both changes move pixels, and the title move touches nearly every full-app golden (647
PNGs live in `tests/ace/tui/visual/snapshots/png/`).

- Run `just test-visual`, inspect actual/expected/diff artifacts under
  `.pytest_cache/sase-visual/`, and accept only reviewed header-title relocation and
  0%-run emphasis changes with `--sase-update-visual-snapshots`, then rerun clean.
- Named zero windows already have dedicated fixtures — the `weekly:claude-fable-5`
  entries at `remaining_percent=0.0` in
  `tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator_states.py` — so
  confirm the emphasis change is visible there before mass-accepting.
- Watch for goldens where the centered title now collides with or ellipsizes differently
  than before; that is expected at narrow widths but should never overlap the icon or
  the usage cluster.
- Prefer stable waits over sleeps if a snapshot proves timing-sensitive; do not relax
  the comparator with a pixel tolerance.

## 6. Non-goals

- No feature flag and no new config option: this is a presentation default with no old
  branch that must stay reachable.
- No change to which windows are selected, to the projection/peek pipeline, to the
  tooltip prose, or to `ProviderDisablesIndicator` and the `#top-bar` row below the
  header.
- No change to Textual's default `Header` for other screens or modals; only
  `UsageHeader` / `#ace-header` is affected.
- No new runtime option for a whitespace separator or an alternate title alignment.

## 7. Rejected alternatives

- **Auto-width usage plus two-sided title padding.** Keeps today's click surface but
  requires the header to learn the indicator's rendered width through a message on every
  content change; a stale or late measurement visibly decenters the title. The fixed
  reserve removes that coupling entirely.
- **Keep the current budget and center only when the title happens to fit.** Keeps
  maximum usage density but makes the title snap between centered and left-aligned as
  the terminal or the cluster changes, and it would truncate the title to show more
  usage — the opposite of the documented priority.
- **Leave `!` outside the emphasized run.** Produces a striped run and an effectively
  invisible red-on-red marker.

## 8. Verification

- `just check` is mandatory because tracked files change; run it through `/sase_monitor`
  if it runs long.
- `just test-visual` for the PNG suite, with the review/update loop above.
- Focused reruns while iterating: `tests/ace/tui/test_usage_header.py`,
  `tests/test_provider_usage_indicator_widget.py`,
  `tests/test_provider_usage_indicator_presentation_style.py`,
  `tests/test_provider_usage_indicator_presentation_layout.py`.
- Expect `just check`'s scoped lane to escalate, since `styles.tcss` and shared widget
  modules are in the broadening set.
- Leave the tree clean and finish through `/sase_final`.
