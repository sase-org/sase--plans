---
tier: tale
title: Agents-tab panel focus follows the selection on background refresh
goal:
  Background Agents-tab refreshes never move the user's cursor to another node while the
  selected row is still rendered, and navigation made during a slow load survives the
  apply.
size: medium
proposed_by: bbugyi200.athena.0oq
create_time: 2026-09-21 14:21:51
status: wip
---

# Agents tab: panel focus follows the selection, and background refreshes stop moving the cursor

## Context

The research report
`research:202609/agents_tab_panel_focus_snap/agents_tab_panel_focus_snap.md`
(consolidated from reports `__a` and `__b`) traced the Agents-tab "cursor jumps to some
other node" bug. It used athena's TUI trace, where 76 of 76 non-adjacent cursor moves
followed `agents.refresh_work` and none followed a widget highlight echo. The Agents tab
tracks two selection-like states: the selected agent (`current_idx`) and the focused
tribe panel (`_panel_group.focused_idx`). Explicit navigation makes panel focus follow
the cursor. Every background display refresh does the reverse and moves the cursor to
the focused panel.

I checked each finding against current `master`, 52 commits past the report's revision
`e1ba4851c`. None of those commits touch the code below.

| Report item                                                                                                            | Still relevant?                                                                                                                                                                       | Decision                                                                                        |
| ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| §1 / §7.1: `_sync_panel_group` snaps the cursor to the focused panel (M1b, M1c)                                        | **Yes.** Lines 204-214 of `src/sase/ace/tui/actions/agents/_display_panel_collection.py` are unchanged.                                                                               | **Fix** (Step 1)                                                                                |
| §7.2: panel group built from occupancy only, not sticky `live_keys` (M1a)                                              | Yes, the code is unchanged.                                                                                                                                                           | **Do not implement.** See "Out of scope".                                                       |
| §5.1 / §7.3: stale `selected_identity` re-applied after three more awaits, and the drift token compares it with itself | **Yes**, in `_loading_disk_full.py`. The same bug is in `_loading_disk_delta.py`, which the report missed: it captures at ~L98 and then awaits three times before the apply at ~L157. | **Fix** (Step 2)                                                                                |
| §5.2 / §7.4: refresh finalize never passes `prior_pos`, so a lost identity falls back to a global-index clamp          | **Yes.** `_apply_loaded_agents_prepared_inner` calls `_finalize_agent_list` without `prior_pos`.                                                                                      | **Fix** (Step 3)                                                                                |
| §5.3 / §7.5(b): `Agent.identity` is unstable for container rows                                                        | Yes, but it is the lowest priority, needs a model-level identity change, and carries medium risk.                                                                                     | **Out of scope.** File a follow-up.                                                             |
| §3: A's H3, `ProgrammaticSelectionGuard` for `SelectionChanged`                                                        | The trace refutes it.                                                                                                                                                                 | **Do not action.**                                                                              |
| §6 / §7.5: athena's roughly 39 duplicate daemon sets (the amplifier)                                                   | These are the leaked pytest `sase service run` hosts.                                                                                                                                 | **Owned by another in-flight plan** (`plan:202609/leaked_test_service_hosts.md`). Do not touch. |

The fix is presentation-only Textual state: panel focus, cursor index, and the refresh
apply seam. Under `rust_core_backend_boundary` nothing here goes into `sase-core`.

## Coordination with the in-flight `leaked_test_service_hosts` plan

Another agent is implementing that plan concurrently. To avoid conflicts, this work MUST
NOT edit:

- anything under `src/sase/service/` (`control.py`, `platform_runner.py`,
  `platform_models.py`, `host.py`);
- `src/sase/ace/tui/actions/axe_display/` (the `_loader_refresh*` modules);
- the four TUI tests that plan may change:
  `tests/ace/tui/test_startup_stopwatch_live_update.py`,
  `tests/ace/tui/test_residual_freeze_soak.py`,
  `tests/ace/tui/test_artifacts_agents_loading.py`, and
  `tests/ace/tui/test_agents_pane_mount.py`;
- the service-host tests `tests/service/*`, `tests/test_mobile_gateway.py`,
  `tests/test_chat_install.py`, `tests/ace/tui/test_axe_startup_host_lifecycle.py`, and
  `tests/ace/tui/actions/test_service_host_keys.py`.

This work also does not reap processes or perform any other ops remediation. If
`just check` fails in one of those areas, treat it as that plan's in-flight work: re-run
it once, and do not "fix" it here. Any new AceApp-booting test added by this plan should
pass `auto_start_axe=False` (or reuse a harness that already stubs host startup) so it
cannot contribute to the host leak.

## Step 1: invert the dependency in `_sync_panel_group` (main fix)

File: `src/sase/ace/tui/actions/agents/_display_panel_collection.py`, the tail of
`_sync_panel_group` (after `panel_index = self._agent_panel_index()`).

Today, when the selected row is not in the focused panel's rendered slice, the code
calls `_snap_current_idx_to_focused_panel`. That moves the cursor to the first row of
the focused panel, or to row 0. Replace it with this decision order:

1. Compute `selected_key = keys_per_agent[self.current_idx]` when `current_idx` is in
   range. The selection is **renderable** when
   `panel_index.local_idx_for(selected_key, self.current_idx) >= 0`.
2. If the selection is renderable and already in the focused panel, do nothing. This is
   today's no-op case.
3. If the selection is renderable, `selected_key` is in `self._panel_group.panel_keys`,
   and focus is **not deliberately parked elsewhere** (defined below), then **focus
   follows the selection**:
   `self._panel_group.focused_idx = self._panel_group.panel_keys.index(selected_key)`.
   Leave `current_idx` unchanged. Also reset `self._expanded_panel_focus = False`,
   because focus is now on a row and not on a whole panel. Drop nothing from
   `_panel_selection_memory`.
4. Otherwise, keep today's behavior and call `_snap_current_idx_to_focused_panel(...)`.
   This covers a genuinely unrenderable selection, such as the hidden `STARTING` row
   guarded by `test_sync_panel_group_snaps_selection_off_hidden_starting_row`, an
   out-of-range `current_idx`, and deliberately parked focus.

**Deliberately parked focus** means the user put focus on something other than a row,
and a refresh must not undo that. Treat focus as parked when the previously focused
panel is still present (`focused_key == prev_focused`, i.e. `from_panel_keys` did not
reset it) **and** any of the following holds:

- `getattr(self, "_expanded_panel_focus", False)` is true. This is whole-panel focus
  from `_focus_whole_panel_target` in `_panel_navigation.py`, which intentionally leaves
  `current_idx` pointing into the old panel when the target panel renders no rows.
- The focused panel renders no rows: `rendered_panel_slice(self, focused_key)[0]` is
  empty. This is a collapsed strip reached with lowercase j/k whole-panel cycling.
- `getattr(self, "_current_group_key", None)` is not `None`. This is banner focus inside
  the focused panel.

When `from_panel_keys` reset focus because the old focused panel vanished (M1a), focus
is never parked. The selection wins, so the cursor stays put instead of jumping to the
top of panel 0.

Before finalizing these predicates, check them against the explicit-navigation writers
of `focused_idx` (`_panel_navigation.py`, `_event_widgets.py`, `_unread_navigation.py`,
`_folding_agent_groups.py`, `_revive_state.py`, `navigation/_entry_jump_dispatch.py`,
`navigation/_member_jump.py`, `agent_workflow/_kill_last_launch.py`). Refine the
"parked" predicate if any of those paths legitimately leaves `current_idx` outside the
focused panel. Keep the rule simple and add a short comment above the new branch
explaining it: the selection is the user's intent, and panel focus is derived from it
except when focus is parked on a non-row target.

Do **not** change `AgentPanelGroup.from_panel_keys` or the `live_keys` / fold-intent
retirement code above this block.

## Step 2: re-capture the live selection at the apply seam (async load paths)

This implements tui_perf rule 4 ("re-capture UI state after every await") at the last
seam.

Files:

- `src/sase/ace/tui/actions/agents/_loading_disk_full.py`: `on_agents_tab` and
  `selected_identity` are captured at ~L274-279, before three awaits (worker prep, the
  content search index, and the finalize-plan attach).
- `src/sase/ace/tui/actions/agents/_loading_disk_delta.py`: the same pattern, captured
  at ~L98-103 and followed by awaits before `_apply_loaded_agents_prepared` at ~L157.

Change:

- Extract one small helper on the loading mixin, for example
  `_capture_agents_apply_selection() -> tuple[bool, identity | None]`. It holds the
  existing capture logic: current tab, plus the identity at `current_idx` on the Agents
  tab or `_agents_last_identity` off it. Use it for the existing early capture, which
  still feeds the worker snapshot and `record_agents_tab_loader_result`.
- Call it again **synchronously, immediately before**
  `self._apply_loaded_agents_prepared(...)` in both files, with no await between the
  re-capture and the apply. Pass the fresh `on_agents_tab` / `selected_identity` to the
  apply.
- The worker's precomputed plan was built from the early capture. Because the fresh
  identity now reaches `_select_finalize_plan`, the stale-token comparison
  (`make_finalize_stale_token` over the live snapshot against the worker snapshot's
  token) now detects navigation made during the load. The plan is then discarded and
  selection is recomputed on the UI thread. This is the intended correctness-over-reuse
  tradeoff. It only costs extra work when the user actually moved during a load. The
  token already includes `on_agents_tab`, `selected_identity`, and `prior_visual_row`
  (`_loading_compute_finalize.py`), so no token change is needed.

Optional, only if it stays small: mirror the fleet path's apply-seam `NavigationGate`
deferral (`_fleet_refresh.py`, around the apply) in the main refresh. Skip it if it adds
a new refresh code path, which tui_perf rule 5 forbids.

## Step 3: give the refresh path a real neighbor fallback (`prior_pos`)

File: `src/sase/ace/tui/actions/agents/_loading_apply.py`,
`_apply_loaded_agents_prepared_inner`.

- At the top of the method, **before** `self._agents` is replaced, capture
  `prior_pos = self._capture_focused_visible_pos()` when `on_agents_tab` is true, and
  `None` otherwise. `_capture_focused_visible_pos` is in `_navigation_order.py` and is
  already used by kill and dismiss. Use `getattr` or a callable check if bare test hosts
  lack the method, matching the `_proc_shell_dismiss.py` pattern.
- Pass `prior_pos=prior_pos` to `self._finalize_agent_list(...)`. `finalize_agent_list`
  and `_apply_finalize_plan` in `_loading_finalize.py` already route through
  `_restore_focus_after_removal(prior_pos)`, but only when the selected identity was
  **not** restored. The common case, identity found, is unaffected.
- Verify that `_panel_navigation_stops()` (and its `_nav_stops_cache`) reflects the
  **new** roster when `_restore_focus_after_removal` runs from the refresh finalize, the
  same way it does for kill and dismiss. If the cache is keyed on stale state,
  invalidate it before the restore. Keep the capture cheap: it runs once per apply and
  must not rebuild the nav stops when they are already cached.

## Step 4: regression tests

Add focused unit tests near `tests/ace/tui/test_agent_panel_index_integration.py`,
reusing its `_Bare` host, or in a new sibling test module:

1. **M1b, key change under the selection.** Split tribe panels A and B. Focus is on A
   and the selection is a row in A. Rebuild `_agents` so the selected agent's `tribe` is
   now B while A still has rows, then call `_sync_panel_group()`. Assert that
   `current_idx` still points at the same agent identity and that
   `_panel_group.focused_key == B`.
2. **M1a, focused panel vanishes.** Focus is on panel C, and `current_idx` points at a
   rendered row in panel B, for example after an identity restore. C then drops out of
   the roster. After `_sync_panel_group()`, the cursor stays on the B row and focus
   moves to B. It must not jump to the first row of panel 0.
3. **Hidden STARTING row.** The existing
   `test_sync_panel_group_snaps_selection_off_hidden_starting_row` must stay green and
   unmodified.
4. **Parked focus is preserved.** (a) With whole-panel focus
   (`_expanded_panel_focus = True`) on a panel, plus a `current_idx` pointing into
   another panel, a sync does not move focus back. (b) The same holds for focus on a
   collapsed or empty panel strip. (c) The same holds with `_current_group_key` set. In
   each case, assert the pre-change snap behavior holds.
5. **Apply-seam re-capture (Step 2).** For both the full and the delta async load paths,
   move the selection (change `current_idx` to a different agent) after the early
   capture but before the apply. For example, monkeypatch the awaited worker step
   (`asyncio.to_thread` target or `_prepare_agent_content_search_index_async`) to move
   the cursor. Assert that the user's new selection survives the apply. Reuse whatever
   harness existing tests for `_loading_disk_full` / `_loading_disk_delta` use, found by
   grepping `tests/ace/tui` for them.
6. **Refresh `prior_pos` fallback (Step 3).** A refresh removes the selected agent from
   the middle of a panel. Assert that the cursor lands on the row that was visually
   below it in the same panel (the `_restore_focus_after_removal` semantics), not on the
   global-index clamp.

Also, if it fits cleanly, add one arrival window driven by the real
`_apply_loaded_agents_prepared` to the `tests/ace/tui/_epic_arrival_frames.py` harness,
or to a new sibling test that reuses it. In that window the parked selected agent's
tribe changes (M1b). Assert that the **parked identity is still selected** after the
window. Do not change the expectations of existing windows in
`tests/ace/tui/test_epic_panel_arrival_frames.py`. If adding the window would perturb
them, put it in a new test module instead.

## Out of scope (record, don't implement)

- **§7.2, the panel group built from sticky `live_keys`.** Once Step 1 lands, a vanished
  focused panel no longer moves the cursor as long as the selected row still exists. If
  the selected row itself left the roster, no panel-group change can keep it selected.
  Meanwhile, adding row-less sticky strips to `_panel_group` would change
  panel-navigation stops, J/K skip logic, and jump hints, all of which assume group keys
  are navigable. The benefit is small and the behavioral surface is large.
- **§5.3 / §7.5(b), a stable `Agent.selection_key` for container rows.** It needs a
  model-level identity change. After Steps 1-3, the failure is survivable (neighbor
  fallback in the same panel). File it as a follow-up task bead via `/sase_new_task`,
  sized `large`, and cite the research report.
- **A's H3 (`ProgrammaticSelectionGuard`)**, which the trace refutes.
- **Ops remediation of athena's duplicate daemons.** The `leaked_test_service_hosts`
  plan owns it. Restarting the athena TUI to pick up these fixes is the user's call.

## Verification

- Read `sase memory read lint_and_test.md` and follow it. Run `just install` if needed,
  then `just fix`, then `sase tool run check` (`just check`). Do not run
  `just check-full`.
- This change does not alter rendered layout, so no PNG golden updates are expected. If
  `just check` or a targeted visual run shows golden diffs, inspect them rather than
  blindly regenerating them.
- Report which report items were fixed, which were deliberately skipped (with the
  reasons above), and the follow-up bead ID for §5.3.
