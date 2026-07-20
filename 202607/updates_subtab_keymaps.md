---
tier: tale
title: Restore Updates sub-tab bracket navigation
goal: 'The SASE Admin Center Updates tab reliably handles [ and ] from every Updates
  sub-tab, and every Updates footer presents those keys with the same wording as the
  Projects tab.

  '
create_time: 2026-07-20 07:56:03
status: done
prompt: 202607/prompts/updates_subtab_keymaps.md
---

# Plan: Restore Updates sub-tab bracket navigation

## Context and root cause

The Updates pane already declares `left_square_bracket` and `right_square_bracket` bindings for cycling its Core,
Plugins, and Agent CLIs sub-tabs. The failure is focus ownership rather than a missing binding or a default-keymap
configuration problem. `PluginsBrowserPane` is not focusable, and its `focus_default()` method only focuses an option
list. Core has no option list, so opening the Admin Center directly on Updates/Core leaves focus on a widget in the
hidden Config pane. Textual resolves widget bindings through the focused widget's ancestor chain; because that chain
does not include the Updates pane, neither bracket binding is dispatched. The same gap can recur whenever Core is
activated without first establishing focus in another Updates sub-tab.

Current coverage masks the regression in two ways: the shared Updates test helper immediately switches from Core to
Plugins, and the sub-tab cycling test invokes action methods directly rather than sending keyboard events. The focused
filter-input test proves only that its explicit forwarding works.

The three Updates footers also render the navigation hint as `]/[ sub-tab`, while the Projects, Repos, and Workspaces
footers consistently render `[ / ] sub-tab`.

## Implementation

- Give the Updates pane a safe local focus fallback for Core while preserving browse-first option-list focus on Plugins
  and Agent CLIs. Entering Updates or switching to Core must always leave the focused widget inside the Updates pane so
  its existing bindings remain in Textual's active binding chain.
- Keep bracket handling synchronous and presentation-only: reuse the existing sub-tab cycling actions and do not add
  I/O, workers, refreshes, or global Admin Center routing to the keystroke path.
- Define one shared Updates sub-tab navigation hint and use the Projects wording exactly: `[ / ] sub-tab`. Apply it to
  the Core, Plugins, and Agent CLIs footers so the split rendering modules cannot drift apart again.
- Leave the configured/default keymap data unchanged because the key names and actions are already correct; this change
  repairs focus-based dispatch rather than remapping either key.

## Regression coverage

- Add an Admin Center interaction test that opens directly on Updates/Core, verifies focus is owned by the Updates pane,
  sends real `]` and `[` key events, and checks forward, reverse, and wraparound sub-tab transitions. Include
  transitions back to Core so the test protects the focus fallback, not merely the action methods.
- Retain the filter-input forwarding scenario and list-focused behavior for Plugins and Agent CLIs, ensuring the focus
  repair does not insert bracket characters into the filter or disturb browse-first selection.
- Assert that all three Updates hint builders contain `[ / ] sub-tab` and no longer emit the reversed compact wording.
- Run the focused Updates pane tests first. Then run the affected Admin Center PNG snapshot tests, inspect the rendered
  diffs, and update only snapshots whose expected footer text changed. Check narrow and standard layouts for clipping or
  unwanted focus styling.

## Validation

- Run `just install` before repository checks, as required for an ephemeral workspace.
- Run the targeted Updates interaction and visual snapshot suites while iterating.
- Run `just check` after all source, test, and intentional snapshot changes; investigate any failure rather than
  accepting unrelated visual differences.

## Risks and safeguards

- Making a fallback surface focusable could introduce visible focus styling or alter traversal. Exercise direct entry,
  mouse/tab switching, and both list sub-tabs, and rely on the Admin Center's existing priority Tab/Shift+Tab bindings
  to preserve main-tab navigation.
- Footer normalization affects several deterministic Updates screenshots. Review generated actual/expected/diff
  artifacts before accepting goldens and keep changes scoped to the deliberate hint text.
