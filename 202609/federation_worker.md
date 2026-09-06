---
tier: tale
title: Local federation worker and Python remote facade
goal: "Remote fleet consumers use a secure, on-demand Rust worker for bounded cached
  reads, host-isolated reconciliation, and restart-transparent access through a thin
  Python facade, while local-only SASE remains network- and worker-free.

  "
size: medium
proposed_by: bbugyi200.athena.sase-xe.10
bead: sase-xe.10
create_time: 2026-09-06 19:53:16
status: wip
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.10](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.10.md)

# Plan: Local federation worker and Python remote facade

## Scope and settled boundaries

Implement the `federation-worker` phase as one Rust/Python vertical slice. The Rust
runtime belongs in the `sase_gateway` crate family and consumes the existing fleet v1
HTTP/JSON and SSE contracts; it does not scan agent state itself, import Python, or add
Textual policy. Python resolves the already-enrolled dispatch configuration, provider
connection plans, and credential references, then communicates only through the local
IPC contract. Mutation admission, `%dispatch`, lifecycle actions, and Focus/Fleet
presentation remain owned by their later phases.

Keep local behavior genuinely unchanged: constructing or starting ACE with no enabled,
enrolled machines must not resolve providers, read credentials, spawn the worker, open
network connections, or install remote timers. A missing or crashed worker must never
break direct local reads and actions.

## Rust worker and local IPC contract

1. Replace the unused daemon socket skeleton in `crates/sase_gateway` with a narrowly
   scoped federation runtime and a dedicated `sase_federation_worker` binary. Reuse the
   existing host-local run-root/socket derivation, but add single-owner locking, stale
   socket recovery, signal-driven cleanup, and an idle timeout so an unused worker exits
   cleanly. The binary accepts explicit SASE-home/socket/idle settings for tests and
   supervision without enabling the mobile HTTP listener.
2. Define a versioned, length-prefixed JSON IPC envelope with request IDs, absolute
   deadlines, typed result/error codes, and a hard frame limit. Include operations for
   health/capabilities, replacing the validated host configuration, cached/refreshing
   summary, catalog, followed-ID batch, detail, content-range, project-eligibility, and
   orderly shutdown. Preserve fleet v1 response shapes so Python is an adapter rather
   than a second domain implementation.
3. Secure the endpoint before accepting requests: create the runtime directory with
   user-only permissions, reject unsafe pre-existing non-socket/symlink targets, bind
   atomically, set the socket to mode `0600`, and verify same-user peers on supported
   Unix platforms. Bound connections and in-flight work; malformed, oversized, expired,
   or unsupported-version requests return typed failures without taking down the
   listener.

## Host isolation, cache, and reconciliation

1. Maintain one host state keyed by pinned installation identity, with its own reused
   `reqwest` client, cancellation token/generation, request deadline, SSE cursor,
   exponential backoff, and health/freshness status. Reconfiguration replaces only
   changed hosts and cancels removed hosts. Validate every connection plan with the
   existing core contract, authenticate fleet requests with the resolved bearer token,
   negotiate the fleet protocol header, and quarantine an endpoint whose authenticated
   hello reports a different installation identity.
2. Route interactive requests through bounded per-host and global concurrency so one
   hung host cannot consume the fleet. Apply the caller's remaining deadline to queue
   admission, connect, body streaming, and JSON decoding. Backoff is per host, capped
   and jittered; an interactive read may use a still-valid cached projection while a
   failed/background refresh updates only that host's stale/offline metadata.
3. Cache authoritative summaries, bounded catalog pages, followed-ID batches, lazy
   details, and bounded content ranges by installation identity plus normalized request
   key/revision. Enforce explicit page, entry, and byte budgets with deterministic LRU
   eviction, never persist bearer credentials, and atomically persist the bounded
   projection cache under the local fleet state directory so restart/offline display can
   retain age and origin. Treat corrupt or newer cache files as host-local cache misses,
   not startup failures.
4. Start an SSE subscription only for configured demand. Track the generation-plus-
   sequence cursor, apply invalidations to the affected cache entries, replace state
   from `resync_required` snapshots, and periodically run bounded summary/followed-ID
   reconciliation to repair missed events. Reconnect with the host's backoff and keep
   every other host live. Load persisted follow state at worker startup/reconfigure so
   followed IDs get priority; tolerate an absent future dispatch-intent store and leave
   a versioned startup/recovery input for the reliable-launch phase rather than
   inventing mutation semantics here.

## Python supervision and facade

1. Add a small `sase.dispatch` facade whose public methods mirror the IPC read
   operations and return plain fleet wire mappings suitable for ACE and CLI callers.
   Build worker configuration from the phase-7 typed dispatch loader/provider helper and
   credential store, preserving aliases as presentation/routing metadata while pinning
   host state by installation ID. Reject missing credentials or invalid plans per host
   without exposing secrets in errors, logs, process arguments, or cache files.
2. Add a race-safe on-demand supervisor modeled on the mobile gateway's command
   resolution and health polling. Resolve the packaged worker first and linked-core
   development binaries second; spawn it detached from a single ACE callback, wait for
   its IPC health response, and let the worker's ownership lock collapse concurrent
   starters. Retry one idempotent read after a broken socket by re-probing/restarting;
   never replay a future mutation implicitly.
3. Make each async facade method perform socket I/O and JSON decoding through a worker
   thread so neither Textual's event loop nor serial message pump is blocked. Every
   method requires or derives a finite deadline, propagates cancellation by closing the
   IPC request, distinguishes deadline/unavailable/quarantined/stale results, and can
   explicitly request cache-only behavior for first paint. An empty registry returns an
   empty/disabled facade result without invoking supervision.
4. Package the standalone Rust executable with the `sase-core-rs` platform wheels and
   keep the development resolver compatible with `cargo build -p sase_gateway`. Update
   wheel smoke coverage to assert that the executable is installed and can answer a
   local health request. Update the main repository's core revision contract after the
   linked core change is finalized so CI and released SASE consume the same worker
   implementation.

## Verification

Add Rust unit and integration coverage for IPC schema/frame bounds, directory and socket
modes, peer rejection, ownership races/stale recovery, idle/signal cleanup, cache
budgets/eviction/persistence, per-host deadlines/backoff, and secret redaction. Use real
loopback fleet gateways for end-to-end summary/catalog/batch/detail/content reads and
SSE cursor replay/resync; include a hanging host beside a healthy host and a worker
restart mid-subscription to prove isolation and recovery.

Add Python tests for empty-registry laziness, typed config/credential projection,
command resolution, concurrent startup, health polling, one-read restart recovery,
deadline propagation, cache-only calls, async offloading, and failure mapping. Include a
process-level smoke that starts the packaged/development worker on a temporary Unix
socket, performs a facade health/read exchange, kills that worker, and observes the next
idempotent read restart it without affecting a local read.

Run focused Rust and Python suites during development. Then run the complete `sase-core`
`just check`, install the linked core build into the SASE environment as required by the
repository workflow, run the main repository's mandated `just check`, inspect both
worktrees for unintended generated changes, run `sase bead epic-symbols sase-xe.10`, and
close only `sase-xe.10` with a note naming the successful fault tests and verification
lanes.
