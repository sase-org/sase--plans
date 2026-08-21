---
tier: tale
title: Make file-hook dispatch reliable and diagnosable
goal:
  Matching artifact and commit events run their configured hooks or leave durable,
  actionable evidence, and finalization repairs a missed commit dispatch idempotently.
size: medium
proposed_by: bbugyi200.athena.research.0v.final.f0
create_time: 2026-08-21 20:28:20
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.research.0v.final.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.research.0v.final.f0.md)
- **COMMITS:**
  - [4dde458](https://github.com/sase-org/sase/commit/4dde458f593687ee4e8fb6734c1dbd1b0fef1215)
    — fix(file-hooks): record dispatch outcomes and repair missed commit batches

# Plan: Make file-hook dispatch reliable and diagnosable

## Objective

Ensure that a configured file hook either receives every matching artifact/commit event
or leaves durable, actionable evidence explaining why it did not, while retaining the
existing rule that file hooks never gate artifact creation or commits. Repair the missed
`research-highlights` output for the consolidated finalizer report after the
implementation is verified.

## Diagnosis and constraints

- The effective user entry is valid: `sase-research-artifacts@research-highlights`
  supplies the research-sidecar, `ADD`, path, and agent filters, while the chezmoi
  override supplies `bob highlights create --include-id`. `sase file-hook list --json`
  and `sase doctor -C config.file_hooks` currently resolve it successfully.
- The source report registered by `research.0v.final` as
  `file:explicit:f1b93f5b944d5086b78c2dde` reconstructs to an `ADD` event for
  `202608/finalizer_integrity_and_capabilities/finalizer_integrity_and_capabilities.md`
  in the `research` sidecar, attributed to `research.0v.final`; the current matcher
  selects `research-highlights` for that event.
- The run produced no file-hook batch, runner log, run log, or notification after the
  artifact was registered, even though earlier research lead runs did. Its later
  successful `sase stitch create` commit (`203881ee93a9a5ea785c4239483fa4538055980b`)
  also produced no batch.
- The event therefore returned before durable batch creation in both producer paths. The
  exact precipitating branch cannot be recovered: `sase artifact create` and
  `CommitWorkflow._run_file_hooks()` catch every exception with a bare `pass`, while a
  no-hooks or no-match outcome is represented only as `None`. This evidence-destroying
  fail-soft boundary is the confirmed root cause of the incident being undiagnosable and
  allowed the missed dispatch to pass as success.
- Keep hooks post-write and non-gating. A hook configuration, matching, persistence, or
  spawn failure must not turn a successful artifact copy or VCS commit into a failure.
- Do not change the `sase-research-artifacts` provider filters or the chezmoi command:
  they accept the intended consolidated report and are not the defect.
- Artifact registration and the later commit can both describe the same logical file.
  Preserve the existing event-source semantics in this fix; do not add broad
  cross-source deduplication without a separately specified identity and replay
  contract.

## Implementation

1. Introduce a typed dispatch result in `src/sase/file_hooks/engine.py` that
   distinguishes at least: no configured hooks, no matching hooks, batch already
   present, batch dispatched, and producer error. Include the captured event identity,
   matched hook names, batch path/id when available, and a safe diagnostic for errors.
   Keep the existing convenience return behavior only where compatibility requires it;
   route in-repo producers through the typed API.
2. Persist a bounded producer audit record under the existing file-hook state root
   before returning from every configured-event dispatch attempt. The record must make
   zero-match and pre-run failures inspectable without relying on Python logging, must
   not contain secrets or full environment dumps, and must follow the existing
   retention/pruning policy. Continue using batches and run logs as the execution record
   after a successful match.
3. Replace the bare exception swallowing in `src/sase/artifact_cli/create.py` and
   `src/sase/workflows/commit/workflow.py` with one shared, best-effort producer helper.
   It should capture source/repository/agent attribution, request dispatch, record the
   typed outcome, and log unexpected failures with traceback at debug/warning level.
   Artifact creation and commit success must remain unchanged regardless of outcome.
4. Add an idempotent finalizer reconciliation after each successful built-in commit
   marker is verified. Re-derive the committed file events from the marker's repository
   and SHA and ensure the deterministic commit batch exists. If the commit workflow
   already dispatched it, reuse the existing batch without spawning again; if the
   workflow missed it before persistence, this second producer boundary must retry it
   and preserve its own typed outcome. Carry the repository/sidecar and agent
   attribution explicitly rather than depending on a completed-run artifact.
5. Surface producer failures through one non-gating `file-hooks` error notification that
   points to the retained audit record. Do not notify for ordinary no-hooks or
   filter-miss outcomes. Ensure notification creation is itself fail-soft and cannot
   recurse into file hooks.
6. Extend `sase file-hook` inspection with a small read-only history/detail surface (or
   equivalently expose the audit records in the existing list JSON if that remains
   clear) so an operator can answer whether an event was unmatched, dispatched, or
   failed. Document the new evidence and the distinction between producer failures and
   command-run failures in `docs/configuration.md` and the CLI reference as needed.
7. Add focused tests for each typed outcome, audit retention, redacted error evidence,
   and fail-soft notification handling. Replace tests that merely assert the producer
   was called with assertions over the durable outcome.
8. Add an integration regression using a materialized numbered-workspace research
   sidecar, agent metadata, the real `research-highlights` filter shape, and a harmless
   fake command/spawn boundary. Prove separately that:
   - `sase artifact create` dispatches the consolidated report path and attributes it to
     a research lead;
   - the commit workflow dispatches the `ADD` from the resulting sidecar commit;
   - finalizer reconciliation does not refire an existing deterministic commit batch and
     recreates a batch omitted by an injected first-producer failure;
   - draft `__a`/`__b` reports remain ordinary recorded filter misses rather than runs;
   - an injected config/capture/persistence failure leaves an error audit and
     notification while the artifact or commit still succeeds.
9. Run `just install`, the focused file-hook/artifact/commit/finalizer tests,
   `sase validate`, and `just check`. If scoped selection escalates or reports an
   unusual selection, follow the repository guidance for `just check-full` through
   `/sase_monitor`.
10. After all tests pass, use the retained explicit artifact path to invoke the
    effective `research-highlights` command once for the missed consolidated report.
    Confirm the Highlights-ready PDF exists and record the one-time repair in the
    handoff; do not synthesize a fake historical batch or notification.

## Acceptance criteria

- The exact consolidated research event dispatches under both artifact and commit
  integration tests with the effective provider filter shape.
- Every configured-event producer attempt has a durable outcome explaining dispatch,
  filter miss, or producer failure; unexpected failures are no longer swallowed.
- A successful built-in finalizer commit idempotently reconciles its deterministic
  commit batch, repairing a missed first dispatch without duplicating a successful one.
- Producer failures create actionable but non-gating notifications, while ordinary
  exclusions remain quiet.
- Artifact creation and commits retain their prior success semantics when hook loading,
  matching, audit persistence, runner spawn, or notification creation fails.
- Existing file-hook batches remain idempotent and backward-compatible.
- The missed consolidated report receives exactly one deliberate backfill invocation,
  and no provider or chezmoi filter change is required.
