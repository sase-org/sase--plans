---
tier: tale
goal: "Retire the partially dismissed sase-5y.5 run by terminating its orphaned runner
  and reconciling only its workspace claim, pending question, artifacts, dismissal
  archive, notifications, and index projections, while preserving landed work and
  unrelated agent state.

  "
create_time: 2026-09-09 19:53:21
status: wip
---

# Plan: Repair the partially dismissed `sase-5y.5` agent

## Context

The run named `sase-5y.5` at artifact timestamp `20260714061831` is split between active
and dismissed state:

- PID `1910044` is still a live `run_agent_runner.py` process, the project still claims
  workspace 10 for that timestamp, and the run still owns a `pending_question.json`
  marker and unanswered question session.
- The run has no `done.json`, but its dismissal transaction already saved the root and
  workflow-child bundles, persisted dismissed identities, removed the run from the
  artifact index, and consequently hid it from the ACE Agents tab.
- The archived root says the commit finalizer failed, while the live runner is paused at
  the permission question. That disagreement, plus the surviving process and claim,
  shows that dismissal stopped partway through its intended side effects.
- Phase bead `sase-5y.5` and parent epic `sase-5y` are closed. The phase commit is
  already in the current upstream history, and the claimed workspace is clean, so there
  is no unfinished source diff to recover before retiring the run.

This is a targeted operational repair, not a broad artifact-index cleanup or a new
product feature. Current source already has the question-pause and user-kill cleanup
behavior needed to finish the lifecycle. If the repair reproduces a defect in current
code, stop and propose that prevention work separately instead of expanding this cleanup
in place.

## Desired final state

`sase-5y.5` has no live process, workspace claim, actionable question, active-artifact
row, or orphaned artifact directory. Its durable dismissed archive remains as one
coherent terminal family so failure history is not lost, and all archive/index
projections agree about the run. Beads, landed commits, agent-name history, workflow
logs, other workspaces, and unrelated agents remain unchanged.

## Repair sequence

1. **Revalidate identity and preserve a narrow rollback snapshot.** Immediately before
   mutation, resolve `sase-5y.5` again and require its metadata name, timestamp, PID
   command line, project claim, workspace number, artifact path, and question session to
   match the values above. Confirm the PID has not been recycled and that the phase
   commit remains upstream. Recheck the primary and any opened linked worktrees for
   uncommitted or unpushed work; if any exists, stop and preserve it rather than
   deleting state. Snapshot only this run's artifact metadata, dismissed
   bundles/identities, question files, and claim record so destructive steps can be
   rolled back without copying or rewriting unrelated global stores.

2. **Quiesce the live runner through the supported kill lifecycle.** Use the named-agent
   kill path for `sase-5y.5`, which records explicit user-kill intent, terminates the
   process group with the existing bounded escalation, releases the matching workspace
   claim, and dismisses the matching notification. Do not answer the stale
   protected-file question. Wait until the exact PID/process group is gone before
   touching artifacts. If lookup, identity, signalling, or permission checks fail, stop
   with the rollback snapshot intact; do not substitute an unscoped process kill or
   manual claim edit.

3. **Finish the interrupted dismissal transaction for this timestamp only.** Use the
   same cleanup ordering and Rust-backed/host-owned primitives as ACE: serialize or
   refresh the root and workflow-child dismissed bundles first, release any
   still-matching dead claim idempotently, reconcile the exact dismissed identities for
   `20260714061831`, delete artifact-index rows and aliases for the run, and only then
   remove its artifact directory. Remove the exact unanswered question session after its
   notification is no longer actionable. If termination was forced and the runner could
   not write normal terminal markers, represent the explicit killed outcome and stop
   time in the archive rather than fabricating a successful answer or completion.

4. **Preserve history and constrain collateral effects.** Keep the normalized dismissed
   archive, workflow log, closed bead records, landed commit attribution, and permanent
   agent-name reservation. Do not revive or reuse the name, reopen/close beads,
   regenerate protected instruction files, delete workspace 10, or run whole-host
   `sase agent index gc`; those actions are not required to retire this run and could
   affect unrelated agents. Remove the temporary rollback snapshot only after the
   retained archive and every targeted postcondition validate.

## Validation and revalidation

Validate after each mutating stage, then repeat the complete read-only audit after the
repair has settled:

- `sase agent list -j` and `sase agent list -a -j` contain no live
  `sase-5y.5`/`QUESTION`/`ANSWERED` row, and the old PID and process group do not exist.
- The project has no claim for timestamp `20260714061831` or the old workflow; workspace
  10 is unclaimed and reusable without deleting its checkout.
- The run artifact directory and exact question-session directory are absent, and no
  pending notification can still answer that question.
- The dismissed archive loads one coherent terminal `sase-5y.5` family, its bundle
  summaries and dismissed identities agree, and `sase agent archive verify` passes
  before and after cleanup.
- The artifact index has no active row or alias for the removed artifact path, while its
  dismissed projection matches the retained archive. Record the existing global
  `sase agent index verify` drift before repair and ensure the targeted suffix is clean
  and the global counts do not worsen; do not use the unrelated pre-existing
  missing/stale rows as permission for broad GC.
- `sase-5y.5` and `sase-5y` remain closed, the primary and linked repositories remain
  clean at their prior revisions, and no other process, claim, artifact, dismissed
  identity, notification, or index row changed.

Because the repair should change only SASE host state, it requires no repository source
edits or repository-wide test run. If implementation discovers that a source change is
necessary, stop before editing and create a separate validated plan with focused
regression tests and the repository's required `just check` validation.
