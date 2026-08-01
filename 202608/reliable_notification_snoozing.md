---
tier: epic
title: Reliable notification snoozing and resurfacing
goal: 'Snoozed notifications use one durable time contract, resurface as visible unread
  activity at the requested deadline, and are delivered and ordered consistently across
  ACE, CLI, mobile gateway, and Telegram consumers.

  '
phases:
- id: core-expiry
  title: Canonical snooze state and expiry contract
  depends_on: []
  size: medium
  description: 'core-expiry: make the Rust notification store own validated deadlines,
    atomic active-row expiry, resurface metadata, and next-deadline projection.'
- id: ace-deadlines
  title: Deadline-driven ACE reminders
  depends_on:
  - core-expiry
  size: medium
  description: 'ace-deadlines: add a pump-free nearest-deadline coordinator that remains
    reliable across refresh settings, restarts, suspend, clock changes, and notification
    mutations.'
- id: downstream-resurface
  title: Cross-surface resurface ordering and delivery
  depends_on:
  - core-expiry
  size: medium
  description: 'downstream-resurface: adopt current-state reads and resurface activity
    cursors in CLI, mobile gateway, and Telegram projections so old snoozed rows become
    new visible activity.'
- id: regression-docs
  title: End-to-end regression matrix and documentation
  depends_on:
  - ace-deadlines
  - downstream-resurface
  size: small
  description: 'regression-docs: verify state, timing, ordering, concurrency, and
    downstream delivery together and document the resulting guarantees and recovery
    behavior.'
proposed_by: bbugyi200.athena.qu
create_time: 2026-08-01 06:45:47
status: wip
bead_id: sase-cy
---

- **PROMPT:** [202608/prompts/reliable_notification_snoozing.md](prompts/reliable_notification_snoozing.md)
- **BEAD:** [sase-cy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-cy/README.md)

# Plan: Reliable Notification Snoozing and Resurfacing

## Why this is an epic

The snooze picker and basic persistence work today, but reliable expiry crosses three independently released
repositories and several long-lived consumers. The shared Rust store must define the state transition first. ACE and
downstream delivery can then adopt that contract in parallel, followed by one phase that tests the seams between them.
Trying to repair only ACE polling would leave CLI, mobile pagination, and Telegram high-water behavior inconsistent.

Implementers working in the linked `sase-core` or `sase-telegram` repositories must open each repository with the
`/sase_repo` workflow before reading or changing it. Do not use workspace-specific paths in code, tests, or docs.

## Audit findings

The targeted baseline suites currently pass (`63 passed` across store, ACE polling/picker, and mobile bridge tests), but
the following behaviors are either wrong or untested:

| Area                    | Current behavior                                                                                                                                                                                                         | Consequence                                                                                                                                               |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ACE scheduling          | `_poll_agent_completions()` is the only ACE expiry pass. With inotify active, a deadline creates no file event, so a clean store waits for the 60-second sanity poll; `--refresh-interval 0` prevents the poll entirely. | A live ACE session can resurface late or never.                                                                                                           |
| Startup and wake        | Startup seeds counts from a non-expiring snapshot, and no deadline-specific task exists. Monotonic timers alone also do not account for wall-clock jumps or a suspended host.                                            | Expired reminders can remain muted after restart/resume until unrelated refresh work occurs.                                                              |
| Core expiry             | `read_notifications_snapshot_expiring_snoozes()` scans dismissed rows and reports their IDs, clears mute/deadline but preserves `read`, and exposes no next deadline or durable resurface event.                         | Dismissed rows can ring later; a snoozed row marked read can fail to re-enter the inbox; only the first process to mutate the row observes `expired_ids`. |
| Mutation feedback       | Single-row snooze persistence runs synchronously from the modal callback and the UI updates even if the store reports no matching row.                                                                                   | The message pump can block, and stale rows can show a false success.                                                                                      |
| Time arithmetic         | Relative deadlines are computed as configured-zone wall time plus a `timedelta`. Across the 2026 New York DST transitions, a requested four hours currently produces three or five elapsed hours.                        | Relative presets do not always expire after the requested duration.                                                                                       |
| Ordering and pagination | ACE, the CLI catalog, the Python mobile bridge, and the Rust gateway all sort/filter by the original `timestamp`.                                                                                                        | An old resurfaced row remains buried or can fall outside a limited/newer-than page.                                                                       |
| Telegram delivery       | The outbound chop reads raw rows and advances a timestamp-only high-water mark based on the original `timestamp`.                                                                                                        | A previously delivered old row is not eligible for a new snooze-expiry reminder.                                                                          |

Baseline verification note: the snooze-focused baseline is green, but the repository-wide `just check` currently fails
in `test_write_sdd_files_supports_flat_sidecar_plans_root` and `test_write_sdd_files_rebases_seeded_parent_section`.
Both fixtures create tale plans without the newly required `title` and `goal`; the parallel run also reported a leaked
`/var/tmp/.../opencode` entry. These failures predate and are unrelated to this scratch plan. Recheck them before final
verification and follow the project’s discovered-work policy if they remain; do not weaken the notification acceptance
criteria or silently attribute them to this epic.

## Target contract

Keep `timestamp` as the immutable original creation time. Add durable resurface activity metadata (use a name such as
`resurfaced_at`) and a single shared helper for the effective activity key:

```text
activity_at(notification) = resurfaced_at ?? timestamp
activity_cursor(notification) = (activity_at, notification_id)
```

The ID tie-breaker is required anywhere a high-water cursor is persisted so two notifications with the same timestamp
cannot hide one another. New fields must default cleanly for legacy JSONL rows and remain additive for existing readers.

The canonical state transitions are:

| Event             |   `muted` | `snooze_until`                 |    `read` | `resurfaced_at` | Delivery result                                       |
| ----------------- | --------: | ------------------------------ | --------: | --------------- | ----------------------------------------------------- |
| Snooze active row |    `true` | validated future UTC instant   | unchanged | unchanged       | hidden from active delivery until due                 |
| Resnooze          |    `true` | replacement future UTC instant | unchanged | unchanged       | only the replacement deadline is scheduled            |
| Expire active row |   `false` | `null`                         |   `false` | expiry instant  | one new activity generation becomes visible           |
| Explicit unmute   |   `false` | `null`                         | unchanged | unchanged       | timer is cancelled, not treated as an expiry          |
| Dismiss           | unchanged | `null`                         | unchanged | unchanged       | pending snooze is cancelled and can never alert later |

Snooze writes must accept only timezone-aware, parseable, future instants and normalize them to a canonical UTC RFC-3339
representation before persistence. Relative choices represent exact elapsed seconds and therefore add their duration on
the UTC timeline. Calendar choices such as “tomorrow at 09:00” resolve in the configured IANA timezone first and then
convert to UTC. A legacy malformed or timezone-naive deadline must not remain silently muted forever: the current-state
reconciliation path should conservatively resurface it immediately and report it as an expiry, while all new mutation
paths reject such values.

Raw/audit reads may remain non-mutating, but user-facing “current inbox” reads must use one clearly named store API that
atomically expires due rows under the existing lock before projecting counts, rows, `expired_ids`, and the next active
snooze deadline. This distinction avoids scattering boolean flags whose omission recreates the bug.

## Phase details

### 1. Canonical snooze state and expiry contract (`core-expiry`)

In the linked `sase-core` repository, extend the notification store/wire implementation in
`crates/sase_core/src/notifications/{wire,store}.rs` and its Python bindings so that the core owns all temporal
semantics:

- Add the additive resurface activity field to `NotificationWire` and add the earliest active snooze instant to store
  snapshot/update metadata. Compute both counts and the next deadline from the post-mutation row set.
- Validate and normalize single and bulk snooze deadlines before changing any row. A rejected bulk request must remain
  atomic. Reject snoozing missing/dismissed rows through the existing outcome/error conventions.
- Make single/bulk dismissal clear `snooze_until`. During expiry, ignore dismissed rows, transition every due active row
  to unmuted and unread, stamp one resurface activity instant for the batch, and return only rows that actually made
  that transition in `expired_ids`.
- Make malformed/naive legacy deadlines follow the explicit recovery rule above, and ensure permanent mutes (no
  deadline) remain untouched.
- Preserve lock/tempfile atomicity and counts-only binding behavior. Two concurrent expiring reads must converge on one
  store state without corrupting appended rows; `expired_ids` remains transition metadata, while the persistent unread
  and resurface fields let every later consumer observe the result.

Update `crates/sase_core/tests/notification_store_parity.rs`, binding tests in `crates/sase_core_py`, and the main
repository’s `src/sase/core/notification_store_{wire,facade}.py` plus facade/store tests. Cover offset-equivalent
deadlines, validation failures, legacy recovery, read-to-unread expiry, dismissal cancellation, earliest-deadline
selection, equal activity timestamps, and a concurrent append/expire race.

### 2. Deadline-driven ACE reminders (`ace-deadlines`)

Replace ACE’s dependence on unrelated refresh cadence with one notification-specific coordinator:

- Track at most one Textual timer for the nearest deadline reported by the current snapshot. The timer callback must be
  thin and synchronous; it may compare cached wall-clock/epoch values, then launch the existing notification read and UI
  application through a coalesced pump-free task. Do not perform disk I/O or await worker I/O on Textual’s serial
  message pump.
- While a snooze exists, cap the in-memory wall-clock recheck interval (target at most one second) so suspend/resume and
  forward/backward clock changes re-evaluate the authoritative UTC deadline promptly without polling the JSONL each
  second. Once due, perform one current-state snapshot read, apply counts/toasts/status projections, and schedule the
  next future deadline.
- Start the coordinator after first paint and keep it active even when `--refresh-interval 0` disables general
  auto-refresh. Cancel its timer/task during normal and controlled teardown.
- Route notification-file watcher events, startup reconciliation, modal snooze/resnooze/unmute/dismiss completions, and
  ordinary polling through the same coalescing guard. An external store mutation must be able to replace or cancel the
  cached nearest deadline immediately rather than waiting for the general refresh tick.
- Preserve alert policy: one toast and one tmux bell per observed resurface batch, including rows that were marked read
  while snoozed; no repeat on subsequent polls; no bell for cancelled, dismissed, permanently muted, or not-yet-due
  rows. Persistent `resurfaced_at` plus unread state should make a transition observable even when another process won
  the expiry lock.
- Compute duration presets on the UTC timeline and calendar presets in configured local time. Keep display formatting
  local and leave the original sent-at timestamp intact.
- Move single-row snooze persistence onto the same tracked/background mutation pattern used by bulk actions. Apply the
  modal state and success toast only when the expected row was matched; on validation, I/O, or stale-row failure,
  retain/reload authoritative state and show an actionable error.

Add deterministic fake-clock/timer coverage around `_notification_polling.py`, event-refresh/watcher routing, startup,
lifecycle cleanup, and modal mutation completion. Required cases include refresh disabled, clean inotify state, multiple
deadlines, resnooze to earlier/later, unmute/dismiss cancellation, poll/timer overlap, process A expiring before process
B, startup with a past deadline, simulated suspend/clock jump, and spring-forward/fall-back exact elapsed durations.
Keep existing TUI performance constraints: no scaled startup work, no blocking pump callbacks, and no full agent-list
rebuild merely because a snooze expired.

### 3. Cross-surface resurface ordering and delivery (`downstream-resurface`)

Adopt the core’s current-state and activity cursor consistently outside ACE:

- In the main repository, route `sase notify list/show` and Python mobile notification list/detail projections through
  the current-state read. Sort and apply `newer_than`/`limit` using `(activity_at, id)` while still displaying the
  immutable original sent time and exposing the optional resurface time.
- In the linked `sase-core` repository’s gateway, update the mobile wire/schema and route filtering, ordering,
  `next_high_water`, and event payloads to use the activity cursor. A list read that performs an expiry must publish a
  notification-change event after the lock is released so connected clients can refresh without a second unrelated
  action. Preserve backward-compatible parsing for the prior timestamp-only cursor during rollout.
- In the linked `sase-telegram` repository, replace raw `load_notifications()` selection with the current-state
  snapshot, exclude still-muted snoozes, and migrate the timestamp-only last-sent marker to a versioned
  `(activity_at, id)` cursor. A notification delivered before it was snoozed becomes eligible exactly once when its
  later resurface generation appears. Preserve first-run backlog suppression and atomic marker writes.
- Order outbound events oldest-to-newest by the activity cursor and do not advance past an event that failed to send;
  this is necessary for the new cursor to avoid losing one of several simultaneous resurface events. Do not expand this
  phase into a general Telegram retry-queue redesign.

Add CLI JSON/pretty tests, bounded-page/newer-than mobile tests, Rust gateway contract snapshots, Telegram cursor
migration tests, and scenarios for a snooze suppressed before initial Telegram delivery versus a previously delivered
notification resurfacing later.

### 4. End-to-end regression matrix and documentation (`regression-docs`)

Exercise the completed contract through real store bindings rather than mocks alone:

1. Create active, read, dismissed, silent, and permanently muted notifications; snooze them at distinct and identical
   deadlines; advance a controlled clock; and assert the exact persisted JSONL state, counts, activity ordering, and
   next deadline after each transition.
2. Run concurrent ACE-style and mobile/Telegram-style reads around one deadline and assert idempotent storage, no
   dismissed reminder, persistent unread resurfacing, one delivery per consumer cursor, and no later duplicate.
3. Verify an old notification resurfaces into the first limited ACE/CLI/mobile page and crosses Telegram’s migrated
   cursor without changing its original sent timestamp.
4. Run the relevant Rust workspace tests, `sase-telegram` checks, focused Python suites, then `just install` followed by
   the main repository’s mandatory `just check`. If notification UI rendering changes, run the dedicated visual suite
   and update snapshots only for intentional output.

Update `docs/notifications.md`, `docs/ace.md`, `docs/integrations.md`, `docs/rust_backend.md`, the mobile gateway
contract/runbook, and `sase-telegram` documentation. State the timing guarantee precisely: while a supporting long-lived
consumer is active, the notification becomes current within the coordinator tolerance; if every consumer is offline, the
first later current-state read atomically catches up. Document exact elapsed versus calendar-time choices, dismiss/read
semantics, activity ordering, cursor migration, malformed legacy recovery, and the fact that raw audit reads
deliberately do not mutate time-driven state.

## Acceptance criteria

- A 15-minute, one-hour, or four-hour relative snooze expires after that exact elapsed duration across both DST
  transitions; “tomorrow morning” remains 09:00 in the configured timezone.
- A live ACE process resurfaces within one second of the wall-clock deadline even with clean inotify flags or
  `--refresh-interval 0`, and catches up promptly after suspend, restart, or a system-clock change.
- Expiry atomically clears mute/deadline, marks the active row unread, records resurface activity, updates indicator
  buckets, emits one toast/bell batch, and schedules the next deadline.
- Dismissed or explicitly unmuted snoozes never create a later snooze-expiry generation or reminder; a transport’s
  separately defined policy for an original, never-delivered event remains independent. Permanent mutes remain muted.
  Resnoozing replaces rather than duplicates the deadline.
- Store failures or stale notification IDs never produce a false “Snoozed” success state in the modal.
- An old resurfaced row is first-class recent activity for ACE, CLI, mobile pagination/newer-than queries, and Telegram
  delivery while retaining its immutable original creation timestamp.
- Concurrent consumers converge without JSONL loss or duplicate state transitions; consumer-specific cursors prevent
  lost equal-timestamp events and repeated Telegram delivery.
- New writes cannot create malformed or naive deadlines, and legacy bad deadlines surface visibly instead of remaining
  silently snoozed forever.
- All repository-specific checks and the main repository’s `just check` pass.

## Non-goals

- Per-sender recurring snooze rules, business-hours calendars, or natural-language dates beyond the existing choices.
- A general notification retry queue or cross-machine state synchronization redesign.
- Changing the immutable meaning of the original notification `timestamp`.
- Guaranteeing an audible alert while no ACE, gateway, Telegram chop, or other notification consumer is running; the
  durable guarantee in that case is immediate catch-up on the next current-state read.
