---
tier: tale
title: Complete bounded fleet reads and recoverable gateway events
goal:
  The Rust fleet gateway serves safe bounded authoritative reads and loss-aware live
  invalidations for downstream federation clients.
size: medium
proposed_by: bbugyi200.athena.sase-xe.5
bead: sase-xe.5
create_time: 2026-09-06 18:41:28
status: wip
---

- **BEAD:**
  [sase-xe.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.5.md)

# Complete bounded fleet reads and recoverable gateway events

## Goal

Complete phase `sase-xe.5` in the linked `sase-core` repository. Add the authenticated,
versioned `/api/fleet/v1` read surface needed by the later federation worker:
authoritative counts, bounded catalog pages, followed-ID batch lookup, lazy detail,
opaque bounded content reads, project eligibility, and an SSE invalidation feed that
delivers both replayed and post-connect events. Keep all owner-only filesystem and
process effects in the gateway; keep validation, request/response records, bounds, and
cursor decisions transport-free in `sase_core`.

Do not route fleet reads through `AgentHostBridge`, invoke Python, expose absolute paths
or PIDs, or change the Python repository/core revision pin unless a binding caller is
actually required. Each host exports only records it owns. Feed cursors are
generation-plus-sequence positions and must remain distinct from row/action revisions.

## Implementation

1. Extend `crates/sase_core/src/fleet_contract.rs` and its public exports with strict
   version-1 wire records for summary, catalog query/page, logical-ID batch lookup,
   detail, content range/offset reads, bounded project eligibility, authoritative
   snapshot, invalidation events, and resync results. Define conservative named limits
   for page rows, batch IDs, project IDs, query/filter text, content bytes, and replay
   capacity; validate schema versions, cursors, locators, handles, offsets, and limits
   before any I/O. Specify stable ordering and continuation semantics, retain explicit
   freshness/revision/cursor metadata, and reuse `ResolvedAgentSummaryWire`,
   `ResolvedAgentDetailWire`, `ContentHandleWire`, `StoreCursorWire`,
   `count_logical_agents`, and `classify_cursor_replay`. Add focused unit tests for
   malformed/unbounded requests, deterministic paging and batch ordering, content
   continuation/EOF metadata, invalid locators/handles, and every replay/resync reason.

2. Add a focused module under `crates/sase_gateway/src/` for the fleet read service.
   Derive the artifact-index and project roots only from the configured SASE home, then
   call `query_agent_artifact_index`, exact indexed record loads, project lifecycle
   helpers, and core projection/count functions directly in Rust. Centralize owner
   resolution: build stable logical/exact locators, resolve process liveness locally,
   derive lifecycle/attention/current-instance state and deterministic row revisions,
   normalize capabilities, and mint opaque content handles whose private path mapping
   never enters a wire record. Fence handles to the snapshot generation and row
   revision, canonicalize allowed files under the owning artifact directory, and return
   capped byte ranges with digest, total length, next offset, EOF, and growth semantics.

3. Make snapshot refresh bounded and shared. Execute SQLite/filesystem/process work via
   `tokio::task::spawn_blocking` behind explicit timeouts, coalesce concurrent
   refreshes, cache one small resolved authoritative snapshot, and preserve the last
   good snapshot when revalidation fails or times out while exposing stale/partial
   freshness. Keep counts independent of catalog page size and make batch lookup query
   requested logical IDs rather than relying on the current page. Add one jitter-ready
   reconciliation operation that diffs a refreshed snapshot against the prior snapshot
   and emits invalidations for launches, lifecycle/attention/revision changes,
   deletions, and owner-observed process exits without causing one archive scan per SSE
   client.

4. Replace the replay-only event implementation in `crates/sase_gateway/src/routes.rs`
   with an atomic hub that owns a store generation, monotonically increasing sequence,
   bounded durable-invalidation replay ring, and a Tokio broadcast sender. Subscribe and
   capture replay state under one synchronization boundary so no
   snapshot-to-subscription race can lose an event. Publish each durable invalidation
   once to both ring and connected subscribers; when the generation changes, a cursor is
   ahead, the ring rolled over, deletion history is incomplete, or a broadcast receiver
   lags, emit `resync_required` with the same bounded authoritative snapshot/cursor.
   Generate heartbeats per connection without consuming sequence numbers or replay
   capacity, while retaining compatible payload behavior for the existing mobile
   `/api/v1/events` stream.

5. Wire the service into `GatewayState` and add authenticated fleet scopes/routes for
   summary, catalog, batch, detail, content, project eligibility, and events. Reuse the
   existing uniform bearer authentication and protocol-version negotiation on every
   route, apply scope checks before service work, preserve the fleet request body cap,
   map validation/not-found/stale/timeout/resync failures to stable `ApiErrorWire`
   responses, and include authoritative counts/count revision in `hello`. Update
   `crates/sase_gateway/src/wire.rs`, `fleet_auth.rs`, `lib.rs`, `contract.rs`, and the
   committed `contracts/api_fleet_v1/fleet_api_v1.json` snapshot so route, scope,
   record, limit, error, content, and cursor semantics are reviewable and cannot drift.

6. Add deterministic gateway tests with temporary SASE homes, project specs, artifact
   trees, and rebuilt index fixtures. Prove summary work/counts are independent of page
   size; stable paging has no duplicates; a followed logical ID outside page one is
   batch-readable; detail and every JSON error leak neither paths nor PIDs; content
   handles reject traversal/revision mismatch and support capped continuation reads of
   growing files with stable per-read digests; eligibility is bounded; and missing,
   wrong-scope, revoked, or protocol-incompatible credentials are rejected before read
   work. Add event fault tests for post-connect publication to multiple consumers,
   reconnect replay, ring rollover, receiver lag, generation replacement, incomplete
   deletion history, snapshot/cursor consistency, process-exit reconciliation, refresh
   coalescing/failure retention, and heartbeats leaving replay capacity and sequence
   unchanged.

## Verification and completion

- Run focused `sase_core` fleet-contract tests and `sase_gateway` tests during
  implementation, then run `just check` from the linked `sase-core` repository as the
  mandatory full gate.
- Inspect both the linked repository and primary Python repository worktrees. Leave the
  primary repository unchanged unless an unavoidable caller/binding change was made; if
  it was, read that repository's verification memory and run its required install and
  `just check` workflow too.
- Do not create task beads. Record any genuinely out-of-scope discovery on `sase-xe.5`
  as `PROPOSED FOLLOW-UP: ...`.
- Before closing, run `sase bead epic-symbols sase-xe.5`; resolve every symbol or re-key
  its Justfile owner to `sase-xe` or a still-open later phase. Close only `sase-xe.5`
  with a note naming the focused tests, event/content fault cases, contract snapshot,
  and `just check` result. Do not close the parent epic or any ancestor.
