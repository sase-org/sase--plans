---
tier: epic
status: done
title: Finish the status-strip integration after planner restoration drift
goal:
  Make gate shells the sole plan/question status publisher while preserving concrete
  post-gate handoff labels on the current integrated tree.
parent_bead: sase-ud.13.1.3.1
phases:
  - id: status-reconcile
    title: Reconcile the restored planner and timestamp status machinery
    depends_on: []
    description:
      "status-reconcile: remove the reintroduced synthetic planner and
      timestamp-reconstruction paths, preserve concrete handoff labels, and realign
      tests to the gate-shell contract."
    size: medium
proposed_by: bbugyi200.athena.sase-ud.13.1.3.1.land
bead_id: sase-ud.13.1.3.1.5
create_time: 2026-09-09 19:50:23
---

- **PROMPT:**
  [prompts/202608/finish_status_strip.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/finish_status_strip.md)
- **PARENT:** [202608/status_strip.md](status_strip.md)
- **BEAD:**
  [sase-ud.13.1.3.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.13.1.3.1.5.md)

# Plan

## Context

Epic `sase-ud.13.1.3.1` proved the gate-shell projection contract and removed the
notification-driven writers, but it is not complete on the current tree.

- Phase `sase-ud.13.1.3.1.3` removed synthetic planner rows in commit `b69b07bc9`.
  Concurrent Master Gate work then restored them in `4d3156363`, changed the tests to
  require them in `9c3764539`, and extended their status policy in `69527b84a`.
- Phase `sase-ud.13.1.3.1.4` recorded that timestamp reconstruction helpers were
  deleted, but its landing commit `8efce6de9` did not contain those deletions. The
  current `apply_status_overrides` still performs `DONE` to plan/question/feedback
  reconstruction, and `WORKING_PLAN_STATUS_TO_APPROVED` is still present without a
  consumer.
- The nine gate-shell family contract tests pass. The current tree still has the
  expected phase-2 result: `_notification_status_overrides.py`,
  `_agent_pre_question_status`, and the `_agent_status_overrides.py` facade are absent;
  legacy notification lifecycle reconciliation remains isolated in
  `_notification_plan_reconciliation.py`.

The Master Gate restoration tests exercise pre-gate/root-only fixtures. They must be
realigned to the modern contract rather than keeping a second publisher alive: a real
gate member owns pending and settled plan/question status, the family container mirrors
that gate, and the planner member remains `DONE`. Historical families without a gate may
degrade to `DONE`, as the approved parent plan explicitly allows.

## Implementation

1. Remove the restored synthetic planner machinery from the current integrated source:
   `ensure_synthetic_planner_children`, `sync_planner_child_from_parent`,
   `planner_child_status`, `answered_asker_freeze_time`, their private approval helpers
   and follow-up-only helpers, the `is_synthetic_planner` state field, every guard whose
   only purpose is excluding synthetic rows, and the facade exports/imports that keep
   those symbols public. Preserve the metadata-copy helpers used by real family members.

2. Finish the timestamp-reconstruction deletion that phase 4 reported but did not land.
   Remove the `DONE` to `PLAN`/`TALE`/`EPIC`, `QUESTION`, and `FEEDBACK` passes from
   `apply_status_overrides` and delete their now-unreachable helpers, including
   `has_unreviewed_submitted_plan`, `is_awaiting_plan_review`,
   `has_unanswered_completed_question`, `has_inherited_family_question`,
   `superseded_by_feedback_round`, `_is_planner_family_row`,
   `feedback_child_progressed_past_review`, `pending_plan_status_for_agent`, and
   `latest_non_workflow_child_launch_by_parent`. Delete the unreferenced
   `WORKING_PLAN_STATUS_TO_APPROVED` constant and rewrite the override-pass docstring to
   describe only the surviving family mirroring, metadata propagation, and concrete
   handoff labeling.

3. Keep and protect the concrete post-gate behavior that remains reachable:
   `active_approved_plan_handoff_status`, `approved_followup_planner_status`,
   `is_completed_plan_handoff_child`/`done_handoff_status`,
   `is_completed_epic_followup_child`, `is_answered_continuation_asker`, and
   `is_answered_root_asker_step`. These label real coder, epic, planner-continuation,
   and answered-handoff rows after a gate settles; they are not substitutes for the gate
   status itself.

4. Realign tests with the integrated gate-shell contract. Restore the no-synthetic-row
   expectations in the retry-family, PID-dedup, and host-owned epic-metadata tests that
   `9c3764539` changed. Delete tests whose sole subject is retired timestamp
   reconstruction or synthetic materialization. Rewrite any surviving projection test to
   use real `FamilyShellWire`/`FamilyShellGateWire` metadata, following
   `tests/test_agent_loader_status_override_gate_shell_family.py`, and keep its pending,
   settled, running-coder, completed-coder, and mirrored-gate-pair assertions intact.

5. Inspect every touched status-override test rather than weakening assertions. Confirm
   modern pending plan/question families get their status only from a gate row, legacy
   root-only artifacts no longer synthesize a planner, and concrete post-approval rows
   still render `WORKING PLAN`/`WORKING TALE`, `PLAN DONE`/`TALE DONE`, `EPIC CREATED`,
   and `ANSWERED` where their durable inputs warrant those labels.

## Verification

- Rebuild the local Rust binding first if its scan wire reports schema 6 while Python
  expects schema 7; do not treat that already-tracked parent-epic drift as a status
  regression.
- Run the gate-shell contract module plus the retry-family, PID-dedup, epic-created, and
  affected `test_agent_loader_status_override_*` / `tests/ace/tui/models` suites.
- Run `just check` as the required repository gate.
- Because this changes which family rows render, run `just check-full` and
  `just test-visual` through the SASE monitor workflow, inspect visual diffs before
  accepting any intentional golden update, and leave unrelated fail-then-pass nodes for
  the land agent's collected follow-up triage.

This plan intentionally excludes bead closure, epic-symbol retirement, the final
Symvision confirmation, unrelated proposed-follow-up triage, and marking the parent plan
file done; the land agent performs those after this child work lands.
