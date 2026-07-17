---
tier: epic
title: Residual ACE TUI freeze elimination and sub-threshold stall observability
goal: 'The ACE TUI never blocks from the user''s perspective: the remaining UI-thread
  I/O hazard (config-token freshness scans) is moved off latency paths, agent-list
  refreshes can no longer wedge behind artifact-index / SQLite contention, and the
  stall watchdog records sub-threshold hitches so any future perceived freeze leaves
  an attributable forensic record instead of vanishing below the 5-second floor.

  '
phases:
- id: hitch-telemetry
  title: Record sub-threshold loop and pump hitches in the stall watchdog
  depends_on: []
- id: config-token
  title: Serve stale config tokens and revalidate freshness off-thread
  depends_on: []
- id: loader-decouple
  title: Apply loaded agents before cleanup and bound cleanup contention
  depends_on: []
- id: handler-hygiene
  title: Take slow awaits out of mount and modal-open handler paths
  depends_on: []
- id: verify
  title: End-to-end freeze verification under lowered thresholds
  depends_on:
  - hitch-telemetry
  - config-token
  - loader-decouple
  - handler-hygiene
create_time: 2026-07-17 07:49:20
status: wip
bead_id: sase-6j
---

# Plan: Residual ACE TUI freeze elimination and sub-threshold stall observability

## Context and evidence

The user reports the ACE TUI still freezes intermittently after the sase-6c epic ("Eliminate ACE TUI multi-second
freezes and startup regressions") landed on 2026-07-16. A forensic pass over `~/.sase/logs/tui_stalls.jsonl` (+ rotated
file), `tui.log`, `tui_git_ops.jsonl`, `tui_launch_timing.jsonl`, and `tui_toasts.jsonl` established:

1. **All 21 recorded stall events (2026-07-15 09:30 through 2026-07-16 06:26) predate the sase-6c fixes.** Their
   pump-task await chains show exactly the two signatures sase-6c.1 fixed (`_on_auto_refresh` and
   `_run_live_hint_refresh` awaited on Textual's message pump), plus one 11s `tui_stall` where the UI thread sat inside
   `config/core.py::_get_overlay_paths` → `glob/scandir` (the signature sase-6c.2 throttled). The TUI process running
   since then (restarted several times via dev-update re-exec, most recently 2026-07-17 07:17) runs post-fix code and
   has logged **zero** stall records.
2. **The watchdog cannot see the freezes the user now reports.** Both the loop and pump watchdogs record only stalls ≥
   5s (`DEFAULT_THRESHOLD_SECONDS` in `src/sase/ace/tui/util/stall_watchdog.py`). A 1–4s freeze is extremely noticeable
   to a human and currently leaves no record at all.
3. **The config-token freshness scan still runs inline on the UI thread.** sase-6c.2 added a 0.75s time gate
   (`_CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS` in `src/sase/config/core.py`), but when the gate expires the next caller —
   frequently a render-path caller such as the slow-tool render tick → `build_header_text` → `resolve_model_alias` —
   synchronously runs `_compute_current_config_token()`: several `stat` calls, the `CONFIG_DIR.glob("sase_*.yml")`
   overlay scan, `discover_project_root()`, and `resolve_project_layout()`. Under disk-pressure storms (this host
   routinely runs 3+ axe runners plus agent fan-outs; the recorded stall shows a single `scandir` blocking 11s) each
   expiry is an unbounded synchronous I/O window on the UI thread. Throttling reduced frequency, not worst-case latency;
   rule 8 of the TUI perf guidance ("render paths never stat/glob") is still violated at every gate expiry.
4. **Agent-list refreshes can wedge behind background contention.** The worst recorded freeze (159s pump stall,
   2026-07-16 06:21) shows `_load_agents_async` awaiting `compute_loader_cleanup` (in
   `src/sase/ace/tui/actions/agents/_loading_disk.py`, cleanup runs _between_ the disk load and `_apply_loaded_agents`)
   while a kill-task worker thread sat in the dismissed-bundle SQLite index (`dismissed_bundle_index/_schema.py`,
   `BEGIN IMMEDIATE` + 30s busy_timeout) and another worker sat in the Rust artifact-index delete under the process-wide
   `agent_artifact_index_operation_lock`. sase-6c.1 took this off the pump, so keys now keep working — but the freshly
   loaded rows still cannot apply until cleanup finishes, and the `_agents_loading` guard suppresses all further agent
   refreshes for the duration. From the user's perspective the Agents tab is frozen (stale) for minutes even though the
   cursor moves.
5. **A full sweep of all 88 `call_later` / `call_after_refresh` / `call_next` / `set_timer` / `set_interval` call sites
   under `src/sase/ace/tui/` found no remaining pump-blocking async callbacks** except the intentional,
   `wait_for`-bounded quit-path flush (`lifecycle.py::_flush_then_do_quit`).
6. **A companion sweep of every async `action_*` / `on_*` / `watch_*` / `key_*` handler (20 total) found six that await
   slow work on a pump.** Excluding the intentional quit path, the live ones are: the App's `on_mount`
   (`actions/_startup_mount.py`) awaits `asyncio.to_thread` six times (notifications, prompt-stash counts, changespecs,
   last selection, query save, version resolution), suspending App-level key dispatch during startup;
   `modals/prompt_history_modal.py::on_mount` and `modals/revive_agent_modal.py::on_mount` await `to_thread` disk page /
   corpus loads on the modal's pump at open; and `modals/xprompt_browser_actions.py::action_edit_xprompt` (also reached
   via the filter input's `action_forward`) awaits `to_thread` xprompt file reads. Under disk pressure each is an
   unbounded input freeze at startup or modal-open — intermittent, below the watchdog's floor, and exactly the shape of
   the current reports.

## Design

### Phase `hitch-telemetry` — sub-threshold hitch records

Extend `_EventLoopStallWatchdog` with a second, lower detection tier for both the loop and the pump:

- New thresholds (default ~1.5s, env-overridable via `SASE_TUI_HITCH_THRESHOLD_SECONDS` /
  `SASE_TUI_PUMP_HITCH_THRESHOLD_SECONDS`, plus disable flags following the existing `SASE_TUI_STALL_*` naming scheme).
  The existing 5s tier and its record shapes are unchanged.
- New compact record events (`tui_hitch`, `tui_pump_hitch`, and matching `*_recovered` rows with durations) written
  through `log_tui_stall` to the same JSONL. Compact means: timestamp, pid, stall/duration seconds, context fields (tab,
  index, last action, keypress age), and the main-thread stack captured from the watchdog thread via
  `sys._current_frames()` — **no** full asyncio-task dump and no worker thread dump. For pump hitches the main-thread
  stack alone already names the stuck await when the loop is executing the pump coroutine; capturing every task's chain
  (the ≥5s tier's behavior, ~100KB/record) is too heavy for a tier that may fire on 1.5s hitches.
- Keep the hitch state machine separate from the stall state machine so a 6s freeze yields one hitch record, one stall
  record, and their recovery rows — no double-fire loops and no suppression of the richer record.
- Rate-limit hitch records (e.g. a small per-minute cap with a suppressed-count field) so pathological churn cannot
  flood the log.
- Unit tests live alongside `tests/ace/tui/util/test_stall_watchdog.py`: threshold plumbing from env, hitch → recovery
  sequencing, hitch+stall interplay on one long freeze, rate limiting, and compact record shape.

This phase is first-class product observability, not scaffolding: it is the mechanism by which any _future_ report of
"the TUI froze" becomes attributable from logs alone, exactly like the 5s tier made this diagnosis possible.

### Phase `config-token` — stale-serve freshness revalidation

Make `current_config_token()` non-blocking after first use:

- When a cached token exists and the freshness window has expired, return the stale token immediately and kick a
  single-flight background recompute (module-owned lazily-started daemon worker; thread-safe swap of the cached token +
  deadline under a lock). Callers on latency-critical threads never pay for stats, the overlay glob,
  `discover_project_root()`, or `resolve_project_layout()` again.
- Inline (synchronous) compute remains only where freshness is semantically required and cheapness is guaranteed: the
  very first call in a process and the first call after an explicit invalidation (`clear_config_cache()` /
  `_reset_current_config_token_cache()` — config writes, the config edit modal, and tests all funnel through these
  already, so config edits made through sase surfaces keep their read-your- writes behavior).
- External edits (someone editing `~/.config/sase/sase.yml` in an editor) are now noticed within roughly two windows
  instead of one; that trade-off is acceptable and should be stated in the module docstring.
- No thread is spawned at import time; short-lived CLI processes that call `current_config_token()` once never start the
  worker.
- Tests: stale-serve returns instantly with a recompute in flight, single-flight coalescing, post-`clear_config_cache()`
  inline recompute semantics preserved (existing tests already cover this contract), eventual freshness after an
  external file edit, and thread-safety of the swap.

### Phase `loader-decouple` — apply before cleanup, bound cleanup contention

Restructure `_load_agents_async` in `src/sase/ace/tui/actions/agents/_loading_disk.py`:

- Move the `compute_loader_cleanup` await out of the load→apply pipeline: apply the freshly loaded rows first, then run
  cleanup as a separate coalesced follow-up step (same pump-free-task pattern the loader already uses;
  scheduled/running/pending guards per the established coalescing idiom). The `orphaned` set subtraction and
  `_CLEANED_ARTIFACT_DIRS` bookkeeping move with it and still apply their UI-visible effects on the UI thread after the
  await.
- Consequence: artifact-index / SQLite contention can delay _cleanup_, but can no longer delay row application, and
  `_agents_loading` clears as soon as rows are applied — the stale-tab wedge disappears.
- Bound cleanup's contention windows so a wedged cleanup also cannot pile up behind locks for minutes: cleanup is
  self-healing (it re-runs on the next load), so on lock/database contention it should skip and retry later rather than
  wait. Prefer existing bounded-wait APIs on the Rust core side for the artifact-index operations; only extend
  `../sase-core/crates/sase_core` (wire/API + bindings + tests there) if no bounded variant exists. For the
  dismissed-bundle SQLite index, the TUI-side cleanup path should use a short busy timeout rather than the 30s default,
  treating "database is busy" as "skip this round".
- Promote the loader's existing per-stage debug timings to a durable telemetry record when cleanup (or any stage)
  exceeds a threshold (~2s), so future wedges are visible in JSONL without debug logging.
- Tests: rows apply and `_agents_loading` clears even when cleanup blocks (fake cleanup that hangs), cleanup coalescing
  (burst of loads → one trailing cleanup), contention-skip behavior, and orphaned-set bookkeeping equivalence with the
  current behavior.

### Phase `handler-hygiene` — no slow awaits on mount/open handler paths

Convert the six hazardous handlers found by the sweep to the established sync-spawner shape (validate synchronously,
spawn the slow body with `spawn_pump_free_task()` or the widget-appropriate retained-task pattern, apply results back on
the UI thread, show a lightweight loading state where the surface would otherwise be empty):

- `actions/_startup_mount.py::on_mount`: move the six `asyncio.to_thread` awaits into the existing post-mount
  background-load pattern (`_start_post_mount_background_loads` is already the sync spawner for neighboring work) so
  App-level key dispatch is live as soon as the first frame paints. Preserve the current first-paint ordering contract
  and the startup stopwatch semantics (rule 9: first paint never waits on data-scaled work — this phase must not _add_
  work before the stopwatch ends).
- `modals/prompt_history_modal.py::on_mount` and `modals/revive_agent_modal.py::on_mount`: open instantly with a loading
  row, load pages/corpora in a spawned task, patch options on completion (both already have async loaders — only the
  awaited-on-pump seam moves).
- `modals/xprompt_browser_actions.py::action_edit_xprompt` (and its `action_forward` caller in
  `xprompt_browser_filter_input.py`): spawn the file read; the quit-path `action_quit` flush stays as-is (intentional,
  bounded, terminal).
- The quit-path `_flush_then_do_quit` and `action_quit` are explicitly out of scope (intentional, `wait_for`-bounded,
  exit-only).
- Tests: each converted handler gets a regression test asserting the handler coroutine completes without awaiting the
  slow body (e.g. slow body stubbed with an event that only the spawned task sets), plus existing modal behavior tests
  keep passing. Follow the perf-memory rule: after converting, re-sweep the five scheduling APIs and the async handler
  classes for this file set to confirm nothing new slipped in.

### Phase `verify` — end-to-end freeze verification

- Soak the real TUI with hitch thresholds lowered (`SASE_TUI_*STALL*`/hitch env vars) through the flows that
  historically froze: startup + immediate typing, launch fan-out, bulk kill/dismiss while auto-refresh runs, opening the
  prompt-history and revive-agent modals with large histories, tab switching during heavy agent churn, and a synthetic
  lock-contention scenario (hold the artifact-index operation lock and the dismissed-bundle index write transaction from
  a helper while driving a refresh) — assert zero hitch records from the fixed paths, rows still apply, and cleanup
  defers.
- Run the existing key-to-paint benches (`tests/ace/tui/bench_tui_jk.py`, `tests/perf/bench_tui_trace.py`) and confirm
  p95 targets still hold, plus the startup stopwatch is unchanged.
- Update `docs/perf_runbook.md` with the hitch tier (what fires, how to lower thresholds, how to read compact records).
- Full `just check`.

## Explicitly out of scope

- The SDD sidecar `git pull --rebase` retry storms observed in `tui_git_ops.jsonl` (24 pulls × 20s timeouts around
  2026-07-17 01:26, rebase-conflict wedge in a plans clone). They run in worker threads / axe-side processes and do not
  block the TUI pump; they deserve their own investigation.
- Live-hint VCS probe batching (`_compute_live_hints` runs candidates' `git diff` probes serially in one worker thread).
  It only delays pencil badges, never input.
- `sase/memory/tui_perf.md` updates (memory edits require explicit user approval; the landing agent should surface a
  suggested addendum instead of editing memory).

## Risks

- **Stale config tokens**: callers that key caches on the token now see a slightly older token during revalidation. All
  known callers use the token for cache-freshness only, and explicit invalidation still forces inline recompute, but the
  phase must grep token consumers and confirm none require synchronous freshness.
- **Cleanup reordering**: applying rows before cleanup means one refresh can briefly show rows whose artifacts a
  concurrent kill already deleted; the loader already tolerates missing artifact dirs (it self-heals on the next pass),
  but tests should pin that tolerance.
- **Hitch-tier noise**: a too-low default threshold could spam records on slow machines; the rate limiter and env
  overrides are the escape hatch, and the default (~1.5s) sits well above normal frame costs measured by the jk benches
  (p95 < 16ms).
