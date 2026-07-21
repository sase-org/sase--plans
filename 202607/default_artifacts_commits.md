---
tier: tale
title: Default the Artifacts tab to Commits
goal: 'A fresh ACE session opens the Commits sub-tab the first time the user enters
  Artifacts, while preserving lazy pane loading, explicit navigation order, and the
  user''s in-session sub-tab selection on later visits.

  '
create_time: 2026-07-21 10:01:33
status: wip
prompt: '[202607/prompts/default_artifacts_commits.md](prompts/default_artifacts_commits.md)'
---

# Plan: Default the Artifacts tab to Commits

## Context

Artifacts currently has two independent startup defaults: the app's reactive sub-tab state is hard-coded to PRs, while
the Artifacts view initializes from `DEFAULT_ARTIFACTS_SUBTAB`, which is also PRs. The view's mount lifecycle then
activates the PR pane specifically. Merely changing the shared constant would therefore leave the app state, visible
content, action availability, and pane lifecycle out of sync. It could also start the Commits collector while Artifacts
is hidden, violating the established lazy-loading and TUI startup performance contract.

The canonical sub-tab order is already Commits, Plans, Bugs, PRs. This change does not reorder tabs or numbered
shortcuts; it makes Commits the initial selection for a fresh app instance. After the user chooses another sub-tab,
leaving and returning to Artifacts must continue to restore that in-session selection rather than resetting it to
Commits.

## Implementation

### Make one default drive app and view state

Change the canonical Artifacts default to Commits and use that shared value for both the
`AceApp.current_artifacts_subtab` reactive initializer and the `ArtifactsView` initial tab/content selection. Keep
PR-specific predicates and defensive fallbacks semantically PR-specific; they control PR-only behavior and are not
alternate user-facing startup defaults. Keep `ARTIFACTS_SUBTAB_ORDER`, pane IDs, key bindings, and command-palette
numbering unchanged.

### Align first-entry lifecycle and startup behavior

Generalize the Artifacts view mount lifecycle so it activates the selected pane only when Artifacts is actually visible.
A normal ACE startup on Agents must leave every Artifacts pane inactive and must not start Commits collection; the
existing top-level tab watcher should activate Commits when the user first navigates to Artifacts.

Also make the less-common path that starts directly on Artifacts equivalent to entering it through navigation: Commits
should be the active lifecycle pane, the non-PR footer should be shown, and project-scope inventory should be scheduled
through the existing off-thread, coalesced path. Reuse or factor the existing non-PR entry behavior so mount and
tab-switch paths cannot drift. Do not add synchronous I/O or data-scaled work to mount, render, or action handlers.

### Preserve navigation semantics

Verify that direct numbered jumps and bracket cycling still follow Commits, Plans, Bugs, PRs; PR-only actions remain
unavailable on Commits; and explicit PR navigation continues to activate the existing ChangeSpec surface. Once a user
selects Plans, Bugs, or PRs, switching to Agents or AXE and back should reactivate that selected pane without applying
the default again.

## Test coverage

Update the Artifacts scaffold/navigation coverage to assert the complete first-entry contract from an Agents start:
before navigation, Commits remains inactive and uncollected; after entering Artifacts, app state, tab-strip state, the
content switcher, action availability, footer mode, and lifecycle all agree that Commits is selected. Cover
direct-on-Artifacts startup separately so it uses the same non-PR initialization, and retain coverage that subsequent
visits remember an explicitly selected sub-tab.

Audit mounted TUI tests that implicitly relied on PRs being the initial Artifacts sub-tab. Make PR-specific setup
explicit by navigating to PRs rather than weakening the production default or hiding it in the shared test harness.
Update assertions that genuinely describe the default to expect Commits. For visual coverage, retain an intentional
snapshot of the new default Commits surface and make PR-specific snapshots explicitly select PRs, accepting golden
changes only where the requested default is what the snapshot is meant to exercise.

Run focused Artifacts, Commits, saved-query, and PR navigation tests while iterating, then run the dedicated PNG
snapshot suite. Because source or test files will change, run `just install` before validation commands and finish with
the repository-required `just check`.

## Acceptance criteria

- A fresh ACE session does no hidden Commits collection while starting on Agents, and its first navigation to Artifacts
  selects and activates Commits.
- Starting ACE directly on Artifacts also presents a fully initialized Commits pane without blocking the event loop.
- App reactive state, active tab styling, visible content, footer/action scope, and pane lifecycle never disagree about
  the selected sub-tab.
- Leaving and revisiting Artifacts preserves the user's latest in-session sub-tab choice.
- Numbered jumps, cycling order, PR behavior, and non-PR lazy loading remain intact.
- Focused tests, PNG visual snapshots, and `just check` pass.

## Scope boundaries and risks

This is presentation and Textual lifecycle behavior, so no Rust core, CLI, configuration-schema, or default-keymap
changes are needed. The main risks are an eager hidden Commits load, duplicated first activation, stale PR footer or
command availability, and broad test failures caused by implicit PR setup; the lifecycle and mounted regression coverage
above directly guard those cases.
