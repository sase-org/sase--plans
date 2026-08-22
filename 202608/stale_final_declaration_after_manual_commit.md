---
tier: tale
title: Reject stale final declarations before acceptance
goal:
  Prevent an attributable manual commit between final-context publication and
  declaration submission from turning a completed agent into a discarded-work failure.
size: medium
proposed_by: bbugyi200.athena.0b6
---

# Reject stale final declarations before acceptance

## Goal

Make the final-declaration boundary accurately reflect live repository state. If work
changes after `sase final context` publishes a manifest template, `sase final submit`
must reject that stale template before recording an accepted declaration and tell the
agent to refresh the context. Generated skills must also make clear that a
`builtin@commit` declaration is executed by the host and must not be preceded by a
manual `/sase_git_commit` pass.

## Root cause and current state

Agent `0b5` successfully implemented and published its intended release-index fix as
`c89fede54`. The subsequent release metadata commits show that the product change was
not lost. The agent still finished as `FAILED` because its finalization sequence was:

1. `sase final context -f json` published a dirty `main` repository obligation for
   `.github/workflows/publish.yml` and `tests/test_github_actions_ci.py`.
2. The agent interpreted the `/sase_git_commit` description's “post-completion
   finalizer” language as an instruction to commit those paths manually. The wrapper
   recorded the attributable `c89fede54` marker and left the checkout clean.
3. The agent submitted the older dirty-repository manifest. Submission re-read only the
   on-disk context artifact, not live repository state, so it accepted the stale
   declaration.
4. `builtin@commit` correctly refused to count a marker already present in its
   pre-execution ledger as a new execution-time commit and emitted
   `dirty_work_discarded` / `HEAD did not advance`.

No commit after `0b5` changes final-declaration acceptance or the generated commit-skill
guidance. The active `mixed_reconciliation_declaration` implementation addresses a
related but later window: machine-owned partial commits performed by finalizer
preparation after an accepted submission. It does not cover repository mutation between
`sase final context` and `sase final submit`. This plan should compose with that work
and must not weaken its execution-time attribution checks.

## Implementation

1. Refactor `src/sase/finalizers/declaration.py` so context publication and submission
   share one internal live-context builder. Under the existing declaration lock, build
   requirements, repository obligations and digests, model-visible context, and
   host-only repository identities from one dirty-state snapshot. Preserve the current
   opaque wire format and lock order.

2. In `submit_final_manifest()`, after validating the supplied envelope against the
   published context and re-reading the context artifact, recompute the live context
   with the same authenticated plan and run identity before accepting the submission. If
   its digest or host repository set differs from the published snapshot, append a
   rejected submission-attempt record with `stale_final_context`, do not write
   `final_submission.json` or `final_submission_host.json`, and return an actionable
   error directing the caller to rerun `sase final context`. A refreshed clean context
   must require no commit payload; a refreshed still-dirty context must provide a new
   template and digest.

3. Keep the executor fail closed after acceptance. Do not treat pre-existing
   `commit_results.json` markers as new finalizer evidence, do not permit mutations that
   occur after a successful `sase final submit`, and do not weaken the discarded-work,
   stale-fingerprint, checkout-matching, or publication guards. Reconcile any nearby
   edits from the approved mixed-reconciliation work by preserving its distinction
   between submission-time state, pre-reconciliation state, and post-reconciliation
   execution state.

4. Update the canonical generated-skill sources in
   `src/sase/xprompts/skills/sase_final.md` and
   `src/sase/xprompts/skills/sase_git_commit.md`. State that a `commit` action in the
   `/sase_final` manifest is declarative and `builtin@commit` runs `sase stitch create`;
   agents must not invoke `/sase_git_commit` after reading a required final context.
   Make `/sase_final` rerun `sase final context` and rebuild or abandon the manifest on
   `stale_final_context`. Narrow `/sase_git_commit`'s “post-completion finalizer”
   wording so it applies only to an explicit host instruction to invoke that skill, not
   the provider-neutral `/sase_final` flow. Update
   `tests/main/test_init_skills_source_content.py` to lock in these boundaries. Preview
   generated output with `sase skill init --diff`; do not deploy generated skills from
   the implementation workspace.

5. Add focused declaration-channel tests in
   `tests/test_finalizer_declaration_channel.py`: publish a dirty context, change its
   fingerprints or make the repository clean before submit, and prove that the old
   manifest is rejected without an accepted-submission artifact. Assert the attempt
   diagnostic, refreshed-context path, stable behavior when state is unchanged, and
   serialization with a concurrent republish.

6. Add a real-git regression in `tests/test_finalizers_live_e2e.py` matching `0b5`:
   publish a dirty context, create an attributable run-owned stitch and marker before
   submitting the old manifest, verify submission rejects it, then republish the clean
   context and complete without a recovery turn or `dirty_work_discarded`. Retain a
   negative case proving that a repository mutation after an accepted submission still
   fails at execution.

7. Update the final-declaration and discarded-work sections of
   `docs/commit_workflows.md` to document the three boundaries: context publication,
   submission acceptance, and finalizer execution. Explain that repository changes
   before submission require a refreshed context, while changes after acceptance remain
   a fail-closed protocol violation even when they carry this run's provenance.

## Verification

- Run the generated-skill source tests and preview with `sase skill init --diff`.
- Run `tests/test_finalizer_declaration_channel.py`, the focused commit-reconciliation
  and protocol-harness modules, and the relevant real-git finalizer tests, including the
  existing stale post-submit edit and mixed machine-reconciliation cases.
- Run `just install`, then `just check` as required for repository changes. If scoped
  selection escalates or reports unusual coverage, use `/sase_monitor` for
  `just check-full` with `TESTING` / `TESTED` statuses.

## Acceptance criteria

- Replaying `0b5`'s context → manual stitch → submit sequence yields a
  `stale_final_context` rejection at submission, not a failed agent after execution.
- Rerunning `sase final context` after that stitch produces a clean, no-payload context
  and normal completion.
- The ordinary `/sase_final` flow never instructs an agent to run `/sase_git_commit`;
  the host remains the sole executor of accepted `commit` declarations.
- A real mutation after accepted submission, a reset or discard without new evidence, an
  unrelated checkout marker, or unpublished machine-owned state still fails closed.
- The active mixed-reconciliation fix and its tests remain intact, focused tests pass,
  and `just check` passes.
