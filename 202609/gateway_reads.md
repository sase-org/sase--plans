---
tier: tale
title: Complete bounded fleet reads and recoverable gateway events
goal:
  Serve safe owner-resolved fleet data from bounded in-process Rust queries and make
  event-stream recovery correct for connected and reconnecting clients.
size: medium
bead_id: sase-xe.5
proposed_by: bbugyi200.athena.sase-xe.5
bead: sase-xe.5
create_time: 2026-09-06 18:17:07
status: wip
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.5.md)

# Plan: Complete bounded fleet reads and recoverable gateway events

## Context

Phase `sase-xe.5` builds the read-only fleet protocol on the bounded local index work,
the portable identity/projection contracts, and the authenticated fleet namespace that
are already present. The implementation belongs in the linked `sase-core` repository:
transport-free validation and cursor/query records stay in `crates/sase_core`, while
filesystem ownership, asynchronous HTTP handling, caching, content resolution, and SSE
delivery stay in `crates/sase_gateway`. Fleet reads must never invoke the Python mobile
agent bridge and must never serialize local paths, PIDs, credentials, or other
owner-only resources.

The read surface must provide independently bounded authoritative counts, catalog pages,
batch lookup of followed logical IDs, lazy detail, bounded opaque content reads, and
project eligibility. The feed cursor is a store generation plus sequence and is separate
from per-row action revisions. Connected SSE clients must receive post-connect changes;
heartbeats must not consume replay-ring capacity. Local lifecycle changes that bypass
gateway mutations are repaired through a shared bounded reconciliation path.

## Implementation

1. Extend the transport-free fleet contract with strict request/response records for the
   summary, page, batch, detail, content-range, project-eligibility, snapshot, and event
   surfaces. Give every list operation explicit page/item/byte limits, stable ordering,
   cursor/revision metadata, and validation that rejects unbounded or malformed inputs.
   Reuse `ResolvedAgentSummaryWire`, `ResolvedAgentDetailWire`, `ContentHandleWire`,
   `StoreCursorWire`, `count_logical_agents`, and `classify_cursor_replay`; keep feed
   positions distinct from `ResourceRevisionWire`. Add focused core tests for bounds,
   deterministic paging/batch behavior, invalid locators/handles, and generation/replay
   decisions.

2. Add a gateway-owned fleet read service that derives its paths from the configured
   SASE home and calls `query_agent_artifact_index`/exact indexed record loads directly
   in Rust. Build bounded owner-resolved projections in one place: derive stable logical
   and exact locators, resolve process liveness on the owner, calculate row revisions,
   expose only normalized capabilities and opaque content handles, and maintain a small
   shared resolved snapshot for summaries. Run SQLite/filesystem work through
   `spawn_blocking` with explicit timeouts, coalesce concurrent refreshes, and preserve
   a bounded cached snapshot when revalidation cannot complete. Never route these
   handlers through `AgentHostBridge` or a subprocess.

3. Implement authenticated `/api/fleet/v1` read routes and their read scopes: hello's
   authoritative summary/count revision, bounded catalog paging with an independent
   query/filter contract, POST batch lookup for requested logical IDs, lazy per-agent
   detail, opaque content-handle range/offset reads, project eligibility, and fleet SSE.
   Validate protocol version and credential scope uniformly. Resolve content handles
   server-side only, fence them to the snapshot/row revision and canonical allowed file,
   cap every response, return digest/length/range metadata, and distinguish EOF from a
   growing file. Update public exports and the generated fleet API contract snapshot so
   route, scope, record, limit, error, and cursor semantics cannot drift.

4. Replace the replay-only event hub with one atomic publication path backed by both a
   bounded replay ring and a Tokio broadcast sender. Allocate one generation-plus-
   sequence cursor per durable invalidation, subscribe without a snapshot race, replay
   events newer than the supplied cursor, and return `resync_required` with the same
   bounded authoritative snapshot when the generation changes, the cursor is ahead, the
   ring rolled over, deletion history is incomplete, or a receiver lags. Deliver newly
   published mutations to every connected consumer. Emit heartbeats directly to each
   connection without adding them to the shared sequence or replay buffer, while
   preserving the existing mobile event stream's compatible payload behavior.

5. Cover non-gateway lifecycle changes with one shared, jitter-ready bounded
   reconciliation operation owned by the read service. Diff refreshed authoritative rows
   against the maintained snapshot and publish invalidations for local launches,
   lifecycle/attention transitions, deletions, and owner-observed process exits. Ensure
   concurrent SSE clients share refresh work and do not trigger one archive scan per
   connection; a failed or timed-out pass must retain cached data and make freshness or
   resync state explicit.

6. Add deterministic gateway tests using temporary SASE homes and index fixtures. Prove
   summary work stays bounded independently of page size; followed IDs outside page one
   remain batch-readable; lazy detail never leaks paths/PIDs; growing output supports
   capped continuation reads and stable digests; project eligibility is bounded; and
   unauthorized or wrong-scope callers are rejected. Add fault tests for post-connect
   publication to multiple SSE consumers, reconnect replay, replay-ring rollover,
   broadcast lag, snapshot/cursor consistency, generation replacement, incomplete
   deletion history, process-exit reconciliation, and heartbeats leaving replay capacity
   unchanged.

## Verification and completion

Run focused `sase_core` and `sase_gateway` tests while iterating, then run `just check`
from the linked `sase-core` repository as the required pre-commit gate. Inspect both
repository worktrees and avoid changing the Python repository or its core revision pin
unless a real Python caller/binding change became necessary; if it did, follow that
repository's required install and `just check` workflow as well. Do not create follow-up
beads: append any out-of-scope discovery to `sase-xe.5` as a `PROPOSED FOLLOW-UP:` note.

Before completion, run `sase bead epic-symbols sase-xe.5` and resolve every reported
symbol or re-key its `Justfile` ownership to the parent epic or a still-open later
phase. Close only `sase-xe.5` with
`sase bead close sase-xe.5 --note "<specific tests and fault cases verified>"`; do not
close `sase-xe` or any ancestor.
