---
tier: tale
title: Reduce live notification store size
goal:
  Dismissed rows leave the live store promptly and wait-check payloads stay bounded
  while the notification UI retains useful evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1d7.13
bead: sase-1d7.13
create_time: 2026-09-30 17:15:22
status: wip
---

- **PARENT:**
  [202609/unread_ack_reliability_and_tui_responsiveness.md](https://github.com/sase-org/sase--plans/blob/main/202609/unread_ack_reliability_and_tui_responsiveness.md)
- **BEAD:**
  [sase-1d7.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d7/sase-1d7.13.md)

# Notification store retention and wait-check payload diet

## Goal and scope

Complete phase `sase-1d7.13` of the unread-ack epic. Keep dismissed notifications
available in the live store for three days, then archive them during existing
compaction. Bound `wait_checks` notification rows to the newest 32 `+1` entries while
preserving `plus_ones_dropped`, and remove redundant `action_data` from the wait-check
producer without losing the information shown in notes and files. Do not add a user
setting: the retention window is a storage policy and there is no identified per-user
tuning need.

This is one bounded follow-up implementation across the linked `sase-core` repo and
sase's wait-check producer, so it is a tale plan. The shared store policy belongs in
Rust; the producer's presentation payload belongs in Python. No new binding or wire
schema is needed.

## Evidence and choice of limits

On 2026-09-30, a read-only sample of the configured live `notifications.jsonl` found
3,934 rows and 19,144,508 bytes. Dismissed unsnoozed rows older than 1, 2, 3, 5, and 7
days account respectively for 12,002,828, 11,455,012, 10,311,676, 7,621,063, and
5,312,680 bytes. A three-day window retains several days of recent dismissed history and
makes about 10.3 MB eligible for archive. `wait_checks` contributes 179 rows, 5,030,439
bytes, and 22,923 stored `+1` entries; 73 rows exceed 32 entries. Keeping their newest
32 entries would remove about 4.16 MB before overlap with archiving. The existing
`action_data` on those rows is only about 58 KB, so trimming it is a consistency and
future-growth measure; the `+1` cap does the heavy lifting. A JSON re-encoding
projection for three-day retention plus the 32-entry cap yields about 4.94 MB live (74%
below the sampled 19.14 MB). This is a projection, not a measured post-change result.

## Implementation

1. In `sase-core`, update `crates/sase_core/src/notifications/store.rs` to use three
   days for `notification_should_archive`, preserving its dismissed-only,
   unsnoozed-only, and activity-time (`resurfaced_at` fallback to `timestamp`) rules.
   Keep archived rows in `notifications-archive.jsonl`. Confirm that a compaction pass
   is idempotent and a second read does not append duplicates.
2. Introduce a sender-specific live cap of 32 for `wait_checks` in Rust alongside the
   existing general 500-entry cap. Apply it on `+1` upsert and while compacting an
   existing large store so legacy live wait-check rows shrink without waiting for a
   future upsert. Increment `plus_ones_dropped` by exactly the removed entry count with
   saturation, preserve the newest entries in order, and make the compaction result
   report any row normalization so the caller writes once and bumps the store generation
   even if no row was archived. Preserve the existing 500 cap for other senders. Ensure
   ordinary reads do not rewrite an already normalized store.
3. In `src/sase/scripts/_chop_wait_checks_terminal.py`, keep the notes and files that
   the notification modal displays and the recovery instructions they contain. Remove
   only redundant `action_data` keys whose values are already present in those fields;
   inspect all consumers first. Keep any key that drives a user action or is otherwise
   consumed. Update producer tests that currently assert the redundant keys.
4. Make a private copy of a realistic notification store for a before/after measurement.
   Run the new Rust compaction behavior on the copy, never on the production store.
   Record byte size, row count, archived count, and the `wait_checks` contribution
   before and after in a note on `sase-1d7.13`. Delete the disposable copy after the
   measurements. Compare with the projection above and investigate a large difference.
5. Add focused Rust tests for the three-day boundary, old dismissed versus
   active/snoozed rows, archival preservation, legacy `wait_checks` trimming,
   newest-entry order, exact dropped-count accounting, unchanged non-wait-check cap,
   generation/idempotence, and no duplicate archive append. Add or update Python tests
   to show the notification's visible notes, files, recovery command, search data, and
   `+1` badge/count remain meaningful after the producer change.

## Verification and completion

Run targeted tests while iterating, then `sase tool run check` in both changed repos (as
the epic requires). Follow the sase lint-and-test reference note before finishing. If a
check fails identically on the clean base tree, record a `PROPOSED FOLLOW-UP:` note on
this phase and close it anyway. Run `sase bead epic-symbols sase-1d7.13`, resolve any
phase symbol or re-key it to an open bead, then close only `sase-1d7.13` with a note
describing verified behavior and measured before/after sizes. Do not close the epic or
create follow-up beads.
