---
tier: tale
title: Add Gemini 3.7 Flash Medium to the cheaper model pool
goal:
  The shipped @cheaper alias load-balances onto Gemini 3.7 Flash Medium when Antigravity
  is available.
size: small
proposed_by: bbugyi200.athena.01w.f1
create_time: 2026-08-14 20:14:03
status: done
---

- **PROMPT:**
  [prompts/202608/gemini_37_flash_cheaper_pool.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gemini_37_flash_cheaper_pool.md)
- **AGENTS:**
  - [bbugyi200.athena.01w.f1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.01w.f1.md)
- **COMMITS:**
  - [97e12b2](https://github.com/sase-org/sase/commit/97e12b29e4c0a72425396f5a2baca8c751801e80)
    — feat(llm): add antigravity flash to cheaper pool

# Add Gemini 3.7 Flash Medium to the `@cheaper` pool

## Objective

Extend SASE's shipped `@cheaper` load-balanced model alias with Antigravity's Gemini 3.7
Flash Medium model. Extra-small tale and epic-phase launches that inherit through
`@xsmall_worker` should be able to rotate onto Antigravity whenever the `agy` CLI is
installed, while preserving the existing Claude, Codex, and Grok members and all user
override behavior.

## Current state and decisions

- The canonical shipped alias policy is
  `src/sase/llm_provider/model_alias_defaults.yml`; `@xsmall_worker` falls back to
  `@cheaper`, whose current round-robin pool is
  `claude/sonnet@medium | codex/gpt-5.5@medium | grok/grok-4.6@medium`.
- SASE already publishes the stable Antigravity slugs `gemini-3.7-flash-high`,
  `gemini-3.7-flash-medium`, and `gemini-3.7-flash-low`. The provider's large and small
  tier defaults remain High and Low respectively; this change does not alter the
  Antigravity registry or tier mapping.
- Add `agy/gemini-3.7-flash-medium` as the final pool member. Medium is the direct
  Antigravity counterpart to `@cheaper`'s existing medium-effort members. Do not add an
  `@medium` suffix: Antigravity encodes that choice in the stable model slug and exposes
  no independent reasoning-effort mechanism, so a suffix would only create an
  unsupported-effort warning.
- Pool resolution already filters members by registered, installed provider CLI.
  Therefore the new member participates only when `agy` (or `SASE_AGY_PATH`) is
  available and does not disturb machines without Antigravity beyond changing the
  shipped pool fingerprint.
- Bryan's chezmoi-managed SASE config has no built-in `cheaper` override, and its
  `m_cheap` xprompt delegates to `%m:@cheaper`. No chezmoi change or deployment step is
  needed; the shipped SASE policy remains the single source of truth.
- This is a value-only change to the existing alias graph. Do not update
  `tests/_model_alias_defaults_fixture.py`, whose frozen values intentionally isolate
  behavioral tests from shipped-value changes and whose graph shape is unchanged.

## Implementation

1. Update the `cheaper.target` value in `src/sase/llm_provider/model_alias_defaults.yml`
   by appending `agy/gemini-3.7-flash-medium`, preserving the existing three members and
   single-pipe round-robin semantics.
2. Add a focused regression in `tests/llm_provider/test_load_balanced_alias_defaults.py`
   that opts into the real packaged defaults, makes only `agy` targets available,
   resolves `@cheaper`, and asserts the selected target is `agy/gemini-3.7-flash-medium`
   with no resolved effort. This covers both pool membership and the provider-specific
   no-suffix decision. Keep the existing registry/model guard in
   `tests/llm_provider/test_model_alias_defaults.py` as the independent check that the
   new target names a published model on a registered provider.
3. Run `just fmt-docs` to regenerate the marked implicit-alias table in `docs/llms.md`
   from the canonical YAML. In the Antigravity model-mapping prose, document that Gemini
   3.7 Flash Medium can also be selected automatically through `@cheaper` when the
   Antigravity CLI is available. Leave illustrative custom-alias examples unchanged
   because they demonstrate override syntax rather than shipped values.

## Verification

1. Run the focused real-default and provider tests:

   ```bash
   .venv/bin/pytest -q \
     tests/llm_provider/test_model_alias_defaults.py \
     tests/llm_provider/test_load_balanced_alias_defaults.py \
     tests/llm_provider/test_agy_provider_core.py
   ```

2. Confirm `just fmt-docs` is idempotent by rerunning it and checking that it produces
   no further diff. Inspect the generated `@cheaper` row to ensure it contains the four
   intended members in order and renders the pipe separators correctly.
3. Run `just check` as the required repository gate. If its scoped lane escalates or
   reports unusual selection, follow the repository guidance and run `just check-full`
   through `/sase_monitor` with an explicit next action.
4. Review the final diff and verify it is limited to the canonical alias default,
   focused regression coverage, and generated/adjacent documentation; confirm the linked
   chezmoi checkout remains untouched.

## Acceptance criteria

- The shipped `@cheaper` selector is exactly
  `claude/sonnet@medium | codex/gpt-5.5@medium | grok/grok-4.6@medium | agy/gemini-3.7-flash-medium`.
- With only Antigravity marked available, resolving `@cheaper` yields
  `agy/gemini-3.7-flash-medium` and no separate reasoning effort; with Antigravity
  unavailable, the existing availability filter continues to skip it.
- `@xsmall_worker` continues to inherit through `@cheaper`, existing configured and
  temporary overrides retain precedence, and no other built-in alias or provider tier
  changes.
- The generated model-alias documentation reflects the new shipped pool and the
  Antigravity section explains its automatic `@cheaper` route.
- Focused tests and `just check` pass, with any required escalation completed according
  to the repository's verification policy.
