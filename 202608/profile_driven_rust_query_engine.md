---
tier: tale
title: Parameterize the Rust query engine by compiled profile
goal:
  The Rust parser, persistent corpus, evaluator, and Python binding consume the shared
  compiled query profile and generic precomputed artifact rows while every existing
  Patch query entry point remains compatible.
size: medium
proposed_by: bbugyi200.athena.sase-m6.6.1.2
bead: sase-m6.6.1.2
create_time: 2026-08-15 07:10:19
status: wip
---

- **PARENT:** [202608/unified_artifacts_query_1.md](unified_artifacts_query_1.md)
- **BEAD:**
  [sase-m6.6.1.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.2.md)

# Plan: Parameterize the Rust query engine by compiled profile

## Goal

Complete phase bead `sase-m6.6.1.2` by turning `sase-core`'s Patch-specific query parser
and persistent corpus into a profile-driven engine over generic, precomputed artifact
rows. Expose the new calls through `sase_core_rs` while keeping all existing Patch APIs
and wire shapes working unchanged. This phase stops at the binding boundary; the SASE
host adapters and Python reference evaluator remain owned by their sibling phases.

The accepted input profile is the deterministic `CompiledQueryProfile.to_wire()` shape
already produced by `sase.ace.query_profile`. It is a per-call argument, not a persisted
provider/spec wire and does not change the provider schema version.

## Implementation

1. Add Rust query-domain records for the compiled profile and generic rows.
   - Deserialize fields, sigil mappings, selected closed predicates, macro mappings,
     boolean mode, hints, and digest from the existing Python wire shape.
   - Represent each corpus row with precomputed field values, searchable values/text,
     and selected host-predicate facts so evaluation does not call providers or rebuild
     pane data per query.
   - Validate defensive invariants at the Rust boundary (unique/filterable fields,
     declared sigil and macro targets, supported closed predicate names, compatible
     repeatability/negation/value kinds) and return structured query errors rather than
     panicking or silently accepting unknown semantics.

2. Parameterize tokenization, parsing, compilation, and canonicalization.
   - Resolve property keys from the supplied profile instead of `VALID_PROPERTY_KEYS`;
     resolve sigils and macros from profile tables; enable only the declared
     zero-argument predicates and `any_special` expansion.
   - Preserve the current Boolean grammar, precedence, implicit `AND`, quoting,
     case-sensitive literals, error spans/messages, and canonical output when the
     built-in Patch compatibility profile is used.
   - Implement `boolean=false` as the flat token grammar: whitespace conjunction,
     profile-gated leading-token negation, comma expansion only for repeatable fields,
     rejection of Boolean operators/parentheses/case-sensitive literals, and
     profile-driven enum/bool/int validation. Preserve source ordering and quoting in
     the profile-aware canonical form while compiling repeated positive values for one
     field as an any-match constraint and exclusions as negated constraints.
   - Keep `tokenize_query`, `parse_query`, `canonicalize_query`, and `compile_query` as
     Patch-compatible wrappers so existing Rust and Python consumers see the same
     tokens, AST, canonical strings, and failures.

3. Generalize the persistent corpus and evaluator.
   - Make the primary corpus own generic query rows plus their precomputed lowercase
     searchable text and field indexes, and evaluate field matches against the row
     values instead of a `ChangeSpecWire` switch statement.
   - Evaluate free text exclusively from values selected as searchable by the profile;
     evaluate the three closed host predicates from row facts; make exact/string-list,
     enum, boolean, integer, and date-shaped values deterministic and case-insensitive
     where legacy dialects require it.
   - Build a Patch adapter that precomputes status, project, ancestry, exact name,
     sibling family, origin, searchable text, and operational predicate facts from
     `ChangeSpecWire`. Retain `QueryCorpus::new`, `evaluate_query_many`,
     `evaluate_query_many_in_corpus`, `evaluate_query_one`, and public Patch matcher/
     searchable helpers as compatibility entry points backed by the generic engine.

4. Extend the PyO3 binding without breaking callers.
   - Add profile-aware tokenize/parse/canonicalize/compile functions and a generic
     corpus compiler accepting the compiled profile dict and generic precomputed row
     dicts. Ensure profile and row conversion occurs while holding the GIL, while corpus
     indexing and evaluation remain outside it.
   - Associate compiled programs and corpora with the profile digest (or equivalent
     semantic identity) and reject mismatched handles before evaluation.
   - Preserve existing `compile_corpus(specs)`, `compile_query(query)`,
     `evaluate_many(program, corpus)`, and `evaluate_query_many(query, specs)` behavior
     by routing them through the built-in Patch profile and Patch-row adapter.

5. Prove compatibility and profile-driven behavior.
   - Expand Rust tokenizer/parser/canonicalization unit tests for arbitrary fields,
     profile-selected sigils/macros/predicates, disabled syntax, flat negation and comma
     rules, typed literal validation, invalid profiles, exact source positions, and
     digest mismatches.
   - Expand corpus/evaluator tests with non-Patch generic rows covering searchable-field
     selection, repeated field values, every closed predicate, positive/negative flat
     constraints, Boolean precedence, and reuse across repeated evaluations.
   - Extend `query_evaluator_parity.rs` to run the legacy Patch golden matrix through
     both compatibility and explicit-profile paths and assert identical results and
     canonical forms.
   - Add PyO3 tests for round-tripping the real `CompiledQueryProfile.to_wire()`-shaped
     dictionaries, generic corpus evaluation, invalid inputs, profile mismatch, and
     unchanged legacy handle calls.

## Verification

From `sase-core`, run focused formatting, lint, query-library tests,
`query_evaluator_parity`, and binding tests during implementation, then run the required
whole-repository `just check` gate. Confirm both repository worktrees contain only the
intended changes, record any out-of-scope discovery as a `PROPOSED FOLLOW-UP:` note on
`sase-m6.6.1.2`, and close that phase bead with a note listing the successful Rust and
binding verification. Do not close the parent epic.
