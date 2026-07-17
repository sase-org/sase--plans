---
tier: tale
title: Prompt-local word completion fallback
goal: 'Ctrl+T completes plain prose from words already present in the active prompt,
  filtered by the prefix immediately left of the cursor, without shadowing any existing
  completion provider.

  '
create_time: 2026-07-17 16:38:16
status: done
prompt: 202607/prompts/prompt_word_completion.md
---

# Plan: Prompt-local word completion fallback

## Context and behavior contract

`PromptTextArea._try_file_completion_tab()` is the manual `Ctrl+T` dispatcher for placeholder, VCS
project/ref/repository, Jinja, directive, xprompt/skill, xprompt-argument, path, and recent-file completion. Structured
contexts claim the key even when they currently have no matching rows, while whitespace keeps its existing recent-file
behavior. The new behavior belongs at the final plain- token fallthrough: only a non-empty prose-word prefix that no
existing provider claims may use words from the current `PromptTextArea.text` as candidates.

Define a prompt word as a maximal Unicode alphanumeric/underscore run, matching the widget's existing identifier-like
word semantics. Derive the filter from the run between the current word's start and the cursor, but replace the complete
word under the cursor so accepting in the middle of a word never duplicates its right-hand suffix. Scan every pane-local
line in the active prompt text, omit the occurrence being edited and candidates that would leave the word unchanged,
deduplicate exact spellings, filter case-insensitively while preserving the original spelling, and return rows in
deterministic case-insensitive lexical order. An empty prefix, no non-redundant matches, or a cursor outside a word is a
no-op. This is manual completion only; it does not become a live soft-completion source and does not change
whitespace/file-history behavior.

Keep the existing `Ctrl+T` interaction contract: one match is accepted immediately, multiple matches may extend a shared
prefix before opening the menu, `Ctrl+N`/`Ctrl+P` and arrows navigate, and `Enter`/`Ctrl+L` accepts. The word menu
should refresh from the current in-memory prompt after edits, preserve the selected spelling when possible, and dismiss
if the cursor loses its word prefix, candidates disappear, or a higher-priority structured context becomes active. All
work on this explicit keystroke path must remain a bounded in-memory scan—no filesystem access, subprocesses, provider
resolution, or shared-store locks.

## Pure prompt-word completion model

Add a focused pure-logic module alongside the existing completion engines. Give the new provider a named completion-kind
constant and a result type containing the left-of-cursor prefix, absolute replacement range, candidates, and shared
extension. Build candidates with the existing `CompletionCandidate` model so the common scrolling and selection
machinery remains reusable. Cover lexical boundaries and result construction independently from Textual: multiline text,
cursor-at-end and cursor-in-middle ranges, punctuation and underscore boundaries, Unicode words, case-insensitive
filtering with spelling preservation, deduplication/exclusion, deterministic ordering, shared-prefix calculation, and
empty/no-match inputs.

This helper stays in the Python TUI layer because it operates only on the currently mounted prompt widget's transient
text and produces presentation candidates; it introduces no shared repository/domain behavior that belongs in the Rust
core.

## Ctrl+T state-machine integration

Extend the completion open/context/accept/refresh mixins with an explicit prompt-word branch rather than letting the new
kind fall through to file-path logic:

- Invoke it only after every current provider has had the opportunity to claim the cursor context. Preserve the special
  empty-token branch that opens recent files and preserve "claimed but no rows" behavior for structured triggers.
- Use absolute offsets from the pure result for shared-prefix insertion and acceptance, then restore the cursor at the
  end of the replacement. Do not add prompt-word candidates to the recursive file-finder completion kinds.
- Recompute prompt-word results after typing, deletion, or cursor movement while the menu is active; keep the current
  selection by insertion text where possible and close cleanly when the fallback is no longer eligible.
- Continue using the existing common clear/reset path so submit, cancel, `Escape`, pane changes, and completion
  replacement cannot leave stale state.

Update stale comments/docstrings that describe `Ctrl+T` as file/path-only so the dispatcher is accurately documented in
code. No keymap or configuration default is added: this is the unconditional final fallback for the existing manual key.

## Completion panel and user documentation

Teach the shared prompt completion panel to recognize the new kind, render word rows as words rather than with file
icons, and use a clear title such as `prompt words`. Reuse the existing selection marker, ten-row scrolling cap, height
reservation, and navigation subtitle behavior.

Update the authoritative ACE keybinding/completion documentation to list prompt-local word completion after the
higher-priority providers and explain the left-of-cursor filter, whole-word replacement, and current-prompt-only scope.
Leave automatic completion settings unchanged and avoid rewriting the historical blog post or implying that the separate
Neovim integration gained this widget-specific fallback.

## Verification

Add focused widget tests that exercise `Ctrl+T` end to end for multiple buffer matches, immediate single-match
acceptance, mid-word whole-range replacement, multiline sources, navigation/acceptance, candidate refresh after edits,
and clean no-match dismissal. Assert the panel kind/title and non-file row rendering. Add precedence regressions showing
that representative structured tokens still use their existing provider and that whitespace still opens recent-file
history; the existing provider-specific suites and full test run cover the remainder of the dispatch chain. Add or
update a deterministic ACE PNG snapshot for the new word-completion panel so its title, rows, selection, and
prompt-stack height are pinned.

Before verification, run `just install` as required for an ephemeral workspace. Run the new pure/widget tests and the
affected completion/history/provider tests, run `just test-visual` for the snapshot, then finish with the
repository-required `just check` and resolve every lint, type, unit, and visual regression.
