---
tier: tale
title: Deduplicate ACE agent-completion toasts by activity generation
goal:
  Each notification activity generation produces exactly one ACE toast and bell while legitimate snooze resurfaces still
  alert once.
proposed_by: bbugyi200.athena.ra
create_time: 2026-08-01 10:01:47
status: done
---

- **PROMPT:** [202608/prompts/deduplicate_agent_completion_toasts.md](prompts/deduplicate_agent_completion_toasts.md)
- **AGENTS:**
  - [bbugyi200.athena.ra](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.ra.md)
- **COMMITS:**
  - [d2d8151](https://github.com/sase-org/sase/commit/d2d8151165ac64ca67e1a4c44fe9feb1cf648ecf) — fix: dedupe
    notification toasts by activity cursor

# Deduplicate ACE agent-completion toasts by activity generation

## Context and diagnosis

ACE can display the same agent-completion toast more than once even though the completion producer persisted only one
notification. The supplied screenshot shows two identical
`CODEX(gpt-5.6-sol) @sase-d9.2 completed: ace(run)-260801_084622` toasts. The notification store contains exactly one
matching row (`66184172-a633-4ef4-887d-6dc975dae55f`, created at 2026-08-01 09:50:01 EDT), while the durable toast log
records the same message twice in the same ACE process at 09:50:02 and 09:50:05. Other completion rows repeat with the
same pattern, which rules out duplicate agent finalization or duplicate notification writes.

The defect is in ACE's consumer-side state model. `_last_unread_ids` is used both as the current unread projection and
as the toast-delivery deduplication ledger. Current unread state is intentionally non-monotonic: notification refreshes,
acknowledgement, dismissal, muting, and cache reconciliation can remove or replace IDs. When an unchanged active
notification is observed after its ID has fallen out of that transient set, `_poll_agent_completions_once()` classifies
the same persisted event as new and emits another toast and bell. Pump-free refreshes make those state transitions able
to interleave, and serializing poll calls alone cannot make a non-monotonic unread set a reliable delivery ledger.

SASE already defines the correct delivery identity for this purpose through
`notification_activity_cursor(notification) == (resurfaced_at ?? timestamp, id)`. A notification's original activity
generation should toast once per ACE session; a later snooze expiry changes `resurfaced_at` and therefore creates one
new, intentionally deliverable generation.

## Implementation

1. In ACE app state and startup lifecycle, add a session-local set of notification activity cursors that is independent
   of `_last_unread_ids`. Seed it from the same active, unread, non-silent, non-muted startup rows used to suppress
   replay of pre-existing alerts, and return/apply that seed through the existing single-pass startup notification load
   without adding another disk read or any work to the Textual message pump.

2. In `src/sase/ace/tui/actions/agents/_notification_polling.py`, compute new toast candidates by comparing each active
   notification's canonical activity cursor with the session-local delivered-generation set. Record observed candidate
   generations before toast callbacks, bell awaits, or status-reconciliation side effects can interleave. Continue to
   maintain `_last_unread_ids` solely as the current unread projection for indicators and agent-row state, and preserve
   the existing filters for muted, silent, read, auto-dismissed, and snooze-resurfaced notifications.

3. Keep activity generations in the delivery set when unread/cache state shrinks so stale refreshes, acknowledgement,
   dismissal, or reclassification cannot replay the same event. Do not collapse solely by notification ID: when the same
   notification later has a new `resurfaced_at`, allow that new cursor to toast and ring exactly once, while older or
   already-observed generations remain suppressed.

4. Add focused regression coverage in `tests/test_notification_toast_polling.py` proving that clearing or rebuilding
   `_last_unread_ids` between identical snapshots no longer produces a second toast/bell, and that one notification ID
   with a changed resurface activity emits exactly one new reminder generation followed by no replay. Extend the startup
   lifecycle test in `tests/ace/tui/test_startup_stopwatch_live_update.py` to verify that existing active activity
   generations are seeded from the single snapshot and do not toast after startup. Update shared fake-app state/type
   annotations only where required by those production changes.

## Validation

1. Run the focused notification polling and startup tests, including the existing overlapping-poll and snooze-expiry
   matrix, to verify ordinary arrivals, auto-dismissal, mute/silent behavior, startup suppression, and legitimate
   resurface reminders retain their current behavior.
2. Run `just install` and then the repository-mandated `just check` to cover formatting, linting, type checking, and the
   full test suite.

## Acceptance criteria

- A single stored agent-completion activity generation produces at most one ACE toast and one bell per ACE session, even
  if unread IDs or notification snapshots are refreshed, cleared, or replayed.
- A snoozed notification whose durable `resurfaced_at` advances produces one new toast/bell for that new activity
  generation, and repeated reads of that generation remain quiet.
- Startup continues to suppress pre-existing active notifications without an extra store read or event-loop/message-pump
  blocking work.
- Notification counts, unread agent-row projection, dismissal/mute behavior, batching, and toast formatting remain
  unchanged.
