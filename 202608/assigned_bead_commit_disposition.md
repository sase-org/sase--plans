---
tier: tale
title: Make assigned-bead completion explicit and recoverable
goal:
  Assigned agents cannot commit or finish successfully without an explicit, verified
  bead disposition.
size: medium
proposed_by: bbugyi200.athena.0br
create_time: 2026-09-09 19:59:42
status: wip
---

# Plan: Make assigned-bead completion explicit and recoverable

## Context and confirmed root cause

The `sase-s9.3` agent was launched with `%id(3, clan=sase-s9, bead=sase-s9.3)`, and its
metadata and commit both retained that association. Its ordinary provider turn started
`just install`, allowed that command to continue as a provider-native background task,
and returned before running `just check`, `sase bead epic-symbols sase-s9.3`, or
`sase bead close sase-s9.3`. The restricted final-declaration recovery turn then
committed the dirty implementation as `3d20654` and reported success. The bead remained
`in_progress` until another actor closed it at 2026-08-23 10:06 EDT with the reason that
the agent had forgotten it.

This became a silent success because the commit CLI reads the assigned bead from
`SASE_BEAD_ID` but currently makes its disposition implicit: an eligible task bead is
auto-closed unless `-B/--do-not-close-bead` is present, while phase and plan beads only
produce a warning and are deliberately not auto-closed. The built-in commit finalizer
invokes `sase stitch create` without any bead-disposition option, and there is no
provider-neutral completion check between the ordinary model response and finalizers.

Requiring an explicit commit option is feasible and desirable, but it is not sufficient
on its own. In this incident it would make the machine-owned finalizer fail without
giving the ordinary agent a chance to finish verification, while having that finalizer
choose closure would incorrectly invent verification. The fix therefore combines an
explicit CLI contract with a pre-finalizer assigned-bead completion guard.

## Behavioral contract

### Explicit commit disposition

- Add mutually exclusive `-C/--close-bead` and `-B/--leave-bead-open` options to the
  shared parser used by both `sase commit` and `sase stitch create`. Retain
  `--do-not-close-bead` only as a hidden compatibility alias for `--leave-bead-open`, so
  existing `-B` callers keep the same intent while the public vocabulary becomes
  affirmative and symmetric.
- On every fresh CLI invocation with a nonblank `SASE_BEAD_ID`, require exactly one of
  the two dispositions before any bead sync, hook, staging, VCS, or Patch mutation. If
  neither was supplied, exit 1 and name the assigned bead plus both remediation
  commands. If no bead is assigned, reject either option as meaningless. A resume uses
  the disposition persisted in the original checkpoint and does not require or change
  it.
- Persist one canonical payload value such as `bead_disposition=close|leave_open`.
  Remove the implicit task-only default. `--leave-bead-open` must never close the bead
  and should report clearly that the assigned bead remains open.
- For `create_commit` and `create_pull_request`, `--close-bead` becomes a hard
  post-dispatch obligation after commit publication and tracking succeed. Close the
  exact assigned bead regardless of whether it is a task or phase, relying on
  `sase bead close` to enforce status, descendant, and epic-symbol guards. Treat an
  already-closed bead as success. If closure fails, return failure while retaining the
  commit checkpoint; `sase stitch create --resume` must retry only the unfinished
  close/tracking step and must not create a second VCS commit. Reject `--close-bead` for
  non-landing proposal mode with a useful instruction to use `--leave-bead-open`.

### Completion before finalization

- Add a provider-neutral assigned-bead completion guard immediately after the ordinary
  `provider.invoke(...)` returns and before any host finalizer runs. Resolve the exact
  assigned bead and workspace-bound store from the run's persisted metadata rather than
  guessing from the agent name or consulting a potentially stale primary-store
  projection.
- Continue normally when no bead is assigned, the bead is already closed, or an
  intentional plan, monitor, pipe, or questions handoff marker is pending.
- When an assigned bead remains open, invoke exactly one focused recovery turn with the
  same provider, model, invocation options, and artifact directory. Give it the prior
  response and current bead state; require it to finish only the missing work and
  verification, run the epic-symbol check for a phase, and close only the assigned bead
  with an honest note. If it still needs a long command or user input, it must use a
  SASE monitor/questions/plan handoff rather than provider-native background or wakeup
  tools. It must not submit a final declaration because finalizers run after this guard.
- After recovery, proceed only if the bead is closed or a supported handoff now exists.
  A missing/unreadable bead, an open bead after the one recovery turn, or a retry after
  the one-shot recovery budget has already been consumed must raise a typed completion
  error so the run cannot be recorded as `DONE`.
- Persist immutable recovery prompt/response artifacts and merge recovery text and token
  usage into the invocation result for diagnosis and accounting.

### Automated callers and instructions

- Make `builtin@commit` pass `--leave-bead-open` on each internally dispatched
  `sase stitch create` when `SASE_BEAD_ID` is set. This is an explicit, auditable
  disposition: the machine-owned finalizer is not allowed to assert verification or
  close the bead. By the time it runs, the completion guard must already have proved
  closure or preserved an intentional handoff.
- Update the shared finalizer commit instructions and generated
  `sase_git_commit`/`sase_hg_commit`/`sase_final` skill sources. Assigned agents must
  choose `--close-bead` when the landed commit completes verified work and
  `--leave-bead-open` for a genuinely mid-flight commit; phase workers must still run
  their required epic-symbol check. Explain that intentional open-bead completion uses a
  SASE handoff, not a normal successful return. Update wrapper/source-content tests with
  the new invocations; deployment of generated skills remains a post-land action, not
  part of this workspace change.

## Implementation outline

- Refactor the commit parser and handler to normalize and validate the new disposition
  before constructing `CommitWorkflow`, keeping the legacy alias isolated from the
  canonical payload.
- Generalize the task-only autoclose helpers into explicit assigned-bead disposition
  handling. Make post-commit closure strict and checkpointed, preserve bead commit tags
  and sync behavior, and teach both the normal and resume paths to distinguish `closed`,
  `left_open`, and `failed` outcomes.
- Extend the built-in finalizer stitch runner just enough to append the explicit
  leave-open option for bead-associated runs, without changing finalizer declaration
  schemas or allowing finalizers to close lifecycle objects.
- Add a focused runtime module for metadata/store resolution, handoff bypass, one-shot
  recovery prompting and artifacts, status revalidation, result merging, and the typed
  unresolved-bead error. Call it from the shared invocation pipeline before
  `run_finalizers(...)` so behavior remains uniform across providers.
- Update the in-repo generated-skill templates and shared commit instruction builder to
  describe the same contract and examples as the CLI.

## Tests and verification

- Cover both public command spellings, both new short/long flags, mutual exclusion,
  missing disposition for a nonblank assigned bead, blank/unset bead IDs, meaningless
  flags without an assignment, the hidden compatibility alias, proposal-mode rejection,
  and resume preservation of the original disposition.
- Cover explicit leave-open without closure; explicit close for task and phase beads;
  already-closed idempotence; lifecycle/epic-symbol close rejection; a strict
  post-commit close failure that resumes without a duplicate commit; and unchanged
  behavior for callers with no assigned bead.
- Cover built-in finalizer argv construction with and without `SASE_BEAD_ID`, proving
  that its internal calls explicitly leave the bead open.
- Cover completion-guard no-ops, every intentional handoff, successful one-shot
  recovery, merged response/usage and immutable artifacts, missing/unreadable bead
  state, an open bead after recovery, and exhausted recovery budget. Add an invocation
  ordering test proving this guard runs before finalizers for every provider path.
- Update generated-skill source-content and wrapper forwarding assertions. Preview the
  rendered skills with `sase skill init --diff` only; do not deploy global generated
  copies from the unlanded workspace.
- Run focused commit workflow, finalizer, runtime, and skill-template tests, then run
  `just install` followed by `just check` as required for repository changes. Escalate
  to monitored `just check-full` only if the scoped selector broadens or reports an
  unusual selection.

## Non-goals

- Do not silently auto-close an assigned bead merely because a finalizer found dirty
  files or a VCS commit landed.
- Do not make provider-specific exceptions for Claude background tasks or scheduled
  wakeups.
- Do not change the historical `sase-s9.3` close event, close ancestor plan/epic beads,
  or deploy generated skills globally before the implementation is committed and landed.
