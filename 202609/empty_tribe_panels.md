---
tier: tale
title: Retire Agents-tab tribe panels when their last node is dismissed
goal:
  An Agents-tab tribe panel disappears as soon as its last node is dismissed, while
  panels emptied by an incomplete load still survive.
size: medium
proposed_by: bbugyi200.apollo.14
create_time: 2026-09-20 11:17:16
status: wip
---

# Plan: Retire Agents-tab tribe panels when their last node is dismissed

## Problem

On the Agents tab, a tribe panel whose last node has been dismissed stays mounted
forever as an empty title strip (observed as an `@research · 0` panel sitting below a
populated `@default · 1 [D1]` panel). It survives every subsequent auto-refresh and only
goes away when the committed Agents query changes or the TUI restarts.

## Root cause

Two independent layers conspire. Both must be fixed; fixing either one alone leaves the
bug reachable.

### Layer A — the session-sticky panel-key set never retires a key

`src/sase/ace/tui/actions/agents/_display_panel_collection.py` keeps
`_session_mounted_panel_keys`, a set of every panel key that has ever had a rendered row
this session:

- `_remember_session_mounted_occupancy()` adds a key for every rendered agent on each
  `_sync_panel_group()`.
- `_widget_panel_keys()` returns occupancy ∪ sticky, and `_sorted_widget_panel_keys()`
  feeds that union to `_sync_mounted_panel_widgets()` (`_display_panel_widgets.py:185`),
  which mounts one `AgentList` per key and unmounts only keys absent from the union.
- The **only** thing that ever clears sticky is a committed-query change, in
  `_session_mounted_panel_key_set()`.

So once `research` becomes sticky it stays mounted with zero rows. The model layer is
already correct: `panel_keys_for()` in `models/agent_panels.py` drops the tribe, and
`AgentPanelGroup.from_panel_keys` drops it from `_panel_group.panel_keys` — the stale
panel exists purely at the widget-mounting layer.

Confirmed directly against the real helpers: after starting with agents in `@default`
and `@research` and then removing the `@research` agent, `panel_keys_for` returns
`[None]` while `_sorted_widget_panel_keys` returns `[None, 'research']`. With every
agent removed it still returns `[None, 'research']`.

`_sorted_widget_panel_keys` also force-collapses any key without rendered rows, which is
why the leftover renders as a one-line title strip rather than an empty box.

### Layer B — the fast-path row remover deliberately leaves an emptied panel mounted

`_try_remove_agent_rows()` in `_display_panel_patches.py:214` has an explicit branch:

```python
if target_identities and target_identities <= removed_identities:
    # Last visible rows of this tribe: keep the tribe-stable widget
    # as a title strip instead of rebuilding siblings.
    target_widget.render_collapsed()
    ...
    return True
```

Returning `True` tells the dismissal caller the fast path succeeded, so
`_apply_dismissal_in_memory_fast_finish()` (`_dismiss_memory.py:162`) finishes with
`_refresh_agents_display(list_changed=False, defer_detail=True)`. With
`list_changed=False`, `_refresh_agents_display_impl()` (`_display.py:537`) skips
`_sync_panel_group()` and the whole panel-widget resync, so nothing re-evaluates the
mounted panel set. The emptied strip is what the user sees the instant they press `x`,
and Layer A then keeps it there across every later refresh.

### Why the sticky set exists (the constraint the fix must respect)

Sticky keys were added by `45a7895b6b` ("key AgentList widgets by tribe and skip sibling
rebuilds") so that a roster that momentarily reports no rows does not unmount and
remount tribe widgets.
`tests/perf/test_agents_display_rebuild_guard.py::test_empty_incomplete_apply_keeps_session_sticky_epic_widget`
locks this in: applying `_agents = []` from an incomplete load under an active search
query must keep the already-mounted `@epic` widget. The
`if not rendered_present and sticky:` branch in `_widget_panel_keys()` is that
protection.

Therefore **absence of a tribe's agents from the roster must never, by itself, retire a
sticky key** — an incomplete or bounded load is not proof a row is gone
(`_loading_compute_merge.py:312` states exactly this rule for the loader). Retirement
must be driven by the explicit local removals that _are_ proof.

### The removal sites that are proof

Three places mutate the roster in response to a user action, and each already computes
the removed identity set immediately before mutating:

| Site                                                                                                                              | Removed-identity variable |
| --------------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| `_dismiss_memory.py:130` (`_apply_dismissal_in_memory`, covers `x`, `X`, and marked dismiss via `_dismissing.py` / `_marking.py`) | `removed_identities`      |
| `_kill_identity.py:183` (kill + hide)                                                                                             | `identities`              |
| `_proc_shell_dismiss.py:73` (proc-shell dismiss)                                                                                  | `removed`                 |

`_display.py:461` also calls `_try_remove_agent_rows`, but for _loader-driven_ diffs —
that call site must **not** retire anything, which is precisely the case the perf test
guards. This is why the hook goes on the three user-action sites and not on the shared
`_try_remove_agent_rows` helper.

## Design

Make the sticky store identity-backed and retire keys only on explicit removal.

1. Replace `_session_mounted_panel_keys: set[PanelKey]` with
   `_session_mounted_panel_identities: dict[PanelKey, set[tuple[AgentType, str, str | None]]]`
   — panel key → the identities that mounted it.
2. `_remember_session_mounted_occupancy()` records each rendered agent's identity under
   its normalized panel key instead of just the key.
3. Add `_retire_session_mounted_identities(identities)`: drop those identities from
   every key's set and delete any key whose set becomes empty. Return the retired keys.
4. Call it from the three user-action removal sites, before the roster mutation's
   refresh runs.
5. `_session_mounted_panel_key_set()` keeps its current signature and returns
   `set(self._session_mounted_panel_identities)`, so `_widget_panel_keys()`,
   `_sorted_widget_panel_keys()`, and the `live_keys` computation in
   `_sync_panel_group()` need no logic change.
6. When a dismissal retires at least one key, the dismissal must take a refresh path
   that actually resyncs panel widgets: in `_apply_dismissal_in_memory_fast_finish()`
   (and the equivalent finish in the proc-shell and kill-identity paths), pass
   `list_changed=True` instead of `False` when the retirement returned a non-empty key
   set. The fast path stays intact for the common case where the panel still has rows.

This keeps retirement decoupled from loading semantics entirely: absence never retires,
so the incomplete-load invariant holds by construction rather than by a heuristic guard.

Prototyped against the real `panel_key_per_agent` / `agent_is_rendered_in_agents_panel`
helpers, this rule gives: dismissing the last `@research` node retires `research` and
leaves `{None}`; dismissing every node retires everything so occupancy `[None]` renders
the empty state; and an emptied roster with no dismissal keeps `{epic, review}` sticky.

## Steps

1. **Rework the sticky store** in
   `src/sase/ace/tui/actions/agents/_display_panel_collection.py`: identity-backed dict,
   `_remember_session_mounted_occupancy()` recording identities,
   `_session_mounted_panel_key_set()` returning a key snapshot, and a new
   `_retire_session_mounted_identities()` returning the retired keys. Preserve the
   existing query-change clearing behavior.
2. **Update the state declarations**: the initializer at
   `src/sase/ace/tui/actions/_state_init_agents.py:300` (keep and refresh the comment
   explaining the sticky lifetime) and the mixin contract in `_display_panel_state.py`
   if the new method needs declaring there.
3. **Hook the three removal sites** listed in the table above, each passing the identity
   set it already computed.
4. **Make retirement visible immediately**: have the dismissal/proc-shell/kill finish
   paths request a panel-resyncing refresh (`list_changed=True`) when a key was retired,
   so the panel disappears on the keypress rather than at the next auto-refresh.
5. **Tests**:
   - Unit: dismissing the last node of a tribe retires the key, and
     `_sorted_widget_panel_keys()` no longer returns it (extend
     `tests/ace/tui/test_agent_panels_display.py`, whose `_FakeApp` already stubs the
     sticky attributes at lines 162-163 and will need updating for the new shape).
   - Unit: a tribe with a remaining node keeps its key after a sibling is dismissed.
   - Regression:
     `tests/perf/test_agents_display_rebuild_guard.py::test_empty_incomplete_apply_keeps_session_sticky_epic_widget`
     must still pass unchanged — it is the guard for the incomplete-load case.
   - Behavioral: extend `tests/ace/tui/test_agent_collapsed_panel_kill.py`, which
     already drives dismiss-all through the real app and today only asserts the _model_
     dropped the key (line 343) with a comment conceding that "session-sticky widgets
     may keep" the panel. Assert the widget is unmounted too.
   - Also update any fake app that sets `_session_mounted_panel_keys` directly:
     `tests/ace/tui/test_agent_panels_display.py:162` and the helper used by
     `tests/perf/test_agents_display_rebuild_guard.py:284`.
6. **Verify**: run `just fix` inline, then `sase tool run check`. Do not run
   `just check-full` — nothing here asks for it. This changes rendered TUI output, so
   also run `just fix-tui-screenshots` (targeted with `--` selectors for the Agents
   corpus first) and inspect the report before accepting any golden change.

## Out of scope

- Changing when a panel is force-collapsed versus expanded.
- Changing the loader's bounded/incomplete-load merge semantics.
- Retiring sticky keys for agents that vanish for non-dismissal reasons (external
  `sase agent gc`, for example). Absence alone is deliberately not treated as proof; if
  that case is ever reported, it needs its own authoritative-snapshot signal.
