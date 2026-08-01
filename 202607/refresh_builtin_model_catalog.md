---
tier: tale
title: Refresh the builtin model catalog
goal: Builtin pickers omit redundant Claude 5 point versions, retain floating Claude defaults, and offer Codex Spark.
create_time: 2026-07-31 06:59:40
status: done
---

- **PROMPT:** [prompts/202607/refresh_builtin_model_catalog.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/refresh_builtin_model_catalog.md)

# Refresh the Builtin Model Catalog

## Objective

Remove the redundant explicit `claude-opus-5` and `claude-sonnet-5` entries from SASE's builtin model pickers while
keeping the floating Claude `opus` and `sonnet` defaults intact, and register `codex/gpt-5.3-codex-spark` so Spark is
available in both the modal model picker and `%model` completion.

## Current Behavior and Contract

- Provider `llm_known_model_names()` metadata is the shared source for the modal picker, the `%model` completion catalog
  (including the JSON snapshot consumed by the Rust xprompt LSP), and automatic provider inference for bare model names.
  `llm_model_short_aliases()` supplies display/filter hints and same-provider fan-out suffixes.
- Claude's large/small tier defaults already resolve to the provider-owned floating aliases `opus` and `sonnet`; SASE
  does not map those aliases to the explicit `claude-opus-5` or `claude-sonnet-5` strings. Preserve the tier mapping and
  keep `opus` and `sonnet` registered.
- Remove `claude-opus-5` and `claude-sonnet-5` from Claude's known-model and short-alias metadata. They will disappear
  from both builtin picker surfaces and cease to receive implicit Claude routing when written as unqualified bare names.
  Explicit `claude/claude-opus-5` and `claude/claude-sonnet-5` values remain valid through the existing provider/model
  syntax and Custom picker path.
- Register the unqualified model id `gpt-5.3-codex-spark` under the Codex provider so selecting it from a picker or
  writing it without a provider infers Codex. Give it the display-only short alias `gpt53spark`, following the existing
  `gpt53`, `gpt55`, and `gpt56sol` convention; the explicitly qualified spelling remains `codex/gpt-5.3-codex-spark`.

## Implementation

1. Update builtin provider metadata.
   - In `src/sase/llm_provider/claude.py`, remove only the Opus 5 and Sonnet 5 point-version entries and their
     `opus5`/`sonnet5` short aliases. Leave `opus`, `sonnet`, `haiku`, `claude-haiku-4-5`, `claude-fable-5`, and the
     `_TIER_TO_MODEL` defaults unchanged.
   - In `src/sase/llm_provider/codex.py`, add `gpt-5.3-codex-spark` to the known-model list and map it to the
     `gpt53spark` short alias. No invocation or reasoning-effort branch is needed because Codex already passes arbitrary
     selected model ids through its existing invocation path.

2. Lock down registry and picker behavior with focused tests.
   - Update `tests/test_llm_provider_core.py` to assert that Spark implicitly resolves to Codex and exposes its short
     alias, while the two removed Claude point-version names no longer implicitly resolve or appear in the aggregated
     short-alias map. Retain coverage proving `opus` and `sonnet` still resolve to Claude.
   - Update `tests/test_model_picker_options.py` to assert that both Claude point-version rows are absent, Spark is
     present in the Codex group with `gpt53spark`, and the floating `opus`/`sonnet` plus the remaining Claude models are
     still present.
   - Add a real-registry regression in `tests/test_xprompt_model_completion.py` (without replacing its isolated metadata
     fixtures) that inspects concrete completion entries and proves the same include/exclude set reaches the `%model`
     completion catalog. This covers the second builtin picker and its Rust LSP payload source without adding a separate
     production filtering layer.

3. Bring user-facing documentation and intentional visual goldens in line.
   - In `docs/llms.md`, clarify that `opus` and `sonnet` are floating aliases resolved by Claude Code rather than SASE
     pins, update the automatic-provider table to remove the two Claude ids and add Spark under Codex, and update the
     shorthand table to remove `opus5`/`sonnet5` and document `gpt53spark`.
   - Run the focused Models-panel visual tests. The filtered `fable` picker snapshots render the Claude provider count
     from live registry metadata, so regenerate only the affected PNG goldens with `--sase-update-visual-snapshots` if
     the expected count changes, inspect the diffs, and rerun those tests without update mode. Do not rewrite the fixed
     fake rows in the model-completion rendering snapshot merely because one uses `claude-opus-5`; that fixture tests
     generic row rendering, not builtin catalog membership.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run focused unit tests for provider resolution/aliases, modal picker rows, and `%model` completion catalog behavior:
   `tests/test_llm_provider_core.py`, `tests/test_model_picker_options.py`, and
   `tests/test_xprompt_model_completion.py`.
3. Run the affected Models-panel visual snapshot cases in
   `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_navigation.py`, accepting only the intentional provider
   count changes and then verifying the updated goldens in normal comparison mode.
4. Run the required full `just check` gate and address any formatting, type-checking, unit-test, or visual-snapshot
   regressions before handoff.

## Acceptance Criteria

- Neither builtin picker offers `claude-opus-5` or `claude-sonnet-5`; both continue to offer `opus` and `sonnet`.
- Both builtin pickers offer `gpt-5.3-codex-spark` under Codex, and selecting it routes to the Codex provider.
- Claude tier defaults remain `opus` for large and `sonnet` for small; no point-version pin is introduced.
- Registry behavior, docs, and any affected visual goldens agree with the catalog, and `just check` passes.
