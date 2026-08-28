---
tier: tale
title: Restore settled monitor nodes in the ACE Agents tab
goal:
  A settled monitor family member renders as its own node in the ACE Agents tab, both
  for monitors created from now on and for the ~600 already-settled monitors already on
  disk.
size: medium
proposed_by: bbugyi200.athena.0fn.f2
---

# Plan: Restore settled monitor nodes in the ACE Agents tab

## Symptom

The `0fl` family ran `just check-full` through a monitor (`0fl--mon`), and the monitor
launched the follow-up `0fl--1`. The ACE Agents tab shows

```
0fl
  └─ (TALE APPROVED) 0fl--plan
  └─ (TALE APPROVED ✓) 0fl--gate
  └─ (TALE DONE) 0fl--code
  └─ (WORKING TALE) 0fl--1
```

with no `0fl--mon` node between `0fl--code` and `0fl--1`. The `FAMILY SHELLS` roster on
the prompt panel likewise reads `3` instead of `4`, and the container row carries no
`⚙N` monitor lane badge.

This is not specific to `0fl`. The precise, sharper statement of the bug is:

> A monitor node is visible while the monitor is running, and disappears the instant the
> monitor settles.

## Verified root cause

A monitor's `done.json` never records `finished_at`, and the artifact index's Tier 1
"recent completed" window is ordered by that column.

1. Every monitor done-marker payload is hand-built and omits `finished_at`. The four
   writers are:
   - `src/sase/monitor/supervise.py` (~line 478)
   - `src/sase/monitor/proc_adapter.py` (~line 260)
   - `src/sase/monitor/reconcile.py` (~line 192)
   - `src/sase/monitor/start.py` (~line 499, the `_teardown_failed_member` path)

   Both peer writers do record it: `sase/gate_shell/settlement.py::_done_marker` sets
   `"finished_at": time.time()`, and `sase/axe/run_agent_markers.py::build_done_marker`
   always sets it. Monitors are the only shell kind that omits the field.

2. The Rust core indexes that field verbatim:
   `crates/sase_core/src/agent_scan/index.rs::RecordSummary::from_record` sets
   `finished_at: done.and_then(|d| d.finished_at)`, so every settled monitor row lands
   in `agent_artifacts` with `finished_at = NULL`.

3. The Tier 1 completed tier (`completed_where`, same file) orders by

   ```sql
   ORDER BY COALESCE(finished_at, 0) DESC, timestamp DESC
   ```

   with `recent_completed_limit = 200` (`_TIER1_RECENT_COMPLETED_LIMIT` in
   `src/sase/ace/tui/models/_agent_loader_artifacts.py`). `COALESCE(finished_at, 0)`
   sorts a NULL row behind _every_ row that has a real `finished_at`, so a monitor that
   settled ten minutes ago sorts behind a six-month-old agent. Settled monitors never
   reach the 200-row window.

4. While a monitor is still running it has no done marker, so it is picked up by the
   _active_ tier (`active_where`), which does not order by `finished_at`. That is why a
   running monitor is visible and a settled one is not.

### Evidence gathered while diagnosing

- 600 of 600 `done_outcome='monitored'` rows in `~/.sase/agent_artifact_index.sqlite`
  have `finished_at IS NULL`. Zero exceptions.
- 600 of 600 monitor `done.json` files on disk omit `finished_at`; 600 of 600 of their
  sibling `agent_meta.json` files _do_ carry `stopped_at`.
- Running the top-200 completed-window query by hand returns **0** monitor rows.
- `load_tiered_agents()` (the production TUI path) returns **1 of 58** monitors — the
  one still running. `load_all_agents()` (the full filesystem scan, used only by
  fallback/tests) returns **58 of 58**. Gates are unaffected: **12 of 12** both ways.
- Patching only `finished_at` onto `0fl--mon`'s row in a _copy_ of the index makes the
  row reappear in the Tier 1 query result with nothing else changed. This is the
  controlled experiment that pins causation.

Everything downstream of the loader is already correct and needs no change: with the row
present, `sort_and_reorder` places it directly after `0fl--code`,
`agent_gating_fold_key` already climbs past a mid-family starter to the family
container's fold, `filter_agents_by_fold_state` keeps it visible at `EXPANDED`,
`shell_lane_counts` reports `monitor.settled=1`, and `concrete_family_shell_rows`
returns `[0fl--plan, 0fl--gate, 0fl--code, 0fl--mon, 0fl--1]` in that order. All five
were verified directly against the real on-disk artifacts.

## Changes

Two repositories are involved. Change 1 stops the bleeding for every future monitor;
change 2 recovers the ~600 monitors whose `done.json` is already written and will never
be rewritten.

### 1. `sase` repo — monitor done markers record `finished_at`

Give the four monitor done-marker writers the same `finished_at` stamp the gate shell
already writes. Rather than repeating `"finished_at": time.time()` in four places (which
is exactly how the field went missing), add one small shared helper in
`src/sase/shells/settlement.py` — the module both shell kinds already share — along the
lines of:

```python
def stamp_shell_finished_at(marker: MutableMapping[str, Any]) -> None:
    """Record the settlement instant on a shell done marker.

    Every done marker the artifact index can see must carry ``finished_at``:
    the index's recent-completed tier orders by it, and a marker that omits
    it sorts behind every marker that has one.
    """
```

Call it from all four monitor writers, and switch
`gate_shell/settlement.py::_done_marker` to it too so the two shells share one
definition of "settled at". Keep the gate's observable payload byte-identical apart from
provenance (it already writes `time.time()`).

Note that `src/sase/monitor/start.py::_teardown_failed_member` writes a marker for a
monitor that never really ran; it still needs the field, because a `failed` monitor is
just as much a settled row the Agents tab should be able to reach.

### 2. `sase-core` repo — derive a missing `finished_at` at index time, and force one rebuild

Open the core repo with `sase repo open sase-core -r "<reason>"` and use the printed
path; do not clone or path-guess it.

In `crates/sase_core/src/agent_scan/index.rs`:

- **`RecordSummary::from_record`**: change

  ```rust
  finished_at: done.and_then(|d| d.finished_at),
  ```

  to fall back to the agent meta's `stopped_at` (parsed with the existing
  `parse_runtime_timestamp`) when the record has a done marker but the marker omits
  `finished_at`. There is already an in-file precedent for exactly this derivation:
  `terminalized_abandoned_record` builds `finished_at` from
  `meta.stopped_at → latest_modified` and marks the result
  `finished_at_estimated: true`. Only the _summary column_ changes; `record_json` and
  therefore everything the Python loader displays stay untouched, so this is a pure
  ordering/visibility repair.

  Do **not** change `completed_where`'s `ORDER BY`. A `COALESCE`/`CASE` over a computed
  expression would defeat `idx_agent_artifacts_recent_completed`
  (`hidden, has_done_marker, finished_at, timestamp`) and force a sort over the whole
  table on a TUI hot path. Keeping the column populated preserves the index.

- **Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION`** from `24` to `25`. This is what
  recovers history:
  `sase.core.agent_artifact_index_lifecycle_schema .refresh_agent_artifact_index_if_schema_stale`
  reads the stored version directly and runs `rebuild_agent_artifact_index` exactly once
  when it is behind, which re-derives every row from disk. Without the bump the 600
  existing rows keep their NULL and stay invisible forever, because their signatures are
  unchanged and nothing re-materializes them.

Follow the standard core workflow: update the Rust code and its tests, then rebuild the
binding into the workspace (`just install`) before running the Python suites, since the
Python side imports `sase_core_rs`.

## Tests

### `sase-core`

Add unit tests next to the existing index tests in
`crates/sase_core/src/agent_scan/index.rs`:

- A record with a done marker that omits `finished_at` but whose `agent_meta` carries
  `stopped_at` indexes with a non-NULL `finished_at` derived from `stopped_at`.
- A done marker that _does_ carry `finished_at` keeps its own value; the meta fallback
  never overrides an explicit marker value.
- A record with a done marker and neither `finished_at` nor `stopped_at` still indexes
  as NULL rather than panicking or inventing a value.
- The regression that motivated the change: a settled monitor-role record and a much
  older settled agent record, queried through the Tier 1
  `include_active + include_recent_completed` path with a small
  `recent_completed_limit`, return the monitor. Assert this fails against the old
  behaviour (the monitor sorted to position zero) so the test cannot silently stop
  covering the bug.

### `sase`

- `tests/monitor/` — one assertion per settled path that the written `done.json` carries
  a numeric `finished_at`: the supervise path (`test_monitor_supervise.py`), the proc
  adapter (`test_monitor_proc_facade.py`), the reconcile path
  (`test_monitor_store_reconcile.py`), and the failed-teardown path
  (`test_monitor_start_teardown.py`). Place each beside that file's existing done-marker
  assertions rather than adding a new file per path.
- A shared regression test asserting that both shell kinds stamp the field through the
  one helper, so a fifth monitor writer added later cannot reintroduce the gap.
- Do not add a snapshot/visual test; nothing about rendering changes.

## Verification

1. `just install` (the Rust binding must be rebuilt before any Python test can see the
   core change).
2. `just check` inline.
3. `just check-full` through the `/sase_monitor` skill with the `TESTING` / `TESTED`
   status pair — never inline.
4. End-to-end confirmation on real data, after the schema bump has triggered its
   rebuild: `load_tiered_agents()` should return substantially all monitors rather than
   only the running one, and `0fl--mon` specifically should be present with
   `agent_family == "0fl"`. Before the fix these numbers are 1 of 58 and absent; after,
   expect 58 of 58 and present.

## Out of scope

- **The `followup_agents` orphan.** `apply_status_overrides`
  (`src/sase/ace/tui/models/_agent_status_apply.py`) builds `parent_by_suffix` from rows
  where `not agent.is_child_row`, so a monitor started by a non-root family member (its
  `parent_timestamp` points at `--code`, not at the family root) is never appended to
  any `followup_agents` list — 50 of 58 monitors are orphaned this way. This is real but
  **not** the reported bug and needs no fix here: `_attach_runtime_children` puts the
  monitor in the starter's `runtime_children`, and every consumer that matters
  (`shell_lane_counts`, `concrete_family_shell_rows`, `_descendant_family_shells`,
  `_append_followup_subtree`) traverses both collections. All four were verified to
  produce the correct result with the monitor row present. Do not widen the change to
  cover it.
- The 1,642 non-monitor rows that also carry a NULL `finished_at`. They are legacy rows
  predating the field; the core-side derivation will repair whichever of them have a
  `stopped_at`, but they are not a goal and their behaviour should not gate the change.
- Any change to `completed_where`'s `ORDER BY`, the Tier 1 limits, or the fold/ordering
  code — all verified correct once the row is loaded.
