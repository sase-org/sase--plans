---
tier: tale
title: Preserve the prompt editor row beneath long completions
goal:
  Keep at least one visible prompt-editor row whenever a completion panel contains long
  descriptions or the terminal is resized.
size: small
proposed_by: bbugyi200.athena.0ac
create_time: 2026-08-22 10:52:35
status: wip
---

# Plan: Preserve the prompt editor row beneath long completions

## Diagnosis

`PromptInputBarCompletionMixin` renders the shared completion panel into an auto-height
`Static`, but reserves bar height from `_content_line_count(content)`, which counts only
explicit newline-delimited rows. Rich/Textual may wrap any one of those logical rows
into multiple visual rows. A long skill description such as the `/sase_monitor` row
therefore makes the actual panel taller than `_completion_line_count` reports.

`PromptInputBarStackLifecycleMixin._update_height()` trusts that undercount when it
sizes the bottom-docked prompt bar. With a one-row prompt, the panel consumes the
unreserved space and Textual lays out `#prompt-stack` at or below the bar's bottom
border, so the editor exists with a one-row region but is outside the visible content
box. The existing height tests use short, non-wrapping candidates, so their assertion
that reserved and rendered panel heights agree does not exercise this failure.

A live layout probe with the reported skill description measured the mismatch:

- At 220 columns, four rows were reserved while the panel plus margin occupied five; the
  editor began exactly at the screen bottom.
- At 120 columns, four rows were reserved while the panel plus margin occupied seven;
  the editor began two rows below the screen.

## Implementation

1. In `src/sase/ace/tui/styles.tcss`, make `#prompt-completion` content explicitly
   single-line per logical row with Textual's `text-wrap: nowrap` and an ellipsis
   overflow policy. This matches the existing completion contract: the renderer emits
   one explicit newline per candidate (plus explicit group, overflow, or argument-hint
   rows), and `completion_visible_rows()` budgets those rows against the panel's content
   capacity. Long descriptions remain available in the completion metadata but cannot
   silently allocate extra layout rows.

2. Keep the policy on the shared panel selector so ordinary completions, xprompt
   argument hints, and Jinja diagnostics all obey the same explicit-row height
   accounting. Document beside the CSS/Python mirror that wrapping must stay disabled
   for `_content_line_count()` and `_reserved_panel_rows()` to remain authoritative; do
   not add per-provider truncation branches or width-dependent asynchronous work.

3. Preserve the current candidate-row budget and cap semantics: do not increase
   `COMPLETION_PANEL_MAX_HEIGHT`, change selection/scroll-window behavior, or remove the
   prompt stack's one-row minimum. After layout, the reserved panel rows must equal the
   rendered panel border-box height plus its bottom margin, and the active editor must
   remain inside the prompt bar's visible content region.

## Regression coverage

1. Extend `tests/ace/tui/widgets/test_prompt_completion_height.py` with a real
   xprompt/skill completion candidate carrying a long, single-logical-line description.
   Assert at wide and narrow terminal widths that:
   - the rendered description exceeds the available width but remains a single
     ellipsized visual row;
   - the recorded reservation matches the panel's rendered height plus margin;
   - the prompt bar includes the panel, its border, and at least one editor row; and
   - the active `PromptTextArea` is contained by the bar/screen instead of starting at
     or below the visible bottom edge.

2. Exercise a width change while that completion remains open and assert the long
   logical row stays one visual row, the reservation remains synchronized, and the
   editor stays visible. Retain the existing short-menu and max-height tests to prove
   the fix does not alter ordinary or capped cases.

3. Add or extend an ACE PNG snapshot fixture for a one-line prompt with the long skill
   completion open, pinning both the ellipsized menu and the still-visible editor row.
   Accept only the intentional prompt-bar height/golden change, then rerun that visual
   case without snapshot-update mode.

## Verification

Run the focused prompt completion-height and resize tests first. Run the affected ACE
PNG snapshot case, inspect its generated actual/expected/diff artifacts if it fails, and
re-run it after accepting the intentional golden. Finish with `just check` so all
whole-repository lint gates and the diff-scoped test lane pass. If scoped selection
escalates or reports unusual coverage, use the project-prescribed monitored
`just check-full` workflow.
