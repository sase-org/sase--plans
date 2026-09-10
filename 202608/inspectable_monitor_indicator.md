---
tier: tale
title: Make the top-bar monitor count directly inspectable
goal: "Preserve the truthful global running-monitor count while making the badge explain
  monitor lifetime semantics and provide a direct route to identify or stop each active
  monitor.

  "
size: small
proposed_by: bbugyi200.athena.0ak
create_time: 2026-09-09 20:00:10
status: wip
---

# Plan: Make the top-bar monitor count directly inspectable

## Outcome

The amber top-bar gear will remain a count of every active monitor shell represented by
ACE's proc projection, including sessionless monitors started by other agent families.
Hovering it will explain both the count and the important lifecycle rule that monitors
outlive their starter agents. Clicking it will open the existing SASE Admin Center
directly on the Procs tab, where the counted monitor rows can be identified, inspected,
jumped to, or stopped through their established actions.

The adjacent blue proc badge should receive the same click-through behavior and a
lane-specific tooltip so both gear chips share one predictable interaction without
blurring their distinct meanings.

## Diagnosis and invariants

- The reported `2` was an accurate snapshot, not a stale-record bug. The observer,
  `sase monitor list -j`, `sase agent list -j`, and the supervisor processes agreed on
  two nonterminal monitors: one running `just check-full` and one waiting for release
  `v0.17.0`.
- The monitor associated with the user's recent stop was already terminal with
  `monitor_state: stopped`, an exit code from SIGTERM, and no presence in the active
  monitor list. It did not contribute to the `2`.
- A monitor is deliberately a detached, family-attached proc shell. It survives the
  starter agent and ACE itself; changing that contract would break long-running
  verification and wait workflows.
- `ProcProjection.active_monitor_rows()` and `recount_projection()` are the source of
  truth for the badge. Keep their active-status, session, and origin rules unchanged;
  this plan fixes discoverability rather than changing which processes are counted.
- Opening the Procs tab must reuse the existing `open_tasks_panel` app action and cached
  proc projection. The indicator interaction must not perform disk I/O, subprocess work,
  reconciliation, or a second refresh on Textual's event loop.

## Implementation

1. Extend `ProcIndicator` and `MonitorIndicator` in
   `src/sase/ace/tui/widgets/proc_indicator.py` so each widget maintains a tooltip in
   lockstep with its cached count. Use concise singular/plural copy. The monitor copy
   must explicitly say that monitors outlive starter agents and that the Procs tab is
   where they can be inspected or stopped; the proc copy must describe ordinary
   background procs without implying they are monitors.
2. Give both widgets an async click handler that dispatches the existing
   `open_tasks_panel` app action. Do not duplicate Admin Center construction or proc
   loading in the widgets. A hidden zero-count chip should remain zero-width and should
   not alter top-bar layout.
3. Keep the existing observer split in
   `src/sase/ace/tui/actions/_proc_action_observer.py`: blue remains
   `active_count - active_monitor_count`, amber remains `active_monitor_count`, and a
   terminal or stopped monitor remains excluded automatically.
4. Update the Proc Indicator and Monitor Indicator sections in `docs/ace.md` to document
   the hover text, click-through destination, global detached-monitor scope, and the
   fact that killing or finishing a starter does not implicitly stop its monitor.

## Tests and verification

- Expand `tests/ace/tui/widgets/test_proc_indicator.py` to pin zero, singular, and
  plural tooltip copy for both lanes and to prove `set_count()` updates content and
  tooltip without any data access.
- Add mounted interaction coverage showing that clicking either badge dispatches
  `open_tasks_panel`; where practical, exercise the real app action far enough to assert
  that the Admin Center opens on `procs` rather than only mocking the widget.
- Preserve or strengthen the existing projection-split regression in
  `tests/ace/tui/test_proc_actions_session_workers.py`, including the invariant that a
  monitor is counted only in the amber lane. No PNG golden should change because
  tooltips and click handlers do not change the rendered chip; if a rendered pixel does
  change unexpectedly, inspect it rather than accepting the snapshot automatically.
- Run `just install`, the focused proc-indicator and proc-observer tests, and then
  `just check`. If scoped verification escalates or reports unusual selection, follow
  the repository guidance and run `just check-full` through `/sase_monitor`.

## Acceptance criteria

- With two active monitors, the amber chip still renders `⚙ 2`; with no active monitors,
  it remains hidden.
- Hovering the amber chip makes clear that the two entries are detached monitors that
  can outlive their starter agents and tells the user how to inspect them.
- Clicking either gear opens the Admin Center's Procs tab, whose orange rows account for
  the amber count and expose the existing monitor stop workflow.
- Stopped, completed, failed, timed-out, and lost monitors do not re-enter the active
  badge, and no monitor is stopped merely because its starter agent exits or is killed.
- The interaction introduces no synchronous store read, subprocess, or refresh path on
  the TUI event loop.

## Non-goals

- Do not stop either of the currently legitimate monitors as part of implementation.
- Do not couple agent termination to monitor termination or change follow-up launch
  policy.
- Do not rescope the global monitor count to only the visible Agents rows, the current
  query, or the current ACE session.
- Do not redesign the Procs pane or monitor-row stop confirmation.
