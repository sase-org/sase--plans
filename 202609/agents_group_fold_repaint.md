---
tier: tale
title: Repaint Agents-tab grouping folds on the incremental display path
goal: Folding or unfolding an Agents-tab grouping banner (H / l, any incremental grouping
  mode) immediately repaints the affected panel, so what is drawn always matches the
  fold state that j/k navigation uses, while unchanged refreshes stay on the cheap
  patch path.
size: medium
proposed_by: bbugyi200.apollo.29
status: done
---

# Plan: Repaint Agents-tab grouping folds on the incremental display path

## Problem

On the Agents tab, pressing `H` to fold a grouping banner (for example the `Done` status
group under `group: by status`) updates fold state right away: `j`/`k` already skip the
hidden rows. The panel keeps drawing the group expanded, though, and the hidden rows
stay on screen. Pressing `L` draws the fold correctly, because arming the `L` fold hints
repaints the focused panel.

## Root cause (confirmed with a repro)

Grouping-banner folds live in the per-panel `GroupFoldRegistry`
(`panel_fold_registry(app, panel_key)`). They are not part of the agent roster: a
collapsed status, machine, or project banner still keeps its member rows in
`app._agents`. Only the tree builder (`build_agent_tree(..., fold_registry=...)`) hides
them at render time.

The `H` path is `action_hooks_or_collapse_all` → `_collapse_group_fold`
(`src/sase/ace/tui/actions/agents/_folding_agent_groups.py`). It does
`registry.collapse(target)` and then `_refilter_agents()`. `_refilter_agents` snapshots
`previous_agents`, rebuilds the same roster, and ends in
`_refresh_agents_display_after_finalize(previous_agents=...)`. That call takes
`_try_refresh_agents_display_incremental`
(`src/sase/ace/tui/actions/agents/_display.py`) whenever the grouping mode is STANDARD,
BY_STATUS, or BY_MACHINE. BY_STATUS was admitted there by
`1fa7e5fc3 feat(tui): admit by-status incremental agent refresh`; before that it always
fell back to a full rebuild.

The incremental path picks panels to repaint only from the roster diff
(`build_agent_display_diff`) and `panel_rebuild_scope`. A fold-only change leaves the
roster identical, so the diff is empty, `PanelRebuildScope()` is empty, no panel is
rebuilt, and only `_refresh_panel_highlights()` runs. The `AgentList` widget keeps the
rows it built under the old fold state. Navigation, which rebuilds the tree from the
live registry, and the screen now disagree. Auto-refresh ticks also stay on this patch
path, so the stale paint survives until something else forces a panel repaint.

Why `L` "fixes" it: `L` → `action_toggle_selected_agent_panels` →
`_arm_panel_fold_hint_mode` → `_refresh_panel_fold_hint_display` →
`_refresh_affected_panel_widgets({focused_key})`. That call repaints the focused panel
with `skip_content=False`. `panel_paint_key` already includes
`(id(fold_registry), fold_registry.version)`, so the paint key no longer matches and
`update_list` rebuilds the rows with the collapsed registry.

Repro (via the `tests/ace/tui/_agent_display_diff_helpers._DisplayDiffApp` harness):

1. Use BY_STATUS with one `epic` panel holding RUNNING + DONE agents, and paint it.
2. `panel_fold_registry(app, "epic").collapse(("Done",))`.
3. `_apply_roster(app, agents, agents)`.

The result is `update_list_calls == 0`, `full_rebuilds == 0`, an unchanged option count,
and no trace record.

The same defect affects other callers that change a grouping registry and then rely on
`_refilter_agents()`:

- `l` on a collapsed banner (`_expand_fold` banner branch in the same file), which
  expands the registry but leaves the banner drawn collapsed.
- `reconcile_panel_fold_registries` pruning (`clear_unknown`) during a finalize.

The fold-persistence installer already works around this locally with
`refilter(previous_agents=[])` in `_install_loaded_agents_fold_state`. Its comment says
it does that "so changed fold structure cannot be mistaken for an identity-only
incremental patch". The panel fold-hint path also calls
`_refresh_agents_display(list_changed=True)` explicitly.

## Fix design

Make the incremental display path fold-aware instead of patching each call site. Every
panel whose rendered rows were built under different grouping-fold state than its
registry holds now is rebuilt through the existing per-panel rebuild lane. Unchanged
refreshes stay on the cheap patch path, as the perf guards require.

Rejected alternative: pass `previous_agents=[]` (or call
`_refresh_agents_display(list_changed=True)`) from `_collapse_group_fold` and the
`_expand_fold` banner branch. That is a two-line fix, but it keeps the "every mutator
must remember" contract that caused this regression. It would also leave reconcile
pruning and any future fold mutator exposed. Do not also add call-site overrides once
the display path handles this; one mechanism is enough.

### 1. Record the fold state each `AgentList` was built with

- Add a small pure helper, `group_fold_snapshot(fold_registry) -> frozenset[GroupKey]`.
  A good home is `src/sase/ace/tui/models/group_fold.py`, next to `GroupFoldRegistry`.
  - `None` → `frozenset()`.
  - An object with `snapshot()` → that.
  - Otherwise `frozenset(getattr(registry, "collapsed", ()))`.
- Compare logical collapsed-key sets, not `id()`/`version`. Registries can be
  re-allocated (`AgentGroupFoldRegistry.restore`, or reconcile dropping and re-creating
  a scope), and a version bump with no net change (collapse then expand) must not force
  a repaint.
- In the widget build paths, which already receive `fold_registry`, record
  `widget._rendered_group_folds = group_fold_snapshot(fold_registry)`:
  - `build_list` in `src/sase/ace/tui/widgets/_agent_list_build_rebuild.py` (reached
    from `AgentList.update_list`).
  - The successful-insert branch of `try_insert_rows` in
    `src/sase/ace/tui/widgets/_agent_list_build_patching.py`. Do not record on a
    declined insert.
- Initialize `self._rendered_group_folds: frozenset[tuple[str, ...]] | None = None` in
  `AgentList.__init__` and reset it to `None` in `AgentList.render_collapsed`.
  `src/sase/ace/tui/widgets/agent_list.py` is already at the 700-line `toobig` tier, so
  keep its net growth to these couple of lines. Put all logic in the build modules and
  the helper.

### 2. Detect fold-stale panels in the incremental display path

- Add a method, e.g. `_group_fold_stale_panel_keys() -> set[PanelKey]`. Put it in a
  small panel-display mixin or module rather than growing `_display.py` (685 lines)
  much; `_display_panel_widgets_keys.py` or a new focused module are both fine. For each
  key in `self._panel_group.panel_keys`:
  - Resolve its widget with `panel_widget_id_for_key(key)` / `query_one`. Skip
    `NoMatches` and retiring widgets (`panel_widget_is_retiring`).
  - Skip widgets whose `_panel_collapsed` is true (whole-panel collapse renders no rows;
    re-expanding already forces a repaint through `_panel_content_is_unchanged`).
  - Skip widgets with no rendered agents.
  - Report the key when `getattr(widget, "_rendered_group_folds", None)` differs from
    `group_fold_snapshot(panel_fold_registry(self, key))`.
  - No disk I/O, no tree building: this runs on every finalize (tui_perf rules 1, 6, and
    8).
- In `_try_refresh_agents_display_incremental_impl`, after `_sync_panel_group()` /
  `_snap_focus_after_agents_fold_restore()` and before panel rebuild keys are assembled:
  - Compute the stale keys.
  - Record each one with
    `self._record_display_panel_rebuild_fallback("group_fold_change", key)` so per-panel
    attribution stays observable.
  - Merge them into the forced rebuild set: `forced_rebuild_keys = scope.keys | stale`.
    They then flow into `panel_rebuild_keys` and `rebuild_only_keys`, which also rules
    out the in-place row-insert shortcut.
  - `_refresh_affected_panel_widgets` then repaints only those panels. The changed
    registry version makes `panel_paint_key` differ, so `_panel_content_is_unchanged` is
    false and `update_list` runs. `_reapply_panel_heights()`, which already runs
    afterwards, shrinks or grows the panel.
- Do not touch the `diff.has_changes` short-circuit in `panel_rebuild_scope`. It stays a
  pure roster predicate; fold staleness is a separate widget-vs-state check, parallel to
  `_agent_display_widgets_match_grouping_mode`.

### 3. Trace/docs plumbing

- Add `"group_fold_change"` to `AgentRefreshFallbackReason` and
  `ALL_AGENT_REFRESH_FALLBACK_REASONS` in
  `src/sase/ace/tui/actions/agents/_refresh_trace.py`.
- Mention it next to the other per-panel `display_fallback` reasons in
  `docs/perf_runbook.md`, which lists `panel_membership_change`,
  `status_membership_change`, and `workflow_tree_change`.

Leave the call sites (`_collapse_group_fold`, the `_expand_fold` banner branch, the
panel-sweep and fold-hint paths, `_install_loaded_agents_fold_state`) behaviorally
unchanged.

## Tests

1. Display-layer regression, in a new
   `tests/ace/tui/test_agent_display_group_fold_repaint.py` or in
   `tests/ace/tui/test_agent_display_diff_grouping.py`. Use `_DisplayDiffApp`,
   `_apply_roster`, `_widget_sel`, `_panel_attributions`, and `_display_costs`.
   - BY_STATUS, one `epic` panel with RUNNING + DONE agents:
     - Collapse `("Done",)` via `panel_fold_registry(app, "epic")` and apply the
       unchanged roster. The panel's `update_list_calls` goes to 1, `full_rebuilds`
       stays 0, `_panel_attributions` contains the epic panel with `group_fold_change`,
       and the widget no longer holds agent rows for the DONE members (for example,
       check the option count or `_row_by_agent_idx`; the `("Done",)` banner row is
       present and collapsed).
     - Expand it again and apply: another repaint, and the rows are back.
   - A second apply with no fold change triggers no extra `update_list` (the perf guard
     is preserved).
   - A sibling panel whose registry did not change is not repainted.
   - A whole-panel-collapsed panel is ignored.
   - At least one STANDARD-mode case with a project/name banner (the stub fold tests use
     keys like `("proj", "demo", "coder")`), since STANDARD is also on the incremental
     path.
   - A case where the registry `version` changes but the logical set does not (collapse
     then expand before the apply): no repaint.
2. Widget-level checks, in the existing `tests/ace/tui/widgets/` agent-list tests:
   - `update_list(..., fold_registry=r)` records
     `_rendered_group_folds == r.snapshot()`.
   - A successful `try_insert_rows` records it; a declined one leaves it unchanged.
   - `render_collapsed` resets it to `None`.
3. Mounted end-to-end regression, with no `visual` marker so `just test` runs it. Model
   it on `tests/ace/tui/test_agents_panel_fold_mounted.py` and the flow in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_group_lane_collapse.py`, using
   `AcePage`, `patch_startup_loaders`, `patches`, `wait_for_startup`, and
   `wait_for_visual_idle`.
   - Switch to BY_STATUS (`o`, `s`), select an agent in a multi-row status group, and
     press `H` until the group registry collapses.
   - Assert that the focused `AgentList` actually renders the collapsed banner: its
     `_banner_row_by_key` contains the group key, and no option rows remain for the
     group's members.
   - Assert that no `L` or other full repaint was needed.
   - Optionally, press `l` on the collapsed banner and assert that the rows come back.
4. Run the existing guards unchanged:
   - `tests/perf/test_agents_display_rebuild_guard.py`, especially
     `test_by_status_unchanged_membership_finalize_does_not_rebuild_panels`.
   - `tests/ace/tui/test_agent_display_diff*.py`.
   - `tests/ace/tui/test_agents_refresh_trace.py`.
   - `tests/ace/tui/test_agent_fold_transitions_*.py`.
   - `tests/ace/tui/test_agent_panel_hint_collapse*.py`.
   - `tests/ace/tui/widgets/test_agent_list_try_insert_rows.py`.
   - `tests/ace/tui/widgets/test_agent_list_try_remove_rows.py`.

## Verification

- Run `just fix`, then `sase tool run check`. Read the `lint_and_test` memory first; run
  `just install` if the workspace venv is stale.
- This change should not alter any existing PNG golden, because it only fixes a stale
  repaint that the goldens never captured. Do not run `just check-full` unless
  instructed.
- Optional manual confirmation: in `sase ace` → Agents with `group: by status`, press
  `H` on a row in a multi-agent status group. The group should draw collapsed
  immediately, with no need for `L`.
