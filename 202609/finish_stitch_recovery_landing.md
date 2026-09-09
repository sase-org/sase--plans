---
tier: epic
status: done
title: Finish stitch recovery deadline and published-core landing
goal:
  Hidden artifact-sidecar setup obeys the chop deadline, and SASE's published Rust core
  floor, lock, and revision pin select the first complete release containing the
  authenticated checkpoint recovery contract without regressing newer core consumers.
parent_bead: sase-yh.5
phases:
  - id: finish-landing
    title: Finish deadline propagation and published-core integration
    size: medium
    depends_on: []
    description:
      "finish-landing: propagate the chop deadline through hidden-sidecar clone and
      integration work, ratchet SASE to the first complete published core release with
      authenticated checkpoint recovery, integrate post-start drift, and run focused,
      clean-floor, repository, and full landing verification."
proposed_by: bbugyi200.athena.sase-yh.5.land
bead_id: sase-yh.5.4
create_time: 2026-09-09 19:52:26
---

- **PROMPT:**
  [prompts/202609/finish_stitch_recovery_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_stitch_recovery_landing.md)
- **PARENT:**
  [202609/stitch_recovery_landing_repairs.md](stitch_recovery_landing_repairs.md)
- **BEAD:**
  [sase-yh.5.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yh/sase-yh.5.4.md)

# Finish stitch recovery deadline and published-core landing

## Audited starting point

The `sase-yh.5` land audit confirmed the phase-1 checkpoint implementation in SASE
commit `27bbd2f4e` and sase-core commit `7af2640`: automatic recovery sends normalized
full commit payload, repository, run, agent, and operation identity to the Rust schema-2
decision; foreign or missing identity and unproved legacy ownership fail closed; and
push-failure persistence distinguishes checkpoint-only, marker-only, and total write
failure. The phase-2 implementation in `a1b08d06c` preserves mismatched, dirty,
missing-upstream, divergent, and unpublished hidden clones, records durable retry state,
and rotates retry roles.

Two epic-owned gaps remain:

1. `src/sase/sdd/_artifact_link_machine_store.py:_ensure_hidden_document_root()`
   receives the chop's monotonic `deadline` and uses it for preliminary Git probes, but
   drops it when calling `ensure_sidecar_sdd_clone()`. Missing-clone setup and fresh
   pull/integration can therefore use their normal network timeouts after the chop
   budget expires. Existing chop tests prove the deadline reaches the resolver, not that
   this final call preserves it.
2. sase-core `v0.32.54` (`e2836d8`) contains `7af2640`, the later xprompt completion
   contract at `cb669ec`, and the release formatting repair at `c2161c7`, but
   `just ratchet-core-window --report-only` still reports 0.32.53 as the newest
   _complete_ PyPI release. SASE consequently remains at
   `sase-core-rs>=0.32.53,<0.33.0`, a 0.32.53 lock, and revision `cb669ec`; the revision
   ratchet already reports `e2836d8` pending. Do not hand-pin an incomplete release.

Post-start integration was reviewed through current master. `1852f091a` advanced the
core revision for ACE/LSP star-alias completion, and `f4ca78c0f` raised the published
floor for the unconditional queue directive. The final ratchet must preserve both
contracts. The sase-github checkout has no post-start commits and still imports
`reconcile_managed_checkout_origin` from the post-split `sase.workspace_provider.utils`
facade before provider assertion, so it needs no source change unless new drift appears
before implementation.

The unrelated phase-1 proposal about `sase-core/scripts/check.sh` was reproduced and
corroborated on existing task `sase-xv`; it is outside this plan.

## Implementation

1. Thread `deadline` through the final `ensure_sidecar_sdd_clone()` call in
   `_ensure_hidden_document_root()`. Add focused tests that observe the exact deadline
   for both missing-clone materialization and existing matching-clone fresh integration,
   while retaining the non-destructive mismatch/dirty/unpublished guards. Recheck every
   clone, pull, Git probe, state-lock, and publication-worker wait reachable from the
   chop path for another dropped deadline; fix and regress any equivalent epic-owned
   omission found.
2. Wait until `just ratchet-core-window --report-only` identifies a complete published
   release containing sase-core commit `7af2640` (expected `0.32.54` or a later normal
   patch release). Verify the published wheel exposes pending-checkpoint schema 2 and
   accepts the authenticated request fields; a tag or local editable build is not
   evidence. Apply the supported `just ratchet-core-window` and
   `just ratchet-core-revision` commands. Confirm `pyproject.toml`, `uv.lock`, and
   `sase-core-revision.txt` agree with the complete release/default-branch revision and
   still include the xprompt star-alias and unconditional queue contracts.
3. Re-audit commits landed after this plan was written for overlap with checkpoint
   recovery, workspace-origin facades, artifact-link eligibility/outbox and sidecar
   materialization, publication retry, and core dependency metadata. Integrate actual
   conflicts or duplicate paths without weakening the established fail-closed and
   non-destructive behavior.

## Verification

- Run `just install` before Python checks.
- Run the focused checkpoint/finalizer/workflow/marker suites and the focused
  artifact-link machine-store, publication-retry, and chop integration suites, including
  the new end-to-end deadline assertions.
- Exercise the installed published minimum rather than only the editable core build;
  confirm schema 2 and the xprompt/queue bindings required by current Python.
- Run `git diff --check`, `just fmt`, and `just check` in every changed repository.
- Run `just check-full` only through `/sase_monitor`, with the required `TESTING` and
  `TESTED` status labels, and preserve unrelated failures as evidence-backed follow-up
  proposals rather than changing baselines.

## Boundaries

- Shared recovery authorization remains in Rust core; do not add a Python fallback.
- Never delete, reset, replace, or integrate away a dirty or unpublished sidecar.
- Do not ratchet to a merely tagged or partially published core release.
- Do not implement `sase-xv` here.
- Do not close `sase-yh.5`, retire epic-symbol entries, run the final Symvision landing
  pass, or mark either linked epic plan done. The `parent_bead` handoff returns those
  responsibilities to the waiting land agent.
