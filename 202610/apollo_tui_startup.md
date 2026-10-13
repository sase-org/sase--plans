---
tier: epic
title: Stop sase TUI startup starvation on loaded hosts
goal: 'On a loaded host such as apollo, the Agents roster is usable soon after first
  paint. Hidden-surface and maintenance work no longer competes with the Agents loader
  through the GIL. The verified CPU hogs (plugin rescans, the Patch relation treadmill,
  and the app-wide link walk) are fixed. Startup cliffs can be diagnosed from telemetry
  alone.

  '
decisions:
  fallback_seconds:
    ask: How long should the startup safety timeout wait for Agents before releasing
      hidden work?
    choices:
      sec_30: A stuck Agents load delays hidden work 30 s; very slow loads may still
        release early
      sec_45: A stuck Agents load delays hidden work 45 s; covers today's slow apollo
        loads
      sec_60: A stuck Agents load delays hidden work a full minute; rarely releases
        mid-load
    default: sec_45
    why: Leaves headroom over apollo's 11–14 s post-fix loads and still bounds a stuck
      load
    answer: sec_45
phases:
- id: plugin-discovery-memo
  title: Memoize macro-plugin entry-point discovery
  depends_on: []
  size: small
  description: 'plugin-discovery-memo: memoize macro-plugin entry-point discovery
    per process, keyed by a cheap install signature, so get_plugin_input_type_registry
    stops rescanning every entry point on every macro input it parses.'
- id: toolrun-glance-coalesce
  title: Coalesce deferred ToolRun glance refreshes
  depends_on: []
  size: small
  description: 'toolrun-glance-coalesce: keep the ToolRun refresh marked scheduled
    while it waits behind an Agents load, so only one deferred chain and one glance
    load can exist.'
- id: loader-path-trims
  title: Trim self-inflicted Agents-loader critical-path costs
  depends_on: []
  size: small
  description: 'loader-path-trims: read the dismissed index without write transactions
    (which fixes its never-hitting cache), skip the repro-capture and update-receipt
    imports unless needed, and resolve artifact dirs only for running agents.'
- id: tool-wait-polling
  title: Back off sase tool wait and join polling
  depends_on: []
  size: small
  description: 'tool-wait-polling: carry poll backoff across follow slices, throttle
    the store-wide reconcile inside waiters, and skip re-ingesting unchanged event
    files.'
- id: patch-relation-index
  title: Stop the Patch relation-index treadmill
  depends_on: []
  size: medium
  description: 'patch-relation-index: memoize the Patch relation index, build its
    edges only when the Patches pane is visible (keeping a cheap targets-only index
    for link-follow), take the startup build off the event loop, and debounce visible
    reloads adaptively.'
- id: startup-readiness-gate
  title: Gate hidden startup work on Agents readiness
  depends_on:
  - patch-relation-index
  size: medium
  description: 'startup-readiness-gate: replace the 3 s release with a long safety
    timeout, keep auto-refresh ticks from collecting axe or reloading before release
    (fixing the skipped axe auto-start), and run heavy maintenance jobs one at a time.'
- id: startup-telemetry
  title: Make startup cliffs diagnosable from telemetry
  depends_on:
  - loader-path-trims
  - patch-relation-index
  - startup-readiness-gate
  size: small
  description: 'startup-telemetry: always-on loader disk substage timings, plus release
    reason and time, host load, and relation-index build counts on tui_startup rows.'
- id: core-link-index
  title: Move the artifact-link walks into sase-core
  depends_on:
  - patch-relation-index
  - startup-readiness-gate
  size: large
  description: 'core-link-index: replace the Python per-row walks behind the app-wide
    LinkIndex and the Patch relation projection with a sase-core inverted index memoized
    by the aggregate signature, with single-flight Python adapters and a sase-core
    pin move.'
proposed_by: bbugyi200.athena.102
decided_by: reviewer
decided_via: tui
create_time: 2026-10-13 11:47:14
status: wip
bead_id: sase-1kj
---

- **PROMPT:** [prompts/202610/apollo_tui_startup.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/apollo_tui_startup.md)
- **BEAD:** [sase-1kj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1kj/README.md)

# Stop sase TUI startup starvation on loaded hosts

## Background

The research report
`research:202610/apollo_tui_startup_slowness/apollo_tui_startup_slowness.md` profiled
`sase tui` on apollo (16-vCPU 2017 Xeon droplet, system CPython 3.12.3, load 13–31 from
agent pytest-xdist runs). First paint is fine, but the Agents roster took 52–79 s in the
user's sessions (this week's p90 is 55 s; the all-time p50 is about 6 s).

The report's central finding holds up. The Agents loader takes 1.5–3.9 s when run alone,
but it is starved inside the TUI. CPython drops the GIL around every SQLite row step and
every `stat`, and has to win it back each time. When CPU-bound Python threads in the
same process are running, each win-back can wait up to the 5 ms switch interval. The TUI
starts those competing threads itself. The lead researcher tried three process-local
monkeypatches on apollo: (a) memoized plugin discovery, (b) no Patch relation index, and
(c) holding deferred work until Agents is ready. Together they cut mean Agents-ready
from 42.2 s to 12.0 s at load 21–29, and the variance disappeared.

## Verification of the report's recommendations

Every recommendation was checked against the current tree before this plan was written.
A recommendation is planned only if it is correct and desirable.

| #   | Report recommendation                        | Verdict                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Disposition                                     |
| --- | -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| 1   | Stop the Patch relation-index treadmill      | **Correct.** `_build_patch_relation_index` (`src/sase/ace/tui/actions/patch/_loading.py`) has no memo. The startup path `_run_deferred_mount_state_loads` → `_apply_patches` builds it on the event loop. The watcher reloads Patches on every project-dir or beads change, whatever the visible tab, and every artifact-link write rewrites `projects/<key>/artifact-links.json` inside the project dir. Watcher reloads build in a worker thread, not on the loop, but they still hold the GIL.                                                                                                                                                                                              | `patch-relation-index`                          |
| 2   | Gate hidden startup work on Agents readiness | **Correct, plus a new hazard.** Release already happens at visible-ready, but the 3 s fallback almost always wins on apollo. The auto-refresh tick is armed at first paint and has no startup guard. Its first run is a full sanity refresh (`_last_full_sanity_refresh = 0.0`) that collects axe with full snapshots. If release ever slips past that first 10 s tick, the tick's axe load sets `_axe_first_load_done`, and the startup axe init (auto-start, `--restart-axe`) is silently skipped. So suppressing the tick is mandatory once the fallback grows.                                                                                                                             | `startup-readiness-gate`                        |
| 3   | Fix plugin-discovery rescans                 | **Correct.** `get_plugin_input_type_registry` computes its cache key by re-running discovery. Reproduced on athena: `build_prompt_catalog_snapshot(generation=0, projects=())` made 446 `entry_points()` calls in 3.86 s; with discovery memoized it made 6 calls in 0.56 s.                                                                                                                                                                                                                                                                                                                                                                                                                   | `plugin-discovery-memo`                         |
| 4   | Give interactive work headroom on apollo     | Ops, no code. The suite-gate pool formula is confirmed. The gate's 700 MiB-per-worker figure cites a measured 489 MiB peak, and the report's 1.1–1.4 GB observation is unconfirmed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Out of scope (user-run ops)                     |
| 5   | Make the 44.7k-row link walks cheap          | **Correct.** The app-wide `LinkIndex` makes about 3 binding calls per raw row. The Patch projection re-walks the same rows. Both share 8192-entry ref caches that miss on about 36k unique refs. No bulk or inverted-index API exists in sase-core today. `link_index_for_snapshot` also checks its cache outside the lock, so two callers can build concurrently.                                                                                                                                                                                                                                                                                                                             | `core-link-index`                               |
| 6   | Fix ToolRun glance coalescing                | **Correct, exactly as described.**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | `toolrun-glance-coalesce`                       |
| 7   | Take convoy-prone loops off the loader path  | **Partly.** The dismissed-index read is worse than reported. `query_summary_identities` opens the index with `create=True`, and its `INSERT OR REPLACE` of the schema version changes `index.sqlite`'s mtime. That mtime is part of the signature `dismissed_bundle_identities_snapshot` caches on, so the cache never hits (reproduced on a temp index). The `AgentIdentitySnapshot` realpath "storm" is already `lru_cache`d per process, so that claim is mostly false. Per-record commit/revert probes are real, but removing them needs new sase-core scan-wire fields. New finding: `agent.get_artifacts_dir()` runs for every agent before the RUNNING filter in `_loading_helpers.py`. | `loader-path-trims`; per-record probes deferred |
| 8   | Trim imports                                 | **Partly.** The `repro.capture` import (on the loop; about 293 modules on first load) and the post-update-toast import (61 modules) are cheap, safe wins. Deferring `_agent_enter_action` or `_metadata_search` is infeasible because both are `AceApp` mixins. `agent_detail` and `prompt_panel` render at mount. Moving apollo to Python 3.14 is a human-only global install.                                                                                                                                                                                                                                                                                                                | `loader-path-trims`; the rest rejected          |
| 9   | Make cliffs diagnosable                      | **Partly.** The loader `disk` stage is one aggregate, and the release reason is stored but never recorded. The footer stopwatch already stops at visible-ready, so that claim is false. `interpreter_cli_import_seconds` was fixed in 55eec1b986; only a sanity guard remains.                                                                                                                                                                                                                                                                                                                                                                                                                 | `startup-telemetry`                             |
| 10  | Steady-state slowness                        | **Mixed.** The inotify claim is false: watches are shallow and capped, and hitting the budget does not escalate to full loads. `sase tool wait` polling is worse than reported: `follow_run` restarts the backoff every 0.5 s slice, so a waiter makes about 9 store-wide reconciles per second, each re-ingesting every unsettled run's events from offset 0. A GC policy already exists (`gc_policy.py`). Gateway containment is unverified.                                                                                                                                                                                                                                                 | `tool-wait-polling`; the rest out of scope      |
| 11  | Faster droplet                               | Ops.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Out of scope                                    |

## Rules for every phase

- Before changing TUI code, read `sase memory read tui_perf.md -r "<why>"` and follow
  it. Rule 2: off the event loop is not off the pump; use `spawn_pump_free_task()` and
  keep the scheduled/running/pending guards. Rule 4: re-capture UI state after every
  await. Rule 5: route refreshes through the existing paths. Rule 9: add no work before
  the startup stopwatch ends. Rule 13: respect activity gates.
- Shared backend behavior belongs in the linked `sase-core` repo (core memory
  `rust_core_backend_boundary`). Only `core-link-index` crosses that boundary. The other
  phases change TUI glue, process-local caches, or Python-only modules (the dismissed
  index and plugin discovery have no Rust counterpart).
- Measure with the report's method: time isolated costs with `perf_counter` and
  `thread_time`, never under a profiler. Record before/after numbers in the phase's
  close note, with the host's load at the time.
- Verify with `sase tool run check` per `sase/memory/lint_and_test.md`, and run
  `just fmt` first. Do not run `just check-full` unless the phase bead explicitly says
  to. Each phase adds focused regression tests in the existing test modules named below.
- Epic phase workers do not create task beads. Record out-of-scope discoveries with
  `sase bead note <phase-bead> 'PROPOSED FOLLOW-UP: ...'`.
- The prior startup epic `sase-132` (still open) designed the current startup
  coordinator, and its phase `sase-132.2` chose the "few seconds" fallback. This epic
  supersedes that value deliberately. Keep the coordinator's other invariants:
  - release exactly once;
  - never mark visible-ready from a fallback;
  - `all_surfaces_ready` must still converge.

## Phase: plugin-discovery-memo — memoize macro-plugin entry-point discovery

**Problem.** `_discovery_snapshot(None)` in `src/sase/macro/plugin_input_types.py` runs
`discover_macro_plugin_input_type_files()` and `discover_macro_plugin_distributions()`
before it checks the cache. Each of those calls `_discover_macro_plugin_entry_points()`
(`src/sase/main/plugin_discovery.py`), which scans two entry-point groups.
`_plugin_registry_snapshot()` (`src/sase/macro/_loader_parsing_inputs.py`) runs this
once for every macro input it parses. On a cache miss, `get_plugin_input_type_registry`
calls `discover_macro_plugin_distributions()` a second time. The cost hits every CLI
command that loads macros, not just the TUI.

**Change.**

1. Memoize `_discover_macro_plugin_entry_points()` per process, along with the derived
   results of `discover_macro_plugin_input_type_files()` and
   `discover_macro_plugin_distributions()`.
   - Key the memo by a cheap install signature: `tuple(sys.path)` plus `st_mtime_ns` for
     each existing `sys.path` directory. This mirrors the mtime-keyed path cache
     `importlib.metadata` already uses, so installing or uninstalling a distribution
     invalidates the memo.
   - Also key on the disable controls. Call `macro_plugins_disabled()` on every call,
     before the memo lookup, so its retired-alias collision error is preserved.
   - Return fresh list copies so no caller can mutate the cached value.
2. In `get_plugin_input_type_registry`, compute the known-distribution list once per
   call and reuse it for both the key and the binding payload.
3. Add a private cache-clear hook next to the memo. Find every test that monkeypatches
   `importlib.metadata.entry_points`, the discovery functions, or the disable env vars
   (`rg "entry_points|discover_macro_plugin|SASE_DISABLE_PLUGIN" tests`). Make those
   tests clear the memo, using a fixture in the nearest `conftest.py` if many need it.
4. Known edge, which needs a docstring note only: a dist-info edited in place without a
   directory-level change keeps the old entry points until restart. This matches how a
   running TUI already keeps its imported code (tui_perf rule 15).

**Tests.** Add tests in the existing plugin-discovery and plugin-input-type test
modules:

- Repeated `get_plugin_input_type_registry()` calls do not call `entry_points()` again.
- Changing a disable env var, or touching a `sys.path` directory's mtime, invalidates
  the memo.
- The catalog output is identical with and without the memo, compared on the snippet
  set.

**Measure.** Record the `entry_points()` call count and the thread CPU of
`build_prompt_catalog_snapshot(generation=0, projects=())` before and after. The athena
prototype went from 446 calls in 3.86 s to 6 calls in 0.56 s.

## Phase: toolrun-glance-coalesce — coalesce deferred ToolRun glance refreshes

**Problem.** `_run_tool_runs_refresh` (`src/sase/ace/tui/tool_runs/loader.py`) clears
`state.scheduled` before its `_agents_loading` check. That branch then re-arms a 50 ms
timer with both `scheduled` and `running` false. The drift probe runs at most every 2 s
from both countdown branches. During an Agents load the drift persists, because
`last_token` updates only after a load, so each probe starts another chain. When Agents
finishes, every chain sets `running = True` without checking it and runs
`load_glance_blocking` concurrently. The first chain to finish clears `running` while
the others are still in flight.

**Change.**

1. Clear `state.scheduled` only after the `_agents_loading` deferral, so it stays true
   while deferred. This is how the navigation-gate path already behaves.
2. If arming the deferral timer fails, reset `scheduled` to false. Spawn failures
   already reset it.
3. Add an entry guard: if `state.running`, set `pending` and return.

**Tests.** Add tests next to `tests/ace/tui/test_tool_runs_glance.py`:

- With `_agents_loading=True`, schedule; run the coroutine; schedule several more times.
  Assert one spawn, one live timer chain, and `scheduled` still true.
- Clear `_agents_loading`, run the timer callback, and assert exactly one
  `load_glance_blocking` call.
- A deferral-timer failure leaves the state schedulable.

**Coordination.** In-progress bead `sase-1ih.2` touches the ToolRun live glance. Rebase
onto whatever has landed and keep its fields intact.

## Phase: loader-path-trims — trim self-inflicted Agents-loader critical-path costs

1. **Dismissed index reads must not write.**
   - `query_summary_identities` (`src/sase/ace/dismissed_bundle_index/_api.py`) opens
     `connection(root)` with the default `create=True`. That path runs `_ensure_schema`
     (`_schema.py`): `BEGIN IMMEDIATE`, DDL, `INSERT OR REPLACE` of the schema version,
     and a commit.
   - The write changes `index.sqlite`'s mtime, which is part of the signature cached by
     `dismissed_bundle_identities_snapshot` (`src/sase/ace/dismissed_agents.py`). So
     every Agents load re-queries and takes the write lock, waiting up to 30 s behind a
     rebuild.
   - Open pure-read helpers with `create=False`: `query_summary_identities`, plus
     `query_summaries` if all of its callers are reads. Keep schema creation and
     migration on the write paths and `ensure_dismissed_archive_ready`.
   - A missing DB still returns `None`. A schema-less or incompatible file must return
     `None` (the query raises `sqlite3.Error`) and must never drop tables from a read.
   - Test against a real temp index, not mocks:
     - two consecutive `dismissed_bundle_identities_snapshot()` calls with no writes run
       the SQL once and leave `index_signature` unchanged;
     - a missing DB and an old-version DB behave as above. The current unit test
       (`tests/test_dismissed_bundle_index.py`) mocks both functions and cannot catch
       this.
2. **Repro-capture import only when capture is on.**
   - Find every `from ...repro.capture import ...` inside a load path:
     `actions/agents/_loading_disk_full.py` (two sites), `_loading_disk_delta.py`, and
     `_loading_apply.py` (`record_agents_tab_app_projection`).
   - Guard each with `getattr(self, "_agents_repro_capture", None) is not None`. A
     non-`None` ring implies `capture.py` is already imported, so behavior is identical.
   - Keep the tests that patch
     `sase.ace.tui.repro.capture.record_agents_tab_loader_result` green. Set the ring in
     tests that need the call.
3. **Post-update toast.**
   - `_maybe_show_post_update_toast` → `post_update_toast.py` imports
     `sase.ace.update_receipt` (about 61 modules) on the event loop before checking
     whether `pending_update_toast.json` exists.
   - Stat the receipt path first, and import only when it exists.
   - Keep one source of truth for the filename. If the constant lives in the heavy
     module, move it (with its path helper) to an import-light module that both modules
     use.
4. **Resolve artifact dirs only for running agents.**
   - In `src/sase/ace/tui/actions/agents/_loading_helpers.py`, move
     `artifacts_dir = agent.get_artifacts_dir()` below the
     `agent.status != "RUNNING" and not agent.runner_is_live` skip.
   - `get_artifacts_dir` is pure (`models/artifact_files.py`), but for snapshot-built
     done agents it walks the project-alias Rust lookup once per agent.
   - Add a test that done agents never call it.

## Phase: tool-wait-polling — back off sase tool wait and join polling

**Problem.**

- `follow_run` (`src/sase/tool/follow_run.py`) calls `wait_for_settlement`
  (`src/sase/tool/control_wait.py`) in 0.5 s slices, and each call restarts the backoff
  at 0.05 s. Steady state is therefore about 9 polls per second, and the 0.5 s cap is
  never reached.
- Every poll runs `reconcile_unsettled_tool_runs()` (`src/sase/tool/liveness.py`). It
  lists every unsettled run in the store, re-ingests each run's `events.jsonl` from
  offset 0 (`ingest_event_file` builds a fresh `StageIngestor`), and always calls
  `tool_run_reconcile`. Each of those steps commits in the Rust store.
- Both `sase tool wait` and `_join` (`join_worker.py`) use this loop. The constant
  `runs.sqlite-wal` churn also keeps the TUI's ToolRun drift probe firing.

**Change.**

1. Carry the backoff delay across `follow_run` slices. Either `wait_for_settlement`
   accepts and returns a delay, or both share a small poll-state object. Deadline and
   stop-event responsiveness must stay at 0.5 s or better.
2. Inside `wait_for_settlement`, run the store-wide reconcile on the first poll, then at
   most once every 2 s. `tool_run_show` keeps polling at the backoff cadence, so normal
   settlement latency does not change. A dead owner is still noticed within about 2 s.
3. In `reconcile_unsettled_tool_runs`, skip `ingest_event_file` when the events file's
   `(st_size, st_mtime_ns)` is unchanged since the last successful ingest of that
   `(run_id, path)` in this process. Use a bounded process-local map. Ingest is
   idempotent by event id, so skipping an unchanged file loses nothing. Keep the reaping
   (`reap_orphans=True`) executor path behaviorally identical.

**Tests.** Use a fake clock or monkeypatched `time`:

- A 3 s follow of an unsettled run makes at most 2–3 reconcile calls and polls
  `tool_run_show` at a backed-off cadence.
- An unchanged events file is not re-ingested; an appended one is.
- Settlement, deadline, and stop outcomes are unchanged.

**Follow-up only.** The Rust store's `touch_write_meta` still commits on replayed
appends and on empty reconciles. Record that as a `PROPOSED FOLLOW-UP:` for sase-core;
do not change sase-core here.

## Phase: patch-relation-index — stop the Patch relation-index treadmill

**Facts that shape the design:**

- `_all_patches` has consumers outside the Patches pane: workflow context, the runners
  and project-select modals, `agent_loader`, and navigation `_tree`. So the Patch _list_
  must stay fresh while hidden. Only the relation _edges_ are safe to defer.
- Link-follow from any tab (`$` on Agents or Services) resolves Patch targets through
  `entry_target_for_ref` (`widgets/artifacts/entry_navigation.py`), which reads
  `relation_index().known_targets`. When the index is `None` it returns no answer.
- `PatchSnapshotCache` returns the same Patch objects while a file's `(mtime_ns, size)`
  is unchanged, but a new list on every call.

**Change.**

1. **Memoize.** Wrap `_build_patch_relation_index` in a process-local, single-entry
   memo.
   - Key it on the artifact-links snapshot's `source_key` plus the Patch-set identity.
     Hold a tuple of the patch objects (strong references, so ids cannot be recycled)
     and compare it element by element by identity.
   - Query commits (`_commit_patch_query` → `_load_patches` → `_apply_patches`)
     currently rebuild the full index on the main thread, though the index does not
     depend on the query. They must become memo hits.
   - Emit the `relations.index.patches` span only for real builds.
   - Keep a module-level counter of real builds (`startup-telemetry` reads it).
2. **Build edges only when Patches is visible.**
   - Add one visibility predicate: Artifacts tab with the Patches pane selected. Reuse
     the predicate the auto-refresh tick already uses for Patches in
     `event_refresh/_auto_refresh_surfaces.py`, and do not create a second one.
   - Capture the predicate on the loop before each `asyncio.to_thread`, and pass it into
     `_prepare_patch_load_from_disk(build_relations=...)`.
   - When hidden, store a targets-only `RelationIndex`: the same `known_targets` the
     full builder computes, with no artifact-link edges. Mark the index deferred.
   - Apply the same rule to the synchronous paths (`_apply_patches`, and the
     `_apply_reloaded_patches` fallback): never build full edges while hidden.
   - The watcher keeps scheduling Patch list reloads through the existing
     `_schedule_patches_async_refresh` path (tui_perf rule 5). They just stop paying for
     edges while hidden.
3. **Rebuild on entry.**
   - When the user enters the Artifacts tab (`watch_current_tab` in
     `src/sase/ace/tui/_app_watchers.py` has no Patches consumer today) or switches to
     the Patches pane, schedule the existing async Patch refresh if the index is
     deferred or `_dirty_patches` is set.
   - The relation panel shows its existing loading or empty state until the refresh
     lands.
   - After any await, re-check visibility. If Patches became visible while a hidden
     build was in flight, schedule one follow-up.
4. **Startup off the loop.** In `_run_deferred_mount_state_loads`
   (`actions/_startup_loads_index.py`), replace `to_thread(_read_patches_from_disk)` +
   `_apply_patches` with
   `to_thread(_prepare_patch_load_from_disk, build_relations=<visible>)`. Then apply the
   prepared query and relation indexes without rebuilding on the loop: extend
   `_apply_patches` to accept them, as `_apply_reloaded_patches` already does. Mirror
   the change in the fast test harness `src/sase/ace/testing/_startup.py`.
5. **Adaptive debounce.**
   - When `_run_patches_async_refresh` finishes with `_patches_refresh_pending` set,
     start the follow-up after `min(last_reload_duration, 5 s)`, not immediately.
   - Keep `_patches_refresh_scheduled` true during that delay so requests keep
     coalescing.
   - Clear it if arming the timer fails.

**Tests.**

- Update `tests/ace/tui/test_event_handlers_artifact_dirty_flags.py` and
  `test_event_handlers_nav_gate.py` only where their expectations change.
- Add tests:
  - a hidden-tab watcher reload reloads the list but builds no edges;
  - entering Artifacts → Patches with a deferred index schedules exactly one refresh,
    which builds edges;
  - an unchanged `source_key` plus the same Patch objects is a memo hit, and a changed
    `source_key` rebuilds;
  - the startup deferred-mount path never calls the full builder on the loop;
  - link-follow from the Agents tab still resolves a Patch target through the
    targets-only index (`test_relation_reveal_navigation.py` or the link-follow tests);
  - the adaptive follow-up waits and coalesces.

**Measure.** Run a scripted churn: rewrite the artifact-link aggregate 10 times with the
Agents tab visible. Report real relation-index builds before and after; expected 0
after, against 10 before.

## Phase: startup-readiness-gate — gate hidden startup work on Agents readiness

1. **Long safety timeout.**
   - Set `_STARTUP_DEFERRED_FALLBACK_SECONDS` in `actions/_startup_loads_core.py` from
     the `fallback_seconds` decision: 30.0, 45.0 (default), or 60.0.
   - Release still happens at visible-ready through `_maybe_end_startup_stopwatch`. The
     tab-switch fast path `_maybe_start_startup_surface_for_tab` still starts a hidden
     surface immediately.
   - Update the docstring and comments that say "a few seconds".
   - Update `tests/ace/tui/test_startup_stopwatch_live_update.py`, which asserts the
     literal `3.0` and the timer name.
2. **No heavy auto-refresh before release.**
   - Today the tick is armed at first paint (`actions/_startup_mount.py`). The axe
     branch in `event_refresh/_auto_refresh_surfaces.py` runs before the
     `_agents_loading` skip. The first tick is a sanity refresh because
     `_last_full_sanity_refresh = 0.0` (`_state_init_runtime.py`).
   - While `_startup_deferred_loads_released` is false, the tick must not:
     - collect axe;
     - run sanity or broad Agents loads;
     - reload Patches. Cheap stat-only token probes may still run.
   - At release, set `_last_full_sanity_refresh` to the release time. The startup loads
     are themselves the first full loads, so the first sanity refresh comes a full
     interval later.
   - Do not touch the `refresh_interval` setting. Keep `ace_refresh_tokens` behavior
     intact (tui_perf rule 14).
3. **Never preempt or overlap the startup axe init.**
   - `_run_axe_startup_init` (`axe_display/_refresh_full.py`) does not set
     `_axe_status_refresh_running`, so a tick and the init can collect concurrently. A
     tick that loads axe first sets `_axe_first_load_done`, which makes
     `_start_startup_axe_surface` return early and skips auto-start or `--restart-axe`.
   - Make the startup init own the running flag, or make the tick skip axe while the
     startup axe surface has started but not finished.
   - Add tests:
     - a tick before release never sets `_axe_first_load_done`;
     - the startup init after release still performs auto-start;
     - a tick during the init does not start a second collect.
4. **Honest axe chip before the first axe load.** The footer chip
   (`widgets/_keybinding_status.py`, default `_axe_running=False` in
   `widgets/keybinding_footer.py`) shows a red STOPPED until the first axe load. Later
   release would stretch that false state, so render a neutral or loading state until
   `_axe_first_load_done`.
5. **Run heavy maintenance one job at a time.**
   - In `_schedule_deferred_startup_maintenance`
     (`actions/_startup_loads_maintenance.py`), keep cheap or I/O-only items immediate:
     the named-proc prune, the post-update toast, and the update-attempts refresh,
     unless measurement shows it is CPU-heavy.
   - Run the CPU-heavy jobs as one pump-free chain (`spawn_pump_free_task`), in this
     order:
     1. prompt-catalog warm;
     2. app-wide link-index refresh;
     3. startup update-toast check;
     4. usage-refresh fallback;
     5. prompt-bar import warm.
   - Each step starts its job through its existing scheduler and waits for that job's
     existing in-flight signal to clear: the catalog rebuild's `finally`,
     `_link_index_loading`, `_automatic_update_check_in_flight`, or equivalent.
   - Each step has a 20 s cap. Treat "already in flight or fresh" (for example a lazy
     link-index build triggered by selection, or the prompt bar opening) as that step's
     start.
   - Between steps, yield while `NavigationGate` reports navigation or the prompt input
     is active.
   - Cancel the chain at teardown with the existing pump-free cancellation.
   - The hidden-surface starts, deferred mount-state loads, post-roster warmups, and
     monitor reconcile flush stay at release, unchanged: they serve visible state and
     already coalesce.
6. **Docs.** Update the startup sections of `docs/perf_runbook.md` and `docs/ace.md`
   that describe the deferral, fallback, or tick behavior.

**Tests.** Extend `tests/ace/tui/test_startup_stopwatch_live_update.py` and
`tests/ace/tui/test_axe_startup_host_lifecycle.py`:

- Nothing deferred starts before visible-ready, even when the old 3 s mark passes.
- The fallback still releases everything (reason `fallback`) without marking
  visible-ready.
- Maintenance jobs start strictly one after another, in order, and a stuck step times
  out and the chain continues.
- The behaviors from item 3.

The dirty-flags helper pins `_last_full_sanity_refresh`, which hides first-tick
behavior. Add one test that does not pin it.

## Phase: startup-telemetry — make startup cliffs diagnosable from telemetry

1. **Loader disk substages, always on.** The `disk` stage (`_loading_disk_full.py`) is
   one aggregate, and its substage spans exist only under `SASE_TUI_TRACE=1`.
   - Add cheap `perf_counter` timers in the worker for `dismissed_snapshot`, `index`,
     `decode`, `provider` (identity plus status normalization), and `projections`.
     Today's spans live in `_loading_helpers.py` and `models/agent_loader.py`.
   - Return the timers on `_AgentDiskLoadResult`.
   - Write them as a `disk_substages` object in the agent-loads telemetry record
     (`SASE_TUI_AGENT_LOADS_PATH`), with the existing threshold rules unchanged.
2. **Startup record fields** (`actions/_startup_telemetry.py` →
   `logs/tui_telemetry.py`):
   - `deferred_release_reason`: `visible_ready`, `fallback`, or
     `fallback_schedule_failed`;
   - `deferred_release_seconds`: from `on_mount`;
   - `load1` and `cpu_count`: from `os.getloadavg()` and `os.cpu_count()` at mount;
   - `patch_relation_index_builds_before_ready`: the real-build counter from
     `patch-relation-index`, sampled at visible-ready;
   - `auto_refresh_ticks_before_release`.
   - Keep existing field names and meanings unchanged.
3. **Import-time sanity guard.** `interpreter_cli_import_seconds` falls back to the
   `/proc` start time when `SASE_TUI_EXEC_MONO_NS` is absent. Record `None` instead of a
   value that is negative or over one hour.
4. Document the new fields in `docs/perf_runbook.md`.

Bead `sase-1f4` (git source revision on `tui_startup` rows) is separate. Do not
implement it here, but avoid conflicting with it.

**Tests.** Extend `tests/ace/tui/actions/test_startup_telemetry.py` and
`test_startup_observability.py`:

- the new fields are present and typed;
- the substage timers sum to no more than the `disk` stage;
- the import guard drops bogus values.

## Phase: core-link-index — move the artifact-link walks into sase-core

This phase is `large`: plan it before implementing. Open the linked repo with
`sase repo open sase-core -r "<why>"`, work only in the printed path, read its
`AGENTS.md`, and run `sase tool run check` there.

**Today.**

- `link_index_for_snapshot` → `_build_link_index`
  (`src/sase/ace/tui/relations/link_index.py`) walks every aggregate row loaded by
  `load_artifact_links_snapshot` (`relations/artifact_links.py`). Each row makes
  `artifact_relation_lookup` plus two `artifact_link_canonicalize` binding calls, and
  each canonicalize repeats the `require_rust_binding` lookup. The build then makes
  per-edge label and ref-parts calls, plus two alias passes per ref.
- `artifact_link_edges`, used by `build_patches_relation_index`
  (`relations/patches.py`), re-walks the same rows to match refs against Patch lookup
  keys.
- A third walk, `build_agent_catalog_link_facets` (`agents/catalog/_query.py`), may be
  able to reuse the result.
- sase-core has per-ref helpers (`artifact_link_ref_parts`, `artifact_row_index_keys`,
  `artifact_row_ref_lookup_keys`, `artifact_link_canonicalize`,
  `artifact_relation_lookup`, `artifact_relation_label`) but no aggregate reader or
  inverted index.

**Target.**

1. **In `sase_core`**, add an artifact-link index built from the per-project aggregates.
   It must:
   - canonicalize and deduplicate rows, keeping the highest `uses`, with semantics
     identical to today's Python;
   - index both endpoints by `artifact_row_ref_lookup_keys` (with the row's project as
     hint);
   - expose by-ref queries: chips for a ref, the target for a ref, and the rows for a
     set of lookup keys. Expose it through `sase_core_py` as an opaque handle, so Python
     never materializes every chip. Add Rust parity fixtures.
2. **In this repo**, make `_build_link_index` and the Patch projection thin adapters
   over that handle:
   - memoize by the aggregate signature (`snapshot.source_key`);
   - make the build single-flight per scope, with check-and-build under one lock or
     in-flight future, so the link rail and the Record deck never build concurrently;
   - make the Patch projection use Patch index keys (already batched) → matching rows,
     so it scales with the number of Patches, not the number of rows;
   - keep the `patch-relation-index` memo and deferral on top;
   - add a trace span for the build;
   - delete the dead Python walk code, keeping symvision clean.
3. **Parity.** Before switching, compare the old and new outputs (chips per ref,
   targets, Patch edges) on fixture data and on a copy of a real aggregate, and keep a
   parity test.
4. **Pin and range.** Move `sase-core-revision.txt` past the sase-core commit. Update
   the `sase-core-rs` range in `pyproject.toml` if a new wheel is required, per
   `docs/rust_backend.md` ("The CI source revision pin"), and keep
   `tools/check_sase_core_rs_bindings` green.

**Measure.** Report `_build_link_index` and Patch-projection CPU before and after on the
same aggregate. The report measured 7.5–8.5 s and 2.2–2.4 s on apollo.

## Out of scope

- Apollo ops from report items 4 and 11: setting `SASE_TEST_GATE_SLOTS`, running tests
  under `nice`/`ionice`, `%dispatch` to athena, a Python 3.14 tool env (human-only
  global install), and a faster droplet. These go to the user directly.
- sase-core scan-wire fields that remove per-record commit/revert probes, the
  `touch_write_meta` replay commits, gateway containment, and GC changes. Phases may
  record these as `PROPOSED FOLLOW-UP:`.
- Changing the suite-gate worker memory estimate (the 1.1–1.4 GB observation is
  unconfirmed).

## Expected outcome

`plugin-discovery-memo`, `patch-relation-index`, and `startup-readiness-gate` ship
proper versions of the three monkeypatches in the report's A/B (about 42 s → 12 s from
init on apollo at load ≈25). `toolrun-glance-coalesce` removes the end-of-load glance
stampede. `loader-path-trims` and `tool-wait-polling` remove contention the report
under-counted. `core-link-index` removes the largest remaining steady-state CPU walk.
After landing, the user can confirm on apollo by updating it and comparing
`agents_ready_seconds` and the new fields in `~/.sase/logs/tui_startup.jsonl`, bucketed
by SHA.
