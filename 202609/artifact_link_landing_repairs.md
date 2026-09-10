---
tier: epic
title: Complete artifact-link event identity, publication, and cutover guarantees
goal: Repair the reproduced sase-yy landing failures so immutable link operations
  survive retries, partial publication, reconciliation, and legacy cutover without
  lost counts or manual metadata repair.
parent_bead: sase-yy
phases:
- id: producer_identity
  title: Freeze derived and alias operation identity across retries
  size: medium
  depends_on: []
  description: 'producer_identity: make repeated derivation and rename discovery reuse
    byte-identical immutable events, reject collisions before persistence, and preserve
    machine eligibility without releasing unrelated agent reads.'
- id: publication_durability
  title: Require durable owners and publish complete event files atomically
  size: large
  depends_on:
  - producer_identity
  description: 'publication_durability: prevent false acknowledgements for missing
    owners, preserve non-document operations, install event files atomically across
    process death, and route manual document mutations through hidden machine stores
    with synchronous publication.'
- id: event_reconciliation
  title: Reduce event unions and keep bead projections consistent
  size: large
  depends_on:
  - publication_durability
  description: 'event_reconciliation: union immutable operations before cross-clone
    reduction, preserve add-wins and alias semantics, and make bead projection and
    operation receipts agree with the same reduced truth.'
- id: cutover_recovery
  title: Make legacy cutover resumable and preserve frozen history
  size: large
  depends_on:
  - producer_identity
  - publication_durability
  - event_reconciliation
  description: 'cutover_recovery: preserve legacy outbox rows during ordinary drains,
    resume interrupted multi-root import safely, require operator capability confirmation,
    and make post-import rename and maintenance event-only with shared policy in Rust.'
- id: acceptance
  title: Verify real producer, crash, and reconciliation paths end to end
  size: medium
  depends_on:
  - producer_identity
  - publication_durability
  - event_reconciliation
  - cutover_recovery
  description: 'acceptance: extend multi-clone tests through production producers
    and actual process death, cover every reproduced landing defect and public mutation
    path, and complete the existing flag-retirement verification.'
proposed_by: bbugyi200.athena.sase-yy.land
create_time: 2026-09-10 14:27:19
status: wip
bead_id: sase-yy.8
---

- **PROMPT:** [prompts/202609/artifact_link_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/artifact_link_landing_repairs.md)
- **PARENT:** [202609/artifact_link_events_v2.md](https://github.com/sase-org/sase--plans/blob/main/202609/artifact_link_events_v2.md)
- **BEAD:** [sase-yy.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.8.md)

# Remaining artifact-link event work

This is a repair child of `bead:sase-yy`, whose original seven phases are closed. The
landing audit found required behavior still missing, so the parent stays open. Implement
only the remaining work below. The `parent_bead` relationship is the handoff back to the
interrupted parent landing; no phase closes the parent or changes its plan status.

Read the original requirements with
`sase artifact read plan:202609/artifact_link_events_v2.md "Need parent event guarantees"`.
Read the audit and reproducible evidence through the same audited artifact command:

- Audit: `file:explicit:e7416d60ce1cf8d99a71fe3c`.
- Reproduction script: `file:explicit:7f5e04013d3b9e6e1b5a54db`.
- Reproduction output: `file:explicit:a6e3025e88ceccabf89d82ae`.

The script uses isolated temporary repositories and machine state. Its publication
fixture suppresses runtime commit tags; any file-hook warnings about missing fixture
remotes are incidental. Port the scenarios into ordinary regression tests using the test
suite's normal environment fixtures.

## Verified baseline and integration constraints

The audit was performed at primary `abdcb86d6`, also the fetched origin/master. All 35
focused existing tests passed, and `tools/validate_sase_core_rs` exited 0. No full
landing verification has run. The existing acceptance suite uses prebuilt events and
already-converged snapshots, so those passing tests do not cover the failures below.

The main epic commits are `232ffbba3`, `9e52abc5a`, `fb89440a5`, `37ab56bd9`,
`ba73bc30e`, `a8d99d295`, and `abdcb86d6`. Rust core commits are `528c3db` (event
contract/reducer), `55770cb` (index merge), and `1842f29` (bead receipts). The current
Python dependency floor is already `sase-core-rs>=0.33.0,<0.34.0`; the `.2` and `.3`
notes proposing that floor ratchet require no separate task.

Keep the intervening module splits `81033f4de`, `59c4d1186`, `9c738c25f` and public
facade repair `024e01b70`. Preserve sidecar eviction protection from `54d9c112a` and
clone fallback from `74de0fa7c`. Reuse the landed per-root publication ledger and its
deadline, fairness, aging, unpublished-head discovery, and release-evidence behavior;
never introduce a second push-retry ledger.

Shared identity, ownership, reduction, migration, and projection policy belongs in Rust
`sase-core`. Open that repository with `sase repo open sase-core`, use the printed path,
and follow its AGENTS.md. Python handles filesystem/locking/process glue and thin
binding adapters. Where repair needs a missing shared contract, add the Rust API and
PyO3 tests first, then update `tools/validate_sase_core_rs` and release-compatible
dependency/revision pins. Do not hand-edit release versions.

## producer_identity

`_persist_derived_link_candidates_as_events` in
`src/sase/sdd/artifact_link_derivation.py` computes a stable ID from candidate fields
and creator, but fills `created_at` from the current clock. Two identical sweeps one
second apart enqueue the same ID with different bytes. The Rust reducer correctly
refuses them, potentially blocking every later aggregate rebuild/drain for the project.
`_alias_operation_id` similarly ignores the actor while alias payloads use the current
agent identity.

Define the shared producer identity contract in Rust and freeze the entire event payload
before first enqueue. Repeated derivation/rename discovery of the same fact must reuse
identical bytes across calls, machines, and different discovering agents. Use real
source provenance or an explicitly defined stable derived-fact identity; do not append a
changing timestamp to the hash merely to manufacture another event on every sweep.
Actual read/prompt occurrences remain distinct operations.

Make the journal and publisher reject a reused operation ID with different bytes before
adding a second conflicting payload to the queue or event set. Preserve the first valid
immutable operation; surface existing corrupt collisions without rewriting published
history. Define tests for queued retries and already-committed retries. Audit rename
queue metadata: background aliases currently use `sase.artifact-link-renames` and
sometimes no run ID, which does not meet the trusted machine eligibility check. Give
machine facts appropriate provenance without weakening exact agent/run release evidence
for user reads.

Tests: same candidate at different clock times; different discovering agents for one
rename; repeated sweeps after successful publication; genuine repeated reads; collision
refusal before persistence; machine aliases drain while unrelated ineligible agent
observations remain queued.

## publication_durability

`_roots_by_operation` in `_artifact_link_event_publish.py` accepts an empty owner set.
The readiness check then acknowledges the operation, while `apply_events_to_aggregate`
rebuilds excluding pending operation IDs. A read between two agent refs, or a read of a
document whose sidecar could not be resolved, reports `published=1` with no event paths
and an empty aggregate. Drain can retire the only copy. Missing one of two document
owners must also prevent complete acknowledgement.

Use explicit per-operation owner requirements and durable receipts. An unresolved
required document/bead owner remains pending with a useful diagnostic. Preserve the
established non-document/aggregate-only behavior with recoverable operation identity:
never claim it was durably published merely because there are zero document roots.
Ensure repeated delivery cannot inflate counts. Follow the core boundary for
ownership/receipt semantics and keep the existing per-root ledger as the sole push-retry
policy. A bead no-op after a failed commit must still verify the previous mutation
became committed before acknowledgement.

`_create_event_file` currently opens the final content-addressed name with O_EXCL and
writes into it. Actual SIGKILL between creation and writing leaves a zero-byte final
file and permanent corruption errors on retry. Install complete, fsynced bytes using an
atomic **no-replace** operation, including directory durability as needed. Clean only
identifiable unpublished staging remnants; never overwrite or delete a published object
to hide corruption. Preserve symlink and same-path/different-bytes rejection and cover
process death before installation, after installation, after commit, after push, and
before acknowledgement. Review outbox fsync/atomic rewrite ordering so acknowledgement
never outruns durable receipt state.

Manual `link add/rm` in `artifact_cli/link_ops.py` and plan links ingestion in
`artifact_link_inlet.py` currently pass checkout stores directly to the publisher. Route
document writes through `resolve_machine_artifact_link_store`; preserve explicit
synchronous published-on-success behavior and authored plan-body handling. Retain scoped
commits, project publisher lock then store lock ordering, and the recent fix that
releases the store lock before pushing. Keep automatic writers enqueue-only from
numbered agent checkouts.

Tests must exercise real public manual/inlet callers with separate agent and hidden
stores, unavailable owners, two document endpoints with one failed root, non-document
observations, commit failures/replays, and push failure followed solely by ledger
recovery. Assert agent clones have no event or legacy-link mutation.

## event_reconciliation

`_artifact_link_store_reconcile.py` reduces each clone independently to rows and then
deduplicates those rows. Two clones each holding a distinct observation for one edge
reconcile to `uses=1`; Rust reduction of the event union yields `uses=2`. The same
pattern loses causal information for removals, aliases, and supersession.

Gather validated immutable event identities across eligible clones, union/deduplicate
operations, overlay local pending operations once, and reduce the union in Rust. Keep
legacy pre-import inputs and projected/aggregate-only sources under explicit ownership
rules. Equal event sets must produce equal logical rows regardless of clone traversal,
synchronization state, duplicate endpoint copies, or arrival order. Do not use row
upsert's read-counter increment as a projection merge operation. Retain removal
knowledge so a stale clone or legacy bead backfill cannot resurrect a removed document
edge.

Make bead endpoint projection agree with reduced truth. The current adapter imperatively
applies each event and only passes an operation ID; add-wins removes, out-of-order
updates, aliases, and baseline batches need explicit parity coverage. Core bead receipts
currently deduplicate per holder and operation ID: ensure a baseline operation
containing multiple edges to one bead retains every edge while retry remains idempotent.
Preserve bead store ownership and legacy CLI compatibility; do not mint another logical
link operation for repair. Confirm all read surfaces (bead neighborhoods, store list,
aggregate/index list, managed Markdown) agree.

Tests: unsynchronized disjoint observations of the same edge; duplicate copies;
remove-before-add and concurrent unseen addition; stale late publication; pending plus
durable copies; alias plus late old-ref read; repeated bead/doc observations; baseline
with several edges touching one bead; and replay after bead mutation/commit
interruption. Doctor may report temporarily missing predecessors, but reader policy must
respect the core event contract rather than invent different reduction rules.

## cutover_recovery

Three existing paths violate the migration contract:

1. A mixed `[v1,v2]` outbox becomes `[v2]` during a normal **ineligible** drain with
   `drained=0,dropped=0`. Parsing rejects v1 and the acknowledgement rewrite drops
   anything the reader skipped. Preserve opaque/unconverted valid legacy entries until
   explicit deterministic conversion; do not silently discard malformed rows either.
   Conversion must retain original run eligibility and reconcile already represented
   legacy uses, with exactly-once replay.
2. Import writes all role markers but requires every marker to be identical before
   planning a retry. Failure during the second marker write leaves a partial set which
   every retry rejects. Create a recoverable import protocol with stable frozen source
   heads, per-root progress, validation and publication receipts. Ordinary reads may
   fail closed during an incomplete cutover, while the importer can resume exactly the
   same operation after partial marker write/commit/push, baseline write/commit/push,
   and final-marker transitions. Refuse conflicting source/import identities without
   destructive resets. Handle projects with no legacy rows explicitly; an empty valid
   project must have a defined cutover path.
3. `_apply_artifact_renames` always rewrites/deletes legacy indexes after queueing
   aliases. A post-import rename currently reports one changed and one removed legacy
   index. Once fenced/imported, all maintenance leaves `links/**` untouched and consumes
   event truth. Audit `backfill_bead_endpoint_links`, reconciliation, and other legacy
   index consumers so they cannot reintroduce frozen rows or use direct imperative
   writes to sidestep event operations.

Move shared marker validation, migration identities, state transitions and baseline
policy out of `_artifact_link_cutover_state.py` / `artifact_link_import_indexes.py` into
Rust contracts; leave Python filesystem and CLI adapters. The existing CLI prints a
warning but has no operator fleet-capability confirmation. Follow the original plan's
confirmation requirement and CLI rules (read the CLI memory first): preview remains
default, apply requires an explicit capability attestation before mutation. This phase
adds and tests the product mechanism, not a live fleet rollout; no production import or
machine upgrade is authorized by the repair plan.

Validate baseline parity, source-head fencing, publication-before-resume, failed marker
retry with unchanged bytes, simultaneous independent-machine import attempts,
post-import legacy straggler reporting, and byte-for-byte preserved frozen indexes after
rename/backfill. Keep the conservative legacy conflict resolver available for old
history and stragglers, plus the existing retry ledger for old unpublished heads.

## acceptance

Extend `tests/sdd/test_artifact_link_event_acceptance.py` and focused sibling tests
using real producer and public command paths. Keep module sizes within repository limits
by splitting tests by scenario rather than adding another oversized file. Use
independent machine-local homes and independent repositories, not a shared queue/bead
store pretending to be two machines. Exercise the seven reproduced failures plus the
specified producer, ownership, bead, and cutover regressions.

At least one test must kill a separate publisher process during object installation;
creating a complete file before a replay does not exercise partial-write recovery. At
least one recovery must consist solely of the existing per-root ledger publishing a
committed head. Verify no lost/double-counted operation IDs, content hashes, convergent
rows from equal event unions, no metadata merge pauses, unchanged authored Markdown, and
clean agent checkout state. Retain emitted queue p95 age and link-only commit counts.
Ensure retirement assertions actually inspect committed receipt state.

Review `bead:sase-z0` under the flags memory. If the repaired event-only path and
acceptance evidence satisfy its existing removal condition, close that flag bead with a
verification note; its registry/schema code is already removed. Do not create a
replacement task for it. Record all out-of-scope proposals on phase notes using
`PROPOSED FOLLOW-UP:` for the land agent.

Each phase runs required local checks (`just check` here, the full `sase-core` gate
including binding tests when core changes). The combined repaired tree needs
`just check-full` through `/sase_monitor` with TESTING/TESTED statuses before landing.
The child land agent must recheck this plan and parent requirements against later
commits. Parent closure, parent symbol cleanup, and parent plan status updates remain
landing actions, not child phases.
