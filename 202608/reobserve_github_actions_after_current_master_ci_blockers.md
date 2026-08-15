---
tier: tale
size: medium
title: Reobserve GitHub Actions after current-master CI blockers
goal:
  Reconcile the current default-branch CI failures that appeared after the GitHub
  Actions stabilization commit, then close epic sase-m4 only after the latest CI, Deploy
  Docs, and Publish workflows for a master tip containing the stabilization commit are
  terminal and successful.
proposed_by: bbugyi200.athena.sase-m4.land--a
bead: sase-m4
create_time: 2026-08-15 05:12:53
status: wip
---

- **PROMPT:**
  [prompts/202608/reobserve_github_actions_after_current_master_ci_blockers.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/reobserve_github_actions_after_current_master_ci_blockers.md)
- **BEAD:**
  [sase-m4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m4/README.md)

# Context

The approved tale `plan:202608/finish_github_actions_stabilization.md` landed commit
`5601920c9dc66259eb858dc7c851e6d4801014a8` (`test: stabilize GitHub Actions checks`).
Local `just check-full` passed for that commit. That commit is an ancestor of current
`origin/master`.

The exact CI run watched for current master commit
`d19d08641246a2b0f9276fded07d93004815d640` was `31861402259`, and it completed failed.
The failures were not the two stabilization defects from the previous tale:

- `perf-floors`, lint, Python 3.12, and Python 3.14 passed.
- `test (3.13)` failed only in the post-pytest test-cost budget gate after pytest
  completed. This was corroborated on task bead `sase-j0`.
- `coverage-contexts` failed on
  `tests/ace/tui/test_config_center_state.py::test_save_atomically_replaces_existing_state`.
  This was recorded on active epic `sase-j7`, where that full-lane flake was already
  routed.
- `visual-test` failed after Artifacts shell/contract commits. This was recorded on
  active epic `sase-m6`, which owns the Artifacts contract work.

Do not weaken tests, skip tests, relax budgets ad hoc, or absorb visual drift without
identifying whether the new screenshots are intentional. Do not make changes in
`sase/memory/`, `AGENTS.md`, or generated provider instruction shims.

# Plan

1. Refresh repository and workflow state.
   - Run `git fetch origin master`.
   - Confirm `5601920c9dc66259eb858dc7c851e6d4801014a8` is still an ancestor of
     `origin/master`.
   - Inspect the latest master runs for CI, Deploy Docs, and Publish. Use exact run IDs
     and exact head SHAs. Do not watch an older queued or superseded run.

2. If a newer master CI run is still in progress, watch that exact run through
   `/sase_monitor`, with a continuation that resumes this plan. The Python 3.13 leg can
   take about an hour, so use a timeout sized for the full run.

3. If the latest master CI run is green and Deploy Docs and Publish are terminal
   successful for the same master tip, finish the original closeout:
   - Run `actstat` and confirm the `sase` project reports a stable passing last GitHub
     Actions run.
   - Close epic `sase-m4` with `sase bead close sase-m4 --note "<...>"`. The note must
     include the original plan's required evidence: verified phases, the clipboard-race
     and agent-scan floor repairs, integration with later commits, and the outcome of
     every proposed follow-up including the corroborations recorded on `sase-j0`,
     `sase-j7`, and `sase-m6`.
   - Run `just symvision` and remove only stale `sase-m4` findings if any are reported.
   - Ensure `sase/repos/plans/202608/stabilize_github_actions.md` has `status: done`.
   - If any file changed during closeout, run `just check` before committing and commit
     only intended changes through `/sase_git_commit`.

4. If CI is still red only for blockers already recorded on `sase-j0`, `sase-j7`, or
   `sase-m6`, do not duplicate the work. Add fresh evidence to the owning bead only if
   the new run adds materially new information, then stop without closing `sase-m4`. The
   stabilization work is locally verified, but the original closeout requires a green
   default-branch Actions run, so the epic must remain open until those active owners
   clear the default branch.

5. If CI fails for a new or attributable regression:
   - Use `/sase_new_task` before recording unrelated discovered work.
   - If the failure is attributable to the stabilization diff, fix it directly, run
     focused checks, run `just check`, then run `just check-full` only through
     `/sase_monitor`.
   - Commit through `/sase_git_commit`, push, obtain the exact new CI run ID, and repeat
     the observe loop above.
