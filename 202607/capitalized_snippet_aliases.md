---
tier: epic
title: Capitalized aliases for every SASE snippet
goal: Make every effective SASE snippet available through a generated initial-capital
  trigger with a correspondingly capitalized expansion, consistently across ACE, editor
  helpers, and the native LSP fallback.
phases:
- id: core-composition
  title: Define capitalized snippet composition in sase-core
  depends_on: []
  size: medium
  description: Add the shared catalog-composition contract, native fallback integration,
    PyO3 binding, and core tests.
- id: host-integration
  title: Integrate capitalized aliases into SASE catalogs and live saves
  depends_on:
  - core-composition
  size: medium
  description: Consume the core contract from Python, preserve source metadata and
    raw cache state, and cover ACE and helper behavior.
- id: parity-verification
  title: Document and verify cross-surface parity
  depends_on:
  - host-integration
  size: small
  description: Finish user documentation and run the complete Rust and Python validation
    suites after the core release boundary is satisfied.
create_time: 2026-07-23 08:10:52
status: wip
bead_id: sase-8u
---

# Plan: Capitalized aliases for every SASE snippet

## Context

SASE currently builds the effective snippet catalog in three related paths:

- ACE merges xprompt-derived templates, merged `ace.snippets`, and session-local pending saves in
  `src/sase/ace/tui/prompt_catalog.py`, then resolves `#[trigger]` references.
- `sase editor helper-bridge snippet-catalog` performs a metadata-preserving version of that merge in
  `src/sase/integrations/_editor_helper_snippets.py`.
- The linked `sase-core` repository independently builds the native editor catalog in
  `crates/sase_core/src/xprompt_catalog.rs` so `sase lsp` can degrade gracefully when the Python helper is unavailable.

The alias rule is shared domain behavior, so it belongs in `sase-core` and must be exposed through a thin Python
binding. Applying it only in the prompt widget would make helper output, editor completion, and the native fallback
disagree. Applying it while individual sources are loaded would also disturb the existing first-xprompt/user-override
precedence and make generated aliases shadow explicit definitions.

## Behavioral contract

Compose aliases only after every explicit source for a catalog has been merged with today's precedence:

1. For each explicit effective trigger, uppercase only its first character and preserve the rest of the trigger
   byte-for-byte. A distinct generated trigger is created only when this changes the name. Thus `foo_bar` produces
   `Foo_bar`; already-capitalized, digit-leading, and underscore-leading triggers do not produce an extra entry.
2. Resolve the source template's existing snippet references, uppercase only the first Unicode scalar of the resulting
   template, and preserve every remaining character, tabstop, escape, line break, and letter case. An empty template or
   a template whose first character has no uppercase form remains textually unchanged, although its distinct trigger
   alias is still valid.
3. Generated aliases are runtime catalog entries only. Do not write them into `sase.yml`, xprompt frontmatter, or
   chezmoi source files.
4. Explicit definitions always beat generated aliases. If both `foo` and `Foo` were authored, each retains its
   explicitly merged template and metadata. User-config precedence over xprompt-derived snippets remains unchanged
   before this collision rule is applied.
5. Generated aliases inherit their source entry's source kind, xprompt name, description, and source display path.
   Catalog statistics count them because they are usable completion/expansion entries.
6. Both spellings participate in `#[trigger]` composition. Use a two-stage composition: resolve the explicit catalog,
   derive non-conflicting aliases from those resolved source templates, then resolve the combined catalog again so
   references to generated names such as `#[Foo]` work without changing tabstop renumbering or cycle handling.
7. Generate aliases from the explicit input set only; do not recursively alias generated entries.

The representative result is:

```text
foo -> "foo bar baz"
Foo -> "Foo bar baz"
```

## Phase 1: Define capitalized snippet composition in sase-core

In the linked `sase-core` repository:

1. Refactor the private reference-resolution path in `crates/sase_core/src/xprompt_catalog.rs` behind a public,
   deterministic snippet-composition function. Its result must return both the final trigger-to-template map and alias
   provenance (`generated trigger -> explicit source trigger`) so metadata-bearing callers never have to infer whether
   `Foo` was explicit or synthesized.
2. Implement the behavioral contract above with ordered maps and set-if-absent collision handling. Keep the existing
   resolver's literal behavior for missing/cyclic references and its tabstop splicing/renumbering semantics.
3. Make `load_editor_snippet_catalog` use the shared composer after its xprompt/user merge. Clone source metadata only
   for aliases identified by provenance, update all entry templates from the final composed map, and retain stable
   output ordering.
4. Re-export the pure contract from `crates/sase_core/src/lib.rs`. Add a `compose_snippet_catalog` PyO3 binding in
   `crates/sase_core_py/src/lib.rs`, including the module API inventory, registration, conversion tests, and a simple
   plain-dict wire shape for templates plus alias provenance.
5. Add focused Rust coverage for:
   - simple aliases and preservation of the source template's remaining case;
   - empty/non-letter-leading templates and non-lowercase-leading triggers;
   - explicit `Foo` collisions and user-config-over-xprompt precedence;
   - aliases based on reference-composed templates and references that target generated aliases;
   - unchanged positional arguments, tabstop renumbering, missing-reference, and cycle behavior;
   - metadata/stat counts in the native catalog and aliases in the LSP native-fallback test.
6. Follow `sase-core` release policy: do not manually edit Cargo workspace/crate versions or local version pins.
   Release-plz owns the version that makes the new binding publishable.

## Phase 2: Integrate capitalized aliases into SASE catalogs and live saves

After Phase 1's core contract is available:

1. Add a thin Python facade using `sase.core.rust.require_rust_binding("compose_snippet_catalog")`. Validate/normalize
   the returned templates and alias-provenance mappings at the boundary; do not reimplement capitalization or
   composition in Python. The literal binding name must remain statically visible to
   `tools/check_sase_core_rs_bindings`.
2. In `src/sase/ace/tui/prompt_catalog.py`, keep the xprompt/user/pending source merge unchanged, then pass that merged
   raw catalog through the facade. Extend `PromptCatalogSnapshot` with the uncomposed source catalog (or equivalently
   sufficient explicit-source/provenance state) so later overlays can be recomposed without mistaking an old generated
   `Foo` for an explicit collision.
3. Update `compose_pending_snippet_saves` and
   `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_save_xprompt_snippets.py` to base immediate live-save
   recomposition on raw explicit sources plus the complete pending-save overlay. This must update both `foo` and `Foo`
   on a second save, must preserve a genuinely explicit `Foo`, and must keep composition inside the existing
   `asyncio.to_thread`/catalog-worker paths rather than adding work to the Textual event loop.
4. In `src/sase/integrations/_editor_helper_snippets.py`, compose the merged raw template map through the same facade,
   create generated wire entries by cloning metadata from the provenance source, and publish the final templates and
   total count. Invalid triggers must continue to be filtered before composition.
5. Keep `PromptCatalogSnapshot.user_snippets` and config-location/save collision checks limited to authored user
   entries. Generated aliases must not look persisted or prevent a user from explicitly defining the capitalized name.
6. Add Python coverage for:
   - xprompt-derived, `ace.snippets`, and pending-save aliases in a prompt catalog snapshot;
   - explicit capitalized collisions at both xprompt/user precedence levels;
   - helper-bridge metadata inheritance, ordering, templates, and statistics;
   - `#[Foo]` references and reference-composed source capitalization;
   - immediate mounted-prompt expansion of a newly saved `Foo`;
   - a second save refreshing the generated template without overwriting explicit `Foo`;
   - facade wire validation and the required-binding inventory.
7. Before landing a Python caller that requires the new binding, advance SASE's `sase-core-rs` compatibility floor to
   the first published release-plz version containing it and refresh the repository's dependency metadata as required.
   Verify the exact-minimum binding smoke; do not point released SASE at an unpublished or older core wheel.

## Phase 3: Document and verify cross-surface parity

1. Update the snippet sections in `docs/configuration.md`, `docs/xprompt.md`, `docs/ace.md`, `docs/editor.md`, and the
   helper description in `docs/integrations.md`. Document the generated `foo`/`Foo` pair, exact first-character
   transformation, explicit-collision rule, runtime-only nature, reference support, and availability through both ACE
   and editor completion.
2. Re-read the final implementation for catalog-order drift and ensure all composition remains off the UI event loop. No
   default-config or keymap change is needed.
3. From the SASE repository, run `just install` first so the editable environment rebuilds the linked core, then run
   targeted Python tests for the snippet bridge, prompt catalog, live save flow, prompt expansion, and editor helper.
4. Run the linked core's focused tests while iterating, then run `just rust-check` for Rust formatting, Clippy, and the
   full workspace tests.
5. Finish with the mandatory `just check` in the SASE repository and the published-core-minimum binding smoke after the
   new core release is available. Fix and rerun until every check passes.

## Acceptance criteria

- With only `foo: "foo bar baz"` authored, every SASE catalog exposes both `foo` and `Foo`, and the expansions are
  exactly `foo bar baz` and `Foo bar baz`.
- The rule applies uniformly to xprompt-derived snippets, merged user snippets, and snippets saved into the current ACE
  session, including updates to an already-pending save.
- An authored `Foo` is never replaced by an alias generated from `foo`.
- Snippet references can target either spelling, and generated templates preserve tabstop and escape behavior.
- ACE, `sase editor helper-bridge snippet-catalog`, normal LSP completion, and native Rust fallback agree on triggers
  and templates.
- Generated aliases are never persisted, do not contaminate authored-name collision checks, and do not add synchronous
  work to prompt keystroke or render paths.
- Rust and Python full checks pass against a published core compatibility floor that contains the required binding.
