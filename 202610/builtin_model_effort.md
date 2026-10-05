---
tier: tale
title: Builtin model and effort types with one routing classifier
goal: "Macro inputs declared as model or effort share one Rust classifier with the
  runtime binder, sase doctor, and the macro LSP. Model arguments complete from the
  %model catalog, warn with a replace fix, and hover with a classification. Effort is
  the closed seven-level enum. %model itself stays open and still falls back.

  "
size: medium
proposed_by: bbugyi200.athena.sase-1g4.4
bead: sase-1g4.4
create_time: 2026-10-05 09:11:14
status: wip
---

- **PARENT:**
  [202610/macro_named_input_types.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_named_input_types.md)
- **BEAD:**
  [sase-1g4.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g4/sase-1g4.4.md)

# Plan: Builtin model and effort types with one routing classifier

Implement phase `model-core` of epic `sase-1g4` (bead `sase-1g4.4`). The contract is the
epic plan `plan:202610/macro_named_input_types.md`, section "Builtin model and effort
types with one routing classifier", plus "The model contract" and "Messages". Where that
plan names the diagnostic `invalid_xprompt_arg_model`, emit `invalid_macro_arg_model`.
The choice-wire landing renamed the sibling code to `invalid_macro_arg_choice`; this
phase matches that `*_macro_arg*` spelling.

Before editing, read the epic plan and
`research:202610/macro_enum_inputs_named_types/macro_enum_inputs_named_types.md` with
`sase artifact read`. Open the linked core with `sase repo open sase-core` and read that
checkout's `AGENTS.md`. Work only in the printed path. Do not edit crate versions or
`CHANGELOG.md`. Never run bare `cargo`. Use `just fast` and
`just test -p <crate> <filter>` as the inner loop, and `sase tool run check` in
sase-core as the gate. Before finishing the sase repo, read `lint_and_test.md` with
`sase memory read`.

## Out of scope

Leave these to their own phases:

- Prompt-bar model menus, `ModelPickerModal`, ACE snapshots, and `docs/ace.md` (bead
  `sase-1g4.6`).
- Plugin `input_types.yml`, `sase macro types`, and `plugins.required` (bead
  `sase-1g4.5`).
- Retyping bundled or `sase-research-artifacts` macros, the memory rewrite, and the
  end-to-end parity macro.
- `%model` directive fallback. `%model:opsu` must still parse and fall back to the
  default provider exactly as it does today.

Do not create beads. Record a discovered follow-up as
`sase bead note sase-1g4.4 'PROPOSED FOLLOW-UP: …'`. A check that fails the same way on
the clean base tree is a follow-up, not a reason to keep the phase open.

## Catalog entries

In `crates/sase_core/src/macro_input_types/catalog.rs`, append two advertised builtin
rows after `agent`. Keep the existing rows and their order.

- `effort`: `InputTypeKind::NamedEnum`, base `enum`, no `value_role`. Choices are
  `EFFORT_LEVELS_ORDERED` from `effort.rs` (`none`, `minimal`, `low`, `medium`, `high`,
  `xhigh`, `max`). Description text is today's `EFFORT_SUGGESTIONS` documentation in
  `editor/directive/metadata.rs`. Move those seven strings next to
  `EFFORT_LEVELS_ORDERED` and build both the directive suggestions and the catalog
  choices from that one table. `label` stays unset.
- `model`: `InputTypeKind::Domain`, base `word`, `value_role` `model`, no choices.
  Description and rule: a model token that `%model` would accept and that routes to a
  provider without the silent default-provider fallback.

`builtin@effort` and `builtin@model` already resolve through the bare-name alias. Do not
add catalog aliases. `resolve_input_type` already copies choices and `named_type` for
`NamedEnum` and `Domain`.

Update the catalog name assertions in `macro_input_types/tests.rs` and the generated
schema tests. Advertised type names become the current list plus `effort` and `model`,
in that order:

`word`, `line`, `text`, `path`, `int`, `integer`, `float`, `bool`, `boolean`, `code`,
`enum`, `agent`, `effort`, `model`.

Regenerate the input sections of `src/sase/macros/workflow.schema.json` and
`src/sase/config/sase.schema.json` with `tools/sync_macro_input_schemas`. Update
`tests/test_macro_input_schemas.py`, which hardcodes the old name list. `string` stays
the deprecated non-advertised alias.

Effort then rides the existing closed-set path: LSP completion is `MacroArgumentValue`,
a non-member argument is `invalid_macro_arg_choice`, and a non-member frontmatter
default is an Error from `check_closed_set_default`. No snapshot is required for effort.

## Classifier

Add `ModelValiditySnapshot` and `classify_model_value(name, value, snapshot)` in a new
`sase_core` module, re-exported from that module's facade. Use a `*Wire`
request/response and a `thiserror` error only for malformed snapshots, not for rejected
values. A rejected value is a successful classification with `ok: false`.

Snapshot fields, serialized under the catalog file's `routing` object:

- `schema_version` (integer `1`)
- `providers`: registered provider names, including hidden providers such as `fakey`
- `models`: the `model_to_provider` map, including hidden models such as `fakey-large`.
  Store the provider value, not only the key, so hover can say which provider a known
  model uses. Validity tests key membership.
- `aliases`: `model_alias_names()` (configured, plugin-provided, and implicit size
  aliases). Names only, without a leading `@`. Do not store alias targets.
- `effort_levels`: the seven levels, in order

Classification never resolves an alias target, never takes the `consume` path, never
reads `~/.sase/llm_lb.json`, never makes a network call, and never consults provider
availability or disable state. The promise is "accepted without fallback", not
"executable right now".

Apply the steps in this order. `split_model_effort` peels a trailing `@<level>` only
when `<level>` is in the snapshot's effort levels. Any other `@` stays in the token so a
model id that contains `@` survives.

1. Peel a known trailing effort level.
2. `@name` is accepted when `name` is an alias, even if that alias's provider plugin is
   not installed. Otherwise it is rejected.
3. A bare token that is an alias name is rejected. Aliases need `@`.
4. `provider/rest` is accepted when `provider` is in `providers`. The rest stays open,
   including a rest that still contains `@` (`claude/opus@turbo` is accepted when
   `claude` is registered, because the model id is open and the suffix was not a known
   effort level). Otherwise the provider is not installed.
5. Any other bare token is accepted when it is a key of `models`. Otherwise it is
   rejected.
6. Step 6 only replaces the message of a value that steps 2–5 already rejected. If that
   value has a trailing `@suffix` whose suffix is not an effort level, and the body
   before that `@` would itself be accepted, the message says the suffix is not an
   effort level. `opus@turbo` takes this path because `opus` is a known model.
   `notaname@turbo` stays the generic rejection because the body would not route.
   `cluade/opus@turbo` stays the provider message because `cluade/opus` would not route.

Messages use backticks and the existing `suggest_closest` helper (at most three
suggestions). Canonical text:

- ``Argument `claude_model` expects a model, got `opsu`: not a known model, `@alias`, or `provider/model`; did you mean `opus`?``
- ``Argument `claude_model` expects a model, got `large`: model aliases need `@`; did you mean `@large`?``
- ``Argument `claude_model` expects a model, got `@lareg`: not a known model, `@alias`, or `provider/model`; did you mean `@large`?``
- ``Argument `claude_model` expects a model, got `cluade/opus`: provider `cluade` is not installed; did you mean `claude/opus`?``
- ``Argument `claude_model` expects a model, got `opus@turbo`: `turbo` is not an effort level (none, minimal, low, medium, high, xhigh, max)``

Suggestion pools: model keys for a bare unknown token, `@` plus alias names for an alias
miss, and `<provider>/<rest>` for a provider miss. The effort-suffix message lists the
seven levels and does not add did-you-mean.

The result also carries `kind` (`alias`, `provider_model`, or `known_model`) and
`provider` when one is known. Provider is the token prefix for `provider/rest` and the
map value for a known model. An alias has no provider, because the snapshot has no alias
targets.

Bind it in `crates/sase_core_py/src/macro_input_types/` next to `check_input_value`:
`#[pyfunction]` name `classify_model_value`, parse a request dict
`{name, value, snapshot}`, return the serialized result, and register it in that
domain's `register_*`. Add a binding round-trip test.

## LSP

### Routing block

`model_catalog.json` stays `schema_version: 1`. Add an optional top-level `routing`
object with the snapshot. `load_model_catalog` in
`sase_macro_lsp/src/server/catalogs.rs` keeps returning completion entries and ignores
`routing`. Add a sibling reader that returns the snapshot when `routing` is present and
well formed. A missing, malformed, or unreadable routing block skips model diagnostics
and the hover classification. It does not empty the completion list and it does not fail
catalog load. Log a warning for a malformed block.

`editor_analyze_document` today calls argument and frontmatter diagnostics with no
snapshot, and `push_diagnostic_with_data` always uses Error. Keep the current function
as the snapshot-absent path so existing tests stay valid. Add a with-snapshot entry and
thread `DiagnosticSeverity` through `MacroArgValidation` (default Error).
`diagnostics_for_document` in `sase_macro_lsp/src/server/actions.rs` loads routing from
the configured model catalog path and uses the with-snapshot entry.

### Completion kind

Add `CompletionContextKind::MacroArgumentModel`. `completion_kind_for_input` returns it
when `value_role == "model"`, in the same early branch as `agent`, before the choices /
bool / path fallback. A model hint has base `word` and no choices; without this branch
it would become `MacroArgumentTypeHint`.

Match the live serde spelling of the sibling variants. The derived `snake_case` name is
`macro_argument_model`. The pin test
`completion_context_macro_variants_pin_legacy_output` claims the siblings emit legacy
`xprompt_argument_*` while the enum derive emits `macro_argument_*`. Confirm which
spelling `MacroArgumentAgent` actually serializes. Follow that spelling for the new
variant and extend the pin only if the test is already green. Do not retarget the
sibling variants. If that test is already red on the untouched tree, leave it and record
a `PROPOSED FOLLOW-UP`.

Handle the new kind out of band in `completion_for_text`, the same way
`MacroArgumentAgent` and `MacroArgumentValue` are handled, and give
`completion_list_for_context` an empty arm so the match stays exhaustive. Include the
kind in the hover match in `editor/hover.rs`.

Serve completions from `model_completion_list` (the rich `%model` list, including `@`
alias rows). Filter with the current argument text, quotes stripped, using the same
partial window as `choice_completion`. Each item's `textEdit` spans the whole argument
value (`context.replacement_range`), not only the typed prefix. For one catalog file and
one prefix, the insertion texts equal the `%model` list for that prefix.

### Diagnostics and hover

An invalid model argument is a Warning, code `invalid_macro_arg_model`, with the
classifier message. Put suggestions in the existing `EditorDiagnosticData` shape. The
quick-fix title is ``Replace with `<value>` ``, the first suggestion is preferred, and
the edit replaces the whole value span. `diagnostic_quickfixes` already turns that data
into a code action. Do not warn when the snapshot is absent. `null` and unresolvable
argument values stay unchecked, as they are for choices.

A frontmatter `model` default that fails the contract is a Warning on the default's
range, code `invalid_macro_frontmatter_input_default`, using the classifier message and
the same replace fix. Skip it when the snapshot is absent. A default that fails the base
`word` rule (empty or whitespace) stays an Error. A non-member `effort` default stays an
Error through the existing closed-set default check.

Hover for a model argument is the existing argument hover (type label, source `builtin`,
default) plus one contract line: a model `%model` would accept and route without the
default-provider fallback. When the snapshot is present and the current value
classifies, add `Routes to `claude`.` for a known provider, or `Alias `@large`.` for an
alias. Thread the snapshot through `editor_hover_at_position_with_flags`; the LSP hover
handler in `server/actions.rs` loads the same routing block. No snapshot means the
contract line only.

## sase

### Snapshot builder

Add `model_validity_snapshot()` under `src/sase/llm_provider/`. Build it from
`_provider_names()`, `model_to_provider_map()`, and `model_alias_names()`, plus
`EFFORT_LEVELS_ORDERED` from `sase.macro.effort`. Include hidden providers and hidden
models. Do not call `resolve_model_provider` and do not read the load-balance cursor.

Cache it by `current_config_token()` the way `model_completion.py` caches its static
catalog. Export a cache clear. A monkeypatch of provider config does not move that
token, so doctor and binder tests that patch aliases or the registry clear the cache
before they assert.

`integrations/macro_lsp.py` `_materialize_model_catalog` writes the snapshot as
`routing` on the existing payload. If the snapshot cannot be built, still write the
completion entries and omit `routing`, so LSP startup is not blocked and model
diagnostics stay skipped. Do not change `MODEL_COMPLETION_CATALOG_SCHEMA_VERSION`.

### Binder

`InputArg.validate_and_convert` handles `named_type == "model"` before the `word`
branch. Call `classify_model_value` with `model_validity_snapshot()`. A rejected value
raises `MacroValidationError` with the classifier message. An accepted value stays a
string. `bind_input_args` already assigns `input_arg.default` without
`validate_and_convert`; do not add a default check there.

### Doctor

`check_config_model_macros` stops calling `_model_token_routes`. Delete that helper.

The directive parser strips `@` and peels a known effort suffix before
`PromptDirectives.model` is stored (`normalize_model_directive`). Classifying that
stripped string would warn on every healthy `%model:@large` preset. Reconstruct the
user-facing token first:

- When `model_alias` is set, classify `@` plus that alias. Do not glue
  `reasoning_effort` back on; it may have come from a separate `%effort` directive, and
  `@large` is already valid.
- Otherwise classify `directives.model` as stored (`claude/opus`, `codex/new-model`,
  `opus@turbo`).
- Classify `%model(alias=target)` override targets as stored.

Keep the retired-alias branch and its message ahead of classification. Keep today's
problem message:
`{name} -> {token} does not resolve to a provider; it will fall back to the default provider`.
Put the reconstructed token in `{token}`. Directive errors, including unknown `@alias`
and a bare alias name, stay the parser messages the doctor already records.

This deletes the loose "`/` means it routes" rule. Update
`test_model_macros_ignores_explicit_provider_model_token`: an unregistered prefix such
as `jetski/jetski-default` warns with that message. A registered prefix stays OK even
when the model id is unknown (`codex/new-model`). Hidden `fakey` stays OK. The other
doctor cases in `tests/doctor/test_checks_config_model_macros.py` keep their current
status and message text.

`check_config_macro_input_types` also warns when a loaded input has
`named_type == "model"` and a non-null string default the classifier rejects. Wording
includes "model default does not route" and the classifier message. Do not record a load
issue and do not skip the macro. Count the row with the existing warning rows.

### Parity

Add a hermetic parity test. One fixture registry supplies providers (including hidden
`fakey`), the model map (including `fakey-large`), and aliases (including `large`).
Build the snapshot from the builder under that fixture. For each corpus token, run the
classifier and run `resolve_model_provider_with_effort(..., consume=False)` together
with the directive parser:

- accepted ⇒ the resolver returns a provider, and the directive parser does not raise
- rejected ⇒ the resolver returns no provider, or the directive parser raises

Corpus, at least:

- accepted: `@large`, `claude/opus@xhigh`, `codex/new-model`, `fakey-large`,
  `fakey/fakey-large`
- `opsu` rejected, suggestion `opus`
- `@lareg` rejected, suggestion `@large`
- bare `large` rejected, suggestion `@large` (the directive parser raises)
- `cluade/opus` rejected with the provider message
- `opus@turbo` rejected with the effort message
- `%model:opsu` still parses and the resolver still returns no provider

Assert the cursor file `~/.sase/llm_lb.json` is unchanged across the test. Use fixture
snapshots for the classifier unit tests. The parity test is the one place that builds a
snapshot from a patched registry so both sides see the same world.

## Docs

In `docs/macros.md`, add `effort` and `model` rows to the Supported Types table. State
that `effort` is the seven `%effort` levels, matched exactly, and that `model` uses the
contract below. Keep `string`, unknown-type, and enum-choice text as it is.

In `docs/llms.md`, add a short "Macro model inputs" subsection beside per-prompt
provider switching. State the contract: valid exactly when `%model:<value>` would be
accepted by the directive parser and would route without the default-provider fallback;
aliases need `@`; `provider/model` is open for unknown model ids and closed for unknown
providers; validation does not move the alias cursor, check availability, or change
`%model` fallback.

## Verification

Must pass, using fixture snapshots rather than the live registry except for the parity
test's patched registry:

- The accepted and rejected corpus above, including suggestion text.
- `~/.sase/llm_lb.json` untouched by classification.
- `%model:opsu` still falls back.
- `type: effort` completion lists the seven levels in order, with the shared description
  text.
- A model argument's completion insertions match `model_completion_list` for the same
  prefix, and the text edit covers the whole value.
- An invalid model argument is a Warning `invalid_macro_arg_model` with a preferred
  `Replace with` fix. No warning when `routing` is absent.
- A bad `model` default is a frontmatter Warning. A bad `effort` default is an Error.
- Hover shows the contract line and `Routes to `claude`.` for `claude/opus@xhigh`.
- Doctor preset messages stay stable except the unregistered-provider case, which newly
  warns.
- Schema drift check passes.

In sase-core, run the touched `sase_core`, `sase_core_py`, and `sase_macro_lsp` tests,
then `sase tool run check`. In sase, run the binder, doctor, schema, and parity tests,
then the check named by `lint_and_test.md`.

After the core commit exists, move `sase-core-revision.txt` with
`just ratchet-core-revision`, as `docs/rust_backend.md` describes under the CI source
revision pin. sase stays red against the new binding until that pin moves. Do not invent
a hash for uncommitted core work.
