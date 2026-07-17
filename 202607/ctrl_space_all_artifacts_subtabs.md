---
tier: tale
title: Restore Ctrl+Space across Artifacts sub-tabs
goal: 'The configured Ctrl+Space repeat-agent shortcut dispatches consistently from
  PRs, Commits, Bugs, and Plans without re-enabling PR-only actions on hidden Artifacts
  panes.

  '
create_time: 2026-07-17 09:31:56
status: wip
prompt: 202607/prompts/ctrl_space_all_artifacts_subtabs.md
---

# Plan: Restore Ctrl+Space across Artifacts sub-tabs

## Context

ACE canonicalizes `<ctrl+space>` to Textual's `ctrl+@` key name and already binds it to `start_agent_from_changespec`,
whose current behavior is to repeat the last launchable selection made through `+` or Ctrl+Space. The binding and
repeat-selection action are healthy: a mounted-app reproduction dispatches the shortcut on the PRs sub-tab.

The failure is at the Artifacts action boundary. Because every Artifacts sub-tab retains the historical top-level
`changespecs` ID, `AceApp.check_action` uses `NON_PRS_ARTIFACT_ACTIONS` to prevent PR-specific bindings from acting on a
hidden PR selection. That allowlist contains the other global agent entry points but omits
`start_agent_from_changespec`. As a result, the same mounted reproduction reports the action as allowed and dispatched
on PRs, but rejected and undispatched on Commits, Bugs, and Plans.

## Implementation

1. Treat `start_agent_from_changespec` as a global action at the non-PR Artifacts boundary by adding it to
   `NON_PRS_ARTIFACT_ACTIONS` in the shared Artifacts actions module. Keep the surrounding deny-by-default guard and all
   pane-specific action gates intact, so historical PR actions remain blocked whenever Commits, Bugs, or Plans is
   visible.
2. Do not introduce per-pane bindings or change the default keymap. The existing configurable action binding, Ctrl+Space
   alias normalization, and repeat-selection implementation should remain the single dispatch path for every top-level
   tab and Artifacts sub-tab.
3. Extend the mounted Artifacts scaffold coverage with a regression that replaces the repeat-selection action with a
   recorder, visits each of PRs, Commits, Bugs, and Plans, presses Textual's canonical `ctrl+@` input, and verifies
   exactly one dispatch from every sub-tab. Also assert the action gate is enabled on each pane so a future allowlist
   regression is diagnosed at its source rather than only as a missing callback.

## Validation

1. Run `just install` first, as required for an ephemeral SASE workspace.
2. Run the focused mounted-TUI and keymap tests covering the Artifacts scaffold and Ctrl+Space binding/normalization
   behavior.
3. Run `just check` to exercise formatting, linting, type checks, SASE validation, committed-plan validation, the full
   fast test suite, and visual snapshots.

## Acceptance criteria

- Pressing `<ctrl+space>` dispatches the repeat-last agent-selection action on all four Artifacts sub-tabs: PRs,
  Commits, Bugs, and Plans.
- The behavior uses the configured `start_agent_from_changespec` binding, so canonical aliases and user remaps continue
  to flow through the existing keymap registry.
- Non-PR panes still reject unrelated PR-only actions and retain their existing pane-specific key handling.
- Focused regression tests and `just check` pass without requiring visual snapshot changes.
