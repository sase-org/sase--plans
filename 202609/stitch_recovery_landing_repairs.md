---
tier: epic
status: done
title: Finish stitch recovery ownership and publication preservation
goal:
  Automatic stitch recovery resumes only an authenticated run-owned durable checkpoint,
  artifact-link retry never destroys or falsely clears unpublished work and remains fair
  and deadline-bounded, and clean installs require the first published Rust release
  containing the corrected contracts.
parent_bead: sase-yh
phases:
  - id: checkpoint-proof
    title: Bind automatic checkpoint recovery to authenticated durable evidence
    size: medium
    depends_on: []
    description:
      "checkpoint-proof: compare checkpoint run, agent, repository, and complete
      accepted payload identity through the Rust-backed recovery decision; prove legacy
      ownership instead of asserting it, surface checkpoint and marker persistence
      failures, and cover foreign-run, same-subject/different-payload, malformed, and
      retry cases."
  - id: publication-preservation
    title: Preserve unpublished sidecars under retry and configuration drift
    size: medium
    depends_on: []
    description:
      "publication-preservation: make missing upstream an explicit failure, prevent
      mismatched-remote discovery or later store resolution from replacing unpublished
      clones, bound every probe and lock by the chop deadline, rotate retry roots
      fairly, and add the missing restart, aging, drift, contention, ownership, and
      deadline regressions."
  - id: published-integration
    title: Publish and ratchet the corrected recovery contract
    size: medium
    depends_on:
      - checkpoint-proof
      - publication-preservation
    description:
      "published-integration: publish the first normal sase-core-rs release containing
      the corrected checkpoint contract, ratchet sase's minimum and lock/revision files
      to that release, recheck post-epic workspace/commit/artifact-link integrations,
      and run focused, repository, clean-floor, and monitored full landing verification."
proposed_by: bbugyi200.athena.sase-yh.land
bead_id: sase-yh.5
create_time: 2026-09-09 19:52:48
---

- **PROMPT:**
  [prompts/202609/stitch_recovery_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/stitch_recovery_landing_repairs.md)
- **PARENT:**
  [202609/stitch_resume_publication_recovery.md](stitch_resume_publication_recovery.md)
- **BEAD:**
  [sase-yh.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yh/sase-yh.5.md)

# Finish stitch recovery ownership and publication preservation

## Why this remaining-work child exists

This is the child handoff from the interrupted landing of
`plan:202609/stitch_resume_publication_recovery.md` (`sase-yh`). It does not repeat the
managed-origin implementation, ordinary unpushed-marker settlement, the basic pending
after-hook resume, or the basic unchanged-sidecar retry that already landed. The parent
epic stays open because its source and test audit found acceptance boundaries claimed by
the closed phases but not actually enforced.

The audited baseline is `sase` commit `4068437a2`, `sase-core` implementation commit
`03ec116` followed by release tag commit `53fd0d1` (`v0.32.53`), and `sase-github`
commit `8e47665`. At proposal time the release tag existed on `sase-core`'s fetched
`origin/master`, but PyPI still returned 404 for `sase-core-rs==0.32.53`; do not treat a
tag or a local editable build as published clean-install evidence.

The original implementation commits were inspected directly:

- `sase` `46f7f549e`, `3ec9b78b2`, and `4068437a2`;
- `sase-core` `ff0a72e`, `d9ee8c2`, and `03ec116`;
- `sase-github` `d09ee25`.

Intervening `sase` changes include the artifact-link eligibility work (`dd1f829c2`),
sidecar hardening (`a5e642c92`), the workspace-utility split (`4de990b36`), the
commit-tracking split (`c6387f3b6`), and the queue migration through `ff6271e53`. The
origin reconciler remains re-exported by the post-split `workspace_provider.utils`
facade, and commit recovery landed after the tracking split. Preserve the eligibility
gate and unconditional queue contract. No remaining change has been found in
`sase-github`; its preallocated setup still calls origin reconciliation before provider
selection.

## Reproduced remaining gaps

1. `CommitCheckpoint.run_id` is written by `CommitWorkflow`, and the authenticated
   `FinalizerExecutionContext.run_id` is available, but
   `finalizers/commit_checkpoint_recovery.py:_recovery_request()` never compares them.
   It also sets `independent_ownership_evidence=True` for every legacy checkpoint and
   considers payloads matched from method/action plus only the first message line. A
   foreign-run checkpoint or a different full message with the same subject can
   therefore be automatically resumed. The phase note claimed foreign-run and payload
   mismatch refusal, while its only integration test uses matching `run-1`; the core
   `foreign_pending_checkpoint` test covers a foreign repository, not a foreign run.
2. `_record_unpushed_commit_marker_if_present()` ignores the return from
   `checkpoint_save()`, and `write_result_marker()` / its results upsert have no success
   result for the caller. If both writes fail, the function still returns true and the
   user is told recoverable evidence was durably recorded. This contradicts the parent
   requirement to surface persistence failure instead of representing missing evidence
   as success.
3. `_unpublished_sidecar_error()` returns `None` immediately when a committed root has
   no tracking upstream. That is false success, despite the parent contract requiring
   missing upstream to remain a diagnostic.
4. `machine_document_sidecar_roots()` calls
   `ensure_sidecar_sdd_clone(..., strict=True, fresh=False)` when an existing hidden
   clone's origin differs from current configuration. That strict path atomically
   replaces the clone and then deletes its backup. If the old clone has unpublished
   commits, discovery destroys the work before retry can revalidate it. Even if lazy
   discovery skips it, `_run_project()` subsequently calls
   `resolve_machine_artifact_link_store()`, whose strict materialization can perform the
   same replacement. Configuration drift must preserve and diagnose local-only work.
5. Retry deadline propagation stops at the publication worker. Local Git probes always
   receive a ten-second timeout and state locks always receive two seconds, regardless
   of remaining chop budget. Root iteration also always starts from the first document
   role and breaks when the deadline expires, so a slow first role can starve later
   roles in the same project across hourly ticks. The existing project cursor does not
   rotate roles.
6. The only end-to-end retry module test covers one successful plans publication. The
   approved plan also required regressions for legacy discovery, custom roles,
   repeated/backed-off/aged ticks, concurrent locking, already-published cleanup,
   divergent history, dirty and primary roots, missing upstream, remote changes,
   resolver failure, and time-budget fairness. Several underlying helpers have unit
   coverage, but these retry-specific guarantees are absent.
7. `pyproject.toml` and `uv.lock` currently select `sase-core-rs>=0.32.50`; the pending
   checkpoint binding first appears after the `v0.32.52` release. The new release must
   include the corrected ownership wire rather than merely ratcheting to the incomplete
   `03ec116` behavior.

## Constraints

- Read the parent plan through `sase artifact read` and reference memory through
  `sase memory read`. Open `sase-core` and any other repository through `/sase_repo`.
- Shared recovery authorization stays in Rust core. Python supplies authenticated host
  facts and performs filesystem, Git, subprocess, and persistence operations; do not add
  a Python policy fallback.
- Keep manual `sase stitch create --resume` compatibility distinct from automatic
  builtin-finalizer recovery. A legacy checkpoint may be manually recoverable while
  automatic recovery fails closed without independently verified ownership.
- Preserve conflict repair's one-shot budget, no-progress protection, bounded output,
  protected-path accounting, checkpoint evidence on failure, and single-operation marker
  settlement. Do not accept subject equality as operation ownership.
- Never delete, reset, replace, or integrate away an unpublished or dirty hidden sidecar
  to follow a changed remote. Emit actionable identity diagnostics and retain the old
  root/state for explicit recovery. Missing upstream or remote evidence is not
  publication success.
- State writes remain atomic and lock ordering stays compatible with the managed sync
  worker. All Git, lock, and worker waits must be clipped to the caller's remaining
  monotonic deadline. Fairness state must survive process restarts without letting an
  early project or role monopolize later ticks.
- Release-plz owns Rust versions. Publish through the normal release process, then use
  the supported core-window/revision ratchet. Never hand-pin an unpublished version.
- Read `lint_and_test.md` and `symvision.md` as applicable. Run `just install` before
  dependent Python checks and complete `just check` in every changed repository.
  `sase-core` verification includes PyO3. Run `just check-full` only through
  `/sase_monitor` with `TESTING` / `TESTED` status labels.
- Phase workers record outside-scope findings as `PROPOSED FOLLOW-UP:` notes. They do
  not create task beads, close this child or its parent, or mark either plan done.

## Phase 1: Bind automatic checkpoint recovery to authenticated durable evidence

Extend the versioned pending-checkpoint request/decision only as needed to distinguish
matching current-run evidence, foreign run/agent identity, and independently proven
legacy ownership. The Python adapter must derive those facts from the authenticated
finalizer context, checkpoint fields, accepted host repository identity, and the
accepted complete commit message. Do not turn artifact-directory colocation into an
unconditional proof. Compare normalized complete payload semantics relevant to the
builtin commit, not just the subject. Unknown method/action combinations fail closed.

Ensure new checkpoints persist stable operation, run, and agent identity before
dispatch. Automatic recovery must refuse a nonempty foreign `run_id`, a foreign
publication agent, a missing current identity, same-subject/different-body work, and a
legacy checkpoint without actual independent evidence. Preserve supported legacy manual
resume and the existing owned current-run after-hook/push/tracking cases.

Make checkpoint and unpushed-marker persistence observable to callers. A failed
checkpoint save or marker upsert must return an actionable stitch failure while leaving
whatever evidence did persist intact; never print that recovery evidence was recorded
unless at least the required durable representation exists. Cover checkpoint-only,
marker-only, and both-write failures without deleting the local commit or starting a
duplicate operation on retry.

Add Rust unit/PyO3 tests and Python integration tests for the above, including attempt
budget consumption, repository ordering, repeated failure, successful settlement, and
the existing conflict/no-progress lanes. Run the complete `sase-core` check and focused
commit/finalizer suites.

## Phase 2: Preserve unpublished sidecars under retry and configuration drift

Change inline publication verification so no tracking upstream returns a concrete error
and, for eligible machine roots, durable retry/diagnostic state where it can be
represented safely. Do not invent an upstream or report publication.

Separate non-destructive retry inventory from clone materialization. A missing hidden
clone may be materialized from its configured remote, but an existing Git clone with a
different remote, dirty worktree, local-only commits, missing upstream, or divergent
history must not be replaced or reset. Inspect and retain it, emit its project/role/root
and current/configured identity, and prevent the later machine-store resolver in the
same chop tick from deleting it. A clean fully-published mismatched clone may follow the
existing safe replacement path only after that state is positively proven.

Thread the remaining chop deadline through local Git inspection, state locking, clone
setup, and the managed publication worker. A timeout is a deferred/failed detail with
durable next-due state, never an unbounded overrun. Add a persistent per-project role
cursor or equivalent deterministic rotation so each eligible role gets bounded
opportunities across ticks, including when the first role consumes its allowance.
Remote/upstream identity changes should either migrate a positively safe record or
retain the old record with a diagnostic; do not silently orphan state.

Add isolated disposable-repository tests for restart with no mutation, legacy discovery,
custom roles, stable first-pending age across HEAD changes, exponential backoff and
six-hour warnings, repeated ticks, concurrent state locks, already-published cleanup,
rejected/divergent remote history, dirty roots, missing upstream, configured remote
changes, one broken role beside a successful role, deadline clipping and role fairness,
resolver failure, and refusal to mutate primary roots. Preserve the later artifact-link
eligibility gate from `dd1f829c2`.

## Phase 3: Publish and ratchet the corrected recovery contract

After phase 1 reaches `sase-core`'s default branch, wait for the first normal PyPI
release that contains the corrected wire and PyO3 bindings. Verify the exact published
wheel, then update `pyproject.toml`, `uv.lock`, and `sase-core-revision.txt` with the
supported ratchet commands. Clean-floor validation must install that exact minimum and
exercise all origin, publication-retry, eligibility, queue, and checkpoint bindings
consumed by current Python. A local editable extension is not clean-floor evidence.

Re-audit commits landed since this child began for overlap with workspace origins,
commit checkpoint/tracking, sidecar materialization, artifact-link eligibility/outbox,
and core dependency metadata. Keep the post-split facades and cross-repository callers
aligned; update `sase-github` only if drift actually changes its reconciliation entry
point.

Run focused real-Git recovery and publication suites, core unit/PyO3 tests, every
changed repository's required `just check`, the published-minimum/core-floor smoke, and
the monitored `just check-full` landing gate. Preserve exact unrelated failures as
follow-up proposals with evidence rather than weakening tests or baselines.

## Follow-up dispositions already completed by the parent land audit

- `sase-yh.1` note #1 was the exact post-close recurrence of the monitor supervisor
  flake tracked by `sase-lk`; the land agent added verified-after-close +1 evidence,
  reopening that task.
- `sase-yh.1` note #2 duplicates the clan-summary timeout flake `sase-xb`; the land
  agent added +1 evidence identifying the proposing phase.
- `sase-yh.4` note #1 is the queue migration failure already recorded three times on
  causally responsible active epic `sase-yj` and its flag bead `sase-yl`; no duplicate
  task or fourth copy of the same evidence was created.
- `sase-yh.4` note #2 duplicates deterministic pager collection task `sase-ym`; the land
  agent added +1 evidence identifying the proposing phase.
- `sase-yh.4` note #3 is one unrerun `WaitForScreenTimeout` observation for
  `test_config_jump_hint_moves_cursor_and_repaints_detail`. It does not yet meet the
  flake task type's fail-then-pass requirement. Re-run it on the installed combined tree
  during child landing; corroborate an existing task or create one through
  `/sase_new_task` only if it becomes a confirmed distinct flake.

The parent's dependency-floor note is not an unrelated follow-up: it is acceptance work
for this child. Record all final proposal outcomes, including the unconfirmed TUI row,
in the resumed parent close note.
