---
tier: tale
title: Reduce bead stream sync conflicts
goal: Concurrent bead writers converge through deterministic append-only event-stream
  union, derived store metadata is repaired before it can block work, and rollback
  syncs recover without leaving an unsafe repository.
create_time: 2026-07-20 16:44:50
status: wip
prompt: 202607/prompts/reduce_bead_sync_conflicts.md
---

# Plan: Reduce bead stream sync conflicts

## Context

Phase `sase-8g.7` addresses the high-frequency bead-store rebase and push failures found by the 24-hour audit. Canonical
bead state lives in `beads/events/streams/*.jsonl`; `beads/events/manifest.json` and `beads/issues.jsonl` are derived
metadata/projections. The current implementation already has a useful foundation: Python rebase recovery identifies
supported bead conflicts, reads Git conflict stages, calls Rust-backed stream merge/reduction helpers, rewrites the
derived files, stages them, and continues the rebase transactionally.

Two gaps prevent that foundation from handling the observed production cases. First, generic SDD auto-commit pushes
(including `sase bead work` rollback commits) start the managed sync worker with the repository root in the
bead-directory argument. Conflict resolution consequently computes `.` as the bead prefix and reports real `beads/...`
paths as non-bead conflicts. Second, the event manifest is recounted only when Git reports an explicit conflict. A clean
rebase can combine distinct newly added stream files while both branches make the same numeric manifest increment,
leaving `stream_count` stale even though Git reports a successful merge; strict Rust event-store loading then blocks the
next bead read or mutation.

The fix must preserve the Rust core boundary: event-stream invariants, deterministic union semantics, stream
enumeration, and manifest derivation belong in `sase-core`; Git stage handling, repository path plumbing, transactional
rebase continuation/abort, repair commits, and sync logging remain in the Python repository. `issues.jsonl` must never
be line-unioned: it is regenerated from the reduced canonical streams whenever stream reconciliation changes the store.

## Rust event-store merge and manifest repair

Harden the Rust event helpers so automatic recovery is explicitly limited to append-only journals. Preserve the common
base once, retain every exact branch-only event once, resolve concurrent ordinal/event-id collisions deterministically,
and preserve a stable causal order for the merged stream. Reject deletion or rewriting of base events rather than
treating a non-append-only edit as a safe union. Extend Rust tests for two writers appending different events to the
same stream, exact duplicate appends, colliding generated ordinals, deterministic output independent of repeated
execution, and refusal of non-append-only edits.

Add a Rust-owned event-manifest recount/repair operation that enumerates and validates the actual
`events/streams/*.jsonl` files without trusting the stored count, derives the canonical manifest from those streams, and
atomically rewrites the manifest only when the repairable derived metadata differs. Return a structured result that
distinguishes no-op, repaired, and invalid-stream outcomes. A stale or missing derived count is repairable; malformed
streams, duplicate stream identities, unsupported schema versions, and other canonical-data errors remain hard failures.
Expose the operation through the Python binding and the existing bead conflict facade without duplicating the invariant
logic in Python.

## Transactional sync and rollback recovery

Make the managed SDD integration path carry an actual bead-store directory separately from the repository root. Update
all callers, especially generic SDD post-commit pushes used by rollback, to pass or safely discover the canonical
`beads`/`sdd/beads` directory; a repository without a bead store should continue to abort unsupported conflicts instead
of guessing. Keep path validation confined to the current repository.

Use the Rust recount result after every successful upstream integration, not only after an explicit stream conflict, and
before the repository is declared healthy or pushed. When a clean rebase reveals stale derived metadata, persist the
corrected manifest as part of the integration in a clean, reviewable commit (or the rebase commit being continued),
record a structured repair event, and include the repaired path in the integration outcome. Re-run repository-health
checks after repair so success guarantees an attached branch, no operation markers or unmerged entries, and no dirty
index/worktree.

For explicit conflicts, continue to union only event streams. Rebuild `issues.jsonl` and the manifest from the full
reconciled stream set, stage every generated path, and continue through all rebase conflict rounds. This store-level
reconstruction is the fallback for rollback commits whose only conflicts are canonical streams plus derived bead files.
If a canonical stream is invalid/non-append-only, a non-bead file conflicts, or rebase continuation fails, abort and
verify restoration to the exact starting state; never leave `rebase-merge`, `rebase-apply`, or unmerged index entries
behind.

## Regression coverage

Add fixture-repository tests that exercise the real managed sync transaction rather than only unit-level command
behavior:

- Two diverged clones append different valid events to the same existing stream, both update the derived
  projection/manifest, and sync retains both events, regenerates `issues.jsonl`, recounts the manifest, completes the
  rebase, and pushes with a clean repository.
- Two writers add distinct new streams while Git can merge without an explicit manifest conflict; post-rebase recount
  repairs the otherwise stale count before any strict read or mutation.
- The generic SDD/rollback post-commit path starts from the repository root yet resolves the real bead directory,
  reconstructs supported bead conflicts, and finishes without operation markers or unmerged files.
- Exact duplicates remain single events, projection rows are not line-unioned, repeated repair is a no-op, and
  structured sync logs identify repaired files.
- Corrupt/non-append-only streams and mixed bead/non-bead conflicts abort and restore the starting branch, HEAD, index,
  worktree, and untracked files.

Run focused Rust core and binding tests first, then the Python conflict-resolver, repository-transaction, managed-sync,
and bead rollback suites. Reinstall the editable project so the workspace uses the updated Rust binding, then run the
repository-required `just check`. Do not edit release-managed Rust crate versions.

## Completion

After all targeted regressions and `just check` pass, close only phase bead `sase-8g.7`. Leave parent epic `sase-8g`
open and create no additional beads.
