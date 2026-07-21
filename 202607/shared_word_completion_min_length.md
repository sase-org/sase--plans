---
tier: tale
title: Share the minimum word length across prompt completions
goal: 'Ctrl+T completes prompt-local and history words only when candidates meet one
  shared, clearly named configurable minimum length, defaulting to five.

  '
create_time: 2026-07-21 08:22:31
status: done
prompt: 202607/prompts/shared_word_completion_min_length.md
---

# Plan: Share the minimum word length across prompt completions

## Context and outcome

ACE's plain-prose `Ctrl+T` flow first searches words in the active prompt and then falls back to a cache of words from
prompt history. The history cache already excludes words shorter than `ace.prompt_completion.history_word_min_length`
(default `5`), but the newer prompt-local provider has no equivalent filter. As a result, short words in the current
pane can appear as candidates and can also prevent the history provider from taking over.

Make the threshold a property of word completion as a whole. Rename the public setting to
`ace.prompt_completion.word_min_length`, retain its default of `5` and its existing clamp to at least `1`, and use it
for both prompt-local candidates and history-word collection. The threshold applies to the full candidate word, not to
the prefix the user has typed: a short prefix may still complete a candidate whose total length meets the threshold.
Structured-token, path, file-history, and non-word completion providers remain unchanged.

## Configuration contract and documentation

- Rename the parsed `PromptCompletionSettings` attribute, bundled `src/sase/default_config.yml` key, and public JSON
  schema property from `history_word_min_length` to `word_min_length`. Describe it as the shared minimum for
  prompt-local and prompt-history word candidates, with an integer minimum of `1` and default `5`.
- Treat this as a clean public configuration rename: remove the history-specific spelling from the accepted schema and
  update parser coverage to establish the new canonical key, its default, invalid-value fallback, and lower-bound clamp.
  Because the old key was tied to only one provider, document the replacement so existing overrides have an explicit
  migration path rather than silently suggesting that it still controls runtime behavior.
- Update the configuration reference and ACE completion guide so their YAML examples, field tables, provider
  descriptions, and fully qualified setting references consistently use `word_min_length`. State that prompt-local words
  below the threshold are skipped before the history fallback is considered, while `history_word_count: 0` continues to
  disable only the history fallback.

## Completion behavior

- Extend the prompt-local word result construction in `src/sase/ace/tui/widgets/prompt_word_completion.py` to exclude
  candidate words shorter than the configured minimum while retaining the existing Unicode/underscore word boundaries,
  case-insensitive prefix matching, exact spelling, deduplication, ordering, complete-word replacement, and
  shared-prefix insertion behavior.
- Thread the parsed shared threshold through every prompt-local result computation in the manual completion lifecycle:
  initial `Ctrl+T` dispatch, recomputation after a shared extension, active-menu refresh, selection movement/display,
  acceptance, and the history-menu refresh that rechecks whether prompt-local completion has regained precedence.
  Centralize the threshold-aware call where practical so a stale or unfiltered candidate cannot be accepted through a
  secondary path.
- When all matching prompt-local words are too short, treat that as a normal local miss and continue into the existing
  history-word provider. Keep this work entirely in memory on the keystroke path; the history cache remains warmed and
  rebuilt off-thread, with the shared setting passed to its existing collection/source-token logic so config changes
  invalidate the cache as they do today.

## Tests and verification

- Add pure prompt-word regressions showing that the default/shared minimum excludes four-character candidates, includes
  five-character and longer candidates, still permits a shorter typed prefix, and honors a configured lower threshold.
  Retain coverage for Unicode, underscores, case folding, deduplication, shared extensions, and mid-word replacement.
- Add widget-level `Ctrl+T` coverage for the user-visible boundary and provider interaction: short local candidates do
  not open or auto-accept, an eligible history candidate is reached when local matches are below the threshold, and
  active-menu edits/refreshes cannot reintroduce undersized local candidates. Confirm that lowering `word_min_length`
  restores completion of shorter prompt-local words.
- Update settings, history-cache, and prompt-history derivation tests for the renamed attribute without changing MRU,
  count-limit, cold-cache, or off-thread behavior. Add schema coverage as needed to pin the new field/default and the
  removal of the old spelling; the repository-wide default-config/schema test must continue to pass.
- Before verification, run `just install` as required for an ephemeral SASE workspace. Run the focused prompt
  completion, history-word, cache, and config-schema test modules during development, then run the mandatory
  `just check` for the completed implementation. Inspect any visual snapshot failure rather than updating goldens unless
  the completion panel's intentional rendered output actually changes.

## Risks and boundaries

The main regression risk is enforcing the minimum only during initial menu construction while refresh or acceptance
paths rebuild an unfiltered result. Exercising every result-construction call site and the local-to-history handoff
guards against that drift. Filtering remains linear over the already in-memory prompt text and introduces no disk I/O,
subprocess work, or new event-loop waits. This change does not alter what constitutes a word, impose a minimum typed
prefix length, add automatic prose completion, or apply the threshold to structured/path candidates.
