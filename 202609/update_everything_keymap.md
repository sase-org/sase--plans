---
tier: tale
title: Add a global update-everything shorthand
goal: A configurable `,E` leader chord safely performs the same previewed, no-confirmation
  Everything update as `,UE`.
size: small
proposed_by: bbugyi200.athena.092
status: done
---

# Add a `,E` update-everything shorthand

## Objective

Add a configurable, global leader-mode `,E` action that is behaviorally equivalent to
opening the Update panel with `,U` and selecting capital `E`: plan the Everything scope
from the current cached update/provider projection, skip the final confirmation only
when that preview is runnable, and run the existing tracked comprehensive-update proc.

## Behavioral contract

- `,E` works from Artifacts, Agents, and Axe and submits a `ComprehensiveUpdateRequest`
  with `scope=UpdateScope.EVERYTHING` and `auto_approve=True`, using the provider names
  present at invocation time.
- The shortcut does not mount the Update panel or a confirmation modal. It does not
  bypass planning: preview failures and already-current/non-runnable previews keep the
  existing non-mutating behavior, while only a runnable preview advances to the
  established mutation proc.
- Dispatch remains keystroke-safe: no disk, network, subprocess, or other blocking work
  runs on the Textual event loop. The existing durable preview and update proc paths,
  deduplication, exclusive scopes, history trigger, result reporting, and restart
  behavior remain authoritative.
- `,U` and every Update-panel key retain their current behavior. `,,` remembers and
  repeats `,E` as the direct auto-approved action, subject to the existing
  preview/update deduplication.
- The binding remains user-remappable through `ace.keymaps.modes.leader_mode.keys`, and
  the effective key appears consistently in leader footers, Help, and the command
  palette.

## Implementation

1. Add a distinct `update_everything` leader action with default subkey `E` to both
   `src/sase/default_config.yml` and the typed `LeaderModeKeymaps` defaults in
   `src/sase/ace/tui/keymaps/mode_keymaps.py`. Keep all default leader subkeys unique.
2. Refactor the Update-panel result callback in `src/sase/ace/tui/actions/base.py`
   behind one small request-submission helper that reads the current provider-name
   projection, constructs the typed scoped request, and delegates to
   `_submit_update_preview_proc`. Reuse that helper for the existing panel callback and
   for a new `action_update_everything_shortcut`, so the shorthand cannot drift from
   capital `E` semantics or duplicate preview/execution logic.
3. Dispatch the configured `update_everything` subkey in
   `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`, remembering it for leader
   repeat and refreshing the current tab through the same path as `update_sase`.
4. Make the action discoverable everywhere leader commands are surfaced: give it a
   precise no-confirmation label in `src/sase/ace/tui/commands/_mode_commands.py`, add
   it to the leader footer in `src/sase/ace/tui/widgets/_keybinding_modes.py`, and add
   the configured chord to the Artifacts, Agents, and Axe Help sections under
   `src/sase/ace/tui/modals/help_modal/`.
5. Update the user documentation in `docs/ace.md` and `docs/configuration.md` to list
   `,E` in each global leader table and describe it as the direct alias for `,U` then
   capital `E`, including the retained preview/no-op/error guarantees. Adjust the
   focused update documentation in `docs/plugins.md` and `docs/agent_providers.md` where
   comprehensive-update entry points or history are enumerated, without presenting the
   shortcut as bypassing the preview stage.

## Verification

- Extend the default-keymap tests to assert the typed and merged defaults bind
  `update_everything` to `E`, the leader defaults remain collision-free, and all three
  Help binding lists expose the effective shortcut.
- Extend leader dispatch coverage and its fake app harness to prove `E` invokes only the
  new direct action, records `E` for `,,`, works on every tab, and still follows a
  configured remap.
- Extend command-catalog tests to assert `leader.update_everything` has the intended
  label, key display, all-tab scope, and `leader_mode_key` executor.
- Extend Update shortcut tests to prove the direct action pushes no `UpdatePanel`,
  submits exactly one Everything/auto-approved request with the current provider
  projection, and performs no synchronous file or subprocess work. Retain the existing
  tests proving auto-approved runnable previews skip confirmation while failed and
  non-runnable previews do not mutate.
- Extend leader-footer tests for all tabs, regenerate the intentional wide and narrow
  LEADER footer PNG goldens, run the targeted keymap/update/help/catalog and visual
  snapshot tests while iterating, and finish with the repository-required `just check`.

## Non-goals

- Do not add `,S` or `,P` global aliases, change Update-panel row bindings, broaden the
  provider inventory captured for an invocation, or introduce a second update pipeline.
- Do not move shared update execution into Python or Rust core; this is presentation and
  TUI glue around the existing backend behavior.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact | Why | Uses |
| --- | --- | --- | ---: |
| cited-by | [agent:bbugyi200.athena.092--code][1] | prompt reference @plan:202609/update_everything_keymap.md | 1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.092.md

<!-- sase:referenced-by:end -->
