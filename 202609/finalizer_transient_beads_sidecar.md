---
tier: tale
title: Avoid false commit-finalizer failures during bead-sidecar sync
goal:
  A transient bead-sidecar sync cannot fail an otherwise valid declared commit, while
  persistently dirty repositories still require an explicit decision.
size: medium
proposed_by: bbugyi200.apollo.1y
create_time: 2026-09-26 09:38:10
status: wip
---

# Avoid false commit-finalizer failures during bead-sidecar sync

## Context

Agent `sase-1aa.5.2` completed its documentation phase, verified the relevant checks,
and closed its bead. Its final declaration was accepted with one dirty repository: the
main checkout. During host commit finalization, `dirty_repos_in_context_order` saw an
additional dirty repository (`repo-0581640e801e`, the beads sidecar classified as
`sibling:beads`) and raised `stale_commit_declaration`. The same sidecar was clean at
run start, absent from the accepted declaration, and clean when inspected afterward. A
bead-sidecar sync/rebase ran during the finalizer window. The run had already spent its
one declaration-recovery turn on the initial missing declaration, so the finalizer
failed and left the completed docs work uncommitted.

This needs a narrow fix for inconsistent or short-lived dirty-state observations. A
persistently dirty repository still requires a fresh declaration; the finalizer must
never commit it under a decision for another repository.

## Implementation

1. In `src/sase/llm_provider/commit_finalizer_state/`, make repository discovery use a
   stable identity for configured SDD sidecar paths. Scan each normalized repository
   path once per collection, then classify a path known to be an SDD sidecar as `sdd`
   even when a configured sibling target also names it. Preserve the existing baseline
   subtraction and dirty-file union behavior. Remove the path-order race that can expose
   the beads sidecar as `sibling:beads` on one scan and `sdd:beads` on another.
2. In `src/sase/finalizers/commit.py` and its declaration/reconciliation helpers, handle
   a newly observed dirty repository by refreshing the dirty state after machine-owned
   reconciliation, before reporting a stale declaration. If a bounded recheck shows the
   extra repository is clean, continue with only the accepted decisions. If it remains
   dirty, fail with the repository's name and paths so the user can distinguish real new
   work from a transient sync. Keep the one-shot declaration-recovery budget and the
   existing fail-closed behavior for genuinely stale accepted obligations.
3. Add focused regression tests around `collect_dirty_state` and the commit finalizer.
   Simulate an SDD sidecar reachable through both sibling and SDD discovery, a
   dirty-then-clean sync window after declaration acceptance, a proven `issues.jsonl`
   reprojection, and a persistently dirty sidecar. Verify the transient case commits
   only the declared main checkout, while persistent or unrelated sidecar edits still
   stop finalization without being committed.

## Verification

- Run the focused dirty-repo discovery, bead reprojection, and finalizer
  staleness/recovery tests.
- Run `just check` per repository guidance. If the pre-existing Symvision
  `--epic-symbol` failures remain, confirm they also reproduce on the clean base and
  report them separately.
