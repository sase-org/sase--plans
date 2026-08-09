---
tier: tale
title: Selected notification snooze countdown
goal:
  Make the selected snoozed notification's remaining sleep and wake time immediately
  visible, trustworthy over time, and visually at home in the ACE notification panel.
size: medium
proposed_by: bbugyi200.athena.vp
create_time: 2026-08-08 11:14:26
status: wip
---

# Plan: Selected notification snooze countdown

## Outcome

When a notification with `snooze_until` is highlighted, the detail pane will show a
dedicated wake-status line immediately below the existing sent/filed metadata. The line
will make the relative answer prominent while retaining an absolute wake instant for
trust and disambiguation, for example:

`☾ Snoozed · wakes in 5d 23h · Fri Aug 14 at 10:39 EDT`

The line will occupy no space for an ordinary notification. It will update while the
modal remains open, follow keyboard and mouse selection immediately, and never disturb
the selected row, attachment scroll position, or gate/report detail content.

This is a presentation-only change. `Notification.snooze_until` already carries the
authoritative deadline, so do not add a store read, provider request, Rust wire field,
or new notification refresh path.

## Current behavior and constraints

- `NotificationOptionMixin._create_styled_label()` already appends a compact snooze
  badge, but it appears after the message, age, action/file metadata, and tags. The
  40%-width list clips that suffix for long rows, as in the supplied screenshot, so it
  cannot answer the question reliably for the selected item.
- `NotificationAttachmentMixin._display_file()` is the common detail-rendering entry
  point for plain, attachment, question, report, and gate notifications. It already
  updates `#notification-sent-at` before branching, which establishes the right pattern
  for selection metadata shared by every pane type.
- Countdown refreshes must honor the TUI performance rules: timer callbacks stay
  synchronous and in-memory, update only the dedicated label, and must not rebuild the
  `OptionList`, rerender the attachment body, read disk, or call a provider.
- The existing compact row badge remains useful when enough width is available. Keep it
  and its current ordering/semantics; this feature makes the selected-row answer
  dependable rather than redesigning the list.

## Design and implementation

### 1. Add a focused snooze-status presentation helper

Create a small notification-modal mixin/module dedicated to selected snooze status,
rather than overloading sent-time semantics. Give it pure, clock-injectable helpers that
parse `snooze_until` through the project's configured-timezone utilities and build the
Rich `Text` line.

Use the following display contract:

- A valid future deadline renders `☾ Snoozed · wakes in <remaining> · <wake instant>`.
- Make `<remaining>` the visual focal point (bold, restrained warm accent); keep the
  state label and absolute instant quieter. Use the existing single-cell Snoozed moon
  and deferred-state visual language so the line relates naturally to the Snoozed tab
  without looking like an error or an actionable gate.
- Render remaining time with at most two useful units: days plus hours, hours plus
  minutes, minutes alone, and `<1m`. Derive both units from the same captured `now` so a
  render cannot straddle a boundary and contradict itself. Do not round an almost-seven-
  day interval down to the misleading single token `6d`; the second unit should preserve
  the useful precision.
- Render the absolute instant in configured local time with friendly tiers: `today`,
  `tomorrow`, weekday/date for a later same-year wake, and a year when needed. Include
  the timezone abbreviation so DST and remote timestamps remain unambiguous.
- If the deadline has just elapsed but the modal still holds the pre-expiry row, show
  `☾ Snoozed · waking now…` (plus the wake instant when available), never a negative
  duration or a stale positive countdown.
- If a legacy or malformed non-empty `snooze_until` cannot be parsed, show a subdued
  `☾ Snoozed · wake time unavailable` instead of crashing, exposing an arbitrary raw
  value, or silently claiming the row is not snoozed.
- A notification with no `snooze_until` produces no status renderable and hides the
  widget completely.

Keep these helpers local to Python presentation code. Do not change the globally used
`format_relative_until()` contract just to obtain the richer two-unit detail treatment;
that formatter also feeds indicator and gate surfaces whose compact snapshots are not
part of this feature.

### 2. Place the status in the shared detail metadata stack

In `NotificationModal.compose()`, mount a hidden one-line label directly after
`#notification-sent-at` and before `#notification-file-scroll`. Style it in
`styles.tcss` with a fixed one-row height when visible, no wrapping, and ellipsis as a
narrow-terminal fallback. The global `hidden` class must remove it from layout so
non-snoozed selections do not gain a blank row.

Mix the new behavior into `NotificationModal`, and invoke its update method at the top
of `_display_file()` alongside `_update_sent_at()`. That one integration point must
cover initial mount, highlight changes, tag changes, list rebuilds after mute/snooze
mutations, attachment cycling, question panes, reports, and gate cards. Clearing the
selection must clear and hide both metadata widgets.

### 3. Keep the countdown fresh without touching expensive UI

While the modal is mounted, maintain one modal-owned, approximately 30-second Textual
interval timer. Its callback must be a thin synchronous function that:

1. re-reads the current highlighted notification identity at callback time;
2. returns immediately unless that current row has `snooze_until`; and
3. rebuilds only the snooze-status `Text` and updates only its label.

The immediate `_display_file()` update remains the source of zero-latency selection
feedback; the timer exists only to keep time-derived text honest and to recover from a
wall-clock correction or suspend/resume. Do not call `_display_file()`,
`_rebuild_list()`, or any notification-store/provider method from the tick. Stop and
clear the timer from the modal's existing unmount cleanup so no callback survives the
screen. Starting one cheap timer for the open modal (even before a row is snoozed) is
preferable to mutation- specific timer wiring, because snoozing a row from inside the
already-open modal must begin updating correctly without another mount.

### 4. Cover semantics, lifecycle, and the actual visual failure mode

Add focused deterministic unit tests for the new builder/updater and timer behavior:

- future durations at the day/hour, hour/minute, minute, and under-one-minute tiers;
- exact expected local absolute labels for today, tomorrow, same-year, cross-year, and a
  DST-relevant aware timestamp;
- elapsed and malformed deadlines;
- hiding for non-snoozed and cleared selections;
- immediate selection changes between snoozed and ordinary rows;
- a timer tick re-resolves the current selection and updates only the status label;
- the tick does not call `_display_file()`, rebuild the option list, rerender
  attachments, or reset scrolling; and
- unmount stops the timer while preserving the existing pump-free-task cleanup.

Extend the ACE PNG snapshot suite with a deterministic Snoozed-tab case modeled on the
supplied screenshot: use a long bead/task title whose left-row suffix is clipped, select
it, pin both `now` and the wake deadline, and assert that the right-pane line visibly
contains `Snoozed`, the two-unit countdown, the absolute wake time, and timezone. Keep a
second ordinary selection in the same fixture or a structural assertion proving the
hidden widget leaves no blank metadata row. Accept the new golden intentionally, then
rerun the visual test with exact pixel comparison.

## Verification

Because this workspace may be old, run `just install` before repository checks.

1. Run the new and neighboring notification-modal unit tests, including sent-time,
   snooze mutation, selection/tab, and attachment cases.
2. Generate the intentional PNG golden with `--sase-update-visual-snapshots`, inspect
   the actual image for hierarchy, contrast, clipping, and unchanged footer/content
   layout, then rerun that visual test without the update flag for exact equality.
3. Run `just check` for the required whole-repo lint gates and diff-scoped tests.
4. Run `just test-visual` because the change intentionally modifies notification modal
   pixels; investigate any unrelated renderer drift instead of broadly accepting
   snapshots.

## Acceptance criteria

- The selected snoozed notification answers both “how much longer?” and “until when?”
  without requiring the user to inspect an attachment.
- The answer remains correct as time passes, across selection changes, and after a
  snooze is created in the already-open modal.
- Due and malformed deadlines degrade honestly and never crash the panel.
- Non-snoozed notifications retain the current one-line sent/filed metadata layout with
  no empty spacer.
- Countdown maintenance performs no I/O and no broad rerender, preserves navigation and
  scroll state, and is fully cleaned up when the modal closes.
- The focused unit suite, exact PNG snapshot, `just check`, and `just test-visual` all
  pass.
