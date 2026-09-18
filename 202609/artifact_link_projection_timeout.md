---
tier: epic
title: Keep artifact-link bead projection within the housekeeping deadline
goal: The artifact_link_backfill chop projects complete event truth into beads through
  bounded bulk mutations, preserves durable receipts and hidden-clone safety, and
  exits normally before Axe's 300-second timeout.
phases:
- id: projection_batch_core
  title: Add an atomic bulk bead-projection core API
  depends_on: []
  size: medium
  description: 'projection_batch_core: add and publish a Rust/PyO3 batch mutation
    that preserves the exact single-projection receipt and convergence contract while
    taking one bead lock, loading once, and saving once per bounded batch.'
- id: deadline_aware_projection
  title: Batch and bound artifact-link projection in SASE
  depends_on:
  - projection_batch_core
  size: medium
  description: 'deadline_aware_projection: pin the published core capability, route
    full-truth bead projection through bounded batches, and propagate the chop deadline
    so partial progress is committed and safely retried instead of being SIGKILLed.'
- id: production_acceptance
  title: Prove convergence and scheduled-job completion
  depends_on:
  - projection_batch_core
  - deadline_aware_projection
  size: small
  description: 'production_acceptance: exercise the production backfill path against
    a scaled fixture and one controlled live run, proving the queued operations converge,
    hidden sidecars remain clean, and the chop completes below its soft budget.'
proposed_by: bbugyi200.apollo.0k
create_time: 2026-09-18 09:47:54
status: wip
bead_id: sase-12y
---

- **PROMPT:** [prompts/202609/artifact_link_projection_timeout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/artifact_link_projection_timeout.md)
- **BEAD:** [sase-12y](https://github.com/sase-org/sase--beads/blob/main/pages/sase-12y/README.md)

# Keep artifact-link bead projection within the housekeeping deadline

## Diagnosis

The `housekeeping/artifact_link_backfill` runs `20260918T084006_123508` and
`20260918T093317_017830` both reached Axe's exact 300-second hard timeout. In each run
`gh_bobs-org__bob-cli` completed in roughly ten seconds, then the log stopped
immediately after `[artifact_link_backfill] gh_sase-org__sase: starting`. The repeat
proves this is not a one-off clone or network stall.

The first failed run durably published three new derived event objects to the plans
sidecar at 12:43:11 UTC, after the `sase` project started and before Axe killed the
process at 12:45:06 UTC. The process was subsequently observed CPU-bound with no Git
child. This places the failure after hidden plans-clone recovery and document-owner
publication, not in the recently repaired clone preflight.

The hot path is `apply_events_to_beads()` in
`src/sase/sdd/_artifact_link_event_project.py`. It correctly reduces the complete
canonical event union, but then loops over every affected bead endpoint and every raw or
active operation receipt. Each iteration calls `set_bead_endpoint_projection()`, whose
Rust binding independently takes the bead mutation lock, loads and reduces the complete
bead store, appends one projection event, and saves the store. A small new publication
therefore multiplies full-store load/save work by historical link-operation count. The
chop's monotonic deadline is not forwarded through derivation persistence, the ordinary
outbox drain, event publication, or bead projection, so its nominal 240-second soft
budget cannot interrupt this CPU path before Axe's 300-second SIGKILL.

The durable sweep checkpoint already contains 4,867 `sase` document refs. Do not reset
it, delete the queued operations, remove the three durable event objects, or treat an
unacknowledged bead projection as published merely to make the job finish.

## Constraints from approved work

This plan builds on, and must not conflict with, these audited approved plans:

- `plan:202609/hidden_artifact_link_clone_recovery.md`: retain lossless recovery refs,
  strict hidden-clone identity/remote checks, upstream alignment, and deadline-aware
  integration. Do not restore the old dirty-clone blocker or bypass recovery.
- `plan:202609/machine_bead_link_writes_hidden_clone.md`: machine bead writes stay in
  the host-owned hidden beads clone, authorization happens before mutation, and the
  primary sidecar remains pull-only. Do not weaken ownership refusal or write into a
  human workspace clone.
- `plan:202609/artifact_link_durable_truth_repairs.md`: projections continue to derive
  from complete canonical event truth; operation receipts, add-wins removal, alias
  behavior, convergence after partial views, and honest acknowledgement remain intact.
  Batching is an execution optimization, not a new source of truth.
- `plan:202609/fix_artifact_link_rename_repair_memoization_1.md`: keep the existing
  240-second soft budget beneath the five-minute Axe timeout and retain controlled live
  chop verification. Do not solve this by increasing either timeout.

Shared bead mutation and projection behavior belongs in `sase-core`; Python remains the
deadline, chunking, publication, and commit/push coordinator. Preserve the current
project-publisher-lock before store-lock ordering, and keep store locks released before
push.

## Phase `projection_batch_core`: atomic bulk projection

Work in the linked `sase-core` repository and follow its release-owned versioning rules.
Do not manually edit crate or workspace versions.

1. Add a typed bulk bead-link projection request and one core mutation that accepts an
   ordered batch under a single bead mutation lock. It must load `MutableStore` once,
   apply every request with the same validation, canonical target resolution,
   undirected-holder selection, receipt lookup, exact-state convergence, and event
   payload used by `set_bead_link_projection()`, then save once only when the batch
   changed state. Refactor the existing singleton API through shared internal logic so
   singleton and batch semantics cannot drift.

2. Make the batch transactional at the persistence boundary. An invalid request must not
   leave a partial event stream or `issues.jsonl` projection. Preserve input order,
   distinct operation receipts, repeated-operation idempotence, and the ability for an
   already-seen receipt to repair a changed reduced state. Return an aggregate mutation
   outcome that reports whether anything changed and which issue IDs were touched;
   stable replay must not rewrite the store.

3. Expose the batch through a documented and registered PyO3 binding, while retaining
   the existing singleton binding for compatibility. Add binding conversion and error
   tests. Cover mixed present/absent projections, directed and undirected holders,
   aliases already canonicalized by the caller, multiple operations on one endpoint,
   invalid-middle-request rollback, stable replay, and byte-equivalence with the
   singleton contract. Include a scaled regression that proves one batch takes one store
   load/save cycle rather than one cycle per request.

4. Run the core repository's required `just check`. Land through the normal host-owned
   flow and let release-plz publish the next compatible `sase-core-rs`; do not claim the
   SASE integration phase is ready from a locally overridden wheel alone.

## Phase `deadline_aware_projection`: bounded SASE integration

Begin only after the core batch capability is published. Advance
`sase-core-revision.txt` monotonically with `tools/ratchet_core_revision`, use the
normal core-window workflow to require the first published version that contains the
binding, and update `tools/check_sase_core_rs_bindings` / `tools/validate_sase_core_rs`
with a behavioral batch probe. Preserve any newer concurrent pin or dependency-floor
advance.

1. Add a thin batch wrapper to `src/sase/core/bead_mutation_facade.py` and
   `src/sase/sdd/artifact_link_beads.py`. The wrapper must keep the existing
   authorize-before-mutate guard and convert the complete request list to the typed core
   wire without reimplementing bead semantics in Python.

2. Refactor `apply_events_to_beads()` to reduce the complete canonical event union once,
   construct the same desired endpoint states and per-operation receipts it constructs
   today, and send them to the bulk API in bounded chunks. Keep all requests for a
   logical endpoint deterministically ordered. An ordinary publication may scope work to
   endpoints whose reduced state or required receipts can be affected by the incoming
   objects, but derive their desired values from the full union; `force=True` must still
   perform a complete convergence pass. Alias, remove, and baseline-import inputs need
   explicit tests proving the scope never misses indirectly affected endpoints.

3. Thread the existing monotonic chop deadline through
   `run_artifact_link_backfill_batch()` -> derived-candidate persistence -> outbox drain
   -> event publication -> bead projection, and through the explicit post-sweep outbox
   drain. Interactive and non-chop callers retain `deadline=None` and complete all work.
   Check the deadline before reduction and between bounded bulk calls. If it expires:
   - stop starting projection batches;
   - commit any bead projection work already saved in the hidden clone;
   - return `receipt=False` with one stable "deferred past job budget" diagnostic;
   - retain the affected outbox operations and unswept refs for the next tick; and
   - let the chop emit a normal deferred summary and persist its project cursor.

   A deadline deferral must never leave the hidden beads clone dirty, acknowledge an
   incomplete projection, or discard an immutable operation. Retrying after partial
   progress must be idempotent and converge without incrementing logical `uses`.

4. Add progress logging at the existing project/stage boundary so a killed or deferred
   run identifies whether it completed publication retry, store resolution, sweep,
   drain, and reconcile. Keep logs bounded and retain the existing final per-project
   timing line and counters.

5. Extend focused tests in the bead facade, event publisher/projection, derivation,
   outbox, machine-store, and chop suites. Required cases include: many historical bead
   operations plus a tiny incoming batch use bounded bulk calls; a deadline between
   chunks commits partial progress but retains the outbox; a retry completes and
   acknowledges exactly once; forced convergence repairs a partial projection; hard
   validation/publication failures remain failures; hidden-root authorization occurs
   before the first batch; and the primary plans/beads clones are untouched.

Consult the repository lint/test memory after tracked edits, run the focused suites,
`tools/validate_sase_core_rs`, and the primary repository's `just check`. Do not mask a
stale installed binding with a source-only import override.

## Phase `production_acceptance`: scaled and live proof

1. Add or reuse an isolated production-path fixture with enough canonical link events
   and bead records to reproduce the old multiplicative behavior. Assert semantic
   convergence and a structural work bound (bulk-call/load/save counts and deadline
   deferral), not a fragile sub-second wall-clock threshold. Exercise a second pass to
   prove stable replay performs no bead-store rewrite.

2. With the released/pinned core and installed SASE built from the implementation, force
   one controlled `artifact_link_backfill` housekeeping run. Record its run ID and
   verify that:
   - both enabled projects finish and the run produces a structured result rather than
     exit `-9`;
   - total runtime remains below the 240-second soft budget with margin, or
     intentionally deferred work exits normally before that budget;
   - the three operations published during the failed 12:43 UTC run and any retained
     outbox entries receive honest durable bead receipts exactly once;
   - the sweep checkpoint/cursor advance and a second no-op run stays bounded; and
   - hidden plans and beads clones are clean and upstream-aligned while primary sidecars
     were changed only by their existing pull-only auto-sync path.

3. Run `just check` in both repositories. Let the host's exhaustive landing lane run the
   full matrix; investigate any unrelated red result rather than weakening the focused
   acceptance criteria or increasing timeouts.

## Acceptance criteria

- `artifact_link_backfill` cannot exhaust Axe's 300-second timeout by replaying bead
  projections one full-store mutation at a time.
- Complete event truth, operation receipts, uses counts, removals, aliases, and forced
  convergence are byte- and behavior-compatible with the approved durable-truth work.
- Deadline exhaustion becomes resumable, visible deferral with committed partial
  progress—not SIGKILL, false acknowledgement, lost operations, or a dirty hidden clone.
- Hidden-clone self-healing and ownership boundaries remain unchanged, and no timeout is
  increased.
- Core and primary focused tests, both `just check` gates, binding validation, the
  scaled regression, and the controlled live run all pass.
