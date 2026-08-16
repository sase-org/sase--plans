---
tier: tale
title: Restore prompt Agents-tab freshness without broad reloads
goal:
  Completion notifications promptly update terminal and unread agent state through
  bounded exact refreshes on every tab.
size: medium
proposed_by: bbugyi200.athena.03d
create_time: 2026-08-16 09:38:22
status: wip
---

# Restore prompt Agents-tab freshness without broad reloads

## Outcome

Make completion state and the Agents-tab unread indicator converge promptly after a
notification, including while the user is viewing another tab and while the completed
agent is hidden by a fold or search. Preserve the existing event-loop protections and
the terminal-only unread invariant, but keep the common path bounded to the changed
agent instead of reloading the complete agent history.

## Confirmed diagnosis

The live TUI confirms the reported lag. A completion toast was emitted at 09:19:52,
while its notification-triggered agent reconciliation did not finish until 09:20:01.
Across the same session, notification-sourced broad loads spent 4.18-5.21 seconds in the
disk stage, with an earlier notification load taking 19.44 seconds. Preparation and UI
application remained only about 40-90 milliseconds. The session held roughly 500 loaded
agents, and the stall watchdog did not report a corresponding event-loop stall. The slow
part is stale data reconciliation, not painting the unread marker.

The delay has two cooperating causes:

1. `EventAutoRefreshMixin._run_auto_refresh_body` defers every Agents refresh while a
   different tab is active, including already-bounded artifact deltas. A completion
   notification can therefore toast immediately while the cached row still says
   `RUNNING`; the terminal-only unread projection correctly refuses to mark it until a
   later tab switch or sanity refresh updates that row.
2. Notification targeting in `find_agent_for_notification` and
   `_completion_notification_delta_dirs` searches `app._agents`, the currently
   visible/folded/filtered projection. Other unread code correctly uses the complete
   `app._agents_with_children` roster. A hidden completed agent is consequently treated
   as missing and routed to a broad Tier-1 load of all roughly 500 rows.

A secondary amplifier is the direct notification provider. The store is currently about
17.8 MB / 7,200 JSONL rows, and one snapshot parse takes about 0.18 seconds.
`_refresh_notification_count_async` performs a count read and then another full read,
even though both direct reads parse the same store and the full snapshot already
contains counts/tabs. Its coalescing guard is also separate from completion polling, so
overlapping triggers can duplicate this work. The Rust core was inspected and ruled out
as the cause: ordinary current-snapshot reads rewrite only when a snooze actually
expires.

## Implementation

### 1. Keep exact lifecycle deltas live off-tab

- Refactor the auto-refresh decision in
  `src/sase/ace/tui/actions/event_refresh/_auto_refresh.py` so the tab gate defers only
  broad Agents loads. Allow a queued, bounded artifact-delta request to run while
  another tab is active.
- Preserve the current navigation/input gates, delta queue bound, load coalescing, and
  60-second broad sanity reconcile. If an exact off-tab delta cannot be applied, retain
  dirty state for the next Agents-tab entry or sanity pass instead of escalating to an
  immediate off-tab broad load.
- Make the completion race converge in either order: a terminal marker arriving first
  updates the cached row and the later notification projects unread; a notification
  arriving first remains cached and the later exact delta projects unread during load
  finalization.

### 2. Resolve notification targets from the complete loaded roster

- Introduce one shared notification-agent roster helper and use it from
  `src/sase/ace/tui/actions/agents/_notification_navigation.py` and
  `_notification_utils.py`. Prefer `_agents_with_children`, fall back to `_agents`, and
  exclude synthetic clan-container rows where a real agent is required.
- Match exact `raw_suffix`/timestamp identities before any cl-name-only compatibility
  fallback, retain the existing root/child matching semantics for plan and question
  notifications, and deduplicate artifact directories before scheduling a delta.
- For a completion whose agent is already loaded but folded or filtered, schedule one
  artifact delta and never call `request_agents_refresh`. Reserve the broad fallback for
  a genuinely absent/new agent that cannot yet be resolved from cached state.
- Keep `_reconcile_unread_from_completion_notifications` terminal-only. Do not create a
  transient unread `RUNNING` row; promptness should come from fixing the data path.

### 3. Make notification snapshot refresh single-pass and single-flight

- Consolidate completion polling and scheduled snapshot/count refreshes around one
  guarded snapshot loader in `src/sase/ace/tui/actions/agents/_notification_polling.py`
  and `_notification_provider.py`. A direct full snapshot supplies notifications,
  counts, tabs, and the snooze deadline, so do not precede it with a second count-only
  parse.
- Reuse an in-flight or just-produced snapshot for overlapping poll/count triggers,
  retaining at most one trailing refresh when the notification store changes during a
  read. Keep UI mutation on the Textual thread and store parsing in pump-free worker
  work.
- Preserve snooze-expiry atomicity, activity-cursor toast deduplication, bell ordering,
  modal/provider metadata, and external notification mutations. Do not change the Rust
  notification-store wire contract unless implementation evidence contradicts the
  confirmed read behavior above.

### 4. Add regression and performance evidence

- Extend notification routing/polling tests to cover a terminal child hidden behind a
  collapsed clan and an agent excluded by the visible search projection. Assert that
  each resolves through `_agents_with_children`, schedules one exact delta, and never
  schedules a broad refresh.
- Extend event-refresh tests with both completion race orders while the current tab is
  not Agents. Assert that exact delta work runs off-tab, broad work remains deferred,
  and load finalization projects the cached completion notification into the unread set
  and refreshes the Agents panel title.
- Add concurrency tests using barriers/events to overlap a watcher poll with a scheduled
  count refresh. Assert one provider parse per generation, one bounded follow-up for a
  mid-read mutation, and no lost indicator or snooze-deadline update.
- Add structured trace fields/events for notification-to-agent targeting and completion
  freshness: resolved versus deferred/broad, data cost, target count, and elapsed time
  from notification observation through delta application/unread projection. Avoid raw
  prompt content and full filesystem paths.
- Exercise a large synthetic roster and notification store in tests or the existing
  hot-path harness. The regression criterion is structural rather than a brittle CI
  wall-clock threshold: a one-agent completion must record `artifact_delta_load`, no
  `tier1_broad_load`, no full display rebuild, and no duplicate notification parse.

## Verification

1. Run the focused notification polling, unread projection, notification status routing,
   artifact-delta loader, and event-refresh dirty-flag suites.
2. Run the large-roster/store performance regression and inspect the structured trace to
   verify the path is notification observation -> exact artifact delta -> terminal
   unread projection, without a broad load.
3. Run `just install` before repository verification, then run `just check`.
4. In a traced ACE session, confirm that a completion received while Artifacts or Axe is
   active updates the Agents-tab unread indicator after the exact delta rather than
   waiting for the 10-second auto-refresh tick, the 60-second sanity pass, or an Agents
   tab switch. Compare `tui_agent_loads.jsonl`, `tui_trace.jsonl`, and
   `tui_stalls.jsonl` against the diagnostic baseline above.

## Acceptance criteria

- A completion for an already-loaded agent updates terminal state and unread state via
  an exact artifact delta regardless of the active tab.
- Folded and filtered agents are resolved from the complete loaded roster; their
  completion path does not perform a full Tier-1 agent load.
- Broad agent loads remain tab-gated unless the safety-net sanity refresh is due or the
  target truly is not loaded.
- Overlapping notification triggers do not perform duplicate direct-store parses, and
  snooze expiry/toast/count behavior remains unchanged.
- The focused suites and `just check` pass, and the structural performance assertions
  show no broad-load or full-display fallback for the reproduced case.

## Non-goals

- Changing unread semantics for non-terminal rows.
- Removing the periodic broad sanity reconcile.
- Reformatting the Agents UI or changing notification presentation.
- Replacing or redesigning the Rust notification store; its conditional snooze rewrite
  behavior is already correct for this incident.
