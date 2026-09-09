---
tier: epic
title: Retire the pluggable finalizers beta and legacy controller
goal:
  Make host-owned pluggable finalization unconditional, remove the deprecated Off path
  and beta compatibility code, prove the complete protocol end to end, and close flag
  bead sase-ro only after the combined tree is green.
phases:
  - id: harden-protocol
    title: Complete the finalizer protocol and parity harness
    depends_on: []
    description:
      "harden-protocol: turn the original acceptance matrix into executable
      generic-controller coverage, finish any missing fixed-point, retry, stale-state,
      conflict, and outside-agent behavior, and leave the flag-On path green before
      retirement."
    size: medium
  - id: retire-legacy
    title: Make pluggable finalizers unconditional and delete the old path
    depends_on:
      - harden-protocol
    description:
      "retire-legacy: delete the pluggable_finalizers flag and Off branch, extract any
      still-needed reconciliation helpers from the deprecated controller, remove
      beta-only config, environment, baseline, metric, and artifact writers, and
      preserve only deliberate historical readers."
    size: medium
  - id: synchronize-contracts
    title: Synchronize CLI, schema, docs, and generated skill source
    depends_on:
      - retire-legacy
    description:
      "synchronize-contracts: describe unconditional finalization across configuration,
      workflows, CLI and xprompt docs, remove beta and legacy guidance, update the
      canonical sase_final skill source and snapshots, and preview generated runtime
      skills without deploying a dirty tree."
    size: small
  - id: e2e-acceptance
    title: Run adversarial and live end-to-end acceptance
    depends_on:
      - synchronize-contracts
    description:
      "e2e-acceptance: exercise the combined unconditional path through disposable
      repositories and real SASE launches, repair every in-scope defect found, inspect
      finalizer artifacts and postconditions, and produce the evidence required for
      landing and closing sase-ro."
    size: medium
proposed_by: bbugyi200.athena.096
bead_id: sase-rr
create_time: 2026-09-09 19:51:29
status: wip
---

- **PROMPT:**
  [prompts/202608/retire_pluggable_finalizers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/retire_pluggable_finalizers.md)
- **BEAD:**
  [sase-rr](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rr/README.md)

# Plan

## Context and current-state findings

The implementation epic `sase-rn` is closed and all seven of its phases landed. Its
dedicated beta-removal task is `sase-ro` (`task(flag)`, currently open), whose removal
rule is to delete the disabled branch and make the enabled branch unconditional. The
current process has `pluggable_finalizers` forced on through `SASE_FEATURE_FLAGS`, but
the code-owned default is still off.

The retirement is not a four-call-site edit. The audit found all of the following:

- `invoke_agent()` still chooses between `run_finalizers()` and the old
  `run_commit_finalizer()` and records a legacy/pluggable parity metric.
- plan resolution, end-of-turn instruction injection, turn nonces, declaration CLI
  access, and declaration recovery are each separately flag-gated.
- the generic built-in commit executor imports machine-owned reconciliation helpers from
  the deprecated controller module, so that module cannot simply be deleted.
- legacy `commit.finalizer.*` settings and `SASE_DISABLE_COMMIT_STOP_HOOK` still map
  into the beta policy; the runner bootstrap also knows about the old environment
  bypass.
- the new baseline writer still emits `commit_finalizer_baseline.json`, and the generic
  commit executor still reads it as a fallback.
- the generic path still writes `commit_finalizer_result.json`; reporting correctly
  prefers `finalizer_result.json` but retains a useful fallback for historical agents.
- much of the mature commit safety corpus calls `run_commit_finalizer()` directly, while
  the generic-controller tests cover only a small happy-path subset.
- the controller is a single ordered pass. The original binding design requires a
  bounded fixed point when a later finalizer creates new repository work, plus explicit
  cycle/no-progress handling. This and the real conflict-resume path need executable
  acceptance evidence before the old path is removed.
- the canonical `sase_final` skill source still calls the protocol beta-only, and it has
  not been deployed into the currently available skill catalog. Generated skills may be
  deployed only from a clean, landed canonical tree.

This plan treats the deprecated code as the flag-Off orchestrator and beta migration
surface. It does **not** delete safety behavior merely because it currently lives in a
`commit_finalizer_*` module. Shared dirty-state discovery, attribution, protected
baselines, discarded-work detection, publication checks, commit evidence, and conflict
checkpoints remain required and should move behind neutral built-in-finalizer APIs where
needed.

Historical artifacts are immutable compatibility data. Stop producing beta-only
projections and filenames where the generic format supersedes them, but retain narrowly
scoped read-only fallbacks in reporting/UI when they are needed to render agents created
by older releases. `commit_results.json` and `commit_state.json` are current proof and
resume ledgers, not obsolete compatibility artifacts.

Memory notes are excluded because the current conversation does not explicitly approve
editing `sase/memory/*.md`. The audited `xprompts.md` note does not yet list `%final`;
the land agent should use `/sase_new_task` to deduplicate and record that memory
follow-up if it is still stale, rather than editing memory under this plan.

## Phase 1: complete the protocol and parity harness

Start with `just install`. Convert the binding design's acceptance matrix and the mature
legacy commit-finalizer corpus into tests that enter through the sealed plan,
declaration, generic controller, built-in commit executor, and host postcondition seams.
Do not preserve tests whose only purpose is to prove that the flag-Off branch exists.

The executable matrix must cover:

- an invocation outside a SASE agent or without an artifacts directory remaining a safe
  no-op rather than trying to publish declaration state;
- omitted `%final`, ordered add/remove operations, `%final:none`, required instances,
  dependency order, and launch-time rejection before the provider's first turn;
- clean commit-only completion without a recovery call, one and only one missing/stale
  declaration recovery, stale nonce/plan/context/repository-state rejection, and
  intentional handoffs never entering recovery;
- ordinary dirty primary repositories plus linked, opened external, SDD, plans,
  research, hidden agents-sidecar, and family-inherited repositories;
- protected pre-existing dirt, a late-opened dirty repository, repeated/concurrent open
  capture, run-owned SHA/tree evidence across rebase, discarded work, partial coverage,
  and publication failure;
- sequential multi-repository commit decisions, exact refusal reason preservation,
  first-repository conflict blocking later dispatch, successful resume of the same
  stitch, contradictory/stale checkpoints, and a second unresolved conflict;
- isolated plugin describe/validate/execute/verify, constrained command execution,
  timeouts, malformed or oversized output, disabled/mismatched providers, and sanitized
  argv/cwd/environment behavior;
- a later mutating finalizer creating attributable work, which must reactivate commit
  before success, bounded by deterministic controller-cycle and no-progress limits;
  per-instance `max_attempts`, usage accumulation, and generic metrics/artifacts must
  describe every retry and terminal outcome.

Implement missing host-owned controller behavior exposed by this matrix. Recompute
triggers and submission requirements after every mutating executor, invalidate stale
declarations before mutation, ask for new model input only when the refreshed obligation
requires it, preserve distinct declaration-recovery and conflict-repair budgets, and
fail closed when progress or budgets are exhausted. Shared wire validation belongs in
`sase-core`; if a test exposes a frontend-neutral contract defect, open that linked repo
through `/sase_repo`, update the Rust wire/bindings and their tests, then adopt the
released floor here. Host orchestration, subprocesses, filesystem evidence, and LLM
recovery remain in Python.

Run the focused finalizer, invocation, baseline, plugin, reporting, xprompt-directive,
and bootstrap suites, followed by `just check`. Fix failures rather than weakening
assertions or carrying an unexplained parity exception into retirement.

## Phase 2: remove the flag and deprecated implementation

Make the sealed-plan path the only normal completion path:

1. Resolve and persist the finalizer plan for every SASE agent invocation; append the
   `sase_final` terminal instruction and mint/restore the turn nonce whenever a selected
   plan and artifacts directory require them; run the generic controller unconditionally
   after a successful provider return.
2. Keep calls outside a SASE agent compatible with the pre-finalizer API contract, and
   make explicit `%final` selection work without an opt-in error.
3. Remove `FeatureFlag.pluggable_finalizers`, its registry definition and schema entry,
   every call-site branch, flag-state tests/overrides, beta-only diagnostics, and the
   legacy/pluggable parity metric.
4. Move the machine-owned auto-commit, bead publication, discarded-work, failure
   rendering, and result helpers still used by `builtin@commit` into appropriately named
   finalizer modules. Then delete `run_commit_finalizer()` and its deprecated direct
   orchestration/import shim. Port surviving tests to public generic entry points or
   focused helper APIs.
5. Delete the legacy `commit.finalizer.enabled` / `max_passes` defaults and schema,
   their config-layer adapter/diagnostics, and the deprecated
   `SASE_DISABLE_COMMIT_STOP_HOOK` bypass. The supported equivalents are trusted
   `finalizers.defaults`, per-instance `max_attempts`, `%final:none`, and
   required-instance policy. Update bootstrap baseline capture so it no longer consults
   the removed bypass.
6. Stop dual-writing `commit_finalizer_baseline.json` and
   `commit_finalizer_result.json`; make `finalizer_baseline.json` and
   `finalizer_result.json` authoritative. Remove dead legacy prompt-artifact recognition
   and compatibility helpers after proving there is no current writer. Preserve only
   explicit read fallbacks that render already archived agents, with regression tests
   that make the historical-only boundary obvious.

Run `rg`-based absence assertions, `tools/check_feature_flags`, focused migrated suites,
and `just check`. `sase flag show pluggable_finalizers` should report an unknown flag,
whereas `sase final list/show/doctor`, `%final`, and active-turn `context`/`submit` must
work without feature configuration.

## Phase 3: synchronize public contracts and generated skill source

Update `default_config.yml`, `sase.schema.json`, CLI help/completion snapshots, ordinary
documentation, examples, and tests so there is one unconditional host-owned finalizer
story:

- `finalizers` is the only completion-finalizer configuration block;
- `%final` is a normal repeatable directive, including `none`, removal, dependency, and
  required-instance semantics;
- `sase final list/show/doctor/context/submit` and `/sase_final` are no longer described
  as beta-only;
- the built-in commit provider's declaration, refusal, sequential stitch, protected
  dirt, evidence, publication, fixed-point, and conflict-repair guarantees are explicit;
- new agents write generic plan/context/submission/result/per-instance artifacts, while
  the reporting fallback for historical commit-finalizer artifacts is labeled read-only;
- removed legacy config and environment switches are not advertised as supported.

Edit only the canonical skill template `src/sase/xprompts/skills/sase_final.md`, update
its source-content/rendering tests, and preview all runtime variants with
`sase skill init --diff` or `--dry-run`. Do not edit chezmoi-managed generated
`SKILL.md` files and do not deploy from a phase workspace or a dirty/unmerged tree. Run
focused docs/schema/completion/skill tests and `just check`.

## Phase 4: adversarial and live end-to-end acceptance

Reinstall from the combined dependency state with `just install`, then run the complete
focused matrix from Phase 1 against real temporary Git repositories and local bare
remotes. Use disposable paths created for the test, never an existing user repository.
Where an actual SASE agent is required, request it through `/sase_run`; inspect the
agent row, response, VCS state, and emitted JSON/artifact files rather than accepting
process exit alone.

At minimum, execute and retain evidence for these live scenarios:

1. default clean completion: no declaration-recovery turn, no commit, successful generic
   result;
2. default dirty completion: `/sase_final` submits one commit decision, exactly one
   stitch is recorded and published, protected starter dirt is excluded, and the tree
   reaches the expected clean-relative-to-baseline state;
3. `%final:none`: no selected instance and no commit mutation;
4. configured `builtin@command` and the installable fixture plugin: deterministic order,
   bounded worker execution, validation and verify evidence, and no installation-only
   activation;
5. refusal: exact nonblank reason in the aggregate result, failed completion, and dirty
   work preserved;
6. stale declaration and post-submit edit: rejection before mutation followed by at most
   one fresh recovery;
7. later-finalizer dirt: commit becomes pending again and the controller reaches a
   bounded fixed point;
8. forced first-repository conflict: the second repo is untouched, one repair turn
   resumes the same checkpoint, and repeated/contradictory failure leaves diagnostic
   state intact;
9. an intentional plan/monitor/question handoff: mechanical termination with no
   declaration-recovery turn.

Fix every finalizer defect found, add the smallest regression that proves it, and rerun
the affected scenario plus `just check`. Do not dismiss a failing acceptance case as a
soak caveat. If a failure is demonstrably unrelated, follow the project's task-bead
policy rather than weakening this gate.

## Combined-tree landing, deployment, and bead closure

The land agent must run `just install`, `just check`, and then `just check-full` through
`/sase_monitor` because this epic touches the invocation seam, configuration/schema,
generated skill source, broad finalizer safety code, and a large cross-cutting test
surface. Review the full-tree diff for accidental removal of current safety primitives,
run `tools/check_feature_flags`, and confirm there are no source, schema, docs, test, or
completion references to `pluggable_finalizers`.

Only after the combined tree is committed and landed on the canonical branch, deploy the
updated generated skill with `sase skill init --force` and apply chezmoi if the command
reports that step was skipped. Verify `/sase_final` is discoverable in each supported
runtime's generated skill inventory.

Record the focused/live/full-suite evidence on `sase-ro`, then close it with:

```bash
sase bead close sase-ro --note "Host-owned pluggable finalizers are unconditional; the deprecated Off path and beta compatibility writers are removed; focused, adversarial, live, and full-suite acceptance passed."
```

Do not close the bead if the flag definition survives, the generic result is not the
only newly written aggregate, the live commit/refusal/conflict matrix has unexplained
failures, generated skills are not deployed from the landed tree, or any required test
is red.
