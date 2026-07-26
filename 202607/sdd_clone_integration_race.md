---
tier: epic
title: Serialize bead-store writes with SDD sidecar integration
goal: 'Concurrent bead-store writers and SDD sidecar integration in the same clone
  can no longer wedge each other: `git rebase --continue` is never fed a foreign un-staged
  worktree, a successful `rebase --abort` is never escalated to `UNRECOVERABLE`, machine-managed
  recovery never discards committed bead claims, and the recurring `sdd-sidecar` /
  `workspace_sdd_clone_recovery` axe error stops.

  '
phases:
- id: serialize
  title: One critical section for bead mutation, commit, and integration
  depends_on: []
  size: medium
  description: '''One critical section for bead mutation, commit, and integration''
    section: put the bead-store worktree materialization and its git commit inside
    the same store-write-lock critical section that SDD integration already uses,
    so a claim''s un-staged JSONL can never land inside another process''s rebase.

    '
- id: verify
  title: Rollback verification asserts only SASE-owned invariants
  depends_on: []
  size: medium
  description: '''Rollback verification asserts only SASE-owned invariants'' section:
    stop treating a foreign worktree or untracked delta as proof that SASE broke the
    clone, so a clean `rebase --abort` reports the benign aborted status instead of
    escalating to `UNRECOVERABLE` and triggering destructive recovery.

    '
- id: probes
  title: Conflict probes stop reporting git failures as "no conflicts"
  depends_on: []
  size: small
  description: '''Conflict probes stop reporting git failures as "no conflicts"''
    section: make the unmerged-path and conflicted-file probes distinguish "clean"
    from "could not tell", route the bead conflict resolver''s probes through the
    shared git-lock retry policy, and stop rewriting every event stream on every conflict.

    '
- id: rerere
  title: Machine-managed SDD git ignores the user's rerere config
  depends_on: []
  size: small
  description: '''Machine-managed SDD git ignores the user''s rerere config'' section:
    disable rerere for every SASE-issued git command in a machine-managed SDD store
    and purge the resolution cache that the user''s global `rerere.autoupdate` already
    leaked into the shared plans clone.

    '
- id: backoff
  title: Bound repeated doomed integration attempts
  depends_on:
  - verify
  size: small
  description: '''Bound repeated doomed integration attempts'' section: add a per-clone
    failed-integration cooldown so a clone that cannot rebase is not re-rebased about
    once per second, and report the suppressed attempts instead of hiding them.

    '
- id: lockwait
  title: Worktree-mutating callers wait for the store write lock instead of failing
    open
  depends_on:
  - serialize
  size: small
  description: '''Worktree-mutating callers wait for the store write lock instead
    of failing open'' section: keep fail-open only where it is safe, and make the
    paths that mutate a shared SDD worktree either wait long enough under contention
    or fail closed.

    '
- id: repair
  title: Reconcile the discarded claims and reap the recovery residue
  depends_on: []
  size: small
  description: '''Reconcile the discarded claims and reap the recovery residue'' section:
    audit every retained recovery ref and stash in the shared plans clone for bead
    state that never reached the remote, reconcile what is still live, and add bounded
    reaping so the residue does not accumulate forever.

    '
- id: soak
  title: Concurrent-writer soak exercise
  depends_on:
  - serialize
  - verify
  - probes
  - rerere
  - backoff
  - lockwait
  size: medium
  description: '''Concurrent-writer soak exercise'' section: drive many concurrent
    bead claims against one clone while integration runs and assert no `UNRECOVERABLE`
    outcome, no discarded local commit, and no axe error digest entry.

    '
create_time: 2026-07-26 06:48:19
status: done
bead_id: sase-9r
---

# Plan: Serialize bead-store writes with SDD sidecar integration

## Problem

The `sdd-sidecar` lumberjack has been emitting `workspace_sdd_clone_recovery` errors into the axe digests roughly once
or twice a day since 2026-07-24. The most recent instance is `~/.sase/axe/error_digests/digest_20260726_062719.txt`:

```
Time:       2026-07-26T06:10:37.319210-04:00
Lumberjack: sdd-sidecar
Job:        workspace_sdd_clone_recovery
Error:      git rebase --continue failed: You must edit all merge conflicts and then
mark them as resolved using git add; rollback verification failed: worktree or index
differs from the starting state; machine-managed recovery failed: could not snapshot
local changes: No local changes to save; snapshot rollback failed: worktree or index
differs from the starting state
```

`~/.sase/axe/recent_errors.json` gives the affected clone: `/home/bryan/projects/github/sase-org/sase/sase/repos/plans`
— the shared `plans` SDD sidecar clone inside the developer's primary checkout, which every axe-side process
(orchestrator, lumberjacks, hooks, agent launches, `sase bead` mutations) writes through.

This is not only log noise. On 2026-07-26 at 06:16:03 the machine-managed recovery path reset that clone to
`origin/main` and permanently discarded six committed bead claims:

```
1621687a chore(beads): claim sase-9q.3 for sase-9q.3
08b6444c chore(beads): claim sase-9q   for sase-9q.land
d570fa6e chore(beads): claim sase-9q.7 for sase-9q.7
088802ec chore(beads): claim sase-9q.5 for sase-9q.5
5d58de8a chore(beads): claim sase-9q.4 for sase-9q.4
34f874d3 chore(beads): claim sase-9q.6 for sase-9q.6
```

Those commits survive only as `refs/sase/recovery/20260726T101602Z-main-48ce14e9f7`. In that tip `sase-9q`, `sase-9q.3`,
`.4`, `.5`, `.6` and `.7` are all `claimed` with an assignee; in the current store every one of them is `open` with no
assignee, while the corresponding agents sit in `WAITING` believing they hold the claim. The clone has also accumulated
35 `refs/sase/recovery/*` refs and 3 stash entries from earlier instances of the same failure.

## Root cause

Two disjoint critical sections write the same clone.

1. **Bead-store materialization** writes `beads/issues.jsonl` and `beads/events/streams/*.jsonl` directly into the
   worktree under the Rust bead-store lock only. `claim_bead_for_waiting_agent` (`src/sase/bead/claims.py:135`) calls
   `project.claim_for_agent_wait(...)` (`src/sase/bead/project.py:248`, which delegates to
   `sase.core.bead_mutation_facade`), and _then_, after that `with` block has exited, calls `commit_bead_claim`.
   `store_git_write_lock` is never held while the worktree is written.

2. **Git integration** holds `store_git_write_lock` across fetch/rebase/repair/rollback
   (`src/sase/sdd/_repository_integration.py:79`).

`_commit_bead_state` (`src/sase/bead/sync.py:519`) does take `store_git_write_lock`, but only around `git add` +
`git commit` — by then the worktree write has already happened outside any lock that integration respects. So the
mutation and its commit are two separate critical sections with an unlocked worktree write in the gap.

Under an epic fan-out this gap is wide open. Between 06:09:38 and 06:11:01 on 2026-07-26, seven `sase-9q.*` agents
claimed beads in this one clone. `~/.sase/logs/tui_git_ops.jsonl` records the exact interleaving that produced the
digest entry:

| time         | op                          | result                                                                                            |
| ------------ | --------------------------- | ------------------------------------------------------------------------------------------------- |
| 06:10:32.605 | `sdd.clone.clean`           | not logged, i.e. rc 0 — the worktree **was** clean pre-rebase                                     |
| 06:10:32.772 | `sdd.clone.rebase`          | rc 1, `CONFLICT` in `beads/events/streams/sase-9q.jsonl` and `beads/issues.jsonl` on pick 1/4     |
| —            | bead conflict resolver      | merges streams and stages them                                                                    |
| 06:10:33.408 | `sdd.clone.rebase_continue` | rc 1, **stdout** `You must edit all merge conflicts and then mark them as resolved using git add` |
| 06:10:33.467 | `sdd.health.branch`         | rc 1 (detached) — `_abort_and_verify` has begun                                                   |
| 06:10:37.230 | `sdd.clone.recovery.stash`  | rc 0, `No local changes to save`                                                                  |
| 06:10:37.916 | `sdd.clone.clean`           | rc 1 — the worktree is dirty again                                                                |
| 06:10:37.994 | `bead.claim.add`            | rc 128, `fatal: Unable to create '.../index.lock': File exists.`                                  |
| 06:10:38.124 | `bead.claim.add`            | rc 0 on retry                                                                                     |

Three things follow from that trace.

**The git message is misleading.** It arrives on _stdout_, because `builtin/rebase.c` emits it with `puts()` from the
`ACTION_CONTINUE` branch guarded by `has_unstaged_changes()`, not by an unmerged-index check. There were no unresolved
conflicts: the very next loop iteration in `_repair_or_abort_rebase` (`src/sase/sdd/_repository_integration.py:198`)
found no unmerged paths and fell straight through to `_abort_and_verify`. What actually blocked the continue was another
process's un-staged bead-store write, visible dirty at 06:10:37.916 and staged at 06:10:38.124.

**A clean abort is reported as an unrecoverable repository.** `_abort_and_verify`
(`src/sase/sdd/_repository_integration.py:375`) aborts successfully and then compares
`git status --porcelain=v1 -z --untracked-files=all` against the pre-rebase snapshot via `sdd_rollback_mismatch`
(`src/sase/sdd/_repository_health.py:193`). Any delta — including a transient file that belongs to another writer and
that SASE never touched — yields `worktree or index differs from the starting state`, which promotes the benign
`ABORTED_UNSUPPORTED_CONFLICTS` to `UNRECOVERABLE`. The repository was provably fine: no operation markers, branch
reattached, HEAD back at the pre-rebase commit.

**`UNRECOVERABLE` then invites destructive recovery.** `integrate_machine_managed_sdd_repository`
(`src/sase/sdd/_repository_transaction.py:115`) calls `recover_machine_managed_sdd_repository`, whose
`snapshot_managed_changes` (`src/sase/sdd/_repository_recovery_snapshot.py:19`) sees a non-empty
`before.status_porcelain`, runs `git stash push --include-untracked`, gets rc 0 and `No local changes to save` because
the foreign transient file is already gone, computes `stash_created = False`, and falls into `_restore_failed_snapshot`,
where `sdd_rollback_mismatch(before, final)` compares the same moving target and fails again. That composes the exact
digest string, and `_pull_sdd_clone` publishes it through `_append_recovery_error` (`src/sase/sdd/_store_link.py:340`).

When recovery does get past the snapshot, it stashes or `reset --hard`s the shared clone — which is how the six
`sase-9q` claim commits were discarded at 06:16:03, and how the 3 stash entries and 35 recovery refs accumulated.

### Amplifiers

- **No backoff.** The clone's reflog holds 311 `rebase (start)` / `rebase (abort)` pairs at roughly one per second
  between 06:10:39 and 06:16:02. Every integration attempt re-runs the full fetch and doomed rebase.
- **The user's global rerere config applies to a machine-generated store.** `git config --global` has `rerere.enabled 1`
  and `rerere.autoupdate true`. The 06:10:39 telemetry shows `Recorded resolution for 'beads/issues.jsonl'.`, and the
  clone's `.git/rr-cache` holds well over a hundred entries. A cached human resolution can be auto-staged over generated
  append-only JSONL.
- **Probes report failure as cleanliness.** `unmerged_paths` (`src/sase/sdd/_repository_health.py:108`) returns `()` and
  `_conflicted_files` (`src/sase/bead/conflict_resolver.py:126`) returns `[]` on _any_ non-zero exit. Both run
  `git diff` without `--cached`, which takes `index.lock` to write the refreshed index, so index.lock contention reads
  as "no conflicts". The resolver's comment at `src/sase/bead/conflict_resolver.py:107` claims its probes are read-only
  and therefore skip `run_with_git_lock_retry`; that is wrong for `git diff`.
- **Oversized resolutions.** `_write_resolved_store` (`src/sase/bead/conflict_resolver.py:243`) rewrites every stream
  file in the worktree on every conflict — the 06:10:39 continue reports
  `79 files changed, 774 insertions(+), 772 deletions(-)` for a single-line claim — widening both the window and the
  blast radius.
- **The store write lock fails open.** `store_git_write_lock` (`src/sase/sdd/_git_contention.py:84`) logs a warning and
  yields `False` after `DEFAULT_STORE_WRITE_LOCK_TIMEOUT_SECONDS = 10.0`. A seven-agent claim burst reaches that bound.

### Prior art

`sase-8g.5` (self-heal wedged SDD clones), `sase-8g.7` (reduce bead stream sync conflicts), `sase-67` / `sase-67.2`
(serialize SDD store git writers, retry index.lock) and `sase-6r.2` (TTL-gate sidecar integration) are all closed. They
built the machinery this plan repairs; none of them closed the mutation/integration gap. No open, claimed, or
in-progress bead covers it, and no currently running agent is working on it.

## One critical section for bead mutation, commit, and integration

Close the gap that lets an un-staged bead-store write land inside another process's rebase.

Every code path that materializes bead state into a shared SDD worktree and then commits it must hold
`store_git_write_lock` across _both_ steps, so that integration — which already holds the same lock — cannot observe a
half-written worktree.

- Wrap the mutation-then-commit sequence in `claim_bead_for_waiting_agent` (`src/sase/bead/claims.py:122`) in one
  acquisition, and audit the sibling writers for the same shape: `commit_bead_work_launch`, `commit_bead_claim_release`,
  `commit_epic_graph_checkpoint`, `commit_epic_creation_rollback` and `git_sync` in `src/sase/bead/sync.py`, plus
  `src/sase/bead/cli_work_from_plan_store.py:77` and `src/sase/axe/run_agent_exec_plan_accept.py:299`.
- `store_git_write_lock` uses `flock` on a freshly opened descriptor, so a nested acquisition in the same process
  self-contends until the timeout and then fails open. Introduce an explicit re-entrant or hand-off form rather than
  nesting. `refresh_bead_store` (`src/sase/bead/sync.py:517`) already models the hand-off by passing
  `lock_factory=lambda _root: nullcontext(True)`, and `_repository_recovery_git.already_locked` exists for the same
  purpose; prefer one shared mechanism over ad-hoc lambdas, and make an accidental nested acquisition fail loudly in
  tests instead of silently degrading to fail-open.
- Keep the lock's scope honest about cost: the mutation is local work, so holding the lock across mutation plus commit
  adds contention but no network time. Integration deliberately fetches _before_ taking the lock
  (`src/sase/sdd/_repository_integration.py:69`); preserve that.
- If the Rust bead mutation API cannot materialize under a caller-held lock without changes, make those changes in the
  sibling core repo at `../sase-core/crates/sase_core` and thread them through the binding, per the repo's Rust-core
  backend boundary. Do not reimplement bead mutation on the Python side to work around the lock.

Cover it with a test that holds a simulated foreign writer mid-mutation and proves integration either waits or reports a
benign outcome, never `UNRECOVERABLE`.

## Rollback verification asserts only SASE-owned invariants

`sdd_rollback_mismatch` currently treats a full `--untracked-files=all` porcelain snapshot as the contract. That is
stronger than what SASE can promise in a shared clone, and the overreach is what escalates a healthy repository to
`UNRECOVERABLE`.

- Split the comparison into invariants SASE owns — branch attached and unchanged, HEAD back at the starting commit, no
  operation markers, no unmerged index entries, no SASE-authored tracked or staged residue — and observations SASE does
  not own, namely worktree and untracked deltas that SASE never wrote.
- When only the un-owned observations differ, report the benign restored outcome (`ABORTED_UNSUPPORTED_CONFLICTS`) and
  do not emit an axe error. Log the delta at warning level so the churn stays visible without paging.
- Apply the same distinction inside `snapshot_managed_changes` and `_restore_failed_snapshot`
  (`src/sase/sdd/_repository_recovery_snapshot.py`), where the identical moving-target comparison produces
  `snapshot rollback failed`. Re-read the "before" state under the lock immediately before comparing, rather than
  reusing a snapshot taken several seconds and several git commands earlier.
- Treat `git stash push` returning rc 0 with `No local changes to save` as "nothing to snapshot, proceed", not as a
  snapshot failure.
- Since `UNRECOVERABLE` is what authorizes `reset --hard` and stashing in a shared clone, narrowing it is also the guard
  against a repeat of the discarded `sase-9q` claims. Add a regression test that a clone whose only difference is a
  foreign untracked file is never classified `UNRECOVERABLE`.

## Conflict probes stop reporting git failures as "no conflicts"

- Give `unmerged_paths` (`src/sase/sdd/_repository_health.py:108`) a three-state result so callers can tell "no
  conflicts" from "could not determine". `_repair_or_abort_rebase` must not fall through to `_abort_and_verify` because
  a probe failed; it should surface the probe failure.
- Route `_conflicted_files`, `_read_stage_stream` and `_git_dir` in `src/sase/bead/conflict_resolver.py` through
  `run_with_git_lock_retry` (or the SDD git runner) and correct the stale comment at line 107 that asserts these probes
  cannot contend on `index.lock`. A failed probe must not be reported as
  `_BeadConflictResolution(True, "no conflicted bead files")`, which is what lets an unstaged resolution reach
  `rebase --continue`.
- Narrow `_write_resolved_store` (`src/sase/bead/conflict_resolver.py:243`) to the streams the merge actually changed,
  plus the manifest and `issues.jsonl` it must re-derive, so a one-line claim conflict stops rewriting 79 files. Keep
  the derived manifest and issues output authoritative; only stop rewriting byte-identical streams.

## Machine-managed SDD git ignores the user's rerere config

- Pass `-c rerere.enabled=false` (and, defensively, `-c rerere.autoupdate=false`) on every git invocation SASE issues
  against a machine-managed SDD store. `run_sdd_git` (`src/sase/sdd/_git.py:26`) is the single choke point for the SDD
  runners; the bead conflict resolver's direct `subprocess.run` calls need the same treatment once phase `probes` routes
  them through a shared runner.
- Explain in the code why: these stores hold generated append-only JSONL whose conflicts are resolved semantically by
  `resolve_bead_conflicts`. A cached textual resolution replayed by rerere — and auto-staged, because the developer's
  global config sets `rerere.autoupdate true` — can silently substitute stale content for a correct semantic merge.
- Purge the resolutions rerere already recorded in the shared plans clone's `.git/rr-cache`, and confirm no other
  machine-managed SDD clone carries one.
- Add a test that a machine-managed integration does not create or consult `rr-cache` even when `rerere.enabled` and
  `rerere.autoupdate` are set in the ambient git configuration.

## Bound repeated doomed integration attempts

- Record a per-clone failed-integration marker next to the existing recovery markers in
  `src/sase/sdd/_repository_recovery_markers.py`, and have `_pull_sdd_clone` (`src/sase/sdd/_store_link.py:262`) skip
  the fetch-and-rebase attempt while that marker is fresh, mirroring how `SddIntegrationStatus.RECOVERY_COOLDOWN`
  already suppresses repeated recovery. Reuse `machine_recovery_cooldown_seconds`
  (`src/sase/sdd/_repository_recovery_git.py:206`) rather than adding a second knob unless the horizons genuinely need
  to differ.
- Clear the marker as soon as an integration succeeds, so an ordinary conflict costs one attempt and not a cooldown.
- Count and report the suppressed attempts — through the existing SDD git telemetry — so a clone that is quietly stuck
  is still visible. Silent suppression must not read as health.
- Add a test asserting that N successive integration calls against a clone that cannot rebase perform one rebase, not N.

## Worktree-mutating callers wait for the store write lock instead of failing open

`store_git_write_lock` fails open by design, on the theory that per-command `index.lock` retries are the final safety
net (`src/sase/sdd/_git_contention.py:84`). That reasoning holds for a single git command; it does not hold for a caller
that mutates a worktree and expects the whole mutate-then-commit sequence to be atomic.

- Keep fail-open for the single-command callers where it is genuinely safe, and give the worktree-mutating callers a
  mode that either waits substantially longer under contention or fails closed with an actionable error. Do not silently
  flip today's behavior for every caller — enumerate the call sites and choose per site.
- Raise the effective wait for the mutate-and-commit critical section from phase `serialize` well above
  `DEFAULT_STORE_WRITE_LOCK_TIMEOUT_SECONDS = 10.0`, since a legitimate holder now does more work under the lock. Make
  the bound configurable through the existing `SASE_SDD_STORE_WRITE_LOCK_TIMEOUT` environment variable rather than a new
  one.
- Keep the fail-open warning, but make it identify the mutating operation, so the next incident names the caller that
  proceeded unlocked.

## Reconcile the discarded claims and reap the recovery residue

- Audit every `refs/sase/recovery/*` ref (35 at the time of writing) and every stash entry in
  `~/projects/github/sase-org/sase/sase/repos/plans` for bead state that was committed locally and never reached the
  remote. `refs/sase/recovery/20260726T101602Z-main-48ce14e9f7` is the known case: it holds `sase-9q`, `sase-9q.3`,
  `.4`, `.5`, `.6` and `.7` as `claimed`, while the live store has all six `open` and unassigned.
- Reconcile only what is still live. Those six beads belong to agents that are currently waiting; re-asserting a stale
  claim for an agent that has since finished or been dismissed would be worse than leaving the bead open. Decide per
  bead against the current agent list and record the reasoning.
- Access that clone through the `/sase_repo` skill, and treat it as production bead state: read first, and gate any
  write on user confirmation.
- Add bounded reaping for `refs/sase/recovery/*` refs and recovery stashes, so retained snapshots age out on a stated
  horizon instead of accumulating indefinitely. Reaping must refuse to drop a snapshot holding commits that are not
  reachable from the remote.

## Concurrent-writer soak exercise

- Build an exercise that drives many concurrent bead claims against a single SDD clone while integration runs,
  reproducing the 06:09–06:11 fan-out shape: several claims within ninety seconds against one clone whose upstream is
  simultaneously advancing with conflicting claim commits.
- Assert, on a pre-fix build, that the exercise reproduces the digest string, and on a post-fix build that it produces
  no `UNRECOVERABLE` outcome, no `workspace_sdd_clone_recovery` entry through `append_error`, no discarded local commit,
  and no new `refs/sase/recovery/*` ref.
- Keep it hermetic. Per `sase-9l`, tests must never resolve to the production plans sidecar; build the clone and its
  remote inside the test sandbox.
- Extend `bead_sync_diagnostics` (`src/sase/bead/sync.py:553`) to report the conditions this epic makes benign —
  retained recovery refs, recovery stashes, unpushed local bead commits — so the next occurrence is diagnosable from the
  clone alone rather than from correlating `~/.sase/logs/tui_git_ops.jsonl` against the reflog.
