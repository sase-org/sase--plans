---
tier: epic
title:
  Stop the ACE TUI's 10% freeze budget (GC off the interactive path, one live snapshot
  per key)
goal: "The long-lived ACE TUI stops spending 10-17% of wall time frozen. Full (gen-2)
  garbage collections no longer start while the user is interacting. The heap stops
  growing about 100 MB/min from superseded snapshot copies. The per-second UI-thread
  residue is gone. Telemetry measures GC pauses, RSS/swap, and the true frozen share
  directly, so the acceptance check is data, not inference.

  "
phases:
  - id: gc-telemetry
    title: GC pause recorder, memory heartbeat, and app-instance identity
    depends_on: []
    size: medium
    description:
      "gc-telemetry: add a lock-free gc.callbacks recorder with collection-trigger
      tagging and a recent-collections ring. A daemon flush thread writes rate-capped
      tui_gc_pause rows and a 5-minute tui_memory_heartbeat (RSS, VmSwap, major faults,
      exact per-generation GC totals, pluggable extra fields). Also add a per-instance
      app ID and an exec-aware startup clock. Telemetry is never auto-installed under
      the ace testing harness."
  - id: watchdog-truth
    title: Make the stall watchdog report whole-process stops and exact totals
    depends_on:
      - gc-telemetry
    size: medium
    description:
      "watchdog-truth: detect whole-process stops from the watchdog's own poll lateness,
      even when the beacon already ran. Add late/poll_lag_s/net_stall_seconds and
      gc-overlap attribution to hitch and recovery rows, and count rate-limited episodes
      and seconds into the heartbeat. Ship a tools/tui_freeze_report script that
      computes the de-duplicated frozen share and the GC share per app instance."
  - id: snapshot-caches
    title: One live version per path or scope in the module snapshot caches
    depends_on: []
    size: medium
    description:
      "snapshot-caches: re-key the artifact-file index cache by resolved path, and the
      artifact-links and LinkIndex caches by project scope, so a changed stat or
      signature replaces the entry instead of piling up 32/64 superseded full copies.
      Add a just-check structural test (N external rewrites leave one entry per key) and
      audit the remaining token-keyed module caches."
  - id: idle-gc-policy
    title: Take automatic gen-2 collection off the interactive path
    depends_on:
      - gc-telemetry
      - snapshot-caches
    size: medium
    description:
      "idle-gc-policy: once startup loads settle and input first goes idle, run
      gc.collect() then gc.freeze() exactly once. Raise threshold2 to 10_000 on the
      classic three-generation collector. Run tagged full collections only when input
      has been quiet for 3 s, no prompt is active, and a collection is due, with
      5-minute and RSS-growth backstops and an off-thread malloc_trim. Includes an env
      kill switch, a clean uninstall, and no install under the testing harness."
  - id: cached-snapshot-sharing
    title: Share immutable cached snapshots instead of copying them on every hit
    depends_on: []
    size: medium
    description:
      "cached-snapshot-sharing: stop the notification facade deep-cloning about 1.6k
      rows on every cache hit. Make shared rows safe through immutability or audited
      copy-on-write callers. Return the cached frozenset of dismissed bundle identities
      and the cached artifact-index tuple instead of fresh copies, with guard tests that
      caller mutation cannot corrupt a cache."
  - id: tick-compare-skip
    title:
      Compare-then-skip on the per-second Agents tick and explicit prompt-active state
    depends_on: []
    size: medium
    description:
      "tick-compare-skip: key runtime-row patches on (membership, displayed second) and
      skip the asdict/Rust aggregation when nothing visible changed. Key the info-panel
      metrics cache on explicit roster/status/unread generations instead of walking
      bulk_ack_roster_universe each second. Back _prompt_input_active() with explicit
      app state that mount, detach, and editor-suspend maintain, plus a parity test
      against the DOM query."
  - id: off-loop-refresh
    title:
      Move fleet projection, digest building, and config-token refresh off the hot path
    depends_on: []
    size: medium
    description:
      "off-loop-refresh: project fleet clan/tribe trees on the worker from immutable
      inputs and revalidate generation and selection on apply. Skip the prompt-panel
      Rich-tree digest on a cheap identity/content token. Replace the per-refresh
      Thread.start() in current_config_token() with one long-lived revalidator thread so
      the getter only peeks."
  - id: acceptance
    title: Live before/after measurement on athena and follow-up capture
    depends_on:
      - gc-telemetry
      - watchdog-truth
      - snapshot-caches
      - idle-gc-policy
      - cached-snapshot-sharing
      - tick-compare-skip
      - off-loop-refresh
    size: small
    description:
      "acceptance: on a TUI restarted onto the landed code, capture a busy hour plus a
      4-hour RSS window. Use tools/tui_freeze_report to compare the frozen share, GC
      share, idle-collection pauses, RSS/swap growth, and j/k p95 against the 10.7-12.9%
      / 16.7%-gen-2 baseline. Record misses and the proposed tui_perf.md rule update as
      PROPOSED FOLLOW-UP notes."
proposed_by: bbugyi200.athena.0vm
create_time: 2026-10-02 16:44:52
status: wip
---

- **PROMPT:**
  [prompts/202610/tui_freeze_gc_heap.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/tui_freeze_gc_heap.md)

# Plan: Stop the ACE TUI's 10% freeze budget

## Why the TUI is frozen about 10% of the time

`prompt_space_and_project_cycle_latency.md` found that the live TUI spent 10% of wall
time inside ≥1.5 s event-loop hitches. The follow-up investigation,
`research:202610/tui_freeze_gc_heap_bloat/tui_freeze_gc_heap_bloat.md`, measured the
cause directly. Phase workers should read it with `sase artifact read` before starting;
the cld and cdx sub-reports in the same directory carry the raw probes.

- **The frozen time is mostly stop-the-world full GC.** A live `gc.callbacks` probe (PEP
  768 `sys.remote_exec`, image about 1 h old) saw these results:
  - 13 gen-2 collections in 179 s, each 1.8–3.1 s (p50 2.16 s);
  - **16.7% of wall time in gen-2, and 25.5% in all GC**;
  - 8 of the 13 collections ran on worker or input threads. They still freeze the UI,
    because the collector holds the GIL. So pushing more work into workers cannot fix
    this.
- **The watchdog undercounts.** 7 of 13 full collections produced no hitch row. Pauses
  under 1.5 s are invisible, and only 4 episodes per minute are recorded. The 10.7–12.9%
  stall-log share is a floor.
- **The heap grows about 100 MB/min** (1.48 GB → 4.66 GB RSS + 1.46 GB swap in 50 min).
  Pause length scales with the old generation. Swap turns 1 s pauses into 3–10 s ones.
- **A confirmed cache-keying bug feeds that growth.** Three module caches key on the
  backing file's `(mtime_ns, size)` and cap the _number of versions_ (32/64), not
  versions per file. Agents append constantly, so each append mints a new key and the
  superseded copies are never read again. The live census showed 22 copies of the
  26.8k-row artifact index (about 40 MB each) and 10 copies of the 35k-row
  artifact-links snapshot.
  - `src/sase/core/artifact_file_explicit.py:44-48, 422-445`:
    `_artifact_file_index_cache` is keyed `(path, (mtime_ns, size))`, with
    `_INDEX_CACHE_MAX_PATHS = 32` acting as a version cap.
  - `src/sase/ace/tui/relations/artifact_links.py:80-139`: `_CACHE` is keyed by the full
    signature tuple, with `_CACHE_MAX = 64`. The comment admits that "a superseded
    signature is never looked up again". `sase-zn.9.3` capped the leak instead of
    removing it.
  - `src/sase/ace/tui/relations/link_index.py:74-99`: `_INDEX_CACHE` is keyed by
    `snapshot.source_key`, with a cap of 64. That is 53 MB and 208k objects per entry.
- **Why fixing the caches alone is not enough.** On CPython's classic collector, a full
  collection needs both of these:
  - more than `threshold2` (10) gen-1 collections since the last full one;
  - promotions above 25% of the old generation.

  The 25% rule binds, so `pause ≈ c·L` and `interval ≈ 0.25·L/r`, and the frozen share
  `≈ 4·c·r` is independent of heap size. A smaller heap gives shorter but more frequent
  freezes. Raising `threshold2` far enough (10_000) makes the count rule bind instead,
  and the full collection becomes an explicit, idle-time decision.

- **The real UI-thread work is a smaller residue, about 1.5–3% of wall time.** It comes
  from these sources:
  - the 1 s countdown tick's runtime-row aggregation;
  - the info-panel metrics cache key walking `bulk_ack_roster_universe`;
  - fleet clan/tribe projection on the loop;
  - the prompt-panel renderable digest;
  - `current_config_token()` calling `Thread.start()`;
  - `_prompt_input_active()` as a DOM query.

  The prompt keys (`<space>`, `<ctrl+n/p>`) are only 1.6% of freeze-seconds.

Restarting the TUI buys about 18 hitch-free minutes; within the hour it is back to about
13%. This epic is the durable fix.

### Verified on current master

Every cited site was re-read at master `2bbc346036`. Only one commit has landed since
the research revision `3691b88ae7`, and it touches deck layout, not these paths.

- No code in `src/sase` tunes, freezes, or instruments the collector. The only
  `gc.collect()` is `src/sase/axe/runner_idle_memory.py`.
- The Rust perf-log aggregator `sase-core` `crates/sase_core/src/perf_logs/aggregate.rs`
  counts only `STALL_EVENTS` / `RECOVERY_EVENTS` and skips other `event` kinds. New
  `tui_gc_pause` / `tui_memory_heartbeat` rows in `tui_stalls.jsonl` therefore need **no
  sase-core change** and do not distort the Admin Center Perf view. Additive fields on
  `tui_hitch` rows are also safe.
- Input quiescence is already tracked: `_last_input_mono`, written by
  `_record_input_event` from `on_key`, `Input.Changed`, `TextArea.Changed`, and
  navigation.

### Relationship to existing work

- **`sase-zn`** (ACE typing lag, in progress, awaiting its land agent) bounded the
  notification re-parse, the reporter retention, and the dismissed reconcile. Its
  `sase-zn.9.3` added the 32/64 caps this epic replaces. This epic does not reopen
  `sase-zn`'s blockers: the bounded-read RLock fallback and the dismissed-family
  fallback parity stay with `sase-zn`.
- **`sase-vc`** (Tier-1 revalidate re-stats hidden rows; the `tier1_index_revalidate`
  loads averaged about 58 s) is a separate, open Rust-core bug. It is out of scope here.
- **The prompt-key epic** recommended by `prompt_space_and_project_cycle_latency.md` is
  separate and out of scope. That covers the MRU snapshot, `ensure_watches()`, the
  de-duplicated `on_mount`, and the hot-spare bar. This epic does deliver its Phase-2
  prerequisite, the explicit prompt-active state, in `tick-compare-skip`.
- **Rust core boundary.** GC policy and GC telemetry are process-local runtime behavior
  of one Python process. The cache fixes are Python module caches. Nothing here belongs
  in `sase_core`. Moving `LinkIndex` into Rust would take it out of Python's GC
  entirely, but it is a wire/binding/pin change and is deferred (see Out of scope).

## Phases

All new modules live under `src/sase/ace/tui/util/` unless noted. Follow `tui_perf.md`:
timer and pump callbacks stay thin and synchronous; file I/O happens off the UI thread.

Every runtime hook added here (gc callbacks, threads, timers, thresholds, freeze) needs
three guarantees:

- **An uninstall path.** App teardown, including the `os.execv` restart path, restores
  state.
- **No auto-install under the `sase.ace.testing` harness.** Follow the existing
  `_ORIGINAL_START_STALL_WATCHDOG` patching pattern in
  `src/sase/ace/testing/_startup.py`. A pytest process hosts many app instances, and
  `gc.freeze()` or a raised `threshold2` there would change GC behavior for the whole
  suite.
- **An explicit opt-in for tests** that exercise the hook.

### Phase `gc-telemetry`: GC pause recorder, memory heartbeat, and app-instance identity

New `gc_telemetry.py`:

1. **Recorder.** Register one `gc.callbacks` function.
   - On `start`, stamp `time.monotonic()` and the thread name. On `stop`, compute the
     duration and read `info["collected"]`/`["uncollectable"]`.
   - The callback is **lock-free**: plain attribute updates and `deque.append`/`popleft`
     only. No I/O, JSON, logging, stack capture, or `threading.Lock`; a collection can
     fire while the flusher holds a lock.
   - It maintains these:
     - exact per-generation running totals (`count`, `total_s`, `max_s`) since the last
       heartbeat;
     - a bounded ring (about 256) of recent collections
       `(gen, start_mono, end_mono, thread, trigger)`, exposed through
       `recent_collections(since_mono)` for the watchdog's overlap attribution;
     - a bounded flush queue holding every gen-2 collection and any collection ≥ 50 ms.
2. **Trigger tagging.** A `gc_trigger(name)` context manager sets a module-level tag
   that the callback copies into each record. Untagged collections are `"automatic"`.
   The `idle-gc-policy` phase uses it for `startup_freeze` / `idle` / `backstop` /
   `rss_backstop`. Define it here so the two downstream phases do not edit the same
   code.
3. **Flush thread.** A daemon thread named `sase-tui-gc-telemetry` drains the queue
   every few seconds into `tui_gc_pause` rows via `sase.logs.log_tui_stall`.
   - Each row carries `ts`, `pid`, `app_instance_id`, `generation`, `duration_s`,
     `thread`, `trigger`, `collected`, and `uncollectable`.
   - Rows are rate-capped (for example 60/min) with a `suppressed_count`. The heartbeat
     totals stay exact either way.
   - No pump, no Textual timer.
4. **Heartbeat.** Every 5 minutes, plus once shortly after startup, the same thread
   writes a `tui_memory_heartbeat` row with these fields:
   - RSS and `VmSwap` from `/proc/self/status`;
   - the major-fault delta from `/proc/self/stat`;
   - `gc.get_count()`, `gc.get_threshold()`, and `gc.get_freeze_count()`;
   - the per-generation totals since the last heartbeat, and the wall-clock span they
     cover, so GC share is exact;
   - instance uptime.

   It degrades cleanly off Linux. It keeps the latest RSS sample in memory behind a
   cheap accessor (`latest_rss_bytes()`) for the GC policy's RSS backstop; the policy
   never reads `/proc` on the UI thread. Add `register_heartbeat_provider(name, fn)` so
   `watchdog-truth` can contribute its counters without editing this module.

5. **App-instance ID.** Mint one short random ID per app instance at construction.
   Expose it through a small accessor and add it to these records:
   - `tui_gc_pause` and the heartbeat;
   - `tui_startup.jsonl`;
   - `tui_agent_loads.jsonl`.

   Watchdog rows get it in the next phase. Add the imported source revision to the
   startup record if it is cheap; `sase_version` alone cannot identify an editable
   checkout.

6. **Exec-aware startup clock.** `src/sase/ace/tui/util/startup_clock.py` derives
   interpreter start from `/proc/self/stat`. That survives `os.execv`
   (`src/sase/main/ace_handler.py:114`), and the result is bogus values such as
   `interpreter_cli_import_seconds = 96,984`.
   - Pass `time.monotonic_ns()` taken just before the exec in an env var.
     `CLOCK_MONOTONIC` survives exec.
   - Prefer that anchor when present; consume and clear it so children don't inherit it.
7. **Install and uninstall.** Install from `_start_post_first_paint_services`
   (`src/sase/ace/tui/actions/_startup_mount.py`) and remove the callback, thread, and
   providers on teardown. Kill switch: `SASE_TUI_GC_TELEMETRY_DISABLE=1`.
8. **Docs.** Extend `docs/perf_runbook.md` "Freeze and hitch capture" with the new row
   kinds and how to read them.

**Tests:**

- The callback records gen-2 collections and collections ≥ 50 ms, and drops shorter
  young collections from the queue while still counting them.
- The callback body performs no I/O or locking. Add a structural test.
- The rate cap and `suppressed_count` work.
- The heartbeat parses `/proc` and falls back cleanly when it is missing.
- A trigger tag lands on the record.
- Uninstall leaves `gc.callbacks` as it found it.
- The exec anchor is used when the env var is present.
- The testing harness does not auto-install telemetry.

### Phase `watchdog-truth`: Make the stall watchdog report whole-process stops and exact totals

Files: `src/sase/ace/tui/util/_stall_watchdog_monitor.py`, `_stall_watchdog_records.py`,
`_stall_watchdog_config.py`, and a new `tools/tui_freeze_report`. Read `tools/AGENTS.md`
first.

1. **Detect stops from the watchdog's own lateness.** In `_poll_once`, compute
   `poll_lag = now − previous_poll_mono − poll_interval`.
   - If `now − previous_poll_mono ≥ hitch_threshold`, the whole process stopped. Record
     one `tui_hitch` with `late: true` and `detected_by: "watchdog_lateness"`, **even if
     the beacon already ran** and the loop gap looks small. This is the GIL-race blind
     spot the probe found.
   - Never double-record an episode that the loop-gap path already recorded.
2. **Additive fields** on hitch/stall rows and their recoveries. Keep `stall_seconds`
   unchanged for compatibility.
   - `late` and `poll_lag_s`;
   - `net_stall_seconds`, the duration minus one poll interval, floored at the threshold
     semantics;
   - `app_instance_id`.
3. **GC-overlap attribution.** On recovery, query
   `gc_telemetry.recent_collections(hitch_start)`. Add `gc_overlap_s`, the overlapping
   generations, and the triggers. A hitch caused by an intentional idle collection is
   then distinguishable from an interactive freeze.
4. **Exact totals.** Track every episode, including rate-limited ones, with its
   recovered duration. Publish `hitch_episodes`, `hitch_seconds`, `suppressed_episodes`,
   and `suppressed_seconds` since the last heartbeat through
   `register_heartbeat_provider`, for the loop and pump tiers separately.
5. **`tools/tui_freeze_report`.** A read-only script that reports per `app_instance_id`
   (falling back to the startup-record windows for old rows) over a `--since/--until`
   window. It computes:
   - the loop-only union frozen share, without double-counting pump and loop tiers;
   - median, p90, and max hitch duration;
   - late versus on-time split;
   - GC share by generation and trigger, from heartbeats and pause rows;
   - idle-collection pause p50/p95/max;
   - automatic gen-2 collections that started within 2 s of input;
   - RSS/swap trajectory.

   The 2 s check uses `last_keypress_age_s` from the context and needs the trigger tag.
   Document it in the perf runbook. It is the measuring tool for `acceptance`.

**Tests:**

- With a fake monotonic clock: a late poll with the beacon already serviced still
  records one `late` hitch.
- No duplicate rows when both detectors fire.
- Overlap attribution works against a stubbed recent-collections ring.
- Suppressed episodes are counted in the totals.
- The report script handles a fixture log that mixes old rows and new rows.

### Phase `snapshot-caches`: One live version per path or scope in the module snapshot caches

1. **`artifact_file_explicit._artifact_file_index_cache`.**
   - Key by resolved path and store `(stat, rows)`.
   - A stat mismatch **replaces** the entry. `_get_cached_artifact_file_index` returns a
     hit only when the stored stat equals the current stat.
   - Keep `_INDEX_CACHE_MAX_PATHS` as a bound on _paths_. Keep the same-process writer
     invalidation (`_invalidate_artifact_file_index_cache`) working.
2. **`relations/artifact_links._CACHE`.**
   - Key by scope, `tuple(project_key for project_key, _, _ in signature)`, and store
     `(signature, snapshot)`. A signature mismatch reloads and replaces that scope's
     entry.
   - Bound the number of scopes, for example 8, LRU.
   - Rewrite the stale comment.
3. **`relations/link_index._INDEX_CACHE`.**
   - Key by the same scope, derived from `snapshot.source_key`; handle the empty
     snapshot's `()` key.
   - A different `source_key` for that scope replaces the old index.
   - Bound scopes the same way.
4. **Concurrency.** Callers already holding an older snapshot keep using it through
   ordinary ownership; the cache simply stops pinning it. A slower worker publishing an
   _older_ signature after a newer one may replace it. That only costs a reload; it must
   never serve a mismatched signature. Add a test for that.
5. **Structural guard** in `just check`. For each of the three caches, perform N
   external rewrites or appends of the backing file and N reads, then assert at most one
   entry per path or scope.
6. **Audit** the other module-level caches. Start from
   `rg "move_to_end|popitem|_CACHE_MAX|_MAX_ENTRIES" src/sase/ace src/sase/core`.
   - The ones keyed by project, identity, or `ref.key` are expected to be fine.
   - Fix any other cache keyed on a file version whose superseded versions are never
     read again.
   - List the audited caches and their verdicts in your bead's closing note.

Expected effect: RSS stays near its warmed-up size instead of growing about 100 MB/min,
and full-collection pauses stop lengthening as the session ages.

### Phase `idle-gc-policy`: Take automatic gen-2 collection off the interactive path

New `gc_policy.py`, driven by a 1 s `set_interval` whose callback is thin and
synchronous.

1. **Freeze once.** After `_mount_state_loads_done` is true, on the first tick where the
   idle gate below passes, run `gc.collect()` then `gc.freeze()` under
   `gc_trigger("startup_freeze")`.
   - Do it exactly once per app instance; a re-exec is a new instance.
   - **Never re-freeze periodically.** Cyclic garbage among frozen objects is never
     reclaimed.
2. **Threshold.** If the collector exposes the classic three-generation threshold
   (`len(gc.get_threshold()) == 3` and the interpreter's documented semantics match),
   set `(t0, t1, 10_000)`, preserving the current `t0`/`t1` (`(2000, 10, …)` on the live
   3.14.7). That makes automatic full collections a rare backstop, about once an hour at
   2.9 gen-1/s.
   - On any other collector shape, leave thresholds alone and record
     `threshold_policy: "unsupported"` in the heartbeat. The idle collector still runs.
   - `requires-python` is `>=3.12`. Cover 3.12/3.13 and 3.14 in the feature test, and do
     not assume the live interpreter's shape.
3. **Idle collection.** Under `gc_trigger("idle")`, call `gc.collect()` on the UI thread
   only when **all** of these hold:
   - `now − _last_input_mono ≥ 3 s`;
   - `not _prompt_input_active()`;
   - the `NavigationGate` is not navigating;
   - no modal screen is accepting text input;
   - at least 60 s since the last full collection of any trigger, from the recorder;
   - a collection is due: at least 100 gen-1 collections since the last full one
     (`gc.get_count()[2]` on the classic collector), or a time-based equivalent where
     that counter is unavailable.

   If mouse clicks and scrolls do not already update `_last_input_mono`, record them,
   without consuming the events. The collection runs while the user is idle, so blocking
   the loop for its duration is the point. The trigger tag lets `watchdog-truth`
   attribute any resulting hitch.

4. **Backstops.**
   - **Time backstop.** After 5 minutes without a full collection, relax the quiet
     window to 1 s, tagged `backstop`.
   - **RSS backstop.** If `gc_telemetry.latest_rss_bytes()` grew by more than 500 MB
     since the last full collection, relax the gate the same way, tagged `rss_backstop`.
   - The raised `threshold2` remains the hard backstop.
5. **Return pages.** After an idle or backstop collection, at most every 10 minutes,
   release freed arenas with `malloc_trim` **on a worker thread**. ctypes releases the
   GIL for the foreign call, so it does not freeze the UI.
   - Refactor `src/sase/axe/runner_idle_memory.py` to expose a public `trim_allocator()`
     (the trim half of `release_idle_memory()`) and use it from both callers.
   - The axe behavior is unchanged.
6. **Kill switch and uninstall.** `SASE_TUI_GC_POLICY_DISABLE=1` disables freeze,
   threshold, and the idle collector; telemetry stays on. Uninstall restores the
   original threshold, calls `gc.unfreeze()`, and stops the timer. Never auto-install
   under the testing harness.
7. **Docs.** Add a perf-runbook section describing the policy, the triggers, the kill
   switch, and how to confirm it from `tui_gc_pause` rows.

**Tests:** drive the gate with a fake clock and stubbed collector/recorder.

- Freeze happens once.
- No collection runs while input is recent, while a prompt is active, or before it is
  due.
- Both backstops fire.
- Thresholds are preserved and restored, and an unsupported collector shape is handled.
- The kill switch works.
- Trim runs off-thread at its cadence.
- The harness does not auto-install the policy.

Expected effect, with `snapshot-caches` in place: the unfrozen heap stays around 0.5–1M
objects. Idle collections take about 0.3–0.7 s and happen only while the user is idle.
No gen-2 collection starts within 2 s of input.

### Phase `cached-snapshot-sharing`: Share immutable cached snapshots instead of copying them on every hit

1. **Notifications.** `src/sase/core/notification_store_facade.py`
   `_read_snapshot_cached` returns `_clone_snapshot(...)` on every hit: `replace()` plus
   five list/dict copies for each of about 1.6k rows. cld's py-spy samples found it the
   hottest worker allocation site.
   - Make cached rows safely shareable. Either give `Notification`'s nested containers
     immutable types, or audit every consumer of
     `read_notifications_snapshot`/`read_current_notifications_snapshot` for in-place
     mutation and convert any mutation to copy-on-write.
   - Then return the cached rows without per-row cloning. A shallow outer list copy is
     fine.
   - Apply the same treatment to the `_INDEX_CACHE` unread-completion copy in the same
     module.
   - Keep the token race guard that `sase-zn` added.
2. **Dismissed bundle identities.** `src/sase/ace/dismissed_agents.py`
   `dismissed_bundle_identities_snapshot()` returns `set(frozenset)` on every hit
   (44k–59k tuples).
   - Return the cached `frozenset`.
   - Update callers that mutate the result to copy explicitly.
3. **Artifact index rows.** `read_artifact_file_index` returns `list(rows)` on every
   hit.
   - Return the cached immutable sequence.
   - Update callers that sort or append in place to copy explicitly.
   - Keep the public type annotation honest.

**Tests:** guard tests that a caller's attempted mutation either fails loudly or cannot
reach the cached copy, and that repeated hits return the same object.

### Phase `tick-compare-skip`: Compare-then-skip on the per-second Agents tick and explicit prompt-active state

1. **Runtime rows.** `_on_countdown_tick` → `_patch_agent_runtime_rows` →
   `AgentList.patch_active_runtime_rows`
   (`src/sase/ace/tui/widgets/_agent_list_widget.py`) → `aggregate_clan_runtime`
   (`src/sase/core/agent_runtime_facade.py`).
   - Key each row on `(membership signature, displayed runtime second)`.
   - Skip the `asdict()` and Rust aggregation when the displayed value cannot have
     changed.
   - **Do not** replace interval-union semantics with naive `+= 1`.
2. **Info-panel metrics.** `_agent_info_metrics`
   (`src/sase/ace/tui/actions/agents/_display_detail_info.py:42`) builds its cache key
   by walking and de-duplicating `bulk_ack_roster_universe` and freezing the unread set
   every second. Key the cache on explicit roster, status, and unread generations that
   the mutating paths bump, so a hit costs O(1).
3. **Explicit prompt-active state.** `_prompt_input_active()`
   (`src/sase/ace/tui/actions/_event_base.py:92`) runs `query(PromptInputBar)` every
   second and on several refresh guards.
   - Back it with an `app._active_prompt_bar` reference that prompt mount, activation,
     and detach maintain, keeping the `_prompt_editor_suspended` short-circuit.
   - Cover every prompt mode: home bar, feedback, approve, and the markdown editor.
   - Add a parity test that drives open, cancel, submit, editor suspend/resume, and
     remount, and asserts the explicit state equals the DOM query at each step.
   - Do not change the hot-spare or `<space>` behavior. That belongs to the prompt-key
     epic.

**Tests:** row-patch skip and invalidation on membership change and on a second
boundary; metrics-key invalidation for each generation bump; the prompt-active parity
test.

### Phase `off-loop-refresh`: Move fleet projection, digest building, and config-token refresh off the hot path

1. **Fleet projection.** `src/sase/ace/tui/actions/agents/_fleet_refresh.py` (about
   line 160) runs `project_clan_tree` → `resolve_clan_tribe`
   (`src/sase/core/agent_clan_tribe.py`) on the loop after its awaits.
   - Compute the projection on the worker from immutable inputs.
   - On apply, re-check the generation and the current tab and selection (`tui_perf.md`
     rule 4).
2. **Prompt-panel digest.** `src/sase/ace/tui/util/renderable_digest.py` walks the whole
   Rich tree (`_update_text_digest`) to produce a digest.
   - Skip the walk when a cheap identity or content token is unchanged.
   - Honor existing `CachedRenderable` digests instead of re-hashing their contents.
3. **Config token.** `current_config_token()` (`src/sase/config/core.py:320-360`) spawns
   a new refresh thread on each deadline expiry. The 5 s `LaunchContextSource` timer
   then waits in `Thread.start()` → `_started.wait()`.
   - Use one long-lived daemon revalidator thread that refreshes on its cadence. The
     getter only peeks at the cached value and never starts a thread.
   - Preserve `clear_config_cache()` semantics and the cwd/config-dir keying.
   - Mind `tests/` that assert refresh-thread naming (`sase-t6` documents one such
     flake).

**Tests:**

- The projection applies only when generation and selection still match.
- The digest skip is honored and invalidated on a real change.
- The config-token getter never calls `Thread.start()` after warm-up, picks up a changed
  config within one cadence, and respects `clear_config_cache()`.

### Phase `acceptance`: Live before/after measurement on athena and follow-up capture

1. **Prerequisite: a restarted TUI.** The user's live TUI must be running the landed
   code; an editable install keeps its imported snapshot (`tui_perf.md` rule 15). **Do
   not kill or restart the user's TUI yourself.**
   - If it predates the landing, ask the user to restart it through `/sase_questions`.
   - Use `/sase_monitor` to wait out capture windows.
2. **Busy-hour capture.** Run `tools/tui_freeze_report` over a busy hour on one
   `app_instance_id` that starts at least 20 minutes after the restart, then over a
   4-hour RSS window. Baselines from the research:
   - 10.7–12.9% frozen by the watchdog;
   - gen-2 at 16.7% and all GC at 25.5% of wall time;
   - gen-2 p50 2.16 s;
   - RSS 1.48 → 4.66 GB plus 1.46 GB swap in 50 minutes.
3. **Targets:**

   | Target                                                                                                              | Measured by                                          |
   | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
   | Zero automatic or backstop gen-2 collections start within 2 s of input. Idle collections p95 ≤ 500 ms.              | `tui_gc_pause` rows                                  |
   | Total GC < 3% of wall time on a busy Agents tab                                                                     | heartbeat totals                                     |
   | < 0.5% of wall time in hitches over the busy hour. No `late` hitches except those attributed to an idle collection. | watchdog fields from `watchdog-truth`                |
   | RSS grows < 0.5 GB over 4 h after warm-up, with `VmSwap` about 0 (note host swap pressure if it is not)             | heartbeat                                            |
   | Structural cache tests pass in `just check`                                                                         | tests                                                |
   | j/k p95 < 16 ms, no regression                                                                                      | `SASE_TUI_PERF=1` or `tests/ace/tui/bench_tui_jk.py` |

4. **Record the results** as a bead note with the before/after table and an evidence
   artifact.
5. **Follow-ups.** Record each miss as a `PROPOSED FOLLOW-UP:` note on this bead, not as
   new beads. Always include one follow-up proposing a `tui_perf.md` memory update with
   three rules:
   - module snapshot caches hold one live version per path or scope, never a
     count-capped history of `(mtime, size)` keys;
   - collector tuning lives only in `gc_policy.py`;
   - in freeze forensics, `late` hitches mean whole-process stops (usually GC); check
     `tui_gc_pause` before blaming the stack the watchdog photographed.

   Also propose a 24-hour soak if one did not fit in the turn budget.

## Out of scope

- **Rewriting `LinkIndex` incrementally or moving it into `sase_core`.** That would be
  the structural fix for both heap size and promotion rate, but it crosses the Rust
  boundary (wire, binding, and `sase-core-revision.txt` pin). Do it as a follow-up only
  if acceptance shows idle collections or RSS still over target.
- **Sharing one parse of `index.jsonl` between the completion catalog and
  `read_artifact_file_index`, and tail-parsing appends.** This is a fixed 40 MB
  duplication, not growth. It is a follow-up candidate.
- **Narrowing which in-flight poll markers dirty a row**, and the 4–6 s artifact-delta
  loads. Once the collector stops amplifying them, re-measure them in `acceptance`.
- **Surfacing GC share in the Admin Center Perf view.** That needs a `sase-core`
  perf-log aggregator change.
- **Tuning `threshold0`.** Only after `acceptance` data shows young-generation pauses
  matter.
- **The prompt-key latency epic, and the `sase-vc` / `sase-zn` blockers.**

## Risks and guardrails

- **Frozen-object retention.** `gc.freeze()` happens once, after startup settles.
  Objects that later become cyclic garbage stay until exit; that is a bounded cost, and
  it is the reason for never re-freezing. RSS heartbeats make any surprise visible.
- **Long idle collections.** If `snapshot-caches` has not landed, an idle collection on
  a bloated heap can still take seconds, and a key pressed mid-collection waits. That is
  why `idle-gc-policy` depends on `snapshot-caches`, and why the trigger tag and overlap
  attribution exist.
- **Deferred garbage.** Each full collection reclaims only 48k–80k objects today, so
  deferring them costs little memory. The RSS backstop bounds the worst case.
- **Test-suite contamination.** Every global GC change is uninstallable and never
  auto-installed under the testing harness. Tests opt in explicitly and restore state.
- **Immutability changes in `cached-snapshot-sharing`** can break a caller that mutates
  in place. The audit and the guard tests are mandatory, not optional.
