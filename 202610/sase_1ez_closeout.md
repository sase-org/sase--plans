---
tier: tale
size: medium
title:
  Finish and close epic sase-1ez (config-token revalidator race, heartbeat window,
  tribe-evidence cache)
goal:
  Concurrent config-token reads start exactly one revalidator thread. That thread stays
  idle instead of polling at 20 Hz, and the test drain never joins an unstarted thread.
  The tui_memory_heartbeat window_s is the span since the previous heartbeat. The
  agent_tribe_evidence cache holds one live version per key. Epic sase-1ez is closed
  with a clean symvision whitelist, and its plan file is marked done.
proposed_by: bbugyi200.athena.sase-1ez.land
bead: sase-1ez
create_time: 2026-10-02 22:41:03
status: wip
---

- **PARENT:**
  [202610/tui_freeze_gc_heap.md](https://github.com/sase-org/sase--plans/blob/main/202610/tui_freeze_gc_heap.md)
- **BEAD:**
  [sase-1ez](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ez/README.md)

# Finish and close epic sase-1ez

Epic `sase-1ez` ("Stop the ACE TUI's 10% freeze budget",
`plan:202610/tui_freeze_gc_heap.md`) has all 8 phase beads closed, and the land agent
verified each phase against the source. That review turned up three defects the epic
itself introduced. This tale fixes them and then lands the epic. Nothing here crosses
the Rust boundary: the work is Python module state, test fixtures, and one docs line.

Read `lint_and_test.md` (via `/sase_memory_read`) before verifying. Do not run
`just check-full`.

## 1. Config-token revalidator: single-flight race, dead wake event, 50 ms polling

`src/sase/config/core.py`, added by phase `sase-1ez.7` (commit f76efbe6b9), replaced the
per-expiry refresh thread with one long-lived revalidator. It has three defects.

**a. Duplicate revalidators leak forever (production bug).**
`_ensure_config_token_revalidator_locked()` registers the new `threading.Thread` in
`_current_config_token_refresh_thread` under the cache lock. `current_config_token()`
then calls `.start()` after releasing the lock. The test
`test_config_token_refresh_thread_starts_after_lock_release` pins that
start-outside-lock contract, so keep it. The single-flight check is
`live is not None and live.is_alive()`, and `is_alive()` is `False` until `start()`
runs. So a concurrent first reader sees the registered-but-unstarted thread as dead,
builds a second revalidator, and overwrites the module's thread/stop/wake pointers. The
first thread still starts, but nothing can ever stop it.

The land agent reproduced this at master 54427ed47c: 8 threads behind a
`threading.Barrier` each calling `current_config_token()` right after a cache reset. In
74 of 100 trials more than one revalidator started, leaving 131 orphaned live
`sase-config-token-refresh` threads. Each orphan polls at 20 Hz and recomputes the token
every 5 s for the life of the process.

- **Fix:** treat a registered thread that has not started yet (`thread.ident is None`)
  as live in the single-flight check, so only the caller that registered it starts it.
  Keep the existing deregistration on a failed `start()`.
- **Regression test** in `tests/test_config_cache_token.py`: N reader threads behind a
  barrier hit a freshly reset cache. Exactly one thread named
  `CONFIG_TOKEN_REFRESH_THREAD_NAME` is started and alive afterwards, and it is the
  registered one. Loop enough iterations to make the race likely.

**b. The wake event is dead, and the loop polls at 20 Hz.** In
`_config_token_revalidator_loop`, every wait is `stop.wait(...)` with a 50 ms cap
(`_CONFIG_TOKEN_REVALIDATOR_POLL_SECONDS`). The getter's `wake.set()` on expiry is never
awaited; the loop only ever calls `wake.clear()`. The comment "A getter signal wakes us
early" is false. Every process that has ever read the token keeps a thread waking 20
times a second.

- **Fix:** have the loop block on the `wake` event, with no sub-second polling while
  idle.
  - Stopping must set `wake` as well as `stop`; the test drain in
    `tests/_conftest_runtime.py` already sets both. Production stop paths, if any, must
    too.
  - The loop must re-check `stop` after every wake.
  - Choose either refresh-on-cadence (`wake.wait(timeout=<time to deadline>)`) or lazy
    stale-while-revalidate, where only an expired getter read triggers a recompute.
    Either way the thread is effectively idle when nothing reads the token.
- Keep the existing fake-clock tests passing: they patch
  `sase.config.core.time.monotonic`, advance it, and then call the getter, which signals
  `wake`. Keep the epoch/cwd publish guard (`_publish_revalidator_token`) and the
  `clear_config_cache()` semantics unchanged.
- Delete `_CONFIG_TOKEN_REVALIDATOR_POLL_SECONDS` if nothing still needs it, and fix the
  stale comments.

**c. The test drain joins an unstarted thread.** `_drain_config_token_refresh` in
`tests/_conftest_runtime.py` reads the registered thread under the lock and calls
`thread.join()`. If it catches the window between registration and `start()`, the join
raises `RuntimeError: cannot join thread before it is started`. That error surfaced as
teardown ERRORs on unrelated TUI and keymap tests.

- **Fix:** after setting stop and wake, wait (bounded by the existing timeout) for
  `thread.ident` to become non-`None` before joining. Alternatively, treat a
  never-started thread as already drained, but only once the registering caller can no
  longer start it.
- Keep the "timed-out join leaves a live worker registered and raises" contract that
  `test_drain_timeout_leaves_live_worker_registered` pins.

**Evidence.** Phase `sase-1ez.7`'s full `just check` (ToolRun
`81b29f848d4646864b4f5eb4b28b247f`, replay with `sase tool show <id> -l`) failed under
load in two ways:

- 8 teardown ERRORs, all `cannot join thread before it is started`, in
  `test_build_app_bindings_count`, `test_agents_pane_inserted_first_with_files_last`,
  `test_marked_commits_copy_in_visual_order_with_labeled_sections`,
  `test_degraded_tab_uses_warning_icon_in_artifacts_strip`,
  `test_subtab_strip_labels_and_accents_cover_all_panes`,
  `test_card_block_bindings_share_paren_keys_with_files_versions_in_order`,
  `test_action_double_dollar_follows_first_link_and_records_origin`, and
  `test_agents_tab_fallback_reaches_reveal_outcome`;
- 5 config-cache failures on one xdist worker, consistent with an orphaned revalidator
  still running: `test_victim_first_reads_use_only_successor_patched_paths` and
  `test_no_live_refresh_worker_after_drain_window` (a live thread still present),
  `test_drain_timeout_leaves_live_worker_registered` ("DID NOT RAISE"),
  `test_prior_refresh_worker_cannot_publish_after_drain`
  (`('inline', 4) == ('inline', 3)`, an extra compute), and
  `test_current_config_token_refresh_is_single_flight`.

**Stability check.** Rerun `tests/test_config_cache*.py` repeatedly, both serially and
under parallel `pytest -n` with other suites competing for CPU, until those 5 nodes are
stable. Also rerun a handful of the 8 ERROR nodes together with the config-cache files.
Keep the `sase-t6` thread-naming contract (`CONFIG_TOKEN_REFRESH_THREAD_NAME`) intact.

## 2. Heartbeat `window_s` must be the span since the previous heartbeat

The plan required each `tui_memory_heartbeat` to carry per-generation GC totals "since
the last heartbeat, and the wall-clock span they cover, so GC share is exact".
`GCTelemetry._write_heartbeat` in `src/sase/ace/tui/util/gc_telemetry.py` resets the
per-generation totals at every heartbeat. But it sets
`window_s = round(now - self._install_mono, 6)`, which is identical to `uptime_s`. After
the first heartbeat, `total_s / window_s` therefore understates GC share.

The watchdog's heartbeat provider totals in `_stall_watchdog_monitor.py`
(`loop_hitch_seconds` and the rest) cover the same since-last-heartbeat window.

- **Fix:** track the previous heartbeat's monotonic time, initialized to install time.
  Emit `window_s = now − previous` and keep `uptime_s` as time since install.
- **Test** in `tests/ace/tui/util/test_gc_telemetry.py`: with the fake clock, two
  heartbeats 300 s apart and then one 120 s later give `window_s` 300 then 120, while
  `uptime_s` reads 300 then 420.
- **Docs:** state the `window_s` meaning in the "GC pause and memory heartbeat rows"
  section of `docs/perf_runbook.md`.
- `tools/tui_freeze_report` derives GC share from its own `--since/--until` window and
  needs no change. Confirm `tests/test_tui_freeze_report_tool.py` still passes.

## 3. `agent_tribe_evidence` cache: one live version per key

Phase `sase-1ez.3`'s audit was told to fix any other module cache keyed on a file
version whose superseded versions are never read again. It flagged
`src/sase/core/agent_tribe_evidence.py` but left it alone.

`stored_tribe_names_for_resolution` keys the unbounded module dict `_CACHE` on
`(sase_home, projects, tribes_path, _stat_token(tribes_path), legacy_path, _stat_token(legacy_path), _CACHE_GENERATION)`.
Every out-of-process write to the tribe store mints a new key and pins the superseded
entry for the life of the process. In-process writers call
`invalidate_agent_tribe_evidence_cache()`, which clears the cache, but other processes
do not.

- **Fix:** key by the path identity
  `(str(sase_home()), str(projects), str(tribes_path), str(legacy_path))` and store
  `((tribes_stat, legacy_stat, _CACHE_GENERATION), tribe_names)`. A version mismatch
  rebuilds and replaces that key's entry. Bound the number of keys with a small LRU (for
  example 8), the same pattern as `relations/artifact_links._CACHE`.
- Keep `invalidate_agent_tribe_evidence_cache()` behavior and the
  `_merge_tribe_names(..., extra_tribes)` result.
- **Structural test** in the style of `tests/ace/tui/test_relation_cache_bounds.py`:
  rewrite the tribe store file N times externally (changing mtime and size), reading
  after each rewrite. The cache must hold exactly one entry for that key, and the last
  read must reflect the last write.

## 4. Verify

Run `just check` through `sase tool run` (per `lint_and_test.md`). Any failure must be
shown to be unrelated, on a clean base tree, before you proceed. The focused suites are:

- `tests/test_config_cache*.py`;
- `tests/ace/tui/util/test_gc_telemetry.py`, `test_stall_watchdog.py`, and
  `test_gc_policy.py`;
- `tests/test_tui_freeze_report_tool.py`;
- the new tribe-evidence cache test;
- `tests/ace/tui/test_relation_cache_bounds.py`.

## 5. Close out epic `sase-1ez`

This is the final step of the landing and runs in this same turn. Nothing resumes the
landing afterwards.

1. Run `sase bead epic-symbols sase-1ez`. For each listed `--epic-symbol` entry in the
   `Justfile`, resolve the symbol: wire it up, privatize it, add a non-test pragma, or
   delete it, following the Symvision epic-whitelist policy in `symvision.md`. The land
   agent found none, so this should be empty unless this tale added one; do not add any.
2. Close the epic with `sase bead close sase-1ez --note "<verification>"`. The note must
   summarize:
   - all 8 phases verified against source (gc-telemetry, watchdog-truth,
     snapshot-caches, idle-gc-policy, cached-snapshot-sharing, tick-compare-skip,
     off-loop-refresh, acceptance with its live capture deferred to `sase-1f3`);
   - integration since the epic started was reviewed: `sase-1ex.10`'s
     `mounted_prompt_bar` accessor builds on the epic's `_active_prompt_bar`, and no new
     module snapshot cache, `gc.*` tuning, or prompt-bar DOM query was added;
   - follow-up triage: tasks `sase-1f0`, `sase-1f1`, `sase-1f2`, `sase-1f3`, `sase-1f4`;
     a +1 on `sase-1ak`; DISCOVERED ISSUE notes on `sase-1eq` and `sase-1es`;
   - the three fixes made in this tale, with their test evidence.

   If the close is rejected for leftover `--epic-symbol` entries, finish that cleanup
   and close again. Never use `--force` merely to make the close succeed.

3. Run `just symvision` through `sase tool run` and confirm the whitelist is clean.
4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
   file, the PLAN path that `sase bead read sase-1ez -r "<why>"` shows for
   `plan:202610/tui_freeze_gc_heap.md`. Change nothing else in that file.
5. `sase-1ez` has no `parent_bead`, so the landing ends there.
