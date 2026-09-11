---
tier: tale
title: Complete bounded fleet catalog snapshots and explicit history
goal:
  Default fleet reads stay bounded while explicit history paging and page merging use
  genuine, trustworthy snapshot identities.
size: medium
proposed_by: bbugyi200.athena.sase-xe.16.11.7.14.6.3
bead: sase-xe.16.11.7.14.6.3
create_time: 2026-09-10 22:23:34
status: wip
---

- **PARENT:**
  [202609/fleet_remaining_acceptance.md](https://github.com/sase-org/sase--plans/blob/main/202609/fleet_remaining_acceptance.md)
- **BEAD:**
  [sase-xe.16.11.7.14.6.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.14.6.3.md)

# Complete bounded fleet catalog snapshots and explicit history

## Objective

Complete phase `sase-xe.16.11.7.14.6.3` without closing its parent or any ancestor. Make
the default fleet catalog serve only active/protected work plus the existing
200-row/seven-day recent-terminal presentation, add a deliberately requested and lazily
built history catalog that can page records outside that bound, and make every catalog
continuation and shared page merge depend on a genuine opaque snapshot identity rather
than the event-store generation or argument order.

The current prerequisites are already on the clean trees: core `381b643` makes
owner-produced display intent safe, core `4eec518` completes dismissal reconciliation,
and SASE `49cddaba3` surfaces dismissed-index sync failures. Preserve all of that work.
Do not add a feature flag, mutate release versions, duplicate presentation logic in
Python, create a task bead, or close an ancestor.

## Current seams and contract choices

- `crates/sase_gateway/src/fleet_reads.rs::build_snapshot_blocking` currently scans a
  bounded index window, applies `decide_fleet_presentation`, and caches only the
  resulting presentation vector. `FleetReadService::catalog` can therefore page only
  that vector; `include_terminal=true` cannot reach excluded archive records.
- `FleetCatalogQueryWire` and its page types in `crates/sase_core/src/fleet_contract.rs`
  expose a bare `off:<n>` cursor. The response carries an event-store `StoreCursorWire`,
  but that generation belongs to the event store lifetime and does not identify a
  rebuilt served set.
- The shared contract must use two explicit scopes: `presentation` (the default) and
  `history`. Retain `include_terminal` as the terminal-row filter within the selected
  scope so current filters continue to work; a deliberate history request uses the
  history scope with terminal rows enabled.
- Define an opaque, validated snapshot ID from a deterministic canonical digest of the
  scope and sorted served summaries (identity, resource revision, and stable projected
  content; exclude only read-time freshness/observation stamps). It must change for an
  equal-count membership replacement and for a row aging out during an event-free
  rebuild, remain meaningful across a gateway restart with unchanged content, and work
  for the empty set. Never order or otherwise interpret opaque IDs.
- Replace bare offsets with a versioned opaque catalog cursor that encodes and validates
  scope, snapshot ID, and offset. A continuation whose ID/scope no longer matches the
  retained catalog snapshot returns a typed `resync_required`/restart page with no rows
  or continuation cursor; malformed cursors remain validation errors. Propagate the
  snapshot ID, scope, continuation state, and safe reset reason through catalog wire
  types and federation normalization.
- Keep authoritative host counts tied to the bounded presentation snapshot and
  independent of catalog page size/scope. History pages must not redefine the normal
  machine summary as the archive size.

## Implementation

1. Extend the transport-free fleet catalog model in
   `crates/sase_core/src/fleet_contract.rs` and its exports in
   `crates/sase_core/src/lib.rs`:
   - Add the typed presentation/history scope and snapshot identity fields to the query,
     catalog page/continuation, normalized-host, and relevant authoritative/summary
     response wires. Use serde defaults only where they preserve a safe presentation
     request; keep `deny_unknown_fields` validation and all label/path/secret bounds.
   - Encode/decode snapshot-bound continuation cursors centrally. Update
     `validate_fleet_catalog_query`, cursor validation, page selection, federation
     payload normalization, and diagnostics so a genuine snapshot mismatch is a typed
     restart outcome rather than an untyped failure or silent union.
   - Add a transport-free catalog accumulation request/state/decision API. It must take
     explicit caller request-generation evidence plus the requested cursor/snapshot
     evidence and incoming page. Older request generations are ignored; a newer initial
     request replaces the entire prior state; same-snapshot continuations union by
     logical/exact identity without duplicates and retain the highest trustworthy row
     revision; a conflicting continuation requests restart. Replacement clears absent
     rows and all obsolete cursor/count inputs, including when the replacement is empty.
     This API must never infer that the second argument is newer or compare snapshot IDs
     lexicographically.

2. Refactor `crates/sase_gateway/src/fleet_reads.rs` around two catalog sources while
   preserving the existing cache guarantees:
   - Keep the current age-rebuilt, coalesced presentation snapshot as the source for
     hello/summary, normal catalog reads, followed/detail behavior, and authoritative
     counts. Compute and stamp its deterministic presentation snapshot ID after sorted
     projection.
   - Add a separate history catalog cache and refresh lock that are first touched only
     by an explicit history-scope catalog request. Build it off the Textual/client path
     with `AgentArtifactIndexQueryWire::include_full_history=true`, reuse the same safe
     record resolution, dismissal-lineage, liveness/protection, and per-row failure
     behavior, but do not apply the 200-row/seven-day terminal exclusion. Continue to
     exclude hidden/dismissed dead lineage while preserving potentially live, Unknown,
     waiting, and pending-question records.
   - Give the history cache the same timeout, coalescing, age revalidation, and
     retain-previous-on-error behavior as presentation. A default catalog/summary call
     must never build or scan it. Select pages against the cache matching the requested
     scope and return typed restart when a continuation names a snapshot no longer
     retained.
   - Keep network/index work lazy and bounded on ordinary requests. Do not add polling,
     eager startup work, or a second backend implementation.

3. Expose the new validation and catalog accumulation API as thin dict-in/dict-out PyO3
   bindings in `crates/sase_core_py/src/lib.rs`, following the existing
   `fleet_validate_catalog_query` and federation-normalization bindings. Register and
   test the functions, but leave replacement of SASE's Python-only merge helper to the
   already-planned `viewer-integration` phase. Do not manually edit workspace/crate
   versions or path-dependency versions; release-plz owns them.

4. Update all gateway transport surfaces that construct or describe the changed typed
   query/page: routes, federation-worker fixtures, core/gateway public re-exports, the
   generated contract in `crates/sase_gateway/src/contract.rs`, and the committed
   `crates/sase_gateway/contracts/api_fleet_v1/fleet_api_v1.json` snapshot. Preserve
   project/status/text filters, page limits, per-host lazy continuation, authoritative
   counts, path/PID secrecy, and strict invalid-external-wire rejection.

## Regression coverage

Add focused Rust gateway/core and PyO3 tests that prove:

- More than 200 completed records and records older than seven days are excluded from
  presentation yet reachable through multiple explicit history pages, while ordinary
  summary/catalog calls never initialize the history scan/cache.
- An age-driven rebuild can change snapshot identity with the event cursor and count
  revision unchanged; equal-count membership replacement also changes it; an unchanged
  snapshot (including empty) has stable identity across rebuild/restart.
- Same-snapshot pages union without duplicates, and a repeated identity keeps the
  current/higher resource revision. A newer evidenced initial request replaces prior
  rows/counts/cursors, including with an empty snapshot.
- A late response from an older request generation is ignored; same event-store
  generation does not imply same snapshot; opaque IDs are never ordered; gateway
  restart, a rebuild during continuation, and a stale snapshot-bound cursor produce the
  documented stable/restart behavior without resurrecting removed rows.
- History and presentation cursors cannot cross scopes, malformed external cursors and
  snapshot IDs are rejected, filters and limits remain effective, and federation
  normalization carries typed restart state without discarding healthy hosts.
- Retain-previous-on-error freshness remains intact for both caches, and authoritative
  counts stay page- and history-scope-independent.

## Verification and bead completion

Read the applicable SASE verification, TUI-performance, and Symvision guidance before
finishing. In the core checkout, first run the focused fleet/gateway/PyO3 tests while
iterating, then run the repository-required full `just check` (or `./scripts/check.sh`),
including bindings. If the known Python shared-library loader omission appears, derive
the selected Python interpreter's library directory and prepend it while preserving any
existing `LD_LIBRARY_PATH`; do not skip PyO3 tests or treat a local package-only run as
sufficient. No SASE tracked source should need changing in this phase; if it does, run
SASE `just install` as needed and `just check` as required.

Before closing, run `sase bead epic-symbols sase-xe.16.11.7.14.6.3`. Resolve every
returned symbol or re-key its Justfile entry to a still-open parent/later phase, then
rerun until clean. Record any out-of-scope discovery only as
`sase bead note sase-xe.16.11.7.14.6.3 'PROPOSED FOLLOW-UP: ...'`. When all acceptance
and verification above pass, close only this phase with
`sase bead close sase-xe.16.11.7.14.6.3 --note "<concise snapshot/history, contract, binding, and test evidence>"`.
Leave `sase-xe.16.11.7.14.6` and every ancestor open.
