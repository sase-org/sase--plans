---
tier: tale
title: Fix the self-perpetuating "Failed to archive approved plan" ResetReplayError
goal:
  Approval-time plan archiving recovers from a rejected push instead of exhausting its
  replay budget, and never leaves an unpublishable commit behind in a leased workspace's
  plans sidecar that poisons every later archive in that workspace.
size: medium
proposed_by: bbugyi200.athena.03e
create_time: 2026-08-16 09:38:47
status: wip
---

# Plan: Fix the self-perpetuating plan-archive `ResetReplayError`

## Symptom

Approving a plan raises a `plan-archive` notification:

```
Failed to archive approved plan: recover_artifacts_conformance_phase.md
ResetReplayError: reset-and-replay exhausted 3 attempt(s):
<workspace>/sase/repos/plans HEAD was not published after the archive commit
The plan of record exists only on this machine until it is archived by a later launch.
```

Two of these arrived in one session (`ctrl_space_stale_prompt_context` at ~09:20,
`recover_artifacts_conformance_phase` at 09:27), and the failures keep recurring.

## Evidence

The TUI process wrote exactly three managed-sync logs per failure — one per replay
attempt — under `~/.sase/bead_push_logs/`. All three logs from the 09:27
`recover_artifacts_conformance_phase` failure report the same thing:

```json
{"event": "conflict_resolution",
 "message": "non-bead conflicts remain: 202608/ctrl_space_stale_prompt_context.md",
 "ok": false}
{"error": "git rebase failed: Rebasing (1/4)\nerror: could not apply c728be76...
           Archive approved plan ctrl_space_stale_prompt_context ...",
 "event": "failed", "integrated": false}
```

Three facts follow directly from those logs:

1. The 09:27 archive of `recover_artifacts_conformance_phase` failed on a **different
   plan's** commit — `c728be76 Archive approved plan ctrl_space_stale_prompt_context`,
   the stale, never-pushed commit left behind by the 09:20 failure. **The second failure
   was caused by the first.**
2. The rebase queue grows across the three attempts — `Rebasing (1/4)`, `(1/5)`, `(1/6)`
   — so each replay attempt _adds_ another local commit rather than converging.
3. The conflicting path is `202608/<plan>.md`, a plans file, not a bead file.

Corroborating repository state: the published `sase--plans` history contains
`788894f1 Add SDD files for ctrl_space_stale_prompt_context` (09:20:25) and
`060e7b0d Add SDD files for recover_artifacts_conformance_phase` (09:27:30) — committed
by a _different_ code path than the one that failed, with a different commit message.

## Root cause

Two independent code paths publish the same canonical plan file into the shared
`sase--plans` sidecar with content that can never be byte-identical, and the archive
path's recovery mechanism resets a repository that does not contain the failing commit.

### 1. Two writers produce divergent content for the same path

- `src/sase/axe/run_agent_exec_plan_accept.py:314` commits `Add SDD files for <plan>`
  with `push_after_commit=True`.
- `sase._plan_archive_approval.archive_approved_plan` commits
  `Archive approved plan <plan>` with a verified synchronous push.

Both write `202608/<plan>.md` in the same remote. They can never agree, because
`archive_plan_file` (`src/sase/sdd/plan_archive.py:111`) calls
`add_create_time_frontmatter(content)`, which **overwrites** any existing `create_time`
with `datetime.now()` (`src/sase/llm_provider/_plan_utils.py:45-62`). Every call stamps
a new timestamp, so the two versions differ by at least one line — a genuine add/add
conflict on rebase.

### 2. The sync worker cannot resolve a plans-file conflict

`_resolve_bead_conflicts` (`src/sase/bead/conflict_resolver.py:84-88`) auto-merges bead
files only. A conflicted `202608/*.md` is a "non-bead conflict": the rebase is aborted
and the push fails. This is correct behavior, not a defect — it is simply the point
where recovery becomes the archive path's responsibility.

### 3. The push failure is invisible to the commit layer

`push_sdd_store_after_commit` (`src/sase/sdd/_commit_store.py:454-487`) discards the
`PushOutcome` and downgrades a rejected push to `_logger.warning`.
`commit_sdd_store_files` still returns `True`. The only detector left is the explicit
`head_is_published` probe in `src/sase/_plan_archive_approval.py:113-121`, which raises
`ReplayConflict`.

### 4. **The load-bearing defect: reset-and-replay resets the wrong repository**

`src/sase/_plan_archive_approval.py:124` calls `lease.reset_and_replay(...)` with no
`repo_root`, so `OperationalLease.reset_and_replay`
(`src/sase/workspace_provider/lease.py:146`) defaults to `self.checkout_dir` — the
**workspace** checkout.

The unpublished commit lives in `<checkout>/sase/repos/plans`, an independent nested
clone that the outer repository **gitignores** (`.gitignore:63` — `/sase/repos/`). A
`git reset --hard` on the outer checkout provably cannot touch it.

So every "recovery" is a no-op: all three attempts re-run against byte-identical
repository state and hit the identical conflict. Exhaustion is guaranteed, not probable.

### 5. Each replay makes the repository worse

Because `create_time` is re-stamped per attempt (defect 1), each attempt writes
different bytes and creates _another_ commit on the unpublishable local branch. That is
the `1/4 → 1/5 → 1/6` growth in the logs. The archive path digs its own hole deeper on
every retry.

### 6. The damage persists across leases, so failures compound

Nothing cleans up the abandoned commits. `_prepare_from_primary_remote`
(`src/sase/workspace_provider/lease.py:530`) fetches and force-checks-out the **outer**
checkout only; it never prepares sidecar repositories. Seven minutes later the
`recover_artifacts_conformance_phase` approval leased the same workspace, and its push
had to replay `c728be76` first — the same unresolvable conflict, again.

That is why the errors keep coming: **one failure permanently poisons a workspace for
every subsequent plan archive.**

### 7. Secondary: lock contention is misreported as divergence

`push_bead_work_launch` is invoked with `worker_lock_wait=0.0`
(`src/sase/bead/sync.py:61`), so any concurrent sync worker yields `skipped_locked` with
`error=None` and no push. The archive path cannot distinguish that from a rejected push
and raises `ReplayConflict` — requesting a destructive reset for what is merely
contention, and burning all three attempts back-to-back with no backoff.

## Fix

### Step 1 — Aim reset-and-replay at the repository that holds the unpublished commit

In `src/sase/_plan_archive_approval.py`, pass the store's own repository root:

```python
result = lease.reset_and_replay(
    _archive_and_publish,
    repo_root=None if sdd_store.is_in_tree else sdd_store.repo_root,
)
```

For sidecar storage `sdd_store.repo_root` is the plans repository
(`SddStore.repo_root_for_kind`, `src/sase/sdd/_store_types.py:146-153`, maps
`PLANS_SIDECAR_ROLE` to `repo_root`), which is exactly the path already named in the
failure message. For in-tree storage the checkout root _is_ the repository, so keep the
default.

This passes the existing authorization guard unchanged: `_require_leased_checkout`
(`src/sase/workspace_provider/reset_replay.py:158-164`) requires the root to be relative
to the leased checkout, and `<checkout>/sase/repos/plans` is. `_resolve_upstream`
(`reset_replay.py:240-247`) resolves `@{upstream}` in that repository — the live plans
clone has `branch.main.remote=origin` / `branch.main.merge=refs/heads/main`, so it
returns `origin/main` rather than falling back to the default branch.

With this alone the reported failure recovers: attempt 1 conflicts, the reset discards
the local commit and lands on the verified upstream tip, and attempt 2 replays onto
linear history and fast-forwards.

### Step 2 — Start every archive from a published tip

Before the first attempt, if the store repository has local commits that are not
ancestors of `@{upstream}`, fetch and hard-reset to the upstream tip. This clears poison
left by _any_ prior failure — including a crash or `SIGKILL` that never reached the
replay loop at all — and is what would have prevented the 09:27 failure outright.

Reuse the existing reset machinery rather than hand-rolling git: expose the reset step
of `reset_replay.py` (`_reset_leased_checkout`, plus its `_snapshot_pre_reset_head`
recovery ref) as a supported entry point, and keep it behind the same
`_require_leased_checkout` authorization so it can never run against primary `#0`, a
read-only canonical store, or a path outside the lease.

Discarding commits is safe here precisely because the target is a machine-owned leased
checkout's disposable clone, and the pre-reset HEAD is preserved under
`refs/sase/reset_replay/<stamp>-<sha>`.

### Step 3 — Leave no residue when the attempt budget is exhausted

Wrap the `lease.reset_and_replay` call so that a `ResetReplayError` resets the store
repository to its upstream tip before propagating. Include the recovery ref in the
re-raised message so `report_plan_archive_failure` tells the user where the abandoned
commit went instead of silently stranding it.

Without this, the final attempt's commit survives and Step 2 becomes the only thing
standing between one failure and the next.

### Step 4 — Make archived plan content idempotent

In `archive_plan_file` (`src/sase/sdd/plan_archive.py:111`), when the destination
already exists and carries a `create_time`, reuse it instead of re-stamping: read it
from the existing destination and pass it explicitly to `add_create_time_frontmatter`.
Leave the shared `add_create_time_frontmatter` default behavior alone — other callers
depend on it.

Two consequences, both required:

- Replays converge to identical bytes, so `commit_sdd_files` returns `False`
  (`src/sase/sdd/_commit_store.py:94`) and no new commit is created per attempt. The
  `1/4 → 1/5 → 1/6` growth disappears.
- After a Step 1 reset the destination returns from upstream carrying the accept path's
  `create_time`, so the archive rewrite becomes a genuine no-op, `HEAD == @{upstream}`,
  `head_is_published` is `True`, and the archive succeeds without fighting the other
  writer at all.

This is also the semantically correct reading of the field: `create_time` is when the
plan was created, not when it was last re-archived.

### Step 5 — Distinguish contention from divergence

Have `push_sdd_store_after_commit` return its `PushOutcome` (it currently returns
`None`) and thread it out through `commit_sdd_store_files`. In the archive path, map the
outcome:

- `pushed` → success.
- `skipped_locked` → `ReplayDeferred`, which retries without a destructive reset, per
  the contract in `reset_replay.py:48-52`.
- rejected / errored → `ReplayConflict`, with the push error in the message rather than
  only in a `_logger.warning` nobody reads.

Keep the existing `head_is_published` probe as the final backstop. Consider a small
non-zero `worker_lock_wait` for this specific synchronous, verified push so three
back-to-back attempts do not all lose the same lock; do not change the global default.

## Tests

1. `tests/sdd/test_plan_archive.py` — archiving over an existing destination preserves
   that destination's `create_time`; two consecutive `archive_plan_file` calls produce
   byte-identical output.
2. New: `archive_approved_plan` resets **the SDD store repository**, not the workspace
   checkout, on `ReplayConflict`. Build a fixture where the leased plans clone holds a
   local commit and upstream has moved with a conflicting version of the same file;
   assert attempt 1 conflicts, the reset lands on the upstream tip, and attempt 2
   publishes.
3. New: a pre-existing unpublishable commit in the leased plans clone (the exact 09:20 →
   09:27 poison scenario) does not fail a fresh, otherwise-unrelated archive.
4. New: an exhausted attempt budget leaves the store repository at its upstream tip,
   with the pre-reset HEAD reachable via the recovery ref, and names that ref in the
   failure notification.
5. New: a `skipped_locked` push outcome raises `ReplayDeferred` and performs no
   destructive reset (assert no `refs/sase/reset_replay/*` ref is created).
6. Keep `tests/workspace_provider/test_ownership_invariant_audit.py:357`
   (`test_approval_time_plan_archive_writes_leased_checkout_only`) green — the widened
   reset target must not widen the ownership blast radius.
7. Keep the authorization suite in `tests/workspace_provider/test_reset_replay.py`
   (`TestRefusesUnauthorizedContexts`) green, and extend it to cover the newly exposed
   reset entry point from Step 2.

## Out of scope

**The dual-writer design itself.** `run_agent_exec_plan_accept.py:314` and
`archive_approved_plan` both publish the same canonical plan file for the same approval,
with different commit messages. After this tale they converge instead of conflicting,
but the redundant second write remains. Unifying them behind a single publisher is a
separate design change — file a task bead via `/sase_new_task` rather than folding it in
here.

**Rust core boundary.** Publication and conflict policy arguably qualifies as core
backend behavior under the `rust_core_backend_boundary` memory, but the entire SDD
publication stack (`sase.sdd.*`, `sase.bead.sync*`, `sase.workspace_provider.*`) is
Python today with no `sase_core` counterpart. This fix stays in Python and does not
widen the boundary; migrating the stack is its own epic.

## Verification

```bash
just install
just check
```

Run `just check-full` through `/sase_monitor` before landing — this touches the
publication and lease paths, which are in the broadening set.

Manual confirmation: approve two plans in quick succession from the TUI and confirm no
`plan-archive` failure notification arrives, that both land in `sase--plans`, and that
the leased workspace's `sase/repos/plans` is left with `HEAD` an ancestor of
`@{upstream}`.
