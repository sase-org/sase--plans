---
tier: tale
title: Suppress bells for already-handled plan approvals
goal: 'ACE does not toast or ring the terminal bell for a plan-approval notification
  that is discovered to have already been answered, while pending plan reviews and
  all other actionable notifications retain their existing alerts.

  '
create_time: 2026-07-21 10:50:31
status: done
prompt: '[202607/prompts/suppress_handled_plan_bell.md](prompts/suppress_handled_plan_bell.md)'
---

# Plan: Suppress bells for already-handled plan approvals

## Context and diagnosis

ACE's notification poll currently classifies every newly unread notification as alert-worthy before applying
notification-driven status reconciliation. During that later reconciliation, a `PlanApproval` or `EpicApproval` whose
response has already been written by another path is dismissed and its agent receives the appropriate approved status.
The poll nevertheless retains the stale decision to toast and ring for that notification. For an approved tale, the
visible sequence is therefore `TALE APPROVED`, an unnecessary bell, and then `WORKING TALE` once the coder follow-up
artifacts become visible.

The runner's direct plan-ready alert and ACE's alert for a genuinely pending, new plan review are separate intentional
behaviors and should remain intact. The defect is specifically that an already-handled notification still alerts in the
same polling pass that recognizes and dismisses it.

## Implementation

Have notification status reconciliation report the notification IDs it auto-dismissed because an external or otherwise
pre-existing plan response was found. Keep its existing responsibilities for persistence, approved-status projection,
refresh requests, and handling legacy gates.

In the notification polling flow, reconcile status before finalizing the alert batch, then remove those newly
auto-dismissed IDs from the notifications used for toasts, bell selection, and the poll's "new actionable notification"
result. Preserve the existing indicator refresh and unread-ID bookkeeping so a dismissed plan notification is not
reintroduced on a subsequent poll. Snooze expiry reminders and unrelated notifications in the same batch must continue
to ring according to their current rules.

## Tests and validation

Extend the focused notification polling tests to cover a new plan approval that is reported as already handled during
status reconciliation: it should produce no toast, no bell, and no positive new-notification result. Add a mixed-batch
case so an already-handled plan does not suppress an unrelated actionable notification or its single batch bell. Retain
the existing assertion that a genuinely pending new plan approval still warns and rings.

Update the status-reconciliation tests to assert the returned handled-ID set for externally answered tale/plan, epic,
and commit responses, while normal pending plans and user questions report no auto-dismissed IDs. Run the targeted
polling and status-override test modules, then run `just install` followed by the repository-required `just check` to
validate formatting, linting, typing, and the full test suite.

## Risk controls

Keep the suppression decision tied to concrete handled IDs returned by the existing response-file reconciliation rather
than to plan-notification type or approved status alone. This avoids muting pending reviews, manual approval prompts,
questions, completion notices, error notices, or snooze reminders, and it makes mixed notification batches
deterministic.
