---
tier: tale
title: Add type filtering to the Artifacts Stitch pane
goal:
  Users can filter the Stitch timeline by provenance, concrete SASE operation type,
  merge structure, and Patch association through one responsive type query facet.
size: medium
proposed_by: bbugyi200.athena.0bw
---

- **AGENTS:**
  - [bbugyi200.athena.0bw](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0bw.md)
- **COMMITS:**
  - [4612a84](https://github.com/sase-org/sase/commit/4612a84c896e912c021788cc212d2e843e20242c)
    — feat(vcs): add stitch type filtering

# Add `type:` filtering to the Artifacts Stitch pane

## Goal

Add a first-class `type:<type>` filter to ACE's Artifacts → Stitch pane so a user can
narrow the loaded commit timeline by meaningful Stitch/commit classifications without
doing new I/O while typing. The query must remain compatible with the pane's existing
`origin:`, `merges:`, free-text, project/repository, date, sidecar, and `limit:`
behavior.

## User-visible contract

- Accept repeatable, comma-list `type:` terms and their negated form, using the same
  flat-query semantics as `repo:`, `author:`, and `origin:`. Multiple positive values
  are ORed within the facet, different facets are ANDed, and an overlapping negative
  value wins. Examples: `type:manual`, `type:automatic,sdd`, `type:stitch -type:patch`,
  and `type:merge`.
- Match values case-insensitively and canonicalize `type:auto` to the user-facing
  `automatic` classification. Do not enum-reject unknown values: concrete `SASE_TYPE=`
  values are extensible, so a future or plugin-authored value must be queryable
  immediately.
- Give each commit a de-duplicated set of type labels:
  - exactly one provenance label: `manual`, `automatic`, or `stitch`, derived with the
    existing legacy-aware origin rules;
  - the normalized terminal `TYPE`/`SASE_TYPE` footer value when present (for example
    `sdd`, `init`, `xprompt`, `bead_work`, or `stitch`);
  - `merge` when the commit has multiple parents;
  - `patch` when the terminal footer has a non-empty legacy `PATCH` or current
    `SASE_PATCH` association.
- Preserve `origin:stitch|auto|manual` as the mechanism/provenance filter and
  `merges:hide|show|only` plus the `z` cycling action as the collection-level merge
  visibility controls. `type:` is an additional row facet: it can express
  structural/Patch associations and concrete automation kinds and can be combined with
  the older filters.
- Offer `type:` in filter-key completion. Seed completion with `manual`, `automatic`,
  `stitch`, `merge`, and `patch`, merge in concrete type values observed in the
  already-built snapshot, and support completion after commas and after a leading `-`
  without filesystem, subprocess, or parser work on the Textual event loop.
- Treat the new query-profile digest as an intentional dialect change. Existing stale
  saved-query/history records must continue to follow the current visible stale-profile
  handling rather than being silently reinterpreted.

## Implementation

1. In the linked `sase-core` repository, add a provider-neutral commit-type classifier
   next to the existing VCS-log origin classifier. Parse the terminal commit footer once
   with the shared footer grammar, reuse the current legacy origin precedence, add
   structural `merge` and footer-based `patch` labels, normalize/deduplicate labels in
   deterministic order, and expose the classifier through `sase_core` and the
   `sase_core_rs` PyO3 module. Keep the existing VCS-log wire shape and schema version
   unchanged; this is a derived query facet, not persisted commit data.

2. In the `sase` repository's `sase.core.vcs_log_facade`, add the typed host wrapper for
   the new binding plus a parity/golden implementation used only to verify the
   Rust-owned behavior. Accept a `VcsCommitWire` (or its full message and merge flag at
   the private seam) so callers cannot disagree about subject/body joining or merge
   detection. Cover current and legacy footer spellings, terminal-footer-only parsing,
   unknown concrete type values, duplicate/footer precedence, legacy stitch inference,
   merges, Patch associations, and combinations such as a tracked Patch merge.

3. Extend `CommitLogFilterValues` and the handwritten Stitch query parser/renderer with
   positive and excluded type tuples. Add `type` to completion-context, repeatable, and
   negatable key sets; accept arbitrary non-empty values; normalize the `auto` alias;
   and ensure parse → canonicalize → parse round trips preserve type selections. Keep
   these values presentation-only when building `CommitFilterSpec`, because the generic
   Rust artifact-query index—not each VCS provider—owns row matching.

4. Extend `stitches_query_schema()` with an exact, repeatable, negatable string `type`
   field. Give it the stable coarse completion values but leave it open to observed
   concrete values. Populate `_commit_query_entry()` with the shared classifier's labels
   while the snapshot/query index is already being built on a worker thread. Feed the
   query index's `type` facet into `CommitFilterBar` alongside repository and author
   sources, without replacing configured project completions.

5. Reconcile collection and preview correctness. Strip only true collection/scope tokens
   before Rust row evaluation, so `type:` reaches the query engine. Treat positive or
   negative type terms like origin/free-text row filters when choosing the backend
   collection limit: collect enough uncapped candidates before applying the host
   `limit:` slice so older matching rows are not lost behind newer nonmatches. Update
   snapshot breadth/coverage comparisons only where necessary so an untruncated broader
   snapshot can satisfy type-only previews and a truncated snapshot never claims exact
   coverage.

6. Update the Stitch help section and the ACE/configuration query documentation to
   describe `type:`, its multi-label semantics, the concrete-versus-coarse values,
   negation/comma behavior, and its relationship to `origin:` and `merges:`. No default
   query change and no feature flag are needed because the new syntax is additive and
   will ship complete with its parser, completion, evaluator, help, and tests.

## Tests and verification

- Add Rust unit/parity and PyO3 binding tests for the classifier contract and Python
  facade parity, including manual, automatic concrete types, tracked stitches, merge,
  Patch, legacy unprefixed tags, malformed/non-terminal tag-shaped text, and multi-label
  de-duplication.
- Extend `tests/test_vcs_log_filter_query.py` and query-profile tests for parsing,
  aliases, comma/repeated values, negation, invalid empty values, canonical round trips
  (including the Hypothesis strategy), field/profile parity, and the expected
  profile-digest change.
- Extend the Artifacts query corpus golden for Stitch rows with multi-valued `type`
  fields and positive/negative cases, then run the Python-reference/Rust-evaluator
  conformance suite to prove exact matching and canonicalization remain aligned.
- Extend Stitch pane/filter-bar tests with observed type completions and an interaction
  case that filters a mixed snapshot across `manual`, `automatic` plus a concrete
  automation type, `stitch`, `merge`, and `patch`. Assert that type filtering uses an
  uncapped backend candidate set, previews from the in-memory index, preserves
  selection/restore behavior, and composes with `origin:`/`merges:`/`limit:`.
- Update help/documentation assertions. Add or refresh a Stitch filter-completion visual
  snapshot only if the existing snapshot suite intentionally captures the new candidate
  row; inspect any changed PNG before accepting it.
- From `sase`, run `just install` so the editable environment builds the opened linked
  core, run the focused Rust/core/query/TUI tests while iterating, then run
  `just check`. Because this change crosses the Rust boundary and alters a built-in
  query profile, finish with `just check-full` through `/sase_monitor` and run
  `just test-visual` when a visual snapshot changes.

## Non-goals

- Do not add a `sase stitch list --type` CLI flag or push type matching into individual
  Git/VCS providers; the requested surface is the Artifacts Stitch sub-tab and its
  shared in-memory query corpus.
- Do not remove or rename `origin:` or `merges:`, change their defaults, alter the
  default Stitch query, or change timeline/type-chip rendering.
- Do not hard-code a closed list of all possible `SASE_TYPE` operation values; only the
  coarse semantic labels are stable, while observed concrete values remain extensible.
