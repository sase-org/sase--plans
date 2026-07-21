---
tier: tale
title: Make Agents-tab H jump to structural parent containers
goal: 'The Agents-tab H key ascends from an agent to its containing family and from
  a family to its containing clan before falling back to grouping-strategy folding,
  while preserving existing panel and Tools behavior.

  '
create_time: 2026-07-21 13:22:39
status: wip
---

- **PROMPT:** [202607/prompts/agents_h_parent_jump.md](prompts/agents_h_parent_jump.md)

# Plan: Make Agents-tab H jump to structural parent containers

## Context and interaction contract

`hooks_or_collapse_all` is a cross-tab, configurable action whose default key is `H`. On the Agents tab it currently
compacts the Tools detail view, toggles whole-panel isolation/restoration when a tribe panel is selected, and otherwise
collapses the grouping-strategy banner enclosing the selected row (for example, the `Running` banner under `by status`).
Agent clans and sequential families are a separate, rendered structural tree inside those grouping banners.

Extend the Agents-tab dispatch with one structural-parent operation while retaining the existing action id and
configured key:

| Current Agents-tab focus                                                                                                  | `H` result                                             |
| ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Tools detail view                                                                                                         | Keep the existing compact-Tools behavior.              |
| Selected whole tribe panel                                                                                                | Keep the existing isolate/restore behavior.            |
| Real agent row whose validated rendered parent is an agent-family container                                               | Jump to that family row without changing folds.        |
| Agent-family container whose validated rendered parent is a clan container                                                | Jump to that clan row without changing folds.          |
| Clan row, grouping banner, standalone agent/family, direct non-family clan member, or invalid/missing parent relationship | Keep the existing grouping-strategy collapse behavior. |

This produces a predictable ladder for a selected member of a family inside a clan: the first `H` selects the family,
the second selects the clan, and the third collapses the clan's active grouping-strategy group. The same behavior must
hold under every grouping mode, including `by status`; structural descendants already inherit the outer row's grouping
identity, so these jumps stay inside the current tribe panel and group.

## Resolve and apply the structural-parent jump

Add a small Agents-tab helper that resolves the selected row's immediate rendered parent from the already-loaded tree
projection. Reuse the canonical tree parent key/lookup semantics rather than inferring ancestry from display names,
family-name prefixes, or artifact reads. Validate both ends of the edge: an agent target must resolve to a real family
container, while a family target must resolve to the synthetic clan container. Treat workflow steps that actually
represent agents as eligible family members, but do not redirect python/bash/hidden workflow rows or a plain agent that
is merely a direct clan member. Return no target for whole-panel focus, group-banner focus, stale rows,
ambiguous/missing parents, or any relationship that does not match the two supported container transitions.

Keep this resolver pure and in-memory so `H` remains a prompt-free keystroke path with no filesystem, subprocess,
refresh, or full-list rebuild work. Share the same resolved target/capability between action routing and footer state so
the advertised behavior cannot drift from the action.

Route the successful parent jump after Tools and selected-panel handling but before `_collapse_group_fold()`. Move focus
to the already-visible parent row, clear attempt and banner focus, retain the current panel and all tree/group/panel
fold state, and update the ordinary selection bookkeeping: remember the row for panel restoration, apply unread
departure/acknowledgement behavior, and record the origin in Agents jump history so the existing back/forward jump
commands can return to the child. Let the existing reactive/debounced focus path repaint the highlight and detail panel;
do not introduce a synchronous full refresh. If resolution fails at action time, fall through to the existing group-fold
operation without partially changing selection or history.

This is presentation-only TUI navigation. It belongs in the Python Agents-tab action/tree layer and does not require a
Rust core API change.

## Keep keymap guidance and conditional affordances accurate

Update the built-in/fallback binding description, command-catalog label, default-config comment, and Agents help-popup
entry to describe the new parent-container jump alongside panel isolation/restoration, group folding, and Tools
compaction. Do not rename the action or alter the configurable `H` default.

Because the parent jump is selection-dependent, expose it in the conditional footer when—and only when—the selected row
has a valid family or clan parent target. Use concise target-specific labels such as `parent family` and `parent clan`.
Suppress that chip when Tools detail, whole-panel focus, or group-banner focus owns `H`, allowing their existing
contextual footer/help behavior to remain authoritative.

## Verification

Add focused transition coverage around the existing Agents folding tests:

- Build a clan containing a sequential family and verify member → family → clan → grouping banner across repeated `H`
  presses, with no structural, grouping, or panel fold mutation during the first two presses.
- Repeat the transition under `BY_STATUS` so the `Running`-style grouping case is explicit, and cover a standalone
  family member jumping to its family before the family falls back to group folding.
- Cover real workflow-agent members of a family, while proving non-agent workflow children, direct non-family clan
  members, stale/malformed ancestry, focused panels, grouping banners, and Tools routing do not incorrectly take the new
  branch.
- Assert selection bookkeeping: attempt/banner focus clears, panel selection memory points at the parent, unread
  handling remains consistent, and jump back/forward can restore the child after a successful `H` jump.
- Extend footer tests for the family/clan labels and precedence suppression, and update binding-description,
  command-catalog, default-config/help tests to lock the public wording and configurable action identity.
- Add or update one focused Agents visual snapshot only if needed to lock the new conditional footer chip; accept no
  unrelated golden changes.

Before handing off, run `just install`, the targeted Agents folding/navigation and footer/help tests, any affected
visual snapshot test, and finally the required `just check`. Inspect visual diff artifacts before accepting any
intentional snapshot update.
