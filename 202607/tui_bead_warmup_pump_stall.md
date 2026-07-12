---
create_time: 2026-07-12 17:00:43
status: done
prompt: 202607/prompts/tui_bead_warmup_pump_stall.md
tier: tale
---
# Fix ACE TUI Freezes: Bead-Confirmation Warmup Blocks the App Message Pump

## Problem

The ACE TUI intermittently freezes for 10–40 seconds: keys are ignored, the cursor doesn't move, and the app appears
hung until it suddenly recovers. `~/.sase/logs/tui_stalls.jsonl` recorded **nine `tui_pump_stall` events on 2026-07-12
alone** (durations 10.5s, 13.5s, 14.5s, 15.0s, 18.0s, 24.5s, 24.6s, 35.0s, 39.5s), most with `last_action: launch`.

## Root Cause (from stall-log forensics)

Two layers combine to produce the freeze:

### Layer 1 — the warmup callback holds the serial app message pump

Every one of the nine stall records captured the same `Task-1` (app message pump) stack:

```
textual/message_pump.py: _process_messages_loop → _dispatch_message → on_event → on_callback → invoke
  → src/sase/ace/tui/actions/agents/_loading_bead_warmup.py:95 in _run_bead_confirmation_warmup
      results = await asyncio.to_thread(warm_confirmed_bead_displays, candidates)
```

`AgentBeadWarmupMixin._schedule_bead_confirmation_warmup()` queues the warmup with
`self.call_later(self._run_bead_confirmation_warmup)`. Textual delivers `call_later` callbacks as `Callback` messages on
the **app's serial message pump**, and `_dispatch_message` awaits async callbacks to completion. So although
`asyncio.to_thread()` keeps the _event loop_ alive (the loop-stall watchdog stayed silent; only the pump watchdog
fired), the pump itself cannot dispatch any further messages — including every key event — until the threaded bead-store
read returns. "Off the event loop" is not the same as "off the pump".

### Layer 2 — the threaded bead lookup can take tens of seconds

`warm_confirmed_bead_displays()` (`src/sase/ace/tui/models/agent_bead.py`) resolves each candidate through
`format_agent_bead_display_for_name(..., require_existing=True)` → `_lookup_bead_issue()`
(`src/sase/agent/bead_display.py`), a five-stage fallback cascade. The expensive stages:

- **`get_read_view()` → `get_project()`** (`src/sase/bead/cli_common.py`) can call
  `resolve_beads_location(materialize=True)` → `materialize_sdd_store()` → `ensure_workspace_sdd_clone()` →
  **synchronous `git pull --rebase` / `git clone` over the network** with a multi-second network timeout. `tui.log`
  confirms the TUI process runs these ("Failed to pull workspace SDD clone …/sase--plans" warnings from
  `sase.sdd._store_link`), and `tui_git_ops.jsonl` shows 4–17s SDD git operations. Materialization also takes
  `materialization_lock(primary)`, so a warmup racing a launch fan-out (which materializes the same SDD stores —
  matching `last_action: launch` on the stall records) can block on the lock for the duration of another process's
  clone.
- **Every `BeadProject()` construction runs `rebuild_from_jsonl()` plus SQLite `init_db()`**
  (`src/sase/bead/project.py`), and `_lookup_bead_issue_in_dirs()` constructs a fresh `BeadProject` per candidate per
  directory — no reuse within a warmup batch. Concurrent launched agents hammer the same `beads.db`, adding SQLite lock
  waits.
- The bead display cache TTL is a flat 60s for hits _and_ misses (`_CACHE_TTL_SECONDS` in
  `src/sase/ace/tui/models/agent_bead.py`), so agents whose bead id resolves to nothing re-run the full worst-case
  cascade every minute — which is why the freeze recurs.

## Fix

### Phase 1 — run the warmup off the message pump (eliminates the freeze)

- In `_schedule_bead_confirmation_warmup()` (`src/sase/ace/tui/actions/agents/_loading_bead_warmup.py`), stop queueing
  the async warmup via `call_later`. Spawn it as a detached asyncio task instead, mirroring the established
  `_spawn_agents_refresh_task()` pattern in `src/sase/ace/tui/actions/agents/_loading_refresh.py:170` ("Run a refresh
  outside Textual's serial app message pump"): `loop.create_task(...)` with a named task, a tracking set, and a
  done-callback that logs failures. The navigation-gate re-arm path (`set_timer` retry) must route through the same
  spawn helper so no code path re-enters the pump with the awaited coroutine.
- Keep the existing coalescing semantics (`_scan_scheduled` / `_scan_running` / `_scan_pending`) and the post-await
  apply leg exactly as they are: `_apply_bead_warmup_results()` already re-matches results against the _current_ agent
  objects by identity, which satisfies the re-capture-after-await rule, and runs on the loop thread just like the
  agents-refresh apply leg.

### Phase 2 — make the TUI bead read path local-only and batched (eliminates the hidden git/IO cost)

Even off the pump, a warmup thread that runs network git syncs every 60s wastes IO, fights launches for the
materialization lock, and keeps the coalescer permanently busy. The TUI's glyph-confirmation read should never mutate or
sync stores:

- Thread a `local_only` (name TBD) flag through `format_agent_bead_display_for_name()` → `_lookup_bead_issue()` in
  `src/sase/agent/bead_display.py`, passed by the TUI's `resolve_bead_display()` only, defaulting to current behavior
  for all other callers (completion notifications, CLI). Under `local_only`, replace the `get_read_view()` stage with a
  non-materializing resolution (`resolve_beads_location(require_existing=True)`; skip the stage instead of materializing
  when the resolved location is unusable) so the TUI path can never trigger
  `git clone`/`git pull`/`materialization_lock`.
- Reuse opened stores within one warmup batch: `warm_confirmed_bead_displays()` should open each distinct beads
  directory at most once per run (one `BeadProject` → many `show()` calls) instead of per-candidate-per-directory,
  avoiding repeated `rebuild_from_jsonl()`/`init_db()` on the same store.
- Add negative-result backoff in `_BeadDisplayCache`: keep the 60s TTL for confirmed hits, but give misses a longer TTL
  (~5 minutes) so names that merely look like bead ids stop re-running the cascade every minute. (Trade-off: a newly
  created bead's glyph may take up to the miss TTL to appear; acceptable for a passive metadata glyph, and a manual
  refresh already forces a reload.)

### Phase 3 — capture worker-thread stacks in pump-stall records (diagnosability)

The current record captures the loop thread (which just shows the watchdog's own capture frame when the loop is alive)
and asyncio task stacks — but not executor threads, which is where the blocking work actually sat. Extend
`_pump_stall_record()` in `src/sase/ace/tui/util/stall_watchdog.py` to dump all non-loop, non-watchdog thread stacks via
`sys._current_frames()` (bounded count and depth, same never-raise discipline). The next stall of this family will then
name the blocking frame directly instead of requiring task-stack inference.

## Out of scope (follow-up candidates)

The same `call_later(async callback that awaits asyncio.to_thread)` shape exists in `_loading_live_hints.py`,
`_index_maintenance.py`, `actions/axe_display/_loaders.py`, and `actions/changespec/_loading.py`. Those do local-disk
work and have not produced recorded stalls, but each holds the pump for the duration of its threaded call. After this
change lands, auditing/converting them to the detached-task pattern is a natural follow-up bead — not included here to
keep the diff reviewable.

## Tests

- `tests/ace/tui/test_agents_bead_warmup.py` — update scheduling tests for the detached-spawn shape; keep all
  coalescing/apply/nav-gate behavior tests green. Add a pump-responsiveness regression test: gate a fake
  `warm_confirmed_bead_displays` on a `threading.Event`, assert a message posted to the app pump after the warmup starts
  is processed _before_ the warmup finishes (fails on the old `call_later` shape).
- `tests/test_agent_bead_display.py` — new tests that the `local_only` path never invokes `materialize_sdd_store` /
  `ensure_workspace_sdd_clone` / git sync (monkeypatch them to raise), and that non-TUI callers keep today's behavior.
- Store-reuse test: count `BeadProject` constructions during a multi-candidate warmup sharing one store.
- Miss-TTL test in the bead display cache (hit TTL unchanged, miss TTL longer).
- Watchdog: unit test that pump-stall records include bounded worker-thread stacks.

## Verification

- `just install`, then targeted suites:
  `pytest tests/ace/tui/test_agents_bead_warmup.py tests/test_agent_bead_display.py tests/ace/tui/test_post_launch_jk_lag.py`
  plus the watchdog tests; then `just check` (modulo known pre-existing failures).
- Manual: run the TUI with agents visible whose bead candidates miss, launch an agent fan-out, and confirm j/k stays
  responsive; confirm no new `tui_pump_stall` entries in `~/.sase/logs/tui_stalls.jsonl` and no SDD `git pull`/`clone`
  attributable to the TUI's warmup path in `tui_git_ops.jsonl`/`tui.log`.
