---
tier: tale
title: Publish artifact-link operations through the immutable event lane
goal:
  Automatic and explicit artifact-link mutations publish conflict-free immutable events
  through serialized hidden clones while legacy behavior remains available behind a
  default-off beta flag.
size: medium
proposed_by: bbugyi200.athena.sase-yy.4
bead: sase-yy.4
status: done
---

- **PARENT:** [202609/artifact_link_events_v2.md](artifact_link_events_v2.md)
- **BEAD:**
  [sase-yy.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.4.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-yy.4](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-yy.4.md)
- **COMMITS:**
  - [1842f29](https://github.com/sase-org/sase-core/commit/1842f29c00a9288f90d8cbc898ef8c8a40731c85)
    — feat(beads): support link operation ids

# Publish automatic artifact links as immutable events

## Goal

Complete phase `sase-yy.4` by putting automatic and explicit artifact-link mutations
through the immutable schema-v1 event lane while the `link_events` beta flag is enabled.
No enabled-path writer may dirty an agent checkout's `links/**` indexes. A serialized
publisher must materialize content-addressed events in the owning hidden document
sidecars, commit them through the existing artifact-link choke point, acknowledge the
outbox only after every owning clone contains the event in `HEAD`, and leave remote push
failures to the existing artifact-link publication ledger. The disabled path must retain
the current legacy-index behavior.

## Current state and boundaries

- The Rust event contract and Python bindings already provide canonicalization,
  canonical JSON, SHA-256 digest/path derivation, byte/path validation, and
  deterministic reduction. The Python dependency floor already includes that contract.
- `artifact_link_outbox.py` already writes schema-v2 entries with full 128-bit operation
  IDs and frozen canonical event payloads. Its current drain intentionally retains event
  entries and publishes only their legacy row projection; this phase replaces that
  enabled-path placeholder without changing schema-v1 queue compatibility or the
  release-evidence/stale-terminal/machine-writability partitions.
- Hidden host-owned document clones are resolved by
  `resolve_machine_artifact_link_store`; `commit_artifact_link_indexes` is the existing
  scoped commit/publication/retry choke point, but its file contract currently
  recognizes only `links/**/*.json` indexes.
- `sase-yy.5` owns general event-store reads, aggregate/neighborhood reduction,
  projections, doctor checks, alias-driven rename behavior, and managed Markdown block
  repair. Keep those concerns out of this phase except for the narrow event metadata
  lookup required to construct correct edge-put/remove operations.
- Preserve bead-store ownership for `bead:` endpoints and carry the document event's
  operation ID through the bead mutation adapter. If the installed Rust bead API cannot
  do this idempotently, extend `sase-core` and its Python binding additively, add
  focused Rust tests, update the binding contract/floor in this repository, and run both
  repos' required verification.

## Implementation

1. Add the `link_events` beta gate and explicit both-state coverage.
   - Scaffold the registry definition using the canonical feature-flag workflow with a
     default-off beta definition whose enabled/disabled/removal prose names this epic's
     event cutover. Do not hand-author a registry member.
   - Centralize the consumer check behind a small artifact-link helper and exercise both
     override states. With the flag off, keep all existing index-writing and
     outbox-drain behavior byte-for-byte compatible. With it on, route the writers and
     publisher below to events.

2. Introduce a thin immutable-event storage/publishing adapter in `src/sase/sdd/`.
   - Build canonical bytes, digest, and `link-events/v1/<prefix>/<digest>.json` solely
     via `sase_core_rs`; do not reproduce validation, canonicalization, hashing,
     relation direction, or reduction semantics in Python.
   - Determine each document-owning sidecar from the canonical event endpoints. Write
     the same event object to both document roots when both endpoints are documents;
     agent/stitch-only pairs remain aggregate-only, and bead endpoints remain bead-store
     owned in addition to any document-side copy.
   - Create files atomically. An existing path with identical canonical bytes is an
     idempotent success; different bytes at the same content-addressed path are reported
     as corruption and must never be overwritten. Track which owning roots have the
     exact event in `HEAD`, so replay after a crash between create, commit, and outbox
     acknowledgement safely finishes missing work without duplicating an operation.
   - Serialize each per-project publisher with an outer worker lock and acquire each
     repository's non-reentrant store write lock only inside it. Hold the store lock
     over event creation plus the scoped commit and hand it into the commit helper,
     preserving the established worker-lock -> store-lock order.

3. Extend the commit choke point to accept canonical event objects safely.
   - Add an event-object classification to `_artifact_link_files.py`; accept only
     regular, non-symlink files at a Rust-derived canonical path whose exact bytes pass
     the Rust validator. Preserve the current rejection behavior for malformed indexes,
     events, locks, and unrelated files.
   - Generalize `commit_artifact_link_indexes` without weakening its path scoping, lock
     handling, or legacy callers. Stage event paths, commit at most one batch per owning
     repository, use machine mutation authorization, and expose enough per-root outcome
     to decide which operations are locally durable.
   - For machine event batches, verify publication with the project/role context. A
     failed or lock-deferred push must register the committed head in the existing
     sase-yh retry ledger, while the locally committed event can still be acknowledged
     from the outbox. Do not add another retry journal or an unbounded publisher loop.

4. Make schema-v2 outbox events publish through the new adapter when the flag is on.
   - Preserve selection by agent, release-evidence gating for agent-originated automatic
     rows, stale-terminal audit/drop behavior, machine-root authorization, and schema-v1
     legacy drain semantics.
   - Permit trusted machine derivation/backfill events to publish without borrowing an
     unrelated agent run's release evidence, while keeping read/prompt events bound to
     their exact `(run_id, agent_name)` evidence.
   - Batch eligible event entries, publish each operation to every owning document root,
     apply the same operation once to any bead endpoints, and acknowledge an operation
     only after all required document copies and bead events are locally committed.
     Entries survive any pre-commit failure; committed-but-unacknowledged entries replay
     idempotently. Return honest event paths/counts in the drain report without claiming
     remote publication merely because the local commit exists.
   - Update the commit-workflow kick and hourly chop drain to resolve the hidden machine
     store before enabled-path publication. The finalizer may kick this publisher but
     must not auto-commit agent-checkout link indexes for enabled event-lane operations.

5. Route every in-scope writer through one operation API.
   - Audited reads remain enqueue-only and report queued state; ensure the enabled path
     never writes an agent sidecar index and the disabled path still drains to the
     legacy projection.
   - For prompt-reference/derived candidates and backfill, mint stable event operations,
     enqueue them before publication, update only the machine-local pending view, and
     publish through the hidden-clone drain. Preserve best-effort error reporting and
     bounded backfill/checkpoint behavior.
   - Switch `sase artifact link add` and `rm` (including create/plan inlet helpers that
     share their manual contract) to edge-put/edge-remove events when enabled. Gather
     observed predecessor operation IDs from canonical event metadata, publish
     synchronously through the machine lane, and report success only after local commit
     plus the existing publication verification succeeds. The disabled path keeps the
     current synchronous legacy behavior.
   - Ensure a bead/document mutation uses one operation ID end to end. Replaying the
     same outbox operation must not append/increment a second bead mutation; a distinct
     read operation must remain distinct.

## Verification

- Add focused unit tests for canonical event path classification, identical replay,
  same-path/different-bytes corruption, per-root durability, and crash points after file
  creation, after commit, and before acknowledgement.
- Extend outbox tests for disabled legacy parity and enabled event batching, distinct
  repeated observations, agent selection/release evidence, trusted machine derivation,
  stale/drop retention, dual-document copies, bead/document operation identity, and no
  agent-checkout dirt.
- Extend commit/retry tests so an event-only hidden-clone commit registers a failed
  push, is retired from the outbox after local commit, and is later published by
  `sweep_artifact_link_publication_retries`.
- Add both-state writer tests for audited reads, derivation/backfill, finalizer
  behavior, and synchronous manual add/remove. Extend the hidden-clone end-to-end
  fixture to prove the event commit lands in the hidden clone and the primary sidecar
  remains untouched.
- If `sase-core` changes, run its repository gate first. In this repository, follow the
  mandatory lint/test memory, run targeted suites during development, then run
  `just check` from a clean understanding of any pre-existing failures.
- Before closing `sase-yy.4`, run `sase bead epic-symbols sase-yy.4` and resolve or
  re-key every remaining phase symbol to an open bead. Close only `sase-yy.4` with a
  note naming the exact checks and both-state/hidden-clone/retry behavior verified; do
  not close the parent epic or any ancestor.
