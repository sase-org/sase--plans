---
tier: tale
title: Make final declaration use unavoidable and auto-close assigned beads
goal:
  Every normal SASE turn declares finalizer intent, and every successfully landed
  assigned-bead commit closes that bead unless explicitly opted out or rejected by
  lifecycle validation.
size: medium
proposed_by: bbugyi200.athena.0bu
---

# Make final declaration use unavoidable and auto-close assigned beads

## Context and root cause

The completed `sase-s9.3` and `sase-s9.4` transcripts show that both phase workers ended
their normal model turns while long verification commands were still running and without
invoking `/sase_final`. The one-shot declaration recovery correctly constrained each
follow-up to `/sase_final`, accepted a commit decision, and dispatched
`sase stitch create`. Both commit artifacts retained the correct `SASE_BEAD_ID`, but the
stitch output explicitly left the bead `in_progress` because the assigned object was a
`phase` bead. The beads were closed manually later.

Two independent gaps therefore need repair:

1. The always-loaded final-declaration instruction says to use `/sase_final` before a
   final response, but it does not explicitly cover responses that promise to wait,
   report incomplete work, or otherwise end a normal provider turn without looking like
   a conventional final answer. The generated `/sase_final` skill description is also
   conditional enough to reduce its trigger salience.
2. `sase.workflows.commit.bead_hooks` intentionally limits post-land auto-close to
   `task` beads. That restriction contradicts the `-B|--do-not-close-bead` contract for
   assigned phase and plan beads, even though the existing close command already
   provides the necessary lifecycle guards.

## Implementation

1. Strengthen the canonical terminal-action contract in the current SASE short-memory
   note and the default `memory-sase` template. State that every normal response which
   ends a provider turn must invoke `/sase_final` as its last action, including an
   incomplete-status or "I will wait" response; only a successfully executed
   plan/monitor/pipe/questions handoff is exempt. Make clear that intending to resume in
   a later turn is not an exemption. Update the canonical `/sase_final` skill source so
   its description and opening rule use the same mandatory language.

2. Run `sase memory init` after the canonical memory edit, as required, so `AGENTS.md`,
   all provider instruction copies, and the memory README are regenerated together.
   Preview the generated skill deployment with `sase skill init --diff`; do not deploy a
   dirty, unlanded skill source globally.

3. Generalize commit-time bead auto-close from task-only to any assigned `in_progress`
   bead after a successful `create_commit` or `create_pull_request` in the primary
   workspace repository. Preserve every existing safety boundary:
   - `-B|--do-not-close-bead` always opts out;
   - proposals and non-landed methods never close;
   - non-`in_progress`, unreadable, or unassigned beads never close;
   - commits in linked repositories and SDD sidecars never close the workspace bead;
   - closure remains best-effort after the commit has landed and reports an actionable
     warning on failure.

   Continue to perform closure through `sase bead close --resolution done --note ...` so
   phase epic-symbol checks and plan/epic descendant guards remain authoritative. Rename
   task-specific helpers, docstrings, and status text to describe the generalized
   behavior. Keep resume/checkpoint behavior idempotent by recording the existing
   `close_bead` completed step only after a successful close.

4. Synchronize user-facing CLI and skill documentation for `-B` so it describes the
   assigned in-progress bead rather than only a task bead. Retain the recovery guidance
   that an already-closed bead is safe and that failed lifecycle validation must be
   resolved explicitly.

## Tests and validation

1. Extend bead-hook tests to prove successful auto-close for `task`, `phase`, and `plan`
   issue types, including the generated close note and generalized success text. Keep
   coverage for the opt-out, non-`in_progress` statuses, unreadable state,
   sidecar/linked repositories, unsupported methods, and a rejected close command.

2. Update commit-workflow tests for the generalized helper name and verify closure only
   occurs after successful dispatch/tracking, including pull-request and resumable
   paths. Update CLI parsing/help and skill-source contract tests so `-B` and the
   mandatory `/sase_final` wording cannot regress to task-only or optional language.

3. Extend memory-generation assertions to require the stronger terminal-turn wording in
   generated home/project instructions and provider copies. Verify
   `sase memory init --check` reports no drift after regeneration.

4. Run `just install` before repository checks, execute the focused commit, finalizer,
   skill-source, and memory-generation tests, then run `just check`. If the scoped check
   escalates or reports unusual selection, use `/sase_monitor` for `just check-full` as
   required by repository policy.

## Expected result

Agents receive an explicit, high-salience rule that ending any ordinary turn requires
`/sase_final`, while intentional mechanical handoffs remain exempt. If an agent still
forgets and declaration recovery creates the final commit, the associated phase, plan,
or task bead closes automatically unless `-B` was used or the canonical bead lifecycle
checks reject the close.
