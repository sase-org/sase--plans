---
tier: tale
title: Make the dashboard BLOCKED badge match the blocked task list
goal:
  The BLOCKED chip in dash.md counts the same dependency-blocked or custom-Blocked tasks
  as blocked.md without changing the active queue counts.
create_time: 2026-09-09 19:53:12
status: wip
---

# Make the dashboard BLOCKED badge match the blocked task list

## Goal

Make the `BLOCKED` count chip in `~/bob/dash.md` report the same set of tasks as the
`BLOCKED Tasks` query in `~/bob/blocked.md`. With the current vault contents, both must
report 90 tasks rather than the dashboard's stale dependency-only count of 12. Preserve
the existing WIP, NEXT, and READY counts and their rule that future-scheduled tasks do
not appear in those active queues.

## Root Cause

The recently updated `blocked.md` Tasks query includes either of the two supported
blocking reasons:

```tasks
(is blocked) OR (status.name includes Blocked)
```

The inline DataviewJS widget in `dash.md` still implements the old definition. It first
removes all future-scheduled tasks and then counts only tasks for which the Tasks API's
dependency-derived `isBlocked(all)` is true. The live vault currently has 12
dependency-blocked tasks, 90 tasks with the custom Blocked status, and 90 in the union,
which exactly explains the badge/list discrepancy.

The supplied screenshot currently depicts the SASE agent UI and macOS app switcher, not
the Obsidian badge, so acceptance must be based on the live note definitions,
deterministic query counts, and a fresh rendered view of `dash.md`.

## Scope

- Edit the inline DataviewJS task-count widget in `~/bob/dash.md`.
- Do not change the CLI task parser, `bob task-status-hooks`, the Navigation Hotkeys
  plugin, custom status configuration, or `~/bob/blocked.md`; those already produce and
  select the intended Blocked statuses.
- Do not remove the future-schedule cutoff from the WIP, NEXT, or READY Tasks queries or
  from their badge-count counterparts.

## Implementation

1. Separate the widget's common vault visibility rules from its two consumers:
   active-dashboard counts and the external blocked-list count. Continue excluding
   template content, `#hide` tasks, and the appropriate query note itself.
2. Retain the current effective-day schedule cutoff and dependency-blocking exclusion
   for the WIP, NEXT, and READY candidate set so their behavior does not change.
3. Build the BLOCKED candidate set without the future-schedule cutoff, because a future
   inline schedule is now intentionally represented by the custom Blocked status. Make
   its exclusion context match the destination query (`blocked.md`) rather than
   accidentally inheriting `dash.md`'s active-queue context.
4. Count a BLOCKED candidate when either:
   - `task.isBlocked(all)` reports an unresolved Tasks dependency; or
   - `task.status.name.includes("Blocked")` reports the custom `[?]` status.

   Keep this predicate visibly aligned with the query in `blocked.md`, including union
   semantics so a task satisfying both reasons is counted only once.

5. Preserve the existing plugin-unavailable fallback (`–`), count-chip styling,
   accessibility labels, hover behavior, and navigation to `blocked.md`.

## Validation

1. Before and after the edit, record deterministic live-vault baselines:
   - `bob query --format json --tasks '(is blocked)' --origin blocked.md` returns 12.
   - `bob query --format json --tasks 'status.name includes Blocked' --origin blocked.md`
     returns 90.
   - `bob query --format json --tasks '(is blocked) OR (status.name includes Blocked)' --origin blocked.md`
     returns 90.
2. Syntax-check the edited DataviewJS block without writing generated artifacts.
3. Run `bob query --format json --tasks-note dash.md` and verify all three Tasks blocks
   still execute without warnings and retain their pre-change result sets (currently WIP
   0, NEXT 5, READY 14).
4. Run `bob query --format json --tasks-note blocked.md` and verify its single block
   still returns 90 tasks with no warnings.
5. Refresh or reopen `dash.md` in Obsidian and verify:
   - the BLOCKED chip renders `90`;
   - WIP, NEXT, and READY remain unchanged;
   - activating the BLOCKED chip opens `blocked.md`; and
   - the destination renders the same 90-task set.

## Acceptance Criteria

- The dashboard BLOCKED chip and `blocked.md` use equivalent dependency-or-status
  blocking semantics and display the same count.
- Future-scheduled `[?]` tasks contribute to BLOCKED even though they remain absent from
  WIP, NEXT, and READY.
- Tasks blocked for both a dependency and a future schedule are counted once.
- Hidden/template tasks and query-note self-content remain excluded.
- No unrelated vault, CLI, or plugin behavior changes.
