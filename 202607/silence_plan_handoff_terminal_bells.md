---
tier: tale
title: Silence plan handoff terminal bells
goal: 'Tale and epic review handoffs remain prominent and actionable across ACE and
  desktop surfaces without writing terminal BEL characters before their coder or epic
  follow-up starts, while genuinely audible notification classes retain their existing
  alerts.

  '
create_time: 2026-07-22 06:50:48
status: wip
---

- **PROMPT:** [202607/prompts/silence_plan_handoff_terminal_bells.md](prompts/silence_plan_handoff_terminal_bells.md)

# Plan: Silence plan handoff terminal bells

## Context and verified diagnosis

The reported `ho` run provides a complete timeline. Its non-silent, priority `PlanApproval` notification
(`7df5c43b-065a-4877-9cd8-4e6a7eb1ca17`) was created at 06:40:17, the gate response was written at 06:41:04, and
`ho--code` began at 06:41:36. No other notification arrived during that interval, and neither the planner nor coder
artifact tree contains a BEL byte. The sound was therefore temporally adjacent to the follow-up but was not emitted by
the coder startup itself.

The planner inherited `TMUX_PANE=%4`, the active tmux pane containing ACE. There are two independent paths that target
that pane for the same still-pending review:

- `handle_plan_approval()` creates the plan gate, sends a desktop notification, and directly calls `ring_tmux_bell()`.
  The resolved `tmux_ring_bell` helper writes a literal ASCII BEL to the target pane's tty.
- ACE later discovers the same new unread notification and treats every new active notification as bell-worthy. Its
  notification poll invokes the helper with a three-bell batch after updating the indicator and toast.

This is distinct from the provider's generic `Agent reply received` sound. The recently added pending-handoff guard is
working and should remain: it suppresses that generic completion sound when `.sase_plan_pending` or
`.sase_questions_pending` exists. The recent handled-plan fix also remains correct, but applies only when ACE discovers
that a plan notification has already been answered. This event was a genuinely pending plan review, so the two
intentional plan-ready bell paths remained active and could deliver four BEL characters for one logical event.

## Notification policy and scope

Make `PlanApproval` and `EpicApproval` terminal-silent on initial delivery. A plan review should continue to produce its
durable gate, priority inbox row, top-bar indicator, warning toast, and existing desktop notification; only the explicit
tmux/terminal BEL side effect changes. This avoids abusing `silent` or `muted`, either of which would also hide or
reroute the actionable review and affect Telegram or other consumers.

Preserve audible behavior for user questions, launch/custom/HITL gates, errors, visible agent completions, ordinary
notifications, and other currently audible classes. Preserve the explicit snooze-expiry reminder contract, including
when a user deliberately snoozed a plan review: this change targets unsolicited initial plan-ready bells, not reminders
the user scheduled. In a mixed polling batch, a plan review remains visually represented but an unrelated audible item
still causes exactly one batch bell.

This policy is presentation behavior. The existing notification action already identifies plan and epic reviews, so no
notification wire/schema, Rust-core API, stored-row migration, or linked `chezmoi` script change is needed.

## Producer-side alert cleanup

In the plan approval handler flow, retain gate creation, desktop notification delivery, approval polling, auto-approval
handling, and every follow-up result exactly as they are, but stop calling the tmux bell helper when a plan/epic gate
notification is created. Update the comment/import boundary so it no longer claims the plan producer emits both desktop
and tmux alerts. Do not remove or weaken the shared bell helper: the question flow and ACE still need it for audible
events.

Add focused coverage around a manually reviewable plan gate showing that the desktop notification is still sent while
the tmux bell is not invoked. Retain coverage for tale, epic, commit-only, feedback/rejection, late auto-approval, and
notification-free immediate auto-resolution so changing the side effect cannot alter gate semantics.

## ACE polling classification

Separate “new notification” from “terminal-audible notification” in ACE's polling decision. Continue using the full
post-reconciliation `new_notifications` collection for toast formatting, indicator/status projection, refresh
scheduling, and the boolean result that reports newly actionable work. Derive the bell decision from that collection
after excluding `PlanApproval` and `EpicApproval`; keep the existing already-handled-ID filtering ahead of this
decision. Keep snooze-expiry selection as its existing independent reminder path.

Keep the classification action-based rather than sender- or filename-based so tale and epic gates behave consistently
across local, Telegram, CLI, and future transport producers. Keep the bell subprocess off the Textual event-loop thread
for the audible cases that remain.

## Regression coverage and documentation

Update the focused notification-polling suite to prove:

- a new pending tale or epic review still increments the priority indicator, emits its warning toast, and returns a
  positive new-notification result, but does not ring;
- an already-handled review remains fully suppressed rather than being reintroduced by the new classification;
- a batch containing only plan/epic reviews remains terminal-silent, while a mixed batch containing a user question or
  another audible notification rings once and keeps every expected toast;
- questions and other audible notifications still execute the bell asynchronously, and snooze expiry retains its
  existing reminder bell;
- silent, muted, read, and previously seen notifications retain their current behavior.

Document the distinction between visual delivery and audible delivery in the ACE/notification documentation: plan and
epic reviews remain warning toasts and priority rows but do not ring the terminal on initial arrival; the other audible
classes and snooze reminders remain unchanged. Do not describe the top-bar glyph as an audible bell.

Run the targeted plan-approval response/auto-approval tests and notification polling/status-reconciliation tests while
iterating. Then run `just install` as required for an ephemeral workspace, followed by the repository-mandated
`just check`. Finally, re-run the focused polling and plan-gate tests after the full suite and audit the changed paths
for any remaining plan/epic call to `ring_tmux_bell`, while confirming the question path and final provider-completion
sound still have explicit regression coverage.

## Acceptance and risk controls

The change is accepted when a manually reviewed tale or epic produces no terminal BEL from either the gate producer or
ACE's first notification poll, yet remains immediately visible and actionable and launches the same approved follow-up.
A question or other audible notification arriving alone or in the same poll still rings according to the existing batch
rules, and a true terminal agent completion still plays its normal completion sound.

Avoid global bell disablement, changes to terminal/tmux configuration, notification-schema expansion, and `silent` or
`muted` flags. Those approaches would either leave the duplicate SASE source intact or suppress unrelated delivery
semantics. Keep the fix at both identified SASE call sites so removing only the producer bell or only the ACE bell
cannot leave a second BEL path behind.
