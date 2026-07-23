---
tier: tale
title: Continue prompt bullets with NORMAL-mode uppercase O
goal: 'Make prompt NORMAL-mode O open a correctly indented sibling hyphen bullet above
  the current physical line when that line belongs to a supported bullet, while preserving
  ordinary Vim open-above, undo, and dot-repeat behavior.

  '
create_time: 2026-07-23 10:46:12
status: done
---

- **PROMPT:** [202607/prompts/prompt_upper_o_auto_bullets.md](prompts/prompt_upper_o_auto_bullets.md)

# Plan: Continue prompt bullets with NORMAL-mode uppercase `O`

## Context and outcome

`PromptTextArea` already uses the shared `prompt_bullet_sibling_prefix()` ownership helper for INSERT-mode `Ctrl+J` and
NORMAL-mode lowercase `o`. Those paths recognize direct and Prettier-wrapped space-indented `- ` bullets, reuse the
owning bullet's indentation, and fall back to a bare newline outside a supported bullet. Uppercase `O` still follows the
base `VimTextArea` implementation unconditionally, inserting only a blank line above the cursor's current physical line.

Extend the prompt-only behavior so uppercase `O` evaluates the current physical line against the same pre-edit ownership
rules and, when owned, inserts `<indent>- ` on the new line immediately above it. Leave the cursor after the marker in
INSERT mode so typing begins as bullet content. Direct top-level bullets, direct nested bullets, and physical
continuation lines must all use the owning bullet's sibling indentation. Lines rejected by the existing helper—ordinary
prose, blank/dedented boundaries, fenced content, thematic breaks, unsupported markers, tight dashes, and tab-indented
content—must retain ordinary bare open-above behavior.

This is deliberately prompt-local. A bare `VimTextArea` must continue to insert a blank line for `O`, and
`SingleLineVimTextArea` must continue suppressing `o`, `O`, and `Ctrl+J`. Do not change bullet ownership semantics,
keymap configuration, the Rust core, or any asynchronous/UI-refresh behavior.

## Editing and extension-point design

Add a base Vim open-above customization seam parallel to `_normal_open_below_insert_text()` instead of branching on
`PromptTextArea` inside the shared NORMAL-mode dispatcher. The base hook should return the existing bare open-above
structure. Override it in the prompt action mixin to reuse `prompt_bullet_sibling_prefix()` and produce the marker
before the newline, since uppercase `O` inserts before the original row whereas lowercase `o` appends after it.

Update the shared uppercase-`O` action to:

1. resolve its structural insertion from the original cursor row before mutating the document;
2. perform the newline plus optional marker as one keyboard replacement/undo checkpoint;
3. leave the newly opened row selected with the cursor immediately after the optional prefix;
4. enter INSERT mode and begin insert-mutation capture only after that structural prefix, so the prefix is not treated
   as user-typed dot-repeat text; and
5. preserve the existing count, cursor, read-only transition, and mutation-recording behavior outside the new hook.

Keep the keystroke path pure and bounded: it should inspect only the in-memory document and call the existing helper and
replacement machinery. It must not introduce disk access, subprocesses, awaited callbacks, new refreshes, or duplicated
Markdown parsing.

## Undo and dot-repeat contract

Preserve the established two-part edit history used by prompt lowercase `o`: after `O`, typing content, and leaving
INSERT mode, the first `u` removes the typed content while retaining the generated marker, and the next `u` removes the
structural open-above edit. The marker and newline must be inserted atomically so there is no intermediate undo state
containing only one of them.

Teach dot replay to treat uppercase `O` as a structural open-line command just as it treats lowercase `o`. Replaying
`O...Escape` must recompute ownership at the destination before opening the line, rather than copying indentation or a
marker from the source location. Add the direction-specific base replay-normalization hook (or a clean generalized
equivalent) and have the prompt override reuse `normalize_prompt_bullet_replay_text()`. This prevents a duplicated
marker when the original mutation was recorded on prose by manually typing `- ` but replay lands in a prompt context
that now supplies the marker. Conversely, a mutation recorded inside a bullet should open a plain line without a marker
when replayed on prose, because only user-typed content—not source structural context—is repeated.

Retain all existing lowercase-`o` replay behavior. If the hooks are generalized, keep inert identity/bare-newline
defaults for non-prompt `VimTextArea` consumers.

## Tests and documentation

Expand `tests/ace/tui/widgets/test_prompt_bullet_editing.py` so its prompt open-line coverage includes uppercase `O`.
Replace the current assertion that prompt uppercase `O` remains bare with parameterized acceptance cases for:

- a direct top-level bullet;
- a Prettier-wrapped top-level continuation;
- a direct nested bullet;
- a wrapped nested continuation;
- ordinary prose retaining a bare line;
- exact text, cursor-after-prefix placement, and INSERT mode for every case.

Add focused regressions showing that uppercase `O` keeps the structural marker as its own undo checkpoint and that dot
repeat re-evaluates the destination. Cover both context changes that matter: bullet to prose must not leak the generated
marker, while prose with a manually typed `- ` replayed into a nested/wrapped bullet must not produce two markers.

Add or extend a bare-`VimTextArea` regression proving uppercase `O` remains a blank open-above operation, alongside the
existing lowercase isolation test. Continue running the existing `SingleLineVimTextArea` suppression test to guard the
subclass boundary. Keep the pure ownership matrix unchanged unless implementation reveals a missing ownership case;
uppercase `O` should consume the same helper contract rather than define new rules.

Update `docs/ace.md` so the NORMAL-mode command table and prompt bullet explanation state that both `o` and `O`
auto-continue containing hyphen bullets in prompt panes, with their normal below/above directions, while non-bullet
contexts remain bare. Adjust nearby INSERT-mode wording so `Ctrl+J`, `o`, and `O` describe one shared ownership rule
without implying the behavior applies to generic Vim text areas.

## Validation and handoff

Before handoff, run `just install`, then the focused prompt bullet-editing, NORMAL-mode dot-repeat, and single-line
widget tests. Run the repository-required `just check` last, including the visual snapshot suite. No visual output is
expected to change; inspect any snapshot failure rather than accepting new goldens for this behavior-only edit. Finish
with a worktree/diff review confirming that changes are limited to the shared Vim extension seam and replay handling,
the prompt override, focused tests, and documentation.
