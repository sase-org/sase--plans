---
tier: tale
title: Share Vim search history across prompt panes
goal: 'A confirmed / or ? search can be repeated with n or N from any pane in the
  same prompt stack, including after stack rebuilds, while live search UI and match
  highlights remain local to the active pane.

  '
create_time: 2026-07-18 17:03:57
status: wip
prompt: 202607/prompts/shared_prompt_search_register.md
---

# Plan: Share Vim Search History Across Prompt Panes

## Context and intended behavior

`PromptInputBar` is one editing surface that can contain several `PromptTextArea` panes. Today each text area
initializes and owns its own `_last_search` tuple. Confirming `/pattern` or `?pattern` records the pattern and direction
only on that pane, so moving with `gk`/`gj` and pressing `n` or `N` produces `no previous search` in the newly focused
pane. Structural operations such as reorder, add, history load, external-editor return, or stash restore rebuild the
text-area widgets and discard the same state even when the `PromptInputBar` itself remains alive.

Treat the last confirmed pattern and its original direction as a search register owned by the `PromptInputBar`. All
panes in that bar share the register, and it survives the bar's internal stack rebuilds. Keep the scope to the lifetime
of that one prompt bar: a newly mounted prompt bar starts without a previous search, and unrelated search surfaces such
as the zoom-panel modal remain independent.

This is presentation-only Textual behavior. It belongs in the existing Python prompt widgets rather than the Rust core
boundary. The register is small, in-memory state; reading or updating it must stay synchronous and perform no I/O or
event-pump work on the keystroke path.

## Shared register and pane-local state

- Add a search-specific register API to `PromptInputBar`'s search mixin and initialize its optional `(query, direction)`
  value with the bar. The API should let a child record a successful confirmed search and retrieve the current record
  without reaching into unrelated bar state.
- Update `PromptSearchMixin` so confirmation writes the host bar's register and `n`/`N` reads it. Remove the pane-owned
  last-search source of truth. If a text area cannot resolve a host bar, repeat-search should retain the existing safe
  `no previous search` behavior instead of failing.
- Preserve all buffer-specific state on the `PromptTextArea`: whether the command line is active, its in-progress query
  and direction, origin cursor, current selection, match spans, and highlights. Existing focus/navigation cleanup must
  continue hiding the command line and clearing the old pane's highlights without clearing the shared register.
- Preserve the existing search semantics while changing ownership: only a non-empty confirmed query with a selected
  match updates the register; cancellation and unsuccessful searches do not replace it; `n` uses the recorded direction,
  `N` inverts it, counts and wrap feedback still work, and a shared pattern absent from the current pane reports
  `pattern not found` rather than `no previous search`.

## Regression coverage

Revise the interactive prompt-search tests to assert behavior through the whole prompt bar rather than inspecting a
pane-local `_last_search` field. Cover these cases:

1. Confirm a forward search in one pane, focus a different pane, and verify `n` finds the next occurrence there while
   the departed pane's transient highlights stay cleared and the search command line stays hidden.
2. Confirm a reverse search and verify cross-pane `n` follows that recorded direction while `N` reverses it.
3. Verify a confirmed register survives a stack rebuild (for example a pane reorder or add) and remains usable by the
   newly mounted active text area.
4. Verify cancellation or a failed candidate search does not overwrite an earlier confirmed register, and verify a
   target pane without the pattern yields `pattern not found` without losing the register.
5. Keep the existing single-pane incremental preview, smartcase matching, count, wrap, editing/highlight cleanup,
   app-binding isolation, and large-buffer overlay regressions passing.

Run the focused prompt-search widget test module during iteration, then run `just check` for the repository-wide
required verification. No visual styling or golden change is expected; investigate rather than accept a visual snapshot
change if one appears.

## Risks and guardrails

- Do not move live query or highlight state onto the bar; doing so could leak a stale selection into another buffer.
  Only the last successfully confirmed query and recorded direction are shared.
- Do not make search history application-global or persistent. Bar-local ownership fixes multi-pane navigation and
  widget rebuilds without coupling separate prompt sessions or zoom-panel search.
- Use direct synchronous child-to-host access so `Enter` immediately updates the register before a following navigation
  or repeat key; an asynchronous Textual message would introduce an avoidable ordering race.
