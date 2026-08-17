---
tier: tale
title: Close the last two sase-mv config-cache full-lane flakes
goal:
  tests/test_config_cache.py::test_load_merged_config_caches_plugin_layer and
  ::test_selector_change_eventually_invalidates_merged_config stop failing under the
  full parallel pytest lane, with the ambient config reader that breaks them either
  removed or proven harmless, and a regression guard that catches the next one.
size: medium
proposed_by: bbugyi200.athena.sase-mv
bead: sase-mv
create_time: 2026-08-17 09:13:01
status: wip
---

- **BEAD:**
  [sase-mv](https://github.com/sase-org/sase--beads/blob/main/pages/sase-mv/README.md)

# Plan: Close the last two sase-mv config-cache full-lane flakes

## Context

Task bead `sase-mv` tracks the class "config-cache tests that fail only under the whole
suite run in parallel and pass in isolation". Epic `sase-ns` phase `sase-ns.2` (commit
`3a22ff04f`) bound the memoized config token to the `CONFIG_DIR` object it was computed
against and made the autouse `_clear_config_caches` fixture a yield fixture that drains
the `sase-config-token-refresh` worker. That retired nine sibling nodes. The bead was
closed and then reopened, because two nodes still fail on trees that carry the fix:

- `tests/test_config_cache.py::test_load_merged_config_caches_plugin_layer` failed at
  HEAD `f9ab15d9c` with `call_count["n"] == 0` instead of `1` (monitored `check-full`
  `92836jkgezbw`, 2026-08-17).
- `tests/test_config_cache.py::test_selector_change_eventually_invalidates_merged_config`
  failed at HEAD `b6246f1cf` on `assert load_merged_config() is first` (monitored
  `check-full` `swk45sjycf4e`, 1 failed / 31845 passed / 11 skipped in 680.50s).

Both passed on an immediate focused rerun. Per the reporting land agent, these two plus
one `sase-n4`-owned usage-limit node are what currently hold
`just selection-health --fail-on-new-flake` red, so this class directly gates landing.

## Why a concurrent reader is required (the diagnosis this plan starts from)

Walking both node timelines against `src/sase/config/core.py` shows neither failure is
reachable from a single-threaded test body, which narrows the search before any run:

**`test_load_merged_config_caches_plugin_layer` (`call_count["n"] == 0`).** The autouse
`_clear_config_caches` setup calls `clear_config_cache()`, which sets
`_plugin_configs_cache = None`. `load_merged_config()` only calls `_load_plugin_configs`
when that module global is `None`. A count of zero therefore means the process-static
plugin layer (and, transitively, `_merged_config_cache_value`) was already warm when the
test made its first call — something read config after the fixture cleared and before
the test body ran. The autouse chain that runs between those two points is fixed, and
the node passes in isolation, so the warming actor is not on the main thread.

**`test_selector_change_eventually_invalidates_merged_config`
(`assert load_merged_config() is first`).** With `time.monotonic` pinned, the first
`load_merged_config()` computes the token inline and sets the deadline; no refresh
worker exists yet. The stale assertion only fails if `_merged_config_cache_token` moved
to the post-selector-change token before the main thread's comparison. The main thread
compares against its own local token, and the worker it starts publishes only the token,
never the merged value — so some other thread had to call `load_merged_config()` (or
`clear_config_cache()`) inside that window.

This also explains the shape recorded on the bead: victims cluster on one xdist worker,
the failing subset differs run to run on an identical tree, and file-scoped contention
never reproduces it. A live poller in one worker process poisons whichever config test
happens to be running when it ticks.

Static candidates worth naming for the run below, all daemon threads started by ACE TUI
code paths and all reaching merged config: `sase-ace-proc-observer`
(`src/sase/ace/tui/proc_observer.py`, `POLL_SECONDS = 0.5`, whose `_build_snapshot()`
reaches `humanize_cl_name` and `sase.core.time.to_local`), `ace-fs-watcher`,
`sase-tui-stall-watchdog`, `sase-tui-toast-log-writer`, and `sase-telemetry-flusher`.
`sase.core.time.get_timezone()` falls back to `load_merged_config()` whenever
`_cached_timezone` is `None`, and the autouse `_pin_configured_timezone` fixture sets it
to `None` on teardown, so the between-tests window is exactly when an ambient tick is
most dangerous. These are candidates, not conclusions: step 1 decides.

## Non-goals

- Changing production caching semantics in `src/sase/config/core.py`. The stale-while-
  revalidate contract and the `CONFIG_DIR` binding added by `sase-ns.2` stay as they
  are.
- Adding any node to `tests/reproducible_flake_baseline.txt`. These nodes are the
  defect, not accepted noise.
- Re-fixing the nine sibling nodes `sase-ns.2` already retired, or the `sase-n4`-owned
  usage-limit node that also holds `selection-health` red.

## Step 1 — Identify the ambient reader with an instrumented full-lane run

Add an opt-in pytest plugin `tests/_config_reader_probe.py`, registered from
`tests/conftest.py` behind a new `--sase-detect-config-readers` flag (mirror how
`register_global_state_leak_detector` is wired in
`tests/_global_state_leak_detector.py`, including its per-xdist-worker JSON payload
handling so the controller can merge results). The plugin must:

1. Wrap the `sase.config.core` module attributes `load_merged_config`,
   `current_config_token`, and `clear_config_cache` at `pytest_configure` time and
   record every call made from a thread that is neither `threading.main_thread()` nor
   the `sase-config-token-refresh` worker, capturing the reader thread name, the nodeid
   that was running, a call count, and one captured stack.
2. After each test, enumerate live non-main threads and record any thread that was first
   seen during an earlier test, keyed by `(thread name, originating nodeid)` with the
   later victim nodeids. Track `sase-config-token-refresh` in its own bucket rather than
   ignoring it, so a drain that timed out is visible too.
3. Write one JSON report (default `.pytest_cache/sase-config-readers.json`) and print a
   terminal summary.

Then run the full fast lane with the probe enabled through `/sase_monitor` (this lane
routinely outruns one agent turn):

```bash
sase monitor start \
  --command 'just test -- -p tests._config_reader_probe --sase-detect-config-readers' \
  --reason 'Identify the ambient config reader behind the sase-mv full-lane flakes' \
  --timeout 40m \
  --next 'Read .pytest_cache/sase-config-readers.json, then continue plan step 2.'
```

A prototype of this probe has already been run as a scratch plugin over
`tests/test_config_cache.py` plus three ACE TUI proc suites serially: 47 passed, zero
cross-test live threads, and the only non-main config reads were the reader threads
`test_current_config_token_refresh_is_single_flight` starts itself. That subset is too
small to reproduce the class, which is why step 1 runs the whole lane.

## Step 2 — Fix what step 1 finds

Act on the report, in this order of preference:

1. **A leaked thread with a clear owner** (for example a `ProcObserver` started by
   `_init_proc_observer` whose `_stop_proc_observer` never ran for some test path): stop
   it where it is started. Prefer a fixture or lifecycle fix in the owning test module
   or in `tests/ace/tui/conftest.py` over changing production code; change production
   code only if a component genuinely starts a thread with no lifecycle owner.
2. **A drained-but-still-alive `sase-config-token-refresh` worker**:
   `_drain_config_token_refresh` joins with a 2.0s timeout and then unconditionally
   clears `config_core._current_config_token_refresh_thread`, which under load can null
   out a _different_, live worker registered by a later test and break single flight.
   Make the drain leave that reference alone unless the thread it observed actually
   exited, and raise or report when the join times out instead of silently continuing.
3. **A reader with no fixable owner within this plan's scope**: record the finding on
   bead `sase-mv`, file a sized follow-up task via `/sase_new_task`, and rely on step 3
   for the bead's own exit criteria. Do not silently skip this branch — say which branch
   was taken in the bead close note.

## Step 3 — Make the two named assertions test their real contract

Independently of step 2, remove the ambient-reader sensitivity from the assertions. Four
tests in `tests/test_config_cache.py` share the fragile
`assert load_merged_config() is first` stale-serve check inside a body whose real
subject is something else:
`test_load_merged_config_eventually_invalidates_on_file_mtime_change` (line 100),
`test_selector_change_eventually_invalidates_merged_config` (line 251),
`test_load_merged_config_caches_default_layer` (line 316), and
`test_load_merged_config_caches_plugin_layer` (line 350).

1. Drop that in-situ stale-serve assertion from all four. Each test keeps its own
   subject: eventual invalidation on mtime change, eventual invalidation on selector
   change, and single-load reuse of the default and plugin layers.
2. Replace the lost coverage with one dedicated, deterministic test that pins
   `_compute_current_config_token` to a token derived from a test-controlled variable
   (not a `side_effect` list, so an ambient read cannot consume a queued value) and
   asserts `load_merged_config()` returns the same merged object while a refresh is in
   flight. Model it on `test_current_config_token_serves_stale_while_refreshing`, which
   already covers the token-level half of this contract deterministically.
3. Convert the layer-caching counters from an absolute count to a delta across the token
   change: record `call_count["n"]` immediately after the first `load_merged_config()`,
   and after `_wait_for_new_merged_config(first)` assert the count is unchanged and that
   the first read loaded the layer at most once. Once the layer cache is warm, only
   `clear_config_cache()` can cool it, so the delta is deterministic even if an ambient
   reader warmed it first — which is exactly the `call_count["n"] == 0` failure mode.

## Step 4 — Regression guard

Keep the step 1 probe as a permanent opt-in tool rather than deleting it, and give it a
gate: a test that asserts a non-main thread reading `sase.config.core` during another
test is reported as poisoning. If step 2 found and fixed a specific leak, add a focused
regression test in the owning module that fails if that thread outlives its test, in the
style of `tests/test_config_cache_isolation.py`.

Add the probe's own unit coverage (wrap-and-record, cross-test thread attribution, JSON
payload merge) next to `tests/test_global_state_leak_detector.py`.

## Step 5 — Verification

1. `just check` inline for the fast gates.
2. Through `/sase_monitor`: `just check-full`. Both named nodes plus every node in
   `tests/test_config.py`, `tests/test_config_cache.py`, and
   `tests/test_config_cache_isolation.py` must be green.
3. Through `/sase_monitor`: repeat the full lane a second time on the same tree. One
   green lane is weak evidence for a class whose failing subset is nondeterministic; two
   consecutive green lanes is the bar this bead has been reopened for failing to meet.
4. `just selection-health --fail-on-new-flake` and record the result honestly. Per the
   note on `sase-mv`, that gate requires an interleaved pass between two recorded
   failures, so historical pre-fix records can keep naming these node IDs; that gate
   limitation is task bead `sase-nv` and must not be presented as evidence the defect
   survives — nor may a still-red gate be presented as fixed.
5. Close `sase-mv` with a note naming: which branch of step 2 was taken, the two
   full-lane run IDs, and the `selection-health` state.

## Risks

- **Step 1 finds nothing.** The probe only sees reads that go through the
  `sase.config.core` module attributes; a caller that bound `load_merged_config` at
  import time bypasses the wrapper. If the report is empty, re-run with the wrap moved
  to `sase.config.core._compute_current_config_token` and `merge_config_sources` before
  concluding there is no ambient reader.
- **The probe perturbs the race.** Wrapping adds work on a hot path and may make the
  class rarer. Steps 3 and 4 do not depend on step 1 reproducing, which is why step 3 is
  written to stand on its own.
- **Step 3 reads as weakening tests.** It is not: each assertion is replaced by one that
  covers the same contract without depending on process-global state the test does not
  own, plus a new dedicated test for the stale-serve behavior. The bead's own scope note
  sanctions "make the assertion robust" as an acceptable fix shape.
