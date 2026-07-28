---
tier: epic
title: Converge diverged bead stores with one-shot semantic merge integration
goal: 'Diverged bead-store clones converge deterministically: SDD sidecar integration
  reconciles divergent event streams with a single semantic three-way merge instead
  of an unwinnable per-commit rebase replay, doomed refresh attempts are cooled down
  instead of retried every few seconds, the periodic canonical-store sync publishes
  converged state to origin, `sase bead` reads never crash or flip-flop while an integration
  is in flight, and the claimed → in_progress → closed bead lifecycle survives multi-clone
  concurrency intact.

  '
phases:
- id: merge_integration
  title: One-shot merge replaces per-commit rebase replay for bead divergence
  depends_on: []
  size: medium
  description: 'merge_integration: when an SDD sidecar checkout has diverged from
    upstream and the divergence touches bead state, integrate with a single `git merge`
    reconciled once by the existing semantic bead conflict resolver against the true
    merge base, instead of replaying local commits one at a time with `git rebase`.
    Keep fail-closed behavior for non-bead conflicts, adapt the post-integration verification
    to accept merge commits, and add regression tests reproducing the colliding-ordinal
    divergence that today aborts with "cannot merge non-append-only bead event stream".

    '
- id: refresh_cooldown
  title: Failed-integration cooldown and TTL gating for blocking bead refreshes
  depends_on: []
  size: small
  description: 'refresh_cooldown: extend the durable failed-integration cooldown (today
    only guarding workspace sidecar clone materialization in `_store_link.py`) and
    the freshness-marker TTL to the blocking `refresh_bead_store` path used by waiting
    agents and the `bead_store_refresh` chop, so a doomed or recently-completed integration
    is not retried every few seconds by every waiter.

    '
- id: canonical_push
  title: Periodic canonical-store sync publishes converged bead state
  depends_on:
  - merge_integration
  size: small
  description: 'canonical_push: after the `bead_store_refresh` chop successfully integrates
    a canonical bead store that has local bead commits, publish them with the managed
    sync worker (fetch → merge-integrate → push) instead of stopping after the pull-only
    refresh, so claims recorded on the primary reach origin within one chop interval.
    Waiter-side read recovery stays pull-only.

    '
- id: reader_resilience
  title: Bead reads survive concurrent store integration
  depends_on: []
  size: small
  description: 'reader_resilience: make `sase bead list` and the other read-facade
    callers retry briefly when the Rust event-stream validation fails while an integration
    is rewriting the worktree, and turn a persistent failure into an actionable single-line
    error instead of a traceback.

    '
- id: claim_lifecycle
  title: Claimed-status lifecycle guarantees and coverage
  depends_on:
  - merge_integration
  size: small
  description: 'claim_lifecycle: lock in the claimed-status contract with tests that
    span divergent clones: every waiting/queued agent holds a claimed bead, starting
    work transitions exactly that bead from claimed to in_progress, releases return
    claimed beads to open, and a one-shot merge of two clones that both recorded claims
    and transitions preserves this history without duplicate or lost claims.

    '
- id: live_verification
  title: Converge and verify the live sase bead store end to end
  depends_on:
  - merge_integration
  - refresh_cooldown
  - canonical_push
  - reader_resilience
  - claim_lifecycle
  size: small
  description: 'live_verification: with the fixes deployed on the primary machine,
    confirm the diverged plans clone converges (ahead/behind 0 against origin), repeated
    `sase bead` invocations are stable and correct, the reflog shows no new rebase-abort
    loops, and `bead_sync_diagnostics` reports a clean store.

    '
create_time: 2026-07-26 09:34:58
status: wip
bead_id: sase-9u
---

- **PROMPT:** [202607/prompts/bead_store_merge_convergence.md](prompts/bead_store_merge_convergence.md)
- **BEAD:** sase-9u

# Converge diverged bead stores with one-shot semantic merge integration

## Problem

`sase bead` (list) on the primary machine flip-flops between two different status snapshots on consecutive runs and
intermittently crashes with:

```
ValueError: validation: invalid bead event stream
.../sase/repos/plans/beads/events/streams/sase-9t.jsonl line 16: expected value at line 1 column 1
```

The canonical bead store for the `sase` project (the primary workspace's `sase/repos/plans` clone) has diverged from
origin (observed: ahead 13 / behind 11), and its git reflog shows an SDD integration rebase being attempted and aborted
every few seconds, e.g.:

```
rebase (start): checkout @{upstream}
rebase (continue): chore(beads): claim sase-9t.2 for sase-9t.2
rebase (continue): chore(beads): claim sase-9t.1 for sase-9t.1
rebase (continue): chore(beads): claim sase-9t.4 for sase-9t.4
rebase (abort): returning to refs/heads/main
```

## Root cause

Bead writes are multi-writer by design: waiting agents commit `claimed` events to the canonical store (unpushed, via
`commit_bead_claim`), while working agents commit claim/`in_progress`/`closed` events in their workspace sidecar clones
and push to origin. Each clone mints per-stream sequential event ordinals independently, so the same stream diverges
with colliding ordinals (both sides created a `sase-9t:000015:*` event).

Reconciliation is where this breaks down, in four compounding ways:

1. **Per-commit rebase replay cannot converge renumbered streams.** `integrate_sdd_repository_transaction` runs
   `git rebase @{upstream}` and resolves each conflicted pick with the semantic stream merge (`merge_event_streams` in
   sase-core). The semantic merge _renumbers_ event ordinals when it interleaves both sides. After the first resolved
   pick, every subsequent pick's three-way merge sees an already-renumbered "ours" side, and the Rust append-only
   validator refuses it. Reproduced deterministically in a scratch clone of the live store: picks 1–3 resolve, pick 4
   fails every time with

   ```
   ValueError: validation: cannot merge non-append-only bead event stream
   sase-9t: theirs rewrote base event 17
   ```

   after which `_repair_or_abort_rebase` aborts the whole rebase. The divergence is therefore _permanently_ unresolvable
   under the current strategy.

2. **Doomed refreshes retry with no cooldown.** `refresh_bead_store` is called from every waiting agent's dependency
   poll (`refresh_bead_wait_store`) and from the `bead_store_refresh` chop. The failed-integration cooldown added for
   sase-9r.5 only guards workspace sidecar clone materialization (`_store_link.py`), and the freshness marker is only
   touched on success — so the unwinnable rebase re-runs every few seconds forever.

3. **Readers observe the churning worktree.** Each rebase attempt checks out upstream, replays commits, writes conflict
   markers, then aborts back to local `main`. `sase bead list` reads the stream files lock-free, so it sees whichever
   history is momentarily on disk (the flip-flop) or a conflict-marker line (`<` is serde's "expected value at line 1
   column 1" — the crash).

4. **Nothing publishes the canonical store.** The refresh path is deliberately pull-only, and pushes only happen on
   work-launch, so the primary's claim commits sit local indefinitely, guaranteeing ever-growing divergence.

**Validation of the fix direction:** in the same scratch clone where the rebase replay fails on pick 4 forever, a single
`git merge` of upstream followed by exactly one `resolve_bead_conflicts` round (three-way merge against the _true merge
base_, where both sides genuinely are append-only) resolved all five conflicted files, committed cleanly, and produced
the semantically correct interleaved history: `sase-9t.1` claimed → in_progress → closed, `sase-9t.2` claimed →
in_progress, `sase-9t.3`–`.6` claimed, with ordinals renumbered coherently once. No sase-core changes were needed:
`merge_event_streams` already handles the true-merge-base case.

## Phase 1 — One-shot merge integration

In `src/sase/sdd/_repository_integration.py` (and its callers in `_repository_transaction.py` / `bead/sync_worker.py`):

- Keep the current fast paths untouched: no upstream → success; upstream already an ancestor of HEAD → success; HEAD
  behind with no local commits → fast-forward via the existing rebase (which cannot conflict).
- When histories have truly diverged, stop replaying commits. Run `git merge --no-ff --no-commit @{upstream}` (machine
  identity, no editor), classify conflicts exactly as `_repair_or_abort_rebase` does today:
  - Any non-bead conflict → abort the merge, restore the starting state, and return `ABORTED_UNSUPPORTED_CONFLICTS`
    (unchanged fail-closed contract).
  - Bead-only conflicts → run `resolve_bead_conflicts` once and commit the merge. The resolver's stage handling already
    distinguishes merge from rebase (`_upstream_and_local_stages`), so ours/theirs orientation is correct without
    changes.
- Adapt `_successful_integration` verification: `merge-base --is-ancestor @{upstream} HEAD` already holds for a merge
  commit; audit the remaining checks (branch unchanged, clean worktree, porcelain-state comparison) and the health
  inspection for assumptions of linear history. Grep the sdd/ and bead/ modules for other is-ancestor / linear-history
  assumptions and adjust or document them.
- Remove or repurpose the now-dead per-commit repair loop; keep `_abort_and_verify` as the shared rollback path for the
  merge case (`git merge --abort` instead of `rebase --abort` when a merge is in flight).
- Tests (`tests/sdd_store/`, bead conflict tests): build two clones from a common base; clone A records claims for
  several beads (colliding ordinals), clone B records claims plus in_progress/closed transitions for overlapping beads
  and becomes upstream. Assert: the old strategy's failure signature ("cannot merge non-append-only bead event stream")
  no longer occurs, one integration call converges A onto B in a single round, the merged stream is valid (every line
  parses, ordinals strictly increasing), and reduced issue state matches the expected per-bead statuses. Also cover:
  non-bead conflict still aborts fail-closed; merge-commit result passes `require_sdd_repository_health` and a follow-up
  push.
- Rust core boundary: no `sase-core` changes are expected. If a validation relaxation ever proves necessary it belongs
  in `../sase-core/crates/sase_core`, not in Python — call this out in review rather than working around it locally.

## Phase 2 — Refresh cooldown

- Generalize the durable failed-integration cooldown marker from `sase/sdd/_repository_recovery_markers.py` +
  `_store_link.py` so `refresh_bead_store` (in `sase/bead/sync.py`) consults and updates it: a refresh that ends in
  `ABORTED_UNSUPPORTED_CONFLICTS` (or any non-success) records a failure and subsequent blocking refreshes within the
  cooldown window return immediately with the recorded reason instead of re-running fetch/rebase.
- Gate blocking refreshes on the existing freshness marker TTL (`integration_is_fresh`) the same way the background
  scheduler already does, so N waiters polling in the same TTL window produce at most one integration.
- Clear the failure marker on the next successful integration (matching the `_store_link.py` semantics).
- Tests: a failing integration is attempted once, then suppressed within the cooldown; a fresh marker suppresses a
  repeat refresh; success clears the failure state.

## Phase 3 — Publish converged state

- In `sase/scripts/sase_chop_bead_store_refresh.py`, when the canonical store integrates successfully and
  `bead_sync_diagnostics`-style probing shows unpushed local bead commits, run the managed sync worker
  (`run_managed_sync_worker`: fetch → integrate → push) instead of the pull-only `refresh_bead_store`, so waiting-agent
  claims recorded on the primary reach origin within one chop interval.
- Keep `refresh_bead_wait_store` (waiter dependency polling) pull-only: a waiting agent must never block on push
  credentials or network write access.
- Respect the existing backoff state in the chop for push failures too.
- Tests: chop-level test that a store with local bead commits gets pushed after successful integration; a store with no
  local commits stays pull-only; push failure feeds the chop backoff without corrupting local state.

## Phase 4 — Reader resilience

- In `src/sase/core/bead_read_facade.py` (and the equivalent read entry points used by `sase bead list` / the TUI beads
  tab), catch the Rust validation `ValueError` for invalid event streams, retry after a short bounded backoff (e.g. 3
  attempts over ~1s) to skate over an in-flight integration rewriting the worktree, and on persistent failure raise a
  single-line actionable error (naming the store path and suggesting `sase bead resolve-conflicts`) instead of surfacing
  a traceback.
- Keep reads lock-free: taking the store write lock in read paths would let a wedged writer hang every `sase bead`
  invocation and the TUI.
- Tests: a stream file containing conflict markers yields the clean error after retries; a file that becomes valid
  between attempts succeeds; the retry path is exercised for both the CLI list and status-summary readers.

## Phase 5 — Claim lifecycle coverage

- Verify and lock in the contract the user depends on, with tests that span divergent clones (building on the Phase 1
  test harness):
  - Every waiting/queued agent holds its bead in `claimed` (`claim_for_agent_wait`), and `sase bead list` renders it as
    such.
  - When an agent stops waiting and starts working, exactly that bead transitions `claimed` → `in_progress`; sibling
    claimed beads are untouched.
  - Claim release (`release_agent_claim`) returns `claimed` → `open` without disturbing other holders.
  - After a one-shot merge of two clones that both recorded claims and transitions for overlapping beads, the reduced
    state keeps exactly one holder per bead, no duplicate claim events survive reduction incorrectly, and closed beads
    stay closed.
- Fix any gaps these tests expose (expected to be small or none — the event semantics already reduced correctly in the
  scratch-clone validation).

## Phase 6 — Live verification

On the primary machine, after the previous phases land:

- Trigger one sync cycle (the `bead_store_refresh` chop or the managed sync worker) against the project's canonical
  plans store and confirm it converges: `git rev-list --left-right --count @{upstream}...HEAD` reports `0 0`, and the
  previously colliding `sase-9t` stream contains the merged, strictly-ordered history.
- Run `sase bead` repeatedly and confirm stable, correct output (no flip-flop, no tracebacks), with
  claimed/in_progress/closed statuses matching the actual agent states.
- Confirm the reflog accrues no new rebase-abort loops and `bead_sync_diagnostics` reports no divergence, recovery refs,
  or recovery stashes.
- Record the observed before/after state in the phase's completion notes so the epic closes with evidence.
