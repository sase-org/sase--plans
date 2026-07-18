---
tier: tale
title: Rebind the xprompt save panel snippet chord to Ctrl+X
goal: 'The prompt bar accepts Ctrl+G Ctrl+X as an alias for Ctrl+G x, and the resulting
  save panel uses Ctrl+X to switch from xprompt mode to snippet mode, so Ctrl+G Ctrl+X
  Ctrl+X completes the requested interaction without changing Ctrl+T completion elsewhere.

  '
create_time: 2026-07-18 08:19:21
status: done
prompt: 202607/prompts/ctrl_x_xprompt_snippet_chord.md
---

# Plan: Rebind the xprompt save panel snippet chord to Ctrl+X

## Context and intended behavior

The prompt input currently opens the unified xprompt/snippet save panel through `gx` in NORMAL mode or `Ctrl+G x`
through the prompt-local `Ctrl+G` prefix. Once the panel is open, `Ctrl+T` toggles between xprompt and snippet save
modes. Make `Ctrl+G Ctrl+X` a second INSERT/NORMAL `Ctrl+G` continuation for the existing save action, and move the
panel-local mode toggle from `Ctrl+T` to `Ctrl+X`.

Keep the scopes distinct:

- Preserve `gx` and `Ctrl+G x` as canonical entry points, and preserve `gX` / `Ctrl+G X` for saving the active pane as a
  frontmatter-local xprompt.
- Treat `Ctrl+X` as an alias only after `Ctrl+G`; do not claim a bare vim `g Ctrl+X` sequence or change NORMAL-mode
  numeric decrement behavior.
- Retire `Ctrl+T` only inside the unified save panel. Prompt-input `Ctrl+T` remains manual completion, and other panels
  retain their existing bindings.
- Do not add a configurable/default keymap entry: both affected interactions are existing hard-coded, prompt-local/modal
  bindings rather than registry keymaps.

## Unified save panel chord

Change `UnifiedXPromptSaveModal` so `Ctrl+X` invokes the existing mode-toggle action and the footer advertises
`^x snippet` instead of `^t snippet`. Account for the focused `UnifiedSaveInput` fields: Textual's base `Input` owns
`Ctrl+X` as cut, so intercept and consume the key at the panel boundary (or forward it from the panel's input subclass)
before the input can cut selected text. The chord must work consistently while the name field, description field, or
destination list has focus, while continuing to preserve the mode-specific name, destination, validation, preview, and
collision state already managed by `action_toggle_mode`.

Remove the panel-local `Ctrl+T` toggle rather than retaining it as an undocumented alias. This makes a `Ctrl+T` press in
this modal inert with respect to save mode and avoids conflicting guidance about which chord owns snippet mode.

## Prompt-local Ctrl+G alias and hints

Extend the declarative prompt `g`-prefix binding metadata so the save-as-xprompt action can expose `ctrl+x` as a
`via_ctrl_g`-only alias of canonical `x`. Reuse the existing action and availability guard instead of introducing a
second save path. Dispatch must continue to normalize real terminal control-byte events via the canonical event key,
clear the pending prefix/hint state, preserve the draft, and post exactly one `SaveAsXpromptRequested` message in both
INSERT and NORMAL prompt modes.

Keep dispatch and the which-key-style hint panel driven by the same metadata. On the `^G` surface, render the save
continuation so users can discover both `x` and `^X`; on the bare `g` surface, show only `x`. Generalize control-key
display formatting if needed so the new alias is rendered as `^X` without special-casing dispatch strings or duplicating
the action row.

## Help and documentation

Update the shared ACE help-modal Prompt Input entry and its assertions to list `Ctrl+G Ctrl+X` alongside `gx` and
`Ctrl+G x`. Update the prompt-input and prompt save documentation to describe the alias and the panel-local `Ctrl+X`
snippet toggle, including the direct `Ctrl+G Ctrl+X Ctrl+X` workflow. Leave every `Ctrl+T` completion reference intact
except the unified save panel's own footer or instructions.

## Regression coverage

Add focused behavioral coverage at the existing modal and prompt-widget test boundaries:

- Drive `Ctrl+X` while a unified save input is focused and verify it toggles to snippet mode instead of cutting input
  text; verify `Ctrl+T` no longer changes the modal mode and that toggling back still restores mode-specific fields.
- Cover `ctrl+x` control-byte normalization, alias dispatch only when `via_ctrl_g=True`, and the distinct bare-`g`
  versus `Ctrl+G` hint entries.
- Exercise `Ctrl+G Ctrl+X` through the real prompt key handler in INSERT and NORMAL modes, asserting one save request,
  unchanged draft content, cleared pending-prefix state, and preserved canonical `gx` / `Ctrl+G x` behavior.
- Add a cross-boundary regression that drives `Ctrl+G Ctrl+X Ctrl+X` from a prompt draft into the opened unified panel
  and lands in snippet mode, using deterministic save destinations so no external configuration is written.
- Update the unified-save PNG goldens whose footer changes, inspect the diffs to ensure the only visual change is the
  intended `^t` to `^x` hint, and retain both xprompt- and snippet-mode snapshots.

## Validation and completion

Run `just install` before repository checks. Run the targeted modal, prompt-prefix, save-capture, and help tests while
iterating, then run the dedicated unified-save visual snapshot test and accept only the intentional golden changes with
`--sase-update-visual-snapshots`. Re-run the visual test without update mode, inspect any `.pytest_cache/sase-visual/`
artifacts on failure, and finish with the required full `just check`.
