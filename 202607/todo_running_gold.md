---
tier: tale
title: Align TODO highlighting with agent running gold
goal: 'TODO headers and their prompt count capsule use the exact gold shown by the
  Agents-tab RUNNING status, paired with a reliably readable foreground in dark and
  light themes without regressing prompt overlay behavior.

  '
create_time: 2026-07-22 09:18:23
status: done
---

- **PROMPT:** [202607/prompts/todo_running_gold.md](prompts/todo_running_gold.md)

# Plan: Align TODO highlighting with agent running gold

## Context and outcome

The Agents-tab row renderer currently presents `RUNNING` in bold `#FFD700`, while TODO styling independently derives its
header chip from the active Textual theme's warning color and blends that color with the canvas. That separate
derivation produces the pinkish-orange mismatch shown in the supplied screenshot. The TODO header palette is also reused
by the `TODO N` prompt-border capsule, so the visual contract needs to cover both surfaces.

Keep this as a presentation-only TUI change. TODO recognition, annotation counting, body-span boundaries, prompt-stack
aggregation, and agent lifecycle semantics remain unchanged. The result should make the exact shared gold obvious, use a
dark high-contrast foreground on that bright background, and retain a quieter theme-aware TODO body note.

## Shared color and contrast contract

- Add a named TUI-facing running-status color alongside the existing agent status presentation constants, and have the
  Agents-tab row renderer use it instead of its inline `#FFD700`. This makes the reference color an explicit semantic
  source without attempting a broad cleanup of every unrelated gold accent in ACE.
- Have TODO palette generation consume that same running-status color as the exact header-chip background. Remove
  warning/canvas inputs that no longer influence the chip rather than retaining misleading theme-derived plumbing.
- Derive the chip foreground from the fixed gold's maximum-contrast text. For `#FFD700`, Textual selects black, whose
  approximately 15:1 WCAG contrast ratio materially improves readability over the current light text. Keep the header
  bold.
- Continue deriving the italic, background-free TODO body note from the active theme foreground, but warm it toward the
  same shared gold instead of the theme warning color. Preserve sensible dark/light fallbacks so only the body note
  remains theme-sensitive.
- Route the prompt-border `TODO N` capsule through the revised palette helper, ensuring it receives the same exact gold
  background and contrast-safe text as inline TODO headers.

## Rendering behavior and documentation

- Preserve the existing overlay ordering and rendering safeguards: TODO syntax remains above Markdown/code/xprompt
  highlighting, while active search, yank, selection, and cursor chrome continue to win over the TODO background.
- Preserve theme re-registration for text areas so theme switches still update the body-note foreground; adjust
  count-capsule expectations to reflect that its shared gold/black pair is intentionally theme-invariant.
- Update the ACE prompt documentation to describe the exact Agents-tab running gold, high-contrast dark chip text, and
  the still-theme-aware warm italic body note. Avoid implying that the TODO chip background comes from the theme warning
  palette.

## Verification

- Extend the TODO palette/theme unit coverage to assert that both dark and light themes return the exact shared
  `#FFD700` chip background and contrast-derived black foreground, while producing appropriate theme-specific body-note
  colors.
- Add or refine Agents-tab row-rendering coverage so the `RUNNING` span is demonstrably styled from the named shared
  color. Keep the existing tests that verify TODO parsing, UTF-8 offsets, syntax coexistence, prompt-stack counts, and
  selection/search/yank/cursor precedence.
- Update only the intentional TODO PNG goldens (restored dark, restored light, and stacked prompts) after inspecting
  actual/expected/diff artifacts. Check that inline headers and the `TODO N` capsule are clearly readable in both themes
  and that inactive-pane rendering remains coherent.
- Run `just install` before repository checks, execute the focused TODO and agent-status tests during iteration, run
  `just test-visual` for the PNG suite, and finish with the required `just check`.

## Acceptance criteria

- Inline TODO headers and the `TODO N` capsule use the exact same `#FFD700` semantic color as the Agents-tab `RUNNING`
  status foreground.
- TODO chip text is dark, bold, and substantially more readable on the gold in both supported themes; the body note
  remains legible, italic, and free of a background fill.
- Theme switching, stacked/restored prompts, and all higher-priority overlays continue to work, and focused tests,
  visual snapshots, and `just check` pass.
