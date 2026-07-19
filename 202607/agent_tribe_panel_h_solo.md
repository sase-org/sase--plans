---
tier: tale
title: Make H isolate the selected agent tribe panel
goal: 'Pressing H while an Agents-tab tribe panel is selected leaves that panel expanded
  and collapses every other tribe panel, without changing H behavior in other contexts.

  '
create_time: 2026-07-19 08:21:12
status: done
prompt: 202607/prompts/agent_tribe_panel_h_solo.md
---

# Plan: Make H isolate the selected agent tribe panel

## Context and behavioral contract

The Agents tab has three independent folding layers: whole tribe panels, grouping-strategy banners inside each panel,
and structural agent/clan/family/workflow folds. The uppercase `H` action currently reaches the grouping-strategy layer
and deliberately returns without doing anything when whole-panel focus is active. Change only that selected-panel case
so `H` becomes a “show only this panel” operation:

- If the selected tribe panel is collapsed, expand it.
- Collapse every other currently expanded tribe panel, leaving already-collapsed panels unchanged.
- Keep the chosen panel selected after the transition, including when expanding it changes the expanded/collapsed panel
  partition and therefore its rendered position.
- Treat every live split-panel key consistently, including the `(untagged)` panel represented by `None`.
- Preserve the chosen panel’s remembered row and all in-panel grouping, clan/family/workflow, and metadata fold state so
  entering the panel later restores the prior context.
- Make an already-isolated selected panel an idempotent no-op.

The new behavior applies only when `_resolve_focused_panel()` reports explicit whole-panel selection (whether the panel
is currently collapsed or expanded). When no tribe panel is selected, `H` must continue collapsing the current
grouping-strategy banner according to the active grouping mode. The existing higher-priority Tools detail routing and
the AXE and ChangeSpecs meanings of the same action remain unchanged. Merged-panel mode remains outside the
selected-tribe branch because it does not expose whole-panel tribe selection.

## Atomic selected-panel transition

Update the Agents folding action in `src/sase/ace/tui/actions/agents/_folding.py` to route selected whole-panel focus to
a dedicated isolate/solo transition before falling back to `_collapse_group_fold()`.

The transition should derive the desired whole-panel collapse set from the current `AgentPanelGroup`: remove the
selected key and add all other live panel keys. Apply only the actual differences, and use the existing panel-fold
persistence hooks for every changed key so both the pre-load intent journal and the coalesced background snapshot writer
continue to work. Do not introduce synchronous I/O or a new persistence path in the key handler.

Maintain the whole-panel focus invariants while applying the state change: set expanded-panel focus for the selected
target, clear stale group-banner and attempt selection state, invalidate the memoized panel/navigation data once, and
perform at most one list-changing Agents refresh when collapse state actually changed. Let the existing key-based panel
synchronization preserve the selected identity and recalculate deterministic layout order after the selected panel
becomes the sole expanded panel. Avoid descending into the panel or replacing its saved in-panel selection. If the
collapse set is already correct, retain focus without rebuilding the agent list or scheduling redundant persistence.

Keep the existing lower-level helpers usable by lowercase `h`/`l`, uppercase `L`, numeric fold selection, agent reveal,
and unread navigation. If a small shared helper is extracted to apply arbitrary whole-panel state, preserve those
callers’ current focus and refresh semantics rather than broadening this change into a fold-state rewrite.

## Discoverability and terminology

Update the user-facing descriptions of the uppercase action to describe both contextual meanings: isolate/show only the
selected tribe panel on Agents, collapse the current grouping-strategy group when an agent or banner is selected, and
retain the existing all-fold/Tools meanings elsewhere. Keep the key name and default `H` assignment unchanged.

Cover the canonical fallback binding metadata, configurable-keymap metadata/comment, command-palette label, Agents help
modal, and the dynamic keybinding footer. When whole-panel focus is active, surface an `H` footer chip such as “only
panel” so the new selected-panel behavior is visible alongside panel navigation and enter/collapse actions. Use “tribe”
or “panel” consistently with the surrounding UI, including the `(untagged)` tribe-summary panel.

Likely touchpoints are:

- `src/sase/default_config.yml`
- `src/sase/ace/tui/bindings.py`
- `src/sase/ace/tui/keymaps/types.py`
- `src/sase/ace/tui/commands/_app_metadata.py`
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py`
- `src/sase/ace/tui/widgets/_keybinding_bindings.py`

## Regression coverage

Extend `tests/ace/tui/test_agent_panel_collapse.py` around the existing split-panel harness to cover the state
transition directly:

- A collapsed selected tribe expands while every other live panel collapses; selection remains on the same key after
  panel reordering, whole-panel focus remains active, internal fold/selection memory is retained, and persistence
  records exactly the changed panel states.
- An expanded selected tribe collapses all other expanded panels while leaving previously collapsed panels alone.
- Repeating `H` when the selected panel is already the only expanded panel causes no extra refresh or persistence work.
- The `(untagged)` `None` key can be the selected survivor, guarding against confusing it with “no panel.”
- The operation does not mutate grouping-strategy registries, and the existing non-selected `H` tests in
  `tests/ace/tui/test_agent_fold_transitions.py` continue to prove the grouping-mode fallback.

Add focused discoverability assertions for the dynamic footer and any help/command metadata whose wording changes. Reuse
the existing fold-persistence tests unless implementation introduces a new batching primitive; if it does, add a
pre-load replay test proving that one isolate action wins over a late persisted baseline as one coherent final state.

## Verification

Run the focused behavioral tests first so failures identify the affected contract:

```bash
pytest tests/ace/tui/test_agent_panel_collapse.py \
  tests/ace/tui/test_agent_fold_transitions.py \
  tests/ace/tui/test_agent_fold_persistence.py \
  tests/ace/tui/widgets/test_keybinding_footer_tools_detail.py \
  tests/test_keymaps_display_help.py \
  tests/test_command_catalog.py
```

Because this is an ephemeral workspace, run `just install` before repository-wide validation, then run the mandatory
`just check`. No visual golden update should be necessary unless implementation changes rendering primitives rather than
only panel state and labels; if a visual snapshot changes unexpectedly, inspect the diff instead of accepting it
automatically.

Manually exercise the interaction in `sase ace` with at least three split tribe panels: select an expanded panel and
press `H`, select a collapsed panel and press `H`, repeat `H`, then enter the surviving panel and confirm its prior
grouping/workflow fold and row selection are intact. Also confirm `H` still collapses grouping banners when row/banner
focus is active and still controls Tools detail when the Tools panel owns the fold keys.
