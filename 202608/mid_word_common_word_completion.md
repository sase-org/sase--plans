---
tier: tale
title: Complete common words from the middle of a word
goal:
  Plain-word completion uses only left-of-cursor context and preserves right-hand text
  as a separately spaced suffix.
size: medium
proposed_by: bbugyi200.athena.03s
create_time: 2026-08-16 11:15:01
status: wip
---

# Complete common words from the middle of a word

## Goal

Make ACE's manual plain-word completion treat the cursor as the boundary of the typed
query. `Ctrl+T` at `foo<cursor>baz` must query with `foo`, preserve `baz`, and commit a
selected `foobar` as `foobar<cursor> baz`. Apply the same behavior to the prompt-local
provider and its prompt-history fallback without changing structured, path, placeholder,
or automatic soft-completion semantics.

## Current behavior

`prompt_word_completion.py` finds the complete identifier-like word surrounding the
cursor and exposes that whole range as `WordCompletionResult.replacement_start` /
`replacement_end`. Both the prompt-local builder and the history-word builder use the
right-hand suffix when excluding the current word. The direct-single-match and
shared-prefix paths in `_file_completion_tab.py`, plus Enter / Ctrl+L acceptance in
`_file_completion_accept.py`, replace that whole range. Consequently a completion in the
middle of a word consumes the suffix instead of retaining it as following text.
Prompt-local candidate extraction also scans the complete prompt, including text after
the cursor.

## Behavioral contract

- Derive the word prefix solely from the identifier-like run immediately to the left of
  the cursor; retain the existing Unicode alphanumeric, underscore, and ASCII-hyphen
  word-character rules.
- For prompt-local completion, source candidates only from complete words before the
  active prefix. Words later in the prompt must neither appear as candidates nor
  override the history-word fallback.
- For history-word completion, filter the warm MRU word list using only that left-side
  prefix. Do not use the suffix under the cursor to include or exclude candidates.
- Treat only the typed prefix as replaceable. If a committed completion preserves an
  identifier-like suffix immediately to its right, insert one ASCII space between the
  completion and that suffix and leave the cursor immediately after the completed word,
  before the new space. Do not add a trailing or duplicate space when there is no
  same-word suffix or an existing boundary already separates the following text.
- Apply the commit rule uniformly to a lone match accepted immediately by `Ctrl+T` and a
  highlighted row accepted by Enter or Ctrl+L, for both prompt-local and history-word
  completion.
- Shared-prefix extension is narrowing, not candidate commitment: extend only the
  left-side prefix, preserve the suffix, keep the menu active, and defer separator
  insertion until a candidate is actually committed.
- Preserve provider precedence, candidate ordering, minimum-length filtering,
  history-cache loading/deletion, selection preservation during refresh, hyphenated
  words, and end-of-word completion behavior.

## Implementation

1. Refine the pure word-completion context in
   `src/sase/ace/tui/widgets/prompt_word_completion.py`.
   - Continue using the full surrounding word only to detect whether a right-hand word
     suffix exists, but set the replacement boundary at the cursor so edits cannot
     consume the suffix.
   - Represent the suffix/separator requirement explicitly in `WordCompletionResult` (or
     an equivalent pure helper) so all acceptance paths share one unambiguous contract.
   - Restrict prompt-local candidate enumeration to complete word ranges before the
     active prefix, while retaining deduplication, case-insensitive ordering, candidate
     validation, minimum length, and shared-extension calculation.
   - Keep an exact-prefix spelling eligible when accepting it still has the meaningful
     effect of separating a preserved suffix; continue suppressing exact no-op matches
     at a word boundary.

2. Update `src/sase/ace/tui/widgets/history_word_completion.py` to build the same
   prefix-only replacement context and filter MRU words without consulting the
   right-hand suffix. Preserve MRU order, exact spellings, deduplication, and shared
   extension behavior, including the exact-prefix/no-op distinction above.

3. Centralize committed word insertion in the shared completion mixin path used by
   `src/sase/ace/tui/widgets/_file_completion_tab.py` and
   `src/sase/ace/tui/widgets/_file_completion_accept.py`.
   - Use the fresh `WordCompletionResult` to replace only the prefix.
   - Append the separator only when the result reports a preserved same-word suffix,
     then explicitly restore the cursor to the end of the chosen word (before that
     separator).
   - Route both local/history lone-match shortcuts and local/history Enter/Ctrl+L menu
     acceptance through this operation.
   - Leave shared-prefix replacement on the prefix-only range without applying the
     commit separator, then rebuild candidates from the updated cursor as today.

4. Replace the old whole-word assumptions and add regressions in
   `tests/ace/tui/widgets/test_prompt_word_completion.py` and
   `tests/ace/tui/widgets/test_history_word_completion.py`.
   - Pure tests should prove the replacement ends at the cursor, suffix text does not
     affect history filtering, prompt-local words to the right are excluded, words to
     the left remain eligible across lines, and end-of-word exact matches remain no-ops.
   - Widget tests should cover the requested `foo<cursor>baz` to `foobar<cursor> baz`
     flow, cursor placement, local and history providers, lone `Ctrl+T` acceptance,
     Enter and Ctrl+L acceptance, and hyphenated suffixes.
   - Cover multiple matches with a shared extension to prove narrowing preserves the
     suffix without inserting a space until final acceptance, and retain assertions for
     structured-provider precedence, cache refresh, and selection preservation.

5. Update the prompt-local and history-word completion descriptions in `docs/ace.md` to
   document left-of-cursor candidate context, preserved suffix separation, and the
   distinction between shared-prefix narrowing and committed selection. No configuration
   schema or default-keymap change is required because the feature keeps the existing
   `Ctrl+T`, Enter, and Ctrl+L bindings and settings.

## Verification

1. Run `just install` to refresh the workspace's editable development environment.
2. Run the focused widget suites:
   `pytest tests/ace/tui/widgets/test_prompt_word_completion.py tests/ace/tui/widgets/test_history_word_completion.py`.
3. Run `just check` for whole-repository lint gates and diff-scoped tests.
4. Manually review the focused assertions for both cursor states: end-of-word acceptance
   must not add trailing whitespace, while mid-word acceptance must preserve the suffix
   as a separate word with the cursor before the inserted space.
