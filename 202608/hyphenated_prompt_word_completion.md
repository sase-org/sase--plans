---
tier: tale
title: Complete hyphenated words from prompt history
goal:
  ACE Ctrl+T completion treats ASCII-hyphenated prompt words as one matchable and
  replaceable candidate.
size: small
proposed_by: bbugyi200.athena.02d
create_time: 2026-08-15 11:01:24
status: wip
---

# Hyphenated prompt-word completion

## Goal

Make the ACE prompt input's manual `<ctrl+t>` word fallback treat ASCII-hyphenated
identifier-like words as complete candidates. In particular, when a prior prompt
contains `bob-mac-capture`, invoking completion on the plain-prose prefix `bob-ma` must
offer or directly insert `bob-mac-capture`, rather than interpreting only `ma` as the
prefix and indexing `bob`, `mac`, and `capture` separately.

## Current behavior and constraints

- `src/sase/ace/tui/widgets/prompt_word_completion.py` owns the word-range helpers
  shared by prompt-local and prompt-history completion. Its current character rule
  accepts Unicode alphanumerics and `_`, but treats `-` as a boundary.
- `src/sase/history/prompt_words.py` uses those ranges to derive the MRU history-word
  cache. Consequently, `bob-mac-capture` is split while indexing, and
  `word_range_at_cursor()` resolves the screenshot's `bob-ma` prefix as only `ma`.
- Structured completions, artifact/file paths, xprompts, directives, and placeholders
  already claim their contexts before the plain-word fallback. Preserve that dispatch
  order and do not alter path/token completion.
- Preserve case-insensitive prefix matching, exact candidate spelling, MRU ordering,
  prompt-local precedence, whole-range replacement, shared-prefix insertion, the
  configured minimum candidate length, and history-word deletion semantics.
- Treat the ASCII hyphen (`-`) as the new identifier connector. Do not silently broaden
  the rule to Unicode dash punctuation. Ensure runs made only from hyphens, and
  digit-only runs merely joined by hyphens, do not become history candidates.

## Implementation

1. Update the shared word token/range semantics in
   `src/sase/ace/tui/widgets/prompt_word_completion.py` so ASCII hyphens remain inside
   the word range used for candidate discovery and for the cursor's replacement span.
   Keep underscores and Unicode alphanumerics working as today. Make candidate
   eligibility explicit enough that punctuation-only hyphen runs cannot appear when
   `word_min_length` is configured as low as `1`, and update docstrings to describe the
   expanded identifier-like spelling.
2. Adjust history extraction in `src/sase/history/prompt_words.py` to retain a full
   hyphenated spelling such as `bob-mac-capture` and apply `word_min_length` to that
   full candidate. Refine its useful-content check so adding `-` to the shared token
   alphabet does not admit `-----` or numeric-only forms such as `123-456`; preserve
   exact-spelling deduplication, MRU order, deletion filtering, and bounded loading. No
   cache migration or persisted format change is needed because the derived cache is
   memory-only and is rebuilt from prompt-history shards in each ACE process.
3. Extend `tests/ace/tui/widgets/test_prompt_word_completion.py` with focused range,
   matching, replacement, and `<ctrl+t>` coverage for hyphenated prompt-local words.
   Include a case that proves the entire hyphenated token is replaced when the cursor is
   in or at the end of it, while ordinary punctuation boundaries and existing
   underscore/Unicode behavior remain intact.
4. Extend `tests/history/test_prompt_words.py` to assert that history derivation emits
   `bob-mac-capture` as one candidate, measures the minimum length against the complete
   spelling, and rejects hyphen-only and numeric-with-hyphen noise without regressing
   MRU/deduplication behavior.
5. Extend `tests/ace/tui/widgets/test_history_word_completion.py` with the screenshot
   regression: seed history with `bob-mac-capture`, type a prompt ending in `bob-ma`,
   press `<ctrl+t>`, and verify the whole prefix becomes `bob-mac-capture`. Add a
   multiple-match or edit-refresh case so shared-prefix insertion and subsequent menu
   refresh continue to operate over the full hyphenated prefix, and verify acceptance
   replaces any right-hand suffix rather than leaving fragments behind.
6. Update the prompt-local and history-word completion descriptions in `docs/ace.md` to
   state that identifier-like candidates may contain ASCII hyphens and that the full
   hyphenated spelling is indexed, matched, length-filtered, and replaced as one word.
   Keep configuration names/defaults unchanged; this feature requires no keymap, schema,
   default-config, help-modal, or visual-snapshot changes.

## Verification

1. Run `just install` in the implementation workspace before project commands, as
   required for an ephemeral SASE workspace.
2. Run the focused non-visual tests:

   ```bash
   .venv/bin/pytest \
     tests/ace/tui/widgets/test_prompt_word_completion.py \
     tests/ace/tui/widgets/test_history_word_completion.py \
     tests/history/test_prompt_words.py
   ```

3. Run `just check` and resolve every lint, type-check, and diff-scoped test failure
   caused by the change. Use `just check-full` through `/sase_monitor` only if the
   scoped selector escalates or reports unusual selection, as required by the project
   verification policy.
4. Confirm the final diff contains only the shared token/extraction logic, focused
   tests, and completion documentation described above; no configuration or persisted
   history format should change.
