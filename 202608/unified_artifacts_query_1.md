---
tier: epic
title: One profile-driven query engine for every Artifacts pane
goal:
  Replace the Artifacts tab's Patch boolean language and four pane-local flat token
  languages with one profile-parameterized engine whose Rust parser, Rust batch
  evaluator, and Python reference evaluator agree; migrate every pane without changing
  its current semantics until explicitly enabled; move Patch onto the inline filter bar;
  and persist saved views, history, and stable selection targets independently per pane.
phases:
  - id: profile
    title: Define and compile the shared query profile
    depends_on: []
    size: medium
    description:
      "profile: define the Python-authored ArtifactQuerySchema and deterministic
      compiled profile passed into Rust, including fields and types, searchable fields,
      negation, closed host sigils, zero-argument predicates, macros, boolean mode,
      canonicalization, validation, and a stable digest; preserve every existing
      dialect's canonical form and prove profiles for Patches, Stitches, Beads, Plans,
      Files, and the synthetic provider."
  - id: rust_engine
    title: Parameterize the Rust parser, corpus, and Python binding
    depends_on:
      - profile
    size: large
    description:
      "rust_engine: in sase-core, replace the hard-coded property allowlist and
      Patch-only QueryCorpus assumptions with the compiled profile and generic
      precomputed rows, parameterize tokenization, parsing, sigils, predicates, macros,
      searchable text and boolean mode, expose the profile-driven calls through the
      Python binding, preserve compatibility entry points, and extend Rust parser,
      corpus, and evaluator parity tests before updating the host adapter."
  - id: python_reference
    title: Generalize the Python reference evaluator
    depends_on:
      - profile
    size: medium
    description:
      "python_reference: make query_facade's Python-owned per-row evaluator consume the
      same compiled profile and typed/coerced Artifact fields as Rust, implement the
      shared field, sigil, predicate, macro, searchable-text and boolean semantics, and
      expand cross-language fixtures so the Python reference evaluator and Rust batch
      evaluator return identical matches and errors for every pane profile."
  - id: persistence
    title: Namespace durable query state by pane
    depends_on:
      - profile
    size: medium
    description:
      "persistence: replace the global saved-query, history, and selection stores with
      pane-keyed records containing source text, canonical text, profile digest, and
      ArtifactEntryTarget tokens; add safe read-time migration and write-then-read
      validation while retaining legacy files until success, surface stale-profile saved
      views as editable errors, and route slots, pickers, history, help, startup, and
      selection restore through the active pane without switching tabs."
  - id: flat_panes
    title: Migrate Stitches, Beads, Plans, Files, and provider panes
    depends_on:
      - rust_engine
      - python_reference
      - persistence
    size: large
    description:
      "flat_panes: configure the shared FilterBar and query engine from each
      ArtifactsPaneContract at boolean=false, migrate Stitches, Beads, Plans, Files and
      arbitrary ref providers while preserving their current tokens and canonical forms,
      derive free-text from searchable fields, generate completion/highlighting/facets
      from declared properties, add the shared host predicates, fix Files negation, and
      cache profiles and results by pane, snapshot generation, profile digest, and
      canonical query off the Textual event loop."
  - id: patch_bar
    title: Cut Patch over to the shared inline filter bar
    depends_on:
      - rust_engine
      - python_reference
      - persistence
      - flat_panes
    size: large
    description:
      "patch_bar: configure the Patch contract with boolean=true and migrate its boolean
      grammar, sigils, status macros and Rust corpus to the shared engine; replace
      QueryEditModal with a persistent inline FilterBar providing completion, live match
      count, coverage and Escape rollback; move #N saves into submit handling, make f
      open the bar, make p rewrite only project scope, and verify slots, history and
      stable selection restoration never affect a hidden pane."
  - id: conformance
    title: Prove parity, migration safety, visuals, and responsiveness
    depends_on:
      - patch_bar
    size: medium
    description:
      "conformance: extend the Artifacts contract harness and golden corpus across all
      five built-in panes plus the synthetic provider, cover invalid profiles and
      queries, legacy persistence migration, changed-profile saved views, selection
      restoration, completion and pane isolation, remove obsolete pane-local parser and
      modal paths only where compatibility permits, review intentional PNG changes, run
      full Python and Rust verification, and demonstrate cached per-keystroke navigation
      p95 below 16 ms for every migrated pane."
proposed_by: bbugyi200.athena.sase-m6.6
parent_bead: sase-m6.6
bead_id: sase-m6.6.1
create_time: 2026-09-09 19:52:01
status: wip
---

- **PROMPT:**
  [prompts/202608/unified_artifacts_query_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/unified_artifacts_query_1.md)
- **BEAD:**
  [sase-m6.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.md)

# Plan: One profile-driven Artifacts query engine

## Scope and invariants

This child epic implements the `query` phase of the parent Artifacts-contract design. It
covers the five built-in Artifacts panes (Patches, Stitches, Beads, Plans, Files),
arbitrary `ref:` provider panes, the Rust syntax and batch paths, the Python reference
evaluator, Patch's inline filter UI, and the three query-state files. The Agents tab is
an informative predecessor but remains out of scope.

The engine is a strict superset of all existing pane dialects. A pane starts with
`boolean=false`, which must reproduce its current flat-token behavior and canonical form
byte-for-byte; Patch uses `boolean=true` only after the common engine and all flat panes
pass parity. Providers declare fields and facts, never executable matchers or new
punctuation. Sigils and zero-argument predicate implementations come from a closed,
host-validated vocabulary. The profile is an argument to parsing and evaluation rather
than a persisted versioned wire, and its deterministic digest identifies the semantics
used by caches and durable saved views.

The implementation must keep all three evaluators aligned:

1. Rust tokenization/parsing and compilation.
2. Rust `QueryCorpus` batch evaluation used by the Patch hot path and generalized for
   other Artifact rows.
3. The Python-owned reference evaluator exposed through `query_facade`.

The existing query goldens and `query_evaluator_parity.rs` are the oracle. Add cases for
precedence, implicit AND, quoting, case-sensitive literals, negation, every legacy
sigil/macro, declared properties and types, repeated values, invalid queries with spans
and messages, and selection results. Do not widen `ArtifactEntryWire.properties` in this
epic: coerce declared `datetime` and `string_list` values on read in Python, with
malformed values degrading per entry rather than removing a pane.

## Architecture

`ArtifactQuerySchema` is authored from each `ArtifactsPaneContract` at snapshot/pane
construction. It describes field names and declared types, which fields contribute to
free-text search, repeatability and negation, the selected closed sigils and predicates,
legacy macros, and boolean mode. Compilation validates that schema against host
registries, produces a deterministic profile and digest, and precomputes completion and
highlighting metadata. No provider resolution, filesystem access, globbing, Markdown
parsing, Git work, or schema compilation may happen on a keystroke.

The Rust core accepts the compiled profile on every parse/compile/corpus boundary. Its
corpus owns generic precomputed query rows instead of reconstructing Patch-specific
search data for every expression. Compatibility wrappers preserve non-TUI consumers
while they are migrated. The Python facade owns wire conversion and uses the identical
profile for its reference matcher. Result caches are keyed by
`(pane_id, generation, profile_digest, canonical_query)` and invalidated when the
snapshot generation or profile changes.

`saved_queries.json`, `query_history.json`, and `query_selections.json` become
pane-keyed. A saved record keeps source text, canonical text, and profile digest. A
selection keeps the canonical `ArtifactEntryTarget` token delivered by the identity
phase. Readers accept legacy Patch-only shapes; migration does not delete or overwrite
the legacy source until the new form has been atomically written and read back. A digest
mismatch is visible and editable, never silently reinterpreted. Missing or corrupt state
falls back to an empty state for that pane without damaging other panes.

## Migration order

After profile and evaluator parity exists, migrate Stitches, Beads, Plans, Files, and
provider panes at `boolean=false`. Each adapter must pass its existing parser tests and
the shared conformance suite before the next layer depends on it. Declared provider
properties immediately drive field matching, completion, highlighting, faceting, and
free-text when marked searchable; the synthetic provider proves this needs no provider
Python release. Files must gain the negation behavior already supported by the other
flat panes.

Patch moves last. Its contract enables boolean syntax and maps the existing sigils,
zero-argument states, and `%` status aliases into the shared profile. The inline bar
replaces the modal without losing canonical queries, live Rust corpus filtering, slot
saves, history, or selection memory. `f` opens the same bar on every pane. Patch's `p`
action edits only the committed `project:` token and removes it for All projects. Slot
loading, `^`/history navigation, and selection restore always act on the active pane and
never hard-switch to Patches.

## Performance and failure behavior

Compile schemas, typed values, searchable text, completions, and Rust corpus indexes
once per snapshot outside the Textual event loop. Filter keystrokes use only immutable
compiled data and bounded cached evaluation. Keep timer and `call_later` callbacks thin;
launch slow work through the existing pump-free task path and re-check active pane and
selection after every await. Preserve immediate highlighting while debouncing only
derived detail work. An invalid provider profile or malformed entry produces a visible
pane/entry diagnostic and cannot remove or poison another pane.

## Verification

Every implementation phase starts with `just install`. Rust phases run the relevant
`cargo test` suites in the opened `sase-core` repository and host binding/facade tests.
Every phase that changes files in the SASE repository runs `just check`. Because these
changes touch broad Artifacts TUI and backend surfaces, run `just check-full` only via
`/sase_monitor` before landing the child epic, with a `--next` action that handles the
result. Run the dedicated visual suite for the Patch modal-to-inline change, inspect
actual/expected/diff artifacts, and accept only intentional goldens.

Extend the existing contract harness so every pane and the synthetic provider executes
the same profile compile, parse, canonicalize, evaluate, persistence, completion, and
selection-isolation cases. Measure each migrated pane with `SASE_TUI_PERF=1`; navigation
p95 must remain below 16 ms, and traces must show no resolver, disk, Git, Markdown,
glob, or profile compilation work in typing/navigation paths. The child epic is complete
only when the Rust batch results, Python reference results, and query goldens agree for
all profiles and Patch no longer uses `QueryEditModal`.

## Explicitly deferred

- Widening `ArtifactEntryWire.properties` to a typed Rust value wire.
- Converting the non-Artifact Agents tab from `ace/agent_query`.
- Removing legacy persistence readers before a measured deprecation window.
- The remaining cross-pane keymap unification owned by the parent's `keymap` phase.
