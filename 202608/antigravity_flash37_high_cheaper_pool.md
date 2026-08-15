---
tier: tale
title: Use Gemini 3.7 Flash High for Antigravity in @cheaper
goal:
  The shipped @cheaper pool selects Gemini 3.7 Flash High for Antigravity while
  preserving medium effort for every other provider in the pool.
size: small
proposed_by: bbugyi200.athena.020
create_time: 2026-08-15 07:17:36
status: wip
---

- **PROMPT:**
  [prompts/202608/antigravity_flash37_high_cheaper_pool.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/antigravity_flash37_high_cheaper_pool.md)

# Plan: Use Gemini 3.7 Flash High for Antigravity in `@cheaper`

## Context and decision

Antigravity does not support SASE's generic reasoning-effort option, so an `@xhigh`
suffix would be rejected rather than making Gemini smarter. Antigravity instead exposes
separate model slugs for its variants. Its current catalog lists `gemini-3.7-flash-high`
as the most capable Gemini 3.7 Flash variant, ahead of the `medium` and `low` variants,
and already uses that high slug as its large-tier default.

Use `agy/gemini-3.7-flash-high` as the Antigravity member of the shipped `@cheaper`
pool. Keep the existing `claude/sonnet@medium`, `codex/gpt-5.5@medium`, and
`grok/grok-4.6@medium` members unchanged. Do not add a generic effort suffix to the
Antigravity target, change Antigravity's provider-wide tier mapping, or alter any other
model alias.

## Implementation

1. In `src/sase/llm_provider/model_alias_defaults.yml`, replace only the Antigravity
   member of the `cheaper` target with `agy/gemini-3.7-flash-high`; retain the order and
   exact medium-effort targets for Claude, Codex, and Grok.
2. In `tests/llm_provider/test_load_balanced_alias_defaults.py`, update the packaged
   `@cheaper` regression coverage to select and expect the Antigravity high model with
   no separate effort value. Extend the real-default assertion so it also locks in the
   requested provider-specific asymmetry: Gemini 3.7 Flash High for `agy`, and medium
   effort for each non-Antigravity pool member.
3. In `docs/llms.md`, change the nearby Antigravity automatic-selection prose from the
   medium slug to the high slug. Run `just fmt-docs` to regenerate the marked shipped
   alias table from the YAML source of truth instead of editing that table by hand.

## Verification

1. Run `just install` first so the workspace's editable environment and development
   dependencies are current.
2. Run the focused load-balanced-alias test module and the model-alias-default tests to
   exercise the packaged target, provider filtering, effort parsing, and defaults
   schema.
3. Run `just fmt-docs` again and confirm it produces no further diff, proving the
   generated documentation agrees with the YAML source.
4. Run `just check` for the required whole-repository lint gates and diff-scoped test
   lane. Inspect the final diff to confirm only the Antigravity `@cheaper` member moved
   from `medium` to `high`, all other pool members remain at `@medium`, and the test and
   documentation describe the same behavior.
