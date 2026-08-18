---
tier: tale
title: Render the settled-monitor gear badge on tribe panel titles
goal:
  Every Agents-tab tribe panel title ends with a grey `⚙N` badge counting the finished
  monitors contained in that tribe, using the same glyph, hue, and settled/running
  partition as the clan/family container-row badge, so a fully collapsed panel still
  says how much monitored work already ran.
size: medium
proposed_by: bbugyi200.athena.06j
create_time: 2026-08-18 13:59:49
status: wip
---

# Render the settled-monitor gear badge on tribe panel titles

## Goal

An Agents-tab tribe panel currently tells you how many sase agents it holds and how they
are distributed across statuses, but says nothing about monitors. Clan and family
**container rows** already carry two gear badges — amber `⚙N` for running monitors in
the subtree, grey `⚙N` for finished ones (commit `845253505`). Lift the grey (settled)
badge to the **panel title** so a tribe whose panel is collapsed, or whose families are
all folded, still reports how much monitored work has completed inside it.

Target rendering — the badge is the last element of the title, one space after the
closing `]` of the existing scoped-metric chip:

```
@review · 5 [R2 D3] ⚙4
▸ ∴ @research · 12 [R2] ⚙1
```

## Current behavior (read this before editing)

Panel titles are built in one place and fanned out to five call sites:

- `src/sase/ace/tui/actions/agents/_display_panel_titles.py`
  - `agent_panel_counts(agents, unread_ids)` → `AgentPanelCounts`: filters the panel
    slice down to `visible_top_level_agents` (`not agent_is_tree_child`) and projects
    statuses through `sase_agent_status_counts`.
  - `agent_panel_border_title(...)` appends, in order: optional `[hint] ` prefix,
    `❖ `/`▸ ` marker, icon + `@tribe` (or `All agents`), `↺`, `↻N` fold-restore,
    ` · <lane_count>`, then ` ` + the `[S1 R2 …]` chip from `format_agent_count_chip`.
    The chip is the **last** thing appended today.
- `src/sase/ace/tui/actions/agents/_display_panel_collection.py:105`
  (`PanelCollectionMixin._agent_panel_title`) is the only builder; it is called from
  `_display_panel_widgets.py` (full rebuild and affected-panel rebuild),
  `_display_panel_layout.py`, `_display_panel_collection.py`
  (`_refresh_agent_panel_titles`), and `_display_panel_patches.py` (row-patch fast path,
  plus a `counts=`-only fallback).

The row-side badge already exists and must be reused, not reinvented:

- `src/sase/ace/tui/models/agent_family_members.py`:
  `MonitorLaneCounts(running, settled)`, `NO_MONITOR_LANES`,
  `monitor_row_is_settled(row)`, `monitor_lane_counts(agent)` (traverses
  `runtime_children` + `followup_agents` of _one container_, cycle-guards on `id(row)`,
  dedupes on `row.identity`, and excludes the container itself).
- `src/sase/monitor_state.py`: `MONITOR_GLYPH = "⚙"`, `MONITOR_GLYPH_COLOR = "#FFAF5F"`,
  `monitor_state_is_terminal`.
- `src/sase/ace/tui/widgets/_agent_list_styling.py`:
  `_MONITOR_SETTLED_COUNT_GLYPH_STYLE = "#9E9E9E"` (deliberately non-bold).
- `src/sase/ace/tui/widgets/_agent_list_render_agent.py:479` renders the two row badges.

## Design decisions

Make these exact choices; each one has a reason a reviewer will ask about.

1. **Count the settled lane only.** The badge mirrors the grey row badge, so it uses
   `monitor_row_is_settled` unchanged: `completed`, `failed`, `timeout`, `stopped`,
   `lost`, or any monitor row with a `stop_time`. A monitor that has not reported a
   terminal state counts as running and is therefore _not_ in this badge. Reusing the
   identical predicate is what makes the panel count and the row counts unable to
   disagree.
2. **Do not add an amber running badge to the panel title.** Out of scope for this tale;
   the shared counter returns both lanes, so a later change is a two-line render
   addition. Do not render it now.
3. **Fold-independent by construction.** Count by traversing subtrees from the panel's
   `visible_top_level_agents`, never by scanning the rendered panel slice for monitor
   rows: a collapsed family contributes no monitor rows to `self._agents`, so a
   slice-based count would drop to zero the moment the user pressed `h`. Top-level rows
   are always in the slice (folding only hides children), and the `Agent` objects in the
   slice keep their full `runtime_children` / `followup_agents`, exactly as the
   container-row badge relies on.
4. **Scope = the population the adjacent total already describes.** The panel slice
   omits hidden agents and transient top-level `STARTING` rows, so their monitors are
   not counted. That is the same population `lane_count` and the metric chip cover; do
   not add a second, wider agent source.
5. **Dedupe across the whole panel, in one traversal.** Two top-level rows can reach the
   same monitor row (`runtime_children` and `followup_agents` overlap), so the
   panel-level count needs one shared `visited_ids` / `seen_identities` pair across all
   roots — not a `sum()` of per-row `monitor_lane_counts` calls, which would
   double-count.
6. **Include a top-level row that is itself a monitor.** A monitor nests under its
   starter today, so this should never fire; counting it anyway keeps the partition
   total honest instead of silently dropping a row if projection ever changes. This is
   the one behavioral difference from `monitor_lane_counts`, which correctly excludes
   its container.
7. **Zero-suppress.** `settled == 0` renders nothing at all — no glyph and no trailing
   space. Every existing exact-title assertion must keep passing byte-for-byte.
8. **Position rule: last element of the title.** "After the `]`" and "after the total
   when the chip is empty" are the same rule — append last. The chip is zero-suppressing
   (`format_agent_count_chip` returns empty `Text` when every metric is zero) and a
   `Starting`-bucket agent counts toward `lane_count` without landing in any metric, so
   `settled_monitors > 0` with no `]` present is reachable and must render
   `@chop · 3 ⚙2`.
9. **Keep the grey hue on selected panels.** The chip re-styles brackets and metric
   letters with the focus accent when a panel is selected but keeps each metric's
   semantic color. The gear is a metric, not chrome: always `#9E9E9E`, selected or not.
   Note for the reviewer: `#9E9E9E` sits very close to the title's neutral chrome
   `#AFAFAF`. That is accepted — cross-surface identity with the row badge matters more
   than separation from title chrome, the `⚙` glyph carries the meaning, and the
   contrast that has to survive is grey-vs-amber, which is unchanged.
10. **Separator space uses `_PANEL_COUNT_STYLE`.** Mirror how the title already appends
    the spaces before `↺`, `↻N`, and the chip.
11. **Carry the count on `AgentPanelCounts`, and keep it out of `metric_items()`.**
    `AgentPanelCounts` is the title's single carrier of numeric data, and the row-patch
    fallback in `_display_panel_patches.py` builds it directly, so riding on it means
    every call site gets the badge with no signature churn. But `metric_items()` feeds
    the disjoint-status invariant asserted in `tests/ace/tui/test_agent_panel_titles.py`
    (`sum(metric_items()) == lane_count`): monitors are not agents, so adding
    `settled_monitors` there would break a true invariant. Leave `_PANEL_METRIC_LABELS`
    / `_PANEL_METRIC_STYLES` untouched. Consequence: `counts=None` renders no badge,
    same as it renders no chip.
12. **Do not reuse `src/sase/ace/tui/proc_gear_chips.py`.** `gear_chip()` builds the
    _filled_ `⚙ N` badge with a background used by the top bar and the Procs tab header.
    This is the bare foreground-hue `⚙N` text badge. Different visual object; reusing
    `gear_chip` here would be wrong.
13. **Stay in Python; no Rust core change.** Per the Rust-core boundary rule this is
    presentation aggregation over TUI `Agent` rows, and the one piece of real domain
    semantics (which monitor states are terminal) already lives in the shared
    `sase.monitor_state.monitor_state_is_terminal`. Do not move traversal into
    `sase-core`.
14. **Add no new refresh path.** Verified live: `Agent` is a plain dataclass whose
    `runtime_children` / `followup_agents` participate in `__eq__`, so a descendant
    monitor settling makes its container compare unequal in `build_agent_display_diff` →
    `changed_same_position` → `_try_patch_agent_row` → that path already rebuilds the
    panel title. Full and affected-panel rebuilds recompute titles too. Do not touch
    `_refresh_agent_panel_titles` or add a timer.

## Implementation

### Step 1 — share the settled hue from the monitor token module

`src/sase/monitor_state.py` already owns `MONITOR_GLYPH` and the amber
`MONITOR_GLYPH_COLOR`, so it is the established home for monitor presentation tokens and
the way to keep two surfaces from drifting.

- Add `MONITOR_SETTLED_GLYPH_COLOR = "#9E9E9E"` beside `MONITOR_GLYPH_COLOR`, with a
  short comment that it is the finished-monitor lane hue, and add it to `__all__`.
- In `src/sase/ace/tui/widgets/_agent_list_styling.py`, redefine
  `_MONITOR_SETTLED_COUNT_GLYPH_STYLE = MONITOR_SETTLED_GLYPH_COLOR` (import it
  alongside the existing `MONITOR_GLYPH` import), keeping the existing explanatory
  comment. Row rendering is unchanged.

Symvision note: the linter checks functions and classes, not module constants, so a
shared constant needs no pragma. The new _function_ in step 2 does need a real non-test
consumer — step 3 is it.

### Step 2 — one traversal core, two entry points

In `src/sase/ace/tui/models/agent_family_members.py`, factor the existing traversal into
a private accumulator and expose a panel-scoped entry point. Both entry points must live
in this file: a private class cannot be imported across files without tripping
Symvision, so the panel function cannot live in `_display_panel_titles.py` and share the
core.

- Add a private `_MonitorLaneTally` (or equivalent) holding `visited_ids`,
  `seen_identities`, `running`, `settled`, with a `visit(row)` that cycle-guards on
  `id(row)`, dedupes on `row.identity`, classifies `row.is_monitor` rows via
  `monitor_row_is_settled`, and recurses over
  `(*row.runtime_children, *row.followup_agents)`.
- Reimplement `monitor_lane_counts(agent)` on top of it with **identical observable
  behavior**: seed the tally with the container's children only, so the container itself
  is still excluded. Every existing test in
  `tests/ace/tui/models/test_agent_family_members.py`,
  `tests/ace/tui/widgets/test_agent_list_monitor_rows.py`, and
  `tests/ace/tui/widgets/test_agent_render_cache.py` must pass untouched.
- Add `panel_monitor_lane_counts(rows: Iterable[Agent]) -> MonitorLaneCounts`: one tally
  shared across all `rows`, visiting each row itself and its subtree. Docstring must
  state the two differences from `monitor_lane_counts` — roots are counted, and dedupe
  spans all roots — and why (decisions 5 and 6).
- Export `panel_monitor_lane_counts` in `__all__` (keep the list sorted).

### Step 3 — compute and render

`src/sase/ace/tui/actions/agents/_display_panel_titles.py`:

- Add module constants `_PANEL_MONITOR_GLYPH = MONITOR_GLYPH` and
  `_PANEL_MONITOR_SETTLED_STYLE = MONITOR_SETTLED_GLYPH_COLOR`.
- Add `settled_monitors: int = 0` as the **last** field of `AgentPanelCounts`, with a
  docstring line saying it counts finished monitors in the panel's subtrees and is
  intentionally absent from `metric_items()` because monitors are not agents (decision
  11). A trailing defaulted field keeps every existing keyword construction valid.
- In `agent_panel_counts`, set
  `settled_monitors=panel_monitor_lane_counts(visible_top_level_agents).settled` — reuse
  the list already computed for the status projection; do not re-filter.
- In `agent_panel_border_title`, after the chip block, append when
  `counts is not None and counts.settled_monitors`: a `" "` with `_PANEL_COUNT_STYLE`,
  then `f"{_PANEL_MONITOR_GLYPH}{counts.settled_monitors}"` with
  `_PANEL_MONITOR_SETTLED_STYLE`.

No other source file changes: all five title call sites route through
`_agent_panel_title`, and `_display_panel_patches.py`'s `counts=`-only fallback picks
the badge up for free.

### Step 4 — docs

- `docs/ace.md`, badge legend table (~line 1840): the existing grey `⚙N` row currently
  reads "N finished monitors in a family/clan subtree (grey)". Extend it to cover the
  tribe panel title.
- `docs/ace.md`, **Tribe Side Panels** (~line 1453, the paragraph describing
  `[S1 R2 W1 F1 U1 D3]`): add that the title can end with a grey `⚙N` badge after the
  metric chip counting finished monitors anywhere in the tribe's subtrees; that it is
  fold- and collapse-independent; that it is zero- suppressed; that it keeps its grey
  hue on a selected panel while brackets and metric letters take the focus accent; and
  that running monitors are not shown here (only on container rows).
- `docs/ace.md`, monitor-row paragraph (~line 1852) and `docs/monitors.md` (~line 277,
  the paragraph about collapsed family/clan container rows): add that the tribe panel
  title aggregates the finished lane across the whole tribe, so a fully collapsed panel
  still reports completed monitored work.
- Leave `src/sase/ace/tui/modals/help_modal/agents_bindings.py` alone. Its entry lives
  under "Agent Row Glyphs" and its text ("N finished monitors (grey)") is already
  lane-scoped rather than surface-scoped, so it reads correctly for the title badge too.
  Deliberate — do not "fix" it.

## Tests

### `tests/ace/tui/models/test_agent_family_members.py`

Add `panel_monitor_lane_counts` coverage (a focused new module is fine if this file is
getting long):

1. Partitions running vs. settled across several top-level rows in one call.
2. Dedupes a monitor reachable from two roots (present in one root's `runtime_children`
   and another's `followup_agents`) — counted once.
3. Counts a top-level row that is itself a monitor (decision 6).
4. Counts a monitor nested two levels down (clan container → family → monitor), proving
   the whole subtree is reached.
5. A monitor with `stop_time` set but a missing/unknown `monitor_state` counts as
   settled; one with neither counts as running.
6. Empty input and monitor-free input return zeroed counts.
7. A cycle (row reachable from itself) terminates.

### `tests/ace/tui/test_agent_panel_titles.py`

Use the existing `AgentPanelCounts(...)` construction style and exact `title.plain`
assertions:

8. With a chip: `assert title.plain == "@chop · 3 [R1 W2] ⚙2"` — badge one space right
   of `]`.
9. Empty chip (all metrics zero, nonzero total): `assert title.plain == "@chop · 3 ⚙2"`.
10. `settled_monitors=0` renders nothing — assert an exact title with no trailing space.
11. `counts=None` renders no badge.
12. Style: the `⚙2` span uses `#9E9E9E` (assert against the imported token, not a
    literal) and the space before it uses `_PANEL_COUNT_STYLE` — use the existing
    `_assert_title_span` / `_assert_title_range_style` helpers.
13. `selected=True`: brackets/letters take the focus accent while `⚙2` stays grey.
14. `collapsed=True`: `▸ @chop · 3 [R1 W2] ⚙2` — the collapsed case is the whole point
    of the feature.
15. `merge_tribe_panels=True`: `All agents · 3 ⚙2`.
16. `agent_panel_counts` is fold-independent: build a family container whose
    `runtime_children` include a settled monitor, pass **only** the container (no
    monitor row in the list, simulating a collapsed fold), and assert
    `settled_monitors == 1`.
17. `agent_panel_counts` does not double-count a clan container and its member family
    rows when both are in the slice (only the container is top-level).
18. Extend/keep the disjoint-metric assertion around line 452 so it still holds with
    `settled_monitors` populated — i.e. `sum(metric_items()) == lane_count` while
    `settled_monitors > 0`.
19. Cross-surface agreement: the panel's `settled_monitors` equals the sum of the grey
    badges its container rows would render — compute the expected value with
    `monitor_lane_counts` per container and compare. This is the test that prevents the
    two surfaces from drifting.

### `tests/ace/tui/test_agent_panel_title_refresh.py`

20. Through `_FakeApp._refresh_panel_widgets`, a family with a settled monitor in tribe
    `@apple` badges the `@apple` panel title and leaves `@banana` unbadged — proves the
    count is panel-scoped end to end, not global.

### Visual snapshots (`just test-visual`)

Two goldens now gain a title badge and **must** be regenerated with
`--sase-update-visual-snapshots`; inspect the diffs in `.pytest_cache/sase-visual/` and
confirm the only change is the new title badge:

21. `tests/ace/tui/visual/snapshots/png/agents_settled_monitor_lane_badge_120x40.png` —
    its fixture (`settled_monitor_family_agents`, 1 running + 3 finished monitors) gains
    ` ⚙3` on the panel title. Extend `test_settled_monitor_lane_badge_png_snapshot` in
    `tests/ace/tui/visual/test_ace_png_snapshots_agents_families.py` with an SVG-text
    assertion for the title badge alongside the existing `⚙1` row assertion.
22. `tests/ace/tui/visual/snapshots/png/agents_family_conversation_monitor_120x40.png` —
    its fixture in
    `tests/ace/tui/visual/test_ace_png_snapshots_agents_family_panel.py:145` has one
    `completed` monitor, so its title gains ` ⚙1`.

Those are the only two visual fixtures with a non-running monitor; every other monitor
fixture is `running` with no `stop_time` and must be unchanged. If any third golden
moves, stop and investigate rather than accepting it.

## Verification

- `just install` first (ephemeral workspaces drift on dependencies).
- `just check` for the lint gates and the diff-scoped test lane. `symvision` must be
  clean: `panel_monitor_lane_counts` is public and needs its real consumer in
  `_display_panel_titles.py`.
- `just test-visual` to regenerate and then re-verify the two goldens.
- `just check-full` before landing (this change touches shared render paths and
  regenerates goldens). Run it **only** through `/sase_monitor`
  (`sase monitor start --command 'just check-full' …`) with a `--next` action — never
  inline.
- Spot-check `tests/ace/tui/test_agent_left_panel_width.py`: the title grows by three
  cells when the badge renders and titles participate in panel width negotiation
  (`AgentList.update_border_title` → `_refresh_requested_width`). Those fixtures have no
  monitors today, so they should be unaffected; if a width assertion moves, the badge is
  being rendered where it should not be.
- Perf: `agent_panel_counts` is on the panel-title path, so this adds one subtree
  traversal per panel per title build — the same order of work container rows already
  pay per render. Do not add a cache. If
  `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` shows a real regression, the fix is
  to fold the panel count into the existing per-row traversal, not to introduce a new
  cache key or refresh path.

## Out of scope

- The amber running-monitor lane on panel titles (decision 2).
- The selected-panel `TRIBE` header in the prompt panel, its `Composition` field, and
  the `Z` tribe metadata zoom document.
- Clan container rows, family container rows, and monitor row rendering — all unchanged.
- `sase stats` and tribe/clan summaries, which by documented design do not count
  monitors as agents.
