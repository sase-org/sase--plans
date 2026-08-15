---
tier: tale
title: Complete and verify the flat Artifacts pane query migration
goal:
  Finish sase-m6.6.1.5 by landing the valid prior migration work, moving every flat
  Artifacts pane onto generation-checked off-thread shared query sessions, and proving
  compatibility, predicate parity, provider isolation, and navigation responsiveness.
size: medium
proposed_by: bbugyi200.athena.02i
create_time: 2026-08-15 13:04:51
status: done
---

- **PROMPT:**
  [prompts/202608/complete_flat_pane_query_migration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/complete_flat_pane_query_migration.md)
- **AGENTS:**
  - [bbugyi200.athena.02i](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.02i.md)
- **COMMITS:**
  - [c62765e](https://github.com/sase-org/sase/commit/c62765eb7f3bca0e3a171ab923a5c25e4d6554e4)
    — feat(ace): complete flat artifact query migration

# Plan: Complete and verify the flat Artifacts pane query migration

## Completion boundary

Complete only phase bead `sase-m6.6.1.5`; do not close its parent epic. The landed
foundation at SASE commit `545cb8e70` already provides compiled pane profiles, the
Rust-corpus facade and exact cache-key type, and profile-configured FilterBar behavior.
The linked `sase-core` host-predicate support is already committed. Workspace 11 holds
an uncommitted continuation that is useful as a starting point, but it currently builds
and evaluates corpora synchronously in refresh/display paths, supplies no real host
predicate facts, and omits the required conformance and benchmark extensions. Port and
correct that work selectively; do not treat its green focused tests as completion.

The phase is complete only when production filtering for Stitches, Beads, Plans, Files,
and arbitrary document-provider panes uses each pane contract's compiled profile and
Rust batch evaluator, with data-scaled work off the Textual event loop and late results
rejected by exact pane/generation/profile/query identity.

## Invariants

- Keep all migrated profiles in flat `boolean=false` mode. Preserve legacy token
  spellings, quoting, repeated-positive OR behavior, exclusions, dates, aliases,
  canonical order, rollback, selection restoration, and coverage labels except for the
  explicitly added Files negation and closed host-predicate atoms.
- Keep collection controls separate from row membership. Stitches may still translate
  project/repository/date/sidecar/merge/limit constraints into backend collection
  requests, and Plans may extend a bounded snapshot through deep-archive loading, but
  the shared evaluator decides visible rows.
- Build typed row inputs, searchable text, observed facets, and Rust corpus handles in
  snapshot/collection/deep-archive worker threads. Evaluate cache misses in thread
  workers too; handlers and render/navigation paths may only parse bounded text,
  schedule/coalesce work, and apply already-computed results.
- Key cached and in-flight results by
  `(pane_id, snapshot_generation, profile_digest, canonical_query)`. Recheck the active
  pane/session and that exact identity before UI mutation; snapshot replacement,
  deep-archive growth, Files full-index extension, pane switches, provider-profile
  changes, and invalid/newer queries invalidate late work.
- Derive provider behavior solely from its `ArtifactsPaneContract` profile and declared
  properties. Malformed typed values degrade per row, while an invalid provider profile
  remains visibly degraded and never falls back to the Plan dialect.
- Populate the closed host predicate facts (`error_suffix`, `running_agent`, and
  `running_process`) from authoritative pane metadata and treat absent facts as false.

## Implementation

1. Rebase the usable workspace-11 continuation onto the current clean checkout and audit
   every hunk against the completion boundary. Preserve the Python/Rust flat grammar
   parity, Files compatibility-value negation, profile declarations, and row adapters
   that pass focused compatibility tests. Drop synchronous corpus/evaluation calls from
   UI refresh/display paths, empty predicate tuples, pane-specific profile lookups where
   the contract is available, and any broad matcher-test deletion that loses
   compatibility coverage.

2. Add one bounded, generation-checked flat query-session abstraction around
   `ArtifactQueryIndex`, canonicalization, `ArtifactQueryCacheKey`, and
   `ArtifactQueryResult`. Compile indexes with owning snapshots and evaluate cache
   misses via `run_worker(..., thread=True)` or an equivalent pump-free worker path;
   coalesce identical requests, bound per-pane result caches, cancel/forget workers at
   teardown or snapshot replacement, and expose a deterministic apply contract that
   rejects stale pane, generation, profile digest, canonical query, or editor state.

3. Finish immutable row/index construction for every flat pane. Map Stitches commit,
   repository, collection, and host-state metadata; Beads task/epic/phase fields and
   operational facts; Plans proposal/active/archive fields; Files logical/version
   metadata; and arbitrary provider frontmatter properties into stable shared row ids.
   Derive free text only from profile-declared searchable fields, preserve current
   display-name/repository/status/archive aliases, compile indexes during existing
   worker collection paths, and feed observed facets to the profile-configured
   FilterBars without disk/provider/project work on keystrokes.

4. Route Stitches, Beads, Plans/providers, and Files sessions through the shared
   asynchronous evaluator while preserving their special behavior: Stitches collection
   cache and exact/capped coverage, Beads parent visibility and section counts, Plans
   deep-archive exact/preview replacement, provider-specific profiles and malformed-row
   degradation, and Files first-page/full-index extension plus logical/version selection
   restoration. Preserve invalid-query rollback, submit/Escape semantics, committed
   chips, jump cancellation, and selection identity. Remove obsolete production matcher
   calls and temporary Symvision allowances only after their public compatibility
   helpers remain covered.

5. Complete the closed predicate and Files-negation proof across Rust, Python reference,
   compatibility values, and TUI rows. Assert enabled-only predicate parsing, canonical
   spelling, spans/errors, truth tables, absent-fact behavior, leading negation for
   every Files field and free-text term, and Rust/Python match parity.

6. Extend the Artifacts conformance harness across Stitches, Beads, Plans, Files, and a
   synthetic provider. Cover canonical compatibility, profile-derived completions,
   highlighting and facets, malformed provider values, predicate facts, cache
   isolation/invalidation, query-worker coalescing, stale-result rejection, rollback,
   selection restoration, deep-archive/full-index replacement, and assertions that
   typing/navigation performs no synchronous corpus, provider, disk, or evaluation work.
   Extend `tests/ace/tui/bench_artifacts_jk.py` to measure Beads and Files (plus a
   synthetic provider if the harness supports it) alongside existing panes at the
   established p95-under-16-ms budget.

## Verification and closure

1. In the opened linked `sase-core` checkout, run focused flat parser/evaluator and PyO3
   binding tests, formatting, and its required check gate; confirm the committed host
   predicate wire/API matches the Python binding consumed by SASE.
2. Run `just install` in SASE, then iterate through focused profile/reference/facade,
   compatibility parser, pane filtering/session, snapshot worker, provider-contract,
   conformance, stale-generation, and navigation tests.
3. Run `pytest -s -m slow tests/ace/tui/bench_artifacts_jk.py` and require every
   measured pane action to stay below 16 ms p95.
4. Run SASE's required `just check`. Because this touches the broad Artifacts TUI and
   shared backend boundary, run `just check-full` through `/sase_monitor` with a
   same-turn follow-up that inspects the terminal result. Fix and repeat until all
   relevant checks pass.
5. Reinspect both repository statuses and the bead history against this completion
   boundary. Record a concise verification note and close only `sase-m6.6.1.5` with
   resolution `done`; leave the parent epic for its land agent.
