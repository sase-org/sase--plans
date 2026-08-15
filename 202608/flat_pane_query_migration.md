---
tier: tale
title: Migrate every flat Artifacts pane to the shared query profile
goal:
  Stitches, Beads, Plans, Files, and arbitrary document providers share one
  profile-driven parser, evaluator, FilterBar configuration, and pane-isolated cache
  while preserving their established flat-query behavior.
size: medium
proposed_by: bbugyi200.athena.sase-m6.6.1.5
bead: sase-m6.6.1.5
create_time: 2026-08-15 08:06:48
status: done
---

- **PROMPT:**
  [prompts/202608/flat_pane_query_migration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/flat_pane_query_migration.md)
- **PARENT:** [202608/unified_artifacts_query_1.md](unified_artifacts_query_1.md)
- **BEAD:**
  [sase-m6.6.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.5.md)

# Plan: Migrate every flat Artifacts pane to the shared query profile

## Goal

Complete phase `sase-m6.6.1.5` by making Stitches, Beads, Plans, Files, and arbitrary
document-provider panes consume the same compiled query profile, Rust parser/corpus, and
Python reference semantics. Preserve each existing flat dialect's source and canonical
forms, keep collection-only controls working, enable Files negation and the closed host
predicates, and ensure profile/corpus/result work is isolated by pane and snapshot
generation without blocking Textual's event loop.

## Grounding and invariants

- Build each query profile from the pane's `ArtifactsPaneContract`; built-ins use their
  host schema and providers derive fields from `ref.properties` without provider code.
- Keep every migrated pane in `boolean=false` mode. Whitespace remains conjunction,
  repeatable values retain their legacy OR semantics, and existing quoting, validation,
  dates, project aliases, and canonical token order remain compatible.
- Treat Stitches controls that affect collection (`project`, date bounds, sidecar,
  merges, and limit) and Plans deep-archive coverage as adapter responsibilities around
  the shared parser/evaluator, not as alternate query languages.
- Precompute typed fields, searchable text, completion facets, and Rust corpus handles
  when snapshots are built on worker threads. Key reusable state and evaluated results
  by `(pane_id, snapshot_generation, profile_digest, canonical_query)` and reject stale
  worker results before touching the UI.
- Keep live editor feedback responsive: query parsing/highlighting is bounded and pure;
  corpus construction, row conversion, provider data access, and result evaluation do
  not run in render/navigation paths or slow Textual pump callbacks.
- Provider rows degrade malformed typed properties per entry, while an invalid profile
  remains a visible pane diagnostic and cannot poison another pane's cache.

## Implementation

1. Promote the contract's query placeholder into the real schema/profile boundary.
   Compile the appropriate built-in or provider `ArtifactQuerySchema` during contract
   construction, store the immutable compiled profile on the contract, include its
   deterministic payload/digest in contract explanation and presentation identity, and
   cover healthy, degraded, built-in, and synthetic-provider contracts.

2. Finish the host facade over the landed Rust profile API. Add generic corpus compile,
   profile parse/canonicalize, and corpus evaluation wrappers while retaining Patch
   compatibility entry points. Define immutable query-row/index/result records with
   stable row ids, observed per-field facet values, generation metadata, and exact cache
   keys. Keep a Python-reference path for parity tests, but route production flat panes
   through Rust handles.

3. Make the shared `FilterBar` instance-configurable from a compiled profile. Derive key
   descriptions, static values, repeatability, negation, free-text hints, accent,
   completion context, and syntax highlighting from the profile, then merge observed
   facet values from the current pane index. Preserve the existing pane-specific
   message/DOM ids where tests and styles depend on them, but remove duplicated dialect
   declarations from the Stitches, Beads, Plans/provider, and Files subclasses.

4. Add row adapters for all flat panes and build their indexes off-thread with the
   owning snapshot. Map Stitches commit metadata and predicate facts, Beads issue/task
   metadata and launched-agent facts, Plan/proposal/archive fields, Files
   logical/version metadata, and arbitrary provider frontmatter properties into the
   shared row shape. Derive free text only from fields marked searchable and expose
   declared/observed facets without filesystem or provider resolution on keystrokes.

5. Migrate each filter session to shared parse/canonical/evaluate results while
   retaining its current session rollback, selection restoration, coverage, and
   background-load behavior. Use generation-checked thread workers for cache misses and
   apply only the latest active-pane result. Translate validated shared queries into the
   minimal legacy collection specification needed by Stitches and Plans rather than
   invoking their old row matchers. Ensure provider panes use their own profile instead
   of the Plan dialect, and ensure pane switches cannot reuse another pane's results or
   facets.

6. Enable negation for Files fields and free-text terms in its profile and compatibility
   value representation, and add the closed host predicate atoms to flat parsing and row
   facts consistently in Rust and Python where the landed engine does not yet support
   them. Preserve canonical spelling and extend Rust/Python parity coverage before using
   the behavior in the TUI.

7. Extend the Artifacts conformance harness and focused tests across Stitches, Beads,
   Plans, Files, and the synthetic provider. Cover canonical compatibility, shared Rust
   versus Python result parity, profile-derived completions/highlighting/facets, Files
   negation, host predicates, malformed provider values, cache isolation and
   invalidation, stale-worker rejection, selection rollback, and absence of
   provider/disk/profile work on typing and navigation paths.

## Verification

1. Run `just install` before repository checks so the editable Python package and linked
   Rust binding reflect the current trees.
2. If the shared flat grammar requires a `sase-core` correction, run its focused query
   and PyO3 binding tests, then reinstall the binding before host tests.
3. Run focused profile, query-facade, filter-session, contract-harness,
   synthetic-provider, Files-negation, and Artifacts pane tests while iterating.
4. Run the Artifacts navigation performance benchmark/trace checks and verify no query
   profile compilation, row construction, resolver, filesystem, or Git work occurs on
   the event loop; preserve the p95 target below 16 ms.
5. Run `just check` for the SASE repository and re-run any focused Rust checks after the
   final host integration change. Record any unrelated discovered issue only as a
   `PROPOSED FOLLOW-UP:` note on the assigned phase bead.
