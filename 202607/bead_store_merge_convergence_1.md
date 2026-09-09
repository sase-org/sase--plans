---
tier: epic
title: Converge diverged bead stores with one-shot semantic merge integration
goal: "Diverged SDD sidecar clones converge deterministically instead of wedging: true
  divergence is reconciled by a single semantic three-way merge against the real merge
  base rather than an unwinnable per-commit rebase replay, a doomed integration is
  recorded and cooled down instead of re-attempted by every waiter, the periodic
  canonical-store chop publishes converged bead state to origin, and `sase bead` reads
  survive an in-flight integration with a clean error instead of a traceback.

  "
phases:
  - id: merge_integration
    title: One-shot merge replaces per-commit rebase replay on true divergence
    depends_on: []
    size: medium
    description: "merge_integration: replace the diverged-history `git rebase` in
      `integrate_sdd_repository_transaction` with a single `git merge --no-commit`
      reconciled by one `resolve_bead_conflicts` round against the true merge base,
      keeping the fast-forward path linear and the non-bead fail-closed contract intact;
      extend `_abort_and_verify` and the machine-managed recovery gates to handle an
      interrupted merge, and add regression coverage for the colliding ordinal
      divergence, the base-rewritten fail-closed case, and claim lifecycle across two
      clones.

      "
  - id: refresh_gating
    title: Record and consult the failed-integration cooldown in blocking refreshes
    depends_on: []
    size: small
    description: "refresh_gating: make `refresh_bead_store` record a failed-integration
      marker on every non-success outcome and consult that cooldown before attempting
      work, add opt-out TTL gating so N concurrent waiters produce at most one
      integration, and return a typed outcome so callers can tell suppression apart from
      success.

      "
  - id: reader_resilience
    title: Bead reads survive a concurrent store integration
    depends_on: []
    size: small
    description: "reader_resilience: make the Rust-backed read facade retry briefly when
      event-stream validation fails while an integration is rewriting the worktree, and
      turn a persistent failure into a single-line actionable error at the `sase bead`
      and ACE entry points instead of a traceback.

      "
  - id: canonical_push
    title: Periodic canonical-store chop publishes converged bead state
    depends_on:
      - merge_integration
      - refresh_gating
    size: small
    description: "canonical_push: after the `bead_store_refresh` chop integrates a
      canonical store that still holds unpushed local bead commits, publish them with
      the managed sync worker instead of stopping at the pull-only refresh, keep the
      waiter path pull-only, and feed push failures into the existing per-project
      backoff.

      "
  - id: live_verification
    title: Converge and verify the live sase bead store end to end
    depends_on:
      - merge_integration
      - refresh_gating
      - reader_resilience
      - canonical_push
    size: small
    description: "live_verification: with the previous phases landed, converge the
      primary machine's canonical plans clone, confirm repeated `sase bead` runs are
      stable and correct, confirm the reflog accrues no new rebase-abort loops and
      `bead_sync_diagnostics` reports a clean store, and record the before/after
      evidence.

      "
create_time: 2026-09-09 19:53:01
status: wip
---

- **PROMPT:**
  [prompts/202607/bead_store_merge_convergence_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/bead_store_merge_convergence_1.md)

# Converge diverged bead stores with one-shot semantic merge integration

## Supersedes

This plan replaces `bead_store_merge_convergence.md`, the epic recorded as bead
**`sase-9u`** (phases `sase-9u.1` through `sase-9u.6`, all open, none started). The
problem statement and Phase 1 direction there are correct and are carried forward; this
revision narrows Phase 2, folds the old Phase 5 into Phase 1, adds work the original
missed, and records a deferred alternative. **Close `sase-9u` and its six phase beads as
superseded when this epic's beads are created** — leaving both open would launch
duplicate agents against the same files.

Changes from `sase-9u`:

- **Phase 2 narrowed.** The old framing ("retried every few seconds by every waiter")
  does not match the code. The chop already backs off per project to a 15-minute ceiling
  (`_MAX_BACKOFF_SECONDS` in `src/sase/scripts/sase_chop_bead_store_refresh.py:26`), and
  the runner's waiter loop refreshes on a 600s fallback interval
  (`_WAIT_BEAD_REFRESH_FALLBACK_INTERVAL` in `src/sase/axe/run_agent_wait.py:37`). The
  real gap is narrower and is what this phase fixes: `refresh_bead_store` _clears_ the
  failure marker on success but never _records_ or _consults_ it, and is not TTL-gated.
- **Old Phase 5 (`claim_lifecycle`) folded into Phase 1.** It was test-only and expected
  "small or none" production fixes; it shares Phase 1's two-clone harness and splitting
  it across agents would duplicate that harness.
- **Phase 1 widened** to cover the machine-managed recovery paths, which hard-code
  rebase as the only in-progress operation they can recover from. Introducing merges
  without this is a regression, and `sase-9u.1` did not mention it.
- **Phase 6 (`live_verification`) re-scoped.** Its premise — a standing diverged clone
  to converge — was stale when written; the wedge had already been cleared by hand. The
  store has since re-diverged on its own (see below), so live verification is again
  meaningful, but the phase now says how to reproduce divergence if the clone happens to
  be converged when the agent runs.

## Problem

`sase bead` (list) on the primary machine flip-flops between two different status
snapshots on consecutive runs and intermittently crashes with:

```
ValueError: validation: invalid bead event stream
.../sase/repos/plans/beads/events/streams/sase-9t.jsonl line 16: expected value at line 1 column 1
```

The canonical bead store for the `sase` project — the primary workspace's
`sase/repos/plans` clone — diverges from origin and its git reflog shows an SDD
integration rebase being started and aborted repeatedly:

```
rebase (start): checkout @{upstream}
rebase (continue): chore(beads): claim sase-9t.2 for sase-9t.2
rebase (continue): chore(beads): claim sase-9t.1 for sase-9t.1
rebase (continue): chore(beads): claim sase-9t.4 for sase-9t.4
rebase (abort): returning to refs/heads/main
```

That loop was last broken by hand, with a `reset: moving to origin/main` followed by a
hand-authored `chore(beads): converge divergent plan state` commit — i.e. by
**discarding the local claim commits**. That is the sharpest statement of the problem:
today the failure mode has no recovery that preserves canonical state.

It recurs on its own. At the time this plan was written the same clone was diverged
again, `4` behind / `2` ahead of `origin/main`, with no operator action in between.

## Root cause

Bead writes are multi-writer by design. Waiting agents commit `claimed` events into the
canonical store without pushing (`commit_bead_claim` in `src/sase/bead/_sync_git.py`,
reached from `claim_bead_for_waiting_agent` in `src/sase/bead/claims.py`), while working
agents commit claim / `in_progress` / `closed` events in their workspace sidecar clones
and push to origin. Each clone mints per-stream sequential event ordinals independently,
so the same stream diverges with colliding ordinals — both sides create a
`sase-9t:000015:*` event.

Reconciliation is where it breaks, in three compounding ways:

1. **Per-commit rebase replay cannot converge renumbered streams.**
   `integrate_sdd_repository_transaction`
   (`src/sase/sdd/_repository_integration.py:162`) runs `git rebase @{upstream}` and
   resolves each conflicted pick with the semantic stream merge, via
   `_repair_or_abort_rebase` → `resolve_bead_conflicts` → the `merge_event_streams`
   binding → `merge_bead_event_streams` in Rust. That merge _renumbers_ every non-base
   event as it interleaves (`renumber_event`, `crates/sase_core/src/bead/events.rs:545`,
   called from the merge loop at `events.rs:484-493`). After a resolved pick, the next
   pick's three-way merge sees an already-renumbered side, and
   `validate_append_only_branch` (`events.rs:498`) — which requires both sides to be
   byte-exact prefix extensions of the base — refuses it:

   ```
   ValueError: validation: cannot merge non-append-only bead event stream
   sase-9t: theirs rewrote base event 17
   ```

   (During a rebase `_upstream_and_local_stages` returns `(2, 3)`, so the accumulating
   rebased side is `theirs`; that is why the message names `theirs`.)
   `_repair_or_abort_rebase` then aborts the whole rebase. This does not fire on _every_
   multi-pick replay — it fires when the interleave actually reorders events — but once
   it fires for a given divergence, that divergence is permanently unresolvable under
   the current strategy.

2. **A doomed refresh is not recorded.** `refresh_bead_store`
   (`src/sase/bead/sync.py:194`) calls `clear_failed_integration_marker` on success but
   never calls `record_failed_integration_marker` or
   `admit_failed_integration_cooldown`. The durable cooldown built for sase-9r.5
   therefore only guards workspace sidecar clone materialization (`_pull_sdd_clone` in
   `src/sase/sdd/_store_link.py:267`). The integration-marker generation check inside
   `refresh_bead_store` coalesces concurrent waiters only when an integration
   _succeeded_, because a failure never touches the marker.

3. **Readers observe the churning worktree.** Each rebase attempt checks out upstream,
   replays commits, writes conflict markers, then aborts back to local `main`.
   `sase bead list` reads the stream files lock-free through
   `src/sase/core/bead_read_facade.py`, so it sees whichever history is momentarily on
   disk (the flip-flop) or a conflict-marker line — a leading `<` is exactly serde's
   "expected value at line 1 column 1".

4. **Nothing publishes the canonical store.** `refresh_bead_store` is deliberately
   pull-only and pushes happen only at work-launch, so the primary's claim commits sit
   local indefinitely and divergence grows without bound.

**Validation of the fix direction.** In a scratch clone of the live store where the
rebase replay fails on pick 4 every time, a single `git merge` of upstream followed by
exactly one `resolve_bead_conflicts` round — a three-way merge against the _true merge
base_, where both sides genuinely are append-only — resolved all five conflicted files,
committed cleanly, and produced the semantically correct interleaved history:
`sase-9t.1` claimed → in_progress → closed, `sase-9t.2` claimed → in_progress,
`sase-9t.3`–`.6` claimed, renumbered coherently once. See "Residual risk" for the limit
of that evidence.

## Phase 1 — One-shot merge replaces per-commit rebase replay on true divergence

All in `src/sase/sdd/_repository_integration.py` unless noted.

### Strategy selection

Keep the existing preflight, lock, fetch, health, and `tracked_changes_error` steps
unchanged. Keep the two existing fast paths: no upstream configured → `SUCCESS`;
upstream already an ancestor of `HEAD` → `_successful_integration` with
`integrated=False`.

Add a **fast-forward** discriminator before choosing a strategy. Probe
`merge-base --is-ancestor HEAD <upstream>`:

- exit 0 — `HEAD` is an ancestor of upstream, i.e. behind with no local commits. Keep
  the current `git rebase <upstream>` call. It fast-forwards, cannot conflict, and
  preserves linear history for what is by far the common case.
- exit 1 — histories have truly diverged. Take the merge path below.
- anything else — return `ABORTED_UNSUPPORTED_CONFLICTS` with the probe error, matching
  how the existing ancestor probe handles a non-0/1 exit.

### Merge path

1. `git merge --no-ff --no-commit <upstream>` through the existing `git_runner` (which
   already routes through `sdd_git_command`, so ambient rerere stays disabled — that
   matters here, because a cached textual resolution must never be replayed over the
   semantic merge).
2. On exit 0 (no conflicts): commit with
   `["-c", "core.editor=true", "commit", "--no-edit"]`. Note that `sdd_git_command`
   injects only the rerere-disabling args, not a committer identity, so this commits
   under the ambient identity exactly as `rebase --continue` does today.
3. On non-zero exit: probe `unmerged_paths`.
   - probe error → `_abort_and_verify`, naming the probe failure, exactly as
     `_repair_or_abort_rebase` does today. A failed probe must never be read as "no
     conflicts".
   - no unmerged paths → `_abort_and_verify` with the merge's own failure text; a merge
     that failed for a reason other than conflicts is not something to resolve.
   - unmerged paths present → run `resolve_bead_conflicts` **once**.
     `resolve_bead_conflicts` already classifies non-bead and unsupported-bead conflicts
     and returns `ok=False` for them, so the fail-closed contract is preserved without
     new classification logic. On `ok=False`, or on any exception, `_abort_and_verify`.
   - after a successful resolution, re-probe `unmerged_paths`; the resolver's `_git_add`
     should have cleared every conflicted entry. If any remain, `_abort_and_verify`
     rather than committing a partial resolution. Then commit as in step 2. Do **not**
     loop: one merge produces one conflict set, and looping is precisely the per-pick
     accumulation that causes the bug.
4. Emit the resolution event through the existing `_emit_resolution` so the managed sync
   worker's log keeps recording `conflict_resolution` entries.
5. Report `REPAIRED_BEAD_CONFLICTS` when the semantic resolver ran, `SUCCESS` otherwise,
   with `integrated=True` — the same status mapping `_successful_integration` already
   applies.

`_upstream_and_local_stages` in `src/sase/bead/conflict_resolver.py:279` already returns
`(3, 2)` when no rebase directory is present, so ours/theirs orientation is correct on
the merge path with no resolver change.

### Rollback and recovery

- `_abort_and_verify` currently only recognizes an in-progress rebase
  (`operation_markers` containing `"rebase"`) and issues `rebase --abort`. Add the merge
  case: `inspect_sdd_repository` already maps `MERGE_HEAD` to a `"merge"` marker
  (`src/sase/sdd/_repository_health.py:24`), so dispatch to `git merge --abort` when a
  merge is in flight. Keep the existing `reset --hard <starting.head>` branch for a
  completed-then-rejected integration; it is what unwinds a merge commit that failed
  post-verification.
- `managed_recovery_branch` (`src/sase/sdd/_repository_recovery_git.py:32-46`) refuses
  recovery when `operation_markers - {"rebase"}` is non-empty and refuses unmerged
  entries "outside a rebase". A process killed mid-merge leaves `MERGE_HEAD`, so without
  this change a wedged merge makes a workspace sidecar clone **unrecoverable** —
  strictly worse than today. Extend both predicates to accept `"merge"`, and derive the
  original branch for the merge case (a merge never leaves a detached HEAD, so
  `state.branch` is authoritative; there is no `head-name` file to read as there is for
  `rebase-merge`/`rebase-apply`).
- `recover_machine_managed_sdd_repository`
  (`src/sase/sdd/_repository_recovery.py:189-206`) issues `rebase --abort` whenever
  `starting.operation_markers` is non-empty, and `_verify_rebase_cleared` afterwards.
  Dispatch on the marker kind and verify the corresponding clearance.
- Audit `_successful_integration`'s post-checks for linear-history assumptions.
  `merge-base --is-ancestor <upstream> HEAD` holds for a merge commit; the
  branch-unchanged, `status_porcelain`-equality, and `tracked_changes_error` checks are
  history-shape agnostic and should need no change — but confirm this rather than assume
  it, and confirm `repair_event_manifest_after_integration` behaves on a merge commit.
- Grep the `sdd/` and `bead/` modules for any remaining is-ancestor or linear-history
  assumption and adjust or document what you find.
- Remove or repurpose the now-dead per-commit repair loop. `_repair_or_abort_rebase` is
  only reachable from the rebase call; once the fast-forward rebase cannot conflict, its
  conflict-handling body is unreachable. Deleting it is preferred over leaving dead
  code, but keep the shared helpers it uses.

### Scope note: this changes integration for every SDD sidecar

`integrate_sdd_repository_transaction` serves all SDD sidecars — `plans`, `research`,
`agents` — not only bead stores. Conditioning the strategy on "the divergence touches
bead state" would need a path-set diff before choosing, and a divergence that touches
both kinds would still need the merge. Do not condition it: **merge on any true
divergence.** Rebase replay is fragile for any machine-managed multi-writer store, and
`resolve_bead_conflicts` already fails closed on non-bead conflicts either way. The cost
is that these sidecars will occasionally carry merge commits; the fast-forward path
keeps history linear in the common case. Call this out in review — it is a deliberate
trade, not an oversight.

### Tests

`tests/test_bead/test_sync_conflict_regressions.py` already has the right harness
(`_seed_same_stream_remote`, `_seed_claim_soak_remote`) and
`tests/sdd_store/test_repository_transaction.py` has `_build_diverged_clones`. Extend
those rather than building new scaffolding.

- **The reported failure, reproduced.** Build two clones from a common base. Clone A
  records claims for several beads (colliding ordinals); clone B records claims plus
  `in_progress`/`closed` transitions for overlapping beads and becomes upstream. Assert
  the failure signature `cannot merge non-append-only bead event stream` no longer
  occurs, one integration call converges A onto B in a single round, every line of the
  merged stream parses, ordinals are strictly increasing, and the reduced issue state
  matches the expected per-bead statuses.
- **Fail-closed preserved.** A non-bead conflict still returns
  `ABORTED_UNSUPPORTED_CONFLICTS` and restores the exact starting state. Extend
  `test_mixed_bead_and_non_bead_conflicts_abort_exactly` and
  `test_unsupported_plan_conflict_aborts_to_exact_starting_state` to the merge path.
- **Base rewritten (see "Residual risk").** Construct a divergence where one side's
  post-merge-base history rewrites base events. Assert the integration aborts
  fail-closed to the exact starting state with an error naming the stream, and that the
  error text is actionable (points at `sase bead resolve-conflicts`).
- **Fast-forward stays linear.** A clone that is purely behind still fast-forwards and
  produces no merge commit.
- **Merge result is durable.** The merge commit passes `require_sdd_repository_health`
  and a follow-up `run_managed_sync_worker` push succeeds.
- **Interrupted merge recovers.** A workspace sidecar clone left with `MERGE_HEAD` is
  recovered by `integrate_machine_managed_sdd_repository`, mirroring the existing
  `test_machine_managed_clone_recovers_preexisting_rebase` parametrization.
- **Claim lifecycle across clones** (folded from `sase-9u.5`; extends
  `tests/test_bead/test_claimed_status.py` using this phase's two-clone harness): every
  waiting/queued agent holds its bead in `claimed` via `claim_for_agent_wait`, and
  `sase bead list` renders it as such; starting work transitions exactly that bead
  `claimed` → `in_progress` and leaves sibling claimed beads untouched;
  `release_agent_claim` returns `claimed` → `open` without disturbing other holders; and
  after a one-shot merge of two clones that both recorded claims and transitions for
  overlapping beads, the reduced state keeps exactly one holder per bead, no duplicate
  claim survives reduction, and closed beads stay closed. Fix whatever gaps these expose
  — expected to be small or none, since the event semantics reduced correctly in the
  scratch-clone validation.

### Rust core boundary

No `sase-core` change is expected: `merge_bead_event_streams` already handles the
true-merge-base case. If a validation relaxation proves necessary it belongs in
`../sase-core/crates/sase_core`, **not** in Python. Do not work around
`validate_append_only_branch` locally — raise it in review instead.

## Phase 2 — Record and consult the failed-integration cooldown in blocking refreshes

All in `src/sase/bead/sync.py` plus its four call sites.

- `refresh_bead_store` must call `record_failed_integration_marker` on every non-success
  outcome, mirroring `_pull_sdd_clone`'s use in `src/sase/sdd/_store_link.py:334-335`,
  and must call `admit_failed_integration_cooldown` before attempting an integration.
  Keep the existing `clear_failed_integration_marker` on success. Pass
  `lock_factory=handoff_store_git_write_lock` for the calls made while the refresh
  already holds the store write lock, as the success path already does.
- Add TTL gating on `integration_is_fresh`, with an explicit opt-out. Signature becomes
  `refresh_bead_store(beads_dir, *, force: bool = False)`.
  - `refresh_bead_wait_store` (`src/sase/axe/run_agent_wait_deps.py:21`) uses the
    default — this is the "N waiters" case the gate exists for.
  - `claim_bead_for_waiting_agent` (`src/sase/bead/claims.py:187`) passes `force=True`:
    it refreshes because a bead was _missing_, and a stale-but-fresh marker must not
    suppress the fetch that would find it.
  - The `bead_store_refresh` chop passes `force=True`: it is the scheduled integrator
    and already owns its own 30s cadence and per-project exponential backoff. TTL-gating
    it would silently halve its effective cadence.
  - The failed-integration cooldown applies **regardless of `force`** — a doomed
    integration is doomed either way.
- Suppression is not success. Return a typed outcome (e.g. `INTEGRATED`,
  `SKIPPED_FRESH`, `SKIPPED_COOLDOWN`, `SKIPPED_NOT_APPLICABLE`) and keep raising
  `_BeadStoreRefreshError` only for genuine failures. Without this the chop would count
  a suppressed refresh as `stores_refreshed` and clear its backoff. Update the chop to
  map the skip outcomes onto `stores_backed_off`.
- Tests (`tests/test_bead/test_sync_remote.py` already imports
  `FAILED_INTEGRATION_MARKER`): a failing integration is attempted once and then
  suppressed within the cooldown window; the cooldown reopens after the window; success
  clears the failure marker; a fresh integration marker suppresses a non-forced refresh
  but not a forced one; the chop reports a suppressed refresh as backed off rather than
  refreshed.

## Phase 3 — Bead reads survive a concurrent store integration

- `src/sase/core/bead_read_facade.py` is the single chokepoint for Rust-backed reads —
  `src/sase/bead/project.py`, `src/sase/doctor/checks_beads.py`,
  `src/sase/scripts/sase_chop_bead_claim_checks.py`, and the ACE bugs tab
  (`src/sase/ace/tui/artifacts_bugs.py:243`) all route through it, and `list`/`show` are
  explicitly excluded from the Rust CLI fast path
  (`src/sase/main/bead_fast_path.py:23`), so they reach it too. Put the retry there.
- Retry only on the _transient_ shape: the `ValueError` raised by event-stream
  validation while an integration is rewriting the worktree. Do not retry
  `Issue not found:` (already translated to `KeyError` by
  `_raise_key_error_for_missing_issue`) or any other deterministic validation error —
  match on the invalid-stream signature, and treat an unmatched `ValueError` as
  immediately fatal. Bound the retry tightly: roughly 3 attempts over ~1s, enough to
  skate over a worktree rewrite and not enough to mask a real problem.
- On persistent failure, raise a dedicated exception carrying the store path and the
  remediation, and render it as a single line at the entry points — the `sase bead`
  handlers and the ACE beads/bugs readers — rather than letting a traceback escape.
  There is no global one-line error renderer in `src/sase/main/entry.py` today, so the
  rendering has to be added where the exception is caught.
- Keep reads lock-free. Taking the store write lock in a read path would let one wedged
  writer hang every `sase bead` invocation and the whole TUI; that is a worse failure
  than a retried read.
- Tests: a stream file containing conflict markers yields the clean single-line error
  after the retries; a file that becomes valid between attempts succeeds; a
  non-transient `ValueError` is not retried; the retry path is exercised for both the
  CLI list and the status-summary readers.

## Phase 4 — Periodic canonical-store chop publishes converged bead state

In `src/sase/scripts/sase_chop_bead_store_refresh.py`:

- After `refresh_bead_store` reports a successful integration, probe for unpushed local
  bead commits with the existing `_unpushed_bead_commit_count` helper
  (`src/sase/bead/sync.py:351`; export it or reuse it via `bead_sync_diagnostics`). When
  the count is non-zero, run `run_managed_sync_worker` (fetch → integrate → push) so the
  primary's waiting-agent claims reach origin within one chop interval. A store with no
  local bead commits stays pull-only and pays no push cost.
- Keep `refresh_bead_wait_store` pull-only. A waiting agent must never block on push
  credentials or network write access — that is why the read-recovery path exists
  separately from the managed worker.
- Feed push failures into the existing per-project backoff (`_next_backoff_entry`) the
  same way refresh failures are, and count them in the chop summary so a persistently
  failing push is visible rather than silent.
- Tests: a canonical store with local bead commits gets pushed after a successful
  integration; a store with no local bead commits stays pull-only and never invokes the
  worker; a push failure extends the backoff without corrupting local state or losing
  the integrated result.

## Phase 5 — Converge and verify the live sase bead store end to end

On the primary machine, after the previous phases land. This phase is deliberately
observational — it changes no production code; its deliverable is evidence.

- The canonical store is the primary workspace's plans sidecar,
  `~/projects/github/sase-org/sase/sase/repos/plans`. Record its starting state first:
  `git rev-list --left-right --count @{upstream}...HEAD`, the last ~40 reflog entries,
  and `bead_sync_diagnostics`.
- If the clone happens to be converged when this phase runs, reproduce a divergence
  rather than skipping: in a scratch clone of the live store, record claims on both
  sides for the same stream so ordinals collide, then integrate. The original epic's
  Phase 6 assumed a standing wedge that had already been cleared by hand; do not repeat
  that.
- Trigger one sync cycle — the `bead_store_refresh` chop or the managed sync worker —
  and confirm convergence: `git rev-list --left-right --count @{upstream}...HEAD`
  reports `0 0`, and the previously colliding stream contains a merged, strictly-ordered
  history in which every line parses.
- Run `sase bead` repeatedly and confirm stable, correct output — no flip-flop between
  snapshots, no tracebacks — with claimed / in_progress / closed statuses matching the
  actual agent states.
- Confirm the reflog accrues no new rebase-abort loops and that `bead_sync_diagnostics`
  reports no divergence, no recovery refs, and no recovery stashes.
- Record the observed before/after state in the phase's completion notes so the epic
  closes with evidence rather than an assertion.

## Residual risk

**Phase 1's core assumption is strong but not proven.** A one-shot merge still trips
`validate_append_only_branch` if either side rewrote base events _relative to the merge
base_. The reasoning that this is not reachable once rebase replay is gone: after a
merge-based integration, the resolved stream is committed as a merge commit, so a later
`merge-base` of the two clones sees byte-identical content on both sides and each side
extends it append-only. The holes in that reasoning are real and worth naming:

- **Transitional histories.** Clones whose history already contains rebase-renumbered
  commits from the current code path can still present a rewritten base until those
  commits are on both sides.
- **Mixed fleets.** A machine still running the old rebase path keeps producing
  rewrites.
- **Manual rebases.** An operator rebasing the store by hand reintroduces the same
  class.

The plan's evidence for the fix direction is a single scratch-clone validation, so treat
"no `sase-core` change needed" as a working hypothesis, not an established fact. Phase
1's fail-closed abort path covers the case safely — the clone is restored exactly, not
wedged mid-merge — and Phase 1 requires a regression test that pins that behavior with
an actionable error. If the case turns out to be reachable in practice, the fix is a
`sase-core` change that reconciles renumbered streams by content identity rather than
ordinal position; it is not a Python-side workaround.

## Considered and deferred: clone-unique event ids

The root cause is that each clone mints per-stream sequential ordinals independently, so
two clones inevitably mint the same `<stream>:<NNNNNN>:<op>:<issue>` id for different
events. Everything above is _reconciliation_ of that collision. The alternative is to
_delete the collision class_: make event ids clone-unique (or assign ordinals only at
reduce time), so two independently recorded events never collide and the append-only
validator never sees a rewrite.

This is less disruptive than it sounds — `merge_bead_event_streams` already renumbers on
every merge, so ordinals are **not** stable identifiers today, and nothing may depend on
them being stable across a merge.

It is deferred because it is a `sase-core` wire-and-storage change with a migration for
every existing stream, whereas the merge fix is local, reversible, and already validated
against the live data. Recording it here so the deferral is a decision rather than an
omission: if Phase 1's residual risk materializes, this is the escalation path, and it
is worth revisiting on its own merits once the store is stable.
