---
tier: tale
title: Reduce artifact-link event unions before projection
goal:
  Reconciliation and bead endpoints converge to one lossless Rust-reduced event truth.
size: medium
proposed_by: bbugyi200.athena.sase-yy.8.3
bead: sase-yy.8.3
status: done
---

- **PARENT:**
  [202609/artifact_link_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202609/artifact_link_landing_repairs.md)
- **BEAD:**
  [sase-yy.8.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.8.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-yy.8.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-yy.8.3.md)
- **COMMITS:**
  - [840824c](https://github.com/sase-org/sase/commit/840824c5bb71a9d78e46ee446625f47cdea0b7d4)
    — feat(sdd): reconcile artifact link event unions

# Reduce artifact-link event unions before projecting rows

## Objective

Complete phase `sase-yy.8.3` by making the immutable artifact-link event set the
authoritative reconciliation input and by making bead endpoint projections converge to
the same Rust-reduced truth. Preserve pre-import legacy ownership, machine-local pending
visibility, existing aggregate-only/projected-row behavior, bead-store ownership, and
legacy CLI compatibility.

## Context and invariants

- Cross-workspace reconciliation currently reduces each clone separately and then
  deduplicates rows. That loses distinct observations, tombstones, supersession, and
  aliases before the clone sets meet.
- Immutable events must be validated, unioned by operation identity across eligible
  stores, overlaid with this machine's pending operations exactly once, and reduced by
  the existing Rust reducer. Duplicate event copies and traversal order must not affect
  the logical rows; conflicting bytes for one operation ID must still fail closed.
- Legacy indexes remain inputs only before their explicit import fence. Bead rows remain
  bead-owned compatibility inputs, but event-reduced rows win for identities covered by
  event truth. Projected rows remain freshly recomputed and aggregate-only prior rows
  remain carried forward only when no consulted authoritative source can disprove them.
- Bead projection must reflect the reducer's add-wins, alias, supersession, and count
  results rather than applying each arriving event as an imperative row update.
  Projection retries must be idempotent, and one baseline event containing multiple
  edges for the same bead must retain every edge instead of treating the operation ID as
  a holder-wide receipt.
- Shared reduction and bead mutation/receipt semantics belong in `sase-core`; Python is
  limited to validated filesystem collection, projection orchestration, locking, and
  commit glue. Do not create a second logical link operation or publication retry lane.

## Implementation

1. Extend the Python event-store snapshot contract to retain the validated immutable
   durable and pending event inputs alongside its existing reduced rows and diagnostics.
   Keep strict local corruption behavior and the existing best-effort treatment of an
   unreadable sibling store.
2. Rework cross-workspace reconciliation and its durable-sidecar health view to collect
   validated durable event operations from every eligible store, include local pending
   operations once, reduce the complete union through `sase_core_rs`, and only then
   combine the reduced rows with explicitly owned pre-import legacy rows, bead rows,
   aggregate carry-forward rows, and recomputed projected rows. Ensure event truth wins
   identity collisions without using row upsert/counter increment as a merge.
3. Add the smallest Rust bead-projection mutation/receipt contract needed to install an
   exact reduced link state (or its absence) for a bead endpoint. Scope idempotency to
   the operation plus projected edge/direction so a baseline operation can project
   several edges to one holder, while exact replay remains a no-op. Expose the contract
   through PyO3 and update the binding contract validator without manually changing
   release-managed crate versions.
4. Change the Python bead adapter to reduce the full durable event union, including the
   current publication batch, derive the desired bead-owned neighborhoods for every
   affected original or alias-resolved edge, and apply exact Rust projection mutations.
   Commit once after the batch and acknowledge only after the bead store is clean and
   durable. Preserve unrelated legacy bead links and existing public manual bead-link
   behavior.
5. Add focused regression coverage for disjoint unsynchronized observations, duplicate
   endpoint copies and traversal order, remove-before-add, concurrent unseen additions,
   stale late publication, pending-plus-durable replay, alias plus late old-ref reads,
   repeated bead/document observations, a multi-edge baseline touching one bead, and
   replay after bead mutation/commit interruption. Assert store list/aggregate and bead
   neighborhood surfaces agree with Rust reduction and that retries do not inflate
   counts.

## Verification

- Run focused Python reconciliation, event-store, publisher, bead, and acceptance tests.
- Run the full `sase-core` repository gate (`just check`), including PyO3 binding tests.
- Install/use the matching local Rust binding as required by the repository workflow,
  run `tools/validate_sase_core_rs`, then run this repository's required `just check`.
- Inspect both repository diffs and working trees for unrelated changes.
- Run `sase bead epic-symbols sase-yy.8.3`, resolve or re-key every remaining entry, and
  close only `sase-yy.8.3` with a note naming the verified event-union and
  bead-projection guarantees. Leave `sase-yy.8` and all ancestors open.
