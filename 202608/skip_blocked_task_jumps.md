---
tier: tale
title: Skip Blocked Tasks During Ctrl+Shift+J/K Navigation
goal:
  Ctrl+Shift+J/K navigate Ready, In Progress, and Next tasks without selecting Blocked
  tasks or changing other open-task workflows.
size: small
proposed_by: bbugyi200.athena.01m
create_time: 2026-09-09 20:00:30
status: wip
---

# Skip Blocked Tasks During Ctrl+Shift+J/K Navigation

## Goal

Change Bob Navigation Hotkeys so both `Ctrl+Shift+J` (next) and `Ctrl+Shift+K`
(previous) navigate only among real `#task` lines whose checkbox status is Ready/open
(`[ ]`), In Progress (`[/]`), or Next (`[*]`). Blocked tasks (`[?]`) must no longer be
jump targets in either direction, including when navigation wraps around the beginning
or end of a note.

The source of truth is the linked `bob-plugins` repository. The command hotkeys and the
Vim-normal-mode physical-key fallback already converge on `jumpToOpenObsidianTask` and
`getOpenTaskNavigationLines`; the change should be made in that shared target-selection
path rather than duplicating behavior in the two dispatch paths.

## Important Existing Contracts

- `OPEN_OBSIDIAN_TASK_STATUSES` and `isOpenObsidianTaskLine` intentionally treat Blocked
  (`[?]`) as open for dependency, scheduling, task-moving, and related workflows. Do not
  narrow that shared definition.
- Proper task navigation continues to require a standalone `#task` token and continues
  to ignore task-shaped text in frontmatter and fenced code blocks.
- Preserve the circular next/previous behavior, cursor movement, deferred centering,
  duplicate-dispatch guard, and existing no-target notices.
- Preserve the separately defined Pomodoro-ledger navigation behavior inside a
  `## Pomodoros` section, including its existing open and completed ledger targets.
  Pomodoro rows are not `#task` records, so this task-status restriction applies only to
  proper tagged tasks.
- Work in the `bob-plugins` checkout returned by `sase repo open`; do not edit the
  deployed vault plugin directly.

## Implementation

1. In `plugins/bob-navigation-hotkeys/main.js`, introduce an explicitly
   navigation-specific definition or predicate for the three allowed proper-task
   checkbox symbols: `[ ]`, `[/]`, and `[*]`.
2. Use that navigation-specific predicate from `getOpenTaskNavigationLines` so
   `[?] #task` lines are omitted before `getOpenObsidianTaskJumpLine` performs
   forward/backward selection and wrapping. Keep the general `isOpenObsidianTaskLine`
   behavior unchanged so Blocked tasks remain open to non-navigation workflows.
3. Keep naming/comments clear about the distinction between broadly open tasks (which
   include Blocked for dependency semantics) and active navigation targets (Ready, In
   Progress, and Next only). Export a new helper only if the focused tests need to
   exercise the classification directly.

## Regression Coverage

Extend `scripts/test-navigation-hotkeys.cjs` with focused tests that prove:

- the navigation scan includes proper `[ ]`, `[/]`, and `[*]` `#task` lines but excludes
  `[?]`, done, cancelled, unknown-status, and untagged checkbox lines;
- frontmatter and fenced-code exclusions still apply;
- existing qualifying Pomodoro-ledger targets are unchanged;
- next and previous jumps skip intervening Blocked tasks and wrap to an allowed task
  rather than a Blocked task;
- a note containing only Blocked proper tasks has no jump target, and the existing
  single-allowed-target/current-line case still returns no target; and
- `isOpenObsidianTaskLine("- [?] #task Blocked")` remains true, guarding the shared
  dependency/scheduling contract from accidental narrowing.

## Release, Validation, and Deployment

1. Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.23.0` to `1.24.0`, and
   update the Bob Navigation Hotkeys row in `README.md` to match and document that
   `Ctrl+Shift+J/K` skips Blocked tasks.
2. Run the focused navigation-hotkeys test file:

   ```bash
   node --test scripts/test-navigation-hotkeys.cjs
   ```

3. Run the complete repository checks:

   ```bash
   npm test
   npm run validate
   ```

4. From the linked `bob-plugins` checkout, preview and then deploy only this plugin from
   the modified checkout (without pulling over the implementation):

   ```bash
   bob plugins sync --no-pull --dry-run --plugin bob-navigation-hotkeys --repo .
   bob plugins sync --no-pull --plugin bob-navigation-hotkeys --repo .
   bob plugins sync --no-pull --dry-run --plugin bob-navigation-hotkeys --repo .
   ```

   The final dry run must report the managed plugin files as unchanged. Do not use
   `--force` if the vault contains protected local edits; report that as a deployment
   blocker instead. The user may need to reload Bob Navigation Hotkeys or restart
   Obsidian before the already-open application process uses the newly synced
   JavaScript.

## Acceptance Criteria

- In both regular Obsidian editing and Vim normal mode, `Ctrl+Shift+J/K` select the same
  filtered target set because both dispatch paths retain their shared implementation.
- Ready/open, In Progress, and Next proper tasks remain navigable in both directions
  with circular wrapping.
- Blocked proper tasks are never selected, including blocked-only notes and boundary
  wraps.
- Dependency and other non-navigation features still recognize Blocked tasks as open
  where they did before.
- Pomodoro navigation, parser context exclusions, cursor/scroll behavior, and notices do
  not regress.
- Focused tests, the full test suite, manifest validation, and post-sync dry-run all
  pass.
