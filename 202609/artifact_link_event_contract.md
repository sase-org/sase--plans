---
tier: tale
title: Immutable artifact-link event contract and reducer
goal:
  Artifact-link operations have a validated content-addressed wire and deterministic
  Rust reduction API exposed to Python for later event-store phases.
size: medium
proposed_by: bbugyi200.athena.sase-yy.2
bead: sase-yy.2
status: done
---

- **PARENT:** [202609/artifact_link_events_v2.md](artifact_link_events_v2.md)
- **BEAD:**
  [sase-yy.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yy/sase-yy.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-yy.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-yy.2.md)
- **COMMITS:**
  - [528c3db](https://github.com/sase-org/sase-core/commit/528c3dbd7ee3dd6a1a6de221287cb73d1b37b7ac)
    — feat: add artifact link event contract

# Plan: Immutable artifact-link event contract and reducer

Implement phase `sase-yy.2` as an additive, inert Rust-core API. The work defines the
immutable event wire, canonical content addressing, alias resolution, deterministic
event-set reduction into the existing `ArtifactLinkRowWire`, and Python bindings. It
must not switch any SASE writer or reader to the event path; later epic phases own that
integration. Preserve the publication-policy APIs already present in the artifact-link
module.

## Rust event contract

Add a focused module beside `crates/sase_core/src/artifact_link/wire.rs`, re-exporting
its public wire types and functions through `artifact_link/mod.rs` and
`crates/sase_core/src/lib.rs`.

Define schema-v1 serde wires with an envelope containing:

- `schema_version`, non-empty canonical `project_key`, and a validated 32-character
  lowercase-hex `operation_id`;
- canonical relation-aware edge identity for edge-bearing events, using the same
  directed/undirected normalization as `artifact_link_dedup_key`;
- `created_by`, `origin`, and `created_at` provenance;
- a tagged `kind` payload for:
  - `observation`: description plus a positive explicit occurrence count (one for an
    ordinary read; prompt batches may carry more under the same stable operation id);
  - `edge-put`: description and a normalized set of operation ids observed and
    superseded by the assertion;
  - `edge-remove`: the edge identity and a normalized set of operation ids observed by
    the remover;
  - `alias`: an old canonical artifact ref mapped to a distinct new canonical ref;
  - `baseline-import`: import identity, source-head evidence, and per-edge legacy rows
    whose existing uses/provenance are preserved without inventing observations.

Reject unsupported schemas, invalid identifiers, blank provenance, bad/self edges, zero
occurrence counts, duplicate or malformed predecessor ids, alias self-maps, and
incompatible payload/edge combinations. Canonicalization must normalize refs, relations,
undirected endpoint order, id lists, and baseline row order so equivalent inputs produce
one wire value.

## Canonical bytes, digest, and path

Expose functions to validate/canonicalize an event, serialize it as compact UTF-8 JSON
with object keys sorted recursively and exactly one trailing newline, compute the
lowercase SHA-256 over those exact bytes, and derive/validate the immutable path
`link-events/v1/<first-two-digest-chars>/<digest>.json`. Reject non-canonical bytes,
digest/path mismatches, and any same-path content mismatch rather than accepting or
rewriting corrupt content. JSON numbers must remain integer-only through this wire; no
floating-point fields are allowed.

## Deterministic reduction

Implement `reduce_link_events(events, aliases)` returning the existing row projection,
plus enough retained deterministic metadata for later readers/doctor code to inspect
competing/superseded versions and tombstones without reimplementing causality in Python.
Reduction must:

- validate and globally deduplicate exact duplicate deliveries by `operation_id`,
  accepting identical owner copies and rejecting reuse of an id for different events;
- resolve alias chains before edge grouping, reject cycles/conflicting alias targets,
  and ensure a late event naming an old ref reduces under the terminal identity;
- calculate `uses` as the saturating sum of distinct observation occurrence counts and
  baseline counts, never counting duplicate delivery twice;
- apply observed-remove/add-wins semantics: a tombstone removes only the event versions
  it explicitly names, remains effective if it arrives before them, and cannot erase a
  concurrent unseen addition;
- treat an `edge-put` as causally superseding only its named versions; among surviving
  concurrent descriptions choose the causally latest version, then use lexicographic
  operation id as the documented deterministic tie-break while retaining all versions in
  reduction metadata;
- produce validated `ArtifactLinkRowWire` values in stable relation-aware edge-key order
  with stable provenance/origin selection, independent of input order and duplicate
  delivery.

Keep baseline imports one-shot by import identity and reject conflicting reuse. Define
how observation-only histories obtain a row description/provenance and cover mixed
baseline/observation/put/remove histories explicitly in tests.

## Python binding contract and SASE floor

Expose schema-version, validate/canonicalize, canonical-bytes/digest/path-validation,
alias-resolution, and reduction helpers from `crates/sase_core_py/src/lib.rs`. Use
ordinary JSON-shaped dictionaries/lists for wire interchange and map structured
`ArtifactLinkError` failures to `ValueError`, matching the existing artifact-link
bindings. Add every public function to the module registration and its binding smoke
test.

In the SASE repository, add the new binding names and semantic probes to
`tools/validate_sase_core_rs` so an older or behaviorally incompatible wheel fails the
contract check. Ratchet the `sase-core-rs` minimum to the release that actually contains
this API; do not claim the already-published `0.32.55` as containing post-tag changes.
If that release is not yet available during phase work, keep the lockfile installable
and leave a precise `PROPOSED FOLLOW-UP:` note on `sase-yy.2` identifying the release
and floor update the land agent must perform, rather than pointing the requirement at a
wheel without the contract.

## Tests and verification

Add focused Rust unit tests and deterministic permutation/duplication tests covering all
event kinds and validation failures. At minimum prove byte-for-byte canonical JSON and
path hashing, directed versus undirected edge identity, exact-retry dedup versus
distinct observations, concurrent put tie-breaking, remove-before-add/add-wins, baseline
preservation, alias chains, alias cycles/conflicts, operation-id collisions, and equal
reduction output for reordered and duplicated event sets. Add PyO3 binding
round-trip/error tests and SASE-side validator tests for required names/schema and a
small behavioral reduction probe.

Run `cargo fmt --all`, then `just check` in `sase-core` (the workspace gate includes the
PyO3 tests; do not substitute a core-only test). In SASE, read the required
`lint_and_test.md` memory after tracked edits, install/rebuild the local core extension
as its runbook requires, run the focused validator tests, and run `just check`. Before
closing only `sase-yy.2`, run `sase bead epic-symbols sase-yy.2` and resolve or re-key
every remaining symbol, then close with a note listing the gates and reducer properties
verified. Do not close the parent epic or any other phase.
