---
tier: tale
title: Restore Gemini 3.7 Flash High to @xsmall only
goal:
  Route the shipped Antigravity Gemini member exclusively through @xsmall as
  gemini-3.7-flash-high, removing Gemini from the @small pool while preserving every
  other shipped size-alias member and behavior.
size: small
proposed_by: bbugyi200.athena.04a
create_time: 2026-08-16 16:04:57
status: wip
---

# Context

The relevant history establishes a clear routing progression:

- `97e12b29e` added `agy/gemini-3.7-flash-medium` to the former `@cheaper` pool.
- `718357102` upgraded that member to `agy/gemini-3.7-flash-high`, with no trailing
  generic effort because Antigravity exposes the capability level in the model slug.
- `2fcca46eb` replaced the older role aliases with the five built-in size aliases,
  carrying Flash High into `@xsmall` and leaving `@small` as the Claude/Codex/Grok
  high-effort pool.
- `85c09a886` changed `@xsmall` from Flash High to Flash Medium and added Flash High to
  `@small`, updating the packaged-default regression and generated documentation at the
  same time.

This change restores the state immediately before `85c09a886`: only `@xsmall` contains
an Antigravity member, and that member is `agy/gemini-3.7-flash-high`. The other
`@xsmall` and `@small` members, their ordering, and their explicit effort suffixes
remain unchanged. The alias graph also remains a set of direct selectors, so no schema,
migration, launch-routing, or Rust-core change is required.

# Implementation

1. Update `src/sase/llm_provider/model_alias_defaults.yml`, the documented single source
   of truth for shipped size aliases:
   - Replace `agy/gemini-3.7-flash-medium` with `agy/gemini-3.7-flash-high` as the final
     `@xsmall` pool member, without an `@<effort>` suffix.
   - Remove `agy/gemini-3.7-flash-high` from `@small` while retaining the existing
     Claude, Codex, and Grok entries in their current order at high effort.
   - Leave `@medium`, `@large`, `@xlarge`, descriptions, and configuration override
     behavior untouched.

2. Revise `tests/llm_provider/test_load_balanced_alias_defaults.py` around the real
   packaged defaults:
   - Assert that provider-filtered `@xsmall` resolution selects
     `agy/gemini-3.7-flash-high` for Antigravity with `effort is None`, while preserving
     the existing medium-effort expectations for its other providers.
   - Preserve coverage for the remaining `@small` Claude/Codex/Grok high-effort members,
     and explicitly lock that the shipped `@small` selector has no Antigravity member.
     Inspect selector membership directly for this absence instead of relying on the
     resolver's all-unavailable fallback-to-first-member behavior.
   - Keep unrelated rotation, availability, override, and cursor-state tests unchanged.

3. Synchronize `docs/llms.md` with the shipped defaults:
   - Update the Antigravity model-mapping prose to say that `gemini-3.7-flash-high` is
     selected automatically through `@xsmall` only; remove the Flash Medium and `@small`
     routing claim.
   - Run `just fmt-docs` to regenerate the marked model-alias defaults table from
     `model_alias_defaults.yml`, rather than editing that generated block by hand.

# Verification

1. Run `just install` before repository checks so the workspace uses current editable
   dependencies.
2. Run the focused packaged-default suites:
   `pytest tests/llm_provider/test_load_balanced_alias_defaults.py tests/llm_provider/test_model_alias_defaults.py`.
3. Run `just fmt-docs` again and confirm `git diff --exit-code -- docs/llms.md` so
   generated documentation is reproducible after the source and prose updates.
4. Run `just check` for the required whole-repository lint gates and diff-scoped tests.
   Escalate to monitored `just check-full` only if scoped selection reports an unusual
   escalation or the implementation expands into a broadening-set change.

# Acceptance criteria

- The shipped `@xsmall` selector ends with `agy/gemini-3.7-flash-high` and resolves it
  without a separate effort value.
- The shipped `@small` selector contains no Gemini/Antigravity member and keeps its
  existing Claude, Codex, and Grok high-effort members unchanged.
- The adjacent Antigravity documentation and generated alias table describe the same
  routing as the YAML source.
- Focused regression tests and `just check` pass.
