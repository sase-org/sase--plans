---
tier: tale
title: Visual-mode surround keymap for prompt input
goal: 'Prompt input users can press S in characterwise or linewise Visual mode and
  choose a delimiter to surround the active selection with predictable Vim state transitions,
  cancellation, undo, and dot-repeat behavior.

  '
create_time: 2026-07-21 16:07:13
status: wip
---

- **PROMPT:** [202607/prompts/visual_surround_keymap.md](prompts/visual_surround_keymap.md)

# Plan: Add visual-mode `S` surround support

## Context and tier choice

The reusable `VimTextArea` editing layer already gives `PromptTextArea` Vim Normal, Visual, and V-Line modes. Normal
mode supports the vim-surround-inspired `ys{motion}{delimiter}` / `yss{delimiter}` flow by resolving a motion into a
pending range, mapping the final key through the shared delimiter table, and applying one replacement. Visual mode has
selection math and structured mutation replay, but uppercase `S` is currently unhandled there; lowercase visual `s`
changes the selection, and Normal-mode uppercase `S` changes whole lines.

This is a `tale` because the work is one cohesive extension to the existing Vim widget tower and its focused tests. It
does not need independently landable phases or cross-project coordination. The behavior belongs in the Python TUI
editing layer: no Rust core API, configurable app-keymap entry, or `default_config.yml` change is needed for an
intrinsic modal-editing command.

## User-visible behavior

- In characterwise Visual mode, `S` starts a surround operation for the normalized active selection. The next literal
  delimiter key wraps that selection and returns to Normal mode.
- In V-Line mode, `S` targets all selected line contents from the start of the first selected row through the end of the
  last selected row. It must not consume or join the newline before or after the selection, so neighboring lines remain
  unchanged.
- Reuse the established Normal-mode surround vocabulary and pairing rules: quotes and backticks pair with themselves;
  opening or closing bracket keys produce their matching pair; `b` / `B` aliases keep working; and an otherwise valid
  single character is a same-character custom surround.
- Visual selection boundaries are authoritative. Characterwise `S` surrounds the selected bytes, including explicitly
  selected leading/trailing whitespace or embedded newlines, instead of applying the `ys` motion-specific whitespace
  trimming used to keep a word motion's trailing separator outside quotes.
- While waiting for the delimiter, keep the Visual/V-Line selection and show `S` as the pending indicator in both the
  prompt bar and the standalone `VimTextArea` border. A delimiter completes one edit, clears the pending/visual state,
  and places the Normal cursor on the inserted opening delimiter.
- `Escape` while `S` is pending cancels without editing and returns to Normal mode. A non-literal continuation also
  clears the pending target cleanly rather than inserting a textual key name or leaving the editor stuck.
- A successful surround is a single undoable edit, does not overwrite the unnamed Vim register or system clipboard, and
  becomes the latest dot-repeat mutation. A canceled, invalid, or empty-range attempt must not replace the prior
  repeatable change.
- Dot repeat reapplies the chosen delimiter to the same visual shape from the current Normal cursor: the same character
  count for characterwise Visual mode or the same row count for V-Line mode. Preserve the existing visual-repeat count
  override semantics, where a count before `.` scales that saved shape.
- Preserve the meanings of Normal-mode `S` (linewise change) and visual lowercase `s` (change selection and enter Insert
  mode).

## Editing-state and surround design

1. Add uppercase `S` dispatch to the visual key handler before the existing lowercase `s` change branch. Resolve and
   snapshot the current characterwise exclusive range or V-Line row range, reject a truly empty target without
   disturbing prior repeat state, and enter a dedicated pending visual-surround state.

2. Share delimiter resolution and the actual replacement transaction with Normal-mode `ys`, but make the target's
   boundary policy explicit. Normal motion surrounds retain their current whitespace-aware behavior; a Visual target
   uses its exact normalized range. Keep the one-character validation, paired-delimiter aliases, read-only toggling,
   cursor calculation, and mutation cleanup in one surround implementation so the two entry points cannot drift.

3. Extend visual pending-key handling to consume the delimiter as data, including a literal space when Textual exposes
   it through the space key event. Apply the saved target only after a valid delimiter is resolved. On completion or
   cancellation, clear the range, pending count/key state, selection anchors, and mode indicator in a defined order so
   no stale Visual state survives the transition to Normal mode.

4. Extend the structured visual mutation record with the surround delimiter (or equivalent replacement payload) in
   addition to operation, selection kind, and selection size. Teach visual dot replay to reconstruct the saved character
   or row range and invoke the shared surround transaction immediately, without reopening a pending prompt. Preserve the
   existing delete/change/case/indent repeat paths and their state types while adding this payload-bearing operation.

5. Keep the implementation in the current responsibility modules:
   - visual command dispatch and counts in `_vim_visual_keys.py`;
   - visual selection/application and mutation recording in `_vim_visual_ops.py` / `_vim_visual_state.py`;
   - delimiter continuation handling in `_vim_visual_pending.py`;
   - shared delimiter mapping and surround edit mechanics in `_vim_normal_surround.py` (renaming private helpers only if
     needed to reflect their now-shared use);
   - Vim state initialization/type declarations and structured dot replay in `vim_text_area.py` and
     `_vim_normal_state.py`.

## Test coverage

Add focused behavioral coverage, preferably in a new `tests/test_prompt_visual_mode_surround.py`. Use `PromptPage` for
editing behavior so events traverse the real prompt text-area dispatch, and use the existing `VimEditorPage` /
`PromptInputBar` host harnesses for chrome that `PromptPage` does not mount:

- characterwise `v...S"` wraps a forward selection and lands in Normal mode on the opening quote;
- a backward selection normalizes correctly, and an explicitly selected boundary space stays inside the surround;
- closing bracket keys, a literal space, and at least one custom same-character delimiter reuse the existing delimiter
  rules;
- characterwise selection spanning a newline wraps only the selected range;
- `V...S)` wraps multiple complete lines without changing neighboring line separators;
- the pending `S` state is visible before the delimiter and is fully cleared after success;
- `Escape`, a non-literal continuation, and an empty selection are no-ops that leave no pending state and preserve the
  previous dot-repeat mutation;
- one `u` reverses the whole surround, and the unnamed register remains unchanged;
- `.` repeats both characterwise and linewise surrounds over the saved shape, including the existing count-override
  behavior;
- visual lowercase `s` and Normal uppercase `S` retain their current change semantics.

Pin the pending indicator with a small bare-`VimTextArea` border-subtitle assertion and a mounted `PromptInputBar`
subtitle assertion, without adding a PNG golden for this transient state. Retain and run the Normal-mode surround suite
as regression coverage, especially word-motion whitespace placement, pair aliases, custom delimiters, cancellation, and
`ys` dot repeat. `PromptTextArea` should inherit the implementation rather than carry a prompt-only fork.

## Validation and completion checks

After implementation:

1. Run `just install` first, as required for an ephemeral workspace.
2. Run the focused suites:
   `pytest tests/test_prompt_visual_mode_surround.py tests/test_prompt_visual_mode.py tests/test_prompt_normal_mode_surround.py tests/test_prompt_normal_mode_small_commands.py`.
3. Run `just check` before handoff, because source/test files changed. Investigate any visual snapshot failure rather
   than accepting goldens automatically; this feature changes only a transient pending subtitle and should not require a
   standing layout/golden update unless a test intentionally captures that state.

## Risks and guardrails

- The existing Normal surround state is cleared by `_enter_normal_mode()`. Do not transition out of Visual mode before
  the saved range is applied (or preserve it explicitly), or `S` will silently lose its target.
- Visual ranges are inclusive at the active cursor but represented internally with an exclusive end. Always use the
  existing normalization helpers so forward, backward, end-of-line, and multi-line selections do not gain or lose a
  character.
- Do not record the repeat mutation until a real replacement succeeds; otherwise cancellation can clobber a useful prior
  `.` action.
- Keep surround application as one replacement so Textual undo, prompt change notifications, and cursor restoration
  observe one coherent edit.
