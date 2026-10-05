---
tier: epic
title: Carry macro input metadata and finish enum assistance in the LSP
goal: 'Complete the wire-lsp scope of sase-1g4.2: preserve resolved choices, named
  types, and value roles across catalogs and projections; share Rust choice candidates
  and type labels; and make enum completion, diagnostics, quick fixes, and hover agree
  with the runtime binder.'
parent_bead: sase-1g4.2
phases:
- id: contracts
  title: Resolved Rust wires, shared choice candidates, and type labels
  depends_on: []
  size: medium
  description: 'contracts: preserve resolved metadata through Rust catalog/editor/mobile
    wires, implement and bind the shared candidate builder and type label, classify
    argument contexts using roles and choices, and establish the parser-driven golden
    corpus and additive wire compatibility tests.'
- id: completion
  title: Enum completion and frontmatter type completion in the LSP
  depends_on:
  - contracts
  size: medium
  description: 'completion: consume shared choice candidates in standard LSP responses,
    replace the bool-only path, implement catalog-backed frontmatter type completion,
    and verify full-value text edits through JSON-RPC.'
- id: diagnostics
  title: Choice diagnostics, diagnostic-driven fixes, and rich argument hover
  depends_on:
  - contracts
  - completion
  size: medium
  description: 'diagnostics: carry structured suggestions and edits in diagnostics,
    classify invalid enum arguments with the required error code, implement invocation
    and frontmatter quick fixes, and render shared type labels, provenance, defaults,
    and bounded choice tables in hover.'
- id: projections
  title: Python catalogs, mobile and highlight wires, and macro show
  depends_on:
  - contracts
  size: medium
  description: 'projections: preserve rich input metadata in Python catalog, mobile,
    and highlight projections, use Rust type labels in signatures and macro show,
    mirror the golden corpus, and test candidate edits against the binder.'
- id: parity
  title: Cross-surface acceptance and phase closure evidence
  depends_on:
  - completion
  - diagnostics
  - projections
  size: small
  description: 'parity: verify real LSP completion and quick-fix edits against the
    runtime binder, finish editor and macro documentation, run required checks in
    both repositories, and record symbol and verification evidence for the assigned
    phase''s completion without closing any ancestor.'
proposed_by: bbugyi200.athena.sase-1g4.2
create_time: 2026-10-05 02:19:19
status: wip
bead_id: sase-1g4.2.1
---

- **PROMPT:** [prompts/202610/macro_choice_wires_lsp.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/macro_choice_wires_lsp.md)
- **PARENT:** [202610/macro_named_input_types.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_named_input_types.md)
- **BEAD:** [sase-1g4.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g4/sase-1g4.2.1.md)

# Plan: Macro Choice Wires and LSP Assistance

## Assignment and boundaries

This is implementation of the already assigned phase `sase-1g4.2`, whose parent is
`sase-1g4`. The controlling design is `plan:202610/macro_named_input_types.md`,
specifically its **Choices, named types, and roles on every wire; enum completion and
diagnostics in the LSP** section. Read that artifact and
`research:202610/macro_enum_inputs_named_types/macro_enum_inputs_named_types.md` through
`sase artifact read` before implementing. The epic design's refinements take precedence
over the earlier research (particularly `named_type`, rather than `type_ref`, and the
model-routing contract).

Choose an epic because this phase crosses the Rust catalog and editor contracts, PyO3,
the LSP transport, and Python projections. Contracts must land first. Python projections
can proceed independently of LSP implementation after that; the two LSP phases are
ordered because they share transport/conversion files. The final parity phase integrates
the resulting behavior.

Shared behavior belongs in linked `sase-core`; Python is a thin adapter and a renderer.
Open core with `sase repo open sase-core -r "<specific reason>"` and use only its
printed path. Read its `AGENTS.md`. Paths below labeled **core** are relative to that
checkout; other paths are relative to the sase checkout.

Keep these related phases out of scope:

- `sase-1g4.3` owns Python TUI hint dataclasses, menus, typed forms, authoring modals,
  and visual snapshots. This work supplies their Rust API and catalog metadata without
  changing TUI presentation or duplicating its detection logic.
- `sase-1g4.4` owns the `model`/`effort` entries, model classifier, and model menus.
- Later parent phases own plugin type loading/discovery, `sase macro types`, plugin
  dogfooding, and memory updates. The APIs here must carry named metadata generically;
  do not implement those features early.
- Do not add signature help, new LSP trigger characters, a second enum spelling,
  whole-frontmatter parsing, or gate-input named types.

## Starting state and preparation

The completed dependency `sase-1g4.1` was closed on 2026-10-05. Its note reports the
shared catalog, Python resolver/declaration adapters, the sunset flag, schema sync,
doctor check, and `#pr.status` enum landed. The planning checkout's Python files still
predate that completion. Refresh clean checkouts through the normal SASE sync path
before editing, then inspect the landed code; preserve any local changes. Do not
recreate vocabulary-phase work merely because this snapshot is old. Read
`sase bead read sase-1g4.1 -r "Need the completed dependency handoff"`.

Core already has `macro_input_types`, `MobileInputChoiceWire.description`,
`MacroInputHint.choices`, enum membership checking, and local choice parsing. However,
the hint lacks named type/role fields, context classification still dispatches by base
type, and the LSP returns a bool-only value list. Most Python projections still drop
choices. Some names in the parent plan have since changed: the LSP crate is
`crates/sase_macro_lsp`; the mobile input wire is `MobileMacroInputWire`; the shared
argument corpus is becoming `macro_args_corpus.json`. Search by symbol rather than
reverting terminology.

Before verification, read `lint_and_test.md` through `/sase_memory_read`. Use
`sase tool run check` in each changed repository; it is the required wrapped form of
`just check`. Read `symvision.md` before touching unused-symbol exceptions. Core
requires its complete check, including binding tests, even after targeted tests pass.
Never run bare cargo. Use `just fast` and `just test -p <crate> <filter>` for the inner
loop.

## Shared behavior contract

`type` on every wire remains the base kind. Resolved named enums are base `enum` plus
choices; open domains keep their base kind and role. Add optional `named_type` and
`value_role` and preserve rich choices with `value`, `label`, and `description`.
Deserialize old payloads with serde/dataclass defaults and omit new empty/null Rust
fields. Keep existing schema versions for these additive changes unless an actual
incompatible contract requires coordinated migration.

The role on editor/mobile hints is the existing `DirectiveValueRole`. Convert the
resolver's string role through one core-owned mapping without changing the resolver's
Python wire contract. Preserve legacy base `agent` handling when old payloads omit the
role. Unknown/unresolved editor inputs retain the existing per-macro fallback and
diagnostics rather than breaking the catalog.

Expose `macro_argument_choice_candidates(hint, partial, replacement, selected)` and
`macro_input_type_label(hint)` from the owning core module and register both bindings in
the existing editor-completion domain. Use one small serializable choice result with
enough explicit metadata for LSP and the subsequent TUI consumer: canonical value,
insertion, label, description, declared index, default marker, and replacement edit. Do
not encode these facts into display strings and then parse them back. Follow existing
request-dict conventions at the PyO3 edge; document and test the binding payloads.

The candidate builder:

- Returns declared order with an empty prefix; otherwise prioritizes case-insensitive
  value prefixes and uses the shared Rust fuzzy matcher for the remaining matches, with
  stable declared-order ties.
- Inserts the exact canonical value, never its label. Quote and escape characters that
  are structural in macro argument syntax, especially commas and shorthand `+`. Use the
  same core insertion helper for quick fixes. Completed edits must round-trip through
  the parser and runtime binder.
- Synthesizes `true`/`false` for bool; retains all existing accepted bool spellings in
  validation. Marks the actual displayed default without reordering or preselecting it.
  Excludes selected values only for repeatable inputs, and keeps the active value
  eligible when editing an existing repeatable element.
- Does no plugin discovery, network lookup, or Python-side filtering. The menu can hold
  the entire set; if the LSP caps it, report real truncation explicitly.

The type label is the canonical value union for up to four declared choices; otherwise
`<named_type> (N)` or `enum (N)`. Domains show their named type, scalars their keyword.
Synthetic bool suggestions do not change the scalar `bool` label.

## contracts: Core wires, candidates, labels, and fixtures

1. Extend **core** `editor/wire.rs::MacroInputHint`,
   `host_bridge.rs::MobileMacroInputWire`, and `macro_catalog/types.rs::CatalogInput`.
   Preserve resolved metadata in both shortform and longform catalog parsing and in
   local-input parsing. Update every constructor and serde fixture, including
   placeholders and tests. `assist_entries_from_catalog` and
   `macro_catalog/entries.rs::structured_inputs` must copy all metadata rather than
   reconstructing it from the base type.
2. Implement the two shared functions in focused modules under the editor or
   macro-input-type domain, exporting through its facade, not crate-root aliases. Use
   the type label in core catalog `format_inputs`. Cover inline enums, manually
   constructed resolved named-enum hints, agents, bool, and scalars.
3. Update `editor/completion/trigger_context.rs::completion_kind_for_input`: agent role
   (or legacy base agent) -> `MacroArgumentAgent`; nonempty choices or base bool ->
   `MacroArgumentValue`; path -> `MacroArgumentPath`; remaining inputs ->
   `MacroArgumentTypeHint`. Model role dispatch remains for its phase.
4. Replace the delimiter-counting value-span shortcuts with the existing shared argument
   parser's spans where needed. Support empty values, mid-value cursors, quoted
   delimiters, named/positional/colon forms, and the final repeatable input. Prefix
   filtering uses text before the cursor; the replacement covers the whole current
   value, including an existing quoted wrapper where appropriate. Keep literal zones and
   dynamically unresolved arguments working.
5. Add **core** `crates/sase_core/tests/fixtures/macro_arg_choice_completion.json`,
   schema v1, with case id, source text, unambiguous cursor, input declarations,
   expected context kind, canonical candidates/insertions, and replacement span. State
   coordinate units explicitly and compare both UTF-8 byte spans and converted UTF-16
   editor ranges for non-BMP cases. Rust tests drive actual detection, builder, and
   applied-edit parsing. Include empty/prefix/fuzzy cases, bool, repeatable exclusions,
   comma/plus quoting, quoted mid-value replacement, and `#pr:ready` binding the word
   input `name`, not enum `status`.
6. Add PyO3 registration and round-trip tests, including old hints without new fields.
   Update sase's `tools/validate_sase_core_rs` binding probes and their tests for the
   new calls; ensure the pin-based binding check sees them. Regenerate the gateway
   contract with `UPDATE_MOBILE_CONTRACT=1 just test -p sase_gateway committed_`;
   inspect the snapshot diff for intended additive changes only.

Acceptance: legacy payloads still deserialize; rich choice metadata survives catalog ->
mobile/editor hint -> binding; fixture edits parse to their declared values; candidate
ordering/defaults and type-label boundaries pass focused tests. Document the binding
contract in the corresponding core API documentation and `docs/rust_backend.md` as
appropriate.

## completion: LSP choice and frontmatter completion

1. In **core** `crates/sase_macro_lsp/src/server/completion.rs`, resolve the active hint
   from the detected macro/input, then call the shared candidate builder. Delete the
   macro bool-only implementation/import once its last consumer is migrated. Preserve
   the existing agent and path completion paths.
2. Add a dedicated conversion in `lsp_convert.rs`: closed-set items use `ENUM_MEMBER`, a
   parser-derived full-value `textEdit`, canonical `filterText`, declared-index
   `sortText`, optional label in `labelDetails.description`, and description
   documentation. Default badges are display metadata; insertion is only the encoded
   value. Return a `CompletionList` with `isIncomplete` when truncated; check the
   untruncated case as well.
3. Add frontmatter input-type value completion from the shared catalog, using the
   existing source index to recognize input slots even while YAML is incomplete. Cover
   shortform scalar declarations and dict/longform `type:` values. Replace the entire
   current type value. Include catalog descriptions and canonical spellings, obey
   current advertised/feature-gated type visibility, and support later registry entries
   without a duplicate hard-coded vocabulary. Do not offer type names in unrelated YAML
   values or the Markdown body.
4. Extend server and JSON-RPC tests: `#deploy(env=` yields `staging`, `prod` with
   metadata; completion in the middle of an existing value replaces its suffix;
   `#deploy:`, named and positional forms, bool, quoted punctuation, and UTF-16
   positions behave like the golden corpus. Verify the initialization response's
   existing trigger characters remain sufficient and unchanged.
5. Update `docs/editor.md` to describe the completion behavior that now ships.

## diagnostics: Structured fixes and rich hover

1. Add optional typed diagnostic data in **core** `editor/wire.rs`: canonical
   suggestions plus explicit quick-fix titles, replacement edits, and preferred status.
   Existing diagnostics have no data. Update constructors, bindings, and
   `lsp_convert.rs::diagnostic` so this survives all the way to JSON-RPC.
2. In `editor/diagnostics.rs`, enum membership uses `check_input_value` with the full
   resolved hint. Invalid closed-set values are Errors with the required externally
   visible code `invalid_xprompt_arg_choice`, rather than the generic type-error code.
   Populate suggestions with the shared Rust ranking helper, not message scraping.
   Validate every repeatable element and retain existing null/default sentinel and
   dynamic-value handling. The same result must drive argument validity and
   semantic-token mismatch classification.
3. Extend frontmatter diagnostic construction to carry source-indexed fixes: unknown
   type -> `Change type to ...` using catalog suggestions; deprecated `string` ->
   `Use line`; unquoted PyYAML-typed choice/default -> a quoting fix. Preserve per-item
   ranges and existing severities from the vocabulary phase. Applying a quote fix
   preserves the intended string and removes that quoting error. Do not weaken
   declaration checks or reimplement their messages.
4. Thread `params.context.diagnostics` through the LSP code-action path, which currently
   calls `code_actions_for_text` without consuming it. Produce standard `QUICKFIX`
   workspace edits named `Replace with ` plus the backticked canonical value, marking
   the top suggestion preferred. Use diagnostic-provided edits, filter to the
   document/request range, ignore unsupported or malformed data, respect requested
   action kinds, and retain existing refactors and refresh actions. Add a direct test
   that a published diagnostic's data is consumed.
5. In `editor/hover.rs`, render the shared type label, the input type's source
   (`builtin` or the distribution from its qualified named type), the default, and a
   value/label/description Markdown table. Escape table content correctly; cap at 12
   rows with an exact remaining count. Include `MacroArgumentAgent` in hover dispatch.
   Do not mistake a macro's source for its input type's source.
6. JSON-RPC tests must prove `edition=breif` publishes an Error with suggestions, its
   preferred quick fix replaces only the value with `brief`, and the edited invocation
   is clean. Cover positional/colon and repeatable failures, case-sensitive membership,
   labels being rejected, valid defaults, non-BMP ranges, frontmatter fixes, and
   agent/large-set hover. Update editor docs.

## projections: Python adapters and presentation

1. Inspect the now-landed vocabulary implementation before editing. It supplies
   `InputChoice.description`, `InputArg.named_type/value_role`, parser resolution,
   handoff JSON, and named-type YAML serialization. Preserve those round trips.
2. Extend `macro/_catalog_models.py::StructuredCatalogInput` and
   `_catalog_structured.py::structured_inputs` with rich choices and optional named
   type/role. Add them to `integrations/_mobile_helper_catalog.py` and to
   `macro/highlight.py::_macro_input_hint_to_wire`. The highlight serializer must
   tolerate old object/dict hints and copy richer hints when provided. Its tests can use
   enriched duck-typed hints; TUI dataclass changes belong to the next parent phase.
3. Keep mobile string defaults redacted by default. Carrying public choice metadata must
   not opt mobile into `include_string_defaults=True` or expose a private string default
   via a newly added badge. Local/editor paths may retain their existing default
   visibility.
4. Add only the thin binding adapters required by real consumers, and call the shared
   type label from `_catalog_format.py::format_inputs`. Keep optional and repeatable
   markers and implicit step-input filtering intact. No Python copy of fuzzy matching,
   membership, quoting, or type-label rules.
5. Extend `cli_show_model.py::ShowInput`, `properties.py::show_inputs`, and
   `cli_show_render.py` so the CLI shows the type label and each choice's label and
   description. Preserve the existing JSON base `type` and string-valued `choices`
   array; add `type_label`, rich `choice_details`, `named_type`, and `value_role`
   additively under the current show schema. Text rendering uses those rich fields
   without losing raw values or existing default information.
6. Mirror the golden JSON into `tests/fixtures/macro_arg_choice_completion.json` and add
   byte-equality verification against the sanctioned opened core tree, following the
   existing argument corpus pattern. Drive the core bindings from Python for every case;
   apply completion edits and run the real macro parser and binder against a fixture
   macro. Preserve full TUI detection parity for `sase-1g4.3` while supplying its corpus
   now.
7. Add regression coverage to catalog, mobile helper, highlight, macro properties, and
   CLI-show tests. Check metadata survives, old input shapes still work, named types
   retain their base kinds, and mobile default redaction stays intact. Document
   signatures and CLI choice display in `docs/macros.md`.

## parity: Integration, verification, and completion evidence

Use fixture macros, not the installed live model/plugin registry. Reuse the real LSP
session helper in `tests/_macro_directive_completion_parity_lsp.py` and existing
argument parity tests. Assert actual JSON-RPC payloads and applied edits:

- `#deploy(env=` lists staging/prod with full-span edits and rich choice metadata.
- Applying each completion yields the exact declared value accepted by the runtime
  binder, including comma/plus and quoted-value cases.
- `edition=breif` is an Error; its preferred `Replace with` action produces `brief`,
  removes the error, and binds successfully.
- Hover lists choices, source, and default, and large tables report truncation.
- Repeatable elements, bool, non-BMP coordinates, and `#pr:ready` agree across the
  corpus, LSP, highlight wire, and runtime. The latter still binds `name`.

Run targeted tests for every touched core crate (`sase_core`, `sase_core_py`,
`sase_macro_lsp`, `sase_gateway` as applicable), then formatting and
`sase tool run check` in both changed repositories. Run `just fix` in sase before its
final gate. Use `/sase_monitor` for a genuinely long verification command and wait for
the handoff command itself to exit; never end with an unmonitored command still running.
Do not run `just check-full`, which was not requested.

The preceding vocabulary phase recorded a setup probe mismatch (Python expected
content-layout schema 5, clean pinned core returned 6, and a stale stats probe). If it
still blocks verification, independently reproduce the identical failure on the clean
base/pinned core, cite the existing dependency note and any tracking bead, and record
`PROPOSED FOLLOW-UP:` on the assigned phase. This is evidence for a pre-existing
failure, not permission to bypass bindings or ignore a new failure. Fix failures
introduced by this work before completing.

Core changes and Python callers must land with a pin that contains the bindings. Read
`docs/rust_backend.md`'s CI source revision section. Declare both changed repositories
through `/sase_final`; the host commits core first and writes its pushed SHA into
`sase-core-revision.txt` before the primary commit. If the core change is already
landed, use `just ratchet-core-revision` and verify the resulting pin instead. Do not
create commits/branches/PRs manually or guess an uncommitted core SHA. Keep versions and
changelogs host-owned.

Before closing a delegated child, run `sase bead epic-symbols <that-child-id>`. Consume
or remove any now-used symbols; re-key genuinely future consumers to a still-open later
phase or the appropriate parent epic. Record verification and close only the child
assigned to that worker. Do not close any ancestor, including `sase-1g4.2` or `sase-1g4`
from a child worker.

The land agent must assemble complete evidence for the original `sase-1g4.2` assignment
and run `sase bead epic-symbols sase-1g4.2` again immediately before its authorized
settlement. Planning found no entries, but later work can add them. Any leftovers must
be resolved or re-keyed before the phase closes. Use
`sase bead close sase-1g4.2 --note "<what was verified>"` only from its assigned
completion context; delegated workers supply evidence for that context and do not close
an ancestor themselves. Never close `sase-1g4`.

No worker may create follow-up beads. Record discovered unrelated work as
`sase bead note sase-1g4.2 'PROPOSED FOLLOW-UP: <summary — detail>'` for the parent
epic's land agent to triage. Do not set reserved/in-progress statuses by hand.
