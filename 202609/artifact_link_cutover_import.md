---
tier: tale
title: Fence and import legacy artifact-link indexes
goal:
  Cut artifact-link storage over to immutable events with a deterministic, resumable
  baseline import and no regression in publication recovery.
size: medium
proposed_by: bbugyi200.athena.sase-yy.6
bead: sase-yy.6
create_time: 2026-09-10 10:19:06
status: wip
---

- **PARENT:**
  [202609/artifact_link_events_v2.md](https://github.com/sase-org/sase--plans/blob/main/202609/artifact_link_events_v2.md)
- **BEAD:**
  [sase-yy.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.6.md)

# Fence, import legacy artifact-link indexes, and cut over

## Outcome

Complete phase `sase-yy.6` by adding an operator-authorized, deterministic cutover from
mutable `links/**/*.json` indexes to immutable `link-events/v1/**` truth. The command
must preview safely by default, fence capable writers, import the frozen legacy graph as
one canonical baseline, make event publication unconditional, retain the semantic legacy
resolver for stragglers, and keep event-only unpublished heads covered by the existing
publication retry ledger.

The implementation stays in the SASE Python storage/CLI layer because the Rust event
contract and reducer already support `baseline-import`; do not duplicate or weaken those
semantics in Python. Do not modify or close the parent epic. This phase is explicitly
authorized to close only `sase-yy.6`, so the separate `sase-z0` flag task must not be
closed here; record its required cleanup as a `PROPOSED FOLLOW-UP:` note on this phase
after removing the code flag.

## Implementation

1. Add a cutover-state adapter for the versioned `link-events/STORE.json` marker. Give
   the marker a strict schema containing the event-store capability/version floor,
   project identity, deterministic import identity, every participating sidecar role's
   frozen HEAD and legacy `links/` tree identity, and the baseline event identity.
   Support explicit `fenced` and `imported` states so an interrupted apply can resume
   without exposing an empty post-cutover graph. Validate that configured roots agree,
   fail closed on malformed/mixed markers, and use canonical JSON bytes. A marker cannot
   stop an older binary that ignores it; surface that limitation in command help/output
   and doctor diagnostics.

2. Build a deterministic import planner over the hidden machine-owned document sidecars.
   Under the artifact-link publication lock, freeze clean authoritative HEADs, read and
   validate every legacy v2 companion index, and deduplicate endpoint copies by the Rust
   relation-aware row identity. Preserve each row's `uses`, description, origin,
   creator, and timestamp exactly; accept byte/logically identical copies and fail
   closed when copies of one edge disagree. Derive the import id, 128-bit operation id,
   source-head summary, and event timestamp solely from the frozen inputs so the same
   heads yield byte-identical canonical baseline-event bytes. Reduce the baseline event
   through `sase_core_rs` and require its rows to equal the normalized legacy snapshot
   exactly before any mutation.

3. Add a resumable apply state machine that keeps event publication paused for the
   entire fence/import/finalize sequence. Preview reports roots, frozen heads, legacy
   tree identities, unique/duplicate row counts, baseline event path/digest, queued
   legacy outbox entries, and every refusal without writing. Apply must: convert any
   valid schema-v1 row-only outbox records into stable schema-v2 event records without
   losing release-evidence metadata; publish `fenced` markers through the machine lane;
   publish the baseline event synchronously to every owning root; verify durable
   reduction parity; then publish `imported` markers and rebuild the aggregate before
   releasing the lock. Preserve recoverable fenced state and the existing per-root
   publication retry record on failure. Reapplying an identical completed import is a
   clean no-op; changed source heads, dirty roots, malformed v1 queue records,
   inconsistent markers, or a reused import/event identity with different content are
   hard refusals.

4. Register `sase artifact link import-indexes` in alphabetic help/dispatch order. Keep
   it preview-only by default, use `-a/--apply` as the explicit assertion that every
   link-writing machine runs a capable release, and add `-j/--json` for a stable
   machine-readable report. No option is required. Make the help explain the fleet
   prerequisite, the old-binary limitation, resumability, and the fact that legacy
   indexes are retained as read-only history.

5. Enforce the fence and post-import read model throughout the store adapter. Direct
   legacy sidecar upsert/removal helpers must refuse once a valid fence exists. Before
   import completes, readers continue the current legacy-plus-event behavior; after an
   `imported` marker, aggregate/neighborhood/durable-truth paths ignore raw legacy index
   rows and use the baseline plus subsequent durable/pending events. Keep the existing
   semantic `links/**/*.json` conflict resolver wired for old binaries and history.
   Extend artifact doctor health with marker consistency/capability findings and a loud
   post-import straggler finding whenever the current legacy `links/` tree differs from
   the frozen tree recorded in the marker.

6. Graduate the event lane and delete obsolete compatibility branches. Make manual link
   operations, inlets, derivation/backfill, audited-read outbox draining, finalization,
   workflow publication, and rename repair use the event lane unconditionally. Remove
   `artifact_link_event_flags.py`, the `FeatureFlag.link_events` registry/schema entry,
   both-state tests, the finalizer's legacy link-index commit path, and general
   schema-v1/row-only outbox drain support after the cutover importer owns the one-time
   conversion. Keep fixture-only legacy index construction available for import,
   straggler, and resolver tests.

7. Re-examine the hourly backfill/retry chop without creating a second ledger. Keep the
   generic per-root unpublished-HEAD discovery, backoff, aging, and state for existing
   unpublished legacy heads. Retain derivation backfill only as an event producer, and
   remove or rename comments/state paths only if they incorrectly claim mutable indexes
   are the publication unit. Extend coverage to prove marker and event-only commits are
   discovered, registered on push failure, and republished by the existing sweep after
   legacy-index discovery is no longer part of normal writing.

## Verification

- Add focused parser/handler tests for sorted help, dry-run purity, JSON output,
  explicit apply semantics, completed-import idempotence, and every refusal path.
- Add cutover tests for deterministic baseline bytes, endpoint-copy deduplication,
  conflicting-copy rejection, preserved counts/provenance, v1-outbox conversion, crash
  recovery at each marker/event boundary, and exact reducer parity.
- Add store/doctor tests proving legacy rows are ignored only after completed import,
  direct legacy writes are fenced, a post-import old-writer modification is unhealthy,
  and the legacy semantic resolver still repairs straggler conflicts.
- Update flag-consumer, outbox, finalizer, derivation, inlet, publisher, retry, and chop
  tests for unconditional event behavior, including failed event-only publication
  followed by sweep-driven recovery with no duplicated operation.
- Run the focused test modules while iterating. Because tracked SASE files change, read
  the required lint/test memory before final verification, run `just install` if this is
  a fresh workspace, and finish with `just check`.
- Before closing, run `sase bead epic-symbols sase-yy.6`; resolve every remaining symbol
  or re-key its Justfile entry to the still-open parent/later phase. Record the
  `sase-z0` close requirement as a `PROPOSED FOLLOW-UP:` note, then close only
  `sase-yy.6` with a note naming the deterministic import, cutover, doctor, retry, and
  `just check` evidence.
