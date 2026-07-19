---
tier: tale
title: Make agent jumps descend from selected tribe panels
goal: 'The ,j and ,J leader keymaps select the correct unread-completed or stopped
  agent when whole-panel tribe focus is active, while preserving jump order, acknowledgement
  semantics, folds, and navigation history.

  '
create_time: 2026-07-19 08:24:22
status: wip
prompt: 202607/prompts/agent_tribe_jump_keymaps.md
---

# Plan: Make agent jumps descend from selected tribe panels

## Context

The leader bindings and their default configuration already dispatch `,j` to the unread-completed jump and `,J` to the
stopped-agent jump. The failure is in the shared time-ordered agent navigation state: an expanded tribe panel keeps a
remembered backing `current_idx`, but whole-panel focus deliberately means that no agent row is selected. The jump
helper currently treats that backing index as the active row and leaves expanded-panel focus set after choosing a
target, so a command can report success while the UI remains on the tribe summary. Collapsed tribe panels follow a
separate expansion path and already descend to an agent.

This is a presentation/navigation-state fix in the Python TUI. It does not change key names, configurable defaults,
command availability, status classification, or shared backend behavior.

## Navigation state contract

Update the shared unread/stopped time-ordered navigation in `src/sase/ace/tui/actions/agents/_unread_navigation.py` so
whole-panel focus is an explicit jump origin rather than an implicit agent-row origin.

- When a tribe panel is selected, start at the newest eligible candidate even if the panel's remembered backing index
  happens to name an eligible agent.
- Treat descending from the panel to a row as a real focus change, including when the target row and panel match the
  remembered backing state. Save the tribe-panel anchor before moving so existing back/forward navigation can restore
  whole-panel focus.
- Clear explicit expanded-panel focus before applying the target row in both the direct visible-row path and the unread
  jump path that reveals a member hidden by a collapsed clan. Continue clearing banner focus and pinned attempt state as
  existing row jumps do.
- Route the panel-to-row transition through the existing selective refresh and unread repaint paths. Keep `,j`
  acknowledgement/manual-unread behavior and `,J` non-acknowledging behavior unchanged, and add no I/O, subprocess work,
  or asynchronous work to the keystroke path.

## Regression coverage

Extend `tests/ace/tui/_agent_unread_navigation_helpers.py` with the minimal whole-panel selection state needed by its
production mixins, then add focused coverage in the unread, stopped, and folded-navigation test modules.

- Exercise the real leader dispatch for both `,j` and `,J`, proving that each command leaves tribe focus, selects the
  newest eligible row rather than cycling from the remembered backing row, resets any pinned attempt, and records a
  panel anchor that can be restored.
- Assert the semantic difference between the commands: `,j` acknowledges the selected unread completion (subject to the
  existing manual-unread guard), while `,J` leaves unread state untouched.
- Cover the unread collapsed-clan reveal branch from expanded tribe focus so structural refiltering also lands on the
  target agent and preserves the tribe origin in jump history.
- Retain the existing collapsed-panel, cross-panel, no-candidate, search-filter, fold, recency/wrap, and repeat-command
  tests as regression protection. Where the selected expanded tribe-panel fixture is already established, add a Textual
  interaction assertion using the literal `comma` plus `j` sequence to verify that dispatch, panel chrome, and
  selected-row state agree without introducing a new visual golden solely for this state fix.

## Verification

Run the focused unread, stopped, fold, panel-selection, leader-dispatch, and any updated Textual interaction tests. Then
run `just check` for the repository-wide formatting, lint, type-check, test, and snapshot gates. Confirm the working
tree contains only the intended navigation and test changes.
