---
tier: epic
title: Agents tab freshness on large-archive hosts
goal: "The Agents tab reflects load/capacity, unread notification, and node status
  changes within seconds on hosts with tens of thousands of stored agents, without
  adding per-tick TUI cost.

  "
phases:
  - id: pulse-delta
    size: medium
    title: Stop refresh-pulse writes from poisoning the bounded artifact-delta path
    depends_on: []
    description:
      "pulse-delta: classify project-level .ace_refresh_pulse watcher paths as a pure
      freshness kick instead of an unknown_watcher_path fallback (leaving
      per-agent-directory pulses exact for sase-zr.7.3), and fix sase-zc by routing the
      two misplaced pulse writers through the existing touch_shell_refresh_pulse helper."
  - id: capacity-fresh
    size: medium
    title:
      Give the load/capacity indicator a cheap refresh path independent of broad loads
    depends_on: []
    description:
      "capacity-fresh: recompute the runner-capacity snapshot from the in-memory roster
      on tab entry with a guard so older in-flight loads cannot overwrite it,
      token-track limit-override/holds changes, and fix the stale agent-info metrics
      memo key."
  - id: tick-diet
    size: medium
    title: Take the federation attention RPC off the auto-refresh critical path
    depends_on: []
    description:
      "tick-diet: make per-tick fleet-attention polling cache-backed with a longer
      network recompute cadence, run it concurrently with local surfaces, and add tick
      trace counters for it."
  - id: broad-load-diet
    size: large
    title: Cut broad Tier 1 load and post-apply warmup cost on large archives
    depends_on: []
    description:
      "broad-load-diet: profile then cache/coalesce the unwindowed per-load
      project/patch sweeps, the bead-confirmation/monitor/live-hint warmups, the
      detail-header rebuilds, and evaluate incremental Tier 1 index revalidation."
  - id: ui-hitches
    size: medium
    title: Remove Agents-tab UI-thread hitches (unread ack, info-panel countdown)
    depends_on: []
    description:
      "ui-hitches: move the unread-ack notification-store mutation off the UI thread
      with optimistic row updates, and make the 1 s info-panel countdown patch only the
      countdown segment."
  - id: inflight-status
    size: medium
    title: Bounded marker polling so in-flight node status converges without broad loads
    depends_on:
      - pulse-delta
    description:
      "inflight-status: extend the STARTING-row 1 s stat poll to all in-flight rows so
      marker transitions schedule exact artifact-delta refreshes within seconds even
      when inotify misses events."
  - id: verify
    size: medium
    title: Before/after verification on athena and regression coverage
    depends_on:
      - pulse-delta
      - capacity-fresh
      - tick-diet
      - broad-load-diet
      - ui-hitches
      - inflight-status
    description:
      "verify: capture before/after trace, perf, and stall data on athena against the
      acceptance targets, run the benches, and record results on the epic bead."
proposed_by: bbugyi200.athena.0mc
create_time: 2026-09-17 10:59:38
status: wip
---

# Agents Tab Freshness on Large-Archive Hosts

## Problem

On athena (≈11,400 agent artifact dirs on disk, 10,431 in the `gh_sase-org__sase`
project alone), the Agents tab takes far too long to reflect changes: the load/capacity
indicator, unread notification badges, and node status all lag by many seconds to
minutes. The goal is to keep the tab fresh **without** adding per-tick work (see
`sase/memory/tui_perf.md`, which every phase worker MUST read via
`sase memory read tui_perf.md` before touching this code).

## Evidence (captured 2026-09-17 on athena)

- `~/.sase/logs/tui_agent_loads.jsonl`: repeated `tui_agent_load_slow` events —
  `load_kind=full` broad loads with `disk` stage 2.2–2.9 s (total ≈3–3.5 s) firing every
  30–50 s from `source=auto_refresh`; even a 6-agent `artifact_delta` load hit
  `disk=2.5 s, prep=2.4 s`.
- `~/.sase/perf/tui_trace.jsonl` (SASE_TUI_TRACE run, 2026-09-15): `refresh.auto_tick`
  p50 1528 ms / p95 4057 ms / max 5293 ms (n=116); `agents.load_from_disk` p50 1761 ms /
  p95 9780 ms / max 11982 ms — including `index_freshness=cached` loads at 9–12 s;
  `agents.bead_confirmation_warmup` up to 10794 ms;
  `widget.prompt_panel.build_detail_header_summary` up to 10333 ms with
  `cache_state=warm`; `agents.monitor_reconcile` max 4783 ms; `agents.live_hint_refresh`
  max 2971 ms; `relations.index.patches` p50 420 ms. Ticks reloading only
  `axe,notifications` still cost 4–4.8 s, some with `axe_file_opens` of 437–444.
- `~/.sase/logs/tui_stalls.jsonl`: a 2.338 s main-thread hitch inside
  `_clear_agent_unread_and_dismiss_notifications` →
  `notifications/store.py:461 dismiss_agent_completion_notifications_matching_agents` →
  synchronous Rust `apply_notification_state_update` on the UI thread; and a 2.152 s
  hitch under the 1 s countdown tick in `_update_agents_info_panel_impl` →
  `AgentInfoPanel.update_countdown_only` → full `_render_panel_text` rebuild.
- Prior benchmark
  `~/.sase/perf/agent_load_tiering_sase-zu.8.5_athena_real_20260913.json`: with this
  archive, `index_bounded` p50 ≈980 ms, `production_full_history` p50 ≈2989 ms, raw
  `source_scan` p50 ≈10866 ms.

## Root causes

1. **Refresh-pulse writes poison the delta path.** Many writers touch an
   `.ace_refresh_pulse`. Most go through the shared
   `touch_shell_refresh_pulse(project_name)` helper
   (`src/sase/shells/settlement.py:200`), which writes
   `<project>/artifacts/.ace_refresh_pulse`: gate-shell settlement and handoff launch
   (`src/sase/gate_shell/settlement.py:220`, `handoff_launch.py:214`), monitor
   settlement via `touch_monitor_refresh_pulse` (`src/sase/monitor/supervise.py`,
   `reconcile.py`, `proc_adapter.py`), gate-decision acceptance
   (`src/sase/notification_gates/decision.py:247`), and agent done/exec markers
   (`src/sase/axe/runner_artifacts.py:295`, `run_agent_exec_markers.py:51`). The epic
   launch handoff (`src/sase/bead/epic_launch_handoff_monitor.py:89`) writes the same
   path inline. The pending-handoff mutation (`src/sase/shells/handoff.py:90`) and plan
   proposal (`src/sase/main/plan_propose_handler.py:239`) compute their own path (see
   below). That path passes `artifact_path_affects_agents`
   (`src/sase/ace/tui/actions/event_refresh/_artifact_paths.py`) but maps to no exact
   artifact dir, so `_enqueue_agent_artifact_delta_paths`
   (`src/sase/ace/tui/actions/event_refresh/_artifact_delta.py:81`) sets
   `_dirty_agent_artifact_fallback_reason="unknown_watcher_path"`. That disables the
   bounded delta (`agent_delta_ready` requires `fallback_reason is None`,
   `_auto_refresh.py:261-266,338-344`) and forces a broad load that is tab-gated
   (`_auto_refresh.py:274-280`) and floored at `AGENTS_LOAD_MIN_INTERVAL_SECONDS=5`.
   Net: the pulse — a mechanism designed to nudge the Agents tab — routinely converts a
   cheap exact refresh into a 1.8–12 s broad load (on-tab) or total staleness until tab
   entry / the 300 s sanity pass (off-tab). Worse, `handoff.py` and
   `plan_propose_handler.py` compute the pulse path as `Path(artifacts_dir).parents[1]`,
   which for the sharded `artifacts/ace-run/<YYYYMM>/<DD>/<ts>` layout lands at the
   month dir instead of `<project>/artifacts/`, where neither the surface-token probe
   (`_surface_tokens.py` stats only `<project>/artifacts/.ace_refresh_pulse`) nor an
   unwatched shard sees it. That wrong-path bug is already tracked as task sase-zc.
2. **The load/capacity indicator has no cheap refresh path.** `RunnerCapacitySnapshot`
   is written in exactly one place — `actions/agents/_loading_apply.py:459` — inside an
   agents load apply. `_refilter_agents` (the tab-switch fast path,
   `_app_watchers.py:125-143`) and the 1 s countdown tick only re-render the stale
   snapshot. Config limit / hold changes never dirty the agents surface.
   `_agent_info_metrics` is memoized on `(id(self._agents), frozenset(unread_ids))`
   (`actions/agents/_display_detail_info.py:41-78`), so in-place status mutations keep
   serving stale counts.
3. **The auto-refresh tick is serial and fronted by a network RPC.**
   `_poll_fleet_attention_inventory` runs
   `fetch_remote_attention_inventory(cache_only=False)` on every tick
   (`_auto_refresh.py:295-301`, `actions/agents/_remote_attention.py:57-102`) before any
   agents work. Athena has a configured dispatch machine (apollo via tailnet gateway),
   federation is enabled, and the default worker timeout is 5 s
   (`src/sase/dispatch/federation/_settings.py:28`), so every 10 s tick can spend
   seconds in the network before the agents surface even gets a chance. This violates
   the established "revalidate on ticks; recompute on a longer cadence" rule (tui_perf
   rule 10).
4. **Broad Tier 1 loads and their follow-on warmups are too expensive on this archive.**
   Beyond the bounded index query, every broad load runs unwindowed O(#projects +
   #patches) sweeps (`models/agent_loader.py:268-350` — `get_all_project_files()`,
   per-patch HOOKS/MENTORS/COMMENTS loops; `relations.index.patches` p50 420 ms), and
   each apply schedules warmups (`agents.bead_confirmation_warmup` up to 10.8 s,
   `agents.monitor_reconcile` up to 4.8 s, `agents.live_hint_refresh` up to 3 s,
   `widget.prompt_panel.build_detail_header_summary` up to 10.3 s "warm") that saturate
   the disk and stretch the next refresh.
5. **UI-thread hitches on the Agents tab.** The unread-ack path performs a synchronous
   Rust notification-store mutation on the UI thread (2.3 s observed); the info-panel
   countdown tick rebuilds the whole panel text every second (2.1 s hitch observed under
   load).

Related active beads (do not duplicate; reference where relevant): sase-kh (hidden
agent-artifact index rows are never pruned — archive bloat), sase-v3 (idle Agents-tab
RSS), sase-wr (retire `ace_refresh_tokens` sunset flag — keep all changes compatible
with tokens-on, never build on the off branch), sase-vi (filesystem workflow loaders
miss sharded ace-run timestamps), sase-zc (shell-handoff and plan-propose pulse writers
compute the wrong pulse path; the pulse-delta phase fixes it).

Coordinate with the in-progress epic sase-zr.7 (sase-zr close-out; plan
`plan:202609/sase_zr_close_out.md`), which also changes Agents-tab refresh:

- **sase-zr.7.3 (ace-fast-refresh)** makes the gate-acceptance pulse
  (`notification_gates/decision.py::_touch_gate_shell_refresh_pulse`) target the exact
  agent directory, and routes gate receipts to exact shell, planner and family row
  deltas. It also protects newer state from older in-flight snapshots. The two phases
  complement each other: pulse-delta stops _project-level_ pulses from disabling a
  queued exact delta, while 7.3 makes the gate pulse itself exact. The pulse-delta rule
  must therefore stay limited to project-level pulses (see that phase). Do not change
  the gate-acceptance writer or its row routing here; 7.3 owns them. 7.3 was told to
  touch the sase-zc call sites only if it adds a shared pulse-path helper, and this epic
  adds no such helper.
- **sase-zr.7.2** changes when gate settlement writes its pulse
  (`gate_shell/settlement.py`). Expect a file overlap only; do not reorder that pulse
  here.
- **sase-zr.7.1 / sase-zr.7.1.1** add a durable failure-outcome record. If it lives in
  an agent's artifact directory, the inflight-status poll may include it (see that
  phase).
- **sase-zr.7.5** records targeted latency evidence on overlapping paths (acceptance
  receipt → row paint, settlement → pulse visibility). See the verify phase for
  attribution.

Both epics keep the five-second full-load floor (`AGENTS_LOAD_MIN_INTERVAL_SECONDS`) and
the idle cadence.

No new feature flags: every change here restores intended behavior or applies an
established pattern; none is a beta surface or a deprecation (per
`sase/memory/sase_flags.md`).

For any phase below: run `just check` before finishing (two-speed verification —
`just check-full` belongs to landing, not phase workers). Phase workers must not create
beads; record discovered work as `PROPOSED FOLLOW-UP:` notes on their own phase bead.
Read `sase memory read tui_perf.md` and, if touching loaders/refresh scheduling, keep to
the existing fast paths (`_refilter_agents`, `_schedule_agents_async_refresh`,
`patch_row`) instead of adding new refresh code paths.

## Phase: pulse-delta — stop pulse poisoning

**Goal:** a project-level `.ace_refresh_pulse` write must never disable the bounded
delta path, and pulses must land where both the watcher and the surface-token probe see
them.

1. In `_enqueue_agent_artifact_delta_paths`
   (`src/sase/ace/tui/actions/event_refresh/_artifact_delta.py`), classify
   **project-level** pulse paths as a pure freshness kick: set `_dirty_agents = True`,
   but do **not** set `_dirty_agent_artifact_fallback_reason`, and do not discard queued
   exact dirs. Project-level pulses are `<project>/artifacts/.ace_refresh_pulse`
   (artifact-relative parts `(".ace_refresh_pulse",)`) plus the misplaced month-level
   pulses that the sase-zc bug has already written
   (`<project>/artifacts/ace-run/<YYYYMM>/.ace_refresh_pulse`). A batch containing only
   such pulses leaves `agent_delta_ready` False (no dirs) but keeps the surface dirty,
   so the normal gates handle it. A batch with pulses plus exact marker paths keeps its
   delta consumable.
   - Do **not** apply this rule to pulses at any other depth. sase-zr.7.3 plans a gate
     pulse inside the exact agent directory, and any pulse or marker that maps to an
     exact agent artifact dir must still produce an exact refresh for that dir. Leave
     the handling of per-agent-directory pulses to sase-zr.7.3. If 7.3 has already
     landed, keep its mapping intact.
2. Fix sase-zc: the two writers that compute the pulse location with
   `Path(artifacts_dir).parents[1]` (`src/sase/shells/handoff.py:90`,
   `src/sase/main/plan_propose_handler.py:239`) land at the month dir for the sharded
   `artifacts/ace-run/<YYYYMM>/<DD>/<ts>` layout. Do **not** add a new pulse-path
   helper. Replace both inline writes with
   `touch_shell_refresh_pulse(project_name_from_artifacts_dir(artifacts_dir))` from
   `src/sase/shells/settlement.py`, exactly as `src/sase/axe/runner_artifacts.py:295`
   and `src/sase/axe/run_agent_exec_markers.py:51` already do. That writes
   `<project>/artifacts/.ace_refresh_pulse` for both flat and sharded layouts. Delete
   `_touch_agent_artifacts_refresh_pulse` in `handoff.py` once it is unused, and keep
   both writes best-effort (a pulse failure must never fail the handoff). Leave
   `epic_launch_handoff_monitor.py` alone: it already targets the right path, and epic
   sase-117.5 owns that handoff code. Add a `PROPOSED FOLLOW-UP:` note on the phase bead
   saying this resolves sase-zc, so it can be closed when the epic lands.
3. Tests:
   - (a) A watcher batch of project-level pulse paths (including a legacy month-level
     pulse) keeps `fallback_reason` None and sets `_dirty_agents`.
   - (b) Pulses mixed with exact marker paths still consume the delta (the
     `_consume_agent_artifact_delta_refresh` path).
   - (c) The shell-handoff and plan-propose writers put the pulse at
     `<project>/artifacts/.ace_refresh_pulse` for both flat and sharded artifacts dirs.
   - (d) A pulse inside an exact agent artifact dir is not swallowed by the
     project-level rule.
   - (e) An integration-style test: a settlement-shaped event sequence reaches a row
     update without a broad load. Assert this through `record_agents_refresh_trace`
     stages or the `_dirty_agent_artifact_fallback_reason` state.

## Phase: capacity-fresh — cheap capacity refresh

**Goal:** the load/capacity indicator reflects reality on tab entry and when
limits/holds change, without waiting for a broad load.

1. Recompute the runner-capacity snapshot from the in-memory roster on the tab-switch
   fast path: after `_refilter_agents()` repaints from cache, schedule an off-thread
   recompute using the existing `refresh_runner_slot_context`
   (`src/sase/ace/tui/models/agent_runner_slots.py:98`) over
   `self._agents_with_children`, current `get_max_running_agents()`, and
   `active_agent_hold_records()`, then apply the snapshot + re-render the info panel on
   the UI thread. Do not add a new refresh code path: hang this off the existing
   tab-switch hook in `_app_watchers.py`, and coalesce with any in-flight agents load.
   - "Last write wins" is **not** acceptable. An agents load computes
     `boundary.runner_capacity` off-thread from inputs it read when it started
     (`_loading_apply.py:459` only stores the result). A load that started before a
     limit or hold change can finish after the tab-entry recompute and overwrite it with
     stale numbers.
   - Guard the snapshot with a capacity-input generation. Bump it on each tab-entry
     recompute and on limit/hold token drift. Capture it when a load or recompute
     starts, and apply only if nothing newer has been applied. If a load finishes with
     an older generation, keep its fresher roster but recompute capacity from that
     roster with the current limit and holds, instead of storing its snapshot. This is
     the same "protect newer state from older in-flight snapshots" rule that sase-zr.7.3
     applies to rows. If 7.3 has landed a reusable guard, reuse it.
2. Make external limit/hold changes dirty the agents surface: add the runner limit
   override file and the agent-holds store path to the agents surface token in
   `src/sase/ace/tui/actions/event_refresh/_surface_tokens.py` (stat-only, respects
   tui_perf rule 8 — no new render-path stats; the probe already runs off-thread per
   tick). Locate both paths via their owning modules (`sase/config/core.py` for the
   override; the holds facade for the store path) instead of hardcoding.
3. Fix the `_agent_info_metrics` memo key
   (`actions/agents/_display_detail_info.py:41-78`) so in-place status mutations cannot
   serve stale running/queued counts — e.g. include a roster status revision counter
   bumped by the paths that mutate `agent.status` in place, or key on a cheap tuple of
   (identity, status) hashes. Keep it O(#rows) worst case, computed only when the panel
   updates.
4. Tests:
   - The capacity strip updates on tab entry without any agents load.
   - An agents load that started before a limit change and applies after the tab-entry
     recompute does not restore the old limit.
   - A runner-limit override change drifts the agents token.
   - An in-place status flip changes the running count on the next panel update.

## Phase: tick-diet — attention RPC off the critical path

**Goal:** an auto-refresh tick never waits on the network before refreshing local
surfaces (tui_perf rule 10: pollers revalidate on ticks, recompute on a longer cadence).

1. In `_run_auto_refresh_surfaces`
   (`src/sase/ace/tui/actions/event_refresh/_auto_refresh.py:295-301`), change the
   per-tick `_poll_fleet_attention_inventory` call to `cache_only=True` semantics
   (thread the flag through `actions/agents/_remote_attention.py:57` into
   `fetch_remote_attention_inventory`), and move the full network recompute to a
   separate, much longer cadence (default 60 s; reuse the existing pending/running
   coalescing flags in `_poll_fleet_attention_inventory`). Event-driven full polls
   (after a remote attention answer, `_on_remote_attention_complete`) stay as-is. Verify
   what the federation worker's cache returns for `cache_only=True` when cold
   (`src/sase/dispatch/federation/_facade.py`, `_supervisor.py`) — if the worker has no
   read cache, add the recompute cadence in the TUI layer instead (skip the network call
   unless the interval elapsed) and keep `cache_only` untouched.
2. Do not let the attention poll block the agents surface even on recompute ticks: run
   it concurrently with the axe/notifications/agents work (e.g. start it as a task at
   tick entry and await it after the agents block, or fully decouple it from
   `_run_auto_refresh_surfaces` into its own pump-free task with its own coalescing
   guards — preserve the guards per tui_perf rule 2).
3. Add tick observability: record the attention poll duration and mode (cache/recompute)
   as counters on the `refresh.auto_tick` trace span so the next regression is visible
   in `~/.sase/perf/tui_trace.jsonl`.
4. Tests: a tick with a slow (mocked) federation fetch still completes the agents
   refresh promptly; recompute happens on the long cadence; a remote-attention change
   still sets `_dirty_notifications` and converges the inbox.

## Phase: broad-load-diet — cheaper broad loads and warmups

**Goal:** on an archive this size, a broad Tier 1 `agents.load_from_disk` lands in well
under 1 s p95 when the index is warm, and post-apply warmups stop saturating the host.
This phase is research-first (`size: large`): profile before changing
(`sase tui --profile`, `SASE_TUI_TRACE=1`, and the recipes in `docs/perf_runbook.md`;
benches in `tests/perf/`).

Known concrete leads, in priority order:

1. The unwindowed per-broad-load sweeps in
   `src/sase/ace/tui/models/agent_loader.py:268-350` (`get_all_project_files()`,
   `load_agents_from_running_field`, the per-patch HOOKS/MENTORS/COMMENTS loops) and
   `relations.index.patches` (p50 420 ms per load): make them mtime-keyed caches so an
   unchanged project file costs ~0 per load (tui_perf rule 8; reuse existing cache
   utilities, clear via the established `clear_config_cache`-style hooks in tests).
2. `agents.bead_confirmation_warmup`
   (`src/sase/ace/tui/actions/agents/_loading_bead_warmup.py`, up to 10.8 s for ≤11
   candidates): find out where the time goes (per-candidate subprocess? bead store
   scans?), then cache by input mtime/signature, coalesce across consecutive applies,
   and skip candidates whose inputs are unchanged.
3. `agents.monitor_reconcile` (max 4.8 s) and `agents.live_hint_refresh` (max 2.97 s):
   same treatment — skip-unchanged, coalesce, and ensure they never run concurrently
   with an in-flight agents load they would contend with.
4. `widget.prompt_panel.build_detail_header_summary` at 10.3 s with `cache_state=warm`
   on the Agents tab: profile what "warm" still recomputes; ensure the debouncer
   (tui_perf rule 7) actually prevents repeated rebuilds under refresh churn.
5. Tier 1 `index_freshness=revalidate` loads (9–9.8 s) fire from
   `_maybe_trigger_tier1_index_revalidate_reconcile` at most every 300 s, and Tier 2
   reconciles (9.8–10.2 s) after 30 s input-quiet
   (`actions/agents/_loading_refresh_polling.py`). Evaluate making revalidate
   incremental on large archives (e.g. restrict source discovery to live month/day
   shards via `agent_artifact_shards.py` helpers, with the full sweep only on the sanity
   cadence). Coordinate with bead sase-kh (index hidden-row bloat) — do not implement
   pruning here, but note findings on the phase bead.

Acceptance: with the index warm and no roster change, a broad Tier 1 load's `disk` stage
on an 11k-dir archive measurably drops (target ≥50% reduction on the athena-shaped bench
fixture), and repeated applies with unchanged inputs skip the warmups entirely.
Add/extend a bench under `tests/perf/` mirroring the `agent_load_tiering` fixture so the
win is regression-guarded (respect perf-floor flake handling; see bead sase-u8 for the
existing floor's flake history).

## Phase: ui-hitches — keep the pump free

**Goal:** no multi-second main-thread stalls on the Agents tab's hot paths.

1. Unread ack: `_clear_agent_unread_and_dismiss_notifications`
   (`src/sase/ace/tui/actions/agents/_unread_state.py:387`) calls
   `dismiss_agent_completion_notifications_matching_agents`
   (`src/sase/notifications/store.py:461`) synchronously on the UI thread from the
   unread-navigation keypress path (`_unread_navigation.py`). Convert to the established
   shape (tui_perf rules 1 and 3): optimistic UI mutation (clear the unread marker
   immediately via `patch_row`), then persist the store mutation off-thread; marshal
   completion back with `call_from_thread`, and reconcile on failure (restore the
   marker + toast). Audit the same file for other synchronous store mutations on key
   paths.
2. Info panel countdown: `AgentInfoPanel.update_countdown_only`
   (`src/sase/ace/tui/widgets/agent_info_panel.py:334`) rebuilds the full panel text
   every second. Make the countdown update patch only the countdown segment (cache the
   rest of the rendered text/spans and splice), or skip the rebuild entirely when
   nothing but the clock digit changed.
3. Verify with the stall watchdog: lower `SASE_TUI_STALL_*` thresholds in a short run
   per `sase/memory/tui_perf.md`; the unread-ack path must not appear in
   `~/.sase/logs/tui_stalls.jsonl` at a 250 ms threshold.
4. Tests: unread ack updates the row immediately with the store write pending;
   store-write failure restores unread state; countdown tick does not call the full text
   builder (assert via a counter/spy).

## Phase: inflight-status — bounded convergence backstop

**Goal:** RUNNING→DONE/WAITING/FAILED transitions appear on rows within ~2 s even when
inotify misses events, without broad loads — the same pattern
`_poll_starting_agent_transitions`
(`src/sase/ace/tui/actions/agents/_loading_refresh_polling.py:183`) already uses for
STARTING rows.

1. Extend the 1 s bounded poll to cover all in-flight rows currently in the roster
   (bounded by the active count, typically ≤ tens): stat only the relevant marker files
   (`done.json`, `waiting.json`, `retry_state.json`, `pending_question.json`) in each
   in-flight agent's artifacts dir, off the UI thread, and on observed change schedule
   an exact artifact-delta refresh for just those dirs (the path built in the
   pulse-delta phase guarantees the delta stays consumable). Respect the navigation gate
   and skip while a load is in flight. If sase-zr.7.1.1 / sase-zr.7.2 have landed a
   durable failure-outcome marker inside the agent's artifacts dir, add it to the polled
   set. Feed the same exact-delta queue that sase-zr.7.3's receipt routing uses; do not
   create a parallel one.
2. Keep it strictly stat-only and coalesced: no JSON parsing on the poll path, one
   pending-flags guard, and a cap (reuse `MAX_LIVE_AGENT_WATCHES`-style bounding — if
   in-flight count exceeds the cap, poll the newest N and rely on the watcher for the
   rest).
3. Tests: a marker write to an in-flight dir with the watcher disabled lands a row
   status update within two poll intervals via the delta path; the poll does zero work
   when nothing is in flight; per-tick stat count is bounded.

## Phase: verify — prove it on athena

**Goal:** demonstrate the end-to-end win on the machine that hurts, and pin it with
regression coverage.

1. Capture before/after using the recipes in `docs/perf_runbook.md` (`SASE_TUI_TRACE=1`,
   `SASE_TUI_PERF=1`, stall watchdog at verification thresholds) during a realistic busy
   window (several running agents) on athena. The "before" baseline is the 2026-09-15/17
   data cited in the Evidence section; re-capture a fresh before if the tree has
   drifted.
   - Record the commit SHA (and which sase-zr.7 phases had landed) for every before and
     after capture. sase-zr.7.5 records latency on overlapping refresh paths (acceptance
     receipt → row paint, settlement → pulse visibility), so neither epic may credit
     itself with the other's improvement.
   - Where a gain may come from sase-zr.7.3 rather than this epic, say so in the bead
     note.
2. Acceptance targets, measured over ≥30 min busy-host sessions:
   - `refresh.auto_tick` p95 < 1000 ms (was 4057 ms) and no tick > 2 s attributable to
     the attention poll;
   - marker-write→visible-row-update latency < 2× the 10 s tick interval in the worst
     case, and < 3 s when the watcher delivers the event (assert via a scripted
     settlement while the TUI runs);
   - zero `tui_agent_load_slow` `load_kind=full` events during a 10 min idle-host
     window; capacity indicator correct within 1 s of tab entry;
   - no main-thread stall > 500 ms from the unread-ack or countdown paths.
3. Run the relevant benches (`pytest -s -m slow tests/ace/tui/bench_tui_jk.py`,
   `tests/perf/bench_tui_trace.py`, plus the bench added in broad-load-diet) and record
   p50/p95 tables on the epic bead.
4. Where a target is missed, file the gap as a `PROPOSED FOLLOW-UP:` note with the
   captured numbers rather than stretching this epic.
