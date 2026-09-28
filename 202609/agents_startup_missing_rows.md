---
tier: tale
title: Load the whole visible Agents roster at startup
goal:
  On TUI startup the Agents tab paints every non-dismissed agent node and clan in one
  read, never silently shows a partial roster, and completes any knowingly partial
  roster promptly behind a visible loading indicator.
size: medium
proposed_by: bbugyi200.athena.0ts
create_time: 2026-09-28 16:26:50
status: wip
---

# Plan: Load the whole visible Agents roster at startup, and say so when it is partial

## Symptom

Right after `sase ace` starts, whole agent clans and many clan members are missing from
the `@default` / `@epic` / `@job` tribe panels. Nothing on screen says rows are still
loading. The rows reappear minutes later, or in some sessions not at all. The user's
screenshot (2026-09-28 16:02 EDT) was taken about two minutes after a 16:00:40 startup,
and it is still missing clans.

## Root cause (measured read-only on the real home dir, 2026-09-28)

Three defects stack.

### 1. First paint is a newest-N viewport window — an arbitrary slice of a grouped roster

`agents_viewport_for_load` (`src/sase/ace/tui/actions/agents/_loading_disk_viewport.py`)
windows every Agents-tab read, including the very first one, to
`start_row + visible_rows + 2 × visible_rows` (≈174 rows on a 66-row terminal). The
index returns every active candidate plus the newest ≈174 completed candidates, and
`load_tiered_agents` then caps the result to 174 rows. That prefix assumes a flat
newest-first list. The Agents tab renders tribe panels → clans → status groups, so the
newest 174 rows are not the rows on screen.

Measured with
`load_agents_from_disk_with_state(dismissed, patch_snapshot=[], search_query="NOT machine:apollo", viewport=...)`
(the same entry point the TUI worker uses):

| read                                          | top-level nodes | clans                 | rows | warm median |
| --------------------------------------------- | --------------- | --------------------- | ---- | ----------- |
| viewport window (first paint today)           | 57              | 3                     | 174  | 952 ms      |
| unwindowed Tier 1 (the completion read today) | 111             | 11, 6 of them partial | ~600 | 807 ms      |
| whole visible inbox (every non-hidden row)    | 152             | 12                    | 766  | ~1.1 s      |

At the index level the window saves nothing: the raw windowed query takes 160 ms, the
unwindowed Tier 1 query 159 ms, and a query for every visible row 219 ms (393 records).

The only thing that completes the window is the one-shot `startup_prefix_completion`
refresh (`_loading_refresh_polling.py`). It waits for 2 s of input quiet, then queues
behind the process-local artifact-index operation lock, which other startup index work
also holds (the dismissed-index sync and the Tier-1 revalidate). In the 16:00:40 session
it landed 38 s after first paint: `~/.sase/logs/tui_agent_loads.jsonl` shows
`source=startup_prefix_completion` with `disk=34.2 s`.

### 2. Even the "completed" Tier 1 set caps completed history at the 200 newest records

`query_artifact_index_for_loader` (`src/sase/ace/tui/models/_agent_loader_artifacts.py`)
sends `recent_completed_limit=_TIER1_RECENT_COMPLETED_LIMIT` (200) on unwindowed reads.
The index held 316 visible completed candidates, so 116 records never load (~168 rows,
41 top-level nodes). Long-lived epic clans straddle that cutoff:

- `sase-1b2` shows 2 of 38 members.
- `sase-1bc` shows 22 of 34.
- `sase-1b1.8` shows 0 of 10, so the whole clan is missing.

`docs/ace.md` calls this set the "visible inbox" ("active rows plus recent completed").
`test_production_oracle_settles_full_history_beyond_tier1_cap` calls the gap
"self-resolving", but on a healthy index nothing resolves it:

- `should_arm_full_history_reconcile` returns False for a healthy Tier 1 load.
- The rows only arrive if an unrelated repair, delta-loss, or pushdown-miss arms the
  Tier 2 reconcile, and even then only after 30 s of input quiet. In the screenshot
  session that happened at 16:05:48, five minutes after startup.

That is why the symptom is intermittent. It is also why the screenshot, taken after the
prefix completion had already landed, is still missing clans.

### 3. Nothing tells the user the roster is partial

The info panel's `Agents: …` treatment ends at the first apply. The only partial-roster
hint (`filtered on recent history; loading full history...` in
`src/sase/ace/tui/widgets/agent_info_panel.py`) renders only when a committed filter's
pushdown missed (`query_incomplete`). A windowed or capped roster renders with no
signal.

## Design

For a committed query, the Agents roster has one correct universe: the **visible
inbox**, meaning every active row plus every non-hidden (non-dismissed) completed row.

1. A **baseline** load is any load with no applied same-query roster yet: the first
   load, a committed-query change, or recovery after a partial replacement. A baseline
   load reads the whole visible inbox in one cached index read. Viewport windows are
   used only for **patch** loads that merge over an existing same-query baseline.
   `merge_incomplete_load_after_complete_history` already patches same-query bounded
   partials over the cached roster.
2. The completed tier's cap is a first-paint safety valve, not a working-set definition.
   When the cap truncates the inbox, the load says so.
3. Whenever the applied roster is knowingly less than the visible inbox for the current
   query, the header says so. A completing load is then scheduled promptly (2 s input
   quiet), not opportunistically.

This intentionally revisits one constraint from the Aug 2026
`agents_window_completed_starvation` plan: "do not make ACE's normal refresh
unwindowed". Normal refreshes stay windowed patches. Only baseline loads go unwindowed,
because the window does not make first paint cheaper (measured above), and a window with
no roster to patch over is simply wrong under grouped rendering. The core
`select_windowed_records` budget fix from that plan stays as is.

**No sase-core change is needed.** Windowed cached index reads already report `has_more`
and `completed_candidate_count` (`AgentArtifactIndexWindowWire`), which is exactly the
truncation signal a baseline read needs. Do not re-implement index selection in Python.

### Alternatives rejected

- **Loading indicator only, firing the prefix completion immediately.** Keeps the
  pop-in, and the completion is still capped at 200 completed, so clans stay partial
  (defect 2).
- **Widening `AgentsViewport`.** Any newest-N slice is still arbitrary under
  tribe/clan/status grouping.
- **Always running Tier 2 at startup.** Tier 2 is an O(archive) source reconcile (11–20
  s on this host) and violates `tui_perf.md` rule 9. It stays the authority for
  complete-history claims and the remedy for truncated or fallback rosters only.

## Part 1 — Loader: baseline reads return the visible inbox and report truncation

Files: `src/sase/ace/tui/models/_agent_loader_artifacts.py`,
`src/sase/ace/tui/models/agent_loader.py`,
`src/sase/ace/tui/actions/agents/_loading_apply_history.py`.

1. Add `_TIER1_VISIBLE_COMPLETED_LIMIT = 2000`. Its comment should say it is a
   first-paint safety valve (about 6× the measured inbox of 316 completed candidates),
   not a working-set definition.
   - Use it instead of `_TIER1_RECENT_COMPLETED_LIMIT` as `recent_completed_limit` in
     the Agents-tab index query built by `query_artifact_index_for_loader`, for both
     cached and revalidate reads.
   - Keep `_TIER1_RECENT_COMPLETED_LIMIT` for the bounded source-scan fallbacks
     (`_TIER1_FALLBACK_SCAN_OPTIONS`) and for `artifact_snapshot_for_live_plan_load`.
     Those caps bound an O(archive) directory walk, which is a different concern.
2. Define a baseline index read as a Tier 1 **cached** read with
   `requested_limit is None`. Send it as a windowed query with
   `window_limit=_TIER1_VISIBLE_COMPLETED_LIMIT` so Rust reports truncation. Windowed
   mode requires cached freshness, active plus completed tiers, and no full history
   (`should_use_windowed_candidate_query` in sase-core). Revalidate reads cannot be
   windowed, so they stay unwindowed with the new cap.
3. Map the load state:
   - `bounded_prefix` is True only when the caller asked for a viewport window
     (`requested_limit is not None`) and the index returned a window. A baseline read is
     never a bounded prefix, even though it used a window internally.
   - For a baseline read, `truncated = index_window.has_more` and
     `complete_visible_inbox = not truncated`.
   - `requested_limit`, `returned_count`, and `has_more` keep describing viewport
     windows only. A baseline read reports `has_more=False` so
     `_maybe_schedule_agents_viewport_expansion` never fires off one.
   - `should_arm_full_history_reconcile` (`_loading_apply_history.py`) must also arm for
     an index-backed load whose `truncated` is True. Today it only arms incomplete
     inboxes from non-index reads.
4. `load_tiered_agents` (`agent_loader.py`): `query_incomplete` used to mean "a
   pushdown-miss query was evaluated only against recent history". Set it only when the
   read was a viewport-bounded read or a truncated baseline read. A non-truncated
   baseline read saw the whole visible inbox, so it is not query-incomplete. The
   full-history path is unchanged.
5. Confirm that baseline and viewport reads get distinct `_ARTIFACT_SNAPSHOT_CACHE` keys
   (the key already includes the query wire).

## Part 2 — Refresh orchestration: window only when patching a same-query baseline

Files: `src/sase/ace/tui/actions/agents/_loading_disk_viewport.py`, `_loading_apply.py`,
`_loading_apply_history.py`, `_loading_state.py`, `_loading_refresh.py`,
`_loading_refresh_delta.py`, `_loading_refresh_polling.py`, `_loading_disk_full.py` (all
under `src/sase/ace/tui/actions/agents/`), plus
`src/sase/ace/tui/actions/_state_init_agents.py` and
`src/sase/ace/tui/actions/_event_countdown.py`.

1. Add a roster latch, `_agents_roster_complete_query_key`: declare it in
   `_loading_state.py` and initialize it to `None` in `_state_init_agents.py`. In
   `_apply_loaded_agents`, after the merge decision:
   - **Set** it to the load's history query key (`history_query_key_for_load`) when the
     applied load is roster-complete. That means either `load_state.complete_history`,
     or all of: `not bounded_prefix`, `complete_visible_inbox`, `not truncated`, and
     `not query_incomplete`.
   - **Leave it unchanged** for artifact deltas and for bounded or partial loads that
     were merged over a same-query cached roster.
   - **Clear** it when an applied partial load replaced the roster instead of patching
     it (no same-query cache).

   Put the roster-complete predicate beside `has_complete_history_for_load_query` in
   `_loading_apply_history.py` so it is unit-testable.

2. `agents_viewport_for_load` returns `None` (a baseline read) whenever
   `_agents_roster_complete_query_key != current_agents_history_query_key(app)`. This
   one rule covers the first load of a session, a committed-query change, and recovery
   after a partial replacement. Otherwise its behavior is unchanged.
3. Delete the one-shot startup prefix completion; rule 2 subsumes it.
   - Remove `_arm_startup_prefix_completion`,
     `_maybe_trigger_startup_prefix_completion`, and the `STARTUP_PREFIX_COMPLETION_*`
     constants, including their re-exports from `_loading_refresh.py`.
   - Remove the `_agents_prefix_completion_*` and
     `_agents_refresh_{pending,scheduled,active}_prefix_completion` flags.
   - Remove the `complete_prefix` parameter everywhere it is threaded:
     `_schedule_agents_async_refresh`, `_run_agents_async_refresh`, the pending-work
     drain in `_loading_refresh_delta.py`, `reschedule_stale_agent_query_load`, and
     `_load_agents_async`.
   - Remove the countdown hook in `_event_countdown.py` and the arm call in
     `_loading_apply.py`.
   - Delete `tests/ace/tui/test_lazy_tier2_prefix_completion.py`. Adjust the other tests
     that reference the removed names: `tests/ace/tui/_lazy_tier2_reconcile_helpers.py`,
     `tests/ace/tui/test_loading_callbacks.py`,
     `tests/ace/tui/test_agents_refresh_coalescing.py`,
     `tests/test_agents_tab_refresh_paths.py`, and
     `tests/test_agents_tab_graph_isolation.py`.
   - Grep `src/` and `tests/` for leftovers, then let symvision confirm nothing is left
     unused.
4. Keep patch loads cheap. Windowed refreshes after a baseline stay windowed and are
   merged by `merge_incomplete_load_after_complete_history` exactly as today. Do not
   widen `AgentsViewport` and do not add a new refresh code path (`tui_perf.md` rule 5).

## Part 3 — Prompt completion when the roster is partial

Files: `_loading_apply.py`, `_loading_refresh_polling.py`, `_loading_state.py`, and
`_state_init_agents.py`.

1. After an apply, if the latch does not match the current query key (the roster is
   partial), arm a roster completion that fires after
   `ROSTER_COMPLETION_INPUT_QUIET_THRESHOLD_S = 2.0` s of input quiet. It fires from the
   existing countdown-tick trigger path, not after the 30 s
   `TIER2_RECONCILE_INPUT_QUIET_THRESHOLD_S`. Pick the completing load by cause:
   - **Transient lock busy**
     (`repair_reason == "artifact_index_lock_busy_bounded_fallback"`): schedule a normal
     refresh. Part 2 rule 2 makes it a baseline read.
   - **Every other cause** (index missing, index query failed, truncated inbox, failed
     schema rebuild, a query-incomplete read): arm the existing Tier 2 reconcile with
     the short threshold. For example, store the threshold at arm time
     (`_agents_history_reconcile_quiet_s`), where re-arming may only lower it.

   Reconciles armed while the roster is already complete (repair-only) keep the 30 s
   default.

   Keep every existing guard: `_agents_loading`, `_agents_refresh_scheduled`,
   `_agents_artifact_delta_scheduled`, the navigation gate, and the
   `schema_rebuild_in_flight` suppression in `_apply_loaded_agents`. Clear pending flags
   before scheduling, the way the other triggers do. The work stays on the pump-free
   refresh path (`tui_perf.md` rules 2, 5, and 13).

2. Do not arm `tier1_index_revalidate` (`_arm_tier1_index_revalidate_reconcile`) while
   the roster is partial. It is 300 s-cadence maintenance, and at startup it held the
   index lock for 17.6 s in front of the completion read. Arm it from the apply that
   makes the roster complete.

## Part 4 — Header indicator

Files: `src/sase/ace/tui/widgets/agent_info_panel.py`,
`src/sase/ace/tui/actions/agents/_display_detail_info.py`, and `docs/ace.md`.

1. Replace the query-only `search_query_partial_history` plumbing with a roster-loading
   flag computed in `_display_detail_info.py`. The flag is True when at least one agents
   load has applied and `_agents_roster_complete_query_key` differs from the current
   query key.
2. Render the flag on the header count line so it is visible with no filter committed,
   for example `⟳ loading agents…`, visible but unobtrusive and matching the header's
   existing styles.
   - With a committed filter, show one filter-specific message instead, e.g.
     `filtered on partial history; loading full history…`. Never show both.
   - The indicator disappears on the apply that sets the latch.
   - Before the first apply, the existing `Agents: …` treatment still applies.
3. Make sure the header repaints on the apply that flips the latch. The finalize path
   normally calls `_update_agents_info_panel()`; verify that, and add an explicit call
   only if the latch can flip without one.
4. Update `docs/ace.md`:
   - The visible-inbox paragraph (~line 1737): the visible inbox is now every active row
     plus every non-hidden completed row, subject only to the first-paint safety cap.
   - The query-history paragraph (~line 2855): the new indicator wording, and the fact
     that pushdown-miss queries are only partial when the read was truncated.

## Part 5 — Tests

1. **Loader tests** in `tests/test_agent_loader_query_window.py` or a sibling. Use
   synthetic fixtures (see `tests/perf/agent_load_tiering_fixture.py`); never read the
   real `~/.sase` index from a test.
   - A baseline read (no viewport) with more than 200 visible completed candidates
     returns all of them, with `bounded_prefix=False`, `truncated=False`, and
     `complete_visible_inbox=True`.
   - With `_TIER1_VISIBLE_COMPLETED_LIMIT` monkeypatched small, a baseline read reports
     `truncated=True` and `complete_visible_inbox=False`, and
     `should_arm_full_history_reconcile` arms.
   - A viewport read still reports `bounded_prefix=True` and caps its rows.
   - A pushdown-miss query does not set `query_incomplete` on a non-truncated baseline
     read, but still sets it on a viewport read.
2. **Oracle tests.** Update
   `test_production_oracle_settles_full_history_beyond_tier1_cap`
   (`tests/test_agent_load_tiering_production_oracle.py`) and
   `tests/perf/_agent_load_tiering_oracle.py`:
   - Rows past the old 200 cap are now part of Tier 1.
   - The beyond-cap case monkeypatches the visible cap and asserts that truncation is
     reported and settled by full history.
   - Keep the fixture in the fast test lane.
3. **Orchestration tests**, following the `_FakeRefreshApp` pattern in
   `tests/ace/tui/test_lazy_tier2_reconcile.py`:
   - The first load and a committed-query change get `viewport=None`; the next refresh
     for the same query gets a viewport.
   - A bounded patch merged over a same-query baseline keeps the latch; a partial load
     that replaces the roster clears it.
   - A roster-partial apply arms completion with the 2 s threshold.
   - Lock-busy selects a baseline retry, and index-missing selects Tier 2.
   - Repair-only arming on a complete roster keeps 30 s.
   - `tier1_index_revalidate` does not arm while the roster is partial.
4. **Header tests.** `AgentInfoPanel` renders the loading hint with and without a
   committed filter, and drops it once the roster is complete.
5. **Regression scenario** for the reported bug. Build a fixture with more than 200
   visible completed rows in clans whose members straddle the old cap, and use a small
   viewport. After the first apply, every clan member is on the roster and the latch
   matches the query.

## Part 6 — Verification

1. Read `sase/memory/lint_and_test.md` first. Run `just install` if the venv may be
   stale, then `just check`.
2. Run this read-only reproduction on the real home dir before and after the change, and
   report both runs in the final message. It uses cached freshness only, so it never
   writes the index.

   ```python
   import collections, statistics, time
   from sase.ace.dismissed_agents import load_dismissed_agents
   from sase.ace.tui.actions.agents._loading_helpers import load_agents_from_disk_with_state
   from sase.ace.tui.data_providers import AgentsViewport

   dismissed = load_dismissed_agents()
   def load(vp):
       t = time.perf_counter()
       r = load_agents_from_disk_with_state(
           set(dismissed), patch_snapshot=[], search_query="", viewport=vp
       )
       return time.perf_counter() - t, r
   load(None)  # warm up
   for label, vp in (("baseline", None), ("viewport", AgentsViewport(0, 58, 116))):
       runs = [load(vp) for _ in range(4)]
       r = runs[-1][1]
       top = [a for a in r.all_agents if not a.parent_workflow]
       clans = {a.agent_clan for a in r.all_agents if getattr(a, "agent_clan", None)}
       print(label, f"{statistics.median(t for t, _ in runs) * 1000:.0f}ms",
             len(r.all_agents), "rows", len(top), "top", len(clans), "clans",
             r.load_state.bounded_prefix, r.load_state.truncated)
   ```

   Expected result: the baseline read's clan count matches the whole visible inbox, and
   the viewport read is still bounded. If the baseline read's median exceeds the
   viewport read's median by more than about 300 ms, report the number. Do not
   reintroduce a startup window to hide it.

3. If you can restart the installed ACE, check the new session's entries:
   - In `~/.sase/logs/tui_startup.jsonl`, `agent_row_count` should now be the
     visible-inbox size; report `agents_ready_seconds` before and after.
   - `~/.sase/logs/tui_agent_loads.jsonl` should contain no `startup_prefix_completion`
     source.

   If a restart is not possible, say so instead of claiming it.

## Non-goals

- No sase-core changes: not to windowing, not to the revalidate or stale-row repair
  path.
- No changes to dismissed/hidden semantics, clan grouping, tribe panel rendering, or
  `merge_incomplete_load_after_complete_history`'s patch rules.
- General startup contention on the process-local index lock (dismissed-index sync,
  maintenance) beyond Part 3.2's revalidate deferral. If it still delays startup
  measurably, file a task bead with `/sase_new_task` rather than widening this plan.
- A visible inbox larger than the safety cap: honest truncation, the indicator, and a
  prompt Tier 2 are enough.

## Acceptance criteria

1. After a restart, first paint shows every non-dismissed agent node and clan in the
   artifact index. On the measured data that is 152 top-level nodes and 12 clans, not 57
   and 3. It needs no input-quiet wait and no Tier 2 reconcile.
2. Whenever a roster is partial (fallback scan, truncated inbox, schema rebuild,
   lock-busy, or a committed-query change still loading), the header shows a loading
   indicator. The indicator clears on the completing apply, and the completing load is
   scheduled within about 2 s of input quiet.
3. Refreshes after a baseline stay windowed patches. The startup prefix-completion path
   is gone, and no new refresh code path exists.
4. `just check` is green, and the before/after reproduction numbers are reported.
