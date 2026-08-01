---
tier: tale
title: Add kind-tagged artifact references to the Rust core
goal:
  The Rust core parses, renders, canonicalizes, resolves, and scans kind-tagged artifact references through stable
  pure-Rust APIs and PyO3 bindings without changing existing plan-reference behavior.
bead: sase-av.1
create_time: 2026-07-29 12:54:11
status: done
---

- **PROMPT:** [prompts/202607/artifact_ref_core.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/artifact_ref_core.md)
- **PARENT:** [202607/artifact_refs_and_prompt_bar.md](https://github.com/sase-org/sase--plans/blob/main/202607/artifact_refs_and_prompt_bar.md)
- **BEAD:** [sase-av.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-av/sase-av.1.md)
- **AGENTS:**
  - bbugyi200.athena.sase-av.1--code
- **COMMITS:**
  - [6c2adc4](https://github.com/sase-org/sase-core/commit/6c2adc420a5ee24aecfe5fae305e2c869ab7b627) — feat: add core artifact reference APIs

# Plan: Add kind-tagged artifact references to the Rust core

## Goal

Implement bead `sase-av.1` in the linked `sase-core` repository: add a pure-Rust artifact-reference grammar,
canonicalization and resolution API, prompt scanner, serde wire records, and six PyO3 bindings. Preserve the existing
`plan/refs.rs` public API, wire schema, error strings, legacy-path behavior, and tests byte-for-byte while moving only
the reusable path and ordered-root resolution mechanics behind shared crate-private helpers.

## Constraints and contracts

- Builtin kinds are exactly `commit`, `chat`, `bug`, and `file`; any other syntactically valid kind is represented as a
  caller-classified document-role kind rather than hardcoding role names.
- Canonical references have the form `<kind>:<payload>[#<fragment>]`; prompt candidates add a leading `@`.
- Kind syntax is `[a-z][a-z0-9_-]*`. Document and chat payloads are safe POSIX relative paths. Commit payloads are
  `<repo>@<sha>` with a path-safe repository name and 7–40 lowercase hex characters, rendered with a caller-resolved
  full 40-character SHA. Bug payloads are `<project>#<positive-decimal-number>`. File payloads are
  `(explicit|default):<hex24>`.
- Fragment values are typed as line ranges, pages, or timestamps. They apply only to document, chat, and file
  references; commit and bug references reject them.
- Unknown syntactically valid kinds remain parseable document-role candidates. Resolution, supplied with configured role
  roots, returns `unknown_kind` when the role is absent.
- Resolution is local and deterministic: it may inspect supplied roots and the artifact index, but never shells out or
  accesses the network. Repository/project resolution validates caller-supplied aliases and echoes locators.
- `plan/refs.rs` remains behaviorally frozen, including its treatment of every non-`plans:` value as a legacy path.
- Release versions and local dependency pins remain owned by release-plz and are not edited manually.

## Implementation

1. In `crates/sase_core/src/artifact_ref/`, define the domain model and wire contract:
   - typed kind/payload/fragment enums and structs with serde-friendly parse and resolution projections;
   - caller-supplied context records for ordered document-role roots, chats root, artifact-index path, known repository
     names with canonical full SHAs, and project display-name/key aliases;
   - separate parse and resolution wire-schema constants following the existing plan-reference convention;
   - structured resolution results carrying status, canonical rendered reference or locator, resolved path when
     applicable, and ordered candidates.

2. Extract crate-private helpers from `plan/refs.rs` for safe relative payload validation/conversion and ordered-root
   exact/drift probing. Generalize drift probing to search ordered root children by basename, which covers both
   `YYYYMM/<name>` plan movement and chat shard movement. Adapt the helpers so plan-reference callers retain their exact
   existing validation messages, candidate ordering, first-root-wins behavior, legacy candidate handling, and
   `exact|drifted|ambiguous|missing` outcomes.

3. Implement artifact-reference parse/render behavior:
   - split kind before applying the kind-specific `#` rules so bug issue numbers are payload while supported path-like
     fragments remain anchors;
   - validate every payload grammar and typed fragment (`L<n>`, `L<n>-L<m>`, `page=<n>`, `t=<seconds>`);
   - preserve dynamic document-role names in the typed representation;
   - make render deterministic, including POSIX paths, canonical fragment spelling, and full commit SHAs supplied by
     canonicalization/resolution context.

4. Implement canonicalization and resolution:
   - canonicalize absolute paths against ordered `(kind, root)` document roots, the chats root, and artifact-index rows,
     returning `None` outside all supplied roots;
   - resolve document and chat references through exact ordered probing followed by unique basename drift detection;
   - read schema-version-1 JSONL artifact-index envelopes, locate matching file ids, and return their stored paths;
   - return `unknown_kind`, `unknown_repo`, or `unknown_project` without filesystem/network side effects, accepting
     project keys as aliases while emitting user-facing project names.

5. Implement the context-free prompt scanner. Enforce the documented left-context rule, trim trailing `.,;:!?)`, retain
   malformed known-shape candidates with `well_formed = false`, and return UTF-8 byte spans for the full candidate plus
   sigil, kind, separator, payload, and optional fragment. Keep candidates inside backticks/fences visible to the
   scanner so callers can filter them with literal-zone ranges.

6. Re-export the module and public domain/wire API from `sase_core`. Add the PyO3 functions `artifact_ref_parse`,
   `artifact_ref_render`, `artifact_ref_canonicalize`, `artifact_ref_resolve`, `artifact_ref_scan_prompt`, and
   `artifact_ref_wire_schema_version`, using JSON-compatible dictionaries/lists for typed inputs and outputs and the
   existing ValueError/serialization conventions. Register all six names in module initialization and cover their
   Python-visible shapes.

7. Add focused Rust and binding tests:
   - round-trip every kind and fragment form; reject unsafe paths, invalid kinds, malformed ids/numbers/SHAs, illegal
     fragments, and fragments on commit/bug;
   - verify ordered-root precedence, exact/drifted/ambiguous/missing results, artifact-index lookup, canonicalization,
     unknown kind/repo/project statuses, project-key alias rendering, and absence of git/network work;
   - verify scanner byte spans at line start and after allowed left context, inside backticks, with Unicode before a
     reference, trailing punctuation, multiple references, and extra `@`/`:` characters;
   - keep all existing plan-reference tests unchanged and add regression assertions for their public error strings and
     legacy parsing contract;
   - exercise all six registered PyO3 functions and their schema-versioned JSON shapes.

## Validation

Run from the linked `sase-core` repository:

1. `cargo fmt --all -- --check`
2. `cargo clippy --workspace --all-targets -- -D warnings`
3. `cargo test --workspace`

Re-run the targeted artifact-reference, plan-reference, and PyO3 binding tests after any correction, then repeat all
three repository-wide gates. Verify `git diff --check`, inspect the final diff to confirm no release version or
unrelated files changed, and record the passing commands when closing `sase-av.1`.
