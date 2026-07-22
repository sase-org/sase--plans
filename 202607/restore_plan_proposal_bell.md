---
tier: tale
title: Restore the tale/epic plan-proposal terminal bell
goal: 'A newly proposed tale or epic review again rings the terminal bell once when
  it arrives in ACE, so the user is audibly alerted that a plan is waiting, while
  the terminal stays silent after the plan is approved and while an agent hands off
  into its coder/epic follow-up.

  '
create_time: 2026-07-22 09:18:34
status: done
---

- **PROMPT:** [202607/prompts/restore_plan_proposal_bell.md](prompts/restore_plan_proposal_bell.md)

# Plan: Restore the tale/epic plan-proposal terminal bell

## Product context

When an agent finishes a plan and proposes it for review, the user (sitting in `sase ace`) expects a terminal bell so
they know a tale or epic is waiting for their approval. That proposal bell was recently lost: a plan can now arrive
completely silently. The user must instead notice the visual toast / indicator on their own, which defeats the purpose
of an audible "come review this" alert. This plan restores the single proposal bell without bringing back any of the
_after-approval_ bells that earlier fixes correctly removed.

## Verified diagnosis

Three fixes landed in sequence, each targeting a real "bell after I approved the plan" report. The first two were
correct and must stay; the third over-corrected and caused the regression.

1. **`suppress_handled_plan_bell`** — ACE's notification poll classified every newly unread notification as bell-worthy
   _before_ status reconciliation ran. When a `PlanApproval` / `EpicApproval` whose response had already been written
   was discovered and auto-dismissed in that same poll, the poll still rang. The visible sequence was `TALE APPROVED` →
   stray bell → `WORKING TALE`. The fix has reconciliation report the auto-dismissed notification IDs and removes them
   from the toast/bell batch. This guard is still in place and is exactly what keeps an already-answered plan silent
   (regression test `test_already_handled_plan_approval_does_not_alert`, which drives the
   `_auto_dismissed_notification_ids` path). **Keep.**

2. **`suppress_intermediate_handoff_bell`** — the bell that remained at the `WORKING TALE` transition came from a
   _different subsystem_: `run_bam_command("Agent reply received")` in successful LLM postprocessing. When
   `sase plan propose` SIGTERMs the planner, the provider's completion sound emitted three BEL bytes that were replayed
   into the terminal exactly as the approved tale started (between the planner's `Received SIGTERM` line and the coder
   prompt). The fix suppresses that completion sound whenever a durable `.sase_plan_pending` / `.sase_questions_pending`
   handoff marker exists (`has_pending_handoff` guard in `src/sase/llm_provider/postprocessing.py`). Its own plan
   explicitly said to **keep** ACE's actionable-notification bell and the plan-ready alert. This guard is still in
   place. **Keep.** This is the fix that genuinely eliminated the after-approval terminal sound.

3. **`silence_plan_handoff_terminal_bells`** — after (1) and (2) had already removed the genuine after-approval sounds,
   the user reported hearing a terminal bell again "right before the coder started up." The plan's own reconstructed
   timeline shows the `PlanApproval` notification was created at 06:40:17, the approval response was written at
   06:41:04, and the coder began at 06:41:36; it also confirms no other notification arrived in that window and that no
   BEL byte exists in either agent's artifact tree (consistent with fix (2) working). The only remaining bell sources it
   could find — the producer's `ring_tmux_bell()` in the plan-approval handler and ACE's poll bell — **both fire at
   proposal time (≈06:40:17), not at 06:41:36.** The plan never reconciled that ~79-second gap; it silenced both
   proposal paths anyway. The net effect: the legitimate _proposal_ bell was removed to "fix" a sound that had already
   been eliminated the day before (and whose reported timing points at the proposal bell itself, not a distinct
   after-approval event).

**Answering the user's "verify this was the case":** The genuine "bell after approval" _did_ exist earlier and was real
— it was ACE ringing for already-handled plans (fixed by (1)) and the `run_bam` completion sound replayed during the
handoff (fixed by (2)). By the time fix (3) shipped, the after-approval terminal sound was already gone. Fix (3)'s
remaining evidence points to the proposal-time bell (06:40:17), which is the bell the user _wants_, mis-attributed as
after-approval. Fix (3) therefore traded away a correctly-functioning feature for no real benefit.

### Where the regression lives in the code today

`src/sase/ace/tui/actions/agents/_notification_polling.py` currently defines
`_TERMINAL_SILENT_ACTIONS = frozenset({"PlanApproval", "EpicApproval"})` and derives the bell decision from
`new_notifications` after excluding those actions, so a genuine new plan review never rings. Fix (3) also removed the
producer-side `ring_tmux_bell()` call from `handle_plan_approval` in `src/sase/llm_provider/_plan_utils.py`, so no bell
fires from that side either. Between the two, a proposed plan is fully silent.

## Fix

Restore the proposal bell at ACE's poll — the aggregation surface the user watches, which sees plan reviews from every
agent and machine — and rely on the still-intact guards (1) and (2) to keep everything after approval silent.

- In `src/sase/ace/tui/actions/agents/_notification_polling.py`, remove the `_TERMINAL_SILENT_ACTIONS` special-casing so
  the bell decision once again rings for any genuinely new unread notification (i.e. `should_ring_bell` is true when
  there are new notifications or expired snoozes). Update the adjacent comment so it no longer claims plan reviews are
  terminal-silent; it should describe that muted arrivals do not ring, already-handled plans have already been filtered
  out of `new_notifications` by reconciliation, and snooze expirations ring independently. Because the auto- dismiss
  filtering from fix (1) runs before this decision, an already-answered plan still contributes nothing to the bell, and
  the `run_bam` handoff guard from fix (2) still keeps the coder handoff silent. The result is one bell when a plan is
  genuinely proposed, and none after approval.

- **Scope guard — do not restore the producer bell.** Leave `handle_plan_approval` in
  `src/sase/llm_provider/_plan_utils.py` as-is (no `ring_tmux_bell()`), so the plan proposal rings from exactly one path
  (ACE) rather than the pre-fix two-path behavior that could deliver several BEL characters for one event. This also
  means the producer-side test `test_manual_plan_gate_sends_desktop_notification_without_terminal_bell` in
  `tests/test_plan_approval_responses.py` remains valid and untouched.

- **Do not touch the question bell path.** `run_agent_helpers_questions.py` still calls `ring_tmux_bell()` for user
  questions and `UserQuestion` is not silenced in the poll; questions are out of scope for this change.

The change is presentation-only ACE behavior driven by the existing notification `action` field, so no notification
wire/schema, Rust-core API, stored-row migration, or linked `chezmoi` change is needed.

## Tests

Update the focused notification-polling suite in `tests/test_notification_toast_polling.py` to encode the restored
policy:

- Flip `test_single_new_plan_review_warns_without_terminal_bell` (parametrized over `PlanApproval` / `EpicApproval`) so
  a genuinely new plan review now rings once (`_bell_rung == 1`) in addition to keeping its warning toast and
  priority-indicator assertions. Rename it to reflect that a new plan review both warns and rings.
- Flip `test_plan_and_epic_only_batch_is_terminal_silent` so a batch of only plan/epic reviews rings exactly once for
  the batch (`_bell_rung == 1`) while keeping its toast and indicator assertions. Rename it away from "terminal_silent."
- Keep `test_already_handled_plan_approval_does_not_alert` and
  `test_already_handled_plan_does_not_suppress_unrelated_alert` green unchanged — these are the after-approval
  regression guards proving an already-answered plan stays silent (via the auto-dismiss filter) while an unrelated
  actionable notification in the same batch still rings once.
- Keep the mixed-batch, single-regular-notification, muted/read, silent, and all snooze-expiry tests green (including
  `test_expired_plan_snooze_keeps_explicit_reminder_bell`).

Leave `tests/test_plan_approval_responses.py` and `tests/test_llm_provider_postprocessing.py` unchanged; the producer
and postprocessing behaviors are intentionally not modified.

## Documentation

Revert the plan-specific "terminal-silent" claims introduced by fix (3):

- `docs/ace.md`: replace the paragraph stating that initial tale and epic reviews are delivered without ringing the
  terminal with wording that a genuinely new tale/epic review rings once on arrival like other actionable notifications,
  while already-answered plans and the post-approval coder handoff stay silent.
- `docs/notifications.md`: update the "Visual and Audible Delivery" section so it no longer says `PlanApproval` /
  `EpicApproval` initial delivery is terminal-silent; state that a new plan review rings on arrival and that the silence
  applies to already-handled reviews and the intermediate handoff. Preserve the accurate terminology cleanups from fix
  (3) (e.g. "top-bar indicator," "arrival bell") where they are not tied to the plan-silence claim.

## Validation

- Run the focused modules while iterating: the notification-polling suite (`tests/test_notification_toast_polling.py`),
  the status-reconciliation tests, and the plan-approval response tests to confirm the producer side is unaffected.
- Because this is an ephemeral workspace, run `just install` first, then the repository-mandated `just check` (ruff +
  mypy + full pytest, including visual snapshots) and ensure it passes.
- Final audit: grep the changed paths to confirm no plan/epic action is still excluded from the ACE bell, that the
  producer `ring_tmux_bell()` remains absent from `handle_plan_approval`, and that the `has_pending_handoff` guard in
  `postprocessing.py` and the auto-dismiss filter in the poll are both still present.

## Acceptance and risk controls

Accepted when: a genuinely proposed tale or epic produces exactly one terminal bell on arrival in ACE and remains
visible/actionable; an already-answered plan discovered by the poll produces no bell; and the approved-plan → coder/epic
handoff produces no terminal sound (guard (2) intact). A user question or other audible notification arriving alone or
in the same batch still rings once, and snooze-expiry reminders (including for a snoozed plan) still ring.

Rejected alternatives:

- **Also restoring the producer `ring_tmux_bell()` in `handle_plan_approval`.** Rejected to keep a single proposal-bell
  path; restoring both paths recreates the multi-BEL-per-event behavior the previous fix was partly reacting to, and ACE
  already covers plans from every agent and machine. Known trade-off: if the user proposes a plan with no `sase ace`
  instance running, there is no proposal bell; this matches the ACE-centric workflow and can be revisited separately if
  it proves limiting.
- **Muting or marking plan notifications `silent`.** Rejected because those flags also hide or reroute the actionable
  review (indicator, modal, Telegram); the fix must stay a pure audible-delivery decision keyed on the notification
  `action`.
