---
tier: tale
title: Add GPT-6 Sol selection and make it the Sol default
goal:
  GPT-6 Sol is selectable everywhere SASE exposes models, every SASE-owned default or
  preset that currently chooses GPT-5.6 Sol chooses GPT-6 Sol instead, and GPT-5.6 Sol
  remains available only when a user selects it explicitly.
size: medium
proposed_by: bbugyi200.athena.0r2
create_time: 2026-09-24 14:05:14
status: wip
---

# Plan: Add GPT-6 Sol selection and make it the Sol default

## Context and scope

OpenAI publishes the new model as `gpt-6-sol`. Its supported reasoning-effort values
include the `high` and `xhigh` rungs already used by SASE's `@large` and `@xlarge` Codex
members, so this migration does not require changing SASE's effort grammar or provider
invocation contract.

SASE has two shared model-catalog consumers rather than separate hard-coded catalogs:

- the provider registry feeds every ACE model picker, including launch, approval,
  provider-routing, alias-edit, and Models-panel flows;
- the same registry feeds the xprompt completion catalog used by ACE `%model`/`%m`,
  `=alias`, and `==model` completion and serialized for the Rust xprompt LSP used by
  external editors.

Therefore the implementation should extend the Codex provider's canonical metadata and
exercise both consumers. The Rust completion implementation only filters the serialized
catalog and does not own model identities, so no `sase-core` change or revision-pin bump
is expected.

This is a default migration, not removal of GPT-5.6 Sol. Keep `gpt-5.6-sol` and its
`gpt56sol` short alias registered so a prompt such as `%m:gpt-5.6-sol` continues to
resolve to Codex. Replace references whose purpose is to choose, recommend, demonstrate,
or prefill the current Sol model. Preserve historical records and a focused set of
tests/fixtures whose explicit purpose is proving legacy-model selection or generic
model-string propagation.

## Implementation

1. Extend the canonical Codex model metadata in `src/sase/llm_provider/codex.py`. Add
   `gpt-6-sol` alongside the other GPT-6 models, give it the compact short alias
   `gpt6sol`, and change the Codex `large` tier fallback from `gpt-5.6-sol` to
   `gpt-6-sol`. Keep the older model and alias in the catalog for explicit selection.
   Confirm that bare and `codex/`-qualified GPT-6 Sol values resolve through the normal
   registry path and that an unqualified/default Codex invocation emits the new model.

2. Move every shipped SASE-owned Sol default to GPT-6 Sol. In
   `src/sase/llm_provider/model_alias_defaults.yml`, replace the Codex members of both
   `@large` and `@xlarge` while preserving their current pool order and `high`/`xhigh`
   effort rungs. Update the checked-in `sase/xprompts/reads.md` Codex research lane and
   any other active preset or launch template found by the final repository-wide audit.
   Update `src/sase/default_config.yml` examples that document these defaults.

3. Refresh user-facing model-selection affordances. Replace GPT-5.6 Sol in current
   picker/selector placeholders, CLI `--model` examples, prompt-completion examples,
   configuration snippets, and current documentation with GPT-6 Sol. Add GPT-6 Sol and
   `gpt6sol` to the documented known-model and short-alias tables without deleting the
   GPT-5.6 Sol entries. Regenerate the model-alias table in `docs/llms.md` with
   `tools/render_model_alias_docs` after changing its YAML source of truth. Update the
   ACE, editor, xprompt, provider, configuration, SDD/bead, and relevant current blog
   examples where the old ID is serving as the recommended/current Codex model. Do not
   rewrite changelog/history, retained artifact critiques, or tests that intentionally
   use GPT-5.6 Sol as an arbitrary explicit legacy value.

4. Add focused regression coverage for every behavior boundary rather than mechanically
   rewriting unrelated fixtures:
   - provider tests assert the default Codex large tier and command now select
     `gpt-6-sol`, while an explicit `gpt-5.6-sol` override still passes through;
   - registry tests assert bare and provider-qualified GPT-6 Sol resolution plus the
     `gpt6sol` short alias;
   - model-picker and shared completion-catalog tests assert a first-class GPT-6 Sol row
     with Codex provenance and the new short alias, covering ACE and the LSP catalog
     source used by external editors;
   - real shipped-alias tests assert the `@large` and `@xlarge` Codex members use GPT-6
     Sol at their unchanged effort rungs and preserve pool rotation/availability
     behavior;
   - the checked-in reads-xprompt test asserts its Codex branch uses GPT-6 Sol;
   - retain or add one explicit legacy-selection assertion so the opt-in
     `%m:gpt-5.6-sol` contract cannot regress.

5. Audit the full repository after the edits. Classify every remaining `gpt-5.6-sol`,
   `gpt56sol`, and `%m`/`%model` occurrence. The only acceptable old-model references
   are the backward-compatible registry/alias entry, deliberate explicit
   legacy-selection coverage, and immutable historical material; no product default,
   built-in xprompt, UI placeholder, CLI help example, generated defaults table, or
   current recommended snippet may still steer users to GPT-5.6 Sol.

## Verification

1. Run `tools/render_model_alias_docs`, formatting as required, and verify a second run
   is idempotent.
2. Run focused tests for the Codex provider, registry resolution, model picker, xprompt
   completion catalog/LSP payload, shipped model-alias defaults, docs renderer, and the
   checked-in reads xprompt.
3. Exercise the catalog behavior directly: confirm `gpt-6-sol` appears in the picker
   rows and serialized completion payload with provider `codex` and alias `gpt6sol`, and
   confirm filtering `gpt6` and `codex/gpt-6` returns it.
4. Exercise routing directly: confirm the Codex large-tier fallback, `@large`, and
   `@xlarge` resolve to `gpt-6-sol` when Codex is the available provider, while an
   explicit `gpt-5.6-sol` still resolves to Codex unchanged.
5. Run repository-wide stale-reference searches and inspect the intentional allowlist.
6. Run the required final repository gate with `sase tool run check`. Do not run
   `just check-full`; the project policy reserves it for an explicit request.

## Acceptance criteria

- `gpt-6-sol` is offered by every registry-backed ACE picker and prompt completion and
  by the xprompt LSP catalog consumed by external editors.
- GPT-6 Sol has the stable display/filter alias `gpt6sol`, and both bare and
  provider-qualified model syntax resolve to Codex.
- Direct Codex large-tier fallback, the Codex members of `@large` and `@xlarge`, and
  bundled presets that previously selected GPT-5.6 Sol now select GPT-6 Sol with the
  same effort and routing semantics.
- No implicit/default launch path chooses GPT-5.6 Sol. A user-authored explicit
  `%m:gpt-5.6-sol` remains supported and unchanged.
- Current UI copy, CLI help, configuration examples, and documentation present GPT-6 Sol
  as the current Sol choice, and generated documentation matches its YAML source.
- Focused tests and `sase tool run check` pass.
