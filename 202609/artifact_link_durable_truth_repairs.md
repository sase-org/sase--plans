---
tier: epic
title: Finish artifact-link durable truth and publication recovery
goal: Artifact-link operations retain replayable history, every projection converges
  from that history, synchronous retries verify publication, and the pinned core exposes
  every required API.
parent_bead: sase-yy.8
phases:
- id: binding_baseline
  title: Restore the required core revision baseline
  depends_on: []
  description: 'binding_baseline: ratchet the core revision to include existing projection
    and cutover APIs and verify the exact pinned build.'
  size: small
- id: bead_history
  title: Persist immutable history for bead-owned link operations
  depends_on:
  - binding_baseline
  description: 'bead_history: require canonical durable operation history for bead-only
    publication and consume it during reduction and replay.'
  size: medium
- id: projection_convergence
  title: Repair bead projections from complete event truth
  depends_on:
  - bead_history
  description: 'projection_convergence: separate occurrence deduplication from state
    projection so old receipts cannot suppress a necessary converged projection.'
  size: medium
- id: reader_causality
  title: Accept valid out-of-order tombstones on read surfaces
  depends_on:
  - projection_convergence
  description: 'reader_causality: keep missing-predecessor diagnostics without rejecting
    Rust-valid event sets or reviving legacy rows.'
  size: small
- id: synchronous_recovery
  title: Verify remote publication on unchanged CLI and import retries
  depends_on:
  - binding_baseline
  description: 'synchronous_recovery: retry or report outstanding publication even
    when the requested link or final cutover marker already exists locally.'
  size: medium
- id: acceptance
  title: Prove durable history and recovery through production paths
  depends_on:
  - binding_baseline
  - bead_history
  - projection_convergence
  - reader_causality
  - synchronous_recovery
  description: 'acceptance: cover all five reproduced failures with production-path
    regression tests and verify the combined pinned build.'
  size: medium
proposed_by: bbugyi200.athena.sase-yy.8.land--1
create_time: 2026-09-11 06:54:35
status: wip
bead_id: sase-yy.8.6
---

- **PROMPT:** [prompts/202609/artifact_link_durable_truth_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/artifact_link_durable_truth_repairs.md)
- **BEAD:** [sase-yy.8.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.8.6.md)

# Remaining artifact-link durability repairs

This is the remaining-work child of `bead:sase-yy.8`, which remains open beneath
`bead:sase-yy`. All five previous repair phases are closed, but the landing audit
confirmed five product failures and a missing required core revision ratchet. Implement
only the repairs below. This plan's `parent_bead` is the handoff back to the interrupted
landing; ancestor closure and ancestor plan-status edits are land-agent actions.

Read the governing requirements through audited artifact reads:

- `plan:202609/artifact_link_events_v2.md` — original immutable-event guarantees.
- `plan:202609/artifact_link_landing_repairs.md` — prior repair requirements.
- `file:explicit:ad257e3976535af8057b15c5` — audit and follow-up dispositions.
- `file:explicit:5c0b68b9f1d02f0c4685ab56` — corrected temporary pytest probes.
- `file:explicit:a7d62bf742d8d718c4494088` — all five corrected product failures.

Use `sase artifact read <ref> "<reason>"`. The temporary probe file was archived and
removed from the source tree; port its useful cases into focused permanent test modules.
The first monitored attempt had two incorrect expected error strings. The archived
corrected probes expect the wrapper's actual `NOT published` message, then reach and
fail the intended remote-publication assertions. They do not merely fail in setup.

## Baseline and integration constraints

The audited primary tree is `8eabf9ecf`, with a locally rebuilt core at `e0f105d`. The
binding validator passes against that local build. Five corrected probes fail in 4.27
seconds. `sase-core-revision.txt` still pins `da0a73895ff8d5aa3597df4abb3fe6004c443489`,
which lacks `bead_set_link_projection` from `717c36e` and cutover APIs from `e0f105d`. A
successful local install therefore has not established correctness of the CI pin or the
published dependency floor.

Repair commits already present are `f5a3f5c99`, `811700bc3`, `840824c5b`, `2dcd6a136`,
and `8eabf9ecf`; preserve their producer identity, atomic no-replace installation,
frozen legacy history, resumable import identity, and owner routing. The intervening
core-pin change `3e32c5cc6` protects the weighted-capacity lineage wire; advance that
pin monotonically, never replace it with an older feature-only SHA. Fetched base
`e62e96f5f` adds query pushdown after `2165fd2d9` splits launch tests. Those commits do
not repair the link defects; they fix the incidental public `legacy_token_hint`
Symvision failure by privatizing it. Integrate current base drift when implementing,
preserving the query and test split work.

Shared ownership, receipt, causal reduction, and projection policy belongs in Rust. Open
`sase-core` through `sase repo open sase-core`, follow its instructions, and use its
printed path. Python remains filesystem, locking, commit/push, and binding glue. Every
phase adding/changing an API updates both bindings and the validator and ratchets the
required revision in the same delivered change. Do not hand-edit release versions. For
published floors, coordinate with existing release owners `bead:sase-z4.6.5.4.5` and
`bead:sase-xe.16.11.7.14.6.4`; do not invent a parallel release repair epic or claim a
local wheel proves a published floor.

Retain the existing per-root publication retry ledger as the only retry/aging policy.
Retain project-publisher-lock before store-lock ordering and release store locks before
push. Preserve automatic enqueue-only behavior, agent checkout isolation, explicit
synchronous manual commands, and bead-store ownership. No live fleet import, hidden
clone cleanup, or production migration is authorized by this plan. Exercise isolated
repositories and homes.

The failure reproductions are minimum acceptance cases. Each phase must also preserve
its existing public contract; the final phase adds independent-machine evidence.

## binding_baseline

Use the existing `tools/ratchet_core_revision` workflow to commit a pin containing all
currently required bindings, at least `717c36e` and `e0f105d` plus the lineage wire. The
ratchet tool's successful change exits 2, which must not be mistaken for a failed
mutation. Inspect the resulting revision and validate an installation from that exact
revision through the ordinary pinned-build path; a manually overridden newer checkout
alone is insufficient. Run `tools/validate_sase_core_rs` and the applicable binding
coverage gate. Preserve later concurrent pin advances. This resolves the unfulfilled
ratchet proposal in `sase-yy.8.4` note #1 and establishes the build baseline for
repairs.

## bead_history

Reproduction: publish two distinct `agent:reader -> bead:<id>` read observations
sequentially with no document sidecar owner. Each report says `published=1`, but both
`event_paths` lists are empty, `active_operation_ids_for_row` returns no IDs, and the
bead's `uses` remains 1. Only the imperative bead projection/receipt survives; the
original canonical events are unavailable to later reduction.

Root causes include `_needs_local_receipt` in
`src/sase/sdd/_artifact_link_event_publish.py`, which excludes bead owners, and the Rust
`artifact_link_publication_receipt` rule in `artifact_link/ownership.rs`, which accepts
the bead projection receipt without requiring replayable event history.

Persist the full immutable canonical event payload durably under bead-store ownership
before acknowledging a bead-only operation. A per-edge operation-ID receipt or mutable
aggregate row is insufficient. Prefer the existing content-addressed event format and
atomic installer, with correct bead commit/publication integration. Extend the Rust
ownership/evidence contract to distinguish canonical event durability from projection
completion; unresolved required owners remain pending. Keep genuinely ownerless non-bead
operations on their existing durable machine-local event path.

Teach event enumeration, collision checks, reduction, active-version lookup,
reconciliation, and recovery to consume bead-owned canonical history exactly once,
including copies also present in document sidecars. Replay after queue retirement,
process restart, and synchronization must preserve operation IDs, counts, removals,
metadata, and aliases. A bead-to-bead manual remove must be able to name its observed
versions. Preserve explicit bead-only legacy API behavior without representing each
projection refresh as a new user occurrence.

Cover two and more distinct reads, exact replay, empty document-owner sets, bead-bead
puts/removes, changed-payload collision refusal, and failure between event persistence,
bead projection, commit, and acknowledgement. Confirm durable canonical bytes exist
before any successful retirement.

## projection_convergence

Reproduction: apply distinct operations separately through two document stores, then
make one store see both events. Rust reduction reports `uses=2`, but forced
`apply_events_to_beads(..., (), force=True)` reports `receipt=True, changed=False` and
leaves the bead at 1. The audit probe intentionally shares one bead store to isolate
receipt behavior; it is not proof of a realistic two-machine sync fixture.

`set_bead_link_projection` in Rust `bead/mutation.rs` returns immediately when it has
seen the holder/operation/edge/direction receipt, before comparing the desired state.
The Python adapter loops over active occurrence IDs. After both IDs have receipts from
partial projections, no ID can install the corrected union. Removal/alias changes can
also change desired state without introducing a previously unseen active addition.

Separate deduplication of user occurrences from idempotent projection installation.
Drive bead projection from complete canonical event truth and an explicit Rust-owned
projection identity/causal contract. An existing occurrence receipt must not suppress
necessary repair, while replay of an unchanged state must remain a no-op. Repair must
not mint new read occurrences or allow a stale subset to overwrite a newer complete
state. Preserve multi-edge baseline receipt scoping, undirected holders, direction,
add-wins removals, alias endpoints, and failed-commit retry.

Prove this with independent bead clones and document clones, then real event union and
projection rebuild. Compare bead neighborhoods, aggregate/store lists, and managed
Markdown against Rust-reduced truth. Include different arrival orders, already-seen
receipts, concurrent unseen additions, baseline batches, and repeated repairs.

## reader_causality

Reproduction: an unrelated valid put plus a valid remove whose observed addition has not
arrived reduces successfully in Rust. `store.load_durable_rows()` instead raises
`artifact-link event store is invalid ... observes no known active version`.

`ArtifactLinkEventSnapshot.healthy` in `_artifact_link_event_store.py` treats
`orphaned_tombstones` as fatal corruption, and `covers_row` uses that same predicate.
The original event contract explicitly retains tombstones before their predecessors
arrive. Align all reader/projection/coverage consumers with that Rust contract. Keep
missing-predecessor evidence visible to doctor; reject actual digest/schema corruption,
operation-ID collisions, and invalid alias/cross-edge predecessor references. Do not
silence all health diagnostics or weaken real corruption checks.

A temporary missing predecessor must not block unrelated valid links or let legacy
fallback resurrect a removed edge. After the predecessor arrives, reduction and every
reader must converge without editing the tombstone. Cover both durable and pending
inputs, diagnostic output, read/list/rebuild/projection entry points, and remaining
hard-failure cases.

## synchronous_recovery

Two confirmed paths bypass outstanding remote publication:

1. `_add_artifact_link_event` in `artifact_cli/link_ops.py` returns `unchanged` before
   publication verification. After the first add commits locally and push fails,
   repeating the same add succeeds with one commit still ahead of the remote.
2. `_apply_import_locked` in `_artifact_link_import_apply.py` sees locally committed
   final markers as `phase=complete`, rebuilds, and reports
   `applied=True, already_imported=True, publication_error=None`. If the final marker
   push previously failed, the retry leaves each affected root one commit ahead.

Verify required publication even when no new bytes need writing. Reuse the existing
publication evidence and per-root retry machinery, with bounded synchronous attempts and
honest failure diagnostics. The importer must distinguish local commit progress from the
publication evidence required by the requested mode; extend Rust cutover
progress/evidence if shared semantics change. A complete local marker is not proof of
remote publication. Preserve stable import identity, original source heads, exact
baseline/marker bytes, and no-remote/local-only semantics.

Audit add, rm, plan inlet, and import no-change paths for the same guarantee;
specifically cover an already-absent remove after its first push failed. Do not
manufacture another logical operation or redundant commit merely to force a push.
Automatic retirement remains based on durable committed operations with root retries
handled by the ledger; manual published-on-success behavior remains synchronous.

Use isolated remotes to fail final-marker pushes and manual mutation pushes. Repeat
while failure persists (must still fail honestly), then restore publication and retry
(must publish without changing operation identity or counts). Cover multiple roots,
partial success, no-remote mode, and ledger-only recovery followed by an idempotent
public-command retry. Verify reachability of the exact committed objects in the remote,
not just an empty local worktree.

## acceptance

Port all five archived failures into permanent focused tests, extending existing
producer/public API fixtures rather than retaining the scratch module. Use independent
machine-local homes and independent bead/document repositories for cross-machine claims.
Existing `_cluster` shares a bead store and is suitable only for narrower unit
scenarios. Include real read/outbox drain, manual add/rm, plan inlet, import CLI, and
bead reconciliation entry points alongside low-level contract tests.

Confirm crash recovery at write/commit/push/ack boundaries, byte-identical replay,
converged operation counts and all projections, no metadata merge pauses, untouched
legacy frozen indexes and authored Markdown, and clean agent checkouts. Reuse the
existing process-death and ledger recovery coverage. Check current base and all required
Rust APIs against the final pinned build, not a stale published wheel silently installed
by `uv run`; use the installed venv directly or `uv run --no-sync` after a local build.

Each implementing phase runs `just check` and changed-core gates required by its repo.
The combined tree requires `just check-full` through `/sase_monitor` with TESTING/TESTED
before landing. Known unrelated failures and their disposition are recorded on
`sase-yy.8` (including existing wait-lint task `sase-zh` and restart-audit task
`sase-zi`); integrate fixes as they arrive rather than deleting coverage or hiding
failures. Full verification is still outstanding, and the previous phase's claim that
lint was its only failure did not establish a full green test run.

The child land agent rechecks its parent and ancestor plans, every descendant and note,
post-child drift, and all recorded follow-up outcomes. It uses normal nested landing
only while each ancestor is actually complete. Do not add ancestor close, Symvision
cleanup, or plan-status updates as child phases, and do not force a successful landing.
