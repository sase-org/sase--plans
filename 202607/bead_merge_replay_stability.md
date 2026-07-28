---
tier: epic
title: Make bead event-stream merges stable under rebase replay
goal: 'Concurrent bead writers in every plans-sidecar clone converge without human
  intervention: a multi-commit rebase replays to completion, derived bead files stay
  byte-stable across the Rust and Python writers, unpushed bead commits are never
  discarded, and recurring sync failure is surfaced as a health signal instead of
  surfacing as a hand-resolved merge conflict.

  '
phases:
- id: merge
  title: Position-independent event identity and replay-stable stream merge
  depends_on: []
  size: large
  description: 'merge: stop renumbering already-recorded bead events during stream
    merge, give newly created events collision-free identities, and replace the positional
    append-only check with an order-preserving containment check so a merge result
    stays a valid ancestor for the next replayed commit.'
- id: encoding
  title: Byte-identical JSONL encoding across both store writers
  depends_on: []
  size: small
  description: 'encoding: make the Python conflict resolver emit the same UTF-8 JSONL
    bytes as the Rust writer so resolution stops rewriting untouched streams with
    escaped-unicode churn that manufactures spurious merge rejections.'
- id: safety
  title: Never discard unpushed bead commits during workspace preparation
  depends_on: []
  size: small
  description: 'safety: preserve or explicitly rescue local-only bead commits before
    a sidecar clone is cleaned and reset to its upstream branch, so preparing a workspace
    cannot silently destroy unpublished bead history.'
- id: divergence
  title: Eliminate the sticky failure loop that deepens divergence
  depends_on: []
  size: medium
  description: 'divergence: remove the dirty-worktree integration refusals and add
    bounded fetch-rebase-retry on push rejection so a transient sync failure stops
    compounding into a deep multi-commit divergence.'
- id: replay
  title: End-to-end multi-commit replay regression coverage
  depends_on:
  - merge
  - encoding
  size: medium
  description: 'replay: add fixture-repository regression tests that replay a deep
    divergence between real clones through the managed sync worker and assert the
    rebase completes, pushes, and converges to identical bytes.'
- id: health
  title: Surface recurring bead sync failure as a health signal
  depends_on:
  - replay
  size: small
  description: 'health: classify managed-sync log outcomes and report recurring failure
    and deep divergence through existing bead diagnostics so recurrence is detected
    before it reaches a human as a merge conflict.'
create_time: 2026-07-27 06:37:04
status: done
bead_id: sase-9x
---

- **PROMPT:** [202607/prompts/bead_merge_replay_stability.md](prompts/bead_merge_replay_stability.md)

# Plan: Make bead event-stream merges stable under rebase replay

## Context

On 2026-07-27 a plans-sidecar clone diverged badly enough that the merge had to be resolved by hand. The managed bead
sync worker had been failing repeatedly, and its logs identify the mechanism precisely.

### Where the state lives

Canonical bead state is an append-only journal: `beads/events/streams/<issue>.jsonl`, one file per root issue.
`beads/events/manifest.json` and `beads/issues.jsonl` are derived. The plans sidecar is cloned **per workspace** — 20
clones of `sase--plans` exist on this host, plus the primary checkout, and every one of them commits bead events and
pushes to the same `main`. Divergence is the normal case, not an exceptional one, so automatic convergence is the only
thing standing between the user and a manual merge.

The recovery path is: `sase.bead.sync_worker` → `sase.sdd._repository_transaction.integrate_sdd_repository` → fetch +
rebase → on conflict, `sase.sdd._repository_integration` calls `sase.bead.conflict_resolver.resolve_bead_conflicts`,
which reads the three git conflict stages and delegates the semantic merge to `sase-core`'s `merge_bead_event_streams`,
then regenerates the derived files and continues the rebase.

### Evidence

Managed-sync logs under `~/.sase/bead_push_logs/` hold 18,362 records: 4,754 sync attempts and **894 failures**. In the
48 hours around the incident the dominant failure class flipped to unresolved rebases (128 of 272). Every one of those
carries the same core error:

```
Could not apply 17e1c56a... chore(beads): claim sase-9w.6 for sase-9w.6;
semantic bead conflict resolution failed: validation: cannot merge
non-append-only bead event stream sase-9w: theirs rewrote base event 21
```

Aggregate failure classes across all logs: 250 dirty-worktree integration refusals, 210 unresolved rebases, 95 push
rejections, 81 staged-change refusals, 72 credential failures on clones that predated the SSH remote, 57 uncommitted
-change refusals.

### Root cause

Bead event identity encodes **position**. `mutation.rs` mints ids as `{stream_id}:{ordinal:06}:{operation}:{issue_id}`
where the ordinal is the event's index in its stream file.

Position is not stable under merge, so the merge is forced to renumber. In `crates/sase_core/src/bead/events.rs`,
`merge_bead_event_streams` collects the events each branch added after the common base, sorts that union globally by
`(timestamp, operation priority, event_id, serialized)`, then appends each one to a clone of the base while calling
`renumber_event` to rewrite its `event_id` from its new position.

That renumbering is exactly what the same function's own guard forbids. `validate_append_only_branch` requires each
branch's first `len(base)` events to be **byte-identical** to base. The two rules are mutually exclusive across
sequential rebase steps:

1. Rebase step 1 replays local commit `C1` onto upstream `U`. The resolver merges and, because upstream contributed
   events whose timestamps sort before the local ones, the local events are pushed to later ordinals. Their `event_id`
   values change. The step succeeds and `HEAD` becomes `O1`.
2. Rebase step 2 replays `C2`. Git now presents base = the _original_ `C1` tree (original numbering) and theirs = `O1`
   (renumbered). `validate_append_only_branch` compares position by position, finds `event_id` ordinal 21 differs, and
   raises `theirs rewrote base event 21`.

**Resolving step 1 is what guarantees step 2 fails.** The deeper the divergence, the more replay steps, and the more
certain the failure — which is why this surfaced only once nine local commits had piled up.

### Contributing causes

- **Encoding drift between the two writers of the same files.** Rust `serde_json` emits unescaped UTF-8;
  `conflict_resolver._write_resolved_store` calls `json.dumps(...)` at Python's `ensure_ascii=True` default, so it
  re-encodes non-ASCII as `\uXXXX`. 80 of the 363 stream files and 234 lines of `issues.jsonl` currently hold non-ASCII
  and none hold escapes, so every resolution rewrites all 80 files, stages them, and folds them into the rebase commit —
  turning streams the merge never touched into fresh `rewrote base event` rejections on the next step. Every other JSONL
  writer in the repo already passes `ensure_ascii=False`; this call site is the outlier. This was observed directly
  while resolving the incident.
- **Failure is sticky and self-amplifying.** A failed integration leaves the local commits unpushed. The next agent adds
  more, deepening the divergence and adding another replay step. Dirty-worktree refusals block integration entirely for
  a clone, and push rejections are reported as terminal instead of retried after a fetch.
- **Recovery can destroy unpublished history.** Workspace preparation (`sase.axe.runner_workspace.prepare_workspace`)
  cleans the sidecar and checks out `origin/main`. During the incident this discarded nine local bead commits; they were
  recoverable only because their hashes were still in the reflog. One clone on this host is currently
  `ahead=1 behind=15` and is one preparation away from losing that commit.
- **Nothing reports the pattern.** 894 logged failures accumulated silently; the signal reached the user as a merge
  conflict.

### Boundary and constraints

Event-stream invariants, identity minting, merge semantics, stream enumeration, and manifest derivation belong in
`sase-core` per the Rust core backend boundary. Git stage handling, path plumbing, transactional rebase
continuation/abort, and sync logging stay in this repo. `beads/issues.jsonl` is never line-merged — it is regenerated by
reduction from the canonical streams. A prior tale, `202607/reduce_bead_sync_conflicts.md`, built this recovery
foundation; this epic fixes the merge-identity defect that foundation still contains.

The `sase-core` repo must be opened with the `/sase_repo` skill; do not clone or web-fetch it.

## Position-independent event identity and replay-stable stream merge

Make a merge result a valid ancestor for the next replayed commit. Two changes in `crates/sase_core/src/bead/events.rs`
and `crates/sase_core/src/bead/mutation.rs` are jointly required — either alone leaves the contradiction in place.

Stop rewriting the identity of any event that has already been recorded. `merge_bead_event_streams` must preserve every
input event's `event_id` verbatim and reach a deterministic total order without `renumber_event`. Retire that helper
from the merge path, or reduce it to minting ids for genuinely new events only.

Give newly created events an identity that cannot collide when two clones mint concurrently.
`BeadEventRecordWire:: validate` constrains `event_id` only to be non-empty and `BeadEventStreamWire::validate` imposes
no ordinal or uniqueness rule, so the encoding is free and existing ids stay valid — prefer a scheme that keeps the
current shape readable while appending a deterministic disambiguator derived from event content or actor, and avoid a
schema-version bump and a rewrite of the 363 existing streams unless the chosen encoding genuinely requires one. Justify
that choice in the phase's own plan.

Replace `validate_append_only_branch`'s positional prefix comparison with an order-preserving containment check: every
base event must still be present in the branch, in the same relative order, with additions allowed anywhere. Genuine
corruption — a deleted or edited base event — must still be rejected with an equally specific message. This is what lets
step 2 accept an `O1` in which upstream events interleave ahead of local ones.

The merged order stays `(timestamp, operation priority, event_id)` over the union of both branches' additions, which
with stable identities makes the merge **commutative, associative, and idempotent**, so every clone converges on
identical bytes regardless of merge direction or how many times a stream is re-merged. Prove these properties in Rust
tests, and add the case this epic exists for: a base stream, N sequentially replayed local commits, and an upstream
branch whose events interleave by timestamp, asserting every step validates and the final stream contains every event
exactly once. Extend `crates/sase_core/tests/bead_event_parity.rs`, whose current assertion on `rewrote base event 1`
encodes the behavior being replaced.

Expose any signature change through the Python binding and `sase.core.bead_conflict_facade` without duplicating
invariant logic in Python, and land the corresponding `sase-core-rs` version requirement in this repo.

## Byte-identical JSONL encoding across both store writers

In `src/sase/bead/conflict_resolver.py`, `_write_resolved_store` serializes stream events and `issues.jsonl` rows with
`json.dumps(..., separators=(",", ":"))` and no `ensure_ascii` argument, so it writes `\uXXXX` escapes where the Rust
writer wrote UTF-8. Pass `ensure_ascii=False` at both call sites so the resolver reproduces the Rust encoding exactly.

Confirm the fix at the level that matters: the function already skips rewriting an unconflicted stream when its
serialization matches the file on disk, and that comparison is what the escaping defeats. After the fix, resolving a
conflict in one stream must leave every other stream file unmodified. Add a regression test that puts non-ASCII content
in an unconflicted stream, resolves a conflict elsewhere, and asserts that file's bytes are untouched and that it is not
staged.

Audit the rest of `src/sase/bead/` and `src/sase/sdd/` for any other writer of canonical or derived bead JSONL with the
same defaulted-`ensure_ascii` bug and fix those the same way.

## Never discard unpushed bead commits during workspace preparation

`prepare_workspace` cleans a workspace and checks out the update target; for a plans sidecar that is `origin/main`, and
local-only commits are lost. `run_sase_hg_clean` rescues uncommitted changes to a backup diff, but
committed-and-unpushed work has no such protection.

Before a sidecar clone is reset, detect local-only commits that touch canonical bead state —
`sase.bead.sync.bead_sync_diagnostics` and `_unpushed_bead_commit_count` already compute exactly this against
`@{upstream}`. When any exist, attempt a managed sync to publish them first, and if that cannot succeed, retain them at
a durable recovery ref and report where they went, rather than resetting over them. Reuse the existing recovery-ref
mechanism in `sase.sdd._repository_recovery_reaper` rather than inventing a second one, and make sure whatever is
retained is still reachable after the reaper runs.

Cover the case that occurred: a sidecar with unpushed bead commits and a remote that has moved ahead, prepared for a new
agent, must not lose those commits. The clone on this host currently sitting at `ahead=1 behind=15` is a live instance.

## Eliminate the sticky failure loop that deepens divergence

Divergence deepens because integration is refused or abandoned, never because merging is impossible. Two classes account
for most of it.

**Dirty-worktree refusals (388 across all classes).** `require_sdd_repository_health` refuses to integrate a clone with
tracked worktree changes, staged changes, or uncommitted changes. Establish what leaves a sidecar dirty at sync time —
including whether a derived file such as `issues.jsonl` or a rebuilt `beads.db` is being written outside the store write
lock — and fix the source. Where a dirty state is legitimately transient, the worker should serialize behind the store
write lock and retry rather than fail the sync outright. Do not weaken the health check into ignoring genuine
uncommitted user work.

**Push rejections (95).** `_run_locked_sync` treats a rejected push as terminal, so a clone that lost a race stays
diverged until some later sync happens to win. Add a bounded fetch-rebase-retry loop around the push, reusing the same
integration and conflict-resolution path so a retry is subject to the identical semantic merge, with a small attempt cap
and a clear terminal error when exhausted.

Also review the failed-integration cooldown marker in `sase.sdd._repository_recovery_markers`: confirm that a clone
holding unpushed bead commits cannot be parked in cooldown long enough for divergence to grow, and shorten or bypass it
for that case.

## End-to-end multi-commit replay regression coverage

Unit coverage of the Rust merge is not sufficient — the incident lived in the interaction between merge output and git's
per-step conflict staging. Add fixture-repository tests that drive the real managed sync worker.

The primary case reproduces the incident: two clones of one fixture sidecar, one accumulating several bead commits
across multiple streams while the other pushes interleaved events to the same streams, then a single
`run_managed_sync_worker` call on the diverged clone. Assert the rebase completes without human intervention, the push
succeeds, no `rebase-merge`/`rebase-apply` directory or unmerged index entry remains, every event from both sides is
present exactly once, and `issues.jsonl` and `manifest.json` match a fresh reduction of the merged streams.

Add convergence and encoding assertions: two clones that integrate the same divergence in opposite directions end at
byte-identical stream files, and a resolution touching one stream leaves unrelated stream files byte-unchanged. Add the
failure direction too — a genuinely corrupted stream must still abort the rebase and restore the clone to its exact
starting state.

Keep these tests fast enough for `just test` and deterministic on timestamps, which the merge order depends on.

## Surface recurring bead sync failure as a health signal

894 failures accumulated in `~/.sase/bead_push_logs/` without ever being reported. Make recurrence visible.

Extend `sase.bead.sync.bead_sync_diagnostics` — which already warns about divergence, mid-rebase state, and unpushed
bead commits — to also classify recent managed-sync log outcomes: repeated failures for this clone, the dominant error
class, and the count of consecutive failed integrations. `_latest_bead_sync_log` already locates the log directory.

Surface it where a human or agent will see it before divergence becomes a hand-merge, following whatever the repo
already does for store health warnings rather than adding a new notification channel. Keep the log read bounded and
best-effort: diagnostics must stay cheap and must never raise into a bead command.

Verify the check fires on a fixture log containing the incident's own failure signature, and stays quiet for a healthy
clone.

## Validation

Run `just install` before anything else — this workspace's virtualenv may be stale — then `just check` for all changes
in this repo. Changes in `sase-core` must pass that repo's own test suite, and the `sase-core-rs` version requirement
here must be updated in the same epic.

The epic is done when a clone carrying a deep multi-commit divergence against a moved-ahead `main` integrates and pushes
through `run_managed_sync_worker` alone, with no manual conflict resolution.
