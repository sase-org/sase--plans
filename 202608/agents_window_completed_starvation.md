---
tier: tale
title: Stop the bounded Agents window from starving completed rows at startup
goal:
  The Agents tab's first paint after a TUI restart lists the same recent DONE agents,
  clans, and tribes it lists once the session is warm, instead of only live rows.
size: medium
proposed_by: bbugyi200.athena.0fr
---

- **AGENTS:**
  - [bbugyi200.athena.0fr](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0fr.md)
- **COMMITS:**
  - [45a0a88](https://github.com/sase-org/sase/commit/45a0a8880a4e0c7f55e15ca30959fe8f63b7fde3)
    — fix(ace): complete the Agents window prefix once after first paint

<!-- sase:links:start -->

## Links

| Relation | Artifact                              | Why                                                                              |
| -------- | ------------------------------------- | -------------------------------------------------------------------------------- |
| related  | [plan:202608/agents_viewport_1.md][1] | introduced the windowed index read whose completed-budget math this plan repairs |

[1]: https://github.com/sase-org/sase--plans/blob/main/202608/agents_viewport_1.md

<!-- sase:links:end -->

# Plan: Stop the bounded Agents window from starving completed rows at startup

## Symptom

After restarting `sase ace`, the Agents tab shows only a handful of nodes — the live
agents and whatever DONE rows happen to hang off them. Every other DONE agent and agent
clan that was visible before the restart is missing from both the `@default` and `@epic`
tribes. Minutes later they all reappear at once, with no user action. The regression
started 2026-08-27.

## Root cause (confirmed by reproduction, not inference)

`select_windowed_records` in `sase-core` (`crates/sase_core/src/agent_scan/index.rs`)
budgets the completed tier out of whatever the active tier leaves behind:

```rust
let completed_budget =
    requested_limit.saturating_sub(active_candidates.len() as u32) as usize;
```

Every ACE Tier-1 read is windowed and cached, so `requested_limit` is the
`AgentsViewport` prefix (`start_row + visible_rows + prefetch_rows`, which is at most
`240` at the top of the list and only grows as the user scrolls). When the index holds
more visible _active_ candidate rows than that limit, the subtraction saturates to `0`
and **no completed row is selected at all**. The window then returns active records
only, and the Agents tab has no DONE history to render.

That is exactly the state of a working home dir. Measured on this machine's
`~/.sase/agent_artifact_index.sqlite` (10,426 rows):

| load                                             | records | agents | median |
| ------------------------------------------------ | ------- | ------ | ------ |
| cached + window 126 (what ACE does today)        | 588     | **14** | 208 ms |
| cached + window 716 (all active + 126 completed) | 716     | 258    | 313 ms |
| cached, unwindowed (`requested_limit=None`)      | 788     | 326    | 496 ms |

`AgentArtifactIndexWindowWire` for the 126 window reports `active_candidate_count=590`,
`completed_candidate_count=2725`, `selected_candidate_count=590` — 590 active candidates
against a 126 budget, so all 2,725 completed candidates are discarded.

Reproduce with the installed interpreter (read-only, cached freshness never writes):

```python
from sase.ace.tui.models.agent_loader import load_tiered_agents
for limit in (126, None):
    agents, state = load_tiered_agents(
        patch_snapshot=[], full_history=False, use_artifact_index=True,
        index_freshness="cached", search_query=None, requested_limit=limit,
    )
    print(limit, len(agents), state.record_count, state.has_more)
```

The active tier is that large because the index calls a row active whenever it has no
done marker (`has_done_marker = 0 OR workflow_status NOT IN (...)`), and agents that die
without writing one stay there forever: 3,180 such rows exist, 590 of them not
dismissed. Python drops them later — `_filter_dead_pids` in
`src/sase/ace/tui/models/_agent_loader_normalization.py` discards any non-terminal row
whose PID is gone — which is why 588 records normalize down to 14 agents. So _records
are not rows_: budgeting the completed tier against the active record count is budgeting
against a number that has almost no relationship to what the user sees.

### Why the rows come back on their own

Nothing recovers the missing history except an **unwindowed** read.
`_arm_tier1_index_revalidate_reconcile`
(`src/sase/ace/tui/actions/agents/_loading_refresh_polling.py`) arms one after each
cached Tier-1 load and fires it 2 s after input goes quiet, at most every 300 s;
`should_use_windowed_candidate_query` refuses to window a `Revalidate` read, so that
load returns the full Tier-1 set and the agents appear. On 2026-08-28 that recovery load
took **137 s** of disk time (`~/.sase/logs/tui_agent_loads.jsonl`,
`source=tier1_index_revalidate`, `agents=316`), because `repair_stale_rows_for_query`
re-stats every `hidden = 1` row — 4,507 of them — on a revalidating read. Hence "they
reappeared while I was typing".

Once a warm session has the full list, the damage is hidden:
`merge_incomplete_load_after_complete_history`
(`src/sase/ace/tui/actions/agents/_loading_compute_merge.py`) patches a bounded partial
over the cached list instead of replacing it. At startup there is no cached list to
patch over, so the starved window _is_ the entire user-visible universe — which is why
this only looks broken right after a restart.

Viewport expansion is starved the same way: `_maybe_schedule_agents_viewport_expansion`
grows `requested_limit` with `current_idx`, so on this index the user would have to
scroll past row ~590 before a single completed row entered the window.

## Non-goals

Two real defects found while diagnosing this are **out of scope**; file them as task
beads with `/sase_new_task` rather than widening this plan:

1. The index's active tier grows without bound (3,180 rows whose runner is long dead).
   Terminalizing rows that still carry `workflow_state.json` or a waiting marker needs
   its own liveness design; getting it wrong hides a live agent.
2. A revalidating Tier-1 read re-stats every hidden row (`repair_stale_rows_for_query`),
   which is what turned a 3 s recovery into 137 s.

Neither is required to fix the reported symptom, and this plan must not depend on either
landing. Do **not** "fix" this by widening `AgentsViewport`, by making ACE's normal
refresh unwindowed, or by resurrecting the reverted daemon provider.

## Part 1 — `sase-core`: give the completed tier its own budget

Open the core repo with `/sase_repo` (`sase repo open sase-core -r "<why>"`) and use
only the path it prints.

1. In `crates/sase_core/src/agent_scan/index.rs`, `select_windowed_records`: make the
   completed budget independent of the active count — `requested_limit` completed
   candidates, newest-first, regardless of how many active candidates were selected.
   Keep the existing invariant that **every** matching active candidate is returned
   (`plan:202608/agents_viewport_1.md`, correctness invariant 2); the defect is the
   phrase "fill the _remaining_ budget", not the guarantee above it.
2. Keep `has_more`/`truncated` honest. With active rows never truncated, both now mean
   "completed candidates were truncated". Update the doc comment so the next reader
   knows the window bounds the completed tier and the `active_limit` bounds the active
   tier.
3. `windowed_query_preserves_active_rows_and_fills_with_completed` (same file) encodes
   the old math — with `limit=4`, 3 active and 2 completed rows, it asserts the older
   completed row is dropped. Update it to the new contract and rename it if the name no
   longer describes the behavior.
4. Add a regression test that fails on today's code: more active candidates than the
   requested limit (e.g. `limit=2` with 5 active and 3 completed rows) must still return
   all 5 active records **and** the 2 newest completed records, with
   `completed_candidate_count=3` and `has_more=true`.
5. Do not change the wire types, the unwindowed `select_records` path, or revalidation
   behavior. Cached reads stay read-only.
6. Run the core's own checks; then, back in the sase repo, move the pin forward with
   `just ratchet-core-revision` once the core change is on sase-core's remote HEAD.
   `sase-core-revision.txt` is what CI builds from — the Python-side fix is unverifiable
   until it moves.

## Part 2 — `sase`: complete the prefix once after first paint

Part 1 makes the first paint a true prefix (~126 rows here). It does not restore the
pre-regression startup behavior, where the whole Tier-1 set was present immediately:
until the prefix is completed, the header counts, tribe summaries, and clan grouping are
all computed over a partial set, and the only thing that completes it is the ≥300 s
revalidate cadence. A cached _unwindowed_ read costs 496 ms off-thread (measured above)
— the expensive part of the revalidate path is the stale-row repair, not the absence of
a window — so take it once per session, after first paint.

1. Add one-shot state next to the existing reconcile flags in
   `src/sase/ace/tui/actions/agents/_loading_state.py` (initialized in
   `src/sase/ace/tui/actions/_state_init_agents.py`), e.g.
   `_agents_prefix_completion_pending` / `_agents_prefix_completion_done`.
   `_agents_seen_complete_history` is not a substitute: it is only set by a Tier-2 load,
   and this completion read is Tier 1.
2. Arm it in `_apply_loaded_agents`
   (`src/sase/ace/tui/actions/agents/_loading_apply.py`, beside the existing
   `_arm_tier1_index_revalidate_reconcile` call) when the applied `load_state` has
   `bounded_prefix and has_more` and the session has not completed its prefix yet.
3. Trigger it from the same countdown-tick path as the other deferred reconciles in
   `src/sase/ace/tui/actions/agents/_loading_refresh_polling.py`, gated on a short input
   quiet window so it never lands inside first paint or a `j`/`k` burst. Respect the
   existing `_agents_loading` / `_agents_refresh_scheduled` /
   `_agents_artifact_delta_scheduled` guards, and clear the pending flag before
   scheduling exactly as the revalidate trigger does.
4. Thread it through `_schedule_agents_async_refresh`
   (`src/sase/ace/tui/actions/agents/_loading_refresh.py`) the way `revalidate_index` is
   threaded, including the coalesced `_agents_refresh_pending_*` mirror, and have
   `_agents_viewport_for_load` (`src/sase/ace/tui/actions/agents/_loading_disk.py`)
   yield `None` for that one refresh so `DirectAgentsDataProvider` sends
   `requested_limit=None`. Freshness stays `cached`: this must not become a second
   revalidate.
5. Mark the completion done when a load with `bounded_prefix=False` applies, so a
   session takes this read at most once and every later refresh stays windowed.
6. Source label: `startup_prefix_completion` (free-form; `normalize_refresh_source` has
   no whitelist). It must show up in `record_agents_refresh_trace` and therefore in
   `~/.sase/logs/tui_agent_loads.jsonl`, so the next diagnosis can see it.

Honour `sase/memory/tui_perf.md` throughout: the read runs on the existing pump-free,
last-request-wins path (rules 2 and 5), first paint still never waits on it (rule 9),
and the navigation gate still defers it (rule 13). Add no new refresh code path.

## Part 3 — Tests and verification

1. Python unit tests in `tests/ace/tui/test_lazy_tier2_reconcile.py`, following its
   `_FakeRefreshApp` pattern: the completion arms on a bounded partial apply; it does
   **not** arm when `has_more` is false or when it already ran this session; the trigger
   respects input quiet and the in-flight guards; the scheduled refresh reaches
   `_run_agents_async_refresh` with an unwindowed viewport and cached freshness.
2. A loader test in `tests/test_agent_loader_query_window.py` (or a sibling) proving the
   contract that broke: with more active records than the requested limit, a bounded
   load still returns the newest completed rows. Fixture-driven — do not read the real
   `~/.sase` index from a test.
3. `just check` in the sase repo. Run `just install` first: this workspace's venv may be
   stale, and the pin move requires a rebuilt `sase_core_rs`.
4. `just check-full` before landing, through `/sase_monitor` only, never inline
   (`sase/memory/lint_and_test.md`).

## Acceptance criteria

1. On this machine's real index, a cached load with the default viewport limit returns
   the newest completed rows: `state.record_count` includes a completed contribution and
   the normalized agent count is a full screen (≈126), not ≈14. Record before/after
   numbers from the reproduction snippet above.
2. Restarting `sase ace` shows the recent DONE agents and clans in `@default` and
   `@epic` on first paint — no 2-minute wait, no dependence on `tier1_index_revalidate`.
3. Warm-load median for the windowed read stays well under the unwindowed 496 ms; the
   expected cost on this index is ≈313 ms. If it regresses past that, report the number
   rather than silently widening the window.
4. `just check-full` is green and `sase-core-revision.txt` points at the commit
   containing the Part 1 fix.
