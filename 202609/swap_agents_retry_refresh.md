---
tier: tale
title: Swap Agents retry and refresh keys
goal: Make r refresh and R retry on Agents without changing the other tabs.
size: medium
proposed_by: bbugyi200.athena.55
create_time: 2026-09-13 15:13:08
status: wip
---

# Swap Agents Retry And Refresh Keys

## Goal

On the ACE TUI's **Agents** tab, make lowercase `r` run the existing refresh action
(opening the Refresh panel while the `refresh_panel` flag is enabled, or immediately
refreshing the tab while it is disabled) and make uppercase `R` retry the selected local
or remote agent. Preserve the existing meanings on every other top-level tab: `r`
continues to run a Patch workflow or run/re-run an Axe item, and `R` continues to
refresh the active Artifacts or Axe surface.

The effective keys must remain configurable, correctly scoped, and accurately shown in
the footer, Help modal, and command palette. The palette's semantic actions must remain
direct: choosing Refresh refreshes, and choosing Retry retries, independent of the
contextual keyboard routing.

## Current Behavior

- `ace.keymaps.app.run_workflow` defaults to `r`. Its `action_run_workflow` handler is
  contextual: Patch workflow on Artifacts, local/remote retry on Agents, and chop/bgcmd
  run or re-run on Axe.
- `ace.keymaps.app.refresh` defaults to `R` and calls `action_refresh` on every tab.
  With `refresh_panel` enabled it opens `RefreshPanelModal`; with the flag disabled it
  schedules the existing current-tab refresh directly.
- Both actions are app-level bindings and command-palette entries. Merely reversing
  those two defaults would therefore also reverse Patch/Axe behavior, contrary to the
  Agents-only scope of this change.
- The keymap registry already supports tab-disjoint actions sharing a physical key:
  duplicate pairs are declared in `_CONTEXTUAL_APP_DUPLICATES`, Textual consults
  `check_app_action`, and only the action valid for the active tab fires.

## Design

Introduce two explicit Agents-only app actions while retaining the existing generic
actions for the other tabs:

| Keymap action    | Default | Active tabs    | Behavior                                            |
| ---------------- | ------- | -------------- | --------------------------------------------------- |
| `agents_refresh` | `r`     | Agents         | Delegate to the existing refresh action/panel path  |
| `agents_retry`   | `R`     | Agents         | Delegate to the existing local-or-remote retry path |
| `run_workflow`   | `r`     | Artifacts, Axe | Keep Patch workflow and Axe run/re-run behavior     |
| `refresh`        | `R`     | Artifacts, Axe | Keep the existing refresh action/panel behavior     |

Declare `agents_refresh`/`run_workflow` and `agents_retry`/`refresh` as intentional
contextual duplicate pairs. Scope all four through both Textual action availability and
command-palette tab/applicability metadata, so keyboard dispatch cannot fall through to
the wrong same-key action and palette rows keep their semantic action IDs. Keep
`action_run_workflow` and `action_refresh` as the established implementation paths; the
Agents-only methods are thin delegates, with the retry delegate preserving the existing
remote-capability check and local `_retry_edit_agent()` behavior.

Custom overrides continue to follow the semantic action names. For example, a user can
rebind `agents_retry` without moving Patch/Axe `run_workflow`; intentional same-key
overrides for the declared tab-disjoint pairs remain valid, while unrelated collisions
continue to revert under the existing duplicate-key policy.

## Implementation Steps

1. **Add the scoped keymap actions and defaults.**
   - Extend `AppKeymaps` and the app binding metadata with `agents_refresh` and
     `agents_retry` in the Agents section.
   - Add `agents_refresh: "r"` and `agents_retry: "R"` to `src/sase/default_config.yml`,
     retaining the existing `run_workflow: "r"` and `refresh: "R"` defaults for
     non-Agents tabs.
   - Update the class-level fallback bindings in `src/sase/ace/tui/bindings.py` so
     startup-before-registry behavior has the same contextual pairs (and uses the
     canonical `R` refresh default rather than the stale fallback spelling).
   - Add both intentional pairs to `_CONTEXTUAL_APP_DUPLICATES`; do not weaken collision
     validation for any other actions.

2. **Route keyboard actions by tab without duplicating business logic.**
   - Add an Agents refresh action that delegates to `action_refresh`, preserving both
     feature-flag states and the existing background/coalesced refresh path.
   - Add an Agents retry action that delegates to the current Agents branch of
     `action_run_workflow` (or extracts that branch into one shared helper used by both
     entry points), preserving local prompt editing and remote lifecycle retry.
   - Update `check_app_action` so `agents_refresh` and `agents_retry` are enabled only
     on Agents, while generic `refresh` and `run_workflow` are disabled there and stay
     active in their existing non-Agents contexts. Preserve prompt-input ownership and
     remote capability gating for retry.

3. **Keep command discovery semantic and correctly scoped.**
   - Split command metadata so `app.agents_retry` is the Agents-only retry row,
     `app.agents_refresh` is the Agents-only dynamic Refresh/Open Refresh panel row,
     `app.run_workflow` covers Artifacts/Axe run behavior, and `app.refresh` covers
     Artifacts/Axe refresh behavior.
   - Apply the feature-flag-dependent refresh label to both refresh command IDs.
   - Move Agents retry applicability checks (selected agent, proc/monitor exclusion, and
     remote `lifecycle.retry`) from `app.run_workflow` to `app.agents_retry`. Refresh
     remains available on Agents even when no agent row is selected.

4. **Update every user-facing key surface.**
   - In the Agents footer, render retry from `agents_retry` so eligible local and remote
     rows advertise `R: retry`; continue omitting retry for unsupported
     proc/monitor/gate rows.
   - In Agents Help, render retry from `agents_retry` and refresh/full-history-panel
     entry points from `agents_refresh`. Leave Artifacts and Axe Help wired to the
     generic actions.
   - Update `docs/ace.md` Agent Actions and global/contextual keybinding text to say `r`
     refreshes/opens the Refresh panel and `R` retries on Agents, while explicitly
     documenting that the other tabs retain their current `r` run and `R` refresh
     behavior.

5. **Add regression coverage for routing, configuration, and presentation.**
   - Keymap tests: assert all four defaults, binding order for each same-key pair,
     acceptance of the two contextual duplicate pairs under custom overrides, and
     rejection/reversion of unrelated collisions.
   - Action/availability tests: assert local and remote Agents retry through
     `agents_retry`; Agents `agents_refresh` uses the panel when the flag is on and the
     immediate async refresh when it is off; generic same-key actions are unavailable on
     Agents; and Agents-only actions are unavailable on Artifacts/Axe.
   - End-to-end key tests: from Agents, pressing `r` opens/executes refresh and pressing
     `R` retries an eligible row; from Artifacts and Axe, retain representative checks
     that `r` runs and `R` refreshes.
   - Footer/Help tests: assert `R: retry` and `r: Open Refresh panel` (or `Refresh` with
     the flag off), including custom key overrides and remote-row eligibility.
   - Command-catalog/palette tests: assert the Agents-only command IDs, labels, keys,
     aliases, scopes, and direct execution, and confirm the non-Agents semantic rows
     still call `action_run_workflow`/`action_refresh`.

## Performance And Safety Constraints

- Do not add an `on_key` interception, synchronous I/O, subprocess work, or a new
  refresh implementation. Let Textual's existing action availability resolve the two
  tab-disjoint bindings, and reuse `_schedule_agents_async_refresh()` plus the current
  durable local/remote retry machinery.
- Keep refresh feature-flag behavior unchanged apart from the Agents key that reaches
  it. Keep Refresh-panel internal chooser keys (`r`/`R` aliases for “This tab”) and the
  `,y` full-history migration unchanged.
- Do not change retry semantics, prompt construction, remote protocol behavior, or key
  behavior inside focused modals and text inputs.

## Verification

1. Run the focused keymap, action-dispatch, command-catalog/palette, footer/Help, and
   Refresh-panel tests updated above. Include the existing Patch/Axe `r`/`R` tests to
   catch cross-tab regressions.
2. Run `just install` if the workspace environment is stale, then run `just check`
   (using `/sase_monitor` if it becomes long-running) and fix all reported lint, type,
   size, and scoped-test failures.
3. Sanity-launch ACE: on Agents verify `r` opens the Refresh panel and `R` opens the
   retry prompt for an eligible local row; switch to Artifacts and Axe and verify their
   existing `r` run and `R` refresh gestures are unchanged. Also check Agents Help and
   the command palette for matching labels and effective configured keys.
