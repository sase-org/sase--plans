---
tier: tale
title: Make repository opens non-destructive and reject discarded finalizer work
goal:
  Repository opens preserve agent work, and finalization cannot mistake discarded
  changes or another agent's HEAD for a successful commit.
size: medium
proposed_by: bbugyi200.athena.sase-ly
bead: sase-ly
create_time: 2026-08-14 10:43:41
status: wip
---

- **BEAD:**
  [sase-ly](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ly/README.md)

# Make repository opens non-destructive and reject discarded finalizer work

## Objective

Complete task bead `sase-ly` by ensuring that `sase repo open` never destroys an
existing checkout's tracked or untracked changes, and by preventing the commit finalizer
from treating vanished dirty work as a successful commit.

## Context and design

Configured primary, linked, and sidecar repository opens currently share the legacy
workspace-open preparation path. That path invokes the general agent-runner workspace
preparation routine, which stashes/cleans the tree, checks out the default revision, and
synchronizes it. The clean can permanently delete an untracked file before the
repository is recorded as opened, after which the commit finalizer sees a clean tree and
can mistake an unrelated existing `HEAD` for the agent's commit.

Keep clone/materialization, Git exclude setup, SDD initialization, open-marker
recording, audit logging, and path output intact. Make the normal `sase repo open` path
skip destructive clean/checkout/sync preparation for an already materialized checkout.
Retain the deprecated `sase workspace open` compatibility behavior behind an explicit
internal preparation choice so unrelated runner setup semantics do not change.

Independently harden commit finalization at its existing per-pass fingerprint boundary.
If a repository was dirty before a finalizer pass and becomes clean after the pass,
require evidence that its `HEAD` advanced. When current agent identity is available,
require one of the newly reachable commits to carry that agent lane's `SASE_AGENT`
provenance. If the dirty paths merely disappear, fail immediately with a distinct result
reason and an error naming the affected repository and paths rather than emitting
`clean_after_pass`.

## Implementation

1. Add an internal non-destructive mode to the shared opened-checkout helper and have
   `sase repo open` select it while the deprecated workspace alias retains preparation.
   Update command help and focused repo-open tests to state and prove that dirty tracked
   and untracked files survive a reopen, the checkout is still recorded, and the
   destructive runner preparation helper is not invoked.
2. Extend commit-finalizer Git helpers to compare the pre-pass repository/HEAD
   fingerprint with post-pass state and, when possible, validate commit provenance
   against the current agent lane. Return structured evidence for repositories whose
   dirty paths vanished without an attributable commit.
3. Make the finalizer persist a failed result with a dedicated discard/reset reason and
   raise its normal finalizer error when that evidence is present. Add regression tests
   for deletion/reset without a commit, a foreign-agent commit, and a valid
   current-agent commit; adjust existing success fixtures so they create real commits
   instead of simulating success by deleting dirty files.
4. Review the separate concurrent-research same-output-path race called out by the bead.
   Since it is independent of repository-open cleanup, route it through `/sase_new_task`
   with `sase-ly` as causal context rather than expanding this fix.

## Verification

- Run focused repo-open and commit-finalizer regression tests during development.
- Run `just install` because the workspace may be stale, followed by `just check` as the
  required repository-wide lint and diff-scoped test gate.
- Re-run focused tests if formatting or lint fixes touch the safety logic, inspect the
  final diff and worktree status, then close `sase-ly` with a note that names the
  non-destructive reopen cases, discarded-work failure case, valid attributed-commit
  case, and the successful verification commands.
