---
tier: tale
title: Make epic bead work a single pre-spawn commit
goal:
  Both epic launch entry paths preassign all rendered beads and publish one consolidated launch commit before any agent
  spawns, while preserving dry-run, retry, and failure recovery behavior.
bead: sase-aj.3
create_time: 2026-07-28 17:57:15
status: wip
---

- **PROMPT:** [202607/prompts/single_commit_epic_launch.md](prompts/single_commit_epic_launch.md)
- **PARENT:**
  [202607/beads_commit_consolidation.md](https://github.com/sase-org/sase--plans/blob/main/202607/beads_commit_consolidation.md)
- **BEAD:** [sase-aj.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-aj/sase-aj.3.md)

# Plan: Make epic bead work a single pre-spawn commit

## Goal

Complete phase bead `sase-aj.3` by making both forms of `sase bead work` preassign every rendered, non-closed phase bead
and the epic bead to their exact runner names before any agent is spawned. Persist readiness and all preclaims in one
beads commit, cross a synchronous visibility barrier before launch, make runner wait/promotion/release calls quiet
no-ops, and preserve the established dry-run, retry, zero-spawn rollback, and partial-spawn recovery boundaries.

The plan-file form must fold readiness and preclaims into the existing approved-graph checkpoint and push. The bead-ID
form must move its current post-launch commit and configurable push to a strict pre-launch commit and synchronous push.
No path should retain a second successful-launch beads commit.

## Context and constraints

- Rust core already provides `bead_preclaim_epic_work(beads_dir, epic_id, assignments, epic_agent_name, now)` from
  prerequisite `sase-aj.1`. It validates the entire batch, assigns all targets in one save, includes the epic when a
  land-agent name is supplied, reassigns already-in-progress phases on retry, excludes closed targets through the work
  plan, and returns `rollback_preclaims` containing each prior status and assignee.
- Prerequisite `sase-aj.2` made runner waiting claims, launch promotions, and claim releases no-op safely when the
  launched bead is already `in_progress` for the same agent. This phase must pass exactly the rendered agent names so
  those idempotence checks match.
- `epic_plan_launch_lock` remains the outer transaction lock. Short-lived bead-store mutation/commit locks stay nested
  beneath it.
- Detached workers must never spawn before the consolidated revision is visible. A failed visibility barrier preserves
  its locally committed launch state as a resumable point; a failed uncommitted zero-spawn attempt restores readiness
  and preclaims.
- A partial spawn terminates launched runners and preserves all premarked bead state. A zero-spawn launcher failure
  restores the core rollback payload and readiness, then records the existing single recovery commit.
- `--dry-run` and confirmation/forced-reuse failures must remain mutation-free.
- Only source templates under `src/sase/xprompts/skills/` may be edited for generated skills. Preview generation with
  `sase skill init --diff`; do not deploy from an uncommitted workspace.
- Do not close the parent epic or create any beads. Close only `sase-aj.3` after all validation passes.

## Implementation

### 1. Add the Python preclaim adapter and typed rollback boundary

- Extend `src/sase/core/bead_mutation_facade.py` with a guarded wrapper around `bead_preclaim_epic_work`, converting the
  returned issues through the existing wire helpers and exporting the wrapper.
- Add a `BeadProject` operation that accepts phase `(bead_id, agent_name)` assignments plus the land-agent name, records
  the mutation outcome, refreshes the compatibility database, and returns the prior-state rollback records in a
  well-defined Python representation.
- Keep mutation ownership in Rust; the Python layer only adapts wire values, tracks whether the project changed, and
  carries rollback metadata into launch cleanup.

### 2. Preclaim after identities are final and before the visibility barrier

- In `src/sase/bead/cli_work_handler.py`, derive the batch from `EpicWorkPlan.waves` and `EpicWorkPlan.land_agent_name`
  after confirmation and forced-reuse rewriting have completed.
- In the same launch transaction, mark the epic ready when needed and then batch-preclaim every rendered phase plus the
  epic. Closed phases are absent from the rendered plan and therefore untouched.
- Carry the preclaim rollback payload through every pre-spawn exception and launcher exception. Extend
  `rollback_work_launch` so zero-spawn recovery restores each target's prior status and assignee before its one recovery
  commit, while partial-spawn recovery only terminates runners and preserves preclaims.
- Ensure an already-ready retry still batch-reassigns all rendered non-closed phases and the epic to their current
  planned names; preserve previously closed phases exactly.

### 3. Consolidate commit and publication ordering for both entry paths

- For plan-file launches, keep `before_agent_launch` as the visibility barrier but make its existing
  `commit_epic_graph_checkpoint` capture graph creation, readiness, and the complete preclaim batch. Keep its strict
  synchronous publication before spawn and remove the later successful-launch commit.
- For bead-ID launches, replace the post-spawn `commit_successful_work_launch` behavior with a pre-spawn checkpoint:
  commit the readiness and batch preclaim once, synchronously publish it when a remote is configured, and only then call
  the launcher. Preserve explicit `--no-push` behavior only where launch visibility remains safe; reject an unsafe
  detached launch rather than spawning agents against stale remote state.
- Make commit/push failures report as pre-spawn visibility failures, not successful-launch failures. If no durable
  checkpoint was created, use zero-spawn rollback. If a local checkpoint exists but publication fails, preserve it for
  an explicit retry and expose the existing retry-requires-push signal.
- Retire obsolete post-launch helper usage and update commit messages/docstrings as needed so one semantic launch has
  one honest beads commit.

### 4. Update epic-launch status guidance

- Update `src/sase/xprompts/skills/sase_beads.md` to distinguish ad-hoc runner wait claims from `sase bead work`.
  Document that epic-launched phase and land beads are preassigned directly to `in_progress`, runner claim transitions
  then become no-ops, a runner dying while waiting no longer reopens the bead, and rerunning `sase bead work` is the
  recovery/reassignment path.
- Preview the generated result with `sase skill init --diff`; do not write to or deploy global generated skill files.

### 5. Add and revise regression coverage

- Add facade/project tests for the epic-inclusive preclaim call, exact phase/land assignments, returned rollback state,
  and reassignment of an already-in-progress phase.
- Update launch and lifecycle tests so the launcher observes every rendered bead already `in_progress` with the exact
  agent name. Assert the commit and synchronous push occur once and before the launcher for the bead-ID path.
- Update plan-file tests to assert the approved-graph checkpoint contains readiness and all preclaims, is published
  before spawn, and is the only successful-launch beads commit.
- Replace JIT-claim integration expectations with premarked-state expectations. Exercise waiting claim, promotion, and
  release against the premarked store and assert zero additional commits or pushes.
- Cover zero-spawn rollback to open/unassigned state, restoration of prior in-progress assignments on retry failure,
  partial-spawn state preservation, rerun reassignment, closed-phase exclusion, publication failure, and dry-run
  immutability.
- Keep existing ad-hoc bead runner claim tests unchanged to prove that non-epic launches still use the authoritative
  runner-side claim path.

## Validation

1. Run focused tests while iterating over the changed facade, project, launch, cleanup, plan-file publication, relaunch,
   dry-run, and runner-claim integration modules.
2. Run `sase skill init --diff` and confirm only the intended generated beads-skill wording would change.
3. Run `just install`, then the repository-mandated `just check`.
4. Review `git diff --check`, the final diff, and `git status --short` for accidental or unrelated changes.
5. Close only the assigned phase with:
   `sase bead close sase-aj.3 --note "<focused tests, skill preview, and just check verified>"`.
