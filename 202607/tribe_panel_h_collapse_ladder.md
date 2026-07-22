---
tier: tale
title: Add the selected tribe-panel H collapse ladder
goal: 'Uppercase H on a selected Agents-tab tribe panel collapses all open houses
  in that panel, then one expanded top-level group at a time from the bottom, and
  finally the panel itself, with accurate hints and persisted fold state.

  '
create_time: 2026-07-22 08:11:46
status: wip
---

- **PROMPT:** [202607/prompts/tribe_panel_h_collapse_ladder.md](prompts/tribe_panel_h_collapse_ladder.md)

# Plan: Add the selected tribe-panel `H` collapse ladder

## Context

The Agents tab currently gives uppercase `H` a row-focused collapse ladder: Tools detail takes precedence, then the next
grouping scope's open agent houses are fully collapsed, followed by structural and grouping folds. Whole- panel focus is
deliberately excluded, so `H` is currently a no-op when a split tribe panel is selected; lowercase `h` is the only
collapse action in that context and immediately collapses the panel.

Extend the existing `hooks_or_collapse_all` action rather than checking the literal `H` key. Custom keymaps must receive
the same selected-panel behavior. Once whole-panel focus owns the action, apply exactly one step per press in this
priority order:

1. If the selected expanded panel contains any valid canonical house whose structural fold is not fully collapsed, drive
   every such house in that panel directly to fully collapsed in one press. This is panel-wide rather than limited to
   one grouping banner, because whole-panel focus has no selected row or group.
2. Otherwise, find the level-0 grouping banners for that panel in their actual rendered order, discard banners already
   collapsed, and collapse only the last expanded banner. Repeated presses therefore walk backward through the panel's
   top-level groups (for example, `Done` in a status fixture or `Yesterday` in a date fixture) without changing nested
   group folds.
3. When neither an open house nor an expanded level-0 group remains, use the same whole-panel collapse path as lowercase
   `h`.

A panel that is already collapsed is a saturated terminal state: `H` should follow lowercase `h`'s existing
already-collapsed no-op/notification rather than mutating hidden house or group state. Merged layout has no whole-panel
focus and must retain the existing row-focused behavior. The Tools-detail branch and the ChangeSpecs/Axe meanings of
this shared action remain unchanged.

## Resolve panel-scoped collapse targets

Add focused-panel target resolution alongside the existing group-scoped house and group fold helpers. Keep the probes
pure and entirely in memory: use the current panel group/key, cached panel slice/index, active grouping mode, fold
counts, fold manager, and that panel's `GroupFoldRegistry`; do not perform I/O, reload agents, or mutate state while
deciding the next step.

For the house step, share the existing canonical-owner validation rather than loosening it. A candidate must still have
one unambiguous global fold-key owner, a valid standalone/direct-clan-member shape, collapsible descendants, and a
non-collapsed fold level. Duplicate or malformed owners must fail closed. Limit candidates to agents rendered in the
focused tribe panel, but validate fold-key uniqueness across the full loaded agent set because the fold manager is
global. Include houses even when a grouping banner currently hides them; group folds and structural folds are
independent, and the first priority is explicitly every open house in the selected panel. A whole-panel action has no
row to re-anchor, so it must preserve `current_idx`, the remembered in-panel selection, the focused panel key, and
explicit whole-panel focus.

For the group step, derive level-0 banners from the same fully expanded tree projection/order used by the renderer.
Choose the last key whose panel-local registry says it is expanded. This must work uniformly for `STANDARD`,
`BY_STATUS`, and `BY_DATE`, including absent buckets, singleton suppression at deeper levels, the reserved `@default`
panel key, and equal group names in other tribe panels.

## Apply one transition per action

Route Agents-tab `action_hooks_or_collapse_all` through the focused-panel ladder before the existing row-focused
house/structural/group ladder. Return after the first successful step so a single press never collapses houses and a
group, multiple groups, or a group and its panel together.

The house transition should reuse the fold manager's saturating bulk collapse and run the normal in-memory agent
refilter exactly once with content-index refresh disabled. The group transition should mutate only the focused panel's
registry, persist the change with the explicit panel scope and active grouping mode, and likewise repaint/refilter once
through the established fold-only path. Do not add a full agent-list rebuild, background reload, disk read, or new
async/timer path to the keypress. Preserve whole-panel focus and its remembered row across both inner transitions.

For the terminal step, delegate to the existing lowercase-`h` selected-panel collapse behavior so panel intent,
persistence, layout refresh, isolation- restore invalidation rules, selection retention, and the already-collapsed
notification remain single-sourced. Do not change `Z` isolation/restore state for house or group-only transitions; only
the existing panel-fold path should apply its established isolation rules.

## Discovery surfaces and documentation

Make the footer describe the next distinct uppercase action while a panel is selected. Prefer the configured key for
`hooks_or_collapse_all`, showing `collapse houses` for step one and a concise group-collapse label for step two. Once
the ladder reaches the panel-collapse step, the existing lowercase-`h` `collapse panel` chip already advertises the same
operation, so avoid a duplicate uppercase chip. Keep collapsed-panel and `Z only panel` / `Z restore panels` hints
intact. Resolve footer availability from the same pure target helpers as action dispatch so hints cannot drift from
runtime behavior, and cover a custom key bound to `hooks_or_collapse_all`.

Update the action binding description, command-catalog label/aliases if needed, Agents help modal, default-config
comments, `docs/ace.md`, and `docs/agent_families.md`. Document the whole-panel ordering separately from the existing
row/group-scoped ladder: all houses in the selected panel, last expanded level-0 banner per press, then the panel.
Remove statements that `H` is a no-op on whole-panel focus while keeping the contextual behavior clear for user-remapped
keys.

## Verification

Add focused unit regressions for the transition priority and isolation boundaries:

- One selected expanded panel with open houses spanning multiple top-level groups collapses every valid house in that
  panel in one press, leaves all banners and panel folds open, does not touch an equal/open house in another panel,
  preserves whole-panel focus/selection memory, and uses one fold-only refilter.
- Malformed and globally duplicated house owners remain unchanged while valid siblings still collapse, matching the
  existing fail-closed contract.
- With houses saturated, status- and date-group fixtures collapse only the last currently expanded level-0 banner in
  rendered order; repeated presses walk backward, skip already-collapsed banners, leave nested folds and other panel
  registries alone, and persist each mutation under the selected panel key.
- After all level-0 banners are collapsed, the next `H` produces the same panel fold, focus retention, refresh,
  persistence, and isolation interaction as lowercase `h`. Repeated `H` on an already collapsed panel performs no hidden
  inner mutations and preserves the existing notification.
- Row/banner focus, merged layout, Tools detail, ChangeSpecs, and Axe retain their current `hooks_or_collapse_all`
  behavior.
- Footer tests cover each priority state, omission of a duplicate terminal chip, selected/collapsed panels, `Z` restore
  hints, and a custom action key; command/help/keymap tests assert the revised wording.

Update only ACE PNG snapshots whose selected-panel footer intentionally changes. Inspect their actual/expected/diff
artifacts, then rerun the affected visual scenarios with exact local comparison. In the ephemeral workspace, run
`just install` before checks, run the focused fold/panel/footer/catalog tests during development, and finish with the
repository-required `just check`. Keep unrelated snapshot or renderer drift out of this change.
