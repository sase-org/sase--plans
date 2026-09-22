---
tier: tale
title: "Agents status row: dot grammar, refresh countdown, and a load gauge"
goal:
  "The Agents tab status row separates unbracketed elements with dim dots, shows
  `refresh: <N>s (r)`, and moves runner load to a right-side `load: <load>/<capacity>`
  gauge with integer-preferred numbers, colored by the usage-window free-capacity
  gradient."
size: medium
proposed_by: bbugyi200.apollo.1h.f0.f0.f0.w2
create_time: 2026-09-22 10:00:33
status: wip
---

# Plan: Agents status row — dot grammar, `refresh:` countdown, and a `load:` gauge

## Goal

Redesign the one-line status row at the top of the ACE **Agents** tab so it is calmer
and more consistent:

1. Only the agent status counts keep square brackets. Every other element loses them.
2. The countdown reads `refresh: <N>s (r)` instead of `(auto-refresh in <N>s)`.
3. Top-level elements are separated by a dim `·` (the same dot used inside the status
   counts) instead of runs of spaces.
4. Runner load/capacity moves off the left side. It becomes a labeled
   `load: <load>/<capacity>` gauge at the right of the row, just before the
   model/project cluster, followed by a `·` separator.
5. `<load>` and `<capacity>` render as integers when possible (`7/10`, not `7.0/10.0`).
6. The gauge uses the same ten-step color gradient as the usage-window indicators, so a
   given color means the same amount of headroom in both places.

Rendering is presentation-only (Textual/Rich state in this repo). No `sase-core` or Rust
change is needed. The capacity numbers already come from the shared
`RunnerCapacitySnapshot`.

## Target design

### Before / after

Before:

```
12  7.0/10.0 [4 running · 1 queued · 3 done]   [view: none (p)]   [group: by status (o)]   (auto-refresh in 7s)      model: opus@high · project: +sase
```

After, full density (wide terminals):

```
12 [4 running · 1 queued · 3 done] · view: none (p) · group: by status (o) · refresh: 7s (r)      load: 7/10 · model: opus@high · project: +sase
```

After, compact density (narrow terminals). The micro-labels drop together, which is how
the launch-context cluster already behaves:

```
12 [4 running · 1 queued · 3 done] · view: none (p) · group: by status (o) · refresh: 7s (r)  7/10 · opus@high · +sase
```

### Left side (`AgentInfoPanel`) grammar

Elements, in order, joined by one dim `·` separator:

1. **Counts group** (always present): the headline count (`12`, or `12 agents` when proc
   shells exist), then the unchanged bracketed status strip
   ` [N running · N queued · … ]`, then the unchanged ` ⚙N` proc-shell badge when
   nonzero. The capacity prefix (`  7.0/10.0`) is **removed** from here.
2. **Filter** (when a query is active): unchanged internals (`filter: ` dim italic, the
   highlighted/plain query, ` seeded`, `  M/N` match count, and the partial-history
   notice stay attached to this element). Only the leading `   ` becomes `·`. The click
   span must still cover exactly the query run.
3. **View** (when `view_mode` is set): `view: ` dim + styled mode + ` (<key>)` dim when
   the view picker is available and the key is bound. No brackets.
4. **Group** (always): `group: ` dim + styled label + ` (<key>)` dim when bound. No
   brackets.
5. **Refresh** (only when `interval > 0`, as today): `refresh: ` dim + `<N>s` in the
   existing `bold #FFD700` + ` (<key>)` dim. The key comes from the Agents refresh
   keymap `self._registry.app.agents_refresh` (default `r`), rendered with
   `key_display_name()`. The hint is omitted when `is_unbound_key()` is true. Don't
   hardcode `r`.

The loading state (`Agents: …`) is unchanged.

Use one separator constant (e.g. `_ELEMENT_SEPARATOR = " · "`, style `dim`) and a small
helper to append it, so no element carries its own leading spaces. The countdown fast
path (`_countdown_text_span` / `_countdown_render_template` / `update_countdown_only`)
keeps working unchanged. Only the suffix after the countdown becomes ` (r)` instead of
`)`.

### Right side: new `AgentLoadIndicator`

A new render-only `Static` mounted in `AgentInfoRow` between the panel and the
launch-context bar:

```
AgentInfoRow
├── AgentInfoPanel        (#agent-info-panel, width 1fr)
├── AgentLoadIndicator    (#agent-load-indicator, width auto)   ← new
└── LaunchContextBar      (#launch-context-bar-agents, width auto)
```

It renders one Rich `Text`:

- full density: `load: ` (dim) + value run + `·` (dim)
- compact density: value run + `·` (dim)

The value run is `<fmt(load)>/<fmt(capacity)>`, and one style covers all of it,
including the slash.

The trailing `·` separates the gauge from the model chip. The model chip is never
hidden, so the separator never dangles. Keep the gauge out of `LaunchContextBar`. That
bar is a shared view over `LaunchContextSource` that is mounted on three tabs. Runner
load is Agents-only data.

### Number formatting (integers when possible)

Add a small pure formatter for the gauge. Call it
`format_load_value(value: object) -> str`:

- Non-numeric, `bool`, or non-finite → `—`.
- Round to 2 decimals. Render whole results as integers (`3`, `10`) and otherwise trim
  trailing zeros (`2.5`, `0.75`, `3.33`). `-0` → `0`. Examples: `3.0→"3"`, `10→"10"`,
  `2.5→"2.5"`, `0.1+0.2→"0.3"`, `1/3→"0.33"`, `9.999→"10"`.
- Weighted `%queue(weight=…)` claims can be fractional, so fractions stay visible. The
  2-decimal cap keeps the header compact. The tooltip (below) carries the same rounded
  numbers, and the exact values stay in the queue section.

Do not change `format_capacity_value`. Its other callers (queue section, models panel,
list badges) keep their current format.

### Color: the usage-window gradient, keyed on free capacity

Reuse the usage-window palette in `src/sase/ace/tui/widgets/_usage_indicator_palette.py`
(`usage_percent_color`, `usage_zero_value_style`), keyed on **free capacity percent**:

```
free_percent = round(max(0.0, limit - max(0.0, occupied)) / limit * 100.0, 6)
```

The 6-decimal rounding removes float noise (for example `41.00000000000001` vs
`40.9999999`) before the palette floors to a bucket.

| state                                                      | value text     | style                                                                                              |
| ---------------------------------------------------------- | -------------- | -------------------------------------------------------------------------------------------------- |
| `limit <= 0` (snapshot not loaded yet)                     | `—/—`          | `dim`                                                                                              |
| `occupied is None`, `limit > 0`                            | `—/<cap>`      | `dim`                                                                                              |
| `round(occupied, 2) >= round(limit, 2)` (at/over capacity) | `<load>/<cap>` | `usage_zero_value_style(dark=…)`: the inverted red chip, exactly as an exhausted `0%` usage window |
| otherwise                                                  | `<load>/<cap>` | `bold <usage_percent_color(free_percent, dark=…)>`                                                 |

On the default capacity of 10, each running agent moves the gauge exactly one color step
(dark theme):

| load | free | color                                                 |
| ---- | ---- | ----------------------------------------------------- |
| 0    | 100% | `#65C3ED` blue (calm, idle)                           |
| 1    | 90%  | `#48CCD0` cyan                                        |
| 2    | 80%  | `#4CD4B0` teal                                        |
| 3    | 70%  | `#78DB8D` green                                       |
| 4    | 60%  | `#AADC64` yellow-green                                |
| 5    | 50%  | `#CED44C` yellow                                      |
| 6    | 40%  | `#EBC04F` amber                                       |
| 7    | 30%  | `#FFA552` orange                                      |
| 8    | 20%  | `#FF805F` coral                                       |
| 9    | 10%  | `#FF5F6D` red                                         |
| ≥10  | 0%   | inverted chip `#242830 on #FF5F6D` (new agents queue) |

The light theme uses the palette's light column automatically.

Contrast was measured against the row's actual `$surface`:

- dark `#1E1E1E`: ≥ 5.6:1
- light `#D8D8D8`: ≥ 4.3:1, bold
- inverted chip: ≥ 5.0:1 in both themes

The at-capacity check compares the same 2-decimal rounding that the display uses. So a
rendered `10/10` is always the chip, and the chip never shows a load below capacity. It
also absorbs float summation noise such as `9.9999999999`.

**Decision: color the whole `<load>/<capacity>` run, not just `<load>`.** Reasons:

- The color encodes the _ratio_, not the numerator. A red `9` alone means nothing
  without its `/10`.
- It matches the usage-window precedent the palette comes from, where the whole value
  run shares one color.
- The at-capacity chip has to cover the complete fraction to read as a single gauge. A
  chip over only `10` in `10/10` looks broken.
- The `load:` label stays dim, so the colored run stays compact and framed.

Theme awareness: resolve `dark` from `self.app.current_theme.dark` and default to `True`
on any exception, as `ProviderUsageIndicator._current_dark_theme` does. Watch `self.app`
`"theme"` (`init=False`) in `on_mount` and repaint on change.

Add one sentence to the `_usage_indicator_palette.py` module docstring saying the Agents
row's load gauge shares this palette, keyed on free-capacity percent. That keeps the
"same color = same headroom" contract discoverable.

### Tooltip

Set `self.tooltip` only when the content signature changes:

- normal: `Runner load: 7 of 10 capacity units in use (3 free).` + newline +
  `New agents queue when load reaches capacity.`
- at/over capacity: `Runner load: 10 of 10 capacity units in use (at capacity).` +
  newline + `New agents queue until capacity frees up.`
- `occupied is None`: `Runner load is unavailable; capacity is 10 units.`
- `limit <= 0`: `Runner capacity has not loaded yet.`

Use `format_load_value` for every number. "Free" is `max(0, limit - occupied)`.

### Density and fitting (reliability)

`AgentInfoRow.fit_launch_context_bar()` in
`src/sase/ace/tui/widgets/launch_context_bar.py` becomes the single coordinator for both
right-side widgets:

```python
free = self.region.width - panel.content_width - 2 - _MIN_GAP_CELLS
density = _choose_launch_context_density(
    max(0, free),
    full_cells=load.full_cells + bar.full_cells,
    compact_cells=load.compact_cells + bar.compact_cells,
)
bar.set_density(density)
load.set_density(density)
```

The gauge and the model/project cluster always share one density, so the row never shows
a half-labeled cluster. As today, when even compact doesn't fit, the left panel clips on
its right and the right cluster stays whole. `AxeInfoRow` and `ArtifactsHeader` are
unchanged. `LaunchContextBar.refresh_density` is still used by `AxeInfoRow`.

`AgentLoadIndicator` API (mirror `LaunchContextBar`'s names):

- `update_load(limit: float, occupied: float | None) -> None`: no-op when unchanged.
  Otherwise it re-renders, and walks parents for `fit_launch_context_bar` (same pattern
  as `LaunchContextBar._request_host_fit`) **only when** the rendered cell width
  changed.
- `set_density(density) -> bool`: repaints when changed. It never requests a refit,
  because the row is the caller. This keeps fit → set_density → render convergent with
  no recursion.
- `full_cells`, `compact_cells`, `content_cells` properties: computed from state with
  `cell_len` over the text the density would render (pure and cheap).
- The render signature is `(limit, occupied, dark, density)`. Never read config, stat,
  or glob on these paths (tui_perf rules 1 and 8).

### Data flow

In `src/sase/ace/tui/actions/agents/_display_detail_info.py`, in
`_update_agents_info_panel_impl`:

- Right after `runner_capacity` is read, and before the `update_state` early `return`,
  look up `#agent-load-indicator` (`AgentLoadIndicator`). Tolerate `NoMatches` the same
  way the panel lookup does. Then call
  `update_load(runner_capacity.effective_limit, runner_capacity.occupied_capacity)`.
  `update_load` is a cheap no-op when nothing changed, so countdown ticks cost nothing.
- Stop passing `runner_limit` / `runner_occupied_capacity` to `update_state`. Keep
  `runner_queue_count`, which the status strip still renders as `N queued`.
- In the legacy fallback branch, replace the `update_runner_capacity(...)` call with the
  panel's new `update_runner_queue_count(queue_count)`.

## Implementation steps

1. **`src/sase/ace/tui/widgets/agent_info_panel.py`**
   - Remove the capacity prefix: `_append_capacity_prefix`, `_runner_capacity_style`,
     `_NEUTRAL_RUNNER_LIMIT_STYLE`, the `_runner_limit` / `_runner_occupied_capacity`
     state, and the `format_capacity_value` import.
   - Drop `runner_limit` / `runner_occupied_capacity` from `update_state` (signature,
     `new_stable`, `old_stable`, and the unpack).
   - Replace `update_runner_capacity(effective_limit, queue_count, occupied_capacity)`
     with `update_runner_queue_count(queue_count: int)`.
   - Rebuild `_build_display_text` with the element grammar above: separator helper,
     unbracketed `view:` / `group:`, and `refresh: <N>s (<key>)` with the key from
     `agents_refresh`.
   - Update the class docstring and any docstrings that mention "auto-refresh in".
2. **New `src/sase/ace/tui/widgets/agent_load_indicator.py`**: module-level pure helpers
   (`format_load_value`, a free-percent helper, a value-style helper, and a
   `build_agent_load_text(limit, occupied, *, dark, density) -> Text` builder), plus the
   `AgentLoadIndicator(Static)` widget described above. Import palette functions from
   `._usage_indicator_palette`, a sibling module in the same package, as
   `_provider_usage_indicator.py` does. `launch_context_bar.py` will import this new
   module, so it must **not** import `launch_context_bar.py` (that would be an import
   cycle). Type its density parameter with a local `Literal["full", "compact"]`, which
   is structurally identical to `LaunchContextDensity`.
3. **`src/sase/ace/tui/widgets/launch_context_bar.py`**: update
   `AgentInfoRow.fit_launch_context_bar` as above. Mention the gauge in the module
   docstring's visual grammar, as an Agents-row-only leading group.
4. **`src/sase/ace/tui/_app_layout.py`**: yield
   `AgentLoadIndicator(id="agent-load-indicator")` between `AgentInfoPanel` and
   `LaunchContextBar` inside `AgentInfoRow`.
5. **`src/sase/ace/tui/widgets/__init__.py` and `__init__.pyi`**: export
   `AgentLoadIndicator` (lazy-export map entry, `__all__`, stub). Keep sorted order.
6. **`src/sase/ace/tui/styles.tcss`**: next to the `#agent-info-row` rules, add
   `#agent-info-row #agent-load-indicator { width: auto; height: 1; background: $surface; }`.
7. **`src/sase/ace/tui/actions/agents/_display_detail_info.py`**: data flow as above.
8. **`src/sase/ace/tui/widgets/_usage_indicator_palette.py`**: the one-sentence
   docstring note.
9. **Docs**:
   - `docs/ace.md`, in the Agents header paragraphs (around "The active grouping
     strategy is also surfaced…" through "A nonzero queue count is cornflower blue."):
     - Describe the unbracketed `group: <label> (o)` element and the `·` element
       grammar.
     - Replace the "capacity prefix `C/L` … dim through gold … orange … red" text with
       the right-side `load: <load>/<capacity>` gauge. Cover integer formatting, the
       shared usage-window gradient keyed on free capacity, the inverted chip at/over
       capacity, `—/—` / `—/<cap>` unknown states, and compact density dropping the
       `load:` label.
     - Update the example `8.0/10.0 [8 running · 1 queued]` to
       `8 [8 running · 1 queued]` plus `load: 8/10`.
   - `docs/troubleshooting/runner-slots.md` (lines ~13-15): update the `C/L` example to
     the new `load: 8/10` gauge at the right of the row.
   - Leave `docs/ace.md`'s Patches-tab `[group: <label>]` sentence (~line 1019) alone.
     That is a different panel.

No keymap or config values change (`agents_refresh` already defaults to `r`), so
`src/sase/default_config.yml` needs no edit. No feature flag: this is a finished,
user-requested visual change with no old branch to keep reachable.

## Tests

Update existing expectations:

- `tests/ace/tui/widgets/_agent_info_panel_helpers.py`: drop `runner_limit` /
  `runner_occupied_capacity` from `stable_state_kwargs`.
- `tests/ace/tui/widgets/test_agent_info_panel_counts.py`:
  - Prefixes lose the capacity segment, e.g. `"12 [5 running · 2 stopped · …]"`,
    `"21 agents [6 running · 8 waiting · 7 done] ⚙23"`, `"5 [0 running]"`.
  - Move capacity-pressure style cases to the new indicator tests. Keep the
    running/queued/done style assertions here.
  - `test_running_count_style_is_constant_across_capacity_pressure`: assert the panel
    text no longer contains any `/<limit>` capacity fraction.
  - The `update_runner_capacity` cache test becomes an `update_runner_queue_count` test.
- `tests/ace/tui/widgets/test_agent_info_panel_badges.py`: `[group: … (o)]` /
  `[view: …]` → `group: … (o)` / `view: …`. Assert `"[group"` / `"[view"` are absent.
- `tests/ace/tui/widgets/test_agent_info_panel_state.py`: `auto-refresh in 4s` →
  `refresh: 4s (r)` (use the registry's `agents_refresh` display name, like the helpers
  do for grouping/view keys). Keep
  `test_update_state_full_rebuild_when_runner_capacity_changes` keyed on
  `runner_queue_count`.
- `tests/ace/tui/test_agents_tab_current_project_seed.py` (~line 208): drop
  `"runner_limit": 10`.
- `tests/test_capacity_snapshot_parity.py::test_capacity_header_renders_shared_snapshot_numbers`:
  assert the gauge text built from the shared snapshot. For example,
  `build_agent_load_text(...).plain` contains `0.75/1`, and no longer `0.75/1.0`.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents.py` (~line 177): prefix becomes
  `"3 [0 running · 2 queued · 1 waiting]"`. Also assert the mounted
  `#agent-load-indicator` renders `0/10` in its text.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_panels.py` (~line 151):
  `"[view: tribe]"` → `"view: tribe"`.

Add new tests:

- `tests/ace/tui/widgets/test_agent_info_panel_grammar.py` (or extend the badges file):
  - Exact plain rendering of a representative full row. It must contain
    `view: none (p) · group: by status (o) · refresh: 7s (r)`, have no `"   "` runs, and
    have no brackets outside the status strip.
  - Filter + partial-history element joined by `·`, with the click span still covering
    exactly the query run.
  - Unbound `agents_refresh` omits the ` (…)` hint.
  - `interval == 0` omits the refresh element and leaves no trailing separator.
  - Separators are `dim`.
  - The `update_countdown_only` template path patches `refresh: 4s (r)` correctly.
- `tests/ace/tui/widgets/test_agent_load_indicator.py`:
  - `format_load_value` table (the examples above, plus `None`, `nan`, `inf`, `True` →
    `—`).
  - Style mapping for capacity 10, loads 0..9 → the ten distinct bucket colors in order,
    for both `dark=True` and `dark=False`.
  - Loads 10, 12, and `9.999` (which renders `10/10`) → `usage_zero_value_style`. `9.99`
    stays a red foreground, not the chip.
  - Float-noise cases (`0.1*3` style sums, `5.9` load) land in the expected bucket.
  - Placeholder states `—/—` and `—/10` are `dim`.
  - One style span covers the whole `load/capacity` run including `/`. The `load: `
    label and `·` are dim.
  - Full vs compact text, with `full_cells - compact_cells == len("load: ")`.
  - Tooltip strings for all four states.
  - `update_load` is a no-op (no `update` call) when unchanged, and requests a host fit
    only when width changes.
  - `set_density` returns `True` only on change and never requests a fit.
  - The render path does not read `sase.config.core.get_max_running_agents`, mirroring
    the existing panel test.
- `tests/ace/tui/test_launch_context_bar.py`: add an Agents-row fit test, modelled on
  `test_row_fit_picks_density_from_free_cells`.
  - A narrow panel yields `full` on **both** the bar and the gauge.
  - A panel width that leaves room for `bar.full_cells` but not
    `load.full_cells + bar.full_cells` yields `compact` on both.

## Visual goldens

Every Agents-tab PNG golden changes, including `launch_context_bar_agents_120x40` and
light-theme variants. Regenerate with `just fix-tui-screenshots` through
`/sase_monitor`, as the tui_screenshot / lint_and_test memory notes describe. Then
inspect the retained report under `.pytest_cache/sase-visual/`: every creation, then
each update group's representative, expanding any group whose diff is not limited to the
Agents status row.

Check that:

- the row reads `N [..] · view: … · group: … (o)`,
- the right side shows `0/10 · <model> · +<project>` (compact at 120 cols), and
- nothing else moved.

Add one new golden to lock the full-density look:

- In `tests/ace/tui/visual/test_ace_png_snapshots_launch_context_bar.py`, add
  `launch_context_bar_agents_full_160x40`, the Agents row at `size=(160, 40)`. At that
  width the full `load: … · model: … · project: …` grammar fits.
- If it can be made deterministic, pin the runner capacity to a mid-gradient value (e.g.
  7 of 10). Set
  `page.app._agent_runner_capacity = RunnerCapacitySnapshot(effective_limit=10.0, occupied_capacity=7.0)`,
  call `page.app._update_agents_info_panel()`, and `wait_for_state` on the gauge text
  before capture. Then the golden shows a gradient color.
- If a background reload can race that pin, keep the fixture's natural `0/10` instead of
  adding sleeps.

Optionally take a live `sase screenshot` of the Agents tab at a wide width to eyeball
the final look. Live captures aren't goldens.

## Verification

1. `just install` if the workspace venv is stale.
2. `just fix` (or at least `just fmt`).
3. `sase tool run check` (`just check`). If symvision complains about the new module's
   helpers or private-module imports, read the `symvision.md` memory before changing
   anything. Don't just delete the flagged symbols.
4. `just fix-tui-screenshots` via `/sase_monitor`, with golden-diff inspection as
   described above.

Do not run `just check-full` (explicit-only).

## Out of scope

- The Services (`AxeInfoPanel`), Patches (`PatchInfoPanel`), and axe dashboard rows,
  which still say `(auto-refresh in …)` and use bracketed badges. Aligning them with
  this grammar is a reasonable follow-up, but it wasn't requested.
- Smarter clipping that drops whole trailing elements when the row is too narrow.
  Today's right-edge clip behavior is preserved.
