---
tier: tale
title: Include monitor starters in family and clan total runtime
goal:
  Family and clan total runtime counts every concrete agent shell exactly once,
  including agents that started monitors, while gate windows and container/workflow
  pass-through semantics stay unchanged.
size: medium
proposed_by: bbugyi200.kellys_mbp.q
---

# Fix: family total runtime drops agents that started monitors

## Problem

The Agents-tab family/clan total runtime sometimes excludes the runtime of an agent
shell that started a monitor. Observed live (screenshot evidence
`~/tmp/screenshots/20260904_120718.png`): family `g` shows a total of `13m23s`, which is
exactly `--plan` (10m49s) + `--mon` (2m34s). The `--code` shell's `53m24s` is missing,
even though the FAMILY SHELLS panel correctly shows it per-shell. The `--gate` window
(27m15s) is excluded by design (commit `fbd37ca3d`: gate shells own a human decision
window, not agent runtime) — that exclusion is correct and must be preserved.

## Root cause

`_aggregate_runtime()` in `src/sase/ace/tui/models/agent_time.py` collects
`ClanRuntimeMemberWire` records via the nested `append_runtime_member()` closure. That
closure treats **any** child row with a non-empty `runtime_children` list as a pure
pass-through container:

```python
grandchildren = getattr(child, "runtime_children", ())
if grandchildren:
    for grandchild in _runtime_child_rows(child, include_monitor_shells=...):
        append_runtime_member(grandchild)
    return          # <-- child's own interval is silently dropped
```

That replace-with-descendants behavior is correct for rows whose descendants _represent_
them (clan containers, family container roots, workflow aggregate rows whose loaded
agent steps carry the real intervals — this pattern dates to commit `21d995ce5`, where
children were always family roots). But a monitor shell attaches as a `runtime_children`
entry of its **starter** shell (a monitor "nests under its starter" — see
`_attach_family_containers` docstring in `src/sase/ace/tui/models/_agent_ordering.py`
and the starter-identity resolution in `src/sase/monitor/followup.py`). So a concrete
member shell like `g--code` that started `g--mon` is _replaced by_ its monitor in the
family walk and its own interval never reaches `aggregate_clan_runtime()` — the exact
arithmetic seen in the screenshot.

The correct principle: **a monitor never represents its starter's runtime; it only adds
to it.** The interval-union performed by the Rust `aggregate_clan_runtime` binding makes
overlapping starter + monitor intervals safe to include together.

Two existing tests pin this buggy behavior _incidentally_ (their expectations were
computed from the code, not from intent — the commit that added them, `20f0d1395`,
states only that "monitor runtime belongs to family and clan container rows", never that
the starter's runtime should vanish from those containers):

- `test_compute_row_runtime_family_container_spans_running_monitor` expects `6m` where
  the true union of starter (14:30→14:35) and monitor (14:34→14:40) is `10m`.
- `test_family_container_includes_monitor_but_not_gate` expects `6m` for the same
  fixture plus a pending gate; the true total is `10m`.

### Second manifestation (same root cause family)

When the _aggregating row itself_ is a family root whose only eligible members are
monitor shells (a root agent that started monitors directly — the shape of
`settled_monitor_family_agents()` in
`tests/ace/tui/visual/_ace_agents_png_snapshot_family_fixtures.py`), the aggregate is a
monitor-only union and the root's own work interval is dropped, because
`_aggregate_runtime()` never contributes the aggregating row's own interval. For
plan-chain families this is correct (the root record mirrors the member lifecycle, and
including it would resurrect the excised gate window), but for monitor-only membership
it undercounts, and for a settled starter it makes the row borrow the monitor's terminal
time — the very symptom commit `20f0d1395` fixed for non-root starters.

## Fix

All changes are presentation-layer walk logic over TUI row objects in
`src/sase/ace/tui/models/agent_time.py` (no Rust-core change: the union math in
`sase_core` is already correct; only the Python member collection is wrong).

### 1. Restructure `append_runtime_member` in `_aggregate_runtime`

Split the existing wire-building body into a small helper (e.g.
`_append_member_wire(row)` closure) so it can be reused, then change the walk to:

```python
def append_runtime_member(child: "Agent") -> None:
    child_id = id(child)
    if child_id in seen:
        return
    seen.add(child_id)
    eligible = _runtime_child_rows(
        child, include_monitor_shells=include_monitor_shells
    )
    for grandchild in eligible:
        append_runtime_member(grandchild)
    if getattr(child, "runtime_children", ()) and _represented_by_descendants(
        child, eligible
    ):
        return
    _append_member_wire(child)
```

with a new module-level predicate:

```python
def _represented_by_descendants(
    agent: "Agent", eligible: tuple["Agent", ...]
) -> bool:
    """Return whether *agent*'s runtime is carried by its descendant rows.

    Clan containers, family container roots, and workflow aggregate rows are
    represented by their members/steps. Monitor shells are the exception:
    they add to their starter's runtime but never represent it, so a row
    whose eligible descendants are all monitors still owns its interval.
    """
    if not (
        _aggregates_family_shells(agent)
        or any(
            child.is_workflow_step_child
            for child in getattr(agent, "runtime_children", ())
        )
    ):
        return False
    return not eligible or any(not row.is_monitor for row in eligible)
```

Behavior deltas this produces (all intended):

- A **concrete member shell** (not a container, no workflow steps) with runtime children
  now contributes its own wire _and_ its eligible descendants. This is the screenshot
  fix: `g--code` + `g--mon` both count.
- A **container / workflow aggregate** child with at least one non-monitor eligible
  descendant keeps today's pass-through (gate-window excision from `fbd37ca3d` is
  untouched; plan-chain roots are still represented by their member shells; workflow
  parents are still represented by their steps).
- A **container whose eligible descendants are all monitors** now also contributes its
  own wire (clan-level version of the second manifestation).
- A container with raw children but an _empty_ eligible list (e.g. family parked on a
  pending gate) keeps today's behavior: nothing is contributed for it (`not eligible` →
  represented). A row with no runtime children at all also keeps today's behavior
  (appended, guarded by the raw-children check).

Do NOT import from `agent_family_members.py` (its `_is_workflow_aggregate_row` is
private and importing it would create an import cycle through `agent.py` →
`agent_time.py`); the inline `is_workflow_step_child` check above is deliberately
self-contained.

### 2. Include the aggregating row's own interval for monitor-only members

In `_aggregate_runtime()`, track whether any appended wire came from a non-monitor row
(set a flag inside `_append_member_wire`). After the member collection loop:

```python
if runtime_members and not saw_non_monitor_member and not agent.is_clan_container:
    _append_member_wire(agent)
```

(`seen` already contains `id(agent)` — added by `_runtime_interval` before calling
`_aggregate_runtime` — which is why this must go through the direct wire helper rather
than `append_runtime_member`.) This makes a family root that started monitors contribute
its own work interval to its family total, while plan-chain roots (which always collect
non-monitor member shells) stay excluded, preserving gate-window excision. The
`is_clan_container` guard keeps synthetic clan rows (whose start/stop are synthesized
min/max spans) out of the union.

Note `_row_runtime_terminal_time(agent)` for the aggregating row also feeds
`terminal_times` via the shared helper, so a fully-settled monitor family anchors its
finish timestamp correctly.

### 3. Leave the tick predicates alone

`runtime_suffix_ticks` / `row_runtime_or_wait_ticks` check each child row itself _and_
recurse, so they never drop a monitor starter; no change.

## Tests

In `tests/ace/tui/widgets/test_agent_list_runtime_compute.py` (fixtures from
`tests/ace/tui/widgets/agent_list_runtime_helpers.py`):

1. Update the two incidentally-pinned expectations:
   - `test_compute_row_runtime_family_container_spans_running_monitor`: `"6m"` → `"10m"`
     (union of starter 14:30→14:35 and monitor 14:34→now 14:40).
   - `test_family_container_includes_monitor_but_not_gate`: `"6m"` → `"10m"` (pending
     gate still excluded).
2. Add a regression test mirroring the screenshot: family container with a DONE plan
   shell (own interval), a DONE code shell whose `runtime_children` holds a running
   `monitor_shell`, and (optionally) an approved gate between them. Assert the total
   equals the union including the code shell's interval and that the aggregate ticks.
3. Add a test for the second manifestation: a family container row whose only runtime
   children are monitor shells (mix one settled + one running, like
   `settled_monitor_family_agents`) contributes its own interval — total = union(root
   interval, monitor intervals), not monitor-only.
4. Add a guard test that a workflow aggregate row inside a family/clan is still
   represented by its agent steps only (no own-interval leak): a workflow parent with
   one agent step child must produce the step's interval, unchanged from today.
5. Confirm (existing tests, should still pass unchanged):
   `test_compute_row_runtime_settled_starter_ignores_running_monitor`,
   `..._ignores_settled_monitor` (starter's own row still shows only its own interval),
   `test_family_container_chained_gates_do_not_resurrect_intervals`,
   `test_lowest_row_runtime_drops_family_parked_on_pending_gate`, and the clan rendering
   suite in `tests/ace/tui/widgets/test_agent_list_runtime_clan_rendering.py`.

## Verification

1. `just install` first if this workspace's venv is stale (ephemeral-clone rule), then
   `just check` (runs every lint gate + the diff-scoped test lane). Hand it to a monitor
   via the standard monitor flow if it runs long.
2. PNG snapshots:
   `tests/ace/tui/visual/_ace_agents_png_snapshot_family_panel_fixtures.py` builds a
   family whose monitor nests under the last coder (`with_monitor=True`) and runs
   through the production `sort_and_reorder`, so family-panel goldens that render that
   container's runtime suffix may legitimately change. Run `just test-visual`; inspect
   `.pytest_cache/sase-visual/` diffs to confirm any changed pixels are only the
   expected runtime-total text, then accept with `--sase-update-visual-snapshots`.
   Snapshot deltas beyond runtime text mean the fix leaked wider than intended — stop
   and re-examine rather than accepting.
