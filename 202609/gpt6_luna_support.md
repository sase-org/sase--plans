---
tier: tale
title: Add GPT-6 Luna across SASE model selection
goal:
  GPT-6 Luna is selectable across SASE UX and powers the small size aliases at
  descending effort levels.
size: medium
proposed_by: bbugyi200.athena.0sg
create_time: 2026-09-25 17:06:16
status: wip
---

# Add GPT-6 Luna across SASE model selection

## Goal

Make `gpt-6-luna` a first-class Codex model in SASE, including direct model routing, the
model picker, prompt-input model completion, and external-editor xprompt LSP completion.
Replace the Codex members of the shipped `@medium`, `@small`, and `@xsmall` size aliases
with GPT-6 Luna at descending efforts. Preserve explicit access to `gpt-5.6-terra` and
`gpt-5.6-luna` as legacy models.

OpenAI's
[GPT-6 Luna model page](https://developers.openai.com/api/docs/models/gpt-6-luna)
confirms the exact `gpt-6-luna` identifier and support for `medium`, `high`, and `xhigh`
reasoning effort. The SASE `decisions:size-alias-effort-ladder` record requires the
first appearance, scanning from `@xlarge` downward, to use `xhigh` and each later
appearance to drop one effort rung.

## Implementation

1. Add `gpt-6-luna` to `CodexProvider.llm_known_model_names()` and map it to the unique
   display shorthand `gpt6luna` in `llm_model_short_aliases()` in
   `src/sase/llm_provider/codex.py`. Follow the existing registry path so bare and
   `codex/`-qualified model names resolve to Codex, while model picker rows and agent
   display shorthands inherit the same metadata. Check for other active hard-coded model
   catalogs and update any that would otherwise omit Luna.
2. Change only the Codex members of `src/sase/llm_provider/model_alias_defaults.yml`:
   `@medium` to `codex/gpt-6-luna@xhigh`, `@small` to `codex/gpt-6-luna@high`, and
   `@xsmall` to `codex/gpt-6-luna@medium`. Keep the other providers and
   `@large`/`@xlarge` routing intact. Retain legacy model registration for explicitly
   selected GPT-5.6 models.
3. Verify the shared completion flow rather than creating another model list:
   `build_model_completion_catalog()` feeds the ACE prompt input (including `%model` and
   model shortcuts), and `model_completion_catalog_payload()` is materialized by
   `src/sase/integrations/xprompt_lsp.py` for external editors. Add or adjust focused
   tests asserting Luna's canonical and provider-scoped completion entries, `gpt6luna`
   display/filter hint, model picker visibility, bare and qualified provider resolution,
   and the serialized LSP catalog row. Include a representative Codex launch/effort
   assertion if existing tests do not cover the new alias selection end to end.
4. Update the shipped alias expectations in
   `tests/llm_provider/test_load_balanced_alias_defaults.py` and rely on its
   effort-ladder invariant. Refresh `docs/llms.md`'s generated alias table with
   `tools/render_model_alias_docs`, then update its active provider-resolution and
   short-alias tables. Refresh the current size-alias examples in
   `src/sase/default_config.yml` to use Luna at the matching effort levels; leave
   unrelated historical examples and tests alone.

## Verification

- Run focused registry, alias-default, completion-catalog, model-picker, LSP-catalog,
  and Codex invocation tests. Confirm the LSP JSON exposes `gpt-6-luna` with `gpt6luna`,
  and that prompt completion and provider-scoped filtering expose the same model.
- Check the resulting UI screenshot cases that render changed model or alias rows;
  regenerate only affected snapshots through the project's visual snapshot recipe and
  inspect every generated image change.
- Run `sase tool run check` from this repo after edits. Fix reported lint, formatting,
  documentation, or selected-test failures and repeat the check until it passes. Do not
  run `just check-full` unless explicitly instructed.

## Acceptance criteria

- Users can select and launch `gpt-6-luna` by canonical bare or `codex/`-qualified name;
  picker, prompt input, and external-editor LSP offer the model with its `gpt6luna`
  hint.
- Shipped `@medium`, `@small`, and `@xsmall` resolve their Codex members to GPT-6 Luna
  at `xhigh`, `high`, and `medium`, respectively; other provider members and larger-size
  aliases keep their current targets.
- Current documentation reflects the new defaults and shorthand, and the required check
  passes.
