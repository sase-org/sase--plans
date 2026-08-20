---
tier: tale
title: Make research publication and family handoffs collision-safe
goal:
  Parallel research publication and in-process family successors preserve every output
  and advance deterministic failures to a terminal state.
size: medium
proposed_by: bbugyi200.athena.sase-rm.4
bead: sase-rm.4
create_time: 2026-08-20 16:14:07
status: wip
---

- **PROMPT:**
  [prompts/202608/successor_publication.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/successor_publication.md)
- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/sase-rm.4.md)

# Plan: Make research publication and family handoffs collision-safe

## Outcome

Complete phase bead `sase-rm.4` and leave close-ready evidence for its five assigned
tasks without closing those task beads or the parent epic. Parallel research dispatches
must receive distinct deterministic report targets, deterministic agent-publication
format failures must advance to a recorded terminal state, every in-process successor
must reserve a unique artifact timestamp, plan-feedback replans must use the shared
successor engine, and the default pipe family/workspace path must remain stable under
repeated contention.

The work spans the primary `sase` repository and its linked `sase-research-artifacts`
plugin. Open the linked repository with `/sase_repo` before reading or writing it and
follow each repository's `AGENTS.md`.

## Current behavior and constraints

- `sase-research-artifacts` currently expands both independent researcher segments to
  the same unparameterized `#research` prompt. Each agent chooses its own filename in
  the shared month directory, so identical parallel prompts can choose the same path.
- `create_followup_artifacts()` calls `create_artifacts_directory()` without a reserved
  timestamp. Two in-process successors created in one wall-clock second therefore share
  one directory and the later `agent_meta.json` replaces the earlier one.
- Detached launches already reserve globally unique `YYmmdd_HHMMSS` timestamps through
  `reserve_launch_timestamp_batch()`; the in-process path should use that same boundary.
- The plan-feedback branch in `run_agent_exec_plan.py` still hand-rolls promotion,
  suffix allocation, artifact creation, prompt state, and prompt-artifact storage.
  `continue_as_successor()` already owns that sequence for plan acceptance, questions,
  and pipe handoffs.
- Publication requests are attempted by the active drain. Repeated
  `AgentsSyncFormatError` failures only retire for the exact “no publishable runs” text;
  stable relationship-validation failures instead cycle through quarantine forever.
- The exact `sase-r2` pipe node currently passes alone and is not in
  `tests/reproducible_flake_baseline.txt`. Do not add or retire a baseline entry without
  post-change evidence satisfying that file's contract.
- Preserve the current named and unnamed feedback suffixes, promotion and metadata
  ordering, relationship fields, full-feedback artifact label/content, pipe workspace
  inheritance, and user-facing project display names.
- Do not edit memory files, create task beads, close assigned task beads, or close the
  parent epic. Record any distinct discovery as a `PROPOSED FOLLOW-UP:` note on
  `sase-rm.4`.

## Implementation

1. In `sase-research-artifacts`, extend `#research` with an optional explicit report
   target and make both `#research_swarm` researcher segments pass targets derived from
   their qualified keyed clan/member identities. Render the exact month-relative path in
   the child prompt and require creation without overwrite; a pre-existing target must
   produce a visible collision failure rather than silently replacing content. Keep
   standalone `#research` compatible. Add plugin tests that render/dispatch two
   identical swarms under a frozen launch clock and prove all researcher targets are
   deterministic and distinct across members and dispatches while the dependency graph
   remains unchanged.

2. In `sase`, route in-process follow-up artifact creation through
   `reserve_launch_timestamp_batch(1)` and pass the reserved timestamp to
   `create_artifacts_directory()`. Add a frozen-clock regression using genuine
   `create_followup_artifacts()` calls: create two successors back-to-back, assert their
   directories differ, and verify both `agent_meta.json` files retain the correct name,
   suffix, relationships, and shared workspace metadata.

3. Migrate the feedback branch in `run_agent_exec_plan.py` to `continue_as_successor()`
   with the `--plan-@` suffix template, current reservation set, exact unnamed fallback
   token, feedback relationship block, and `Full feedback prompt` artifact label. If
   needed, add a narrow pre-create callback to the successor engine so
   `record_workflow_metadata()` can still record the resolved follow-up name before
   artifact creation. Assemble the accumulated feedback prompt before the engine call,
   leave the source `feedback_submitted_at` write in place, and set
   `question_base_prompt` afterward. Extend direct successor and plan-feedback tests to
   prove callback/metadata order, named and unnamed suffixes, repeated rounds,
   relationship propagation, promotion order, and full prompt persistence.

4. Treat repeated `AgentsSyncFormatError` preparation failures as deterministic terminal
   candidates, not just the one exact message already recognized. Preserve the existing
   first-failure retry guard: only the same error seen again may retire a request. Add a
   transaction/outbox lifecycle test using the research hood's “invalid hood
   relationships: duplicate or inconsistent container global name” failure. Prove an
   aged `attempts=0` request is attempted, advances to `attempts=1`, retires on the
   repeated identical failure with `terminal_reason`, disappears from the active queue,
   and is rendered by doctor with the `--drop-retired` remediation rather than an
   inapplicable retry command.

5. Strengthen the pipe regressions around the shared successor mechanism. Exercise
   repeated real family transitions with the same frozen wall-clock base, assert unique
   successor artifact directories and intact metadata, and repeatedly run
   `tests/fakey/test_pipe_e2e.py::test_default_pipe_creates_family_member_with_fork_and_shared_workspace`.
   Confirm the family member keeps its fork prompt and the parent's workspace directory
   and number. Do not mask a real collision with sleeps, wider timeouts, or a flake
   baseline entry.

## Verification and handoff

1. Run `just install` before checks in each repository whose environment needs refresh.
   While iterating, run the focused plugin xprompt tests and the focused primary tests
   for artifact helpers, successor/plan feedback, publication transaction/outbox/doctor,
   and pipe E2E behavior. Repeat the exact `sase-r2` node enough times to exercise the
   contention-sensitive path.
2. Run `just check` in `sase-research-artifacts` and then `just check` in `sase`. If a
   primary `just check` escalates or reports unusual selection, use `/sase_monitor` for
   `just check-full` with the required `TESTING`/`TESTED` statuses and a useful next
   prompt.
3. Re-read all five assigned task beads and append one evidence-rich close-ready block
   per task to `sase-rm.4`, naming the cause, changed files, and exact verification.
   Leave the task beads for the epic land agent.
4. Run `sase bead epic-symbols sase-rm.4`. Resolve every remaining phase symbol or
   re-key its Justfile entry to `sase-rm` or a still-open later phase before closing.
5. Close only `sase-rm.4` with `sase bead close sase-rm.4 --note "<what was verified>"`.
   Do not close `sase-rm`, any ancestor plan bead, or any assigned task bead.
