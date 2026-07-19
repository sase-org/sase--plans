---
tier: tale
title: Move the Agents visible-fold selector to L
goal: 'Pressing L on the Agents tab opens the numeric visible-fold selector directly,
  while the former ,H leader binding and the previous Agents-tab L behavior are retired
  without changing L behavior on other tabs.

  '
create_time: 2026-07-19 08:07:05
status: wip
prompt: 202607/prompts/agents_fold_selector_l_keymap.md
---

# Plan: Move the Agents visible-fold selector to L

## Context and intent

The Agents tab currently opens its unified numeric fold selector through the leader sequence `,H`. The direct uppercase
`L` app binding instead expands the focused tag panel (or raises the Tools detail level before the tab-specific fold
dispatch runs). The desired interaction is a direct `L` shortcut for selecting and atomically toggling any visible
panel, grouping banner, clan/family/workflow, or agent-owned fold. The old Agents-tab meaning of `L` does not need a
replacement: lowercase `l` remains the focused expand/enter operation, and uppercase `H` continues to perform contextual
group collapse.

This is presentation-only Textual/keymap work. It remains in the Python TUI and does not require a Rust core API change.

## Implementation

### Route direct `L` to the visible-fold selector on Agents

- Change the contextual `expand_all_folds` app action so its Agents-tab branch enters
  `action_toggle_selected_agent_panels()` immediately. On Agents, this must replace both pieces of the former `L`
  behavior: expanding a focused tag panel and maximizing a visible Tools detail panel.
- Preserve the existing `L` behavior for the ChangeSpecs and Axe tabs, including their all-fold expansion and
  Tools-detail routing. Preserve the existing numeric selector implementation, validation, atomic application,
  persistence, and empty/unavailable-state notifications; only its entry point changes.
- Keep the app-level binding configurable through `ace.keymaps.app.expand_all_folds`, whose bundled default remains `L`.
  Avoid a second active shortcut for the selector in the default registry.

### Retire the `,H` leader command coherently

- Remove `toggle_selected_agent_panels` from both leader-mode default sources: `LeaderModeKeymaps` and
  `src/sase/default_config.yml`.
- Add the removed leader action id to the loader's retired-key filter so an old user override cannot deep-merge the
  obsolete command back into the registry. Cover this compatibility behavior explicitly rather than allowing a stale
  configuration to create an advertised but undispatchable command.
- Remove the leader dispatcher branch and its repeat/refresh bookkeeping, the leader footer entry, the leader
  command-catalog metadata, and the Agents help entry for `,H`. Leave the underlying selector action available for the
  new app-level `L` dispatch.

### Align discoverability and descriptions

- Update the bundled keymap comments, app binding metadata/fallback binding description, Agents help popup, and
  command-palette wording so `L` is described as the numeric visible-fold selector on Agents, not as focused-panel
  expansion. Keep descriptions accurate for the same contextual app action on ChangeSpecs and Axe.
- Continue following the footer convention: because `L` is an app-level/global binding, document it in the help and
  command palette rather than adding it to the conditional footer. When leader mode is open on Agents, it should no
  longer advertise `H` as “toggle folds.”

## Tests and validation

- Update keymap default, registry-loading, app-binding, help-display, leader footer/dispatch, and command-catalog tests
  to establish all of these contracts: the app default is `L`; the leader default has no `toggle_selected_agent_panels`;
  stale overrides are filtered; no `,H` help, footer, or catalog command remains; and the direct Agents command is
  presented with its new behavior.
- Add or adapt focused action tests to prove that `action_expand_all_folds()` on Agents enters the existing fold-hint
  mode, including when Tools detail is visible, and no longer expands a collapsed focused panel. Retain regression
  coverage that lowercase `l` still expands/enters panels and that uppercase `H` still collapses groups.
- Update the Agents interaction visual exercise to press `L` directly instead of `,H`, then retain its assertions and
  PNG coverage for the selector, hint labels, mixed submission, focus, and collapsed-panel result. Accept a snapshot
  update only if the rendered help or selector UI intentionally changes.
- Add or retain non-Agents coverage showing that `L` still expands all folds (and still routes Tools detail
  appropriately) on ChangeSpecs/Axe, preventing the tab-specific remap from becoming a global semantic regression.
- Before handoff, run `just install`, the focused pytest modules for keymap, command/help, folding, and interaction
  coverage, `just test-visual` when a visual golden is affected, and the repository-required `just check`.

## Acceptance criteria

- With default configuration, one press of uppercase `L` on the Agents tab opens the numeric selector for all currently
  visible fold owners.
- `,H` no longer triggers or advertises that selector, including when a stale user configuration still names the retired
  leader action.
- On Agents, `L` no longer expands the focused tag panel or maximizes Tools detail; lowercase `l` and uppercase `H`
  retain their current focused expand/collapse roles.
- ChangeSpecs and Axe keep their existing `L` behavior, all help/catalog surfaces match runtime behavior, and the
  focused, visual, and full repository checks pass.
