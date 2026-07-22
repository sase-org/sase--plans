---
tier: tale
title: Move tribe-panel isolation from H to Z
goal: 'Make the Agents-tab zoom action isolate or restore a selected tribe panel,
  while preserving ordinary detail-panel zoom and returning H to collapse-only behavior
  everywhere else.

  '
create_time: 2026-07-22 06:57:41
status: done
---

- **PROMPT:** [202607/prompts/tribe_panel_zoom_keymap.md](prompts/tribe_panel_zoom_keymap.md)

# Plan: Move tribe-panel isolation from H to Z

## Context

The Agents tab has two mutually exclusive selection contexts. A row selection has a concrete agent and uppercase `Z`
opens the dominant detail panel in the zoom modal. Whole-panel focus instead selects a split-layout tribe container, so
there is deliberately no selected agent and the zoom modal cannot open. Today uppercase `H` detects that whole-panel
context before its normal structural/group collapse behavior and invokes the existing panel isolate/one-step-restore
state machine.

Move that contextual ownership to the `zoom_panel` action. This is an action migration rather than a hard-coded key
check: user-configured bindings must get the same behavior as the defaults. When a split-layout tribe panel has
whole-panel focus, the action bound to `zoom_panel` should keep that panel expanded and collapse its siblings, or
consume the already-armed session-local restore. With an ordinary selected agent, the same action must continue to open
the existing near-fullscreen detail modal. The action bound to `hooks_or_collapse_all` must no longer isolate or restore
panels, but must retain Tools compaction, structural and grouping collapse, and the existing behavior on the ChangeSpecs
and Axe tabs.

## Implementation

### Route the contextual action by selection type

- Reuse the existing whole-panel isolation helper and its persistence, focus-preservation, marker, invalidation, and
  one-step restore semantics; do not create a second fold-state path.
- In the Agents `zoom_panel` action, test whole-panel focus before resolving a selected agent. If isolation/restore owns
  the action, return without opening a modal or emitting the current "No agent selected" warning. Otherwise keep the
  current agent provider, initial target selection, seed capture, refresh cadence, and modal behavior unchanged.
- Remove the isolation branch from `hooks_or_collapse_all`. Preserve its Tools detail precedence and its existing
  structural/group and cross-tab collapse routing. A whole-panel selection without a valid collapse-all target should
  simply leave `H` with no isolation side effect; lowercase `h` remains the explicit selected-panel collapse action.
- Update nearby comments and docstrings that name `H` as the owner of the session-local restore so they describe the
  zoom action/key accurately.
- Keep the keypress path synchronous and free of new I/O, subprocesses, or full data reloads. Isolation should continue
  through the established selective panel/footer refreshes and existing fold persistence scheduling.

### Align discoverability and configurable-key behavior

- Move the conditional footer affordance for `only panel` / `restore panels` from `hooks_or_collapse_all` to
  `zoom_panel`, using action-based key display lookup so custom keymaps render correctly. Continue to show `H` for
  Tools, structural, and grouping collapse only when those operations are available.
- Update the fallback and registry binding descriptions, built-in configuration comments, Agents help, and
  command-catalog metadata so `Z` is described as detail zoom in row context and panel isolation/restore in whole-panel
  context; remove isolation language from `H` without weakening its other tab-specific descriptions.
- Make command-palette availability match keyboard dispatch: `zoom_panel` is available for either a selected agent or a
  focused tribe panel, but remains unavailable when neither target exists or outside the Agents tab. Retain all existing
  panel restrictions on unrelated agent-only commands.

### Update documentation and regression coverage

- Revise the Agents/tribe-panel user documentation and key tables so isolation, restore markers, and the footer hint
  consistently point to `Z`; keep `H` documented as the contextual structural/group collapse key.
- Convert the isolation and restore unit scenarios to exercise the `zoom_panel` action, including collapsed and expanded
  panels, the reserved default panel, arbitrary pre-isolation layouts, idempotence, restore from a different focused
  panel, disappearing/new panels, invalidation after sibling mutations, and focus/remembered-row preservation.
- Add explicit routing regressions proving that focused-panel `Z` neither opens a modal nor warns, ordinary-agent `Z`
  still opens the correct zoom target, and focused-panel `H` no longer changes isolation state while its Tools,
  structural, grouping, ChangeSpecs, and Axe collapse cases remain intact.
- Update footer, help, binding-description, command-catalog, and command availability expectations. Cover custom
  action-to-key display where relevant so the feature is not accidentally tied to literal `H` or `Z` characters.
- Update only the ACE PNG goldens whose visible footer/help text intentionally changes, then rerun the visual suite with
  exact local comparison to catch unrelated rendering drift.

## Verification

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run focused tests for panel isolation/collapse, zoom action routing, footer bindings, keymap/help metadata, command
   catalog, and command availability.
3. Regenerate intentional visual changes with `just test-visual -- --sase-update-visual-snapshots`, inspect the affected
   images, and rerun `just test-visual` without the update flag.
4. Run `just check` for formatting, lint/type/SASE validation, and the complete fast test suite.

## Risks and boundaries

- The isolation helper accepts both collapsed and explicitly selected expanded panels and owns an armed restore
  independent of the original target. Routing must happen before selected-agent lookup or `Z` will retain its
  warning-only behavior in exactly the context being migrated.
- Footer labels, help, the command palette, and configurable bindings are separate discovery surfaces. Updating only
  runtime dispatch would leave the old key advertised or make the new contextual action inaccessible from the palette.
- Do not redesign the isolation state machine, persisted panel-fold intent, zoom modal, grouping model, or default key
  values. This tale changes which existing action owns the contextual behavior and updates its contracts.
