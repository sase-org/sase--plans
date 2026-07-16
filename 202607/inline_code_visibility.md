---
tier: tale
title: Make prompt inline code beautifully legible
goal: 'Inline code in the ACE prompt editor reads as a clear, polished literal chip
  in dark and light themes without overpowering surrounding prompt syntax.

  '
create_time: 2026-07-16 16:17:13
status: done
prompt: 202607/prompts/inline_code_visibility.md
---

# Plan: Make prompt inline code beautifully legible

## Context

The prompt editor already recognizes launch-inert inline-code ranges and paints the entire backtick-delimited span with
`codeblock.inline`, then overlays the opening and closing runs with `codeblock.delimiter`. In the supplied Flexoki
screenshot, that structure works but the current neutral background—only a 13% blend toward the foreground—is too close
to the prompt canvas. The code is technically marked yet does not scan as readily as xprompts, directives, or other
editor syntax. The light-theme fallback also treats a missing foreground as white, so its contrast behavior is not a
reliable inverse of the dark treatment.

This is a presentation change, not a parsing change. Inline-code recognition, literal-zone semantics, overlay ordering,
fenced-code cards, selection/cursor behavior, and prompt performance should remain intact.

## Visual design

Use a restrained, theme-aware **quiet chip** treatment:

- Increase the neutral surface contrast enough that a short literal such as `foo` or `/sase_gate` is recognizable at a
  glance, while keeping the chip subordinate to live green xprompts, amber directives, errors, selections, and search
  matches.
- Keep code content crisp and unitalicized at the normal prompt foreground; do not introduce a saturated semantic color
  or heavy bold treatment that could make inert literals look actionable.
- Keep the backtick runs visible as softer edges within the same continuous background. They should frame the literal
  without becoming the focal point.
- Derive the colors from the active application theme, with an explicit sensible foreground fallback based on whether
  the canvas is light or dark. Tune the neutral blend around the low-twenties-percent range (up from 13%) and judge the
  final value in both PNG snapshots rather than hard-coding a Flexoki-only color.
- Preserve a clear hierarchy: the inline chip is stronger than plain prose and the 8% fenced-code card surface, but
  quieter than selection, cursor, yank, and active-search backgrounds.

## Implementation

1. Refine the inline and delimiter style derivation in `src/sase/ace/tui/widgets/_codeblock_syntax_highlight.py`. Keep
   the existing full-span-plus-delimiter overlay model, but derive a more legible neutral chip background and a
   complementary delimiter foreground from the resolved theme colors. Isolate any light/dark fallback or blend
   calculation in a small helper if that makes the contrast rule explicit and directly testable.
2. Strengthen `tests/ace/tui/widgets/test_prompt_codeblock_highlight.py` so it verifies the design contract, not merely
   that a background exists: inline code and both delimiter runs still receive the expected spans; the chip is visually
   distinct from the prompt/fenced-card surface; delimiters retain the chip background while remaining lower-emphasis
   than the literal; and theme changes rebuild the derived inline styles correctly for both dark and light canvases.
   Keep the existing overlay-suppression and literal-semantics assertions intact.
3. Make the visual fixture in `tests/ace/tui/visual/test_ace_png_snapshots_prompt_stack.py` exercise realistic short and
   path/skill-like inline literals in prose. Confirm the treatment in solo and stacked prompt contexts, including
   inactive-pane dimming where the fixture covers it, and update only the four existing dark/light code-highlight PNG
   goldens after careful before/after inspection.

## Validation

- Run the focused prompt code-highlight unit tests.
- Run the dedicated prompt code-highlight PNG cases in both `textual-dark` and `textual-light`; inspect actual,
  expected, and diff artifacts before accepting intentional goldens. Verify that the Flexoki/default appearance remains
  tasteful against the supplied screenshot's dense prompt prose.
- Run `just install` first as required for an ephemeral SASE workspace, then run `just check` for the repository-wide
  required verification.

## Guardrails

- Do not change Markdown parsing, inline literal ranges, launch behavior, or overlay precedence.
- Do not restyle inline code in rendered `MarkdownBlock` surfaces; this plan is intentionally scoped to the editable
  prompt widget shown in the screenshot.
- Do not add user configuration for this single polish pass. Theme derivation should provide a consistent built-in
  result.
- Avoid custom per-line rendering for inline chips; the existing syntax-style overlay is sufficient and keeps the change
  off the prompt's hot render path.
