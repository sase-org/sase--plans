---
tier: tale
title: Restore Agents-tab row and banner jump hints
goal:
  Pressing ' (and the fold-hint keys) on the Agents tab paints hints on every agent row,
  collapsed banner, and panel title again, and the JUMP footer survives an async
  footer-only refresh.
size: medium
proposed_by: bbugyi200.athena.0o3
create_time: 2026-09-20 11:04:03
status: wip
---

# Restore Agents-tab row/banner jump hints (`'`) after the tribe-keyed panel refresh

## Symptom

Pressing `'` (`jump_to_entry`) on the Agents tab paints **only the panel-title hint
chips**. No agent row and no collapsed-group banner ever shows its `[x]` hint, so the
user sees "only a few of the hints" and cannot jump to a row at all.

Evidence from the reporting screenshot (`~/tmp/screenshots/20260920_103406.png`, three
tribe panels, 16 + 7 visible rows):

- 26 targets were allocated — the visible chips are `[0]` (`@default`), `[h]` (`@epic`),
  `[p]` (`@job`). Those indices are exactly `0`, `17`, `25` in `JUMP_HINT_CHARS`, i.e.
  the allocator correctly reserved `1..16` for `@default`'s 16 rows and `18..24` for
  `@epic`'s 7 visible rows.
- Only the 3 panel chips rendered. The 23 row hints did not.

So hint **allocation** is correct; hint **painting** is broken for the row and banner
channels.

## Confirmed root cause (reproduced)

`_paint_panel_widget` in `src/sase/ace/tui/actions/agents/_display_panel_widgets.py`
gates the row rebuild on `_panel_content_is_unchanged`:

```python
content_unchanged = skip_content or self._panel_content_is_unchanged(
    widget, panel_agents, grouping_mode=grouping_mode, panel_collapsed=panel_collapsed,
)
...
if not content_unchanged:
    widget.update_list(..., jump_hints=local_jump_hints,
                       banner_jump_hints=local_banner_hints, ...)
```

`_panel_content_is_unchanged` compares only `_panel_row_signature` (per-agent
`identity`, grouping signature, `tribe`), the collapsed flag, and the grouping mode.
**Entering jump mode changes none of those**, so it returns `True`, `update_list` is
never called, and the hint maps never reach the rows. The panel _title_ is painted
unconditionally just above that gate, which is why the title chips are the only hints
that survive.

This guard was introduced on 2026-09-19 by `45a7895b6` ("feat(tui): key AgentList
widgets by tribe and skip sibling rebuilds"). Before that commit `update_list` ran
unconditionally for every expanded panel, so hints painted. This is a regression from
that commit.

Reproduced directly against `tests/ace/tui/_agent_display_diff_helpers.py`: with
`_entry_jump_mode_active = True` and populated hint maps,
`_refresh_affected_panel_widgets(all_keys)` returns `True`, both panels report
`update_list_calls == 0`, the border titles contain `[0]` / `[3]`, and every rendered
row prompt is free of any `[n]` marker.

### Blast radius — the same gate breaks fold-hint mode

`_refresh_panel_fold_hint_display`
(`src/sase/ace/tui/actions/agents/_panel_hint_folding.py`) projects fold targets onto
the **same** `jump_hints` / `banner_jump_hints` channels via
`_panel_fold_hint_display_maps`, and reaches rows through the same
`_refresh_affected_panel_widgets` → `_paint_panel_widget` path. The `L`, `H` and `,H`
fold hints are therefore broken in the same way and must be fixed and covered by the
same change.

Both the selective path (`_refresh_affected_panel_widgets`) and the full path
(`_refresh_panel_widgets_impl`) funnel through `_paint_panel_widget`, so one fix at the
gate covers every caller, including the `_refresh_agents_display(list_changed=True)`
fallback.

## Second defect — entry-jump mode is missing from the footer mode ladder

`_apply_agent_footer_update`
(`src/sase/ace/tui/actions/agents/_display_detail_footer.py`, the `if/elif` chain
starting at the `_member_jump_pending_digit` branch) dispatches on
`_member_jump_pending_digit`, `_panel_fold_hint_mode_active`, `_fold_mode_active`,
`_leader_mode_active`, `_bang_mode_active`, `_copy_mode_active` and
`_custom_mode_active` — but **not** on `_entry_jump_mode_active`. Any call to
`_refresh_agent_footer_bindings_only` while jump mode is active therefore clobbers the
`JUMP` footer back to the ordinary Agents bindings. That call is reachable from
asynchronous work that is _not_ suppressed during jump mode, notably the artifact-file
discovery completion in `src/sase/ace/tui/actions/agents/_panel_artifact_cache.py`.

This matches the companion screenshot `~/tmp/screenshots/20260920_103356.png`, taken 10
s earlier: the yellow `[0]` / `[h]` / `[p]` title chips are present (they are produced
_only_ by an active hint mode — `agent_panel_border_title` emits the `[...]` prefix
solely from its `jump_hint` argument) while the footer shows the normal Agents bindings.
Jump mode was live with its footer already reset.

Note that the periodic auto-refresh itself is correctly suppressed during jump mode
(`src/sase/ace/tui/actions/event_refresh/_auto_refresh_surfaces.py`), so this is
specifically the footer-only refresh path, not the auto-refresh tick.

## Why no test caught this

Existing jump-hint tests are split either side of the break:

- `tests/ace/tui/test_jump_hint_rendering.py` and friends call `AgentList.update_list` /
  `_format_agent_option` **directly** with a hint map, so they pass — the widget renders
  hints correctly.
- `tests/ace/tui/test_jump_hint_order_agents.py` covers `_jump_candidate_indices`
  **allocation** order only.
- Every app-level panel-paint test (`test_agent_panel_title_refresh.py`,
  `test_agent_panels_display.py`, `tests/perf/test_agents_display_rebuild_guard.py`)
  calls `_refresh_panel_widgets(jump_hints=None)`.

Nothing asserts that an app-level paint with a non-empty hint map actually reaches the
rows. That is the gap to close.

## Plan

### 1. Record the hint maps each panel widget last painted

In `src/sase/ace/tui/widgets/agent_list.py` / `_agent_list_build_rebuild.py`:

- Have `AgentList.update_list` record the **caller-supplied** `jump_hints` and
  `banner_jump_hints` on the widget (for example `_painted_jump_hints` /
  `_painted_banner_jump_hints`) **before** the hidden-agent `local_index_map` remap
  runs.

  **Trap:** `update_list` rewrites `jump_hints` through `local_index_map` when some
  agents are not rendered in the panel. If the recorded map is the post-remap one it
  will never compare equal to the local map the paint context computes, and every paint
  will rebuild — silently undoing the perf win of `45a7895b6`. Record the incoming maps,
  or normalise both sides identically.

- Initialise both attributes wherever the other per-paint widget state is initialised,
  and clear them in `AgentList.render_collapsed` alongside `_agents` /
  `_row_render_ctx`, so a collapse→expand cycle cannot resurrect a stale comparison.

### 2. Make the unchanged-gate hint-aware

In `src/sase/ace/tui/actions/agents/_display_panel_widgets.py`:

- Extend `_panel_content_is_unchanged` with the local hint maps and return `False` when
  either differs from what the widget last painted.
- Pass `local_jump_hints` and `local_banner_hints` from `_paint_panel_widget` (they are
  already computed immediately above the gate).

Keep the comparison exactly as narrow as this: **only** the two hint channels. Do not
fold marks, unread, `fold_counts`, selection or `current_group_key` into the signature.
Those already have their own incremental paths — marks and unread go through
`_try_patch_agent_row` / `patch_row`, selection through `_refresh_panel_highlights` —
and widening the gate would reintroduce full panel rebuilds on ordinary navigation,
which is precisely what `sase memory read tui_perf.md` rule 6 and
`tests/perf/test_agents_display_rebuild_guard.py` exist to prevent.

In the steady state both sides are empty, so the added comparison is `{} == {}` and
costs nothing.

`patch_row` already carries `ctx["hint_char"]` forward, so single-row patches occurring
during hint mode keep their hints; no change is needed there.

### 3. Add the missing entry-jump footer branch

In `src/sase/ace/tui/actions/agents/_display_detail_footer.py`, add an
`_entry_jump_mode_active` branch to `_apply_agent_footer_update` that calls
`footer_widget.update_jump_bindings(has_back=...)`, computing `has_back` the same way
`_update_jump_footer` (`src/sase/ace/tui/actions/navigation/_entry_jump_mode.py`) does —
for the Agents tab that is `bool(self._entry_jump_agents_anchor_stack)`.

Place the branch so it does not shadow the existing modes; entry-jump mode is mutually
exclusive with them in practice, but order it consistently with
`_active_panel_title_jump_hints`, which checks `_entry_jump_mode_active` before
`_panel_fold_hint_mode_active`. Prefer reusing the existing `has_back` computation
rather than duplicating it.

### 4. Regression tests

Add app-level paint tests (the missing layer), reusing
`tests/ace/tui/_agent_display_diff_helpers.py`, which already counts `update_list_calls`
and exposes the mounted `AgentList` widgets:

1. **Entry-jump rows paint.** With `_entry_jump_mode_active = True` and populated
   `_entry_jump_index_to_hint` / `_entry_jump_panel_to_hint`, calling
   `_refresh_affected_panel_widgets(all_keys)` on a tree whose row membership is
   unchanged must call `update_list` and leave each rendered agent row prompt carrying
   its `[hint]` marker. Assert on rendered row text, not on call counts alone — a
   call-count-only assertion would have passed even before `45a7895b6` broke this.
2. **Collapsed-group banner hints paint** through the same path, via
   `_entry_jump_banner_to_hint`.
3. **Exit clears hints.** After `_exit_entry_jump_mode`, no row prompt and no panel
   title retains a hint marker.
4. **Fold-hint mode rows/banners paint** through `_refresh_panel_fold_hint_display` with
   `_panel_fold_hint_mode_active = True`.
5. **Perf guard holds.** With no hint mode active and unchanged membership, a repaint
   still does not call `update_list` (a focused assertion next to the new tests, so a
   future widening of the gate fails loudly here rather than only in `tests/perf/`).
6. **Footer survives an async footer-only refresh.** With jump mode active, calling
   `_refresh_agent_footer_bindings_only` must leave `JUMP` bindings in place. Extend or
   mirror the existing footer coverage in `tests/ace/tui/test_jump_hint_rendering.py` if
   a suitable fake footer already exists there.

## Verification

- `just fix` (or at minimum `just fmt`) inline first.
- `sase tool run check` (preferred over raw `just check`). Run `just install` first if
  this workspace's virtualenv is stale.
- Rendered TUI output changes, so also run `just fix-tui-screenshots` and inspect the
  report, per `sase memory read lint_and_test.md`. Hand a full visual run to
  `/sase_monitor` with the `TESTING` / `TESTED` pair if it outruns the turn.
- Do **not** run `just check-full`; it is not requested here.
- Manual confirmation in a live TUI: on the Agents tab press `'` and check every agent
  row, every collapsed banner and every panel title carries a hint; press `L` / `,H` and
  check fold hints likewise. Note `tui_perf.md` rule 15 — a long-running TUI keeps
  executing the snapshot it imported at start, so restart the TUI before judging the
  fix.

## Out of scope

- The two-character pending-prefix path does not re-render to narrow the visible hint
  set (`_handle_entry_jump_key` sets `_entry_jump_pending_prefix` and returns without a
  refresh). That is pre-existing behaviour, not part of this regression. File it via
  `/sase_new_task` if it is worth changing.
- No Rust core change: this is presentation-only Textual paint state, on the Python side
  of the `rust_core_backend_boundary` litmus test.
