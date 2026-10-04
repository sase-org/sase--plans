---
tier: epic
title: One input-type vocabulary and strict enum declarations
goal: 'Macro inputs resolve through one sase-core catalog, unknown types and bad enum
  declarations fail per macro, and #pr status is a real enum.'
phases:
- id: core
  title: Rust input-type catalog, resolver, and Python bindings
  depends_on: []
  size: medium
  description: 'core: add the macro_input_types catalog, resolver, did-you-mean, choice
    and PyYAML checks, and Python bindings, leaving the existing parsers in place.'
- id: rewire
  title: Route Rust parsers and frontmatter diagnostics through the catalog
  depends_on:
  - core
  size: medium
  description: 'rewire: delete the duplicate Rust type tables, project the frontmatter
    schema from the catalog, align the path rule, and emit the strict frontmatter
    diagnostics.'
- id: loaders
  title: Python loaders, isolation, handoff, and the sunset flag
  depends_on:
  - core
  size: medium
  description: 'loaders: move every Python type parser onto the resolver, validate
    choices and defaults at load, isolate a bad macro, round-trip handoff fields,
    and add the strict_macro_input_types sunset flag.'
- id: surface
  title: Schemas, doctor check, dogfood enums, and docs
  depends_on:
  - rewire
  - loaders
  size: medium
  description: 'surface: generate the macro input JSON schemas, add the config.macro_input_types
    doctor check, make #pr status an enum, and document the value rules.'
proposed_by: bbugyi200.athena.sase-1g4.1
parent_bead: sase-1g4.1
create_time: 2026-10-04 18:33:10
status: wip
bead_id: sase-1g4.1.1
---

- **PROMPT:** [prompts/202610/macro_input_type_vocab.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/macro_input_type_vocab.md)
- **PARENT:** [202610/macro_named_input_types.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_named_input_types.md)
- **BEAD:** [sase-1g4.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g4/sase-1g4.1.1.md)

# Plan: One input-type vocabulary and strict enum declarations

This epic is the implementation split for phase bead `sase-1g4.1` of
`plan:202610/macro_named_input_types.md`. That phase is size `large`, so it cannot be
one tale. The parent Design section is the contract. A phase that finds the contract
wrong records a `PROPOSED FOLLOW-UP:` note on its own bead and does not diverge.

The research artifact named by the parent plan
(`research:202610/macro_enum_inputs_named_types/macro_enum_inputs_named_types.md`) is
missing from the document root. Use the parent plan's Design section, which already
records the accepted recommendations and the later refinements.

## Rules for every phase

- Read `lint_and_test.md` with `sase memory read` before finishing. In sase, run
  `just fix` and then `sase tool run check`. Do not run `just check-full`.
- A phase that edits sase-core opens it with
  `sase repo open sase-core -r "Implement macro input type vocabulary"` and reads that
  checkout's `AGENTS.md` before editing. Never run bare `cargo`. Iterate with
  `just test -p <crate> <filter>`. Finish with `sase tool run check` from that checkout
  (about five minutes; allow at least ten).
- `core` and `rewire` are the only phases that edit sase-core. Each of those turns also
  moves sase's `sase-core-revision.txt` with `just ratchet-core-revision` so the host
  commits the pinned sibling first. `loaders` and `surface` call only bindings the pin
  already exposes. A missing binding is a gap in `core` or `rewire`: record a
  `PROPOSED FOLLOW-UP:` note. Do not edit sase-core from those phases.
- Do not add items to the crate-root `pub use` list in `crates/sase_core/src/lib.rs` or
  new `core_*` aliases in `crates/sase_core_py/src/prelude.rs`. Import
  `sase_core::macro_input_types::...`. A new module still needs `pub mod` in `lib.rs`.
- No `macro_rules!`. A multi-file module's `mod.rs` is only `mod` and `pub use`. Keep
  new files at or under 1,500 lines. Do not edit versions or changelogs.
- Do not create task beads. Record follow-up work as
  `sase bead note <this-phase-bead> 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.
  The one allowed bead creation is `sase flag new` in `loaders`.
- Do not close `sase-1g4`, `sase-1g4.1`, or any other ancestor. Close only the phase
  bead this child epic assigns. See [Landing this child epic](#landing-this-child-epic).
- Out of scope, owned by later phases of the parent epic: `effort` and `model` catalog
  entries, the model classifier, plugin `input_types.yml` discovery and loading, LSP
  completion, hover, and quick fixes, hint and catalog wires for `choices` /
  `named_type` / `value_role`, TUI choice menus, and the typed-form picker. This epic
  only reserves the empty plugin slot and the two qualified-name errors the resolver
  must already return.

`rewire` and `loaders` both depend only on `core`, so they can run together. They must
not edit the same files. `rewire` owns the Rust parsers, the frontmatter schema
projection, `MobileInputChoiceWire`, and the core pin. `loaders` owns the Python models,
loaders, handoff JSON, and the flag.

## Shared resolver contract

sase-core owns the messages. Use backticks. Add did-you-mean when a close match exists.
Rank case-insensitive equality and prefix matches first, then optimal string alignment
distance at most `max(1, len/3)`, and return at most three suggestions. List at most
eight values, then a count.

Messages this epic must produce verbatim:

- ``input `mode` has unknown type `enmu`; did you mean `enum`?``
- ``input `edition` uses `sase-research-artifacts@audio_edition`, but plugin `sase-research-artifacts` is not installed; run `sase plugin install sase-research-artifacts` ``
- ``plugin `sase-research-artifacts` declares no input type `audio_editon`; did you mean `audio_edition`?``
- ``choice `yes` must be quoted ("yes"): YAML reads it as a boolean``
- ``choice `in progress` contains whitespace; choice values are single words``
- ``choice `null` is reserved (it means "use the default")``
- ``choice `prod` is declared twice``
- ``default `turbo` is not one of fast | thorough``
- ``Argument `edition` expects one of brief | full, got `breif`; did you mean `brief`?``

`builtin()` has an empty plugin list, so every `<dist>@<id>` it sees uses the
not-installed message. The declares-no-input-type message is still implemented and
tested by passing a registry that contains the distribution and not the id. Later phases
fill that registry. Do not discover plugins here.

Bare names resolve case-insensitively to the canonical lowercase spelling.
`builtin@<name>` is an alias of that bare name. `integer` is `int`. `boolean` is `bool`.
`string` resolves to base `line` with a deprecation warning, not an error. An unknown
bare name is a resolver error with suggestions. There is no silent `line` fallback
inside Rust. The sunset flag lives only in the Python loader.

Resolution result fields:

- `base`: the wire kind older consumers see (`word`, `line`, `text`, `path`, `int`,
  `float`, `bool`, `code`, `enum`, or `agent`).
- `named_type`: set for a domain (in this epic, only `agent`). Unset for scalars and for
  inline `enum`.
- `value_role`: `agent` for `agent`, otherwise null.
- `choices`: empty in this epic. Named enums arrive in a later phase.
- `deprecated`: true only for the `string` alias.

Catalog rows for this epic:

| Name                                                    | Kind         | Base    | Role    | Notes                                       |
| ------------------------------------------------------- | ------------ | ------- | ------- | ------------------------------------------- |
| `word` `line` `text` `path` `int` `float` `bool` `code` | scalar       | itself  | none    | today's value rules                         |
| `string`                                                | scalar alias | `line`  | none    | `deprecated_alias_of: line`, not advertised |
| `enum`                                                  | inline_enum  | `enum`  | none    | requires inline choices                     |
| `agent`                                                 | domain       | `agent` | `agent` | word rules                                  |

Choice rules, for inline enums now and named enums later:

- Values must be strings. A non-string JSON or YAML scalar is an error telling the
  author to quote it. Python must pass the raw scalar to Rust. Do not call `str()`
  first: PyYAML has already turned unquoted `yes` into `True`, and the original token is
  gone. Say that the value arrived as a boolean, int, float, or null and must be quoted.
  The Rust frontmatter path still has the source text, so it uses the verbatim
  ``choice `yes` must be quoted ("yes"): YAML reads it as a boolean``.
- Each value is non-empty, free of Unicode whitespace, not the literal `null`, and
  unique. Matching is exact and case-sensitive. A label is never a value.
- Item keys are exactly `value` (required), `label`, and `description`. Both label and
  description are free text and are never inserted. A scalar item stays allowed. An
  unknown key is an error.
- Characters that need quoting in shorthand (`,` `+` `(` `)` `[` `]` `"` `'` and
  backtick) are a warning. Warnings never fail a load.
- A closed-set default must be a string member. That is a load error in Python and an
  Error diagnostic in Rust. Defaults are not checked again at bind time.
- Repeatable enum inputs check each element. `check_input_value` checks one element. The
  caller loops.

These choice rules run in the macro, workflow, and config loaders. They do not run in
`InputArg.__post_init__`. Gates share `InputArg` and keep today's declaration rules
there: enum requires choices, every other type forbids them, and duplicate values are
rejected. The membership check inside `InputArg.validate_and_convert` does move to Rust,
so gates gain did-you-mean.

`path` accepts a single line. Spaces are allowed. A newline is not. That is already the
Python rule in `InputArg.validate_and_convert`. Rust still rejects any whitespace in
`value_matches_input_type` and in `validate_input_default`. `rewire` makes Rust match
Python.

## Rust input-type catalog, resolver, and Python bindings

Create `crates/sase_core/src/macro_input_types/` with the catalog, registry, resolver,
and validators above. `InputTypeRegistry::builtin()` returns the rows in the table and
an empty plugin list. `resolve_input_type(raw, &registry)` returns the resolved type or
a typed error whose message is one of the verbatim strings.

`suggest_closest` is the shared helper used by unknown types, unknown plugin ids, and
enum membership. Do not copy a second distance function into the editor crate.

`validate_enum_choices` accepts JSON values from Python and YAML values from Rust. It
returns the parsed choices plus issues that carry a severity (`error` or `warning`) and
a message. It does not format floats or bools into choice values.

`pyyaml_plain_scalar_is_non_string(text)` ports PyYAML's implicit resolver, not a fresh
reading of the YAML spec. Copy the bool, int, float, null, and timestamp patterns from
the installed PyYAML `resolver.py` (`yaml_implicit_resolvers`). The caller invokes it
only for an unquoted plain scalar. Lock the port with a vector that includes `yes`,
`no`, `on`, `off`, `true`, `false`, `null`, `Null`, `~`, an integer, a float, and a
timestamp, and that rejects `ready`, `brief`, and other ordinary words.

`check_input_value(resolved, name, value)` performs enum membership for one element and
returns the verbatim `Argument ...` message. With no choices it is a no-op. Scalar
checks stay where they are in this phase. `validate_and_convert` keeps its existing
scalar and bool arms.

Python bindings live in `crates/sase_core_py/src/macro_input_types/`, registered from
`crates/sase_core_py/src/lib.rs` next to `register_editor_completion`. Expose the
catalog, `resolve_input_type`, `validate_enum_choices`,
`pyyaml_plain_scalar_is_non_string`, and `check_input_value`. Follow a neighbor:
`#[pyfunction]`, `#[pyo3(name = "...")]`, `fn py_<name>`, request wire in,
`serialize_to_py` out, and a round-trip test in that domain's `tests.rs`. A missed
`m.add_function` compiles and then raises `AttributeError` in sase.

Do not change `editor/frontmatter.rs`, `editor/diagnostics.rs`, or
`macro_catalog/parsing.rs` in this phase. Those parsers keep today's silent `line`
fallback until `rewire`.

This phase's tests, using fixture registries rather than a live plugin install:

- `enmu` suggests `enum`. `builtin@word` and `WORD` resolve to `word`.
- `string` resolves to base `line` with `deprecated` set.
- `agent` has base `agent`, `named_type` `agent`, and `value_role` `agent`.
- An empty registry reports the not-installed message for
  `sase-research-artifacts@audio_edition`. A fixture registry that has the distribution
  and not the id reports the declares-no-input-type message, including the
  `audio_editon` suggestion.
- The PyYAML vector above.
- Choice items: a bool, an int, whitespace, `null`, a duplicate, an unknown key, a
  shorthand comma warning, and a `{value, label, description}` item.
- `breif` against `brief` and `full` produces the verbatim Argument message. `Brief`
  does not match `brief`. A label is not accepted as the value.
- The catalog contains only the rows in the table.

Finish by ratcheting `sase-core-revision.txt`.

## Route Rust parsers and frontmatter diagnostics through the catalog

Delete the duplicate vocabularies and call `macro_input_types` instead:

- `crates/sase_core/src/macro_catalog/parsing.rs`: `parse_input_type` (silent `line`
  fallback), `parse_input_choices`, and `value_as_string` (drops floats).
- `crates/sase_core/src/editor/diagnostics.rs`: `parse_input_type_name`, and
  `parse_local_inputs`, which must now keep choices. `value_matches_input_type` for
  `path` becomes "one line, spaces allowed".
- `crates/sase_core/src/editor/frontmatter.rs`: `InputType::ALL`, `aliases`, `rule`,
  `parse_input_type`, `XPROMPT_INPUT_TYPE_EXPECTED`, `validate_input_choices`, and
  `validate_input_default`. A private base-kind enum may remain only as data the
  resolver returns. It must not carry a second alias table.

Frontmatter diagnostics, pointed at the offending item when there is one:

- Unknown type: Error, with the resolver's suggestion text.
- `string`: Warning that it is deprecated and `line` should be used.
- Unquoted plain scalar that `pyyaml_plain_scalar_is_non_string` flags, on a choice or
  on an enum default: Error, with the verbatim quote message and the source text.
- Shorthand-hostile characters: Warning.
- Non-member closed-set default: Error, with the verbatim default message.
- Per-item choice ranges, not one range on the whole `choices` field.

An unknown type still stored for the editor catalog uses base `line`, and the macro file
still carries the Error diagnostic. That is the editor behavior. It is not the Python
loader's sunset-flag behavior.

`frontmatter_input_type_schema` (`editor/frontmatter.rs` `input_type_schema`, bound as
`frontmatter_input_type_schema`) becomes a projection of the catalog. Add `kind`,
`description`, and `source` to `FrontmatterInputType` with serde defaults so older
payloads still load. `string` is present and `advertised: false`.

`MobileInputChoiceWire` in `crates/sase_core/src/host_bridge.rs` gains `description`,
with `#[serde(default, skip_serializing_if = "Option::is_none")]`. Choice parsers copy
it. Regenerate the mobile contract with
`UPDATE_MOBILE_CONTRACT=1 just test -p sase_gateway committed_`. Do not add `named_type`
or `value_role` to hint or mobile input wires. That is the parent `wire-lsp` phase.

If `docs/editor.md` states that `path` rejects spaces or lists the old type vocabulary
as exhaustive, update that sentence. Do not rewrite `docs/macros.md` here.

This phase's tests:

- `choices: [yes, no]` produces the verbatim quote error in frontmatter validation.
- A non-member enum default is an Error.
- `type: enmu` is an Error suggesting `enum`, and the catalog entry is kept as `line`.
- `type: string` is a Warning and resolves as `line`.
- A path value with a space matches. A path value with a newline does not.
- `parse_local_inputs` and `parse_input_choices` keep `description`.
- The schema projection includes `kind`, `description`, and `source`.

Finish by ratcheting `sase-core-revision.txt` again.

## Python loaders, isolation, handoff, and the sunset flag

Start with the corpus scan the parent contract requires, before any loader rejects a
previously accepted choice. Search installed plugins plus home and project macros for
`choices`. Record the result with `sase bead note` on this phase bead. No bundled macro
uses `enum` today (`src/sase/macros/` has no `type: enum`). If the scan finds an
offender, the same sunset flag also gates the new choice-value and non-member-default
load errors: flag off keeps today's acceptance, flag on applies the Rust rules. If the
scan finds none, those rules are unconditional load errors. Unknown type names always
obey the flag, whether or not the scan finds anything.

Create the flag only with `sase flag new`. Read `sase_flags.md` first. Do not hand-edit
a bead and do not call `sase bead create`.

```bash
sase flag new strict_macro_input_types -k sunset \
  --when-enabled "Unknown macro input type names are a per-macro load error with suggestions." \
  --when-disabled "Unknown macro input type names silently resolve to line, as they did before this flag." \
  --remove-when "No maintained macro source still relies on an unknown type name resolving to line."
```

Paste the printed registry entry into `src/sase/feature_flags/registry.py`. Resolve the
flag through `snapshot.enabled(FeatureFlag.strict_macro_input_types)` at the load site.
No import-time read. Test both states. Regenerate the feature-flag schema block with the
existing `tools/sync_feature_flags_schema` flow if that check fails. Do not build the
macro-input schema tool in this phase.

`InputChoice` gains `description: str | None = None`. `InputArg` gains
`named_type: str | None = None` and `value_role: str | None = None`. `type` stays the
base `InputType`.

Replace `parse_input_type` in `src/sase/macro/loader_parsing.py` with an adapter over
`resolve_input_type`. It must return the base type together with `named_type`,
`value_role`, and the deprecation warning. A bool-only helper will drop those fields.
Update every caller:

- shortform and longform in `loader_parsing.py`
- `workflow_loader_parse.py` `parse_workflow_inputs`, which today drops `choices`
- `ace/tui/modals/input_item_modal.py`
- `ace/tui/modals/macro_item_modal.py`
- `ace/tui/widgets/_frontmatter_panel_cell_editing.py`

The three TUI callers already reject an unknown type against the schema before they call
`parse_input_type`. Keep that. They only need the richer result so `named_type` and
`value_role` survive a save. Do not make the modals consult the sunset flag.

Choice parsing in the macro and workflow loaders calls `validate_enum_choices` with the
raw YAML scalars. A warning is recorded and does not skip the macro. An error raises
`MacroValidationError`. A closed-set default that is not a string member is a load
error. Workflow longform must read `choices` and choice `description`. It keeps today's
rule that a missing longform default means explicit null.

Warnings do not go through `record_load_issue` unchanged.
`check_config_macro_definitions` prints every load issue as `skipped:`. Use kind
`input_type_warning` for the `string` deprecation and shorthand warnings, and teach that
check to ignore the kind. Errors that skip a macro use kind `input_type` and stay
visible as skips.

Per-macro isolation, so one bad declaration never escapes the loader:

- `load_macro_from_file` (`loader_sources.py`) catches `MacroValidationError` around
  `parse_inputs_from_front_matter`, records an `input_type` load issue, and returns
  `None`.
- `parse_macro_entries` catches the same error per entry, records the issue, and
  continues with the other entries. This covers config macros and local macros.
- `load_plugin_markdown_macros` has its own `parse_inputs_from_front_matter` call. Catch
  it there too. Skills already go through `load_macro_from_file`.
- `load_workflow_from_mapping` already catches `MacroValidationError` from
  `parse_workflow_inputs`. Keep that, and make sure a warning does not take that path.

Handoff JSON in `src/sase/agent/multi_prompt_macros.py` round-trips `choices` (`value`,
`label`, `description`), the input `description`, `repeatable`, `named_type`, and
`value_role`. Old files omit those keys and deserialize as empty choices, `repeatable`
false, and null names. `prompt_frontmatter._input_to_yaml` writes `type: <named_type>`
when `named_type` is set, and writes choice `description`. It does not write resolved
choices back onto a named type. This phase has no named enum, so the named-type branch
is only `agent`, whose base is already `agent`.

`InputArg.validate_and_convert` for `InputType.ENUM` calls `check_input_value`. Update
gate tests that assert the old `expects one of a, b, got 'c'` text so they expect the
did-you-mean message. Leave `__post_init__` declaration checks as they are.

This phase's tests:

- A file containing `choices: [yes, no]` fails in the Python loader with a quote error.
  The message says the value arrived as a boolean and must be quoted.
- A non-member enum default is rejected at load, and the sibling macros in the same file
  or config mapping still load.
- A longform workflow enum loads, including `description` on a choice.
- `type: enmu` with the flag on records an `input_type` issue naming `enum` and skips
  only that macro. With the flag off, the input loads as `line` and the macro stays.
- `type: string` loads as `line` and records an `input_type_warning`.
- An enum local macro survives `serialize_local_macros` and `deserialize_local_macros`
  with choices, description, repeatable, and named_type.
- A gate enum value still validates, and a near miss includes the suggestion.

## Schemas, doctor check, dogfood enums, and docs

Add `tools/sync_macro_input_schemas` with `--check`, mirrored on
`tools/sync_feature_flags_schema` and `feature_flags_schema_drift`. It regenerates the
input-type sections of `src/sase/macros/workflow.schema.json` and the
`macroInputDefinitions` block of `src/sase/config/sase.schema.json` from the catalog
binding:

- the type enum is the catalog's advertised names and aliases, plus `string` marked
  `deprecated`, plus a `<dist>@<id>` string pattern
- `choices` items accept a string or `{value, label, description}`
- `repeatable`, `agent`, `enum`, and `code` are present (the config schema currently
  lacks all four; the workflow schema has `agent` but not `enum` or `code`)

A unit test fails when either document drifts, the same way the feature-flag schema
check does. Wire the check into the existing schema-sync gate rather than inventing a
second lint entry point.

Add doctor check `config.macro_input_types` and register it in
`src/sase/doctor/checks_config.py` `config_check_specs`. It lists `input_type` load
issues and `input_type_warning` issues across macro sources, each with the Rust
suggestion text still in the message. Model the check on
`check_config_macro_definitions` in `checks_config_macros.py`. Warnings are WARN, not a
load failure. Document the id in `docs/configuration.md` beside the other `config.*`
doctor ids.

Dogfood, after the loaders accept a real enum:

- `src/sase/macros/pr.yml` `status` becomes `type: enum` with values `wip`, `draft`, and
  `ready`. Descriptions come from the display side of `status_map` in
  `src/sase/workflows/commit/commit_tracking_patch.py` (`WIP`, `Draft`, `Ready`). Keep
  the default `draft`. Do not change that module's `.lower()` map. Case sensitivity is
  the macro contract. `#pr:ready` must still bind the first positional input `name`, not
  `status`.
- `src/sase/macros/eval_ifs_loops.yml` changes both `type: string` inputs to `line`.

Vocabulary parity: one shared corpus runs through Rust `value_matches_input_type` and
Python `validate_and_convert` for every scalar, including a `path` with spaces and a
`path` with a newline. They agree.

Docs:

- `docs/macros.md` Supported Types and Enum Choices: value rules, `description`,
  quoting, the unknown-type error, `string` as a deprecated alias of `line`, and `path`
  as a single line that allows spaces.
- `docs/workflow_spec.md`: the type table's `path` row, and the `enum` row that says
  choices are shortform-only. Longform `choices` works.
- The placeholder in `input_item_modal.py`
  (`word · line · text · path · int · float · bool`) so it includes `agent`, `enum`, and
  `code`.

This phase's tests, which close the parent phase's must-pass list:

- The schema-sync check passes.
- `sase doctor -C config.macro_input_types` reports a fixture `string` warning and a
  fixture unknown-type error, and says nothing about a clean tree of bundled macros
  after the `eval_ifs_loops.yml` edit.
- `#pr(x, status=ready)` binds. `status=Ready` fails with a `ready` suggestion.
  `#pr:ready` binds `name`.
- The parity corpus passes.
- `choices: [yes, no]` was already covered in `loaders` and `rewire`. Do not weaken
  those tests.

## Landing this child epic

This section is evidence for the child epic's land agent. It is not work for a phase
worker, and it is not permission for a phase worker to close an ancestor.

When every phase above is closed, the land agent closes parent bead `sase-1g4.1` only.
Before that close, run `sase bead epic-symbols sase-1g4.1`. Re-key any `--epic-symbol`
entry that still names this phase to a still-open bead (the parent epic `sase-1g4` or a
later parent phase). `sase bead close` refuses while leftovers remain. Do not close
`sase-1g4` or any ancestor above `sase-1g4.1`.

The close note should say what was verified: the Rust catalog and both parsers agree,
unknown types follow `strict_macro_input_types`, a bad enum skips only its macro, `#pr`
status is the `wip | draft | ready` enum, and the schema-sync check passes. A check
failure that reproduces on the clean base tree is a `PROPOSED FOLLOW-UP:` note on
`sase-1g4.1`, not a reason to leave the bead open.
