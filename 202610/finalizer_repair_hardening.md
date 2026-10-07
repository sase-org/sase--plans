---
tier: epic
title: Commit finalizer repair hardening after the sase-1h7/sase-1h8 failures
goal: 'Host-owned commit completion survives a conflict in a revision-pinned sibling,
  a conflict-repair turn that already ran `sase stitch create --resume`, and a paused
  rebase inherited from an earlier run, without stranding or falsely failing work.
  Finalizer-owned repair turns can no longer hand off and kill their own finalizer,
  and wait alerts say "can never self-resolve" only when that is true.

  '
phases:
- id: pin-after-handoff
  title: Revision-pin follow after the repair-handoff check
  depends_on: []
  size: medium
  description: 'pin-after-handoff: move the host revision_pin write for a conflict-repaired
    pinned sibling to after the repair-remaining handoff validates the main repo obligation,
    so the host''s own pin write no longer trips "repository obligation changed after
    submit"; add the missing pin-plus-repair regression tests.

    '
- id: repair-resume
  title: Conflict-repair resume never strands or falsely fails a commit
  depends_on: []
  size: medium
  description: 'repair-resume: make `sase stitch create --resume` publish an unpushed
    rebased HEAD instead of reporting "nothing to finish", and make the host conflict-repair
    path verify repository state (including ahead-of-upstream) when the repair turn
    already consumed the run-owned checkpoint.

    '
- id: owned-turn-handoffs
  title: Finalizer-owned turns refuse turn-ending handoffs
  depends_on: []
  size: medium
  description: 'owned-turn-handoffs: share the SASE_FINALIZER_OWNED_TURN guard that
    gate turns already use, refuse `sase monitor start` and the other turn-ending
    handoffs inside finalizer-owned turns, stop tool-run escalation from advertising
    a monitor join there, and stop a handoff from masking a failed finalizer as a
    completed run.

    '
- id: wait-terminal-alerts
  title: No terminal wait alert for superseded session members
  depends_on: []
  size: small
  description: 'wait-terminal-alerts: limit wait_checks terminal-blocker detection
    to members that are really terminal, so a failed monitor whose follow-up turn
    launched (or any member superseded by a newer one) no longer raises a false "Wait
    dependency can never self-resolve" notification.'
proposed_by: bbugyi200.athena.0xo
create_time: 2026-10-07 07:52:28
status: wip
bead_id: sase-1h9
---

- **PROMPT:** [prompts/202610/finalizer_repair_hardening.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finalizer_repair_hardening.md)
- **BEAD:** [sase-1h9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h9/README.md)

# Plan: Commit finalizer repair hardening

## Background: what failed overnight (2026-10-06 → 2026-10-07)

Three root phase agents failed in the host commit finalizer after their own work was
verified. Their downstream phase agents (`sase-1h7.5`–`.10`, `sase-1h8.9`–`.14`, and
both `.land` agents) are now terminally blocked in WAITING. All failures were
host-tooling defects, not agent mistakes. A parallel epic makes many phases touch
sase-core at once, so `sase-core-revision.txt` and sase-core rebase conflicts are
routine, and every one of these paths runs through conflict repair.

| Agent (run timestamp)                                      | Finalizer error                                                                                                | Defect                            |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `sase-1h8.7--2` (20261006234135)                           | `repository obligation repo-782b3c26a5d4 changed after submit. Already landed: sase-core bff4860c…`            | A (pin-after-handoff)             |
| `sase-1h8.8--3` (20261007004543)                           | `repository obligation repo-8602ab5e4d1a changed after submit. Already landed: sase-core 91e0e49c…`            | A (pin-after-handoff)             |
| `sase-1h7.3--2` (20261006234145)                           | `sase stitch create --resume failed for main: ❌ No commit checkpoint found — nothing to resume`               | B (repair-resume), triggered by C |
| `sase-1h7.3--1` (20261006222036)                           | commit finalizer `controller_exception`: repair-turn provider exited 143, yet the run was recorded `completed` | C (owned-turn-handoffs)           |
| `toobig-75.test_scoreboard_coverage.0--1` (20261007015140) | Same "No commit checkpoint found" error, although the work was already upstream (a false failure)              | B                                 |
| `sase-17d.10.1.1--2` (20260924124053)                      | Same pre-existing-rebase pattern as `sase-1h7.3--2`                                                            | B                                 |

Each failed run's artifact directory keeps its evidence: `done.json`, `error_report.md`,
`finalizer_result.json`, `final_submission.json`, `finalizers/progress.jsonl`, and
`finalizers/commit/*` stitch stdout and stderr plus the conflict-repair prompts and
responses. Use them to build faithful regression fixtures.

### Defect A: the host pin write invalidates the repair declaration

`dispatch_commit_decisions` (`src/sase/finalizers/commit_dispatch.py`) commits the
revision-pinned sibling (sase-core) before main. The sibling stitch hit a rebase
conflict (`crates/sase_core_py/src/beads/mod.rs`). The conflict-repair turn resolved it,
ran `git rebase --continue` and `sase stitch create --resume` (which landed sase-core),
then submitted a fresh declaration for the remaining main repo. That declaration's main
obligation did not include `sase-core-revision.txt`.

The dispatcher then ran the host revision_pin follow (`_maybe_write_revision_pin`, from
`src/sase/finalizers/commit_revision_pin.py`), which wrote the landed sibling SHA into
main's `sase-core-revision.txt`. After that it called `_apply_repair_remaining_handoff`
→ `collect_repair_remaining_handoff`
(`src/sase/finalizers/commit_dispatch_repair_handoff.py`). That function recomputes
`repository_state_digest` for main, which now includes the host-written pin, and
compares it with the repair declaration's submitted digest. The Rust
`select_remaining_commit_obligations` raised `RemainingCommitWorkError`, so sase-core
landed and the main repo never did.

This happens every time a pinned sibling's stitch conflicts while main also commits. No
test covers pin follow combined with conflict repair; the `test_commit_revision_pin*`
and `test_finalizers_protocol_harness_multi_repo_repair_*` suites each test only one
side.

### Defect B: resume ignores an unpushed HEAD and the run-owned checkpoint

The checkpoint lives at `$SASE_ARTIFACTS_DIR/commit_state.json`, so it is owned by a
single run (`src/sase/workflows/commit/checkpoint.py`).

- `sase-1h7.3--2`: its finalizer stitch found a paused rebase left by the previous run.
  `CommitWorkflow` (`src/sase/workflows/commit/workflow.py`, the "cannot start: …
  in-progress rebase/merge" branch) saved a new checkpoint with
  `no_commit_dispatched=True` and returned CONFLICT. The repair turn ran
  `git rebase --continue`, which created the real commit locally, ahead of
  `origin/master` and unpushed. It then ran `sase stitch create --resume`.
  `resume_commit_workflow` (`src/sase/workflows/commit/workflow_resume.py`) took the
  `no_commit_dispatched` + clean-tree branch into `_finish_no_commit_resume`. That
  printed "Checkpointed stitch has nothing to finish", deleted the checkpoint, and
  exited 0, leaving the commit unpushed.
- In both that run and `toobig-75…--1`, the repair turn's own `--resume` consumed the
  checkpoint. No commit marker was recorded, so `resolve_commit_conflict`
  (`src/sase/finalizers/commit_repair_conflict.py`) ran the host `--resume` again, found
  no checkpoint, and failed with `stale_conflict_checkpoint`. In `toobig-75` the
  repaired pick was redundant and already upstream, so this failure was false.
- A latent silent-loss path also exists: `_repo_is_settled_after_repair` checks only "no
  sync in progress, no conflicted files, no changed files". If the host's resume had
  found the checkpoint and taken the "nothing to finish" branch, the finalizer would
  have reported `resolved_without_commit` success while a rebased commit sat unpushed.

### Defect C: a repair turn handed off to a monitor and killed its own finalizer

In `sase-1h7.3--1` (and `sase-1h7.4--1`), the conflict-repair turn ran
`sase tool run check`. The tool-run escalation (`src/sase/tool/routing.py`,
`is_joinable` plus the "Hand the same run to a monitor and end this turn" text) offered
a monitor join. The model used it (`sase-1h7.3--mon-0` started three seconds before the
failure). The handoff sent SIGTERM to the repair provider (exit 143), the finalizer
recorded `controller_exception`, and the paused rebase stayed in the workspace. The run
was still finished as `completed` through the shell-handoff path, and the monitor
continuation told the next turn to "continue the paused operation with
`sase stitch create --resume`". That checkpoint belonged to the previous run's artifact
directory, which set up Defect B.

Gate turns already refuse this (`_finalizer_owned_turn_is_active` and
`_FINALIZER_OWNED_TURN_ERROR` in `src/sase/gate_turn/transaction.py`). Monitor starts
and the other turn-ending handoffs do not.

### Defect D: false "can never self-resolve" wait alerts

At 19:23 `wait_checks` reported waiters blocked on `sase-1h8.2` because member
`sase-1h8.2--mon` "failed". That was a `just check` monitor whose follow-up
`sase-1h8.2--1` was already running, and the session later completed.
`terminal_blocking_artifacts_for_name`
(`src/sase/core/wait_dependency_resolution/_index_queries.py`) returns every done member
of the newest entity with a non-success outcome, even when a newer member exists or the
member is a monitor with `monitor_followup_outcome: "launched"`. The later alerts for
`sase-1h7.3`, `sase-1h8.7`, and `sase-1h8.8` were real and must keep firing.

## Phase pin-after-handoff: Revision-pin follow after the repair-handoff check

1. In `dispatch_commit_decisions`, move the revision-pin follow block out of its current
   position (before the sibling residue check). Put it in a small local helper that runs
   after `_record_executed_repo` and, when `repaired_conflict` is true, after
   `_apply_repair_remaining_handoff` has adopted or validated the repair declaration.
   Compute `main_is_commit` from the post-handoff `pending`, `active_decisions`, and
   `active_deferrals`, so a repair declaration that newly defers main skips the pin with
   the existing `main-deferred-or-not-dirty` reason. After a successful pin write,
   refresh `state` with `prepare_dirty_state(...)` so later bookkeeping sees the pin.
2. Keep the non-repair path's observable behavior identical: same evidence
   (`revision_pin` values), same `revision_pin_skipped` diagnostics, and the same final
   main commit containing the pin.
3. Keep the stale-declaration guard: if the repair turn itself changes main after
   submitting, `collect_repair_remaining_handoff` must still fail with
   `RemainingCommitWorkError`. Only the host's own pin write moves out of the comparison
   window.
4. Tests (extend `tests/test_commit_revision_pin_dispatch.py` and/or
   `tests/test_finalizers_protocol_harness_multi_repo_repair_handoff.py`, reusing
   `tests/_commit_dispatch_conflict_repair_helpers.py`):
   - A pinned sibling stitch returns `EXIT_CODE_CONFLICT`. The repair turn lands the
     sibling (marker recorded) and submits a fresh declaration for main without the pin
     path. The finalizer must succeed, write the pin to the landed sibling SHA, and
     commit main once with the pin included.
   - The same setup, but the repair turn also edits a main file after submitting: it
     still fails with "changed after submit".
   - The same setup, but the repair declaration defers main: the pin is skipped with
     `main-deferred-or-not-dirty`.

## Phase repair-resume: Conflict-repair resume never strands or falsely fails a commit

1. `resume_commit_workflow` (`src/sase/workflows/commit/workflow_resume.py`): before
   `_finish_no_commit_resume` reports "nothing to finish" (both the
   `no_commit_dispatched` branch and the clean subject-mismatch branch), fetch and check
   whether HEAD is ahead of its upstream or the remote default branch. Reuse an existing
   VCS-provider or git helper if one fits; otherwise add a small helper next to the
   resume code.
   - Not ahead: keep today's "nothing to finish" success. That is the `toobig-75` case,
     where the work is already upstream.
   - Ahead, and every unpushed commit carries this run's SASE provenance (the
     `run_owned_commit_tags` from the checkpoint payload match HEAD, or match after the
     existing `_restamp_missing_footer_tags`): adopt HEAD as the dispatched commit and
     continue through the normal finalize path. That path is `provider.finalize_commit`
     (which syncs and pushes), recording `commit_sha`, `commit_tree`, and the
     commit-result marker, then running file hooks, the after-hook, and the tracking
     steps. The finalizer then sees a commit marker for the repo. On a push failure,
     keep the existing `_record_unpushed_commit_marker_if_present` behavior.
   - Ahead, but provenance does not match: fail closed, keep the checkpoint, and print
     the unpushed SHAs and subjects with the recovery command.
2. `resolve_commit_conflict` (`src/sase/finalizers/commit_repair_conflict.py`): after
   the repair turn, if no new marker exists and the run-owned checkpoint
   (`<context.artifacts_dir>/commit_state.json`, via `checkpoint_load`) is gone because
   the repair turn already ran `--resume`, do not call the host resume runner. Go
   straight to repository-state verification instead of failing
   `stale_conflict_checkpoint`.
3. Extend `_repo_is_settled_after_repair` so a repo counts as settled only when it is
   also not ahead of upstream. A clean repo whose HEAD is ahead must fail with a new,
   specific diagnostic (for example `unpushed_after_repair`) whose message names the
   unpushed SHAs and the exact `sase stitch create --resume` recovery. It must never
   report `resolved_without_commit`.
4. Tests (`tests/test_commit_workflow_resume.py`,
   `tests/test_commit_workflow_resume_recovery.py`, and
   `tests/test_commit_dispatch_conflict_repair_resume.py`; use real temporary git repos
   with a bare remote where the existing helpers do):
   - The `sase-1h7.3` case: a pre-existing paused rebase, a `no_commit_dispatched`
     checkpoint, then `git rebase --continue` creates an unpushed commit. `--resume`
     pushes it and records a marker.
   - The `toobig-75` case: the repair turn ran `--resume`, the checkpoint is gone, and
     the work is upstream. The finalizer succeeds with `resolved_without_commit`.
   - The checkpoint is gone and HEAD is ahead: the finalizer fails with
     `unpushed_after_repair`, never success.
   - A provenance mismatch fails closed and keeps the checkpoint.

## Phase owned-turn-handoffs: Finalizer-owned turns refuse turn-ending handoffs

1. In `src/sase/finalizers/owned_turn.py`, add a public
   `finalizer_owned_turn_is_active()` with the same truthy parsing the gate-turn copy
   uses, plus one shared refusal-message builder. Replace the private copy in
   `src/sase/gate_turn/transaction.py` with it. The gate error text may stay
   specialized.
2. `sase monitor start` (the CLI start flow under `src/sase/monitor/start*.py` and its
   `src/sase/main` handler): refuse in a finalizer-owned turn before any monitor record,
   proc, claim transfer, handoff marker, or signal is created. Exit nonzero with a
   message telling the model that this host finalizer turn cannot end the run. It should
   re-wait inline with the printed `sase tool wait …` form or run the command in the
   foreground, then finish the repair in this turn.
3. Apply the same refusal to every other shell-reachable command that ends the runner
   with a handoff marker. Find them through the handoff-marker writers, for example
   `write_pending_handoff_marker` (`src/sase/agent/pending_handoff_write.py`) and
   `src/sase/main/pipe_handler.py`. Cover at least pipe or handoff, plan propose, and
   questions.
4. `src/sase/tool/routing.py`: make `is_joinable` return False while a finalizer-owned
   turn is active. The escalation text and `escalation_json` then offer only the bounded
   inline re-wait, never a monitor join, inside repair and declaration-recovery turns.
5. Defense in depth: when a finalizer-owned provider turn ends because a handoff
   happened anyway (provider exit 143 with a handoff marker present), the finalizer
   should record a named diagnostic (for example `finalizer_turn_handoff`) instead of a
   generic `controller_exception`. The run's done outcome must not be `completed` while
   `finalizer_result.json` reports `failed`; see the exec-outcome handling in
   `src/sase/axe/run_agent_runner_launch.py` and `run_agent_runner_lifecycle.py`. If
   suppressing an already-spawned monitor continuation needs more than a local change,
   record a `PROPOSED FOLLOW-UP:` note on this phase's bead instead of building it here.
6. Tests: monitor start, the other guarded handoffs, and gate creation refuse with
   `SASE_FINALIZER_OWNED_TURN=1` and are unchanged without it. `is_joinable` and the
   escalation text and JSON stop offering a join under the marker. A repair-turn handoff
   yields the named diagnostic and a non-`completed` outcome. If any CLI help or option
   text changes, follow the CLI rules memory note.

## Phase wait-terminal-alerts: No terminal wait alert for superseded session members

1. In `terminal_blocking_artifacts_for_name` and `terminal_blocking_artifacts_for_hood`
   (`src/sase/core/wait_dependency_resolution/_index_queries.py`), a failed member
   blocks terminally only if it is the entity's newest member, no running, queued, or
   waiting member exists after it, and it is not a monitor whose done marker records
   `monitor_followup_outcome: "launched"`. Expose the follow-up outcome on
   `ArtifactCandidate` through `_artifact_state.py` if it is not already there. Keep
   `_identity_terminal_blocker` (explicit artifact-identity waits) unchanged.
2. Before editing, check whether sase-core already owns an equivalent terminal-blocker
   query. If it does, fix it there and keep the Python side a thin adapter, per the Rust
   core boundary rule.
3. Tests in the existing wait-checks terminal-blocker suite:
   - A failed `--mon` member with a launched follow-up that is still running raises no
     alert.
   - The session's newest member failed (the `sase-1h7.3--2` shape): the alert still
     fires.
   - A hood with a superseded failed member raises no alert.

## Verification

Every phase runs its targeted tests plus the repo gate from the lint-and-test memory
note through `sase tool run`. The known master-red symvision items (`_runs`
private-module imports, tracked by `sase-1h6`) are pre-existing and are not a reason to
widen scope.

## Out of scope

- Recovering the three stranded phase runs and unblocking their waiters is an operator
  task outside this epic. Those runs are `sase-1h7.3` (unpushed rebased commit in its
  held workspace), `sase-1h8.7` (bead already closed although its Python side never
  landed), and `sase-1h8.8` (Python side uncommitted, pin written).
- Fixing the master-red symvision lint (`sase-1h6`) and the red sase-core CI (rustfmt
  drift and stale `for_epic` editor-directive contract expectations).
