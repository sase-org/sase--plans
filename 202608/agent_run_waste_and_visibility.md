---
tier: epic
title: Stop long agent runs from being wasted, invisible, or falsely failed
goal: "A run whose own work is committed finishes as a success that releases its
  workspace, even when the workspace it inherited was already dirty; the commit
  finalizer stops as soon as a pass proves it cannot make progress; `sase agent list`
  names the family member that is actually executing; and `just test-cost` has the same
  cross-agent admission control as the other heavy lanes.

  "
phases:
  - id: claim-baseline
    title: Record claim-time dirt and park the claim stash out of reach
    depends_on: []
    size: medium
    description: "claim-baseline: persist the workspace's pre-claim dirty/untracked path
      set as a baseline later stages can read, and move the claim-time stash off the
      stash stack so an agent's ad-hoc git commands cannot resurrect foreign work
      mid-run.

      "
  - id: finalizer-scope
    title: Exclude pre-existing foreign dirt from the finalizer's clean check
    depends_on:
      - claim-baseline
    size: medium
    description: "finalizer-scope: teach dirty-state collection to skip paths that were
      already dirty at claim time and that the agent never touched, while still
      reporting them as pre-existing foreign dirt rather than silently ignoring them.

      "
  - id: finalizer-stall
    title: Exit the finalizer pass loop once a pass makes no progress
    depends_on: []
    size: small
    description: "finalizer-stall: use the stall signals the pass loop already computes
      to break out after a zero-progress pass instead of always running to max_passes,
      and name the stall in the failure message.

      "
  - id: run-outcome
    title: Do not report FAILED for a run whose commit landed
    depends_on:
      - finalizer-scope
    size: medium
    description: "run-outcome: when the only remaining dirt is out of scope and the run
      produced a commit, finish on the completed path with a warning that names the
      commit and releases the workspace, while genuinely uncommitted own work still
      fails.

      "
  - id: agent-list-role
    title: Show the executing family member in sase agent list
    depends_on: []
    size: medium
    description: "agent-list-role: stop the plan-chain root from being the only visible
      row for a family's whole life, so the active row's name, model, prompt snippet,
      artifacts dir, and duration describe the member that is really running.

      "
  - id: test-cost-lane
    title: Give just test-cost cross-agent admission control
    depends_on: []
    size: medium
    description:
      "test-cost-lane: put the cost lane behind SASE's own admission path with a
      bounded, reported wait so agents stop hand-rolling pgrep-based locks against it,
      and re-examine the 600s cap that lane reliably hits."
proposed_by: bbugyi200.athena.y1
create_time: 2026-09-09 19:53:29
status: wip
---

- **PROMPT:**
  [prompts/202608/agent_run_waste_and_visibility.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_run_waste_and_visibility.md)

# Plan: Stop long agent runs from being wasted, invisible, or falsely failed

## Motivating incident

`sase-jd.8` ran for **3h26m** and then reported `FAILED`, even though its work had
already landed correctly. Reconstructed from
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/11/20260811060459` (the
`--plan` member) and `.../20260811082905` (the `--code` member):

| Window (EDT) | Elapsed | What was happening                                                                                         |
| ------------ | ------- | ---------------------------------------------------------------------------------------------------------- |
| 08:13–08:29  | 16m     | `sase-jd.8--plan` authored and proposed the plan                                                           |
| 08:29–09:27  | 58m     | `sase-jd.8--code` session 1 implemented the plan, terminated before committing                             |
| 09:27–11:37  | 2h10m   | commit finalizer pass 1 — re-verified from scratch, fixed ~24 test files, 2 rebases, committed `b5786b57f` |
| 11:37–11:39  | 2m      | commit finalizer pass 2                                                                                    |
| 11:39        | —       | run `FAILED`; workspace #12 held pending manual ACE dismissal                                              |

The failure message was:

```
Commit finalizer failed: uncommitted changes remain after 2 finalizer pass(es)
in main=<workspace>: src/sase/core/pr_mirror_facade.py,
src/sase/doctor/checks_deep_vcs_pull_requests.py,
src/sase/external_mirror/pull_requests.py,
tests/main/test_patch_sync_external_parser.py, tests/test_external_pr_mirror.py
```

Those five files were **never touched by `sase-jd.8`**. They are `external_pr_mirror`
WIP belonging to a different agent that previously held the same workspace. SASE stashed
them at claim time (`git stash push --include-untracked`, stash message
`gh_sase-org__sase-ace`, created 06:05:13), and `git stash show -u stash@{0}` still
lists exactly those five paths. During rebase-conflict recovery the `--code` agent ran
an ad-hoc `git stash` / `git checkout <sha> -- .` / `git reset --hard HEAD` sequence,
and at 11:20:48 all five were re-materialized into the working tree with identical
mtimes. The agent recognized them as foreign and correctly refused to commit them; its
own summary says so explicitly.

So the run did everything right and was still marked `FAILED`: bead `sase-jd.8` closed
at 15:08:17Z with a verification note, commit `b5786b57f` is on `master`, and every
phase of epic `sase-jd` is closed.

This is not a one-off. `grep -rl "Commit finalizer failed" ~/.sase/workflows/` matches
**60 runs** (45 in 202607, 15 in 202608), including run `260811_060456` from this same
epic batch.

Three independent things went wrong, and this epic addresses all three:

1. **The run was thrown away.** Roughly two hours of large-model work produced a correct
   commit that the finalizer then declared a failure, holding a workspace and burning a
   second pass on a goal it could not reach.
2. **Nobody could see what it was doing.** `sase agent list` showed one row,
   `sase-jd.8--plan`, `RUNNING` for the family's whole life, with the planner's model
   (`opus`, not the actual `sonnet`), the planner's prompt snippet, and a
   `live_reply.md` frozen at "Submitting the plan for approval." since 08:28.
3. **Roughly an hour went to verification churn.** `just test-scoped` ran six times
   (~33m total), `just test-cost` hit its 600s cap, and the agent hand-rolled a 590s
   `until ! kill -0 $(pgrep -f "tools/run_pytest cost")` busy-wait because that lane has
   no cross-agent admission control.

## claim-baseline

Record what was already dirty when the workspace was claimed, and make the claim-time
stash unreachable from the working tree.

`vcs_stash_and_clean` (`src/sase/vcs_provider/plugins/_git_core_ops.py:209`) runs
`git stash push --include-untracked` and leaves the result on the normal stash stack,
where any later `git stash pop`, `git stash apply`, or index-shuffling recovery command
can resurrect it. Its callers are
`src/sase/ace/scheduler/workflows_runner/starter.py:167`,
`src/sase/workflows/commit_utils/workspace.py:129`, and
`src/sase/ace/revert_agent_workspace.py:316`.

Do two things:

1. Capture the pre-stash `git status --porcelain` path set (tracked-dirty and untracked,
   for the main workspace and every sibling or linked repo that gets cleaned) and
   persist it as a claim-time dirty baseline the rest of the run can read.
   `agent_meta.json` is the natural home, alongside the existing `sdd_base_sha`.
2. Move the stash commit off `refs/stash` after creating it. Keep the OID under a
   dedicated ref, or record it and drop the stash entry, so it stays recoverable for
   SASE's own recovery paths but cannot be popped back into the worktree by an agent's
   ad-hoc git commands. `src/sase/sdd/_repository_recovery_reaper.py` and
   `src/sase/sdd/_repository_recovery_snapshot.py` already enumerate and drop stashes;
   keep them working against whatever ref layout you choose.

Acceptance: after a claim that stashed foreign work, `git stash list` in the claimed
workspace is empty, the stashed commit is still reachable by OID, and the baseline path
set is readable from the agent's metadata.

## finalizer-scope

Teach the commit finalizer to ignore dirt it did not create.

`collect_dirty_state` (`src/sase/llm_provider/commit_finalizer_state.py:38`) treats
every dirty path equally. Extend the existing exclusion seam —
`filter_sase_reserved_paths` and `_is_sase_reserved_path` in
`src/sase/llm_provider/commit_finalizer_git.py:451` — so a path is excluded when it was
in the claim-time baseline from `claim-baseline` **and** the agent never touched it
during the run.

Excluded paths must still be reported. Surface them in the finalizer result
(`CommitFinalizerResult`, `src/sase/llm_provider/commit_finalizer_types.py`) and in the
operator-facing message as pre-existing foreign dirt, so silently ignoring a path an
agent genuinely should have committed stays impossible to confuse with success.

Acceptance: a run in a workspace whose claim baseline contains untracked foreign files
finalizes cleanly once its own changes are committed, and the result records the skipped
foreign paths.

## finalizer-stall

Stop the finalizer from burning passes it has already proven useless.

The pass loop at `src/sase/llm_provider/commit_finalizer.py:242` computes
`fingerprint_before` and `fingerprint_after`, sets `previous_pass_stalled`, and
increments `no_progress_passes` — but never uses any of it for loop control. With
`_DEFAULT_MAX_PASSES = 2`, a pass that changes nothing is always followed by a second
full pass against an identical dirty set. In the motivating incident pass 1 ran 2h10m
and pass 2 could not have succeeded.

Break out of the loop as soon as a pass makes zero progress, and put the stall in the
error message ("pass N made no progress; remaining: ...") instead of the current
undifferentiated "after 2 finalizer pass(es)". Keep `max_passes` as the ceiling.

Acceptance: a finalizer whose first pass changes nothing exits after that pass, and
`commit_result.json` records `no_progress_passes` plus a stall reason.

## run-outcome

Do not report `FAILED` for a run whose work landed.

`sase-jd.8` committed `b5786b57f` and closed its bead, then the runner raised
`WorkflowExecutionError` and `src/sase/axe/run_agent_runner_lifecycle.py:195` held
workspace #12 with "dismiss the agent in ace to release it". For an epic phase that
means a completed phase reads as failed and its workspace stays occupied until a human
intervenes.

When the finalizer's only remaining dirt is out of scope (per `finalizer-scope`) and the
run produced a commit, report a non-failing outcome that still carries the warning, name
the landed commit SHA, and release the workspace on the normal completed path. A run
with genuinely uncommitted _own_ work must still fail exactly as it does today.

Acceptance: the motivating incident's shape — own work committed, foreign baseline files
remaining — completes with a warning, releases its workspace, and names the commit; an
agent that leaves its own edits uncommitted still fails and still holds its workspace.

## agent-list-role

Show the family member that is actually executing.

`_running_from_snapshot` (`src/sase/agent/running_listing.py`) keeps a record only when
`is_root_user_agent_record(record)` or `_is_visible_runner_slot_child(record)` holds. A
plan chain's `--code` member has `parent_timestamp` set and shares the root's runner
slot, so it fails both tests and is dropped. The `--plan` root's PID is the runner
process, which stays alive for the whole family, so that row remains `RUNNING` and keeps
accruing duration.

The observable damage during the incident: `sase agent list` reported
`sase-jd.8--plan RUNNING 3h23m`, model `opus`, with the planner's prompt snippet and a
`live_reply.md` last written at 08:28 — while the real work was `sase-jd.8--code` on
`sonnet`, in commit-finalizer pass 1, narrating into a different artifacts dir. ACE's
`concrete_family_member_rows` (`src/sase/ace/tui/models/agent_family_members.py`)
already models this correctly; the CLI listing does not.

Make the active row reflect the executing member — name, model, provider, prompt
snippet, artifacts dir, and a duration attributable to that member — rather than the
plan-chain root. Decide deliberately whether the family keeps one row that advances or
gains a second row, and keep `-j` field shapes stable per the `sase_agents_status` skill
contract.

Acceptance: while a plan chain is in its `--code` phase, `sase agent list -j` names that
member and points `artifacts_dir` at the dir receiving live output.

## test-cost-lane

Give `just test-cost` the cross-agent admission control the other heavy lanes have.

`test-scoped` (`Justfile:404`) documents that it "never queues behind another agent's
run", and `check` (`Justfile:572`) is built on that guarantee. `test-cost`
(`Justfile:372`) has no such treatment, so concurrent agents collide on it. During the
incident the agent hit a 600s `just test-cost` — its cap — and then invented its own
lock:

```
until ! kill -0 $(pgrep -f "tools/run_pytest cost" | head -1) 2>/dev/null; do sleep 5; done
```

That busy-wait alone burned 590s. Agents should not be hand-rolling process-level locks
against SASE's own test tooling.

Give the cost lane real admission control — a suite-gate lease, a documented wait, or an
explicit "another agent is running this lane" refusal — so the wait is bounded, visible,
and does not depend on the agent guessing a `pgrep` pattern. Re-examine the 600s cap
while you are there: a lane that reliably hits its own timeout is not usable as a gate.

Acceptance: two concurrent `just test-cost` invocations resolve through SASE's own
admission path with a bounded, reported wait, and no `pgrep`-based workaround is needed.
