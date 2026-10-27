---
tier: tale
title: Keep queued agent sessions and their clans QUEUED in the Agents tab
goal:
  An agent session whose current turn is parked at the runner-slot gate, and the clan
  that contains it, read QUEUED in the Agents tab and stay that way across capacity
  report refreshes instead of decaying to WAITING.
size: medium
proposed_by: bbugyi200.athena.14t
proposed_with: claude/opus
create_time: 2026-10-27 11:52:17
status: wip
---

# Keep queued agent sessions and their clans QUEUED in the Agents tab

## Problem

On athena the Agents tab showed agent session `sase-1ox.1` as `WAITING ⏳1`, and its
clan `sase-1ox` as `WAITING [W4]`. The session's current turn `sase-1ox.1--1` was really
parked at the runner-slot gate. Its own row correctly read `QUEUED #1/3`, and the
session detail panel said `[capacity] needs 1.0 · 0.0 free · queue #1 of 3`. The session
row and the clan row should both read `QUEUED`.

## Root cause (reproduced)

`QUEUED` is a display-only overlay. `refresh_runner_slot_context()`
(`src/sase/ace/tui/models/agent_runner_slots.py`) promotes a row to `QUEUED` in three
cases:

- the row itself is a slot waiter;
- the row's `wait_display_source` is a slot waiter. An agent-session root points this at
  its queued turn (set in `_agent_status_apply.py`);
- for clan containers, `apply_clan_container_status()` aggregates over `clan_members()`,
  which reads the container's `runtime_children`.

Both lookups go through object identity: `id(source) in queue_positions` and
`runtime_children`. The function also reverses itself:
`runner_slot_display_status(..., slot_queued=False)` turns `QUEUED` back into `WAITING`.

**Primary defect (regression from `f843bd8100`, phase sase-1om.6):**
`_run_agents_capacity_refresh_from_roster()`
(`src/sase/ace/tui/actions/agents/_loading_filter.py`) runs the slot pass on shallow
`copy(agent)` rows. The copies' `wait_display_source` and `runtime_children` still point
at the live rows, not at the copies in the batch. As a result:

- `id(source) in queue_positions` is False for the session-root copy, so its status
  reverts to `WAITING`.
- The clan-container copy aggregates the live members.

Before `f843bd8100`, this pass only produced a header snapshot. That commit added
`_apply_agents_capacity_row_updates()`, which copies `status` from those copies back
onto the live rows. The capacity-report feed installs a report at least every 10 s
(`_CAPACITY_REPORT_POLL_SECONDS`), and each install schedules this roster refresh. So
within 10 s of every full load:

1. The first refresh demotes the session root to `WAITING`.
2. The next refresh demotes the clan, because the clan now aggregates the demoted live
   root.

A synthetic reproduction confirmed the cycle. It used a clan with a DONE session root, a
slot-waiting turn, and a WAITING sibling, followed by a full load and then two roster
refreshes with status write-back:

```
after full load          CLAN QUEUED   root QUEUED   code QUEUED   sib WAITING
after roster refresh #1  CLAN QUEUED   root WAITING  code QUEUED   sib WAITING
after roster refresh #2  CLAN WAITING  root WAITING  code QUEUED   sib WAITING   <- screenshot
```

With in-batch references (copy_agent_graph), every refresh keeps
`CLAN/root/code = QUEUED`.

**Secondary defect (ordering):** both `_apply_runner_capacity_snapshot()` and
`_refresh_runner_slot_context_fallback()` aggregate clan containers inside the same
per-row loop that derives `QUEUED` for non-waiter rows. A container listed before its
session-root member aggregates the root's pre-promotion status. Real slot waiters are
promoted before that loop; a session root is not, because it is only queued _through_
its turn. Full loads hide this, because `project_and_fold_rosters()` rebuilds the
containers afterwards. The roster refresh has no rebuild step, so the clan stays one
refresh (up to 10 s) behind its session root. Even the live, non-copied rows showed
`CLAN WAITING / root QUEUED` after one pass.

This is TUI presentation logic (Agents-tab row overlays), not shared backend behavior,
so it stays in this repo and does not touch `sase-core`.

## Changes

### 1. A cheap relinked roster copy (`src/sase/ace/tui/models/_agent_graph.py`)

Add a public helper, for example `copy_agent_rows_relinked(agents) -> list[Agent]`, and
export it in `__all__`. It should:

- Make one `copy.copy()` per distinct row. Memoize by `id()` so a row that appears twice
  stays a single copy, and return the list in input order.
- Rewrite each copy's `Agent`-reference fields so they point at the in-batch copies:
  `followup_agents`, `runtime_children`, `wait_display_source`, `status_display_source`,
  `agent_session_container`, and `retry_chain_siblings`.
  - Single `Agent` values map through the memo.
  - Lists map element-wise into a **new** list, so the copy never shares a list object
    with the live row.
  - A reference to a row outside the batch is kept as-is. Consumers only read those
    rows.
- Keep the field names in one module-level tuple. Extend the module docstring to say
  when to use this helper rather than `copy_agent_graph()`. The consumer must only
  reassign row attributes (as the runner-slot pass does) and never mutate nested leaf
  containers in place. The helper is about as cheap as a plain shallow copy: measured ~7
  ms per 600 rows. `copy_agent_graph()` took ~160 ms per 600 rows, and the roster
  refresh runs twice per report install, at least every 10 s even when idle. That cost
  matters under the TUI perf rules ("periodic ticks revalidate; recomputes get a longer
  cadence").

### 2. Use it wherever the slot pass runs on copies

- `_run_agents_capacity_refresh_from_roster()` in
  `src/sase/ace/tui/actions/agents/_loading_filter.py`: replace both
  `[copy(agent) for agent in roster]` (the estimate rows and the report rows) with the
  helper. Both lists can be written back: the estimate rows are returned as
  `row_updates` when no report exists. Keep the `copy` import if
  `_snapshot_agents_for_local_display()` still uses it.
- `prepare_loaded_agents_apply_boundary()` in
  `src/sase/ace/tui/actions/agents/_loading_compute.py`: switch the
  `estimate_agents = [copy(agent) for agent in prep.filtered_agents]` witness to the
  helper. The estimate snapshot then derives the same session and clan statuses as the
  authoritative pass, which keeps capacity drift telemetry honest. Drop the local
  `from copy import copy` if nothing else uses it.
- Do not change `_apply_agents_capacity_row_updates()`. Once the copies derive the right
  status, the existing identity-keyed write-back is correct.

### 3. Aggregate clan containers after their members (`agent_runner_slots.py`)

In both `_apply_runner_capacity_snapshot()` and
`_refresh_runner_slot_context_fallback()`:

- Keep the first `for agent in agents` loop, which sets the per-row runner fields,
  capacity accounting, and the non-container
  `runner_slot_display_status(... wait_display_source ...)` derivation.
- Move the `apply_clan_container_status(agent, clan_members(agent), fallback=...)` call
  into a second loop over clan containers that runs after the first loop completes. Each
  container then aggregates fully settled member statuses in a single pass.
- In `_apply_runner_capacity_snapshot()`, keep `_apply_clan_capacity_aggregates()` after
  both loops.
- Update the fallback's "Promote real waiters before refreshing synthetic clan
  aggregates below" comment to describe the two-phase order. Containers are never
  members of other containers (see `clan_members()`), so two phases suffice.

### 4. Tests

Use the existing harnesses: `_agent` in `tests/ace/tui/_agent_runner_slots_helpers.py`,
`_apply_status_overrides` from `sase.ace.tui.models.agent_loader`, and
`project_clan_tree`. Put the roster-refresh regression in a new focused module, for
example `tests/ace/tui/test_capacity_roster_refresh_queued_status.py`, rather than
growing `tests/test_agents_tab_refresh_paths.py`. If a minimal `MachineCapacityReport`
builder is needed, lift it into a shared helper rather than duplicating `_report()` from
`tests/ace/tui/test_machine_capacity_report_feed.py`.

1. **Helper unit tests** (extend `tests/test_agent_graph_copy.py`):
   - In-batch `wait_display_source`, `runtime_children`, and `followup_agents` resolve
     to the copies.
   - An out-of-batch reference is preserved.
   - Duplicate input rows stay aliased.
   - Reassigning status or appending to a copy's relinked list leaves the originals
     untouched.
   - A guard test asserts that the helper's field tuple equals
     `{f.name for f in dataclasses.fields(Agent) if re.search(r"\bAgent\b", str(f.type))}`.
     A future `Agent`-reference field then fails loudly instead of silently aliasing
     live rows.
2. **Ordering regression** (`tests/ace/tui/test_agent_runner_slots_agent_sessions.py`):
   - Setup: a clan (`agent_clan`/`agent_clan_generation` set on every row) holding a
     DONE session root, a slot-waiting turn (`parent_timestamp` = root,
     `slot_requested_at` set), and a WAITING sibling. Pass the rows through
     `_apply_status_overrides` and `project_clan_tree`, so the container precedes the
     root.
   - One `refresh_runner_slot_context()` pass must leave root, turn, and container all
     `QUEUED`, and the container's `wait_display_source` must resolve to the queued
     turn.
   - Parametrize over the fallback path (`effective_limit=None`), the estimate path
     (`effective_limit=10`), and the authoritative path. The authoritative variant
     passes
     `raw_snapshot={"occupied_lanes": ..., "effective_limit": ..., "waiters": [{"artifact_dir": turn.artifacts_dir, "queue_position": 1, ...}], "claims": [], "accounting": [], "accounting_valid": True}`.
   - This test fails today: the container reads `WAITING`.
3. **Roster-refresh regression** (new module):
   - Drive the real `_run_agents_capacity_refresh_from_roster()` on a `FakeAgentApp`
     (`tests/_agents_tab_query_helpers.py`). Its `_agents`, `_agents_with_children`, and
     `_agents_capacity_with_children` should hold the projected clan rows after a
     full-load-equivalent pass (one `refresh_runner_slot_context()` and then
     `project_clan_tree()`, matching `prepare_loaded_agents_apply_boundary()`).
   - Set `_machine_capacity_report` to a report whose waiters list the turn.
   - Monkeypatch `sase.config.core.get_max_running_agents` and
     `sase.core.agent_hold_facade.active_agent_hold_records`, as the existing tests in
     `tests/test_agents_tab_refresh_paths.py` do.
   - Run the refresh for two consecutive generations. Assert that the live root, turn,
     and container all stay `QUEUED`, and that the live root's `wait_display_source` is
     still the live turn row, not a copy.
   - Add one variant with `_machine_capacity_report = None`, which covers the
     estimate-rows write-back.
   - Both variants fail today (`root WAITING`, then `CLAN WAITING`).

## Verification

- Run `sase tool run check` and fix any lint or test fallout. Do not run
  `just check-full`; nothing here explicitly requires it.
- Optional live check, read-only: an already-open TUI keeps running the code it imported
  at start (`src/sase/ace/tui/stale_running_code.py`). A fix becomes visible only in a
  restarted TUI. Do not restart or kill the user's running TUI. Tell the user a restart
  is needed to see the fix.

## Out of scope

- `sase agent list` also omits this kind of live, slot-queued monitor follow-up turn.
  The cause is `is_visible_runner_slot_child` / `is_runner_slot_user_agent_record`,
  which assume serial session children never wait at admission. That gap is recorded as
  a `DISCOVERED ISSUE` note on the in-progress capacity epic `sase-1om` (its `.5` phase
  owns that predicate). Do not change the CLI listing here.
- The roster refresh does not re-point a clan container's `wait_display_source` on live
  rows. Full loads rebuild containers, so the clan rank badge is refreshed there; this
  plan only fixes the status word.
