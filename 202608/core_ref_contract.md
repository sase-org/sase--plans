---
tier: tale
title: Add the shared artifact-reference and filter contract to sase-core
goal:
  Make Rust the versioned authority for ref source placement, document filtering,
  resolution, canonicalization, inventories, and native completion.
proposed_by: bbugyi200.athena.sase-ho.1
bead: sase-ho.1
create_time: 2026-08-08 13:46:06
status: wip
---

- **PROMPT:**
  [prompts/202608/core_ref_contract.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/core_ref_contract.md)
- **PARENT:**
  [202608/artifact_reference_xprompts.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_reference_xprompts.md)
- **BEAD:**
  [sase-ho.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ho/sase-ho.1.md)

# Add the shared reference and filter contract to sase-core

## Goal

Implement phase `core-ref-contract` of the Artifact reference xprompts epic in the
linked `sase-core` repository. Rust must become the sole authority for reference source
placement, document path-filter semantics, filtered resolution and canonicalization, and
the artifact inventories used by both `@kind:` and `#ref/kind` completion. The resulting
wire changes must be explicit, versioned, available through PyO3, and ready for the SASE
Python integration phase.

## Contract to preserve

- Physical renderer definitions live in `refs/`; the public namespace is the contextual,
  singular `ref/<kind>` namespace and never includes a project prefix.
- Existing memory and skill layout, placement, catalog, and completion behavior must
  remain unchanged.
- Built-in artifact kinds keep their canonical inputs: `commit: line`, `file_path: path`
  for chat and document kinds, `bug: line`, `artifact_id: line`, `bead_id: word`, and
  `agent_name: agent`.
- Document filters match normalized repo-relative POSIX paths case-sensitively. Positive
  patterns are ORed, matching negations veto them, negative-only filters start from
  allow-all, `**/` includes zero directories, and an explicit empty list allows nothing.
  Unsafe absolute/backslash/parent-traversing payloads remain invalid before matching.
- Filtering applies independently per document root. A file rejected by a matching
  role's policy must return the stable `filtered` status and must not fall through to a
  duplicate root to bypass that policy.
- Exact resolution, drift candidates, absolute-path canonicalization, `@` payload
  inventory, bounded-candidate filtering, and `#ref/` argument completion must all use
  the same Rust matcher.
- Wire/schema mismatches must fail clearly rather than deserialize silently.

## Implementation

1. Extend `crates/sase_core/src/content_layout.rs` with the canonical `refs`
   directory/namespace constants, `refs` paths on project/home/chezmoi layouts, ordered
   ref-source records for project, home, project-specific home, plugin, and package
   scopes, and helpers for contextual `ref/<kind>` naming. Add two-way ref placement
   validation so a ref source requires `ref: true` and ordinary xprompt sources cannot
   claim ref definitions or the reserved namespace. Bump the content layout schema and
   export all new records/helpers from `lib.rs`.

2. Extend the native xprompt catalog and editor-assist wires with additive ref metadata
   (`kind: ref`, `ref_kind`, and existing source provenance). Load Markdown ref sources
   using filename-derived kinds and placement diagnostics, keep their names contextual,
   and synthesize the registry-owned single canonical input rather than accepting a
   renderer-defined input contract. Ensure old catalog payloads without ref metadata
   remain readable while ref entries round-trip consistently through the Rust loader,
   helper bridge, editor assist conversion, and PyO3 JSON surfaces.

3. Add an artifact-context schema version and `path_globs` to
   `ArtifactRefDocumentRootWire`, update parse/resolution schema constants where the
   contract changes, and validate the context version at all public operations. Give
   callers an explicit helper/constant for constructing the current schema and reject
   old or newer context payloads with actionable version errors. Preserve a caller's
   distinction between omitted/default policy and an explicitly empty filter list.

4. Implement one compiled POSIX path-filter abstraction in the pure Rust core. Use
   deterministic positive/negative evaluation and expose both single-path and bounded
   batch APIs. Return validation errors for malformed patterns and invalid payloads;
   never expose absolute local paths in a filtered diagnostic or serialized candidate
   list.

5. Refactor document resolution to retain each root's filters. Check the authored
   normalized path before exact lookup, check every drift candidate before accepting or
   reporting it, and stop duplicate-root fallback when a role policy rejects the
   payload. Add the stable `filtered` result with a deterministic diagnostic carrying
   the kind, normalized payload, and active filter summary. Apply the same matcher
   before canonicalizing an absolute path to a document reference.

6. Filter document scans in the native artifact payload inventory without weakening its
   depth/visit/result bounds. Add a batch PyO3 binding for Python callers that already
   own candidate paths so later phases can share Rust semantics without rescanning or
   reimplementing globs.

7. Teach native xprompt argument completion to detect a catalog entry carrying
   `ref_kind` and source candidates from the artifact payload inventory. Preserve the
   xprompt syntax's replacement/insertion behavior while making candidate identity
   identical to `@<kind>:` completion. Route the Rust LSP through this API so both
   native entry points exercise the shared contract.

8. Update crate dependencies and lockfile only as required for the matcher. Do not edit
   release-plz-owned version fields. Add changelog/release-facing notes if the
   repository convention requires them, using breaking-change metadata at commit time
   because artifact context consumers must move to the new schema atomically.

## Tests and verification

- Add content-layout unit coverage for schema bump, ordered ref sources, physical paths,
  contextual naming, reserved namespace, and both directions of placement rejection
  while rerunning existing skill/memory layout tests.
- Add catalog/bridge/editor wire tests for canonical inputs of all six built-ins and
  document kinds, `kind: ref`, `ref_kind`, source provenance, backward-compatible
  deserialization of ordinary entries, and Rust/Python JSON parity.
- Add matcher table tests for root and nested Markdown under `**/*.md`, custom
  positives, vetoing negatives, negative-only filters, explicit empty filters, case
  sensitivity, zero-directory globstars, malformed patterns, and unsafe paths.
- Add artifact tests for allowed/filtered exact paths, filtered drift candidates,
  ambiguous allowed drift, missing paths, duplicate roots with different policies,
  canonicalization, stable diagnostics, context schema rejection, and absence of
  absolute-path leakage.
- Add inventory/binding/completion tests proving the same sorted allowed payloads for
  direct `@research:` completion, batch filtering, and `#ref/research` colon,
  positional, and named-argument completion, including scan truncation behavior.
- Run `cargo fmt --all -- --check`,
  `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`
  in `sase-core`. Re-run the targeted layout, catalog, artifact-reference, PyO3, and LSP
  tests after any final cleanup. Confirm the worktree contains no release-owned version
  edits and record release readiness for the epic's integration agent.

## Completion and handoff

Summarize the wire versions, exported APIs, and exact filter behavior for downstream
Python phases. Record any genuinely out-of-scope discovery as a `PROPOSED FOLLOW-UP:`
note on `sase-ho.1`, not as a new bead. Close only `sase-ho.1` with the concrete Rust
checks and parity behavior verified; leave parent epic `sase-ho` open.
