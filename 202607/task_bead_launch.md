---
tier: tale
title: Add task-bead work launches and a detached submitter
goal:
  Task beads launch one deterministic, recoverable worker through sase bead work and an idempotent detached submitter.
bead: sase-bg.7
create_time: 2026-07-30 20:08:37
status: wip
---

- **PROMPT:** [202607/prompts/task_bead_launch.md](prompts/task_bead_launch.md)
- **PARENT:** [202607/task_beads.md](https://github.com/sase-org/sase--plans/blob/main/202607/task_beads.md)
- **BEAD:** [sase-bg.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-bg/sase-bg.7.md)

# Add task-bead work launches and a detached submitter

## Goal

Complete phase `sase-bg.7` from the task-beads epic: extend `sase bead work` so a standalone task bead can launch one
deterministic worker, with the launch state published in one commit before spawn, recoverable rollback on failure, exact
single-segment prompt composition, model routing, and an idempotent detached submission API for the subsequent
task-triage gate phase. Preserve all existing epic-plan and plan-file behavior.

## Current state and constraints

- The Python bead model already contains `IssueType.TASK`, `Status.READY`, task validation, size/model metadata, and
  `#bd/work_task` is available through `resolve_work_task_xprompt`.
- Existing epic work orchestration already supplies the reusable VCS context, deterministic-name cleanup, confirmation,
  launch adapter, bead-store checkpoint/push, and failed-launch recovery patterns.
- Task launches accept `ready`, `open`, or recoverable `in_progress` task beads; `claimed` and `closed` tasks are
  rejected. An `in_progress` task whose assigned agent is still live is an idempotent no-launch success rather than a
  destructive relaunch.
- `--dry-run` must render the complete prompt and cleanup preview without mutating bead state, agent state, or git. The
  task prompt must be one segment, contain no top-level `---`, and render in this order: project VCS reference with
  `#commit`, force-reuse `%id(!<bead-id>, bead=<bead-id>)`, `%m:<model>`, resolved work-task xprompt, and optional
  feedback as the final line.
- Model precedence is explicit bead model, then `<size>_phase_worker`, then the new implicit `task_worker` alias whose
  fallback is `@default`.
- The initial `status=in_progress` and `assignee=<bead-id>` transition must be one atomic bead mutation and one
  bead-store commit before agent spawn. Push uses the existing bead-work publication path. A zero-spawn failure restores
  the prior status and assignee in a committed recovery mutation; partial launch cleanup follows the existing agent
  rollback behavior.

## Implementation

1. Add task prompt/model helpers alongside the existing epic rendering code. Reuse the VCS segment-prefix validation,
   format explicit models through the existing alias-aware formatter, map task sizes to the existing size-specific
   phase-worker aliases, default missing size to `@task_worker`, and return the task worker’s single `SASE_BEAD_ID`
   launch environment with the internal deterministic-name bypass. Reject feedback that would introduce a top-level
   segment separator.

2. Add a task-specific launch orchestration path in the bead work modules. Resolve the work-task xprompt and required
   project VCS context; validate issue type/status; skip an already-running assigned task; render and preview
   deterministic-name cleanup; honor the existing `--yes` and `--yes-to-all` confirmation meanings; and keep dry-run
   read-only. For a real launch, complete stale-name cleanup before mutation, atomically update status and assignee,
   checkpoint and best-effort publish that one mutation, invoke the existing planned bead-work launcher with exactly one
   segment/environment, and restore the original fields through a committed/published recovery path if spawn fails.

3. Dispatch task IDs from `cli_work_entry.py` without changing epic or plan-file dispatch. Extend human and JSON
   success/error output with task-specific identifiers and launch state. Add internal optional launch-feedback plumbing
   to the work parser so the later gate can pass feedback through a detached command, while updating public work help
   and examples to describe epic plans and task beads accurately.

4. Add `src/sase/bead/task_launch.py` following the host-side epic submitter contract: canonical argv construction
   (`sase bead work <id> --yes-to-all`, plus optional feedback), gate-source-to-origin mapping, project/cwd resolution
   reuse, a global submission lock, active detached-task deduplication by task bead ID and `("task", "launch")` tags,
   and `submit_detached_task` metadata using label `Task launch · <id>`.

5. Register `task_worker` throughout the implicit model-alias policy and its compatibility exports/presentation:
   constant, `@default` fallback, description, completion/catalog behavior, and the commented builtin `model_aliases`
   example in `default_config.yml`. Update exhaustive alias expectations so configuration, resolution, completion,
   picker/model-panel, and doctor surfaces continue to agree.

6. Add focused tests modeled after the epic work suites:
   - exact prompt order, single-segment/no-fan-out shape, VCS `#commit`, feedback placement, explicit/size/default model
     precedence, and task launch environment;
   - ready/open launch, invalid type/status errors, live-assignee skip, stale relaunch cleanup, dry-run immutability,
     confirmation behavior, JSON output, one combined state mutation/checkpoint before launch, push ordering, and
     restoration after launch/checkpoint failures;
   - detached argv/origin/cwd behavior and active-task idempotency;
   - `task_worker` implicit alias resolution, descriptions, completions, and documented default configuration.

## Validation

1. Run the focused task work, detached submitter, rendering, parser/help, and model-alias test modules while iterating.
2. Run `just install` as required for this ephemeral workspace.
3. Run `just check` and fix every failure.
4. Reinspect `git diff --check`, the final diff, and `git status` to ensure only the planned source/tests/config changes
   are present.
5. Close only `sase-bg.7` with a note naming the successful focused tests and `just check`; do not close the parent epic
   or create beads.
