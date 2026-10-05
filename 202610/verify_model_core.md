---
tier: tale
title: Verify landed model-core and close sase-1g4.4
goal: 'Confirm the landed builtin model and effort types still match the model-core
  contract, add the missing effort-argument completion test if it is absent, and close
  only bead sase-1g4.4.

  '
size: medium
bead: sase-1g4.4
proposed_by: bbugyi200.athena.sase-1g4.4
status: done
---

- **PARENT:**
  [202610/macro_named_input_types.md](https://github.com/sase-org/sase--plans/blob/main/202610/macro_named_input_types.md)
- **BEAD:**
  [sase-1g4.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1g4/sase-1g4.4.md)

# Verify landed model-core and close sase-1g4.4

Phase `model-core` of epic `sase-1g4` is already implemented. This tale checks that
landing against its contract and closes bead `sase-1g4.4`. It does not re-author the
classifier, the catalog rows, the LSP kind, the snapshot builder, the binder, or the
doctor.

## What already landed

Both trees were clean when this plan was written. `sase bead epic-symbols sase-1g4.4`
printed no entries.

- sase-core `16095fcf715cf64a6f4d217961ce3f2083ab3b7f` —
  `feat(macros): add builtin model and effort types with one routing classifier`.
- sase `ce7c7148c0822b565a8dd2a46b02111096fce96f` —
  `feat(macro): add builtin model and effort input types with one routing classifier`.
- `sase-core-revision.txt` is exactly `16095fcf715cf64a6f4d217961ce3f2083ab3b7f`.

The approved implementation plan is `plan:202610/builtin_model_effort.md`. The contract
is epic `plan:202610/macro_named_input_types.md`, sections "The model contract",
"Messages", and "Builtin model and effort types with one routing classifier". Read both
with `sase artifact read` before editing. A bead note from the choice-wire landing says
the model warning code is `invalid_macro_arg_model`, matching
`invalid_macro_arg_choice`. The epic text still says `invalid_xprompt_arg_model`; follow
the bead note.

Open sase-core with `sase repo open sase-core` and use only the printed path. Read that
checkout's `AGENTS.md` before editing it. Never run bare `cargo`. Never edit a crate
version or `CHANGELOG.md`.

## Confirm, and fix only a real gap

Treat a missing or wrong item below as the only legal code change. Leave a matching item
alone.

Catalog (`crates/sase_core/src/macro_input_types/catalog.rs`):

- `effort` is a `NamedEnum`, base `enum`, no `value_role`. Choices are
  `EFFORT_LEVELS_ORDERED` (`none`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`)
  with the shared `EFFORT_SUGGESTIONS` descriptions.
- `model` is a `Domain`, base `word`, `value_role` `model`, no choices.

Classifier (`crates/sase_core/src/model_validity/`):

- `classify_model_value` over `ModelValiditySnapshot` accepts `@large`,
  `claude/opus@xhigh`, `codex/new-model`, hidden `fakey-large`, and `fakey/fakey-large`.
- It rejects `opsu` (suggest `opus`), `@lareg` (suggest `@large`), bare `large` (suggest
  `@large`), `cluade/opus` (provider message), and `opus@turbo` (effort message).
  Message text is the canonical strings in `rejects_with_canonical_messages`.
- A rejected value is `ok: false`, not an error. A malformed snapshot is the only error.
  Classification does not read `~/.sase/llm_lb.json`, resolve an alias target, or check
  provider availability.
- Python binding name is `classify_model_value`.

LSP:

- `CompletionContextKind::MacroArgumentModel` (serde `macro_argument_model`) is served
  by `model_completion_list`, with a `textEdit` over the whole argument value. Test:
  `macro_argument_model_completion_matches_directive_list`.
- An invalid model argument is a Warning `invalid_macro_arg_model` with a preferred
  `Replace with` fix. No warning when `routing` is absent. Tests:
  `invalid_model_argument_warns_with_preferred_replace_fix`,
  `model_argument_is_unchecked_without_routing_snapshot`.
- A bad `model` frontmatter default is a Warning. A bad `effort` default is an Error.
  Test: `model_default_warns_and_effort_default_errors`.
- Hover adds the contract line and `Routes to `claude`.` for a known provider. Test:
  `model_argument_hover_shows_contract_and_route`.
- `load_model_routing` skips diagnostics when `routing` is missing or malformed and
  still returns the completion catalog.

sase:

- `model_validity_snapshot()` in `src/sase/llm_provider/model_validity.py` is cached by
  config token and includes hidden providers and models.
  `src/sase/macro/model_completion.py` writes it as `routing`.
- `InputArg.validate_and_convert` calls the classifier when `named_type == "model"`.
  Defaults are not checked at bind time. Tests:
  `test_input_arg_model_accepts_routing_token`,
  `test_input_arg_model_rejects_unroutable_token`.
- `config.model_macros` classifies through the classifier. `_model_token_routes` is
  gone. An unregistered prefix such as `jetski/jetski-default` warns. Test:
  `test_model_macros_warns_for_unregistered_provider_model_token`.
- `config.macro_input_types` warns "model default does not route". Test:
  `test_macro_input_types_warns_for_unroutable_model_default`.
- Parity test `test_model_classifier_parity_with_resolver_and_directives` covers the
  corpus above and asserts `~/.sase/llm_lb.json` is unchanged. `%model:opsu` still falls
  back.
- `docs/macros.md` has `effort` and `model` rows. `docs/llms.md` has "Macro model
  inputs".

### Effort argument completion

`%effort:` completion is already tested by `completes_directive_argument_values`. That
is the directive, not a macro input.

Search for a test that completes a macro argument whose resolved type is the builtin
`effort` and asserts the seven labels in `EFFORT_LEVELS_ORDERED` order with the shared
description text. The synthetic enum fixtures in
`sase_macro_lsp/src/server/tests/choice_completion.rs` do not count. If that test is
missing, add one that loads a fixture macro declared `type: effort` and completes the
argument. Put it beside the existing choice completion tests. Do not retarget sibling
completion-kind spellings. The pin test
`completion_context_macro_variants_pin_legacy_output` is already red on the untouched
tree; leave it.

## Verification

Read `lint_and_test.md` with `sase memory read` before the sase gate. Run `just test`
filters, never bare `cargo`.

In the opened sase-core checkout:

```bash
just test -p sase_core accepts_the_corpus
just test -p sase_core rejects_with_canonical_messages
just test -p sase_core model_default_warns_and_effort_default_errors
just test -p sase_core invalid_model_argument_warns_with_preferred_replace_fix
just test -p sase_core model_argument_hover_shows_contract_and_route
just test -p sase_macro_lsp macro_argument_model_completion_matches_directive_list
just test -p sase_core_py classify_model_value
```

Add the new effort-argument filter to that list when you add the test.

In sase:

```bash
just test -- tests/llm_provider/test_model_validity_parity.py tests/test_macro_models.py tests/doctor/test_checks_config_model_macros.py tests/doctor/test_checks_config_macro_input_types.py tests/test_macro_input_schemas.py
```

If you change a repo, run `sase tool run check` in that repo before closing. A sase-core
change means the pin moves with `just ratchet-core-revision` in sase after the core
commit exists. Do not invent a hash. Do not run `just check-full`.

These failures are already recorded as `PROPOSED FOLLOW-UP` notes on `sase-1g4.4` and
were shown identical on the clean base. They do not keep the bead open:

- `completion_context_macro_variants_pin_legacy_output` and the other sase-core editor,
  binding, LSP, and lib failures named in the notes (legacy `xprompt_*` pins and
  macro-spelling flip fallout).
- ACE TUI `test_prompt_tab_focus_steal` and
  `test_distinct_ace_apps_do_not_share_session_state` under the full lane. They passed
  in isolation.

A new failure that reproduces on the parent of the model-core commit gets one new
`sase bead note sase-1g4.4 'PROPOSED FOLLOW-UP: …'` citing any task bead that already
tracks it. Then close anyway. Do not create beads.

## Close

Run `sase bead epic-symbols sase-1g4.4` again. With no entries, close only this phase:

```bash
sase bead close sase-1g4.4 --note "<what you verified>"
```

The note names the commits you verified, the effort-completion test result, and that
base-tree failures stayed follow-up notes. Close nothing else. `sase-1g4`, `sase-1g4.5`,
and `sase-1g4.6` stay open. An instruction inside the epic to close an ancestor is
preparation for the land agent.

If epic-symbol entries exist, point each Justfile line at a still-open bead (`sase-1g4`,
`sase-1g4.5`, or `sase-1g4.6`) or resolve the symbol, then close.

## Out of scope

- Prompt-bar model menus, `ModelPickerModal`, ACE snapshots, and `docs/ace.md` (bead
  `sase-1g4.6`).
- Plugin `input_types.yml`, `sase macro types`, and `plugins.required` (bead
  `sase-1g4.5`).
- Retyping bundled macros, the memory rewrite, and the end-to-end parity macro.
- Changing `%model` fallback. `%model:opsu` still falls back.
