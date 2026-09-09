---
tier: tale
title: Complete reduced artifact-link event readers and maintenance
goal: Artifact-link read, projection, health, rename, and managed-conflict paths consume
  validated reduced events plus pending operations without regressing pre-cutover
  legacy indexes.
size: medium
proposed_by: bbugyi200.athena.sase-yy.5
bead: sase-yy.5
status: done
---

- **PARENT:** [202609/artifact_link_events_v2.md](artifact_link_events_v2.md)
- **BEAD:**
  [sase-yy.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.5.md)

# Complete reduced artifact-link event readers and maintenance

## Goal

Complete phase `sase-yy.5` by making artifact-link reads, aggregate projections, health
inspection, rename repair, and managed Markdown conflict recovery consume the
Rust-reduced immutable event set plus locally pending schema-v2 outbox operations, while
preserving the legacy v2-index path needed before the later cutover phase.

## Context and invariants

- The Rust event contract and deterministic reducer are already exposed through
  `sase_core_rs`; Python must remain a storage/presentation adapter and must not
  duplicate reducer or alias semantics.
- Immutable objects live below `link-events/v1/<prefix>/<digest>.json`. Readers must
  validate object bytes against their paths, deduplicate endpoint copies through the
  Rust reducer, and surface corruption rather than silently accepting it.
- Before `sase-yy.6` imports legacy indexes, legacy v2 rows and newly emitted event rows
  coexist. Merge them without double-counting and fail closed if the supposedly disjoint
  legacy/event lanes contain the same stored edge.
- Queued schema-v2 events returned by `pending_artifact_link_outbox_events` are part of
  local truth immediately. The `--source store` view excludes projected rows, while the
  aggregate/index view includes pending, durable, and freshly projected rows.
- Published event files are immutable. A rename records an `alias` event and applies
  aliases during reduction; it never rewrites an event object. Legacy index rewrites
  remain available until cutover.
- Managed Markdown recovery may regenerate managed blocks only when the authored body is
  identical across the conflict. Authored-body conflicts remain unclaimed and fail
  through the normal document-conflict path.

## Implementation

1. Add a focused artifact-link event-store adapter under `src/sase/sdd/` that: discovers
   canonical event paths under each configured document sidecar; validates and
   canonicalizes each object with `sase_core_rs`; reduces durable events together with
   pending outbox events; exposes reduced rows, aliases, per-object validation findings,
   orphaned tombstones, and deterministic pending-age statistics needed by callers. Keep
   filesystem/order concerns in Python and all graph semantics in Rust.

2. Route `ArtifactLinkStore` truth reads through that adapter. Update local and
   cross-workspace aggregate previews/rebuilds, `load_artifact_rows`,
   `load_durable_rows`, and the durable-sidecar publication view so durable events and
   pending events participate exactly once alongside bead rows, legacy v2 rows, prior
   aggregate retention, and projected rows. Add an explicit pre-cutover overlap guard
   for legacy/event stored edges and preserve existing behavior when no event objects or
   schema-v2 pending entries exist.

3. Update neighborhood and CLI list behavior through the store facade, keeping
   `link list --source store` on reduced durable-plus-pending truth and `--source index`
   on the rebuilt aggregate including projected rows. Add focused tests for duplicate
   endpoint event objects, event ordering, pending visibility, legacy-only parity,
   overlap rejection, alias resolution, and projected-row source separation.

4. Extend artifact-link health/doctor reporting with event-object validation failures,
   orphaned tombstones, alias-cycle/conflict failures, pending queue count/age including
   p95 age, and unpublished publication-ledger aging per sidecar root. Render these in
   the existing doctor table and cover healthy and unhealthy reports without weakening
   existing link/index checks.

5. Change rename handling for the event lane to enqueue a canonical alias event with
   stable operation identity and let reduction update late old-ref events. Preserve
   legacy index and aggregate repair until cutover, and reuse the machine-lane/outbox
   API supplied by the adjacent publisher work rather than writing event paths from an
   agent checkout. Cover chained aliases, a late old-ref observation, and flag-off or
   legacy compatibility behavior.

6. Extend the semantic conflict resolver with a narrowly claimed Markdown resolver.
   Compare the staged base/ours/theirs authored bodies using the existing managed-block
   safety helpers; only when authored content agrees, load reduced link truth and
   regenerate the managed `Links`/`Referenced By` blocks, write and stage deterministic
   bytes, and continue the existing resolver chain. Add tests proving managed-block-only
   conflicts resolve in both rebase paths and authored-body conflicts remain untouched
   with an informative failure.

7. Reconcile with any concurrently landed `sase-yy.4` publisher/flag interfaces before
   final verification, remove or re-key this phase's Justfile epic-symbol allowance, and
   avoid changes owned by the cutover/import or acceptance phases.

## Verification and completion

- Run the focused artifact-link store, outbox, neighborhood, CLI list/doctor, rename,
  refresh, and semantic-conflict test modules while iterating.
- Read the required lint/test reference memory after tracked changes, run its mandated
  formatting and verification workflow, and finish with `just check`.
- Run `sase bead epic-symbols sase-yy.5`; resolve every reported symbol or re-key a
  still-needed exemption to an open later bead.
- Close only `sase-yy.5` with a note naming the focused coverage and final verification;
  leave `sase-yy` and all ancestor/other phase beads open.
