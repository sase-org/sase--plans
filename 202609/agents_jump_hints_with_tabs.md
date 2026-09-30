---
tier: tale
title: Keep Agents-tab jump hints truthful when agent tabs are on
goal:
  With agent sub-tabs (including machine tabs), every hint painted after `'` — tab
  chips, panel titles, banners, and rows — matches what its keypress does, stays stable
  while hints are shown, survives in-place row patches, and exiting jump mode restores
  the normal footer.
size: medium
proposed_by: bbugyi200.apollo.3d
create_time: 2026-09-30 08:15:17
status: wip
---

# Keep Agents-tab `'` jump hints truthful when agent tabs (machine tabs) are on

## Problem

On the Agents tab, pressing `'` paints jump hints on the tab-strip chips, the tribe
panel titles, collapsed banners, and agent rows. When agent sub-tabs are in use,
especially machine tabs (`⌂ athena` / `apollo`), the hints act up in several ways:

- Painted row labels stop matching what a keypress does. Pressing a painted label can
  jump to a _different_ agent.
- Labels show up out of order. For example, `[w]` appears between `[s]` and `[t]` after
  a row moves from Queued to Running.
- A panel title loses its hint chip. For example, `@epic` never showed its `[p]`, while
  `@research` kept `[Q]`.
- If `'` is pressed while the TUI is still loading, the tab chips get no labels at all.
  The digits `0`/`1` then go to the first panel and row instead of to the tabs.
- Separately: `<esc>` clears the hints but leaves the footer stuck on
  `JUMP ' first · <esc> cancel`.

In a quiet steady state, the painted labels and dispatch agree. That matches the user's
screenshot of the athena TUI. The breakage shows up as soon as a refresh lands while
hints are on screen. With machine tabs on, that happens constantly.

## Diagnosis (verified)

Reproduced in three ways:

1. **Live `sase screenshot --host athena` captures** (real roster, HEAD `3e6646f8c`):
   - The first `'` gave labels without the tab-chip hints.
   - The `@default` and `@epic` titles had no chip; `@default` got its chip later.
   - A few seconds later, still in JUMP mode, the row labels were out of order.
2. **Deterministic AcePage harness:**
   - Setup: machine mode on, one remote alias, `BY_STATUS` grouping, three tribe panels.
   - Steps: enter jump mode, prepend one local agent to `_agents_local_with_children`,
     then call
     `_reproject_agents_from_current_mode(source="fleet_refresh", force=True)`.
   - Result: 6 of 12 agent hints were painted on one agent but dispatched to another.
     For example, painted `a` sits on `e5`, but `_entry_jump_hint_to_target["a"]`
     resolves to `e4`.
3. **Footer check:** after `escape`, `KeybindingFooter._last_layout_inputs` still has
   mode label `JUMP`. This happens with and without tabs. After selecting a hint, the
   footer is restored correctly.

### Root cause 1 (primary, tab-specific): the hint overlay is a one-shot snapshot keyed by volatile indices, and machine tabs add a refresh source that is not gated during jump mode

- `EntryJumpModeMixin._prepare_agents_jump_maps`
  (`src/sase/ace/tui/actions/navigation/_entry_jump_mode.py`) allocates hints once, when
  `'` is pressed. The targets are:
  - `("tab", AgentTabKey)`, from `_tab_jump_targets()` at that moment;
  - `("panel", key)`;
  - `("banner", panel_idx, group_key)`;
  - `("agent", global_idx)`, where `global_idx` indexes the current `self._agents`.
- Every repaint while jump mode is active reuses `_entry_jump_index_to_hint` and the
  related maps:
  - `_refresh_agents_display` in `actions/agents/_display.py`;
  - `_refresh_affected_panel_widgets` in
    `actions/agents/_display_panel_widgets_refresh.py`;
  - the strip descriptors, via `_agent_tab_jump_hints`. Dispatch
    (`_handle_entry_jump_key` in `_entry_jump_dispatch.py`) resolves `("agent", idx)`
    against whatever `self._agents` currently is.
- The auto-refresh tick deliberately skips while `_entry_jump_mode_active`
  (`event_refresh/_auto_refresh_surfaces.py`). The fleet feed does not:
  - Machine tabs mean a fleet is configured, so remote rows keep arriving.
  - `AgentFleetRefreshMixin._apply_fleet_projection`
    (`actions/agents/_fleet_refresh.py`) calls `_reproject_agents_from_current_mode`
    (`_fleet_projection.py`).
  - That rebuilds `_agents_with_children` and `_agents`, then runs
    `_finalize_agent_list`, which re-indexes tabs, re-scopes the active tab, and
    repaints panels using the stale maps.
  - The only deferral is `_defer_fleet_projection_apply_if_navigating`, which covers the
    j/k nav gate, not hint modes.
  - `_agents_projection_signature` covers every field, so any running remote agent
    triggers a re-apply.
  - Other reload paths can also land mid-mode: an in-flight loader finalize, the startup
    tier-1 → full load, and tab index reconcile/rescope.
- Consequences:
  - Global indices shift, so painted labels and dispatch disagree.
  - Rows re-bucket, so labels come out of order.
  - Newly arrived rows and tabs get no labels.
  - A `'` pressed before the tab catalog has two or more entries never gets tab targets,
    because the maps are never re-derived when the strip appears.

### Root cause 2: in-place row patches repaint the panel title without the hint chip

`_try_patch_agent_row` (`actions/agents/_display_panel_patches.py`, the
`_set_agent_panel_title(... self._agent_panel_title(agent_panel_key, slot.agents, merge_tribe_panels=...))`
call near the end) builds the title without `panel_jump_hints`,
`isolation_restore_marked`, and `fold_restore_marked_count`.

- Any live row patch during jump mode or fold-hint mode erases that panel's `[x]` title
  chip. Examples: runtime or status ticks, the live pencil hint, notification unread
  updates.
- The other two title writers already do this correctly, via
  `_active_panel_title_jump_hints()`:
  - `_refresh_agent_panel_titles` in `_display_panel_collection.py`;
  - the focus-change path in `_display_panel_layout.py`.

### Root cause 3 (adjacent, not tab-specific): `<esc>` never restores the footer

- On the Agents tab, `_exit_entry_jump_mode` → `_refresh_agents_jump_hint_display()`
  returns through the selective `_refresh_affected_panel_widgets` path. That path never
  refreshes the footer.
- Hint-selection exits only look right because they also call
  `_refresh_agent_focus_detail` or change the selection.

## Implementation

This is presentation-only Textual state (a transient hint overlay), so all changes stay
in this repo. No `sase_core` changes. No keymap or `default_config.yml` changes.

### 1. Defer fleet projection application while a hint mode is active

In `actions/agents/_fleet_refresh.py`:

- Where `_defer_fleet_projection_apply_if_navigating` is consulted (both call sites),
  also defer when `_entry_jump_mode_active` or `_panel_fold_hint_mode_active` is set.
- Unlike the nav gate, a hint mode has no time bound, so don't use a timer. Store only
  the newest pending apply, e.g.
  `self._agents_fleet_hint_deferred_apply = (projection, config, generation, source)`.
  Always keep the latest; drop older ones.
- Leave `_agents_fleet_loading` / header semantics as they are for the nav-gate deferral
  (`deferred_apply` handling).
- Add `_flush_hint_deferred_fleet_projection()`. It pops the pending tuple and routes it
  through `_apply_deferred_fleet_projection`, which already drops stale generations and
  re-checks the nav gate.
- Call the flush at the end of `_exit_entry_jump_mode` (the Agents branch, after the
  hint display refresh) and at the end of panel fold-hint teardown
  (`_teardown_panel_fold_hint_mode` in `actions/agents/_panel_hint_folding.py`).
- A tab-chip jump's `_switch_agents_tab` runs before `_exit_entry_jump_mode`, which is
  fine: the flush happens after the switch.
- `remote_attention` / `remote_mutation` sources are deferred the same way. Their
  `force=True` is preserved when the flush applies them.
- Initialize the new attribute wherever fleet state is initialized (look for
  `_agents_fleet_refresh_generation` init) so `getattr` defaults aren't needed
  everywhere.

### 2. Make the jump overlay self-consistent when the roster changes anyway

Other paths can still replace `self._agents` mid-mode (in-flight loader finalize,
startup tier-1 → full load, tab reconcile/rescope, `remove_agents_from_views`). Make
painted labels and dispatch agree by construction.

- In `_prepare_agents_jump_maps`, also record an **allocation token**: a cheap tuple
  such as
  `(id(self._agents), len(self._agents), tuple(panel_group.panel_keys), tuple(entry.key for entry in self._agent_tab_catalog_view()), self._agent_tab_strip_visible(), self._grouping_mode)`.
  Also record an **identity-per-hint map** for agent targets:
  `self._entry_jump_agent_identity_by_hint[hint] = self._agents[idx].identity`. Clear
  both in `_exit_entry_jump_mode`, and declare them in `actions/navigation/_types.py`
  and the state-init module (`actions/_state_init_navigation.py`) next to the other
  `_entry_jump_*` fields.
- Add `_ensure_agents_jump_maps_current()` to the entry-jump mixins:
  - If jump mode is active on the Agents tab and the token differs from the current one,
    re-run `_prepare_agents_jump_maps()`. That also resets `_entry_jump_pending_prefix`.
  - If re-preparation returns False, exit jump mode.
  - Call it before the hint maps are read for painting:
    - at the top of the jump-hint branch in `_refresh_agents_display` (`_display.py`,
      where `_entry_jump_index_to_hint` / `_banner_to_hint` / `_panel_to_hint` are
      copied when `_entry_jump_mode_active`);
    - in `_refresh_affected_panel_widgets` (`_display_panel_widgets_refresh.py`), same
      spot;
    - in `_descriptors_for_strip` / `_refresh_agent_tab_strip`, so the chips pick up
      labels once the catalog reaches two tabs.
  - Keep it cheap. Compare the token first; only walk trees when it changed. Read the
    `tui_perf.md` memory before touching these paths.
- In `_handle_entry_jump_key`, when the resolved target is `("agent", idx)`:
  - Resolve the agent by the recorded identity: find the current index whose `.identity`
    equals `_entry_jump_agent_identity_by_hint[hint]`.
  - If that identity is gone, exit jump mode without moving.
  - This makes dispatch correct even between a mutation and its repaint.
  - For `("banner", panel_idx, group_key)` targets, validate that `panel_idx` still
    points at the same panel key recorded at allocation. Record a small
    `hint → panel_key` map, or switch the banner payload lookup to use the panel key.
    Exit without moving if it doesn't.
  - Tab targets are already stable `AgentTabKey`s.
- Keep existing behavior otherwise:
  - The `'`/`''` back and first semantics;
  - the anchor stacks;
  - the `JumpHintMatchOutcome` handling;
  - tab chips taking the first labels, in strip order.

### 3. Keep the panel title chip on row patches

- In `_try_patch_agent_row`, build the title exactly like `_refresh_agent_panel_titles`
  does. Pass:
  - `panel_jump_hints=self._active_panel_title_jump_hints()`;
  - `isolation_restore_marked` (from `_panel_isolation_marked_keys()`);
  - `fold_restore_marked_count` (from `_panel_fold_restore_marked_keys()`).
- Prefer extracting one small helper, e.g.
  `_agent_panel_title_for_key(key, panel_agents)` on `PanelCollectionMixin`, and use it
  from all three title writers:
  - `_refresh_agent_panel_titles`;
  - the focus path in `_display_panel_layout.py`;
  - `_try_patch_agent_row`.
- Keep the declared-only stubs in `_display_panel_patches.py` /
  `_display_panel_state.py` in sync, so mypy still resolves the attribute.

### 4. Restore the footer when jump mode exits

- In `_exit_entry_jump_mode`, when `current_tab == "agents"`, call
  `_refresh_agent_footer_bindings_only()` after `_refresh_agents_jump_hint_display()`
  (guard with `getattr`/`callable` like the surrounding code).
- `_apply_agent_footer_update` already renders the normal footer once
  `_entry_jump_mode_active` is False.
- Leave the Artifacts and Services branches alone unless a test shows the same stale
  footer there.

## Tests

Add focused tests. Reuse the AcePage startup helpers from
`tests/ace/tui/visual/_ace_png_snapshot_helpers.py`. The `-m visual` marker is not
needed; these are behavior tests:

- `patch_startup_loaders`;
- monkeypatch `agent_tabs_settings.agent_tabs_view_config` to an `AgentTabsViewConfig`
  with `machine_mode=True`, one `machine_order` alias, and `local_machine_name`;
- monkeypatch `agent_tab_descriptors.resolve_agent_tab_style_inputs`;
- monkeypatch `grouping_strategy.load_agent_grouping_mode` → `GroupingMode.BY_STATUS`;
- three tribes;
- a few remote agents with `fleet_origin_alias` / `fleet_origin_installation_id`.

Keep new modules under the repo's per-file size norms (split helpers into a `_...`
helper module if needed).

1. **Fleet projection is deferred during jump mode.**
   - Enter jump mode (`'`).
   - Drive a fleet apply through the real deferral path. Call the method that consults
     `_defer_fleet_projection_apply_if_navigating` with a projection or local cache that
     adds a row.
   - Assert the hint maps and painted labels are unchanged and `self._agents` did not
     grow.
   - Press `escape`, then assert the new row is present.
   - Repeat, exiting via a row hint instead of `escape`.
2. **Painted labels always equal dispatch targets.**
   - Enter jump mode.
   - Force a roster replacement that bypasses the deferral:
     `_reproject_agents_from_current_mode(source="fleet_refresh", force=True)` after
     prepending a local agent.
   - For every `("agent", …)` hint, the row painted with `[h]` (scan `AgentList` options
     across panels) must be the agent that pressing `h` selects.
   - Also press one painted label and assert the selected agent is the labeled one.
   - Before the fix this reproduces 6 mismatches.
3. **Tab chips get labels once the catalog grows.**
   - Enter jump mode while the tab index has one entry: strip hidden, no `("tab", …)`
     targets.
   - Install a two-entry index and trigger the normal display refresh.
   - Assert the chips now show `[0]`/`[1]`, `_entry_jump_hint_to_target` contains both
     tab targets, and pressing `1` switches tabs.
4. **Row patch keeps the title chip.**
   - While jump mode (and, separately, fold-hint mode) is active, call
     `_try_patch_agent_row` for an agent in a non-focused panel.
   - Assert that panel's `border_title` plain text still contains its hint.
   - The `_JumpApp` harness in `tests/ace/tui/test_agent_jump_hint_paint.py` is a good
     fit.
5. **Escape restores the footer.**
   - After `'` then `escape` on the Agents tab, `#keybinding-footer`'s
     `_last_layout_inputs[1]` is not `"JUMP"`.
   - Cover both one-tab and two-tab setups.
6. **Identity-safe dispatch.**
   - Unit-test `_handle_entry_jump_key` with a harness where `_agents` is reordered
     after allocation but before any repaint.
   - The agent hint must select the originally labeled identity.
   - A removed identity must exit without moving.

Re-run and keep green the existing suites that cover this code:

- `tests/ace/tui/test_jump_*.py`
- `tests/ace/tui/test_agent_jump_hint_paint.py`
- `tests/ace/tui/test_agent_tab_*.py`
- `tests/ace/tui/test_agents_fleet_refresh_laziness.py`
- `tests/ace/tui/test_agent_panel_hint_*.py`
- `tests/ace/tui/test_agents_live_hint_refresh.py`
- `tests/ace/tui/test_member_jump_*.py`

Update any assertions that encoded the old stale-index behavior only if they were
asserting a bug.

## Verification

- Follow the `lint_and_test.md` memory (read it before finishing). Run the targeted
  pytest selections above, then `just check` through `sase tool run`.
- If any Agents-tab PNG goldens change (none are expected), run the targeted
  `just fix-tui-screenshots` per `tui_screenshot.md` and inspect the report.
- Manual/live check. Use `sase screenshot --host athena --keep … -- -t agents`, which
  launches its own TUI window. Then press `'`, wait through a fleet refresh (about 10
  seconds), and recapture. Confirm:
  - labels are stable and in order;
  - titles keep their chips;
  - chips show `[0]`/`[1]`;
  - `<esc>` restores the normal footer.
- Only use `sase screenshot --window` on windows created by `sase screenshot`. It sends
  `SIGUSR2`, and a TUI started any other way has no handler for it, so the signal
  terminates that TUI.
