---
tier: tale
title: Shrink the Agents node panel when a group is collapsed
goal:
  Collapsing an agent group on the Agents tab narrows the node panel to the rows that
  are still visible, instead of staying as wide as the hidden rows demanded.
size: medium
proposed_by: bbugyi200.kellys_mbp.07
create_time: 2026-09-12 08:33:07
status: wip
---

# Plan: Shrink the Agents node panel when a group is collapsed

## Problem

On the ACE **Agents** tab, the left node panel is sized to fit its widest agent row.
Rows hidden inside a **collapsed group** still count toward that width, so collapsing
the group that holds the longest agent node leaves the panel just as wide as before — a
wall of empty cells next to short rows, and the detail panel on the right stays
needlessly narrow.

Reproduction (matches the user's screenshot): Agents tab, `group: by machine (o)`, two
machine groups (`here`, `apollo`) where `apollo` owns the longest agent node. Press `H`
(collapse group) on `apollo`. The `apollo` banner collapses to a one-line rule, every
`apollo` row disappears — and the panel keeps the width those now-invisible rows
demanded.

Expected: collapsing a group shrinks the node panel to what is still visible (floored by
the existing minimum-width clamps); expanding it restores the wider panel.

## Root cause

`build_list()` in `src/sase/ace/tui/widgets/_agent_list_build_rebuild.py` measures
**every agent in the panel's agent list**, while the tree it emits rows from **skips
agents inside collapsed groups**:

1. `build_agent_tree(...)` (`src/sase/ace/tui/models/agent_groups/_tree.py:244`) emits a
   `TreeEntry(kind="group")` for a collapsed group and then `continue`s past its members
   — see the `if cur_proj_collapsed: continue` / `if cur_cs_collapsed: continue` /
   `if cur_subgroup_collapsed: continue` / `if cur_root_collapsed: continue` guards.
   Those agents are never emitted as rows.
2. But the measurement pass in `build_list` walks the full list —
   `for i, agent in enumerate(agents):` (`_agent_list_build_rebuild.py:161`) —
   formatting every agent and folding each one into `max_left` / `max_suffix`
   (`_agent_list_build_rebuild.py:244-245`), including the ones step 1 just hid.
3. `target_width = max(_MIN_BANNER_WIDTH, max_left + gap + max_suffix)`
   (`_agent_list_build_rebuild.py:248`) therefore still reflects hidden rows, and
   `optimal_width = max(max_width, banner_width) + _PADDING`
   (`_agent_list_build_rebuild.py:334`) publishes it via
   `widget._content_requested_width` → `AgentList._refresh_requested_width()` →
   `AgentList.WidthChanged` (`src/sase/ace/tui/widgets/agent_list.py:280-292`).
4. The container handler `on_agent_list_width_changed`
   (`src/sase/ace/tui/actions/_event_widgets.py:278-301`) is **not** at fault: it
   already takes the max over mounted panels' `_requested_width` and re-applies the
   clamped result both upward and downward (covered by
   `tests/ace/tui/test_agent_left_panel_width.py::test_collapsing_widest_panel_drops_aggregated_width`).
   Once `build_list` publishes a smaller request, the panel shrinks on its own.

Note that agent-level folds (a parent agent whose children are folded away) are already
filtered out upstream — `self._agents` holds the display list and
`panel_agents = slot.agents` is a slice of it
(`src/sase/ace/tui/actions/agents/_display_panel_widgets.py:117`) — so "width follows
visible rows" is the established contract. Group collapse is the one hidden-row case
that leaks into the measurement, because the full list must still be passed to
`build_list` for banner summaries
(`cached_format_banner_option(..., widget._agents, ...)`) and `GroupRow.agent_indices`.

Collapsing already triggers a full rebuild (`_collapse_group_fold` →
`self._refilter_agents()`,
`src/sase/ace/tui/actions/agents/_folding_agent_groups.py:435-446`), so fixing the
measurement inside `build_list` is sufficient — no new refresh path is needed.

## Secondary hazard this fix must not introduce

Banner rows pad their rule to the requested `width` but never truncate:
`pad_len = max(2, width - used)`
(`src/sase/ace/tui/widgets/_agent_list_render_banner.py:178,186`). Today banners are
always rendered at `banner_width == target_width`, which is driven by the widest agent
row, so a banner effectively never overflows. Once the panel shrinks to only the
_visible_ rows, a collapsed group's banner (whose chip still summarizes every hidden
member, e.g. `apollo ────  23 agents · 1 awaiting`) can be the widest emitted row — and
if the published width ignored that, the banner would spill past the panel and soft-wrap
onto a second line. So the published width must be the max over **emitted** rows,
banners included.

The cheap, drift-free way to get that is to measure the banner Options that are already
being built (`Option.prompt` is a `rich.text.Text`, so `cell_len` is exact), rather than
mirroring `format_banner_option`'s arithmetic in a second helper the way the
Artifacts-tab list does (`banner_natural_width`,
`src/sase/ace/tui/widgets/_patch_list_banner.py:190`). Measuring costs nothing extra and
cannot drift out of sync with the formatter, and it avoids a second
`compute_banner_summary()` pass per banner on the rebuild hot path
(`sase/memory/tui_perf.md` rule 6: full agent-list rebuilds are the most expensive UI
operation).

## Fix design

All changes are presentation-only Textual layout state, so they stay in this repo — no
`sase-core` wire/API surface is involved (per the Rust Core Backend Boundary core
memory).

### 1. Derive the visible-row set from the tree

Add a small pure helper to `src/sase/ace/tui/widgets/_agent_list_build_analysis.py`:

```python
def visible_agent_indices(tree: list[TreeEntry]) -> set[int]:
    """Return the agent indices *tree* actually emits as rows.

    Agents inside a collapsed group are absent: the tree walk skips them,
    so they must not influence row measurement or per-row render state.
    """
    return {
        entry.agent_idx
        for entry in tree
        if entry.kind != "group" and entry.agent_idx is not None
    }
```

Re-export it from the `_agent_list_build.py` facade alongside `compute_tier_styles` /
`compute_visible_parents` (keeps `__all__` sorted; satisfies symvision's cross-module
private-use rules the same way the existing helpers do).

### 2. Measure — and format — only the rows that get emitted

In `build_list` (`_agent_list_build_rebuild.py`), right after
`tree = build_agent_tree(...)` and `compute_tier_styles(...)`:

- compute `visible = visible_agent_indices(tree)`;
- skip hidden agents at the top of the pre-format loop
  (`if i not in visible: continue`), so `agent_parts`, `widget._row_render_ctx`,
  `widget._row_tier_styles`, `max_left` and `max_suffix` all cover exactly the emitted
  rows.

Skipping the formatting (not just the measurement) is deliberate and safe:

- `widget._row_by_agent_idx` / `_row_entries` already only ever contain emitted rows, so
  `patch_row()` (`_agent_list_build_patching.py:147-230`) already returns `False` for a
  hidden agent today — it looks up `_row_render_ctx` with `.get()` and bails when either
  the ctx or the row is missing, and the caller falls back to a full rebuild. Dropping
  the ctx for hidden agents changes no reachable behavior.
- `try_remove_rows()`'s remap reads both maps with `.get()`
  (`_agent_list_build_patching.py:128-133`), so index gaps are fine.
- It is also a rebuild-cost win: a collapsed group's members are no longer formatted on
  every refresh; they are formatted when the group is expanded again (same total work,
  deferred), and the render cache makes the re-expand path cheap.

Keep `_agent_row_chrome_mode(agents, grouping_mode)` and the `semantic_tribes` /
`named_tribe_identity_colors` computation on the **full** list: those decide panel-wide
chrome (machine chip, fleet badge, tribe colors) and must not flicker when a group
collapses.

In the emit loop, look the option up defensively (`option = agent_options.get(i)` and
`continue` when missing) so a future tree change can never raise `KeyError` mid-rebuild.

### 3. Publish the width of the widest _emitted_ row

Keep `target_width` (row alignment column) as it is today, but derived from visible rows
only, and track the widest emitted banner while walking the tree:

```python
gap = 2 if max_suffix > 0 else 0
target_width = max(_MIN_BANNER_WIDTH, max_left + gap + max_suffix)
banner_width = target_width
widget._target_width = target_width
...
max_emitted_width = target_width
# in the banner branch of the emit loop, after `option = cached_format_banner_option(...)`:
prompt = option.prompt
if isinstance(prompt, Text):
    max_emitted_width = max(max_emitted_width, prompt.cell_len)
...
optimal_width = max_emitted_width + _PADDING
```

`cached_format_banner_option`'s cache key includes `width`
(`_agent_list_render_cache.banner_render_key`), so a cache hit is always an Option
rendered at the current `width` — measuring a cached Option is correct.

Behavior this produces:

- Collapsing a group drops its rows out of `max_left` / `max_suffix`, so the panel
  shrinks to the widest remaining row (or to a collapsed group's banner, which can no
  longer be clipped), floored by `_MIN_BANNER_WIDTH` (40) + `_PADDING` (8) and then by
  the container's `MIN_AGENT_LIST_WIDTH` (60) clamp in `on_agent_list_width_changed`.
- Expanding restores the previous width; no other call site changes.
- With **every** group collapsed the panel falls to banner width — the desirable end
  state.

## Scope note the implementer should expect

`#agent-list-container` holds **all** Agents-tab panels stacked vertically and they
share one width (`#agent-list-container AgentList { width: 100% }`,
`src/sase/ace/tui/styles.tcss:3753`), and `on_agent_list_width_changed` takes the max
across mounted panels. So collapsing a group in one panel shrinks the container only if
that group held the widest row **across all panels** — in the user's screenshot the
second (`@epic`) panel also has long rows, so the visible shrink may be partial until
those are collapsed too. That is correct behavior, not a bug; do not add per-panel
widths.

## Steps

1. Add `visible_agent_indices(tree)` to `_agent_list_build_analysis.py` and re-export it
   from `_agent_list_build.py`.
2. In `_agent_list_build_rebuild.build_list`, skip non-visible agents in the pre-format
   loop so measurement and per-row render state cover only emitted rows.
3. Track the widest emitted banner Option and publish
   `optimal_width = max(target_width, widest_banner_cells) + _PADDING`; make the
   emit-loop option lookup defensive.
4. Update the comments around the pre-format pass and `optimal_width` so the next reader
   knows the invariant is "measure what you emit" (and why banners are measured, not
   recomputed).
5. Add the regression tests below; run the verification gates.

## Tests

New module `tests/ace/tui/widgets/test_agent_list_collapsed_group_width.py` (follow the
style of `tests/ace/tui/widgets/test_agent_list_grouping_collapse.py`; build agents with
`tests/ace/tui/widgets/_agent_list_grouping_helpers.make_agent` and collapse groups with
`AgentGroupFoldRegistry`, passing a pinned `now=` so runtime suffixes are
deterministic):

1. **Collapsing the widest group shrinks the panel.** Two projects, `projB` holding
   markedly longer agent names than `projA`. Render expanded, record
   `widget._target_width` / `widget._requested_width`; re-render with `("projB",)`
   collapsed and assert both dropped, and that `_target_width` now matches the width
   implied by `projA`'s visible rows.
2. **Expanding restores the width.** Same fixture, collapse then expand → the original
   `_requested_width` returns (guards against a one-way shrink).
3. **A hidden row's runtime suffix no longer stretches the right-aligned column.** Give
   the hidden agent a long finished-timestamp suffix and the visible agents short ones;
   assert the visible rows' `cell_len` equals the new, smaller `_target_width`.
4. **No emitted row is clipped by the published width.** Construct a collapsed group
   whose banner chip is wider than every visible row; assert
   `max(option.prompt.cell_len for option in widget._options) + 8 <= widget._requested_width`
   and that the collapsed banner's own `cell_len <= widget._target_width` is _not_
   required (i.e. the banner may exceed the row column, but never the published panel
   width).
5. **Hidden agents are not rendered or measured.** Assert
   `set(widget._row_render_ctx) == set(widget._row_tier_styles)` equals the visible
   index set, and that `patch_row(widget, <hidden idx>)` returns `False` (documents the
   pre-existing fallback the optimization relies on).
6. **Floor still holds with everything collapsed.** Collapse every L0 group →
   `_target_width == _MIN_BANNER_WIDTH` and every banner still renders its full label +
   chip.

Existing suites to re-run and reconcile (do not weaken an assertion to make it pass — if
one legitimately changes, update it and say why in the commit body):

- `tests/ace/tui/widgets/test_agent_list_grouping_collapse.py`,
  `tests/ace/tui/widgets/test_agent_list_grouping*.py`,
  `tests/ace/tui/widgets/test_agent_list_runtime_rendering_layout.py` (asserts
  `_requested_width == _target_width + 8`; still true whenever no banner out-measures
  the rows — confirm rather than assume),
  `tests/ace/tui/widgets/test_agent_list_runtime_patching.py`,
  `tests/ace/tui/widgets/test_agent_list_try_remove_rows.py`,
  `tests/ace/tui/widgets/test_agent_list_row_lookup.py`,
  `tests/ace/tui/test_agent_left_panel_width.py`.
- PNG goldens: `just test-visual`. Expect drift in Agents-tab snapshots that render a
  collapsed group — most likely
  `tests/ace/tui/visual/test_ace_png_snapshots_agents_group_lane_collapse.py` and the
  panel / grouping snapshots. Inspect the diffs in `.pytest_cache/sase-visual/` and only
  then accept with `--sase-update-visual-snapshots`; a narrower panel with intact
  banners is the intended change, a wrapped or clipped banner row is not.

## Verification

Per `sase/memory/lint_and_test.md`:

- `just install` first if this ephemeral workspace's venv is stale (`sase_core_rs`
  import errors are the tell).
- `just check` inline during development.
- `just test-visual` for the PNG suite (excluded from `just test` / `just check`).
- `just check-full` before landing, run **only** through `/sase_monitor` with the
  `TESTING` / `TESTED` status pair.
- Optional but valuable given the hot-path edit: capture `SASE_TUI_PERF=1` j/k timings
  or the `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` bench before/after on a list
  with a large collapsed group; the change should be neutral-to-better, never worse.

Manual check against the reported case: `sase ace`, Agents tab, `o` to
`group: by machine`, select a row in the widest machine group, press `H`, and confirm
the node panel narrows and the detail panel widens, with banner rules and chips still
fully drawn.

## Out of scope

- The **Artifacts** tab has the same defect class: `_patch_list_render.py:83` measures
  `for i, cs in enumerate(patches)` — every patch, including those inside collapsed
  groups — while its tree skips them. It is a separate surface with its own banner-width
  helper (`banner_natural_width`), so it should be a follow-up task bead rather than a
  widening of this tale. That bead could **not** be filed while authoring this plan:
  `sase bead` is currently broken in the host checkout —
  `ImportError: cannot import name 'FinalizerAssignedBeadWire' from 'sase.core.finalizer_wire'`
  (`~/projects/github/sase-org/sase/src/sase/finalizers/declaration_manifest.py:17`), an
  unrelated mid-refactor state. File the bead (`task(bug)`) once the CLI imports again.
- No change to `on_agent_list_width_changed`, `MIN_AGENT_LIST_WIDTH` /
  `MAX_AGENT_LIST_WIDTH`, per-panel widths, or the panel-collapse (`render_collapsed`)
  path.
- No feature flag: this is a layout fix with no persisted state, no wire/API surface,
  and no old branch that must stay reachable for backward compatibility.

## Risks

- **Banner overflow** — mitigated by step 3 (measure emitted banners) plus test 4 and
  the PNG suite.
- **Stale per-row state for hidden agents** — `patch_row` / `try_remove_rows` already
  treat missing ctx as "fall back to a full rebuild"; test 5 pins that.
- **Width thrash** — width changes only on rebuild, and `_refresh_requested_width()`
  posts `WidthChanged` only when the value actually changes, so navigation cannot flap
  the panel.
