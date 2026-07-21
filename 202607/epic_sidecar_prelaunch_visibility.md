---
tier: tale
title: Make approved epic bead state visible before runner launch
goal: 'Approved epic plans publish a complete, claimable bead DAG before any phase
  or land runner starts, so fresh plans-sidecar workspaces cannot race missing child
  beads and failed launches remain safely resumable.

  '
create_time: 2026-07-21 16:47:55
status: done
---

- **PROMPT:** [202607/prompts/epic_sidecar_prelaunch_visibility.md](prompts/epic_sidecar_prelaunch_visibility.md)

# Plan: Make approved epic bead state visible before runner launch

## Context and diagnosis

Two recent observations need to be separated rather than treated as one plans-sidecar outage.

The `agents_h_parent_navigation_fix.md` planner completed normally. Its follow-up artifact recorded `plan_submitted_at`
and the durable plan path, the associated `PlanApproval` request remains available with no response, and the runner
process remains alive waiting for that terminal gate response. Ending the provider turn after `sase plan propose` is the
intentional handoff contract. The later `#coder` invocation is a separate manual run, not an automatic retry of a failed
sidecar operation. Preserve this behavior; do not add duplicate coder launches, implicitly approve the outstanding gate,
or reinterpret a submitted plan as a crashed agent.

The `sase-8j` epic failure is a real ordering defect. The first phase runner began at 16:28:52 and prepared a fresh
plans-sidecar clone that did not contain `sase-8j.1`. The approved-plan workflow only committed the accumulated epic,
four phase beads, dependencies, and ready-to-work mutation at 16:29:03, after the runners had been spawned. The phase
therefore failed its startup claim with `Issue not found: sase-8j.1`. A manual `sase bead work sase-8j` retry succeeded
because the delayed commit was visible by then.

The current transaction deliberately archives and links the plan first, creates the bead DAG in the working store, calls
the launcher, and only afterward commits and pushes bead state. Process creation is treated as launch success even
though each runner still has to prepare a new workspace and claim its bead asynchronously. That ordering is invalid for
sidecar-backed and other detached workspaces: a runner can only claim state that has reached the revision from which its
store is prepared.

## Establish a pre-launch publication boundary

Refactor the approved-plan creation path so the complete validated epic graph is checkpointed before `launch_work` is
allowed to spawn anything. The checkpoint must contain the parent epic, every phase in frontmatter order, all dependency
edges, the plan reference/linkage needed by phase prompts, and any launch-readiness state required by the worker claim
path. Keep plan archival and `bead_id` linking deterministic, but make the state transitions and commit messages
describe their true stage instead of relying on the post-launch `mark bead work launched` commit to sweep up graph
creation.

For a Git-backed store whose agents prepare independent workspaces, make successful publication—not merely a local Git
commit—the launch precondition. Use the existing store write lock, repository-health checks, and canonical sync helpers
so archive/link, graph publication, launch, and final state reconciliation remain serialized. The pre-launch push must
be blocking and must propagate failure to the caller; the current best-effort post-launch warning is not strong enough
for state that workers require at startup. Do not silently override `--no-push`: if the created graph cannot be made
visible to detached worker workspaces under that option, stop before spawning agents, preserve the locally materialized
state, and return a clear resumable outcome/command rather than launching a predictably broken clan.

Retain the normal post-launch commit for mutations that genuinely occur at or after launch, such as the ready marker and
runner-owned claims. Avoid publishing unrelated staged files, preserve the launch lock ordering, and keep in-tree/local
stores working without requiring a remote when their launch context shares the authoritative store.

## Preserve recoverability across the new boundary

Update the deterministic epic transaction's failure model to distinguish graph publication, zero-spawn launch failure,
partial spawn, and post-launch reconciliation:

- A validation, archive/link, graph-write, commit, or publication failure must start no runners and report which stage
  failed plus the existing safe resume command.
- If the graph was published but no runner spawned, perform the existing epic/phase removal and plan-content restoration
  as a compensating transaction, commit and publish that rollback when possible, and report any rollback failure without
  hiding the original error.
- Once any runner has spawned or claimed state, preserve the linked epic and its children for `sase bead work <epic>` to
  resume; do not delete claimable state out from under live or failed runners.
- A successful retry must remain idempotent: reuse the linked epic, skip closed phases, and never allocate duplicate
  phase beads or launch duplicate live owners.

Keep the public `BeadWorkError` spawn-boundary semantics accurate after the refactor. Console output, error
notifications, and launch timing should make `graph committed`, `graph published`, `agents spawned`, and
`post-launch state committed` distinguishable enough to diagnose a future failure from logs.

## Regression and acceptance coverage

Add focused unit tests around the creation callback and work handler to prove that graph persistence/publication happens
before the launcher is invoked and that launch is never reached when it fails. Update tests that currently codify a
single push only after launch; the asserted contract should instead require a blocking visibility barrier before launch
and only the necessary reconciliation afterward.

Add a Git-backed plans-sidecar integration regression that starts from an approved epic plan and observes the store from
the revision a fresh worker clone would receive. At the launch callback, assert that the parent epic, all generated
child IDs, dependencies, plan link, and required readiness metadata are already readable. Reproduce the original
missing-child shape by withholding publication and demonstrate that the guard stops launch rather than allowing an
asynchronous `Issue not found` failure.

Cover synchronous push failure, explicit `--no-push`, zero-spawn rollback after publication, partial-spawn preservation,
post-launch commit failure, and retry of an already-linked epic. Keep the existing tale-plan handoff tests passing and
add or sharpen a lifecycle assertion that a submitted question-continuation plan remains a pending approval rather than
being classified as a failed agent; this documents the first incident without changing its intended handoff behavior.

Run targeted bead creation/work/store tests first, then `just install` and the repository-required `just check`. Review
the final diff and emitted messages for accidental changes to unrelated bead stores, plan approval semantics, or agent
family presentation.

## Scope boundaries

- Do not modify or approve the outstanding `agents_h_parent_navigation_fix.md` gate, and do not interfere with its
  separately running coder.
- Do not change the remote-clone model for all linked repositories merely to mask an ordering error in epic creation.
- Do not weaken bead claims to create missing records or allow a runner to proceed without its authored phase.
- Do not add runtime-specific launch behavior; every supported agent runtime must observe the same published bead state.
- Do not edit SASE memory or generated instruction files.
