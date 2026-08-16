---
tier: tale
title: Rebalance Gemini 3.7 Flash size-alias defaults
goal:
  Route Antigravity Gemini 3.7 Flash Medium through @xsmall and Flash High through
  @small while keeping shipped defaults, regression coverage, and documentation
  synchronized.
size: small
proposed_by: bbugyi200.athena.030
create_time: 2026-08-15 19:50:02
status: done
---

# Plan: Rebalance Gemini 3.7 Flash size-alias defaults

## Context

SASE's shipped size-alias pools are defined in
`src/sase/llm_provider/model_alias_defaults.yml`. The current `@xsmall` pool includes
`agy/gemini-3.7-flash-high`, while `@small` has no Antigravity member. Antigravity
publishes separate stable model slugs for reasoning levels, so these pool members must
use the desired slug directly and must not add a generic trailing `@<effort>` suffix.

## Implementation

1. Update the shipped alias defaults so the Antigravity member remains last in each
   load-balanced pool, but resolves to `agy/gemini-3.7-flash-medium` for `@xsmall` and
   `agy/gemini-3.7-flash-high` for `@small`. Preserve all existing Claude, Codex, and
   Grok members, their effort suffixes, pool ordering, and the other size aliases.
2. Extend the real-packaged-default regression coverage in
   `tests/llm_provider/test_load_balanced_alias_defaults.py` to prove that an
   Antigravity-only availability result selects Flash Medium with no separate effort for
   `@xsmall` and Flash High with no separate effort for `@small`. Retain coverage for
   the generic providers' existing target and effort behavior; do not change the frozen
   fixture because only shipped target values, not alias graph shape, are changing.
3. Regenerate the model-alias table in `docs/llms.md` from the YAML source using
   `just fmt-docs`. Also update the nearby hand-authored Antigravity automatic-selection
   prose so it names both routes (`@xsmall` to Flash Medium and `@small` to Flash High)
   instead of referring to the removed `@cheaper` alias.

## Boundaries

- Do not change Antigravity's provider tier defaults, supported-model catalog, short
  aliases, or invocation behavior; both requested model slugs are already registered.
- Do not change selector grammar or load-balancing behavior. This work only adjusts two
  shipped pool memberships and the tests/documentation that describe them.

## Verification

1. Run `just install` before repository checks so the workspace has current editable
   development dependencies.
2. Run `just fmt-docs`, inspect the generated table and hand-authored model-mapping
   prose, and confirm the only default-pool changes are Flash Medium in `@xsmall` and
   Flash High in `@small`.
3. Run the targeted packaged-default tests with
   `just test tests/llm_provider/test_load_balanced_alias_defaults.py tests/llm_provider/test_model_alias_defaults.py`.
4. Run `just check` for the repository-required whole-tree lint gates and diff-scoped
   tests, then inspect the final diff for formatting errors and unrelated changes.
