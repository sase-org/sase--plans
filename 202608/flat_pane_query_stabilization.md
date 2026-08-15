---
tier: tale
title: Stabilize and close the flat Artifacts query migration
goal:
  Reconcile the landed flat-pane query migration with current master, fix its remaining
  selection, visual, compatibility, or verification regressions, and close only
  sase-m6.6.1.5 after focused, performance, and repository checks pass.
size: medium
proposed_by: bbugyi200.athena.sase-m6.6.1.5
bead: sase-m6.6.1.5
create_time: 2026-08-15 18:46:35
status: done
---

- **PROMPT:**
  [prompts/202608/flat_pane_query_stabilization.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/flat_pane_query_stabilization.md)
- **PARENT:** [202608/unified_artifacts_query_1.md](unified_artifacts_query_1.md)
- **BEAD:**
  [sase-m6.6.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.5.md)

# Plan: Stabilize and close the flat Artifacts query migration

## Current state and completion boundary

Phase `sase-m6.6.1.5` owns the flat-query migration for Stitches, Beads, Plans, Files,
and arbitrary document-provider panes. Commits `d580a55c8` and `c62765eb7` already
landed the remaining row adapters, generation-checked off-thread query sessions, Files
negation, host-predicate parity, conformance coverage, and Beads/Files navigation
benchmark extensions on current master. Earlier commits already supplied the compiled
pane profiles, Rust corpus facade, and profile-driven `FilterBar`.

This continuation does not redesign that architecture. It proves the landed migration
against the current Rust binding and current master, repairs only regressions caused by
the migration, and closes only `sase-m6.6.1.5`. It must not close parent epic
`sase-m6.6.1`, ancestor `sase-m6.6`, or any other bead. Any unrelated work discovered
while validating is recorded on this phase as `PROPOSED FOLLOW-UP: <summary — detail>`
rather than becoming a new bead.

## Invariants

- Keep every migrated pane in flat `boolean=false` mode, preserving its legacy source
  and canonical token forms, repeated-positive OR behavior, date and project aliases,
  rollback, coverage labels, deep-archive/full-index behavior, and stable selection.
- Keep the shared production path: pane-contract profile, immutable row adapter,
  worker-built Rust corpus, exact
  `(pane_id, generation, profile_digest, canonical_query)` cache identity, and
  generation-checked off-thread evaluation. Do not restore bespoke row matchers or put
  corpus/evaluation work on Textual's event loop or serial pump callbacks.
- Preserve the intentional additions: Files field/free-text negation, closed host
  predicates, provider-derived properties/facets, and malformed-provider-value
  degradation per row.
- Treat visual updates as evidence, not cleanup: inspect actual/expected/diff artifacts
  and accept a golden only when the rendered change follows from the intended shared
  query/selection behavior.

## Implementation

1. Run `just install` before tests so the editable package and `sase_core_rs` match the
   current compatibility floor. Establish a focused baseline for the query facade,
   cross-language conformance, flat pane filtering/session/navigation tests, and the
   affected Artifacts PNG nodes. Confirm whether the previously reported compiled
   profile digest error is already resolved by the current `sase-core-rs>=0.27.7` floor;
   use `sase repo open sase-core` before reading or changing Rust only if a reproducible
   backend defect remains.

2. Fix migration-caused compatibility failures at their narrowest owner. In particular,
   update stale Beads visual/fixture callers to use canonical `ArtifactEntryTarget`
   identities where the production API is intentionally typed; preserve runtime
   compatibility only where a real non-test consumer still supplies a legacy target.
   Diagnose the Files nested-pane PNG delta and any Commits fixture import failures
   against the migration commits, correcting behavior or fixtures as evidence dictates
   without weakening assertions.

3. Re-run and extend focused proof for the shared path when a gap is found: Rust versus
   Python canonical/evaluation parity for all built-in flat profiles and the synthetic
   provider, Files negation, host-predicate truth tables, observed facets, cache
   isolation/invalidation, stale-worker rejection, query rollback, selection restore,
   and provider/deep-archive/full-index replacement. Keep all data-scaled index and
   evaluation work in worker threads and re-check exact pane/generation/profile/query
   identity before applying results.

4. Run the dedicated affected visual tests and inspect generated artifacts before
   accepting any intentional snapshots. Run
   `pytest -s -m slow tests/ace/tui/bench_artifacts_jk.py` and require every measured
   Artifacts action to remain below the established 16 ms p95 budget, with Beads and
   Files included.

## Verification and closure

Run `just check` after the final SASE file changes. Because this phase spans broad
Artifacts TUI and shared backend/query contracts, also run `just check-full` only
through the `sase_monitor` skill with a continuation that inspects the terminal result;
fix and repeat relevant failures until clean. Reinspect the working tree and bead
history, record unrelated findings only as proposed follow-ups, then close exactly
`sase-m6.6.1.5` with `sase bead close sase-m6.6.1.5 --note "<what was verified>"`.
