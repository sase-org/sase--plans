---
tier: epic
title: Fix Agents-tab flicker and disappearing tribe panels
goal: 'The Agents tab stops flickering under an active filter query and live agent
  churn: unchanged-data refreshes patch instead of full-rebuilding, no-op fleet reprojections
  stop repainting, consecutive load tiers converge on one stable visible agent set,
  and tribe panels such as @epic stay mounted.

  '
phases:
- id: converge-load-tiers
  title: Stop the visible-set oscillation across load tiers
  depends_on: []
  size: large
  description: 'converge-load-tiers: reproduce the bounded-vs-revalidate agent-set
    swing with a failing test, fix the incomplete-load merge and complete-history
    latch so consecutive loads converge, and assert panel-key stability across a bounded
    apply.'
- id: incremental-under-search
  title: Incremental panel refresh with an active filter query
  depends_on: []
  size: medium
  description: 'incremental-under-search: allow the incremental display diff path
    when the search query text is unchanged, keep a distinct full-rebuild fallback
    for query changes, and cover both with tests.'
- id: skip-noop-fleet-repaint
  title: Skip no-op fleet reprojection repaints
  depends_on: []
  size: small
  description: 'skip-noop-fleet-repaint: signature-compare fleet projection inputs
    and skip finalize/repaint when unchanged, preserving forced sources and genuine
    changes, with tests.'
- id: verify-on-athena
  title: Regression coverage and on-host verification
  depends_on:
  - converge-load-tiers
  - incremental-under-search
  - skip-noop-fleet-repaint
  size: medium
  description: 'verify-on-athena: soak the fixed TUI on athena with trace capture,
    compare against the recorded pre-fix baselines, add a guard test against silent
    full-rebuild regressions, and record CPU before/after.'
proposed_by: bbugyi200.athena.0ml
create_time: 2026-09-17 16:26:19
status: wip
bead_id: sase-127
---

- **PROMPT:** [prompts/202609/agents_tab_flicker.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_tab_flicker.md)
- **BEAD:** [sase-127](https://github.com/sase-org/sase--beads/blob/main/pages/sase-127/README.md)

# Fix Agents-Tab Flicker And Disappearing Tribe Panels

## Problem

The Agents tab visibly flickers during normal operation, and tribe panels (for example
`@epic`) disappear and re-appear every few minutes. Live diagnosis on athena (traces in
`~/.sase/perf/tui_trace.jsonl`, PID-tagged slow-load records in
`~/.sase/logs/tui_agent_loads.jsonl`, and a `py-spy` check of the running `sase tui`
process) established three compounding root causes:

1. **Active-search display fallback forces full panel rebuilds.**
   `_try_refresh_agents_display_incremental`
   (`src/sase/ace/tui/actions/agents/_display.py`, the `_agent_search_query` guard)
   refuses the incremental/selective panel path whenever an agent filter query is active
   and falls back to `_refresh_panel_widgets`, which calls `update_list` on every
   rendered `AgentList` panel. Trace evidence: 480 of 486 recent
   `agents.refresh_panel_widgets` full rebuilds carry `display_fallback: active_search`.
   A user who keeps a persistent filter query active therefore gets a full
   teardown-and-repaint of all panels on every refresh cycle.

2. **Every agents apply triggers a second, unconditional repaint.**
   `_apply_loaded_agents_prepared_inner`
   (`src/sase/ace/tui/actions/agents/_loading_apply.py`) ends with
   `_schedule_agents_fleet_refresh(source="apply")`. The fleet refresh
   (`src/sase/ace/tui/actions/agents/_fleet_refresh.py`) always ends in
   `_reproject_agents_from_current_mode(source="fleet_refresh")`
   (`src/sase/ace/tui/actions/agents/_fleet_projection.py`), which calls
   `_finalize_agent_list` and repaints even when the fleet projection and the local
   agent list are byte-for-byte unchanged. Trace evidence: paired
   `agents.refresh_panel_widgets` spans ~0.7-2.5 s apart after every agents apply, the
   second tagged `source: fleet_refresh`. Combined with root cause 1 this doubles the
   flicker.

3. **The visible agent set oscillates between load tiers, so tribe panels flap.**
   PID-tagged `tui_agent_loads.jsonl` records for the long-running TUI show broad
   `auto_refresh` Tier 1 loads applying ~171 agents while `tier1_index_revalidate`,
   `startup_prefix_completion`, and `input_quiet_tier2_reconcile` loads apply ~462-617
   agents, alternating all day (357 slow auto-refresh full loads and 81 revalidate loads
   in one day). Trace corroboration: `agents.incomplete_load_merge` spans shrink the
   merged list (cached=452, incoming=48, applied=436), and every recent
   `agents.apply_loaded_agents_prepared` span reports `complete: false` - including
   revalidate loads - so the complete-history latch never converges and Tier 2
   reconciles keep re-arming. Panel keys are recomputed from the visible agent list on
   every apply (`_sync_panel_group` / `panel_keys_for` in
   `src/sase/ace/tui/models/agent_panels.py`), so when a tribe's query-visible members
   exist only in the larger set, the next bounded load empties that tribe bucket and the
   panel disappears, then the next revalidate/reconcile load restores it.

Context that keeps the cycle hot: the host runs many live agent runners that
continuously write artifacts, so the agents surface token drifts on most auto-refresh
ticks and broad loads run every one to two minutes. The TUI process was observed at ~54%
average CPU.

This work is adjacent to, but distinct from, epic sase-124 ("Agents tab freshness on
large-archive hosts"): sase-124 cut load cost and is in its verification phase; none of
its phases touch the display fallback, the fleet reprojection repaint, or the cross-tier
set convergence. Phases below must rebase on master where sase-124 work has landed and
must not modify sase-124.7's verification artifacts.

All changes are Textual presentation state and TUI loader glue in this repo; per the
Rust core boundary litmus test no `sase-core` change is required.

Relevant tui_perf rules (already read this turn): prefer selective updates over full
rebuilds (rule 6), route refreshes through the existing fast path (rule 5), and measure
with `SASE_TUI_TRACE=1` spans rather than guessing.

## Phases

### Phase converge-load-tiers: Stop the visible-set oscillation across load tiers

A bounded Tier 1 load must never make previously-visible agents (and therefore whole
tribe panels) vanish, and revalidate/Tier 2 results must latch so the system converges
instead of oscillating.

1. Reproduce first. Add a failing test that (a) applies a full/revalidated load with
   agents spanning several tribes while an agent search query is active, then (b)
   applies a bounded Tier 1 load (`bounded_prefix=True`, `has_more=True`) containing
   only a subset, and asserts that the visible agent list and the panel key set are
   unchanged after (b). Model the test on the existing coverage around
   `merge_incomplete_load_after_complete_history`
   (`src/sase/ace/tui/actions/agents/_loading_compute_merge.py`) and the prepared-apply
   pipeline in `src/sase/ace/tui/actions/agents/_loading_apply.py`.
2. Diagnose why convergence fails today. Known leads, in order:
   - Revalidate and reconcile loads apply with `complete: false`
     (`agents.apply_loaded_agents_prepared` trace spans), so
     `_agents_complete_history_query_key` never latches for the active query and
     `_should_arm_full_history_reconcile` keeps re-arming Tier 2 (`_loading_apply.py`,
     `_loading_refresh_polling.py`).
   - The incomplete-load merge shrinks the cached list (observed 452 -> 436): audit the
     dismissed/deleted filtering inside `merge_incomplete_load_after_complete_history`
     and the downstream visibility filtering for rows it wrongly drops.
   - The merge is skipped entirely on some paths (the
     `agents_seen_complete_history`/`is_bounded_partial` guard) even though the cached
     list is the better universe.
3. Fix so that consecutive loads converge: bounded loads patch over the cached universe,
   revalidate/Tier 2 loads latch completeness for their query key, and the
   armed-reconcile counters stop cycling. After the fix, assert in tests that a
   bounded-load apply immediately following a revalidated apply produces an identical
   visible list, identical panel keys, and does not re-arm
   `_agents_history_reconcile_pending`.
4. Guard against regression at panel granularity: extend the repro test to assert
   `panel_keys_for` stability across the bounded apply (the `@epic` flap symptom).

### Phase incremental-under-search: Incremental panel refresh with an active filter query

Remove the blanket `active_search` fallback in `_try_refresh_agents_display_incremental`
(`src/sase/ace/tui/actions/agents/_display.py`) so a _stable_ query no longer forces
full rebuilds:

1. Allow the incremental diff path when the agent search query text is unchanged since
   the previous finalized refresh (track the last-rendered query alongside the existing
   previous-agents snapshot). The diff already operates on the post-filter visible lists
   that panels render, so an unchanged query is safe to patch incrementally.
2. Keep the full-rebuild fallback when the query text changed between refreshes, and
   record it under a distinct fallback reason (for example `search_query_changed`) so
   traces can tell the two cases apart.
3. Tests: with a query active and an unchanged agent list, a finalize pass performs zero
   `update_list` calls (patch path only); with a changed query, the full rebuild runs
   once; the `active_search` fallback reason no longer appears for unchanged-query
   refreshes. Follow the existing incremental-display test patterns around
   `_try_refresh_agents_display_incremental` and `_refresh_affected_panel_widgets`.

### Phase skip-noop-fleet-repaint: Skip no-op fleet reprojection repaints

`_apply_fleet_projection` -> `_reproject_agents_from_current_mode`
(`src/sase/ace/tui/actions/agents/_fleet_projection.py`, `_fleet_refresh.py`) must not
re-finalize and repaint when nothing changed:

1. Compute a cheap signature of the inputs that feed `_agents_source_for_current_mode`
   (fleet rows + dispatch provisionals + local base identity/content revision). When the
   signature matches the previously applied one, update the header/loading indicator
   only and skip `_finalize_agent_list`.
2. Preserve the current behavior for `force=True` sources (`remote_mutation`,
   `remote_attention`) and for genuine projection changes.
3. Tests: an apply-triggered fleet refresh with unchanged rows performs no panel refresh
   (no `agents.refresh_panel_widgets` span / no `update_list` calls); a changed fleet
   row still repaints; forced sources always repaint.

### Phase verify-on-athena: Regression coverage and on-host verification

Prove the symptom is gone with the same instruments that diagnosed it:

1. Run the capture recipes from `docs/perf_runbook.md` with `SASE_TUI_TRACE=1` against a
   TUI session on athena with an active agent filter query and live agent churn. Compare
   against the pre-fix baseline numbers recorded in this plan's Problem section:
   - `agents.refresh_panel_widgets` full rebuilds drop from ~2 per reload to ~0 on
     unchanged-data reloads (no more paired spans).
   - `display_fallback: active_search` disappears from steady-state traces.
   - `agents.apply_loaded_agents_prepared` agent counts stay stable across consecutive
     `auto_refresh` and `tier1_index_revalidate` applies (no more ~171 <-> ~484 swings),
     and armed Tier 2 reconciles stop recycling.
   - Tribe panels (for example `@epic`) remain mounted across at least 30 minutes of
     live refresh cycles.
2. Add or extend a `tests/perf` bench/guard that fails if a finalize pass with an
   unchanged visible list and unchanged query performs a full panel rebuild, so the
   fallback cannot silently return.
3. Record a `PROPOSED FOLLOW-UP:` note on this phase's bead proposing a `tui_perf.md`
   memory addition documenting the active-search fallback gotcha and the load-tier
   convergence invariant (memory edits route through the memory-write procedure, not
   this epic).
4. Watch the TUI's CPU during the soak: steady-state CPU for the `sase tui` process
   should drop well below the ~54% observed pre-fix; record the before/after numbers in
   the phase notes.
