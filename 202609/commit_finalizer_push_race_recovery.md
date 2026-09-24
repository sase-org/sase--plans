---
tier: tale
size: medium
title: Make the commit finalizer survive a bead-store push race
goal:
  A stitch whose primary commit landed but whose bead close lost a push race must resume
  and finish on the finalizer's second attempt instead of failing the agent.
proposed_by: bbugyi200.athena.0r4
create_time: 2026-09-24 14:12:19
status: wip
---

# Make the commit finalizer survive a bead-store push race

## Context: how `sase-17p.5` failed

The `sase-17p.5` tale agent implemented its phase and submitted an accepted final
declaration (`action: commit`, `bead_action: close`). The builtin `commit` finalizer
(`max_attempts: 2`) then failed the whole agent with:

```
commit checkpoint does not match the accepted work; automatic recovery refused
```

Two independent defects chained together:

1. **Attempt 1: a benign push race was treated as fatal.**
   `sase stitch create -M ... -B close` created and pushed the primary commit
   (`71736697d`, now on `origin/master`), then ran `sase bead close`. That committed
   `chore(beads): close sase-17p.5` in the beads sidecar, but the managed bead sync push
   failed on its **first** attempt. GitHub had just accepted another agent's
   `chore(beads): update artifact links` push three seconds earlier, and it rejected
   ours with:

   ```
   ! [remote rejected] main -> main (cannot lock ref 'refs/heads/main': is at bd35e42... but expected b4b58c9...)
   error: failed to push some refs to 'github.com:sase-org/sase--beads.git'
   ```

   This is the server-side form of a lost push race: the remote ref moved between ref
   advertisement and the ref update. `_is_non_fast_forward_rejection` in
   `src/sase/bead/sync_worker.py` only recognizes `non-fast-forward`, `fetch first`, or
   `[rejected]` combined with `failed to push some refs`. `[remote rejected]` does not
   contain the substring `[rejected]`, so the bounded integrate-and-retry loop
   (`_MAX_PUSH_ATTEMPTS = 3`) returned `git push failed` instead of integrating and
   retrying. The bead close was left committed locally but unpublished, and stitch
   exited 1. The sync log shows `event: failed` after a single push.

2. **Attempt 2: checkpoint recovery compared messages that only differ in host-stamped
   footer tags.** The finalizer correctly found the run-owned pending checkpoint
   (`commit_state.json`: same `run_id`, same `publication_agent`, an `operation_id`, a
   `commit_sha`, and `append_commits_entry`/`close_bead` still pending) and asked the
   Rust core whether to resume it. The payload identity it sent is built by
   `_normalized_commit_message` in `src/sase/finalizers/commit_checkpoint_recovery.py`.
   That function strips only `RUN_OWNED_COMMIT_TAG_KEYS = {AGENT, TYPE, BEAD}`. However,
   `sase stitch create` also stamps `SASE_PLAN=[<plan>][2]` (plus its `[2]: https://...`
   link reference) through `src/sase/workflows/commit/plan_hooks.py` whenever the agent
   runs from a plan. The checkpoint side kept the `SASE_PLAN` footer, and the accepted
   declaration message (which agents author without footer tags) did not.
   `payload_matches` was false, so core returned `fail` / `checkpoint_payload_mismatch`.
   The resume that would have re-run the bead close and published it never happened.

   Reproduced against the real artifacts: normalizing the checkpoint message leaves
   `SASE_PLAN=[202609/tool_handoff_settlement.md][2]` and its link line, so
   `normalized(checkpoint) != normalized(accepted)`.

Either fix alone would have saved this run. Fix 1 prevents the transient failure. Fix 2
makes the finalizer's built-in second attempt able to recover from any post-commit
failure in a plan-backed run, which today is every epic phase and every approved tale.
Both are needed.

Current state left behind (recovered operationally, not by this plan): the primary
commit is on `origin/master`. The bead close commit exists only in the failed run's held
workspace beads clone, so `sase-17p.5` is still `in_progress`, which blocks `sase-17p.6`
and `sase-17p.land`.

## Scope boundaries

- Shared push-race classification also exists in the Rust core
  (`sase_core::sidecar_publication::classify_push_failure`, used by launch-time sidecar
  publication). It has the same `[remote rejected] ... cannot lock ref` gap. Per the
  Rust core boundary, the long-term home for push-race classification is sase-core, but
  moving the Python bead/agents-sync classifiers there is a larger refactor that also
  needs a `sase-core-revision.txt` pin bump. This plan fixes the Python classifiers in
  place. Before finishing, use `/sase_new_task` to file one task bead (type `bug`, size
  `small`) for the sase-core classifier gap. Link it to this change, and note that it
  should eventually absorb the Python helper added here.
- Open task `sase-po` (tighten `--resume`'s identity check beyond subject-only matching)
  is related but separate; do not close or rescope it. The fix here keeps full
  subject-plus-body identity and only ignores host-owned footer tags.
- Do not touch the sase-core `pending_commit_checkpoint` decision logic. It behaved
  correctly given the facts it received.

## Changes

### 1. Treat server-side ref-lock races as retryable push rejections

- Add one small shared helper that decides whether a failed `git push` lost a race and
  can succeed after integrating upstream. For example, add
  `is_retryable_push_race(stdout: str, stderr: str) -> bool` in a git-helper module that
  both `src/sase/bead/` and `src/sase/agents_sync/` can import without cycles.
  `src/sase/sdd/_git_contention.py` or a new `src/sase/sdd/_push_race.py` fit; pick
  whichever avoids import cycles. Match case-insensitively on the combined output:
  - `non-fast-forward`
  - `fetch first`
  - `updates were rejected because the remote contains work`
  - `[rejected]` together with `failed to push some refs` (existing behavior)
  - `[remote rejected]` together with `cannot lock ref`, or with
    `incorrect old value provided` (the two server-side race phrasings). Do **not**
    treat every `[remote rejected]` as retryable. Hook declines, protected-branch
    refusals, and permission errors also use `[remote rejected]` and must stay fatal.
- Route `_is_non_fast_forward_rejection` in `src/sase/bead/sync_worker.py` through the
  helper, keeping the function name if tests or callers depend on it.
- Route `is_agents_non_fast_forward` in `src/sase/agents_sync/git_sync_ops.py` through
  the same helper. Preserve its current, slightly broader `[rejected]` acceptance, so
  that call site's existing behavior does not narrow.
- Update `src/sase/bead/_sync_logs.py`'s failure-category text matcher (the branch that
  returns `"push rejection"`) so a `cannot lock ref` failure is still categorized as a
  push rejection. It likely already is, via `failed to push some refs`; add a test if it
  is.

### 2. Ignore host-stamped footer tags when matching a pending checkpoint

- In `src/sase/finalizers/commit_checkpoint_recovery.py`, change
  `_normalized_commit_message` so the payload identity is the agent-authored subject and
  body only. Strip **every** trailing SASE footer tag, and the link reference lines that
  belong to those tags, rather than just `RUN_OWNED_COMMIT_TAG_KEYS`. Use the existing
  footer facade: take the keys from `parse_trailing_commit_tag_values(text)` and pass
  them as `remove_keys` to `update_trailing_commit_tags(text, {}, remove_keys=...)`.
  Apply the same normalization to both the checkpoint message and the accepted decision
  message. The footer is host-owned (TYPE, AGENT, BEAD, PLAN, and any future stamp), so
  enumerating keys would break again the next time stitch stamps a new tag.
- Keep line-ending and trailing-whitespace normalization as-is.
- Confirm by reading, and cover with a test, that link-reference lines (`[n]: url`)
  belonging to removed tags are removed. Otherwise the identity would still differ.
- Leave `RUN_OWNED_COMMIT_TAG_KEYS` and the `--resume` re-stamping in
  `workflow_resume.py` unchanged; they serve a different purpose.

## Tests

- `tests/test_bead/test_sync_worker_hygiene.py`: model on
  `test_managed_sync_worker_reintegrates_after_push_race`. The first push returns the
  exact
  `[remote rejected] main -> main (cannot lock ref 'refs/heads/main': is at <a> but expected <b>)`
  output and exits 1; the worker must integrate and retry, and the second push must
  succeed with `push_attempts == 2`. Add a negative case where a
  `[remote rejected] main -> main (pre-receive hook declined)` push fails immediately,
  with no retry.
- Unit tests for the new helper that cover each accepted phrasing and the rejected
  non-race `[remote rejected]` forms.
- An agents-sync test (next to existing `is_agents_non_fast_forward` coverage, or a new
  focused test) proving the `cannot lock ref` output is now treated as a race.
- `tests/test_finalizers_commit_reconciliation.py`: model on
  `test_pending_checkpoint_resumes_before_clean_acceptance`. The checkpoint message
  carries a realistic linked footer: `SASE_BEAD=[id][1]`, `SASE_PLAN=[plan][2]`,
  `SASE_TYPE=stitch`, `SASE_AGENT=[agent][3]`, and the `[1]:`/`[2]:`/`[3]:` link lines.
  Set `completed_steps` without `append_commits_entry`/`close_bead` and use
  `bead_action: close`. The accepted message is the bare subject and body. Assert the
  finalizer calls resume (not create) and the aggregate status is `success`.
- Keep `test_pending_checkpoint_refuses_same_subject_different_body` passing: a
  different body must still refuse with `checkpoint_payload_mismatch`.
- Run the focused suites, then `just check` through `sase tool run` as the lint/test
  memory note directs. Read that note before finishing.

## Acceptance

- A bead-store push that loses a `cannot lock ref` race is integrated and retried within
  the existing three-attempt budget. Non-race `[remote rejected]` failures still fail on
  the first attempt.
- A plan-backed stitch whose pending checkpoint differs from the accepted declaration
  only in host-stamped footer tags is resumed by the finalizer's second attempt instead
  of refused.
- A follow-up task bead exists for the sase-core `sidecar_publication` classifier gap.
