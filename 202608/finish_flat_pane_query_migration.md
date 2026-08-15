---
tier: tale
title: Finish the shared query migration for every flat Artifacts pane
goal:
  Complete phase sase-m6.6.1.5 by moving Stitches, Beads, Plans, Files, and arbitrary
  document-provider panes from their bespoke matchers to off-thread, profile-driven Rust
  query indexes while preserving their established UI and collection behavior.
size: medium
bead: sase-m6.6.1.5
proposed_by: bbugyi200.athena.026
create_time: 2026-08-15 09:35:14
status: done
---

- **BEAD:**
  [sase-m6.6.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.5.md)

# Plan: Finish the flat Artifacts pane query migration

## Starting point and completion boundary

The earlier work for this phase is already landed in SASE commit `545cb8e70` and linked
`sase-core` commit `f898057`: every `ArtifactsPaneContract` carries a compiled profile,
the host has an immutable Rust-corpus facade with the exact cache-key shape, and
`FilterBar` can derive its dialect from a profile. Preserve that foundation and finish
only the handoff's remaining steps 4–7. The phase is complete when production filtering
for Stitches, Beads, Plans, Files, and arbitrary document providers uses the shared
profile parser/canonicalizer and Rust batch evaluator, with no legacy row matcher on
those panes' hot paths.

Keep these invariants throughout:

- All migrated profiles remain `boolean=false`: whitespace is conjunction, repeated
  positive values of one key are ORed, exclusions are negative constraints, and each
  pane's accepted tokens and canonical order stay compatible except for the explicitly
  added Files negation and closed host-predicate atoms.
- Collection controls remain separate from row matching. Stitches still translates
  project/repository/date/sidecar/merge/limit constraints into the minimum backend
  collection request, and Plans still extends bounded snapshots with deep-archive
  results, but final visible-row membership comes from the shared engine.
- Corpus construction, typed row coercion, provider/frontmatter access, facet
  collection, and cache-miss evaluation run in thread workers. Textual handlers only
  parse bounded input, schedule work, and apply a result after rechecking pane, snapshot
  generation, profile digest, canonical query, and current selection.
- Cache identity is exactly
  `(pane_id, snapshot_generation, profile_digest, canonical_query)`. Pane switches,
  reloads, provider-profile changes, deep-archive extensions, and Files
  first-page/full-page transitions cannot reuse stale results or facets.
- Arbitrary providers consume only their contract's compiled profile and declared
  frontmatter properties. Malformed typed values degrade per row; an invalid provider
  profile remains a visible degraded pane and cannot fall back to the Plan dialect.

## Implementation

1. Finish the flat grammar and compatibility semantics before wiring the UI.
   - Extend the Python reference flat parser and the linked `sase-core` flat parser to
     recognize only profile-enabled closed host predicate atoms (`!`/`!!!`, `!!`,
     `@`/`@@@`, `!@`, `$`/`$$$`, `!$`, and `*`) without enabling Boolean operators or
     parentheses. Keep the Python AST, Rust AST, errors, spans, and canonical flat
     spelling in parity.
   - Enable the host-owned predicate vocabulary on the flat built-in/provider profiles
     that expose row facts; providers select only the closed host implementation and
     never executable matcher behavior. Populate absent facts as false.
   - Enable leading-token negation for every Files filterable field and free-text term.
     Extend `FilesFilterValues`, its legacy parser/renderer/matcher, completion context,
     and compatibility tests with explicit excluded values so existing callers and
     filter chips round-trip the new syntax while production matching moves to the
     shared evaluator.

2. Add immutable pane row adapters and build query indexes with snapshots off-thread.
   - Introduce focused adapters that map Stitches commit/repository metadata, Beads
     task/epic/phase metadata and operational facts, Plans proposal/active/archive
     documents, Files logical/version metadata, and arbitrary provider frontmatter into
     `ArtifactQueryRowInput` records with stable option/entry ids.
   - Derive `searchable_text` only from fields declared searchable in the active
     profile. Preserve legacy aliases and labels as field values where they are part of
     current matching semantics, including project display names, repository aliases,
     derived bead states, plan archive roles, and all logical-file versions.
   - Compile `ArtifactQueryIndex` objects in the existing snapshot/collection/deep-
     archive worker paths and carry their generation alongside the immutable snapshot.
     Feed the resulting observed facets into the profile-configured `FilterBar`; never
     read files, resolve providers, enumerate projects, or rebuild records from a query
     or navigation handler.

3. Give the flat panes one generation-checked query-session path.
   - Add a small shared host/session layer around profile parse/canonicalize,
     `ArtifactQueryCacheKey`, cached `ArtifactQueryResult`, and a thread worker for
     cache misses. Coalesce requests, bound per-pane caches, reject stale completions,
     and cancel/forget workers during teardown or snapshot replacement.
   - Instantiate the existing pane-specific `CommitFilterBar`, `BeadFilterBar`,
     `PlanFilterBar`, and `FileFilterBar` subclasses with their contract profile so DOM
     ids, messages, accents, CSS, and action dispatch stay stable while key/value/
     negation/highlighting metadata comes from the profile and observed facets.
   - Preserve live-edit rollback, submit/Escape behavior, stable selection restoration,
     jump-mode cancellation, match/coverage status, and committed query chips. Invalid
     input must keep the editor open and restore the pre-edit result; a late valid
     worker must not overwrite a newer query error or pane generation.

4. Migrate each production pane without weakening its special behavior.
   - Stitches: replace `compile_commit_matcher` row scans with matched stable commit ids
     from its query index, retain authoritative collection caching and exact/capped
     coverage, and translate validated collection-only constraints into the existing
     backend request without treating that request as the final matcher.
   - Beads: replace `BeadFilterIndex`/`compile_bead_matcher` on the display path with
     shared matched option ids while retaining epic-parent visibility, matched section
     counts, triage/derived status aliases, entry jumps, and snapshot reload behavior.
   - Plans and arbitrary providers: replace `PlanFilterIndex`/`compile_plan_matcher`
     with the contract profile and shared ids for bounded plus deep-archive rows.
     Rebuild or extend the index off-thread when archive coverage changes and preserve
     exact/ preview coverage labels and provider-specific properties/facets.
   - Files: replace `filter_files_snapshot` on the render path with shared matched
     logical ids while retaining first-page/full-index extension, per-version fields,
     project display-name aliases, kind cycling, selection/version restoration, and
     background detail loading.
   - Remove obsolete duplicated completion declarations and production matcher calls
     only after compatibility tests prove the old public parsing/value helpers still
     round-trip. Remove this phase's temporary Symvision `--epic-symbol` allowances once
     all facade symbols have real consumers.

5. Prove cross-language parity, UI safety, and responsiveness.
   - Expand the Artifacts conformance harness over every built-in pane plus the
     synthetic provider with shared canonicalization, Rust/Python match parity,
     profile-derived completion/highlighting/facets, closed predicates, malformed
     provider values, and pane/profile/generation cache isolation.
   - Add focused session tests for live validation, Escape rollback, submit, selection
     restore, exact versus preview coverage, Files negation, deep-archive replacement,
     full-index extension, query-worker coalescing, and stale-result rejection. Assert
     that typing/navigation does not invoke corpus construction, provider/disk access,
     or synchronous result evaluation.
   - Extend `tests/ace/tui/bench_artifacts_jk.py` to exercise Beads and Files in
     addition to Patches, Stitches, and Plans, and include a synthetic provider case
     where useful to prove the generic path. Keep every measured navigation action below
     the existing 16 ms p95 budget.

## Verification and closure

1. Run the focused Rust flat-parser/evaluator and PyO3 binding tests in the opened
   `sase-core` repository while iterating, then run that repository's required
   `just check` gate.
2. Run `just install` in SASE so the editable package uses the final linked binding,
   then run focused query-profile/facade, pane-filtering, contract-conformance,
   synthetic-provider, snapshot-worker, and Artifacts navigation tests.
3. Run the expanded slow Artifacts navigation benchmark and confirm every pane remains
   below 16 ms p95 with no data-scaled work on the Textual event loop.
4. Run SASE's required `just check`. Because this changes the broad Artifacts TUI and
   shared backend boundary, also run `just check-full` through `/sase_monitor` with a
   follow-up action that inspects the result rather than blocking an agent turn.
5. Recheck both worktrees and the bead history against this completion boundary. Append
   a verification note and close only phase bead `sase-m6.6.1.5` with resolution `done`;
   do not close its parent epic.
