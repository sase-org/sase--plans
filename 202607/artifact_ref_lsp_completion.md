---
tier: tale
title: Artifact-reference completion and diagnostics in the xprompt LSP
goal: Editors complete and diagnose the same known kind-tagged artifact references as the SASE prompt surfaces.
bead: sase-av.7
create_time: 2026-07-29 13:58:03
status: done
---

- **PROMPT:** [202607/prompts/artifact_ref_lsp_completion.md](prompts/artifact_ref_lsp_completion.md)
- **PARENT:**
  [202607/artifact_refs_and_prompt_bar.md](https://github.com/sase-org/sase--plans/blob/main/202607/artifact_refs_and_prompt_bar.md)
- **BEAD:** [sase-av.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-av/sase-av.7.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-av.7--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-av.7.md#member-code)
  - [bbugyi200.athena.sase-av.7--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-av.7.md#member-plan)
- **COMMITS:**
  - [3f6e4ea](https://github.com/sase-org/sase/commit/3f6e4ea81a0ea13d5f0427358df02aa0c5cdde0a) — feat(editor):
    materialize artifact reference catalog for LSP

# Artifact-reference completion and diagnostics in the xprompt LSP

## Goal

Complete bead `sase-av.7` by making `sase lsp` recognize the same kind-tagged artifact references as the Python prompt
surfaces, offer catalog-backed kind and local-payload completion, and report malformed or unresolved known references
without treating unknown `@kind:` prose as an error.

The work is one cohesive implementation slice across the primary `sase` checkout and its linked `sase-core` checkout. It
does not create phases, change the artifact-reference grammar, add semantic tokens, add network-backed completion, or
modify the separately distributed Neovim integration.

## Existing contracts to preserve

- Rust `sase_core::artifact_ref` owns parsing, scanning, rendering, and local resolution. The LSP must consume those
  APIs rather than create a second reference grammar.
- Builtin kinds are `commit`, `chat`, `bug`, and `file`; configured document sidecar roles are dynamic kinds. Unknown
  kinds remain ordinary prose.
- `src/sase/artifact_refs.py::artifact_ref_context` is the authoritative Python builder for document roots, chats,
  artifact index, repositories, and project identities.
- Generic `@path` completion remains `FilePath`; artifact classification only takes over kind-prefixed input and must
  run before the generic colon-delimited tokenizer.
- Existing xprompt diagnostics and editor APIs remain usable without an artifact catalog. Artifact diagnostics skip the
  literal/fenced ranges returned by the shared prompt literal-zone scanner.
- `commit` and `bug` payloads are shape-checked but are not enumerated or resolved through git, issue trackers, or the
  network by an LSP request.
- The xprompt LSP binary is installed from a local `sase-core` build; it is not shipped inside the `sase-core-rs` wheel.

## Implementation

### 1. Materialize an authoritative launcher catalog in `sase`

Extend `src/sase/artifact_refs.py` with a versioned, JSON-serializable LSP catalog projection. Enumerate enabled,
non-system projects that have a usable primary workspace, normalize each workspace through the existing plan-resolution
workspace helper, and call `artifact_ref_context` for every project. Store:

- a schema version and best-effort default project derived from the launch workspace;
- each project's display name, storage key, and aliases;
- that project's `ArtifactRefContext.to_wire()` data, including ordered document-role roots;
- shared chats and artifact-index paths and the context's repository/project namespaces without inventing a second
  discovery path.

Failures for one project should omit only that project so a stale or partially configured project cannot prevent editor
startup. Keep ordering deterministic and deduplicate project identities and roots through the facade's existing
behavior.

Extend `src/sase/integrations/xprompt_lsp.py` with `SASE_XPROMPT_ARTIFACT_REF_CATALOG`, a default
`~/.sase/xprompt_lsp/artifact_ref_catalog.json` path, and best-effort materialization alongside the VCS-project and
model catalogs. Export the path even when materialization fails so a later refresh can recover. The existing
`sase.xpromptLsp.refreshCatalog` flow will observe rewrites because the Rust server reads this catalog on demand.

Add focused Python tests in `tests/test_artifact_refs.py` for the catalog projection and in
`tests/main/test_lsp_handler.py` for explicit/default paths, successful JSON materialization, and failure isolation.
Update the autouse LSP test fixture so all catalog writes remain under pytest temporary directories.

### 2. Add a pre-tokenizer artifact completion context in `sase-core`

In `crates/sase_core/src/editor/wire.rs`, add explicit artifact-reference completion context/trigger records that
distinguish kind completion (`@`, `@pla`) from payload completion (`@plans:202607/`). Carry byte spans for the candidate
and replacement query, the parsed kind when present, and enough state for frontends to build edits without reparsing.

In `crates/sase_core/src/editor/completion.rs`, add a detector before VCS, xprompt, directive, and generic-token
classification. Use the shared `artifact_ref::scan` result for colon-bearing candidates and a bounded left-context-aware
prefix path for incomplete kind input. Accept the cursor at every position inside the active reference, reject
whitespace/literal suffixes, and leave colon-free `@path` input to the existing `FilePath` classifier.

Add core completion builders that:

- filter the active project's builtin and dynamic document kinds after `@`, inserting the selected kind plus `:`;
- enumerate document files relative to the selected role's ordered roots, chats relative to the chats root, and artifact
  ids/paths from the local artifact index;
- deduplicate deterministic candidates, prefix-filter case-insensitively, bound recursive filesystem traversal and
  result counts, and expose useful details such as artifact kind/root or indexed path;
- return no payload candidates for `commit`, `bug`, or unknown kinds.

Export the new records and helpers through `editor/mod.rs` and the crate root. Update all `CompletionContext`
constructors mechanically for the new optional trigger field without changing existing serialized behavior.

Unit-test classification at kind and payload cursor positions, dynamic role candidate construction using the role name
`designs`, bounded/deduplicated local payload enumeration, silence for non-enumerated kinds, UTF-16-safe replacement
ranges, and the regression that plain `@path` still classifies as `FilePath`.

### 3. Load project context and serve completion in `sase_xprompt_lsp`

In `crates/sase_xprompt_lsp/src/server.rs`, define a tolerant schema-v1 loader for the launcher catalog and capture its
env path in `ServerConfig`. Malformed, missing, or unsupported catalogs degrade to no artifact assistance.

Resolve the active project for each request by matching the document's leading VCS workspace reference against the
already-loaded VCS project/ChangeSpec catalog; map ChangeSpecs back to their owning project. Fall back to the artifact
catalog's default project and then the initialized workspace project. Match display names, keys, and aliases while
preserving user-facing labels.

Pass the selected `ArtifactRefContextWire` to the new core classifier/builders, route the artifact context before normal
completion dispatch, and convert the result through the existing completion response machinery. Re-read the catalog per
request so the existing refresh command and external rewrites are effective.

Add integration tests using temporary `designs`, chat, and artifact-index fixtures. Assert kind completion, per-kind
payload completion and replacement ranges, active-project selection from a leading workspace ref, fallback-project
selection, empty `commit`/`bug` payload menus, and graceful behavior for absent or malformed catalogs.

### 4. Add known-kind artifact diagnostics

Add a dedicated core diagnostic entry point that accepts an `ArtifactRefContextWire`, scans with `artifact_ref::scan`,
and filters candidates whose byte spans intersect `prompt_literal_zone_ranges`.

For each non-literal candidate:

- skip it when its kind is neither builtin nor one of the context's document roles;
- emit `malformed_artifact_ref` when the known-kind candidate does not parse, including illegal fragments and invalid
  kind-specific payloads;
- for parsed document, chat, and file references, use only local artifact-reference resolution and emit
  `unresolved_artifact_ref` when no target exists;
- perform only shape validation for parsed `commit` and `bug` references.

Use the full candidate range, existing diagnostic severity/range conversion, and clear messages containing the rendered
reference or parse failure. Preserve the existing `editor_analyze_document` API and let the server append artifact
diagnostics only when it has an active catalog context.

Test malformed fragments, missing documents, successful document/chat/file resolution, unknown-kind silence,
fenced/backtick silence, and commit/bug shape behavior in core unit tests and server integration tests.

### 5. Document and verify the editor contract

Update `docs/editor.md` to add artifact-reference kind/payload completion and known-kind diagnostics to the LSP feature
table, describe catalog freshness and the no-network behavior, and explicitly state that development/editor validation
installs the binary from the local `sase-core` build rather than from the `sase-core-rs` wheel.

Run verification in this order:

1. In the linked `sase-core` checkout, run focused `sase_core` editor tests and `sase_xprompt_lsp` tests while
   iterating.
2. Run `cargo fmt --all -- --check`, `cargo test --workspace`, and
   `cargo clippy --workspace --all-targets -- -D warnings`.
3. In the primary `sase` checkout, run `just install` before repository checks, then focused Python tests for artifact
   refs and LSP launch materialization.
4. Run `just rust-lsp-install` to install the locally built server and exercise `sase lsp --version` plus a bounded
   stdio/editor smoke test covering one completion and one diagnostic request.
5. Run the mandatory primary-repository `just check`, fix every failure, and rerun focused tests and `just check` after
   any subsequent edit.
6. Confirm both git worktrees contain only the intended bead changes, then close `sase-av.7` with a note listing the
   successful Rust, Python, local-LSP, and mandatory `just check` verification. Do not close `sase-av`.

## Risk controls

- Catalog generation and loading are best-effort, version-gated, deterministic, and backward compatible; editor startup
  and ordinary completion do not depend on catalog availability.
- Filesystem enumeration is local, bounded, deduplicated, and only performed for completion kinds that require it.
  Diagnostics use the shared resolver and never invoke subprocesses or network providers.
- Literal-zone filtering and unknown-kind silence are explicit tests because false-positive diagnostics would make
  ordinary mentions such as `@user:handle` unusable.
- No crate versions or path-dependency pins are edited; release-plz owns `sase-core` versions.
