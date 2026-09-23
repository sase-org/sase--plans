---
tier: tale
title: Labeled, dot-separated top-bar indicator cluster
goal:
  "The TUI top-bar indicators render as dim `<type>: ` labeled groups (procs, monitors,
  updates, overrides, provider, prompts, inbox) joined by dim ` · ` separators, matching
  the load/model/project status row beneath, reliably across every visibility
  combination and terminal width."
size: medium
proposed_by: bbugyi200.athena.0q9
create_time: 2026-09-23 15:17:18
status: wip
---

# Labeled, dot-separated top-bar indicator cluster

## Goal

Restyle the right-aligned indicator cluster in sase's TUI top bar (the row with the
`Agents │ Artifacts │ Services` tabs) so it speaks the same visual language as the
status-row cluster directly beneath it
(`load: 5/8 · model: opus@high · project: +sase`): every indicator group gets a dim
`<type>: ` micro-label, and visible groups are joined by a dim `·` separator.

Today the top-bar cluster is a run of unlabeled badges (`CODEX +2` `❄ 4`
`?5 #1 f1 ◈128`, plus `⚙ N` proc/monitor chips and a `⬆ N` updates chip when present).
It relies on glyphs and hue alone for identity. After this change it reads as:

```text
Agents │ Artifacts │ Services          provider:  CODEX +2  · prompts:  4  · inbox: ?5 #1 f1 ◈128
                                      load: 5/8 · model: opus@high · project: +sase
```

This is presentation-only Textual work. It does not touch the Rust core boundary, config
schema, keymaps, or `src/sase/default_config.yml`.

## Design

### Grammar

- **A group** is `<label>: <body>`. The label is a dim micro-label with its trailing
  space (the same treatment as `load: `, `model: `, and `project: ` beneath it). The
  body is the indicator's value run.
- **Separators.** Visible groups are joined by a dim `·`. The cluster never renders a
  leading separator, a trailing separator, or two separators in a row. This must hold
  for every combination of visible groups.
- **Bodies keep their color and fill.** The top bar is the alert row, so filled chips
  stay filled and keep their current colors and one-cell inner padding (for example, the
  prompts count stays `bold #1a1a1a on #00D7AF`, rendered `4`). Unfilled bodies (inbox)
  drop the edge pad spaces they used to carry because the label and separators now
  supply the spacing.
- **Labels replace identity glyphs.** A glyph whose only job was to name the indicator
  type is removed, because the label now does that: `❄` (prompts), `⚙` (procs and
  monitors), `⬆` (updates), and the empty-state `✉` (inbox). Glyphs that tell members
  apart _within_ a group stay: notification tab icons (`?`, `#`, `⚑`, `✉`-as-tab-icon,
  `☾`, …), the provider priority `★`, the `core` tag, the `CLI` segment label, `∞`, and
  `+N` overflow counts.
- **Right edge.** The cluster sits flush right, with no trailing pad, so its last cell
  lines up with the `project:` chip in the status row beneath. The cluster background is
  `$surface`, the same as the tab strip and the status rows, so labels and dots sit on
  one continuous strip.

### Groups

The left-to-right order stays as it is today (already pinned by
`tests/ace/tui/test_top_bar_order.py`). It runs from activity (procs, monitors) to
system state (updates) to launch routing (overrides, provider) to personal queues
(prompts, inbox). The always-visible inbox anchors the right edge directly above
`project:`.

| Label       | Widget (id kept)                                             | Visible when                           | Body (full density)                                                                          | Click opens                                          |
| ----------- | ------------------------------------------------------------ | -------------------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `procs`     | `ProcIndicator` (`#proc-indicator`)                          | ≥1 ACE-owned proc running              | `N` filled with `PROC_GEAR_HUE`                                                              | Admin Center Procs tab (`open_tasks_panel`)          |
| `monitors`  | `MonitorIndicator` (`#monitor-indicator`)                    | ≥1 monitor shell running               | `N` filled with `MONITOR_GEAR_HUE`                                                           | Admin Center Procs tab (`open_tasks_panel`)          |
| `updates`   | `UpdatesAvailableIndicator` (`#updates-indicator`)           | SASE/plugin count > 0 or CLI count > 0 | `N` lime-on-moss, then the existing `core` tag, then `CLI N` sage-on-moss                    | Updates tab (`open_updates_panel`, unchanged)        |
| `overrides` | `AliasOverridesIndicator` (`#alias-overrides-indicator`)     | any non-default alias/setting override | the existing violet pill, unchanged (`@medium@max ∞`, `epic lander 2h`)                      | Launch settings (`open_models_panel`, unchanged)     |
| `provider`  | `ProviderDisablesIndicator` (`#provider-disables-indicator`) | any active disable or priority         | the existing routing pill, unchanged (`CLAUDE off 42m`, `CODEX ★ priority 42m`, `CLAUDE +2`) | Launch settings (`open_models_panel`, unchanged)     |
| `prompts`   | `StashedPromptsIndicator` (`#stashed-prompts-indicator`)     | ≥1 stashed prompt                      | `N` filled with the existing teal `#00D7AF`                                                  | Prompt stash picker (`open_prompt_stash`)            |
| `inbox`     | `NotificationIndicator` (`#notification-indicator`)          | always                                 | per-tab chips `?5 #1 f1 ◈128`, dim `+K` overflow, dim snoozed-only `☾3`, dim `0` when empty  | Notification modal (`show_notifications`, unchanged) |

The user named `provider`, `inbox`, `procs`, `prompts`, and `updates`. The two groups
the user did not name get these labels:

- `monitors`: monitor shells are detached supervisors, and the docs already counted them
  separately from ACE procs. Folding them into `procs:` would put two unlabeled numbers
  side by side once the gear glyph is gone.
- `overrides`: the violet pill covers alias overrides _and_ launch-setting overrides
  such as `epic lander`, so `aliases` would be wrong. It also matches the `override`
  label the status row already uses for the default-model override.

Labels are fixed strings. They do not pluralize or change with state, so the row stays
scannable.

Example renders (full density):

- Calm: `inbox: 0` (fully dim)
- Typical: `provider:  CODEX +2  · prompts:  4  · inbox: ?5 #1 f1 ◈128`
- Busy:
  `procs:  2  · monitors:  1  · updates:  3  core  CLI 2  · overrides:  @medium@max ∞  · provider:  CLAUDE off ∞  · prompts:  4  · inbox: ⚑1 ✉18`

### Density

The row mirrors the status-row cluster's full/compact rule:

- **full**: labels shown. Chosen when the full cluster fits in the cells left over after
  the tab strip and a 2-cell minimum gap.
- **compact**: labels dropped on _every_ group at once, separators kept (for example
  ` 2  ·  1  ·  4  · ?5 #1`). Mixed per-group label dropping would be confusing, so it
  is never used. Compact renders even if it also overflows, like the status row. Hue and
  tooltips still identify each group.

The choice is a pure function of `(free_cells, full_cells)`. It is recomputed only when
the top bar resizes or a group's shape changes (visibility or cell width). Because
`full_cells` is measured independently of the current density, the choice is
deterministic and cannot oscillate.

### Interaction

- The label is part of each group widget, so hovering or clicking the label behaves
  exactly like hovering or clicking the value.
- Every group is clickable and opens its home surface (see the table). This adds clicks
  to procs, monitors, and prompts, which have none today.
- Existing tooltips keep their content. Add tooltips where none exist:
  - procs: `N running proc(s)` plus `Click to open the Procs tab`
  - monitors: `N running monitor(s)` plus the same click line
  - prompts: append `Click to open the prompt stash` to the existing tooltip

## Implementation

Use repo-relative paths. Follow the surrounding widget idioms (`Static` subclasses, pure
static builders, `text_signature` dedupe) and the TUI perf rules: no I/O, no
subprocesses, and no new timers on these paths.

1. **New `src/sase/ace/tui/widgets/top_bar_group.py`**: base class and pure helpers (it
   imports no indicator modules, which avoids import cycles).
   - `TopBarDensity = Literal["full", "compact"]`, `TOP_BAR_SEPARATOR = " · "`, and a
     2-cell minimum-gap constant.
   - `filled_count_chip(count: int, hue: str) -> Text`: `N` in `bold #1a1a1a on <hue>`,
     or empty `Text("")` at `count <= 0`. It is the shared body for procs, monitors, and
     prompts.
   - `separator_visibility(visible: Sequence[bool]) -> tuple[bool, ...]`: for n groups
     it returns n-1 flags. The separator before group i shows iff group i is visible and
     some group before i is visible. It is pure and is the single source of truth for
     the separator invariant.
   - `choose_top_bar_density(free_cells: int, *, full_cells: int) -> TopBarDensity`:
     full iff `free_cells >= full_cells`.
   - `class TopBarGroup(Static)`:
     - Class attributes `GROUP_LABEL: ClassVar[str]` and
       `CLICK_ACTION: ClassVar[str | None]`.
     - It holds the current body `Text` and density, and exposes `group_visible` (body
       plain text is non-empty), `full_cells`, and `compact_cells` (0 when hidden).
     - `set_density()` returns True only on change.
     - A protected `_set_body(body)` repaints `Text(f"{label}: ", style="dim")` plus the
       body (body only in compact, empty `Text("")` when hidden). It no-ops when the
       body's `text_signature` is unchanged. When visibility or cell widths changed, it
       asks the hosting cluster to resync by walking `parent` for a
       `sync_top_bar_groups` callable (the same pattern as
       `AgentLoadIndicator._request_host_fit`).
     - `on_click` runs `CLICK_ACTION` when set.
     - The rendered text must be correct before mount, because the constructors already
       build initial content.
2. **Restyle the seven indicators as `TopBarGroup` subclasses.** Keep their public APIs
   (`set_count`, `set_available`, `set_tabs`, `refresh`, the `count`/`pinned_count`/…
   properties) and every widget id, so callers in `actions/_proc_action_observer.py`,
   `actions/update_toast.py`, `actions/lifecycle.py`,
   `actions/agents/_notification_polling.py`, `actions/agent_workflow/_leader_mode.py`,
   and `actions/agent_workflow/_prompt_bar_stash*.py` keep working unchanged.
   - Each static `_build_content(...)` now returns the **body only**. The base class
     adds the label.
   - `ProviderDisablesIndicator._replace_content` keeps its tooltip handling but routes
     the content through `_set_body`.
   - Remove each widget's own `is_mounted` repaint branches in favor of `_set_body`.
   - `proc_indicator.py`: bodies use `filled_count_chip` with `PROC_GEAR_HUE` /
     `MONITOR_GEAR_HUE`; add tooltips and click. `proc_gear_chips.gear_chip` stays for
     the Procs tab header; update its module docstring, which says the top bar shares
     the gear chip.
   - `stashed_prompts_indicator.py`: body is `filled_count_chip(count, _STASH_ACCENT)`,
     with no `❄`. Refresh the class docstring and add click and tooltip line.
   - `updates_indicator.py`: drop `⬆` from both segments (`N`, `CLI N`). Keep the `core`
     tag. Update the `update_accents.py` module docstring. `UPDATE_GLYPH` stays because
     the Update panel and plugins browser still use it.
   - `notification_indicator.py`: body without edge pads. Empty state is dim `0` (no
     `✉`); snoozed-only is dim `☾N`; chips are joined by one space; overflow is dim
     ` +K`. Keep the tooltip logic as is. Update the class docstring.
   - `alias_overrides_indicator.py` / `provider_disables_indicator.py`: bodies unchanged
     (padded pills). Set the label and click action through the base class.
3. **New `src/sase/ace/tui/widgets/top_bar.py`**:
   - `TopBarIndicators(Horizontal)`, id `top-bar-indicators`. Its `compose()` yields the
     seven groups in the order above, each preceded (except the first) by a
     `Static(TOP_BAR_SEPARATOR, classes="top-bar-separator")`. `sync_top_bar_groups()`
     applies `separator_visibility` to separator `display` and then asks the host
     `TopBar` to refit. It caches the last applied `(visible tuple, density)` so
     repeated syncs touch nothing. It also exposes `full_cells` / `compact_cells`
     (visible groups plus visible separators) and `set_density()` (fans out to groups,
     then resyncs).
   - `TopBar(Horizontal)`, id `top-bar`. `on_resize` calls `fit_top_bar_indicators()`,
     which computes `free = region.width - tab_bar.content_cells - gap` and applies
     `choose_top_bar_density`. Model it on `AgentInfoRow.fit_launch_context_bar` in
     `widgets/launch_context_bar.py`.
   - `TabBar` (`widgets/tab_bar.py`) gains a `content_cells` property: the cell width of
     its last-built label run plus horizontal padding. Cache it when content is built;
     do not re-render to measure.
4. **Layout and CSS.**
   - `src/sase/ace/tui/_app_layout.py`:
     `with TopBar(id="top-bar"): yield TabBar(...); yield TopBarIndicators(id="top-bar-indicators")`.
   - Export only the symbols actually imported elsewhere through `widgets/__init__.py` /
     `__init__.pyi` (lazy-export pattern) to keep symvision clean.
   - `src/sase/ace/tui/styles.tcss`:
     - Replace the seven per-id indicator rules with
       `TopBarIndicators { width: auto; height: 1; background: $surface; }`,
       `TopBarGroup { width: auto; height: 1; }`, and
       `TopBarIndicators .top-bar-separator { width: auto; text-style: dim; }`.
     - Leave `#top-bar` / `#tab-bar` rules as they are, and leave the rules for widgets
       that live elsewhere (`#llm-override-indicator`, `#current-project-indicator`,
       `#provider-usage-indicator`).

## Tests

Before finishing, read the TUI memory notes (`tui.md`, `tui_perf.md`,
`tui_screenshot.md`) and `lint_and_test.md` with `/sase_memory_read`.

- **New `tests/ace/tui/widgets/test_top_bar_group.py`** (pure):
  - `separator_visibility` over all 2^7 visibility subsets: joining the displayed
    children equals `" · ".join(visible group texts)`, with no leading, trailing, or
    doubled separators.
  - `choose_top_bar_density` at the fit boundary (`free == full` → full,
    `free == full-1` → compact).
  - `filled_count_chip` style and zero behavior.
  - Label composition in full and compact, and the hidden group rendering as empty.
  - `_set_body` no-ops on an identical body.
- **New mounted test `tests/ace/tui/test_top_bar_indicators.py`** (`AcePage`):
  - Drive groups through show → hide → show transitions and assert the cluster's
    rendered plain text after each step.
  - Assert hidden groups and separators have zero width.
  - At a wide size, all seven groups render labeled.
  - At `size=(80, 30)` with the busy state, the cluster goes compact and stays within
    the top-bar bounds.
  - Resizing back to wide restores full.
  - Clicking each newly clickable group runs its action (monkeypatch the app action).
- **Update existing tests** for the new body/label text:
  `tests/test_stashed_prompts_indicator.py`, `tests/test_notification_indicator.py`,
  `tests/test_updates_indicator.py`, `tests/ace/tui/widgets/test_proc_indicator.py`,
  `tests/test_alias_overrides_indicator.py`,
  `tests/test_provider_disables_indicator.py`,
  `tests/test_provider_disables_indicator_widget.py`,
  `tests/_provider_disables_indicator_helpers.py`,
  `tests/ace/tui/test_top_bar_palette.py` (neighbors built with `filled_count_chip`;
  keep the contrast guards), and `tests/ace/tui/test_top_bar_order.py`:
  - `#top-bar` children become `["tab-bar", "top-bar-indicators"]`.
  - Pin the cluster's group order with separators interleaved.
  - Keep the narrow in-bounds assertions.
  - `tests/ace/tui/visual/_ace_png_snapshot_startup.py`: update
    `DEFAULT_VISUAL_NOTIFICATION_BADGE` (it compares the rendered indicator text, which
    now carries the `inbox: ` label and no edge pads).
  - Grep `tests/` for the old strings (`❄`, `⚙ `, `⬆ `, `✉ 0`) and for `#top-bar`
    children to catch the remaining call sites.
- **Visual coverage.** Add
  `tests/ace/tui/visual/test_ace_png_snapshots_top_bar_indicators.py` with two goldens:
  all seven groups visible at a wide size (full labels), and the same state at a narrow
  size (compact). Build them from the existing indicator snapshot fixtures
  (`test_ace_png_snapshots_updates_indicator.py`, `…_alias_overrides_indicator.py`,
  `…_notification_indicator.py`, `…_prompt_stash.py`).

## Docs

- `docs/ace.md`: add a short "Top-Bar Indicators" overview under "Tab Bar Display" with
  the grammar, the seven labels in order, the density rule, and the click targets.
  Rewrite the "Proc Indicator" and "Monitor Indicator" subsections, keeping their
  headings so existing anchors still resolve, to describe `procs: N` / `monitors: N`
  instead of gear icons. Refresh the provider-pill (~line 3989), violet override-pill
  (~line 4222), notification-indicator (~line 4414), stash-badge (~line 6364), and
  updates-badge (~line 7753) mentions to use the labeled forms.
- `docs/configuration.md` (~line 431, updates badge) and `docs/notifications.md` (~line
  355–372, indicator anatomy): drop `⬆` / `✉ 0` wording and describe the `updates:` /
  `inbox:` labels. The within-inbox "no separator glyph between chips" rule still holds;
  the `·` separates groups, not tab chips.
- Run `just fmt` so the Markdown tables and wraps stay formatted.

## Verification

1. `sase tool run check` (the agent-default `just check`), which must pass.
2. PNG goldens. The top bar appears in nearly every ACE golden (`inbox: …` is always
   visible), so expect broad but _row-local_ churn.
   - Run `just fix-tui-screenshots` (the full form, through `/sase_monitor` if it
     outlasts the turn), then inspect the retained report as `tui_screenshot.md`
     requires.
   - **Acceptance rule:** every updated golden may differ only in the top-bar row
     (screen row 2, below the usage header). Any diff outside that row is a regression
     to fix, not approve.
   - Individually review each created golden (the two new top-bar ones) and each
     expanded update group.
3. Live check with `sase screenshot -s 200x30 -o /tmp/topbar_wide.png` and
   `sase screenshot -s 90x30 -o /tmp/topbar_narrow.png`. Inspect the PNGs:
   - labels are dim, dots are dim, bodies keep their fills and colors;
   - the cluster's right edge aligns with `project: +sase` beneath;
   - the narrow capture is compact with no stray dots.

## Acceptance criteria

- The top-bar cluster renders `<type>: <body>` groups joined by dim `·`, with no
  leading, trailing, or doubled separators in any visibility combination.
- Labels are exactly `procs`, `monitors`, `updates`, `overrides`, `provider`, `prompts`,
  and `inbox`, in that left-to-right order.
- `prompts: N` shows no snowflake, and `N` keeps the teal filled highlight. Procs,
  monitors, and updates drop their identity glyphs. Inbox's empty state is `inbox: 0`.
- Narrow terminals fall back to compact (all labels dropped together) and return to full
  on widening, without flicker or oscillation.
- Every group is clickable and opens its home surface. Existing widget ids and setter
  APIs are unchanged for all callers.
- No new I/O, subprocess, or timer on any render, refresh, or resize path. Unchanged
  bodies are no-ops.
- `just check` passes, goldens are regenerated and inspected under the row-local rule,
  and the docs describe the new labels.

## Out of scope

- The status-row launch-context cluster and runner load gauge (the style reference).
  Leave them unchanged.
- The usage header row above the tabs (`ProviderUsageIndicator`).
- New config knobs for labels or density. The design is intentionally not configurable.
- Changing indicator semantics, counts, colors, tooltips' existing content, or polling
  cadence.
