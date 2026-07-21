---
tier: tale
title: Highlight selected collapsed tribe title chrome
goal: 'A selected collapsed agent tribe panel uses the established yellow selection
  accent for its collapse arrow, total agent count, status-chip brackets, and status
  letters while preserving all semantic colors and expanded-panel behavior.

  '
create_time: 2026-07-21 07:57:35
status: done
prompt: 202607/prompts/selected_collapsed_tribe_title_highlight.md
---

# Plan: Highlight selected collapsed tribe title chrome

## Context

Agent tribe panels already expose collapsed versus expanded whole-panel focus through the Agents-tab selection model.
The focused widget also already receives the yellow border. However, the border-title refresh path reduces this state to
“selected expanded,” so a focused collapsed panel is rendered with unselected title chrome. This leaves the `▸` collapse
marker, total count, `[`/`]`, and status letters such as `R` and `W` gray even though the panel is selected.

The title formatter currently couples two related but distinct concepts: applying selected title chrome and adding the
expanded-panel `❖` selection marker. The implementation should separate those concepts so collapsed focus can receive
the accent without gaining an expanded-panel marker or changing the title's plain-text shape.

## Selected collapsed title behavior

Carry the focused panel identity and its collapsed/expanded state from the existing panel-focus resolver into the shared
border-title builder. Use the existing selected-title accent—the same color path used by selected expanded panels—rather
than introducing another yellow.

For a selected collapsed title such as `▸ ∴ @research · 5 [R3 D2]`:

- Render `▸`, the total `5`, both brackets, inter-metric spaces, and status letters (`R`, `D`, and the other supported
  letters) with the selected yellow chrome.
- Keep each status count digit in its existing semantic metric color, including multi-digit counts and the special
  unread presentation.
- Keep the configured tribe icon and identity label in their configured tribe color.
- Keep the `·` separator, jump/fold hints, and isolation marker behavior unchanged.
- Do not add `❖`; that marker remains exclusive to selected expanded panels.

Unselected collapsed panels must retain their current neutral chrome. Selected expanded panels must retain their
existing marker, selected total/chip chrome, semantic count colors, and configured identity colors. Merged `All agents`
mode and panel collapse/selection semantics are out of scope.

## Refresh integration

Implement the distinction in the centralized panel title construction path so every existing consumer receives the same
result: full panel rebuilds, affected-panel selective refreshes, title-only refreshes, and focused-panel navigation
refreshes. Avoid adding layout work, I/O, data recomputation, or a parallel repaint path; this is a pure Rich `Text`
styling change on the existing title refresh fast path.

## Regression coverage

Extend the focused title-format tests with a selected collapsed case that exercises every status letter and verifies:

- the title begins with `▸` and contains no `❖`;
- the arrow, total, brackets, spaces, and status letters resolve to the selected accent;
- metric count digits retain their existing per-status styles;
- the separator and configured tribe identity colors remain unchanged.

Extend panel display/navigation coverage to prove the title switches to selected-collapsed chrome when focus moves onto
a collapsed tribe, that the previously focused or other collapsed titles remain unselected, and that navigating back
restores the existing expanded selection presentation. This should exercise the shared state propagation rather than
only calling the formatter directly.

Update only the ACE PNG goldens whose selected collapsed title pixels intentionally change, including the collapsed
panel and fold-aware tribe views (and their hint variants when affected). Confirm visually that the result matches the
provided expanded-versus-collapsed reference: the collapsed title keeps its chevron form while the requested chrome
matches the established selected accent.

## Validation

Before verification, run `just install` as required for an ephemeral workspace. Then run the focused title and panel
display tests, regenerate and inspect the affected visual snapshots with the repository's visual-snapshot workflow,
rerun the visual suite without update mode, and finish with `just check`. The final diff should contain only the title
state/styling implementation, focused regression tests, and intentionally updated PNG goldens.
