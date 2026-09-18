---
tier: tale
title: Finish repository declarations introduced during conflict repair
goal:
  Commit validated remaining repository work after conflict repair without replaying
  landed commits or weakening finalizer safeguards.
size: medium
proposed_by: bbugyi200.athena.0mv
create_time: 2026-09-18 09:40:07
status: wip
---

# Finish declared repository work introduced during conflict repair

## Problem and evidence

The `sase-11y.5` coding run failed on 2026-09-18 after its service implementation had
already landed. Its planner and automatic plan approval succeeded. The main commit,
`3fb42fa11ee2ba0539a085484edb2e3f98e6dd1f`, is recorded as pushed. During the commit
finalizer's conflict-repair turn, verification exposed an LSP catalog warmup defect; the
repair agent changed `crates/sase_xprompt_lsp/src/server.rs` in linked `sase-core`. It
reported passing the focused Rust regression, 21 Python parity tests, and the main
`just check`.

The host accepted its subsequent declaration at `2026-09-18T13:11:13Z`, with message
`fix(lsp): warm hold target completions` and `bead_action: keep` for
`sibling:sase-core`. No stitch attempt for that repository followed. The finalizer
failed with `dirty_after_commit_decisions` and left the phase bead in progress. The
final dirty-work guard was correct; the missing dispatch before it was not.

Completed local evidence is under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/18/20260918054429/`:

- `final_submission.json`, `final_submission_host.json`, and
  `final_submission_attempts.jsonl` prove acceptance of the repair declaration.
- `finalizer_result.json` proves the terminal diagnostic and pushed main commit.
- `commit_results.json` and `finalizers/commit/attempt-1.main.*` show the main commit
  and the absence of a linked-repository attempt.
- `finalizers/commit/conflict_repair_response.main.md` records repair and checks.

The stable error transcript is
`~/.sase/chats/202609/gh_sase_org__sase-gh-workflow_gh_main_ERROR-260918_054435.md`. The
originating approved feature plan is `plan:202609/service_platform_units.md`.
Publication warnings in the run are separate from this fatal finalizer error.

Current code reproduces the failure. `execute_commit_finalizer()` snapshots the accepted
context, decisions, and ordered dirty repositories before dispatch.
`dispatch_commit_decisions()` iterates that fixed list. After conflict repair,
`attempt_post_repair_follow_up()` reloads a declaration only if the repaired repository
itself remains dirty. A clean main repository plus a newly dirty linked repository
bypasses that path. An isolated invocation with mocked VCS operations produced
`stitch repositories: ['main']`, `repair declaration loads: 0`, and
`residual repositories: ['core']`. The controller does not retry this diagnostic;
raising the attempt limit alone cannot solve it.

## Outcome and scope

A successfully resumed conflict repair must hand all remaining, explicitly declared
repository obligations back to the host for bounded execution. This includes a linked or
external repository first opened or changed during repair, as well as updated work in
repositories already in the initial dispatch list. Already-landed work must retain its
evidence and must not be committed again. Undeclared or stale work must still fail
safely with actionable diagnostics.

This is one bounded finalizer change with Rust policy, Python integration, and
regressions. It does not redesign commit finalization, alter global retry limits, repair
publication infrastructure, or automatically rerun/close `sase-11y.5`. Recovering that
historical run's retained patch is separate from implementing the general fix; do not
replay its already-pushed service implementation.

## Implementation

1. **Make the repair handoff explicit.** In
   `src/sase/finalizers/commit_repair_conflict.py`, `commit_dispatch.py`, and
   `commit_dispatch_types.py`, propagate successful conflict-repair completion to the
   executor, including the path where resume settled the operation without a new commit
   marker. Before continuing dispatch, load the accepted repair declaration and its host
   repository snapshot through `load_accepted_commit_declaration()`. Refresh dirty state
   after reconciliation. Perform this refresh even when the repaired repository is now
   clean. Keep filesystem observation, provider invocation, and stitch execution in the
   Python host adapter.

2. **Put remaining-work selection in Rust core.** Open `sase-core` through `/sase_repo`.
   Add a small pure policy under `crates/sase_core/src/finalizer/` that takes validated
   declaration/context identities, current repository obligation digests, and
   completed/executed obligation evidence and returns ordered remaining obligation IDs
   or a typed diagnostic. Expose it through `crates/sase_core_py` and the thin
   `src/sase/core/finalizer_facade.py`/wire adapters. Reuse existing submission,
   assigned-bead, and digest validation. Do not duplicate this scheduling policy in
   Python or migrate unrelated finalizer behavior as part of this fix.

   The refreshed snapshot must remain bound to the current run, agent, turn, and
   selected finalizer plan. Match repository IDs to host-owned identities and verify
   current obligation fingerprints before any new mutation; declarations never supply
   execution paths. Select all remaining obligations in host context order, including
   newly added repositories. A repair declaration supersedes pending decisions from the
   old snapshot: use its current message, `bead_action`, and host-adjudicated deferral
   rather than stale values cached before repair. Preserve proof for completed
   obligations omitted from the new context because they are now clean. Fail if a dirty
   obligation has no valid current decision, if its host identity is missing/mismatched,
   or if it changed after submission.

3. **Execute one bounded continuation sweep.** Integrate the selection at
   `src/sase/finalizers/commit.py` and `commit_dispatch.py`; share the existing
   `commit_dispatch_followup.py` safeguards instead of introducing recursive
   `run_finalizers()` calls. Reconcile the original pending queue with the new snapshot
   so no repository is dispatched twice from both queues. Keep one follow-up commit for
   residual changes in the repaired target, and process the other validated remaining
   repositories once in the continuation sweep.

   Preserve the existing per-instance attempt accounting, protected baseline exclusions,
   no-progress protections, accepted deferrals, reconciliation proof, marker
   verification, push/checkpoint recovery, and final dirty-work guard. Treat this as
   continuation of the current dispatch attempt, consistent with existing
   same-repository follow-up accounting. Permit at most one such sweep per top-level
   attempt; do not recursively expand it or launch another repair turn for a conflict
   encountered in the sweep. Report that conflict or further new/changed obligations as
   recoverable terminal failures, retaining their files and evidence. Existing one-shot
   conflict behavior outside this new continuation remains covered by regression tests.

   Resolve linked/external baseline attribution using the existing repository enrollment
   and protection machinery. Never infer ownership merely from a dirty path being
   present. A support repository's `keep` decision must not close the assigned phase or
   overwrite the main repository's recorded bead decision.

4. **Report partial success precisely.** Preserve per-repository commit SHAs and push
   evidence across the handoff. Record which accepted declaration supplied each
   continuation decision and the result of each attempted repository. Errors should
   state which commits are already recorded as landed/pushed, which repository and paths
   remain, and whether the reason is a missing/stale declaration, execution failure,
   unresolved conflict, or the continuation bound. Propagate that message through
   existing finalizer results and error reports. Do not imply the initial implementation
   must be repeated or claim a push without supporting evidence. Keep normal
   non-conflict error behavior stable. The existing repair prompt already requires
   declarations for every changed repository; align its description of bounded host
   execution if needed, without replacing this code fix with prompt advice.

## Verification and acceptance

Extend the existing finalizer harness, particularly
`tests/test_finalizers_protocol_harness_multi_repo.py` and
`tests/test_commit_dispatch_conflict_repair_followup.py`. Use temporary repositories,
real context publication/submission validation, and injected provider/VCS runners; do
not mutate retained historical workspaces or launch live agents to reproduce the
failure.

- Reproduce the incident: only main is initially dirty; repair resumes main, leaves it
  clean with pushed commit evidence, introduces a linked-repository change, and submits
  an accepted declaration. Assert main is not replayed, linked work is committed once
  with its declared message and `keep`, all attributable work is settled, and the
  finalizer succeeds with both results.
- Cover an external repository, multiple remaining repositories, reversed manifest
  order, and a repository already queued whose contents/message change during repair.
  The latest valid snapshot and host order must win.
- Cover simultaneous residue in the repaired repository and a linked repository, plus a
  resume that settles without a new marker. Neither early `continue` path may skip the
  handoff; marker-free resolution must meet existing settled repository checks.
- Cover a missing declaration, missing host identity, wrong run/turn/plan, post-submit
  content changes, protected preexisting files, and accepted deferrals. Prove no
  unauthorized stitch is attempted and deferrals retain their established semantics.
- Prove boundedness and recovery: continuation conflicts, residual dirt, and further
  newly introduced obligations terminate without recursive repair; a later retry honors
  checkpoints and does not duplicate landed commits. Assert attempt accounting and
  retained evidence. Keep existing same-repository follow-up, no-commit resume, and
  non-conflict tests green.
- Add Rust policy and PyO3 binding tests for the corresponding deterministic selection
  and rejection cases, plus Python facade coverage. Add an error-report regression that
  names the landed main SHA and remaining linked work.

Run focused finalizer, declaration, reconciliation, and ledger tests, then read the
current lint/test guidance and run `just check` in SASE. Run the linked core's required
`just check`/`scripts/check.sh`, including PyO3 binding tests; a
`cargo test -p sase_core` run alone is insufficient. Follow the current core dependency
publication/floor workflow for any added binding, without manually bumping Rust release
versions or adding a Python fallback. Use `/sase_monitor` for long verification and for
any required exhaustive `just check-full` gate.

Acceptance requires the incident-shaped test to fail on the old dispatch path and pass
with the change, with no relaxation of declaration, attribution, or dirty-work
validation.
