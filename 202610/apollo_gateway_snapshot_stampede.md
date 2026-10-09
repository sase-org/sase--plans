---
tier: epic
title: Stop the fleet gateway snapshot stampede that melted apollo
goal: 'A slow fleet snapshot rebuild can no longer multiply into hundreds of concurrent
  index-scanning threads. The gateway runs at most one rebuild per scope, always keeps
  late results, backs off after failures, bounds its artifact-index work, and logs
  every build. The artifact index keeps its WAL bounded. Remote clients stop amplifying
  a slow host. apollo''s gateway is restarted only after measurements show it stays
  healthy under athena''s real polling.

  '
phases:
- id: gateway-single-flight
  title: Single-flight, back-off, and bounded index work in FleetReadService
  depends_on: []
  size: medium
  description: 'gateway-single-flight: in sase-core, rewrite the FleetReadService
    snapshot refresh. Each scope gets one detached in-flight build that always fills
    the cache when it finishes. Callers wait up to the timeout. Failures back off
    exponentially. A shared semaphore bounds index work, the overlay pass is coalesced
    and best-effort, and the gateway runtime caps blocking threads. Includes a slow-build
    stampede regression test.'
- id: index-sqlite-hygiene
  title: Artifact index WAL bounds and write batching
  depends_on: []
  size: medium
  description: 'index-sqlite-hygiene: in sase-core agent_scan/index, set journal_size_limit
    and synchronous=NORMAL on every read-write open, and add a WAL checkpoint helper
    for oversized WALs that background maintenance calls. Batch revalidation writes
    into one transaction per pass, and skip unchanged reconcile-watermark meta writes.'
- id: worker-host-backoff
  title: Federation worker keeps host state and backs off slow hosts
  depends_on: []
  size: medium
  description: 'worker-host-backoff: in the sase-core federation worker, reuse unchanged
    RemoteHost instances across replace_config. That keeps the HTTP client, hello
    verification, per-host permits and back-off state alive across TUI refreshes.
    Add exponential per-host back-off that serves cached data instead of calling a
    host that just timed out or failed.'
- id: gateway-observability
  title: Gateway refresh telemetry and WAL housekeeping
  depends_on:
  - gateway-single-flight
  - index-sqlite-hygiene
  size: small
  description: 'gateway-observability: install a tracing subscriber in the sase_gateway
    binary. Emit events for every snapshot build, back-off engagement, overlay skip,
    and long-running build. Call the new WAL checkpoint helper after successful Presentation
    builds while holding the index permit.'
- id: fleet-client-hardening
  title: sase fleet client stops amplifying slow hosts
  depends_on: []
  size: medium
  description: 'fleet-client-hardening: in sase Python, give the IPC socket a grace
    period beyond the worker deadline, classify socket timeouts separately, and stop
    respawning and resending on a slow-but-alive worker. Make the TUI fleet refresh
    single-flight with one pending rerun. Add schema_version to the fallback diagnostics,
    and degrade normalization failures to a visible fleet error.'
- id: apollo-rollout-verify
  title: Gated apollo gateway restart and measured verification
  depends_on:
  - gateway-single-flight
  - index-sqlite-hygiene
  - worker-host-backoff
  - gateway-observability
  - fleet-client-hardening
  size: small
  description: 'apollo-rollout-verify: confirm the fixes have landed and that apollo''s
    install contains them. Probe apollo read-only. Propose the update and gateway
    restart only through a sase gate whose follow-up measures threads, index handles,
    CPU, RSS, WAL size and build logs against the pass criteria, and stops the gateway
    again if any criterion fails.'
proposed_by: bbugyi200.athena.0yx
create_time: 2026-10-09 09:31:09
status: wip
bead_id: sase-1j1
---

- **PROMPT:** [prompts/202610/apollo_gateway_snapshot_stampede.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/apollo_gateway_snapshot_stampede.md)
- **BEAD:** [sase-1j1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1j1/README.md)

# Plan: Stop the fleet gateway snapshot stampede that melted apollo

## Background

The 2026-10-09 investigation (research report
`research:202610/apollo_high_cpu_gateway_snapshot_stampede.md`) found that apollo's
`sase_gateway` service proc burned 7–11 of 16 cores, about 60% of it kernel time,
continuously from about 00:00 UTC. It ran with ~500 threads, ~1,000 open SQLite handles
(at the 1,024 fd limit) and 8–12 GB RSS, caused two OOM kills, and grew a 4.9 GB WAL.
Restarts and a reboot reproduced it within a minute, even with a clean WAL. The user has
since stopped the gateway proc on apollo.

The research's root cause is confirmed against current sase-core code
(`crates/sase_gateway/src/fleet_reads/service.rs`):

1. `current_snapshot` / `current_history_snapshot` run
   `tokio::time::timeout(SNAPSHOT_REFRESH_TIMEOUT = 4 s, spawn_blocking(build_snapshot_blocking))`.
   A timeout drops only the join handle. The blocking build keeps running, and dropping
   `_guard` releases `refresh_lock`.
2. The cache is written only in the `Ok(Ok(Ok(snapshot)))` arm, so a late result is
   discarded. `retain_previous_or_error` re-caches the old snapshot but keeps its old
   `build_instant`, so it stays expired (≥ `FLEET_SNAPSHOT_STALE_SECONDS` = 60). Every
   queued request then takes the lock, misses again, and starts another build. On a cold
   start there is no previous snapshot, so every request errors and leaves another
   orphan behind.
3. Nothing bounds artifact-index work. Each orphan runs a read-write `Revalidate` query:
   a fresh `open_index` connection, per-row `stat` signature checks, and autocommit
   upserts. It then runs a dismissal-lineage pass. `overlay_missing_into_snapshot` adds
   an uncoalesced, untimed full index query plus a lineage pass on every presentation
   `catalog` page. The runtime uses `Runtime::new()`, so up to 512 blocking threads can
   pile up.
4. The index sets no `journal_size_limit` and never checkpoints. Hundreds of overlapping
   readers keep the WAL from ever restarting, and its growth slows every build further.
5. Client amplification:
   - Every athena TUI refresh builds a new federation supervisor, so it re-sends
     `replace_config`.
   - The worker's `replace_config` rebuilds every `RemoteHost`. That resets per-host
     permits, the HTTP client and hello verification on every refresh.
   - The Python IPC socket timeout equals the worker deadline, so the client gives up
     just before the worker's bounded partial response arrives (the worker adds
     `OUTER_DEADLINE_GRACE` = 250 ms).
   - The supervisor treats that timeout as a dead worker. It respawns the worker and
     resends the same request.
   - Refresh tasks overlap with no in-flight dedup.
   - The `_fleet_call` fallback diagnostic lacks `schema_version`. One slow host
     therefore raises `fleet federation diagnostics[0] … missing field schema_version`
     and kills the whole refresh.

This epic fixes the defect in sase-core and hardens the sase client. It also rolls the
fix out to apollo with measured evidence. Per the Rust-core boundary, all gateway,
worker and index behavior changes land in the linked `sase-core` repo (open it with
`sase repo open sase-core`). Only the TUI/IPC client changes land in sase. None of the
phases adds a Python binding, so sase's `sase-core-revision.txt` pin does not need to
move for this epic.

### Invariants the finished system must hold

- At most one Presentation snapshot build and one History snapshot build are running in
  the gateway at any moment. No request ever starts a second build for a scope while one
  is in flight.
- A build that finishes always updates the cache, whether the requests that triggered it
  are still waiting or have timed out.
- After a failed build, no new build for that scope starts until its back-off window has
  passed.
- At most two concurrent artifact-index operations run in the gateway, and the overlay
  never waits for one.
- The artifact index WAL is bounded by `journal_size_limit` and housekeeping
  checkpoints.
- Every snapshot build leaves a log line with its scope, outcome and duration.

## Phase `gateway-single-flight` (sase-core)

Files: `crates/sase_gateway/src/fleet_reads/service.rs` (main),
`crates/sase_gateway/src/fleet_reads/snapshot.rs` (only if needed), and
`crates/sase_gateway/src/cli.rs`. Tests go in
`crates/sase_gateway/src/fleet_reads/tests/` (extend `snapshots.rs`, or add a sibling
such as `refresh.rs` registered in `tests/mod.rs`). Keep every file at or under 1,500
lines. If `service.rs` grows too large, move the refresh machinery into a new
`fleet_reads/refresh.rs` module behind the existing facade.

### Per-scope refresh state

Replace `cache` + `refresh_lock` (and the history pair) with one per-scope state
structure behind a `std::sync::Mutex`. Never hold that mutex across an `.await`. The
state holds:

- `cache: Option<CachedFleetSnapshot>`: written only when a build completes
  successfully.
- `in_flight`: a shared completion handle for the one running build, plus its start
  instant. Prefer `tokio::sync::watch` or `Notify` plus state, because tokio `sync` is
  already enabled and needs no new dependency. Only add `futures-util` if it is clearly
  simpler; if you do, run `cargo hakari generate && cargo hakari manage-deps`.
- Failure tracking: `last_failure: Option<(Instant, FleetReadError safe code/target)>`
  and `consecutive_failures: u32`.

### Refresh algorithm (`current_snapshot(force)`, identical for both scopes)

1. If the cache is unexpired and `force` is false, return it.
2. Under the state mutex, pick one path:
   - **A build is in flight:** join it.
   - **Otherwise, inside the back-off window:** do not start a build. Return the cached
     snapshot marked stale, or the remembered failure error if there is no cache.
   - **Otherwise:** start a build. Spawn a **detached** `tokio::spawn` task that:
     1. acquires the gateway-wide index semaphore (below);
     2. runs `spawn_blocking(build_snapshot_blocking)`;
     3. writes the outcome into the scope state: on success it fills the cache and
        resets the failure tracking; on failure it records the failure and bumps
        `consecutive_failures`;
     4. clears `in_flight` and notifies waiters.

   No caller ever drops or aborts the build.

3. The caller waits for completion for at most `refresh_timeout` (still 4 s).
   - **Build done in time and succeeded:** return the new snapshot.
   - **Build failed:** return the previous snapshot marked stale, partial, with the
     error code (today's `retain_previous_or_error` observable behavior), or the error
     if there is no previous snapshot.
   - **Wait timed out:** return the previous snapshot marked stale, partial, with
     `error = "timeout"`, or `FleetReadError::Timeout("snapshot_refresh")` if there is
     no previous snapshot.

   Stale marking applies to the **returned copy** and must never rewrite the cached
   entry's `build_instant`.

4. `force = true` (used by `reconcile` after a launch settles) skips only step 1's
   freshness check. It must never run concurrently with another build of the same scope.
   - **Build already in flight:** wait for it, then start (or join) a build that began
     **after** the forced call. `reconcile` needs a post-settlement view.
   - **Back-off:** force respects the back-off window.

Back-off schedule: 15 s after the first failure, doubling per consecutive failure,
capped at 120 s, and reset on success. A late-but-successful build is a success.

Keep `stamped_snapshot` / `stamped_history_snapshot` freshness stamping as is. Keep the
public read API, the error mapping (`routes/errors.rs`) and the wire contract unchanged:
no fleet contract snapshot should change.

### Bounded index work

- Add one gateway-wide `Arc<tokio::sync::Semaphore>` with **2** permits to
  `FleetReadServiceInner`. Snapshot builds `acquire_owned` a permit inside the detached
  task and hold it for the whole blocking build.
- `overlay_missing_into_snapshot`:
  - **Best-effort:** `try_acquire_owned` a permit and skip the overlay (return the
    snapshot unchanged) when none is free.
  - **Coalesced:** allow at most one overlay pass in flight; a concurrent caller skips.
  - **Memoized:** remember the last overlay result, keyed by the snapshot's identity
    (for example `refresh_count` + scope) and the sorted missing-key set. A burst of up
    to 16 catalog pages against one snapshot then runs at most one overlay query.
- Keep `reconcile`, `content`, `project_eligibility` and the event stream behaving as
  today. They go through the new path automatically.

### Runtime cap

In `cli.rs`, build the gateway runtime with
`tokio::runtime::Builder::new_multi_thread().enable_all().max_blocking_threads(32).build()`
instead of `Runtime::new()`. Use a small named constant. Leave the federation worker
runtime alone.

### Test hooks

Add `#[cfg(test)]` accessors that count builds started per scope, report whether a build
is in flight, and expire the back-off window without sleeping. Keep the existing
`refresh_count_for_test` / `age_cache_for_test` hooks working.

### Tests (new)

Use `FleetReadService::new_with_liveness` with a short `refresh_timeout` (for example 50
ms) and a closure `OwnerLivenessObserver` that sleeps (for example 150 ms per record)
and tracks the maximum number of concurrent invocations in an `AtomicUsize`. Wait for
detached builds by polling a hook with a bounded `tokio::time::sleep` loop, never with
unbounded sleeps.

1. **Stampede regression.** Fire 20+ concurrent `summary()` calls against a cold cache
   with a slow build.
   - Every call returns promptly with `Timeout`.
   - Exactly one build started.
   - The observer never ran concurrently.
   - After the build finishes, the cache is filled (`refresh_count_for_test() == 1`),
     and the next `summary()` returns `Ok` without starting another build.
2. **Late result fills the cache when a previous snapshot exists.** The waiter gets a
   stale, partial response with `error == "timeout"`. Once the build lands, a later read
   is `Fresh` with the new `refresh_count`.
3. **Back-off.** Use the existing deterministic failure (replace the index file with a
   directory).
   - The first failure records one build.
   - Calls inside the window start no build and return the stale snapshot (or the error
     on a cold cache).
   - After the test hook expires the window, exactly one new attempt runs.
   - A success resets `consecutive_failures`.
4. **Forced refresh.** A `reconcile()` issued while a build is in flight never overlaps
   it, and its "after" snapshot comes from a build that started after the call.
5. **Overlay.** With settled launch receipts missing from the snapshot, several
   presentation `catalog` pages against one snapshot run the overlay index query at most
   once, and the overlay is skipped (catalog still `Ok`) while both index permits are
   held.
6. **Existing tests still pass unchanged:** `concurrent_snapshot_reads_are_coalesced`,
   `aged_cache_triggers_exactly_one_coalesced_rebuild`,
   `failed_rebuild_retains_stale_partial_prior_snapshot`,
   `failed_history_rebuild_retains_stale_partial_prior_snapshot`, the presentation
   tests, and `routes/tests/fleet_handlers.rs`. If one encodes the old orphaning
   behavior, adjust it only to the new invariant and say why in the test.

### Verify

In the sase-core checkout: `just test -p sase_gateway` while iterating, then
`sase tool run check` before finishing. Commit subjects use Conventional Commits, for
example `fix(gateway): single-flight fleet snapshot refresh survives timeouts`.

## Phase `index-sqlite-hygiene` (sase-core)

Files: `crates/sase_core/src/agent_scan/index/storage.rs`, `query.rs`, `refresh.rs`,
`selection.rs`, `maintenance.rs`, and the `index/mod.rs` facade. Tests go in
`crates/sase_core/src/agent_scan/index/tests/` (reuse `support.rs` helpers such as
`rebuild_index`, `full_history_revalidate_query`, `count_sql`).

1. **Connection pragmas.** In `open_index` (read-write), after `journal_mode=WAL`, set:
   - `PRAGMA synchronous = NORMAL`. This is safe in WAL mode; the index is a rebuildable
     derived cache.
   - `PRAGMA journal_size_limit = 67108864` (64 MiB), as a named constant.

   Leave `open_index_read_only` without writes. Its fallback path already goes through
   `open_index`.

2. **WAL checkpoint helper.** Add a public core function in the index module, for
   example
   `checkpoint_agent_artifact_index_wal_if_oversized(index_path, max_wal_bytes)`:
   - Stat `<index>-wal`. If it is missing or at most `max_wal_bytes`, return a "skipped"
     outcome without opening a connection.
   - Otherwise open read-write with a short busy timeout (about 1 s), run
     `PRAGMA wal_checkpoint(TRUNCATE)`, and return an outcome struct with the WAL bytes
     before and after plus the checkpoint's busy, log and checkpointed counts. A busy
     result is a normal outcome, not an error.

   Export it from the index facade, and therefore from `sase_core::agent_scan`, for the
   gateway phase to call. Default threshold constant: 64 MiB.

3. **Background maintenance calls it.** At the end of
   `terminalize_stale_active_agent_artifact_index_rows`, call the helper best-effort and
   ignore its errors; that function is already documented as background maintenance. Do
   not change that function's return wire. Python's startup maintenance
   (`_run_active_tier_maintenance`) then gets WAL housekeeping with no binding change.
4. **Batch revalidation writes.** In the Revalidate paths:
   - `refresh_stale_rows`
   - the per-row signature repair in `selection.rs`
   - `reconcile_source_directories`

   For each of them, keep all filesystem scanning and signature work **outside** any
   transaction. Then apply the collected upserts and deletes inside one transaction per
   pass (`conn.unchecked_transaction()` works on `&Connection`). Preserve today's
   semantics: upsert errors that are ignored today stay ignored and must not abort the
   batch, while the discovery upsert that propagates with `?` still propagates.

5. **Skip no-op meta writes.** `stamp_source_reconcile_watermark` reads the two `meta`
   keys and writes only those whose value would change.

### Tests (new)

- A read-write open reports `synchronous = 1` (NORMAL) and the configured
  `journal_size_limit`.
- The checkpoint helper returns skipped for a small or missing WAL. With an oversized
  WAL it truncates to (near) zero bytes:
  - Build the WAL with a connection that sets `wal_autocheckpoint=0`, then write enough
    rows.
  - Close that connection, or hold a reader to exercise the busy outcome.
  - Call the helper with a tiny threshold.
- An unchanged full-history `Revalidate` query run twice commits nothing on the second
  run. Observe this via `PRAGMA data_version` on a separate connection, or an equivalent
  commit counter.
- A revalidation pass that repairs several stale rows produces the same rows as before
  (extend an existing `self_heal.rs` case), so batching changes no outcome.
- All existing `agent_scan` index tests, `sase_core/tests/agent_scan_parity.rs`, and the
  `sase_core_py` agent-scan binding tests pass unchanged.

### Verify

`just test -p sase_core agent_scan` while iterating, then `sase tool run check` in the
sase-core checkout. Use a Conventional Commit subject such as
`fix(agent-scan): bound artifact index WAL and batch revalidation writes`.

## Phase `worker-host-backoff` (sase-core)

Files: `crates/sase_gateway/src/federation_worker/imp/state.rs`, `imp/remote_hosts.rs`,
and `federation_worker/mod.rs` (constants). Tests go in
`crates/sase_gateway/src/federation_worker/imp/tests/` (`remote.rs`, `operations.rs`,
`support.rs`).

1. **Preserve unchanged hosts.** In `FederationWorkerState::replace_config`, reuse the
   existing `Arc<RemoteHost>` for an installation ID when its validated config is
   identical (alias, connection plan, bearer token). Only build a new `RemoteHost` for a
   new or changed entry; removed hosts drop as today. The HTTP client and connection
   pool, hello verification, quarantine, per-host permits and back-off state then
   survive the `replace_config` that every TUI refresh sends. Keep the per-host result
   rows (`configured`, `invalid`, …) the same as today.
2. **Per-host read back-off.**
   - **State.** Extend `RemoteHostRuntime` with `backoff_until: Option<Instant>`,
     `consecutive_failures: u32` and `last_error: Option<FederationErrorWire>`.
   - **Short-circuit.** In `read_one_host` (non-`cache_only`), check the back-off before
     acquiring any permit. While backing off, return
     `cached_or_error(host, cache, key, Some(err))` without contacting the host. `err`
     reuses the triggering error's `code` (so `status_from_error` yields the same status
     vocabulary: `deadline` / `unavailable`). It sets `target = "backoff"` and a message
     naming the remaining seconds. Introduce **no** new status or error-code value.
   - **What triggers back-off.** Retryable failures of a remote read, including the
     hello step. These are error codes `deadline`, `timeout`, `unavailable` and
     `internal` (HTTP 5xx or connect failures). `unauthorized`, `not_found`, `invalid*`
     and `quarantined` do not trigger it.
   - **Schedule.** 5 s base, doubling per consecutive failure, capped at 120 s, and
     reset on any successful read.
   - **Scope.** Make sure `read_catalog_hosts` goes through the same path. Mutations,
     launches and attention resolution are user-initiated and **must not** be
     short-circuited by read back-off.
3. Add named constants for the schedule next to `DEFAULT_PER_HOST_IN_FLIGHT`.

### Tests (new)

Use the existing fake-remote helpers in `imp/tests/support.rs`.

- Two consecutive `replace_config` calls with the same hosts keep the same `RemoteHost`
  (pointer-equal `Arc`, or an observable proof such as `/hello` being called once across
  two refreshes). A changed endpoint or token builds a new one.
- After a remote read times out, an immediate second read makes no HTTP request and
  returns the cached payload with status `stale`, or the error with status `deadline`
  and target `backoff`. After the window (use a test hook or an injectable clock), a
  read contacts the host again, and a success resets the failure count.
- A launch or mutate during back-off still contacts the host.
- Existing worker tests (per-host deadline / healthy-partial-host, cache, and IPC tests
  in `imp/tests/`) pass unchanged.

### Verify

`just test -p sase_gateway federation` while iterating, then `sase tool run check` in
the sase-core checkout. Use a Conventional Commit subject such as
`fix(federation-worker): keep host state across replace_config and back off slow hosts`.

## Phase `gateway-observability` (sase-core)

Depends on `gateway-single-flight` (refresh state machine) and `index-sqlite-hygiene`
(checkpoint helper).

1. **Subscriber.** Add `tracing` and `tracing-subscriber` (both already workspace deps)
   to `crates/sase_gateway/Cargo.toml`, then run
   `cargo hakari generate && cargo hakari manage-deps` if `./scripts/check.sh features`
   asks for it.
   - Install a `tracing_subscriber::fmt` subscriber writing to **stderr** with an
     `EnvFilter` from `RUST_LOG`, defaulting to `info`. Install it only on the serve
     path of `run_gateway_cli`, not for `--contract-out`.
   - Do not install it in the federation worker or sudo runner.
   - The service host already pumps the proc's stderr into its service-proc log, so no
     sase change is needed.
   - Keep tower-http per-request spans out of the default `info` output.
2. **Events** (target `sase_gateway::fleet_reads`) from the refresh state machine:
   - **Build finished:** scope, outcome (`ok` or the error's safe code/target), duration
     in ms, served rows, `refresh_count`, and how many waiters timed out during the
     build. Log at `info` normally, and at `warn` when the build failed or took longer
     than `refresh_timeout`.
   - **Back-off engaged:** scope, consecutive failures, window seconds. Log at `warn`.
   - **Long-running build:** log one `warn` per build that is still in flight 60 s after
     it started (a watchdog check on the next caller, or a timer in the detached task).
     It cannot be cancelled, so this is the only signal for a wedged build.
   - **Overlay skipped** because of contention or the coalescing slot. Log at `debug`.
   - Per-request wait timeouts are counted, not logged individually.
3. **WAL housekeeping.** After each **successful Presentation** build, still inside the
   detached task's blocking section and while holding the index permit, call
   `checkpoint_agent_artifact_index_wal_if_oversized(index_path, 64 MiB)` best-effort.
   - When it checkpoints, log an `info` event with the bytes before and after.
   - When it fails, log a `warn` event.
   - Never fail the build because of it.
4. Mention the gateway's stderr logging, `RUST_LOG`, and the refresh/back-off behavior
   briefly in `crates/sase_gateway/README.md`.

### Tests

- A unit test asserts that the checkpoint hook is invoked after a successful
  Presentation build (for example a counter hook) and not after a failed one.
- An existing-style test proves the subscriber-install helper is idempotent: calling it
  twice in tests must not panic, so use `try_init`.

### Verify

`sase tool run check` in the sase-core checkout. Use a Conventional Commit subject such
as `feat(gateway): log fleet snapshot builds and checkpoint oversized index WAL`.

## Phase `fleet-client-hardening` (sase)

Read `sase/memory/tui.md` (through `/sase_memory_read`) before touching the TUI. This
phase changes no rendered output, so no PNG golden updates are expected.

1. **Separate IPC timeouts from dead workers**
   (`src/sase/dispatch/federation/_errors.py`, `_ipc.py`).
   - **New error class.** Add `FederationWorkerTimeout(FederationWorkerUnavailable)`.
     Existing `except FederationWorkerUnavailable` sites keep working.
   - **When `_ipc.py` raises it.** When `connect` succeeded but a `send`/`recv` hit
     `TimeoutError`/`socket.timeout`.
   - **When it does not.** `ConnectionRefusedError`, `FileNotFoundError` and "closed
     mid-frame" stay plain `FederationWorkerUnavailable`.
   - **Response grace.** Keep the envelope `deadline_unix_ms = now + timeout`, but set
     the socket timeout to `timeout + _IPC_RESPONSE_GRACE_SECONDS` (a 1.0 s module
     constant). The worker's bounded partial response, sent at deadline + 250 ms, then
     arrives instead of losing the race.
2. **No respawn or resend on a slow worker** (`_supervisor.py`).
   - In `FederationWorkerSupervisor.request`, re-raise `FederationWorkerTimeout`
     immediately. Never call `ensure_started(force=True)` or resend the operation after
     it.
   - In `ensure_started` / `_healthy`, a health probe that connected but timed out means
     the worker is alive but busy. Do not spawn a replacement; proceed to the request.
   - Only connection-level unavailability triggers respawn and retry.
3. **Single-flight TUI fleet refresh**
   (`src/sase/ace/tui/actions/agents/_fleet_refresh.py`).
   - **Coalesce.** In `_schedule_agents_fleet_refresh`, when `force` is false and a
     fleet refresh task in `_agents_fleet_async_tasks` is still running, record one
     pending rerun (a flag plus the latest `source`) and return. Do **not** bump
     `_agents_fleet_refresh_generation`: a bump would make the in-flight result stale
     and discard it.
   - **Rerun.** When the running refresh finishes (its `finally`), if a rerun is
     pending, clear it and schedule exactly one new refresh.
   - **Forced refresh.** Keep today's cancel-and-restart behavior, and clear any pending
     rerun.
   - Initialize the new attributes wherever `_agents_fleet_async_tasks` is initialized.
4. **Valid fallback diagnostics.**
   - In `_fleet_call`, add `"schema_version": 1` to the synthetic diagnostic so it
     passes `normalize_fleet_federation_response`.
   - Do the same in `diagnostic_wire` (`src/sase/dispatch/federation/_hosts.py`).
   - Check the Rust `FleetEnvelopeDiagnosticWire` shape (it denies unknown fields) and
     match it exactly.
5. **Degrade instead of dying.** In `_run_agents_fleet_refresh`, also catch `ValueError`
   raised by fleet-wire normalization or projection. Log it at `warning` with `exc_info`
   and call `_apply_fleet_error(...)`, so a bad remote response becomes a visible fleet
   error instead of a silently dead task.

### Tests

Extend `tests/test_dispatch_federation.py` and the fleet TUI tests in `tests/ace/tui/`
(`FleetRefreshHarness`, `OfflineFleetFacade`, `fleet_host_response` fixtures).

- The IPC client raises `FederationWorkerTimeout` on a recv timeout, and plain
  `FederationWorkerUnavailable` on a refused or missing socket. The socket timeout
  includes the grace while the envelope deadline does not.
- The supervisor does not respawn or resend after `FederationWorkerTimeout`, and still
  respawns and retries once after a refused connection. Use the existing fake
  `client_factory` / `popen` pattern.
- A slow health probe (timeout) does not spawn a worker.
- Scheduling three non-forced refreshes while one is in flight runs exactly two (the
  current one plus one coalesced rerun). The in-flight result is applied, not discarded.
  A forced refresh still cancels and restarts.
- The `_fleet_call` fallback payload round-trips through
  `normalize_fleet_federation_response` without error.
- A refresh whose facade raises `FederationWorkerUnavailable` on `catalog` ends with
  `_agents_fleet_last_error` set (or the per-host diagnostic visible) instead of an
  escaped `ValueError`.

### Verify

`just fmt`, then `sase tool run check` in sase, per `sase/memory/lint_and_test.md`.

## Phase `apollo-rollout-verify`

This phase touches the user's apollo machine, so it acts only through user-approved
gates. The user stopped apollo's gateway proc manually; do not start it outside a gate.

1. **Confirm the fixes landed.** Check that the four sase-core phases are on sase-core
   `master` and that the client phase is on sase `master`. Record the sase-core commit
   that completes the gateway fix.
2. **Probe apollo read-only** over `ssh apollo` (`ssh apollo-do` if Tailscale is down;
   see `sase/memory/tailnet.md`):
   - the gateway proc state (`~/.local/bin/sase service proc` status/list);
   - whether apollo's installed sase / `sase_core_rs` / `sase_gateway` build contains
     the fix commit (use apollo's `~/.sase/logs/dev_update.jsonl` and the installed
     version or commit);
   - the size of `~/.sase/agent_artifact_index.sqlite-wal`.
3. **Propose one gate** with `/sase_gate`. Its command:
   - updates apollo's install if it predates the fix;
   - if the WAL is over 64 MiB, runs `PRAGMA wal_checkpoint(TRUNCATE)` while the gateway
     is still stopped;
   - starts the gateway proc.

   The gate description must:
   - ask the user to keep athena's TUI on the Agents tab during the measurement window
     (worst-case real polling);
   - note that athena's TUI only gets the client-side fixes after the user restarts or
     reinstalls it.

4. **Gate follow-up: measure.** Sample apollo for at least 10 minutes, about once a
   minute, then compare with the pass criteria from the research:

   | Metric                                                                                         | Pass criterion                                                                                   |
   | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
   | Gateway threads                                                                                | Stay around the tokio worker count plus at most ~32 blocking threads, flat (not growing ~50/min) |
   | Open `agent_artifact_index.sqlite` handles (`ls -l /proc/<pid>/fd \| grep -c 'index.sqlite$'`) | ≤ 3                                                                                              |
   | Gateway CPU                                                                                    | < 1 core averaged over the window                                                                |
   | Gateway RSS                                                                                    | < 1 GB                                                                                           |
   | WAL size                                                                                       | ≤ 64 MiB                                                                                         |
   | Gateway log                                                                                    | Shows build events with durations; no repeating back-off or long-running-build warnings          |
   | athena `~/.sase/logs/tui.log`                                                                  | Stops logging fleet-refresh failures for apollo                                                  |
   - **If any criterion fails:** stop the gateway again
     (`~/.local/bin/sase service proc stop gateway`), capture the evidence (thread
     count, fd breakdown, build-duration log lines), and report.
   - **If all pass:** report the measurements.

   Either way, record the results in this phase's bead notes. Also record the
   out-of-scope items below as `PROPOSED FOLLOW-UP:` notes.

## Out of scope (record as PROPOSED FOLLOW-UP notes)

- Per-proc cgroup resource limits (`MemoryHigh`/`MemoryMax`, CPU weight) for service
  procs in the sase service host (research P2). The in-process bounds in this epic
  (single-flight, permits, `max_blocking_threads`) are the primary containment.
- Pooling artifact-index connections and skipping schema DDL on every `open_index`.
- Moving `FleetLaunchStore::settled_unexpired` (a file lock and read) off the gateway's
  async worker threads.
- Reusing one `FederationFacade`/supervisor across TUI refreshes. With the worker now
  preserving unchanged hosts, this is an optimization only.
- Coalescing identical in-flight `(host, cache_key)` reads inside the federation worker.
