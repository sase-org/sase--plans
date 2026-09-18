---
tier: epic
title: Fix artifact_link_backfill hourly 300s SIGKILL
goal: "The hourly artifact_link_backfill chop finishes every project inside its 240s
  budget, bead endpoint projection costs one store open per pass instead of one per
  endpoint, deadline expiry defers publication work cleanly instead of dying by SIGKILL,
  and the artifact-link outbox stops accumulating identical duplicate events.

  "
phases:
  - id: rust-batched-projection
    title: Batched bead endpoint projection in sase-core
    depends_on: []
    size: medium
    description: "rust-batched-projection: add a batched bead endpoint projection
      operation to the sase_core crate and its PyO3 binding that applies N projection
      specs with one store open/load and one persist, with sequential-parity semantics,
      per-spec outcomes, contract-validator updates, and a performance measurement gate.

      "
  - id: python-batched-adoption
    title: One batched Rust call per apply pass
    depends_on:
      - rust-batched-projection
    size: medium
    description: "python-batched-adoption: rework apply_events_to_beads to collect every
      endpoint projection it currently issues one Rust call at a time and submit them as
      a single batched call through a new facade wrapper, preserving affected-key
      computation, changed aggregation, commit-once, and receipt semantics.

      "
  - id: deadline-propagation
    title: Chop budget reaches every publication step
    depends_on:
      - python-batched-adoption
    size: medium
    description: "deadline-propagation: thread an optional monotonic deadline from the
      artifact_link_backfill chop through persist_derived_link_candidates,
      drain_artifact_link_outbox, publish_artifact_link_events, and
      apply_events_to_beads so expiry defers remaining work with diagnostics and no
      receipts, leaving deferred outbox entries queued for the next tick.

      "
  - id: outbox-hygiene-and-chop-hardening
    title: Stop the duplicate and fairness amplification
    depends_on: []
    size: small
    description: "outbox-hygiene-and-chop-hardening: make
      append_artifact_link_outbox_event skip byte-identical duplicate events for an
      already-queued operation_id, and persist the chop's fairness cursor at project
      start so a hard kill still rotates the project order on the next tick.

      "
  - id: end-to-end-verification
    title: Prove the hourly job converges on real state
    depends_on:
      - python-batched-adoption
      - deadline-propagation
      - outbox-hygiene-and-chop-hardening
    size: small
    description:
      "end-to-end-verification: run the chop manually against real machine state,
      confirm gh_sase-org__sase completes well under budget, the duplicated derived
      outbox backlog collapses, checkpoints and cursor advance, and subsequent scheduled
      runs produce no new timeout digests."
proposed_by: bbugyi200.athena.0n5
create_time: 2026-09-18 14:40:52
status: wip
---

- **PROMPT:**
  [prompts/202609/artifact_link_backfill_timeout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/artifact_link_backfill_timeout.md)

# Fix the hourly `artifact_link_backfill` 300s SIGKILL on large projects

## Diagnosis (evidence already gathered; do not re-derive)

Every run of the housekeeping `artifact_link_backfill` chop today dies at exactly 300s
(exit `-9`, axe timeout) with `[artifact_link_backfill] gh_sase-org__sase: starting` as
the last project log line. Two `py-spy dump` samples taken 30s apart on a live
reproduction show the identical active (CPU-bound, no child process) stack:

```
_call_issue_operation (sase/core/bead_mutation_facade.py)
set_link_projection (sase/core/bead_mutation_facade.py)
set_bead_endpoint_projection (sase/sdd/artifact_link_beads.py)
apply_events_to_beads (sase/sdd/_artifact_link_event_project.py)
_publish_event_objects_locked (sase/sdd/_artifact_link_event_publish.py)
publish_artifact_link_events (sase/sdd/_artifact_link_event_publish.py)
drain_artifact_link_outbox (sase/sdd/_artifact_link_outbox_drain.py)
_persist_derived_link_candidates_as_events (sase/sdd/artifact_link_derivation.py)
persist_derived_link_candidates (sase/sdd/artifact_link_derivation.py)
run_artifact_link_backfill_batch (sase/sdd/artifact_link_backfill.py)
_run_project (sase/scripts/sase_chop_artifact_link_backfill.py)
```

Measured facts for `gh_sase-org__sase` on this machine:

- `apply_events_to_beads` reduces the union of ~697 durable link events and then issues
  one Rust `bead_set_link_projection` call per affected
  `(issue_id, target_ref, relation, direction)` × `operation_id` pair: **1,068 calls per
  invocation**.
- A benchmark against a throwaway copy of the hidden beads clone (~5,496 issues in
  `issues.jsonl`) measured **~380ms per call**, i.e. **~406s for one full pass** — alone
  above both the chop's internal 240s budget and the axe 300s SIGKILL. Each call pays a
  full store open/load/persist.
- The sweep triggers this once per 50-document chunk (via the nested outbox drain in
  `_persist_derived_link_candidates_as_events`), and `_run_project` runs one more
  standalone `drain_artifact_link_outbox`, so a full project pass needs several such
  406s passes.
- No deadline reaches any of this code: the chop checks its 240s budget only between
  chunks and between jobs; `persist_derived_link_candidates`,
  `drain_artifact_link_outbox`, `publish_artifact_link_events`, and
  `apply_events_to_beads` accept no deadline parameter.
- The failure self-amplifies. SIGKILL lands before the outbox rewrite, before the
  swept-checkpoint write for the in-flight documents, and before the fairness cursor
  write, so every hourly run re-derives the same candidates and re-appends identical
  events: the project outbox
  (`~/.sase/projects/gh_sase-org__sase/artifact-link-outbox.jsonl`) holds **1,678
  `sase.artifact-link-derived` entries covering only 129 distinct operation_ids** (worst
  duplicate: 33 copies). `_reject_outbox_operation_collision` rejects only
  same-id-different-bytes, never identical duplicates.
- The repeated `Recovered workspace SDD clone .../repos/beads` warnings are a symptom,
  not a cause: SIGKILL lands after bead files are written but before
  `commit_bead_link_events`, leaving the hidden beads clone dirty for the next run's
  fresh integration.

The full-union projection loop was introduced deliberately by commit `840824c5b`
(`feat(sdd): reconcile artifact link event unions`, epic `sase-yy.8.3`, plan
`202609/event_reconciliation.md`): bead projections must converge to the Rust-reduced
truth of the whole event union, and exact replay must be a no-op. That convergence
contract must be preserved — the defect is purely that the apply step pays a full bead
store open per endpoint and cannot be interrupted.

## Design

Three independent defects, three fixes:

1. **Batch the Rust projection** (in the sibling `../sase-core` repo, per the Rust core
   backend boundary): one store open, N endpoint projections, one persist.
2. **Bound the work**: thread an optional monotonic `deadline` from the chop through the
   drain/publish/apply pipeline; expiry defers remaining work with diagnostics and
   without receipts, so deferred events stay queued and retry next tick.
3. **Stop duplicate growth**: make `append_artifact_link_outbox_event` idempotent for
   identical `(operation_id, payload)` entries, and let the chop's fairness cursor
   survive a hard kill.

No feature flag: this is internal housekeeping machinery with no user-reaching behavior
change, and the old per-endpoint call path does not need to stay reachable.

No manual state surgery is planned: once a drain completes within budget, the publish
path already dedupes objects by operation identity and `_rewrite_without_ids` drops
every duplicate outbox entry sharing a drained operation_id, so the backlog self-heals.

## Batched bead endpoint projection in sase-core

In the sibling Rust core repo (`../sase-core`, crate `sase_core`), add a batched bead
endpoint projection operation and expose it through the PyO3 binding (working name
`bead_set_link_projections`):

- Input: `beads_dir` plus an ordered sequence of projection specs, each carrying the
  exact fields of today's `bead_set_link_projection` call (`issue_id`, `target_ref`,
  `relation`, `direction`, `present`, `operation_id`, `description`, `origin`, `uses`,
  `now`).
- Semantics: byte-for-byte equivalent store outcome to issuing the same specs
  sequentially through the existing single-shot operation, but with one store open/load
  and one persist cycle. Preserve the `sase-yy.8.3` idempotency contract (idempotency
  scoped to operation plus projected edge/direction; exact replay is a no-op).
- Output: per-spec outcomes (at minimum the existing `changed` flag) plus whatever final
  issue payloads callers need, so Python can keep its `changed` aggregation.
- Failure semantics: validate specs up front where possible; a failing spec must not
  corrupt the store. Document and test whether the batch applies atomically or stops at
  the first error — pick whichever is simpler to guarantee, and make the outcome
  observable to the caller.
- Keep the existing single-shot `bead_set_link_projection` binding untouched for the
  CLI/manual paths.
- Update the binding contract validator (`tools/validate_sase_core_rs` consumers)
  without manually changing release-managed crate versions.
- Add Rust tests for parity with the sequential path (including replay no-op and
  multi-edge single-holder cases) and a benchmark-style test or measurement showing
  ≥1,000 projections against a store with ~5,000 issues complete in single-digit
  seconds.
- Run the sase-core repository gate.

## One batched Rust call per apply pass

In this repo, rework `apply_events_to_beads`
(`src/sase/sdd/_artifact_link_event_project.py`) to collect the full list of
`(endpoint key, operation_id, desired row)` projections it currently issues one by one
and submit them as a single batched Rust call via a new thin wrapper in
`src/sase/core/bead_mutation_facade.py` / `src/sase/sdd/artifact_link_beads.py`:

- Preserve semantics exactly: same affected-key computation over the full durable union,
  same desired-row/active-operation selection, same `changed` aggregation, one
  `commit_bead_link_events` after the batch, receipt only when the store is clean.
- Install or refresh the matching local Rust binding as required by the repository
  workflow, run `tools/validate_sase_core_rs`, and keep the binding contract in sync.
- Extend the existing regression tests that cover `apply_events_to_beads` (event
  acceptance, event store, publisher, bead tests) to run against the batched path,
  including interruption/replay coverage.
- Run the repository's required verification (`just check`; `just check-full` gates
  landing).

## Chop budget reaches every publication step

Thread an optional monotonic `deadline: float | None` from
`src/sase/scripts/sase_chop_artifact_link_backfill.py` through:

- `run_artifact_link_backfill_batch` → `persist_derived_link_candidates` →
  `_persist_derived_link_candidates_as_events` → its nested `drain_artifact_link_outbox`
  call;
- the chop's standalone `drain_artifact_link_outbox` job (pass the chop deadline);
- `publish_artifact_link_events` / `_publish_event_objects_locked` (check between
  root-group publishes and before the bead apply);
- `apply_events_to_beads` (check before starting the batched projection call).

Deferral semantics: on expiry, skip the remaining work, append a clear deferral
diagnostic to the existing report/diagnostic channels, and grant no publication receipts
for deferred operations — deferred outbox entries must remain queued and be retried on
the next tick, exactly like today's retained entries. Interactive callers that pass no
deadline keep today's run-to-completion behavior. Add tests that a mid-pipeline expiry
defers cleanly (no receipt, entry retained, diagnostic present) and that a `None`
deadline changes nothing.

## Stop the duplicate and fairness amplification

Two small robustness fixes in this repo:

1. In `append_artifact_link_outbox_event` (`src/sase/sdd/_artifact_link_outbox_io.py`),
   when an existing queued entry already carries the same `operation_id` with
   byte-identical canonical event payload, return the existing entry without appending.
   The collision scan already iterates matching entries, so this adds no new I/O. Keep
   the same-id-different-bytes rejection unchanged. Add tests for the skip, the
   rejection, and the normal append.
2. In `src/sase/scripts/sase_chop_artifact_link_backfill.py`, persist the fairness
   cursor when each project starts (before `_run_project`) in addition to the final
   write, so a hard kill mid-project still rotates the starting project on the next tick
   instead of replaying the same order forever. Keep the end-of-run write. Add or extend
   a test asserting the cursor on disk advances past a project as soon as that project
   starts.

## Prove the hourly job converges on real state

With all fixes deployed to the installed `sase` tool on this machine (the axe runs the
primary checkout via the uv tool install), verify against real state:

- Run the chop once manually with a context file copied from a recent run under
  `~/.sase/axe/lumberjacks/housekeeping/chops/artifact_link_backfill/runs/` (redirect
  `result_file` to a scratch path; keep the real `state_dir`).
- Confirm `gh_sase-org__sase` logs a `done in …` line with all four job timings and the
  whole run finishes well under the 240s chop budget.
- Confirm the derived-producer backlog collapses: the project outbox should drop from
  ~1,678 `sase.artifact-link-derived` entries (129 distinct operation_ids) to at or near
  zero for that producer, and a second manual run should show a near-no-op drain.
- Confirm the sweep checkpoint (`artifact_link_backfill.json` under the housekeeping
  lumberjack state dir) now covers the previously unswept documents and the cursor file
  rotates.
- Watch the next scheduled hourly runs' logs and the axe error digests long enough to
  confirm no new `timed out after 300s` entry for this job, and that the repeated
  `Recovered workspace SDD clone .../repos/beads` warnings stop recurring.
- Record the observed timings in the phase notes.

## Risks and mitigations

- **Batched Rust call turns out slower than expected**: the per-call ~380ms is dominated
  by store open/load/persist, so one open plus in-memory application should land in
  single-digit seconds; the sase-core phase includes a measurement gate to catch this
  early. Even if it lands slow, deadline propagation guarantees the chop degrades into
  logged deferral instead of SIGKILL.
- **Semantic drift in the projection rewrite**: parity tests against the sequential path
  plus the existing `sase-yy.8.3` regression suites (replay idempotency, multi-edge
  baselines, interruption recovery) gate both repos.
- **Deferred publications starve**: deferral keeps entries queued and receipts absent,
  the exact mechanism today's retained entries already use; with batching in place the
  deadline is a safety net, not the steady-state path.
