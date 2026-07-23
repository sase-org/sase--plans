---
tier: tale
title: Define capitalized snippet composition in sase-core
goal: Provide one shared, metadata-aware capitalized snippet composition contract
  to Rust, Python, and the native LSP fallback.
bead: sase-8u.1
parent: sase/repos/plans/202607/capitalized_snippet_aliases.md
create_time: 2026-07-23 08:15:37
status: done
---

- **PROMPT:** [202607/prompts/capitalized_snippet_core.md](prompts/capitalized_snippet_core.md)

# Define capitalized snippet composition in sase-core

## Goal

Complete bead `sase-8u.1` by making capitalized snippet aliases a deterministic shared Rust domain contract, using that
contract in the native snippet catalog and LSP fallback, and exposing the same result to Python through PyO3. This tale
implements only phase 1 of the parent epic; Python host integration, documentation, compatibility-floor changes, and
release version changes remain for later phases.

## Behavioral contract

- Accept the effective explicit trigger-to-template catalog only after callers have applied their existing source
  precedence.
- Resolve references in that explicit catalog with the existing literal missing/cycle behavior, positional-argument
  substitution, escape handling, and tabstop renumbering.
- For each explicit trigger, uppercase only its first Unicode scalar and preserve the remaining trigger bytes. Generate
  an alias only when the trigger changes and the generated name is not already explicit.
- Build each alias template by uppercasing only the first Unicode scalar of the source's first-pass resolved template
  and preserving the remainder. Empty or non-letter-leading templates remain textually unchanged.
- Resolve the combined explicit-plus-generated catalog a second time so either spelling can be referenced. Generate
  aliases only from the original explicit entries, never recursively.
- Return both the final ordered trigger-to-template map and an ordered `generated trigger -> explicit source trigger`
  provenance map. Explicit entries always win collisions.

## Implementation

1. In `crates/sase_core/src/xprompt_catalog.rs`, introduce a public result type for composed templates and alias
   provenance, and a public `compose_snippet_catalog` function over ordered maps. Refactor the existing private
   reference resolver for reuse without changing its established semantics. Use deterministic `BTreeMap` iteration and
   set-if-absent alias insertion.
2. Update `load_editor_snippet_catalog` after the current xprompt-first, user-config-overwrite merge. Compose the raw
   templates through the new contract, clone the complete source entry metadata for provenance-identified aliases,
   replace every entry's template with its final composed template, and emit the existing stable trigger order. Ensure
   total statistics include generated aliases.
3. Re-export the composer and result type from `crates/sase_core/src/lib.rs`.
4. In `crates/sase_core_py/src/lib.rs`, add `compose_snippet_catalog(templates: dict[str, str]) -> dict` to the
   documented module inventory and module registration. Convert the input into the ordered Rust map and return a plain
   dictionary with `templates` and `alias_provenance` mappings so Python callers need no Rust-specific classes. Add
   conversion coverage that invokes the binding through a Python module and verifies the public wire shape.

## Tests

- Extend `xprompt_catalog.rs` unit tests for simple aliases and remainder-case preservation; Unicode first-scalar
  behavior; empty and non-letter-leading templates; already-capitalized, digit-leading, and underscore-leading triggers;
  and explicit capitalized collisions.
- Cover aliases derived from reference-composed templates and second-pass references to generated aliases. Retain or
  extend the golden resolver vectors to prove positional arguments, escaping, tabstop renumbering, missing references,
  and cycles remain unchanged.
- Extend native catalog tests to prove authored user snippets still override xprompt snippets, explicit capitalized
  entries are not replaced, generated entries inherit source kind, xprompt name, description, and display path, and
  statistics count aliases.
- Extend the `sase_xprompt_lsp` catalog-cache fallback test to require both the lowercase native entry and its
  capitalized alias with the correctly capitalized expansion.
- Run focused package tests while iterating, then run formatting, workspace Clippy with warnings denied, and the full
  workspace test suite. Fix and rerun until all checks pass.

## Constraints and handoff

- Do not edit Cargo workspace versions, crate versions, or local path dependency pins; release-plz owns the release
  boundary.
- Do not change Python host catalogs, SASE docs, or authored snippet persistence in this bead.
- Do not close the parent epic or create additional beads.
- After implementation and successful validation, close only `sase-8u.1`.

## Acceptance criteria

- `foo: "foo bar baz"` composes to `foo: "foo bar baz"` and `Foo: "Foo bar baz"` with provenance `Foo -> foo`.
- An explicit `Foo` retains its own template and metadata.
- References to both explicit and generated triggers compose with the previous resolver's tabstop, positional-argument,
  missing-reference, and cycle behavior.
- The native editor catalog and LSP fallback include aliases with inherited metadata and correct counts.
- The registered PyO3 binding returns the documented plain-dict shape.
- All focused tests and full Rust workspace checks pass without version edits.
