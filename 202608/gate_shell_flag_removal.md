---
tier: tale
title: Remove the gate-shell handoff flag and blocking fallback
goal:
  Plan and question handoffs use gate shells unconditionally, with the obsolete blocking
  implementation and flag surface removed.
size: medium
proposed_by: bbugyi200.athena.sase-ud.13.1.2
bead: sase-ud.13.1.2
---

- **PARENT:** [202608/gate_shell_status_collapse.md](gate_shell_status_collapse.md)
- **BEAD:**
  [sase-ud.13.1.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.13.1.2.md)

# Remove the gate-shell handoff flag and blocking fallback

## Goal

Make the already-landed plan and question gate-shell paths unconditional, remove the
legacy blocking approval/question flows and their now-dead API surface, update the
generated-skill source prose and feature-flag schema, and preserve the auto-resolved
in-process continuation paths.

The implementation is limited to phase bead `sase-ud.13.1.2`. The launch prompt says to
close only that bead, so this work will not close the flag bead `sase-uo` even though
the older parent design expected flag closure in the same phase.

## Implementation

1. Make both marker handlers use gate shells unconditionally.
   - In `run_agent_exec_plan.py`, remove the flag import/check and the blocking
     `handle_plan_approval` fallback, call the existing gate-shell helper directly, and
     retain `_continue_after_plan_result` for synchronously auto-resolved gates.
   - In `run_agent_exec_questions.py`, remove the flag import/check and the entire
     blocking question path, make the gate-shell helper the direct marker handler, then
     delete only imports/private helpers whose remaining call count is zero.

2. Remove the blocking-only production surface by following the live call graph.
   - Delete `sase/gate_shell/flag.py`, the public `handle_plan_approval` wait loop and
     its private helpers that lose their last caller, and
     `plan_gate.create_plan_approval_gate` plus its export while preserving response
     projection and auto-handled helpers used by `plan_shell`.
   - Delete `run_agent_helpers_questions.py`, its compatibility-facade re-export and
     patch synchronization, and `user_question_actions.create_user_question_gate` plus
     its export while preserving `user_question_gate_spec` for `question_shell`.
   - Keep read-side support for historical `pending_question.json` artifacts, and use
     Symvision plus repository search to identify any additional helpers that became
     unreachable instead of deleting similarly named live APIs.

3. Retarget and prune tests around the surviving gate-shell seams.
   - Remove disabled-branch cases and enabled-flag monkeypatches from the dedicated
     plan/question gate-shell tests.
   - Extend the shared plan marker patch harness with a non-handoff
     `create_plan_gate_shell` result and replace per-test patches of the removed
     `handle_plan_approval` seam with `plan_shell.plan_result_from_gate_creation`, so
     downstream continuation, archive, metadata, effort, prompt, and epic behavior
     remains covered.
   - Delete tests whose sole subject is a removed blocking helper or neutral gate
     creator; update mixed test modules and mutation-audit expectations so all surviving
     APIs retain their behavioral coverage.

4. Remove the flag from configuration and describe the one remaining behavior.
   - Remove the enum member and registry definition, delete the accessor module, and
     regenerate `sase.schema.json` with `tools/sync_feature_flags_schema --write`.
   - Minimally update the `sase_plan.md` and `sase_questions.md` source templates to
     describe unconditional gate-shell handoff; preview generated output read-only and
     do not deploy generated skills from the dirty phase tree.

## Verification and completion

1. Run focused tests for plan/question marker handling, plan-result continuation,
   question/plan gate APIs, and feature-flag schema integrity while iterating.
2. Run `just install` if the workspace environment is stale, then run Symvision and
   `just check`; resolve every caused failure without whitelisting dead symbols.
3. Run `just check-full` through `/sase_monitor`, as required by the parent design for
   this phase, and address any failures attributable to the change.
4. Re-run `sase bead epic-symbols sase-ud.13.1.2`; resolve every entry or re-key it to a
   still-open later bead before closure.
5. Close only `sase-ud.13.1.2` with a note naming the focused, scoped, and full-suite
   verification performed. Do not close `sase-uo`, `sase-ud.13.1`, `sase-ud.13`, or any
   ancestor.
