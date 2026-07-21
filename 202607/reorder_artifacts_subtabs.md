---
tier: tale
title: Reorder Artifacts sub-tabs and numeric shortcuts
goal: 'The Artifacts tab presents Commits, Plans, Bugs, and PRs in that order, with
  numeric jumps 1 through 4 and every related help, palette, test, and documentation
  surface reflecting the same mapping.

  '
create_time: 2026-07-21 07:51:31
status: done
prompt: 202607/prompts/reorder_artifacts_subtabs.md
---

# Plan: Reorder Artifacts sub-tabs and numeric shortcuts

## Context and intent

Artifacts currently has a shared order constant that feeds the numbered tab strip, bracket-key cycling, fixed fallback
and runtime bindings, and command palette hints. Change that canonical sequence to `Commits`, `Plans`, `Bugs`, `PRs`, so
the associated numeric mapping becomes `1=Commits`, `2=Plans`, `3=Bugs`, and `4=PRs` everywhere rather than maintaining
separate mappings.

Keep PRs as the initially active Artifacts pane. The requested change is to ordering and numeric navigation; changing
startup selection would also alter the existing eager PR lifecycle and lazy loading of the other panes.

## Canonical ordering and navigation

- Update the shared Artifacts sub-tab order in `src/sase/ace/tui/artifact_tabs.py` to `commits`, `plans`, `bugs`, `prs`.
- Continue deriving the tab strip, forward/reverse wraparound cycling, class-level fallback bindings, configured runtime
  bindings, and command palette key displays from that single order. Avoid introducing duplicated digit-to-pane tables.
- Preserve stable pane identifiers, action names, colors, content routing, command availability, and the PRs default
  selection; only presentation order and order-derived navigation should change.

## User-facing guidance

- Update the Artifacts quick-start card and Help keymap copy so both direct numeric jumps and previous/next cycling
  describe `Commits, Plans, Bugs, PRs`.
- Update current documentation that states the old order or old numbered mapping, including the ACE navigation guide and
  other active/current descriptions found by repository search. Keep explanatory distinctions between PRs and the three
  project-scoped panes intact.

## Behavioral and visual coverage

- Update focused Artifacts interaction tests to assert the new strip labels, active-label placement, bracket-key
  wraparound, and exact digit-to-pane mapping while retaining coverage that digits are inert outside Artifacts.
- Update runtime and fallback keymap tests plus command-catalog assertions to prove both keymap construction paths and
  palette hints agree on `1=Commits`, `2=Plans`, `3=Bugs`, `4=PRs`.
- Update quick-start/help-facing expectations and any end-to-end tests whose hard-coded numeric shortcuts select a
  particular pane.
- Run focused tests for the Artifacts scaffold, keymap bindings, keymap E2E, quick-start/help content, and any directly
  affected navigation behavior.
- Run the dedicated visual snapshot suite, inspect diffs to ensure they are limited to the intended reordered/renumbered
  Artifacts strip or related guidance, and refresh only the intentional PNG goldens.
- Because source files change, run `just install` before the repository-required `just check`; finish with a clean
  focused and full verification pass.

## Risks and guardrails

- A partial update could leave the visible strip, number keys, and palette advertising different mappings. Tests should
  assert the exact mapping at each public surface while production code continues to share one source of truth.
- Existing tests sometimes press a digit merely to reach Commits, Plans, or Bugs. Audit those call sites semantically so
  each uses the new digit without changing the behavior under test.
- Reordering the strip can affect many PNG snapshots even when pane content is unchanged. Treat broad content or layout
  movement beyond the tab labels and their active state as a regression rather than accepting it wholesale.
