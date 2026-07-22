---
tier: tale
title: Jump from collapsed tribe panels with h
goal: 'Pressing h on a collapsed Agents-tab tribe panel selects the bottom-most expanded
  panel when one exists, preserving panel folds and navigation context while retaining
  the current warning when every live panel is collapsed.

  '
create_time: 2026-07-22 08:36:20
status: wip
---

- **PROMPT:** [202607/prompts/collapsed_tribe_panel_h_jump.md](prompts/collapsed_tribe_panel_h_jump.md)

# Plan: Jump from collapsed tribe panels with h

## Context and behavior contract

Agents-tab panels have a deterministic rendered order: expanded panels form the first partition, collapsed panels form
the second, and both partitions retain canonical `@default`/alphabetical order. Whole-panel focus is keyed by panel
identity even though a backing agent index is retained for rendering and later descent. Today, a lowercase `h` on a
focused collapsed panel stops at `"Panel is already collapsed"`.

Change only that terminal case. When at least one live panel is expanded, `h` on a collapsed panel should select the
last expanded panel in the current rendered `AgentPanelGroup.panel_keys` order—the visually bottom-most expanded panel.
The destination remains expanded and receives whole-panel focus; no panel is expanded, collapsed, or persisted as a side
effect. If every live panel is collapsed, keep the existing warning and no-op behavior. The ordinary `h` ladder (row to
workflow/family/clan/tribe), collapse of an already-selected expanded panel, merged-panel behavior, and `l`, `j`/`k`,
`J`/`K`, and `Z` semantics remain unchanged.

Treat the reserved `@default` panel key (`None`) as a valid destination rather than as a missing result. Determine
expanded state from the existing effective fold helpers so configured initial state and explicit expand/collapse intent
are both honored, and ignore stale fold intent for panels absent from the live rendered collection.

## Navigation and focus transition

In the Agents panel-navigation layer, add a read-only resolver for the last live expanded panel and a direct whole-panel
focus transition. Refactor the state change already used by selected-panel `j`/`k` navigation so both cyclic navigation
and the new `h` path share the same invariants:

- save the origin as an Agents jump anchor and clear forward history only when focus will actually move, allowing
  `Ctrl+O` to return to the collapsed origin;
- focus the destination by stable panel key/index and set explicit expanded whole-panel focus without descending into a
  row;
- clear stale grouping-banner and attempt focus;
- retain the destination panel's remembered row when it is still rendered, otherwise use its first rendered row as the
  backing anchor, so a subsequent `l` enters at the established location;
- refresh only old/new panel chrome, the footer, and the debounced detail view through the existing selective focus
  path—do not rebuild the panel list, mutate fold intent, persist a fold change, perform I/O, or introduce an async
  callback on the keystroke path.

Have the lowercase collapse dispatcher try this transition when its resolved whole-panel focus is already collapsed.
Suppress the toast when a destination was found; emit `"Panel is already collapsed"` only when the resolver finds no
expanded live panel. Keep the existing expanded-panel collapse path intact.

This is presentation-only TUI behavior and should remain in the Python navigation/action layer; it does not require a
Rust core API change.

## Discoverability and documentation

Expose the new action in the contextual Agents footer only when a collapsed focused panel has an expanded destination.
Pass a lightweight availability flag derived from the same resolver through the footer update/binding interfaces and
show an `h` label such as `last expanded panel` alongside the existing `l` expansion action. When all panels are
collapsed, omit that footer affordance and retain the warning fallback. Preserve existing binding ordering and
idempotent footer rendering.

Update the default keymap commentary and command/help description where necessary, plus the tribe-panel interaction
sections in `docs/ace.md` and `docs/agent_families.md`, to document that `h` from a collapsed panel selects the
bottom-most expanded panel and that the warning remains when none exists. No key or configuration value changes.

Because the contextual footer is visible in the collapsed-panel visual fixture, update only the affected intentional PNG
golden(s) after inspecting the actual/expected/diff artifacts. Do not accept unrelated renderer drift.

## Regression coverage

Extend the focused panel-collapse/navigation tests to cover:

- a collapsed panel jumping to the last expanded panel in rendered order rather than merely the preceding panel;
- skipping one or more collapsed siblings between the origin and the expanded partition;
- `@default`/`None` as the only expanded destination;
- all live panels collapsed, which preserves the current toast and makes no focus, refresh, history, or persistence
  mutation;
- preserved collapse/expand intent for every panel, no fold-persistence call, a selective (`list_changed=False`)
  repaint, a reversible jump anchor, and `l` descent to the target panel's remembered row;
- unchanged collapse behavior for selected expanded panels and unchanged merged-layout guards.

Add footer tests for both sides of the availability condition: a collapsed panel with an expanded destination advertises
`h`, while an all-collapsed collection does not. Update command-catalog expectations if the user-facing action
description changes. Add or extend an `AcePage` interaction assertion around the collapsed-panel fixture so a real
keypress verifies the focused panel key and selected whole-panel state; rely on existing selected-panel styling and only
refresh a visual snapshot whose footer genuinely changed.

## Validation

Before testing in the implementation workspace, run `just install` as required for an ephemeral SASE checkout. Then:

1. Run focused tests for `tests/ace/tui/test_agent_panel_collapse.py`, footer binding coverage, command-catalog
   coverage, and the affected collapsed-panel `AcePage`/visual test.
2. Run `just test-visual` and inspect any PNG diff artifacts before accepting the narrowly intentional footer update.
3. Run `just check` to cover formatting, Ruff, mypy, Symvision, repository validation, the full test suite, and visual
   snapshots.

Acceptance requires the rendered-order target, focus/history/remembered-selection behavior, no-target warning, footer
availability, documentation, and all regression checks to agree without adding blocking work to the TUI event loop.
