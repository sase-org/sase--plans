---
tier: tale
title: Raise the ACE history-word completion default to 10,000
goal:
  ACE retains up to 10,000 prompt-history words by default, with runtime, schema, documentation, and tests kept in sync.
create_time: 2026-07-29 08:09:58
status: done
---

- **PROMPT:** [prompts/202607/raise_history_word_count_default.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/raise_history_word_count_default.md)
- **AGENTS:**
  - [bbugyi200.athena.ny--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ny.md#member-code)
  - [bbugyi200.athena.ny--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ny.md#member-plan)
- **COMMITS:**
  - [4ee5cd0](https://github.com/sase-org/sase/commit/4ee5cd092b45fe813c6e359f04f9248f8ff71c6a) — feat(ace): raise history word completion default

# Plan: Raise the ACE history-word completion default to 10,000

## Objective

Increase the default value of `ace.prompt_completion.history_word_count` from `1000` to `10000` everywhere that defines
or documents the default. This lets ACE retain a larger set of unique words derived from prompt history for manual
completion while preserving existing user overrides and the `0` disable behavior.

The adjacent `ace.prompt_completion.common_placeholder_count` setting is not part of this change. It governs the
separate durable store of literal `<placeholder>` tags and currently defaults to `100`; the requested old value of
`1000` uniquely identifies `history_word_count`.

## Current behavior

- `src/sase/default_config.yml` supplies `history_word_count: 1000` to the normal merged configuration.
- `PromptCompletionSettings` in `src/sase/ace/tui/widgets/prompt_completion.py` repeats `1000` as the defensive runtime
  fallback used when the completion section or value is absent or malformed.
- `src/sase/config/sase.schema.json` advertises `1000` as the public schema default.
- `docs/configuration.md` and `docs/ace.md` describe the same default.
- `tests/ace/tui/widgets/test_prompt_live_completion.py` locks in the malformed-value fallback.
- The history-word cache is already loaded off the Textual event loop, and its source token includes the configured
  limit. Raising the default therefore requires no cache-format migration or new synchronous TUI work.

## Implementation

1. Change the bundled `ace.prompt_completion.history_word_count` default in `src/sase/default_config.yml` from `1000` to
   `10000`.
2. Change `PromptCompletionSettings.history_word_count` in `src/sase/ace/tui/widgets/prompt_completion.py` to `10000` so
   empty and malformed raw settings use the same fallback as the bundled configuration.
3. Change the `history_word_count` default annotation in `src/sase/config/sase.schema.json` to `10000`, without changing
   its integer type, minimum of `0`, or disable semantics.
4. Update the prompt-completion example, defaults table, and history-word completion prose in `docs/configuration.md`
   and `docs/ace.md` so all user-facing references say `10000`.
5. Update focused regression coverage:
   - Change the malformed-setting fallback assertion in `tests/ace/tui/widgets/test_prompt_live_completion.py` to expect
     `10000`.
   - Add a targeted config-schema assertion that the bundled YAML value and public schema default for
     `history_word_count` are both `10000`, preventing these independently maintained contracts from drifting.

Do not mechanically replace unrelated `1000` values used as explicit test inputs, source-token fixtures, timestamps,
benchmark sizes, or other feature defaults. Do not change `common_placeholder_count`, history scanning/ranking, cache
scheduling, completion filtering, or user-provided overrides.

## Validation

1. Run the focused settings and schema tests:

   ```bash
   pytest -q \
     tests/ace/tui/widgets/test_prompt_live_completion.py \
     tests/test_config_schema.py
   ```

2. Search the relevant product and documentation surfaces for stale semantic references to the old default, reviewing
   each remaining `1000` occurrence rather than replacing unrelated fixtures:

   ```bash
   rg -n 'history_word_count|defaults: `1000`' \
     src/sase/default_config.yml \
     src/sase/config/sase.schema.json \
     src/sase/ace/tui/widgets/prompt_completion.py \
     docs/configuration.md \
     docs/ace.md \
     tests/ace/tui/widgets/test_prompt_live_completion.py \
     tests/test_config_schema.py
   ```

3. Because this changes files in the SASE repository, first refresh the workspace environment and then run the required
   full validation:

   ```bash
   just install
   just check
   ```

## Acceptance criteria

- A default merged configuration exposes `ace.prompt_completion.history_word_count` as `10000`.
- Missing or malformed prompt-completion settings fall back to `10000`.
- The public config schema advertises `10000`, and a regression test keeps the schema and bundled YAML defaults aligned.
- Documentation consistently identifies `10000` as the history-word default.
- Explicit values, including `0` and custom positive limits, retain their existing behavior.
- `common_placeholder_count` and the durable `<placeholder>` store are unchanged.
- Focused tests and `just check` pass.
