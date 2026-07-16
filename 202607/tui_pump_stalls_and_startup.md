---
tier: epic
title: Eliminate ACE TUI multi-second freezes and startup regressions
goal: 'Typing and navigation in the ACE TUI never freeze behind background refresh
  work (message-pump stalls drop to zero in normal operation), and TUI startup no
  longer pays for artifact-index schema rebuilds or redundant per-call config disk
  I/O before first interactive paint.

  '
phases:
- id: pump
  title: Move pump-blocking async refresh callbacks onto free-standing loop tasks
  depends_on: []
- id: render-io
  title: Stop per-call config stat/glob I/O in main-thread render paths
  depends_on: []
- id: startup
  title: Take the artifact-index schema rebuild off the startup-critical path
  depends_on: []
- id: updates
  title: Make periodic update checks revalidate-only between full recomputes
  depends_on: []
- id: verify
  title: End-to-end freeze and startup verification
  depends_on:
  - pump
  - render-io
  - startup
  - updates
  model: haiku
create_time: 2026-07-16 11:13:13
status: wip
bead_id: sase-6c
---

# Plan: Eliminate ACE TUI multi-second freezes and startup regressions

## Symptoms

1. The ACE TUI intermittently freezes for multiple seconds — including while typing in the prompt input widget — several
   times per session.
2. TUI startup time has increased significantly.

## Diagnosis (evidence-backed; do not re-litigate)

This diagnosis was produced by mining the TUI's own telemetry, not by guessing. Read these sources yourself if you need
to confirm anything:

- `~/.sase/logs/tui_stalls.jsonl` and `~/.sase/logs/tui_stalls.jsonl.1` — the always-on stall watchdog
  (`src/sase/ace/tui/util/stall_watchdog.py`) recorded **9 message-pump stalls and 1 event-loop stall over
  2026-07-15/16**, durations 6.5s–14.5s typical, worst case **159s**, with full `asyncio_task_stacks` captured for each.
- `~/.sase/perf/tui_trace.jsonl` — span timings from a live session: `agents.load_from_disk` max 3.17s (8 runs/session),
  `agents.live_hint_refresh` max 1.79s (7 runs/session).

### Root cause 1 — async callbacks scheduled onto Textual's serial app message pump

Textual runs `call_later`, `call_after_refresh`, and **timer callbacks** (`set_interval` / `set_timer` post via
`call_next`) on the App's serial message pump. When such a callback is an `async def` that awaits, the pump awaits it to
completion before processing the next message — including key events. Typing freezes for the full duration of the
awaited work.

The stall log's `Task-1` (App pump) stacks attribute every recorded pump stall to exactly two call sites:

- **6 of 9 stalls**: `src/sase/ace/tui/actions/agents/_loading_live_hints.py:114`
  (`self.call_later(self._run_live_hint_refresh)`, plus the `set_timer` re-arm at line 126). The pump then sits in
  `await asyncio.to_thread(_compute_live_hints, candidates)` — one batch of per-agent live VCS diff probes (`git diff`
  per active row) that takes seconds under load.
- **3 of 9 stalls** (including the 159s one): `src/sase/ace/tui/actions/event_refresh/_auto_refresh.py` —
  `_on_auto_refresh` is an async timer callback (wired via `set_interval` in `_startup_mount.py` and re-armed via
  `set_timer` at `_auto_refresh.py:65`) that directly `await`s `_load_axe_status_async()` (line ~88),
  `_load_agents_async(...)` (line ~167; measured 1.5–3.2s per run, 159s under I/O pressure), and
  `_reload_and_reposition_async()` for changespecs.

This exact bug class was already fixed once for other paths by commit `b788ca522` ("fix(tui): prevent message pump
starvation", plan `plans/202607/tui_pump_starvation_freeze.md`), which established the repair pattern: a thin
**synchronous** scheduler that spawns the async work as a free-standing named `loop.create_task(...)` with a
done-callback registry. See `_spawn_agents_refresh_task` (`src/sase/ace/tui/actions/agents/_loading_refresh.py:170-198`)
and `_spawn_bead_confirmation_warmup_task` (`_loading_bead_warmup.py:73-96`). The live-hints and auto-refresh paths
simply never got converted.

An AST sweep found the remaining `call_later(<async def>)` sites; the heavy ones (they await disk/subprocess/`to_thread`
work) are:

- `actions/agents/_loading_live_hints.py:114` (+ `set_timer` re-arm at 126)
- `actions/event_refresh/_auto_refresh.py` (`_on_auto_refresh` itself)
- `actions/agents/_index_maintenance.py:68,112` (`_run_artifact_index_maintenance` — O(archive), holds the
  artifact-index RLock)
- `actions/agents/_loading_refresh.py:278` (`_run_agent_artifact_delta_refresh`)
- `actions/changespec/_loading.py:351,365` (`_run_changespecs_async_refresh`)
- `actions/axe_display/_loaders.py:228,320` (`_load_axe_status_async`, `_refresh_selected_axe_item_async`)
- `actions/agents/_cleanup_tasks.py:97` and `actions/agents/_notification_provider.py:308`
  (`_refresh_notification_count_async`)

Lighter async `call_later` sites (modal opens, seeding prompts, `_runner` closures in `actions/base.py`,
`util/io_async.py`, etc.) should be evaluated but only converted where they demonstrably await slow work — some rely on
pump ordering semantics.

### Root cause 2 — per-call config stat/glob on the main thread in render paths

The single recorded **event-loop** stall (13.3s on 2026-07-15 20:55) has this main-thread stack:
`AgentPromptPanel build_header_text → append_model_field → resolve_model_alias → get_llm_provider_config → load_merged_config → current_config_token → CONFIG_DIR.glob("sase_*.yml") → scandir`.

`load_merged_config()` (`src/sase/config/core.py`) memoizes the merged dict, but `current_config_token()` re-stats
`sase.yml`, **globs the config dir for overlays**, and stats the local config on **every call** — synchronous disk I/O
per render call on the main thread. Under heavy system I/O (dozens of running agents doing git work — exactly when the
TUI is busiest) one glob took 13 seconds. Model-alias resolution calls this per rendered agent header/row.

### Root cause 3 — startup gates on artifact-index schema rebuilds and data-scaled loads

The startup stopwatch (footer) ends only when both the agents and axe first async loads apply
(`_startup_mount.py:170-181`). The agents first load (`src/sase/ace/tui/actions/_startup_loads.py:102-131`) runs
`refresh_agent_artifact_index_if_schema_stale()` **before** the first agents query. After every
`AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` bump this performs a full index rebuild — measured **2.76s on an idle warm-cache
system** (33MB index) and materially worse under real startup I/O load. Schema bumps have become frequent: two on
2026-07-15 alone (`75bf5c791`, `79cce7991`), plus 2026-07-06 (`5eb450842`) and 2026-06-23 (`937a5c492`, `a15274177`).
With near-daily dev updates, users now hit a full-rebuild startup multiple times per week — this is the "startup
suddenly got slower" experience.

On top of that, every first load pays data-scaled costs (profiled against real home data): tier-1 Rust index query
0.85–1.1s, and `classify_diff_badges` (`src/sase/ace/tui/models/_agent_status_diff.py:49`) ~0.5s re-reading and
re-parsing persisted diff files (`diff_has_real_edits` → `changed_files_from_diff` → `shlex.split`) on **every** agents
load, even though `diff_path` artifacts are immutable once finalized.

### Root cause 4 (amplifier) — periodic update checks recompute over the network every tick

`feat(ace): add periodic update checks` (`138a600ac`) schedules `get_cached_update_status(ttl_seconds=...)` every 10
minutes. The default check interval (`_AUTOMATIC_UPDATE_CHECK_INTERVAL_SECONDS = 600` in
`src/sase/ace/tui/actions/update_toast.py`) **equals** the default snapshot TTL
(`DEFAULT_UPDATE_STATUS_TTL_SECONDS = 600` in `src/sase/updates/cache.py`), and freshness is strict `age < ttl`. So
essentially every periodic tick finds the snapshot stale and falls into `compute_update_status(offline=False)` — for
editable installs that means real `git fetch` per editable checkout (`_GIT_MUTATE_TIMEOUT_SECONDS = 120`) plus `gh api`
(20s timeout) and PyPI lookups. This all runs correctly off-thread and releases the GIL, so it does not block the pump
directly — but it adds recurring network/disk/CPU pressure that lengthens the pump stalls above (the 159s stall shows
how bad contention gets), and a single worker can occupy a `startup-loads`-group slot for minutes.

## What is explicitly NOT a cause (verified — don't chase these)

- The Rust `sase_core_rs` bindings **release the GIL**: a 0.6–1.1s `query_agent_artifact_index` call was measured to
  leave a concurrent Python thread completely unstalled.
- The new per-keystroke placeholder-completion Rust calls (`b74adbf4c`) are O(document) pure-CPU scans with early-exit;
  sub-millisecond for normal prompts.
- All `sase.updates` functions and the update toast pipeline already run in `thread=True` workers with
  `call_from_thread` marshalling; no synchronous main-thread caller exists.
- On-demand prompt formatting (`fac33c7a2`) is off-thread and explicit-action only.

## Phase: pump — Move pump-blocking async refresh callbacks onto free-standing loop tasks

**Goal:** no async callback that awaits disk/subprocess/`to_thread` work ever runs on the App message pump.

Design:

1. Extract the duplicated create-task pattern from `_spawn_agents_refresh_task` (`_loading_refresh.py:170-198`) and
   `_spawn_bead_confirmation_warmup_task` (`_loading_bead_warmup.py:73-96`) into one shared helper in
   `src/sase/ace/tui/util/` (e.g. `spawn_pump_free_task(owner, coro, *, name, registry_attr)`): named
   `loop.create_task`, task-set registry for teardown, done-callback that logs exceptions and discards from the
   registry. Refactor the two existing call sites onto it.
2. Convert the evidence-backed offenders:
   - `_loading_live_hints.py`: `_schedule_live_hint_refresh` spawns the scan as a task instead of `call_later`; the
     nav-gate deferral re-arm (`set_timer`) must invoke a **sync** spawner, not the async coroutine. Keep the existing
     coalescing flags (`_live_hints_scan_scheduled/_running/_pending`) semantics intact.
   - `_auto_refresh.py`: make the timer-facing `_on_auto_refresh` a thin sync callback that resets the countdown,
     applies the nav/prompt gates, and spawns the actual async refresh body as a task. Add an overlap guard so a slow
     in-flight refresh coalesces with the next tick instead of stacking (the agents leg already has `_agents_loading`;
     the axe and changespec legs need equivalent protection).
3. Convert the same-class heavy sites listed in the diagnosis (index maintenance, artifact delta refresh, changespec
   refresh, axe loaders, notification count refresh). For each, preserve existing coalescing/guard flags. Leave
   pump-ordering-dependent light callbacks alone; document per-site why any surveyed site was left as `call_later`.
4. Re-capture UI state after every await in converted bodies (selection/tab may have changed while the task ran) — this
   is an established repo rule; verify each conversion against it.

Testing: mirror `tests/ace/tui/actions/test_prompt_stash_pump_nonblocking.py` from `b788ca522` — for live hints and auto
refresh, inject a slow `to_thread` body and assert the pump still processes messages (key events) while the work is in
flight, and that coalescing collapses concurrent triggers. Update existing refresh-coalescing and trace tests for the
new spawn shape.

## Phase: render-io — Stop per-call config stat/glob I/O in main-thread render paths

**Goal:** rendering an agent header/row never touches the filesystem for config freshness.

Design:

1. Add a short time-gate to `current_config_token()` in `src/sase/config/core.py`: recompute the stat/glob token at most
   once per ~0.5–1.0s (monotonic), returning the cached token in between. `clear_config_cache()` must also reset the
   time-gate so tests and explicit reload paths stay deterministic. Config edits then propagate within ≤1s, which
   matches current UX. Audit existing tests that mutate config files and immediately re-read; add a fixture or use
   `clear_config_cache()` where the time-gate would mask a write.
2. Memoize the model-alias lookup (`_get_model_aliases` / `resolve_model_alias` in `src/sase/llm_provider/config.py`)
   keyed on the config token so per-row rendering does a dict lookup, not a config re-merge.

Testing: unit tests for the time-gate (token stable within window, refreshed after expiry, reset by
`clear_config_cache()`); a test asserting `resolve_model_alias` performs no disk I/O when config is unchanged (e.g.,
patch `Path.glob`/`os.stat` counters).

## Phase: startup — Take the artifact-index schema rebuild off the startup-critical path

**Goal:** first interactive paint and the startup stopwatch never wait for a full index rebuild; repeat agents loads
stop re-parsing immutable diff artifacts.

Design:

1. Restructure `_run_agent_index_startup_prepare_and_refresh` (`src/sase/ace/tui/actions/_startup_loads.py:102-107`):
   check schema staleness cheaply (read the stored schema meta) without rebuilding. If fresh → current behavior. If
   stale → run the first agents load immediately via the existing bounded fallback scan path
   (`_scan_artifacts_for_loader(_TIER1_FALLBACK_SCAN_OPTIONS)` in `src/sase/ace/tui/models/agent_loader.py`), start the
   rebuild in a background worker, and on completion schedule a follow-up
   `_schedule_agents_async_refresh(source="index_schema_rebuilt")`. Details to get right:
   - While the rebuild holds the in-process artifact-index RLock (`src/sase/core/agent_artifact_index_lock.py`), no
     other startup path may issue an index call that would block behind it — gate index usage on a "rebuild in flight"
     flag so the loader takes the bounded-scan branch directly instead of blocking on the lock.
   - Suppress the `repair_recommended` → index-maintenance reaction while the rebuild is in flight so the stale-schema
     fallback does not trigger a second, duplicate rebuild.
2. Cache `diff_has_real_edits` (`src/sase/ace/tui/models/_diff_badge.py`) keyed on `(diff_path, mtime, size)` so
   `classify_diff_badges` stops re-reading and `shlex`-parsing the same finalized diff artifacts on every load (repo
   rule: cache disk reads keyed by mtime).
3. Measure before/after with the recipe in `tests/perf/README.md` and record numbers in the phase summary: schema-stale
   startup (agents-first-load wall time) must no longer include the ~2.8s+ rebuild; steady-state
   `load_agents_from_disk_with_state` should drop by roughly the diff-badge parse cost (~0.4–0.5s against current real
   home data).

Out of scope but worth noting for a follow-up: the tier-1 Rust index query costs 0.85–1.1s for a bounded query over a
33MB index; if further startup wins are needed, that investigation belongs in the Rust core (`sase-core`) per the
backend-boundary rule, not in this repo.

## Phase: updates — Make periodic update checks revalidate-only between full recomputes

**Goal:** the 10-minute session tick never triggers the full network recompute (`git fetch` per editable checkout /
`gh api` / PyPI) as a matter of course.

Design:

1. Split "read + revalidate the cached snapshot" from "recompute over the network" at the `get_cached_update_status`
   boundary (`src/sase/updates/cache.py`): add an explicit revalidate-only mode (no `compute_update_status` fallthrough)
   and have the **periodic** tick in `update_toast.py` use it. The startup check and explicit user actions (Updates tab,
   `,U` flow) keep today's recompute-when-stale behavior.
2. Give the network recompute its own, longer cadence: on a periodic tick, only fall through to the full recompute when
   the snapshot age exceeds a dedicated recompute interval (default meaningfully larger than the 10-minute tick, e.g. 60
   minutes; keep it configurable alongside the existing `ace.updates.check_interval_minutes` / `check_ttl_minutes`
   knobs). Any new config key must be added to `src/sase/config/sase.schema.json` and documented consistently with the
   existing `updates` keys; also fix the interval==TTL boundary coupling so the default tick no longer lands exactly at
   snapshot expiry.
3. Preserve all current notification surfaces (startup toast once per session, indicator badge, post-update toast) —
   this phase changes only _when_ the expensive recompute runs.

Testing: unit tests that a periodic tick with a stale-beyond-TTL but fresh-within-recompute- interval snapshot performs
no network/`compute_update_status` call; that the startup check still recomputes; and that the recompute interval knob
is honored and schema-validated.

## Phase: verify — End-to-end freeze and startup verification

**Goal:** demonstrate, with the repo's own instrumentation, that the fixes hold.

1. Run the pump-nonblocking regression tests plus the full `just check` gate.
2. Exercise a realistic session under `SASE_TUI_TRACE=1` and the always-on stall watchdog (thresholds can be lowered via
   `SASE_TUI_STALL_THRESHOLD_SECONDS` / `SASE_TUI_PUMP_STALL_THRESHOLD_SECONDS` to make regressions loud in a short
   run): simulate agents-tab refresh churn while typing (Textual Pilot; reuse the harness patterns from
   `tests/perf/bench_tui_trace.py`) and assert zero pump-stall records at a 1s threshold.
3. Re-run the startup timing recipe from `tests/perf/README.md` against a schema-stale index (copy the real index to a
   temp path, rewrite its schema meta to the previous version) and confirm the first agents load completes without
   waiting on the rebuild.
4. Summarize before/after numbers (stall counts/durations, startup stopwatch, load timings) in the phase output so the
   results are auditable.

## Risks

- **Timer/callback semantics:** converting pump callbacks to loop tasks changes ordering guarantees (pump callbacks are
  serialized; tasks interleave). Each converted site must keep its coalescing guards and re-capture UI state after
  awaits, or j/k selection bugs will appear.
- **Config token time-gate staleness:** a ≤1s delay in observing config file edits. Mitigated by `clear_config_cache()`
  on explicit reload paths and the test audit in the render-io phase.
- **Stale-schema fallback correctness:** the bounded source scan shows a slightly less complete inbox until the
  background rebuild lands; the follow-up refresh must be reliable (worker done-callback, not fire-and-forget) or rows
  could stay stale for a session.
- **Update-check behavior change:** a genuinely new release can take up to the recompute interval to surface
  mid-session. Startup checks and explicit user checks are unchanged, which bounds the staleness to long-running
  sessions.
