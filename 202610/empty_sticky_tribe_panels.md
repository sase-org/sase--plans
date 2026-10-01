---
tier: tale
title: Stop empty tribe panels from lingering on the Agents tab
goal:
  "An Agents-tab tribe panel with zero rendered rows is never left mounted: rows placed
  elsewhere retire their old panel in the same sync, and unexplained disappearances
  bridge for at most a few seconds before the empty strip unmounts, while the sase-13i
  anti-flicker guarantee still holds."
size: medium
proposed_by: bbugyi200.athena.0us
create_time: 2026-10-01 10:44:45
status: wip
---

# Stop Empty `@tribe · 0` Panels From Lingering On The Agents Tab

## Problem

The Agents tab sometimes shows tribe panels with no rows, rendered as collapsed
`@default · 0` / `@epic · 0` strips under populated panels. In the user's screenshot,
`@research` (a `research.34` clan launched 1m44s earlier) and `@job` have rows while
`@default · 0` and `@epic · 0` are mounted with nothing in them. An empty panel should
never stay on screen.

## Root cause

Panel occupancy (`panel_keys_for` in `src/sase/ace/tui/models/agent_panels.py`) only
ever yields keys that have rendered rows. Empty strips come from the **session-sticky
panel store** in `src/sase/ace/tui/actions/agents/_display_panel_collection.py`. The
widget layer mounts `occupancy ∪ sticky` (`_sorted_widget_panel_keys` →
`_widget_panel_keys`) and renders every sticky key that has no rows as a collapsed
0-count strip.

The store (`_session_mounted_panel_identities`: scoped panel key → identities that
rendered under it) came from sase-13i.2 to stop the `@epic` panel flickering on
transient zero-row applies. It holds a key until every recorded identity is _proven_
gone. Today the only proofs are:

1. an explicit TUI dismiss / kill / proc-shell dismiss
   (`_retire_session_mounted_identities`);
2. membership in `_dismissed_agents`;
3. the same identity being **rendered** under a different key in the active scope
   (`_remember_session_mounted_occupancy`);
4. a `complete_history` apply under the committed query whose roster omits it
   (`_reconcile_session_mounted_for_apply`). These applies are rare: startup, the
   quiet-time Tier 2 reconcile, and a manual full-history refresh;
5. a committed-query change, which clears the store.

Any other way a row leaves a panel keeps the key pinned until a restart. Those exits
fall into two classes, and three of them were reproduced in a scratch harness built on
the `_FakeApp` from `tests/ace/tui/_agent_panels_display_helpers.py`:

- **Still loaded, but placed elsewhere or not rendered.** The row is still in
  `_agents_with_children` (the tab- and fold-independent roster), but it never renders
  under a new key in this scope, so proof 3 never fires. Cases:
  - It is folded into a collapsed clan, session, or parent whose anchor has another
    tribe. _Reproduced:_ widgets stay `['research', 'epic']` with rows only in
    `research`.
  - It moved to another agent tab, either via `sase agent tab` or through
    machine/dispatch placement. _Reproduced:_ `['research', 'epic']`.
  - It is hidden by the committed query (e.g. a status filter after a status change), or
    went back to `STARTING`.
- **Identity replaced or vanished without a dismissal record.** The old identity is
  absent from the roster forever, so only the rare complete-history apply can retire it.
  Cases:
  - The synthetic clan container identity `(RUNNING, "clan:<name>", generation)` changes
    when a launching clan's generation and clan tribe resolve. _Reproduced:_ widgets
    `['research', None]`, which is exactly the screenshot's `@research` plus
    `@default · 0` right after a research launch.
  - Dispatch-provisional and proc-shell rows are replaced by authoritative rows.
  - Fleet rows vanish after a remote dismissal.
  - A row is dismissed externally without that reaching `_dismissed_agents`.

There is also a latent same-frame bug. `_reconcile_session_mounted_for_apply` returns
the keys it retired, but `_loading_apply.py` discards that return value. The following
`_sync_panel_group` then reports nothing, so on the incremental display path an
**already-empty** strip is never named for a widget sync. It survives even an
authoritative apply until some later full rebuild.

Earlier attempts (8ae9ac1ea, 2bf6f3d87) kept adding proof sources one exit path at a
time. This fix changes the rule itself: positive evidence of where a row is retires a
key immediately, and unexplained absence only bridges briefly.

## Goal and contract

A tribe panel with zero rendered rows may stay mounted **only** as a short bridge across
an unexplained disappearance, and every such bridge expires within a few seconds whether
or not another load happens. In detail:

- **Placed elsewhere ⇒ retire now.** An identity that is present in the full roster and
  whose current placement is not this key in this scope stops pinning the key in the
  same sync.
- **Unaccounted ⇒ bounded bridge.** Unaccounted means absent from the roster, or present
  and placed here but not rendered. Such an identity pins the key for at most
  `STICKY_PANEL_BRIDGE_S` seconds (a named module constant, 5.0) from the first sync
  that saw it unaccounted. If it renders again within the window, the bridge clears and
  the widget is never unmounted. This keeps the sase-13i anti-flicker guarantee for
  transient zero-row publications.
- Dismissals, explicit removals, complete-history omissions, and query changes keep
  retiring immediately, as they do now.
- Retiring a key always unmounts its widget in the same display pass, including when the
  strip was already empty and no row diff names it.

The panel-key model (`panel_keys_for`, the empty tab → `[None]`) does not change.
Folding, tribe stability for populated panels, and the incremental refresh path stay as
they are.

## Implementation

All changes are Textual presentation state in this repo. No `sase-core` change: this is
widget-layer stickiness, not shared backend logic.

### 1. Placement-aware reconcile in `_display_panel_collection.py`

Extend `_remember_session_mounted_occupancy`, which every `_sync_panel_group` runs.
After the existing record / move / dismissed-prune steps, handle each **active-scope**
sticky key that has no rendered rows this sync (`rendered_now` / `skip_keys` already
compute this). Classify each recorded identity in it:

- Build the placement map **lazily**, only when at least one such empty in-scope key
  exists. On the common no-empty-key sync the path must cost nothing extra (tui_perf
  rule 6). The map is built once per sync over
  `getattr(self, "_agents_with_children", None) or self._agents`. It maps
  `identity → (panel key, in active scope)`:
  - Panel key: `panel_key_per_agent(roster, merge_tribe_panels=...)` (root-anchored, the
    same rule the renderer uses), passed through `normalize_panel_key`.
  - In scope: mirror `scope_agents_to_tab` from
    `src/sase/ace/tui/models/agent_tab_index.py`. Scope is `ALL_AGENT_TABS`, or
    `_agent_tab_index` is `None`, or
    `index.key_for(row) == current_agent_tab_scope(self)`. Use the helpers in
    `_tab_scope.py`.
- **Placed elsewhere:** the identity is in the map, and its key differs from this key or
  it is out of scope. Discard it from this key.
- **Synthetic container whose members are all accounted for:** the identity is absent
  from the map, has an entry in the backing map (`_session_mounted_backing_map`), and
  every backing member is placed elsewhere or is in `_dismissed_agents`. Discard it.
  This retires the old container identity the moment its members show up under the new
  container.
- **Otherwise unaccounted:** record a first-unaccounted monotonic timestamp per identity
  (new dict attr, e.g. `_session_sticky_unaccounted_since`). Discard the identity once
  `now - since >= STICKY_PANEL_BRIDGE_S`.
- Clear an identity's unaccounted timestamp whenever it renders. Drop timestamps for
  identities no longer recorded under any key. Clear the dict together with the store on
  a committed-query change (`_session_mounted_identity_map`).
- Delete a key whose set empties, and include it in the returned retired set. Reuse or
  generalize `_prune_session_mounted_gone` rather than adding a second deletion loop.
  Keep its active-scope-only behaviour: other tabs' keys are reconciled when that tab
  becomes active, because the rescope path already calls `_sync_panel_group`.

Read the clock through one module-level seam, e.g. `_sticky_now = time.monotonic`, so
tests can drive expiry without sleeping.

Update the docstrings that state "absence alone never retires a key" to the new
contract: `_remember_session_mounted_occupancy`, `_retire_session_mounted_identities`,
the module docstring, and the panel-key notes in `agent_panels.py` if they mention
stickiness.

### 2. Make the authoritative-apply retirement reach the widgets

`_reconcile_session_mounted_for_apply` keeps its complete-history semantics. Stash the
keys it retires in a pending set (e.g. `_session_sticky_pending_retired`), and have
`_sync_panel_group` union and clear that set into the retired keys it returns. The
incremental path in `_display_incremental.py` already turns `reconciled_retired` into
`panel_rebuild_keys`, so the emptied widget unmounts in the same apply. The full path
re-syncs all widgets anyway. Leave the call site in `_loading_apply.py` otherwise
unchanged.

### 3. Expire bridges on an idle host without a new timer

Auto-refresh skips unchanged surfaces (tui_perf rule 14), so nothing would re-sync a
bridged strip on a quiet host. Hook the existing one-second countdown tick in
`src/sase/ace/tui/actions/_event_countdown.py`, inside the `agents` branch that is
already behind `_nav_gate` / `_prompt_input_active()` and next to the ToolRun drift
probe. Call a thin synchronous owner method, e.g.
`_maybe_expire_sticky_panel_bridges(now_mono=...)`:

- Return immediately (O(1)) when no unaccounted timestamps are pending or none has
  reached `STICKY_PANEL_BRIDGE_S`.
- Otherwise request the established incremental refresh,
  `self._refresh_agents_display_after_finalize(previous_agents=list(self._agents), defer_detail=True)`,
  so `_sync_panel_group` retires the key and the widget sync unmounts it. Do not add a
  new refresh code path or a new `set_timer` (tui_perf rules 2 and 5).

### 4. Diagnostics

Emit one `trace_event` (`src/sase/ace/tui/util/trace.py`, only active under
`SASE_TUI_TRACE=1`) per retired sticky key. Name the reason: `placed_elsewhere`,
`container_members_accounted`, `bridge_expired`, `dismissed`, `explicit_removal`, or
`complete_history`. A future stuck strip can then be diagnosed from
`~/.sase/perf/tui_trace.jsonl` instead of by guesswork.

## Tests

Add the new cases to `tests/ace/tui/test_agent_panels_display_sticky.py`, using its
`_FakeApp`, `_clan_member`, `_folded_epic_clan_app` helpers and a monkeypatched clock
seam. Each case must fail before the fix:

1. **Folded into another tribe's clan.** A loose `@epic` row later becomes a folded
   member of a collapsed `research` clan. After `_sync_panel_group`, `epic` is retired
   and returned as retired, and `_sorted_widget_panel_keys` has no `epic`.
2. **Moved to another tab.** The row stays in `_agents_with_children`, but
   `_agent_tab_index` / `_active_agent_tab` put it in a different tab than the active
   scope. The old key retires in the active scope.
3. **Container identity change.** A clan container first renders with generation `None`
   and no clan tribe (`@default`), then re-projects with a generation and
   `clan_tribe="research"`. `@default` retires in that same sync because its backing
   member is now under `@research`. This is the screenshot scenario.
4. **Unaccounted bridge holds, then expires.** The roster empties (identity absent). At
   `t+1s` the key is still mounted. At `t+STICKY_PANEL_BRIDGE_S` the next sync retires
   it.
5. **Bridge clears on return.** The identity disappears and then renders again within
   the window. The key is never retired and its unaccounted timestamp is cleared.
6. **Query-filtered row.** The row is present and placed in the key but not rendered,
   because it is absent from `_agents`. It is treated as unaccounted, so the bridge
   applies and then retires it.
7. **Complete-history retirement reaches the display.** An already-empty sticky strip
   plus a complete-history `_reconcile_session_mounted_for_apply` makes the next
   `_sync_panel_group` return that key. Through `_refresh_agents_display_after_finalize`
   with an unchanged roster, the widget is removed from the container.
8. **Countdown hook.** `_maybe_expire_sticky_panel_bridges` is a no-op without pending
   bridges, and requests exactly one incremental refresh once a bridge has expired.

Keep these existing tests green, updating docstrings or names where they claim absence
_never_ retires:

- `test_emptied_roster_without_removal_keeps_sticky_panel_keys` and
  `test_bounded_apply_keeps_identities_missing_from_roster`: both still hold inside the
  bridge window.
- `tests/perf/test_agents_display_rebuild_guard.py::test_empty_incomplete_apply_keeps_session_sticky_epic_widget`:
  the anti-flicker guard must still pass unchanged. Add a sibling case asserting the
  same widget unmounts once the bridge expires.
- `tests/ace/tui/test_epic_panel_arrival_frames.py`,
  `tests/ace/tui/test_agent_cleanup_panel_clan_sticky_e2e.py`,
  `tests/test_agent_kill_dismiss_fast_path.py`, and
  `tests/ace/tui/test_agent_collapsed_panel_kill.py`.

Add one real-app regression next to `test_agent_cleanup_panel_clan_sticky_e2e.py` (same
`AcePage` / `patch_startup_loaders` harness). It drives the scenario-3 sequence through
a real apply and asserts that only the `@research` panel widget
(`panel_widget_id_for_key`) is mounted afterwards, with no `#agent-list-panel` default
strip.

If any Agents-tab PNG golden changes because it was capturing a stale 0-row strip,
regenerate only the affected goldens with `just fix-tui-screenshots`. Inspect the report
and explain the diff; generating a golden does not approve it.

## Verification

- Targeted:
  `pytest tests/ace/tui/test_agent_panels_display_sticky.py tests/perf/test_agents_display_rebuild_guard.py tests/ace/tui/test_epic_panel_arrival_frames.py tests/ace/tui/test_agent_cleanup_panel_clan_sticky_e2e.py tests/test_agent_kill_dismiss_fast_path.py tests/ace/tui/test_agent_collapsed_panel_kill.py`,
  plus the new e2e file.
- Then `just check`, following the lint/test memory note. Symvision may flag new private
  helpers; follow its memory note rather than adding pragmas.
- Note in the final summary that a running TUI keeps executing its imported code until
  restarted (tui_perf rule 15). The user must restart `sase tui` before the fix shows on
  their host.

## Out of scope

- Removing the sticky store, or changing loader / merge removal authority
  (`merge_incomplete_load_after_complete_history`). The bridge keeps the sase-13i
  flicker guarantee.
- Clan generation/tribe resolution timing at launch. Containers may still re-key; the
  sticky layer now tolerates that.
- The `sase-13i.4` live soak.
