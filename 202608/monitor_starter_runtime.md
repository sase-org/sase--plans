---
tier: tale
title: Stop settled monitor starters from borrowing their monitor's runtime
goal:
  An agent shell that ended its turn by starting a SASE monitor renders its own settled
  runtime in the ACE Agents tab instead of a live, incrementing one, while family and
  clan container rows keep spanning the running monitor.
size: small
proposed_by: bbugyi200.athena.0c0
create_time: 2026-08-23 15:56:11
status: wip
---

# Settled monitor starters must stop borrowing their monitor's runtime

## Problem

In the ACE Agents tab, an agent shell that ended its turn by starting a SASE monitor
(via `/sase_monitor`) keeps rendering a live, incrementing runtime with the running
marker (`🏃`) even though its runner is long gone.

Observed live example (family `0by`, project `sase`):

```
▸ 0by                                    1 agent · 1 running
  sase (TESTING) ×8 -4 ⚙1 0by            🏃 2m44s / 56m03s
  └─ (TALE APPROVED) 0by--plan             14:37:04 · 9m52s
  └─ (TALE DONE)     0by--code           🏃 46m10s        ← wrong: ticking
  └─ (TESTING)       0by--mon            🏃 2m44s
```

`0by--code` is finished. Its artifacts record it as finished, and every input the TUI
needs is correct:

- `<artifacts>/20260823144427/agent_meta.json` has
  `"stopped_at": "2026-08-23T19:28:06.939205+00:00"` (file written 15:28:06 local).
- `<artifacts>/20260823144427/done.json` has `"outcome": "completed"`.
- The persistent artifact index row carries the same `stopped_at`, indexed 15:28:06.
- The loaded `Agent` has `stop_time = 2026-08-23 15:28:06` and
  `compute_leaf_row_runtime(...) == (('', '15:28:06'), '43m39s')`.

So this is purely a runtime-projection defect, not a loader or runner-marker defect.

There are two visible symptoms, both from the same cause:

1. **While the monitor runs** the settled starter row ticks and shows a growing elapsed
   time (the screenshot above).
2. **After the monitor settles** the starter row keeps a wrong, inflated runtime whose
   finish timestamp is the _monitor's_ end, not the agent's. Corpus evidence from a full
   `load_all_agents()` sweep of the live artifact store: `sase-p4.4--1` renders
   `Aug 17 23:42 · 4m32s` while its own monitor `sase-p4.4--mon-0` renders
   `Aug 17 23:42 · 2m20s` — identical end timestamps, with the starter's own interval
   (`23:40 · 2m27s`) nowhere on screen.

## Root cause

`sase/ace/tui/models/agent_time.py` projects a row's runtime with `_runtime_interval()`,
which prefers `_aggregate_runtime()` over `_leaf_runtime_interval()` whenever the row
has any `runtime_children`.

A monitor shell is linked as a `runtime_child` of the agent shell that started it. That
link exists so `concrete_family_shell_rows()` can place the monitor immediately after
its causal starter in the flattened FAMILY SHELLS roster (see
`_expand_nested_monitor_shells` in `sase/ace/tui/models/agent_family_members.py`); the
monitor is rendered as a _sibling_ row, not nested under the starter.

`_aggregate_runtime()` does not know that. It walks `runtime_children`, replaces any
child that itself has children with that child's descendants, and unions the resulting
intervals. For `0by--code` the members become its own `main` workflow step
(`14:44:27 → 15:28:06`) plus `0by--mon` (`15:27:53 → now`, still running), so the row
reports `active=True` and `43m39s + monitor-elapsed − 13s overlap`. That is exactly the
`46m10s` in the screenshot and the `1h02m / 15:47:04` the same row showed after the
monitor settled.

`runtime_suffix_ticks()` has the mirrored problem: its `runtime_children` loop returns
`True` as soon as any descendant ticks, and that loop runs _before_ the
`if agent.stop_time is not None: return False` guard. So the settled starter is painted
with the live `🏃` marker.

The container/clan rows above the starter are **not** wrong: `0by` legitimately spans
the running monitor, and the clan roster panel already sidesteps the bug entirely by
calling `compute_row_runtime()` on a `_leaf_for_runtime()` copy with `runtime_children`
cleared (`sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py`). The
agent-list projection and the roster panel therefore disagree today for the same row;
this change makes them agree.

## Fix

A monitor shell's runtime belongs to the family, not to the shell that started it. Feed
monitor shells into aggregate rows (family containers and clan containers) and never
into a concrete agent shell's own row.

All of this is in `src/sase/ace/tui/models/agent_time.py`.

1. Add a module-private predicate for "this row is an aggregate row that owns monitor
   runtime":

   ```python
   def _aggregates_monitor_shells(agent: "Agent") -> bool:
       return agent.is_clan_container or agent.is_family_container_row
   ```

   Document _why_ it is container-ness and not `stop_time`: a settled family container
   (`0by`, whose `stop_time` is also set, because the runner records `stopped_at` on the
   root artifacts dir too) must keep ticking while its monitor runs.

2. In `_aggregate_runtime()`, resolve the flag **once from the walk root** and capture
   it in the `append_runtime_member` closure:

   ```python
   include_monitor_shells = _aggregates_monitor_shells(agent)
   ...
   def append_runtime_member(child: "Agent") -> None:
       if child.is_monitor and not include_monitor_shells:
           return
       ...
   ```

   Resolving once at the walk root is load-bearing. The container `0by` reaches
   `0by--mon` only by descending through `0by--code`; recomputing the flag per level
   would drop the monitor from the container's aggregate and settle a family that is
   genuinely still working. Keep the existing `seen` cycle guard semantics — do not add
   a skipped monitor to `seen`, so a later walk rooted at a container can still visit
   it.

3. Thread the same walk-root flag through `runtime_suffix_ticks()` as a private
   keyword-only parameter (default `None` = resolve from the row), and skip monitor
   children when it is false:

   ```python
   def runtime_suffix_ticks(
       agent: "Agent",
       _seen: set[int] | None = None,
       *,
       _include_monitor_shells: bool | None = None,
   ) -> bool:
   ```

   Threading rather than recomputing matters for the same reason as (2): a container
   must still see its monitor grandchild through the settled starter.

4. Apply the identical filter in `row_runtime_or_wait_ticks()`'s own `runtime_children`
   recursion, threading the same walk-root flag. Without it a settled starter stays
   "time sensitive" in `_agent_list_render_cache`, so the row misses its render cache
   every wall-clock second while rendering identical text.

Nothing outside this module changes. The aggregation math itself already lives in the
Rust core (`aggregate_clan_runtime`); this change only decides which TUI rows are
members of an aggregate, which is presentation projection over `Agent.runtime_children`
and stays on the Python side of the Rust core boundary.

## Expected behavior after the fix

Verified by prototyping the change against the live artifact store (2630 loaded rows,
monkeypatched in a scratch script, no repo files touched):

- `0by--code`: `(None, '1h')` ticking → `(('', '15:28:06'), '43m39s')`, `ticks=False`.
- `0by` family container: still ticking, still spans the running monitor.
- `0by--mon`: unchanged, still ticking.
- Exactly one row in the whole corpus changes its ticking decision — the reported one.
- 36 settled monitor-starter rows shed their monitor's runtime and gain their own, e.g.
  `sase-p4.4--3`: `Aug 18 00:52 · 48m01s` → `Aug 18 00:07 · 3m04s` (its monitor
  `sase-p4.4--mon-2` continues to render `Aug 18 00:52 · 45m05s` on its own row).
- Clan roster duration labels: 0 changes across all 218 roster entries, including the 28
  clan family members that own monitor children.

Those 36 rows are corrections, not regressions: in every case the monitor already has
its own roster row carrying the time that was being double-counted onto its starter.

## Tests

Add unit coverage next to the existing runtime tests. `tests/ace/tui/widgets/` already
holds `agent_list_runtime_helpers.py` (`agent()`, `workflow_child()`), which the new
tests should reuse; extend it only if a monitor/family-container builder is genuinely
missing. Keep every touched test file under 500 lines.

In `tests/ace/tui/widgets/test_agent_list_runtime_ticks.py`:

- A settled non-container starter (`stop_time` set) with a running monitor child does
  not tick, and does not tick even though the monitor child alone does.
- A family container row whose monitor is a _grandchild_ (container → settled starter →
  running monitor) still ticks. This is the regression guard for the "resolve the flag
  once at the walk root" requirement; recomputing per level makes this test fail.
- A clan container with a running monitor descendant still ticks.
- A settled non-container starter with a running _non-monitor_ child still ticks (the
  filter is monitor-specific, not a blanket "settled rows never tick").
- The same four shapes for `row_runtime_or_wait_ticks`.

In `tests/ace/tui/widgets/test_agent_list_runtime_compute.py`:

- `compute_row_runtime()` on a settled starter with a running monitor child returns the
  row's own terminal `(timestamp, elapsed)` pair — identical to
  `compute_leaf_row_runtime()` for that row.
- `compute_row_runtime()` on a settled starter with a _settled_ monitor child also
  returns the starter's own interval, not one extended to the monitor's end. This is the
  guard for visible symptom 2.
- `compute_row_runtime()` on the family container still returns an active interval that
  spans the monitor.

## Verification

- `just check` (this touches TUI model code, which is inside the normal scoped-test
  selection).
- If the scoped run escalates or reports an unusual selection, hand `just check-full` to
  `/sase_monitor` with `--start-status TESTING --stop-status TESTED`.
- No PNG snapshot goldens should move: the change only alters runtime text for rows that
  own a monitor child, and no visual fixture builds that shape. If `just test-visual`
  does report a diff, inspect `.pytest_cache/sase-visual/` and treat it as a real
  finding rather than accepting it blindly.

## Out of scope

- The runner-side handoff (`sase/axe/run_agent_exec_monitor.py`, `record_stop_time`) is
  correct and must not be touched; `stopped_at` is written for both the root and the
  current member artifacts dir at handoff time.
- Container and clan aggregate semantics stay as they are.
- No Rust core change.
