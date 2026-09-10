---
tier: tale
title: Resolve archived task-status-hook references
goal:
  Exact Pomodoro links to collected task archives resolve without false warnings while
  archives remain read-only and outside active synchronization.
size: medium
proposed_by: bbugyi200.athena.0f9
create_time: 2026-09-09 20:00:25
status: wip
---

# Resolve archived task-status-hook references

## Goal

Stop `bob task-status-hooks` from emitting false unresolved-target warnings for Pomodoro
block links that `bob move-done-tasks` has correctly rewritten to an exact `done/...`
archive path, while keeping archived notes read-only and outside every active
status/dependency synchronization rule.

## Root cause and constraints

- The current and previous daily ledgers contain
  `[[done/dev_done#^lower-athena-disk-use]]`, and the target task exists in
  `done/dev_done.md`.
- `task_status_hooks::markdown_files` intentionally omits `done/`, then builds both
  `NoteIndex` and `task_blocks` exclusively from that active scan. Current
  `all_references` and previous `recent_references` therefore cannot resolve a link that
  the collection workflow deliberately repaired to its archive location.
- The current retired link and the previous live historical link take separate
  resolution paths, which accounts for the two warnings.
- The archive exclusion is still required: archived task markers must never be promoted,
  cleared, blocked, or used as active dependency-graph nodes, and the existing
  documented warning for a dependency transclusion into `done/` must remain intact.
- Resolution must stay path-safe and deterministic. Only an explicit, normalized
  vault-relative target beneath the top-level `done/` directory may opt into archive
  lookup; no basename guessing, parent traversal, hidden directory scan, or broad
  reintroduction of the archive into the active vault scan is acceptable.

## Implementation

1. Separate active synchronization data from read-only Pomodoro-reference data in
   `src/native/task_status_hooks.rs`.
   - Preserve the existing filtered `files`, active `NoteIndex`, active `task_blocks`,
     dependency-edge construction, transition loop, and `scanned_files` semantics.
   - Collect the exact note targets needed by the current daily's `all_references` and
     the previous daily's non-retired `recent_references`. For explicit targets
     normalized beneath `done/`, read only the named Markdown file and parse its Tasks
     lines with the same Tasks settings used by the active scan.
   - Build a supplemental, read-only reference catalog from those archive notes. Do not
     append them to `files` or expose them to dependency-state, scheduled-state,
     status-write, or dependency-edge processing.

2. Route direct current-ledger and previous-daily reference resolution through one
   shared resolver that can consult the active catalog first and the narrowly loaded
   archive catalog second.
   - Retain existing exact-path/basename behavior for active notes.
   - Allow an exact archived target/block pair to supply statuses for completed link
     retirement, canceled-reference cleanup, duplicate-status diagnostics, and read-only
     recent-activity identity normalization.
   - Keep missing archive files, missing task blocks, duplicate/conflicting block IDs,
     and ordinary ambiguous targets on the existing warning path with truthful
     diagnostics.
   - Keep dependency transclusions on the active-only resolver/map so an edge into
     `done/` remains unresolved and can never propagate rank from or to an archived
     task.

3. Add regression coverage in `tests/cli.rs` for the real post-collection lifecycle and
   the isolation boundary.
   - Model a current daily with an already-struck exact archive link, a previous daily
     with the corresponding non-retired link, and a completed task in a nested `done/`
     note. Assert human stderr has no warning, JSON has no unresolved reference, the run
     is idempotent, and both archive and previous daily remain byte-for-byte unchanged.
   - Cover a live current Pomodoro link whose completed target has already moved to
     `done/`; assert it is still retired/canonicalized without modifying the archive.
     Include canceled terminal behavior if the shared status path handles both DONE and
     CANCELLED archive entries.
   - Assert an explicit dependency transclusion into the same archive is still reported
     unresolved, an archived status is never changed, and the archive does not inflate
     the active `scanned_files` count.
   - Assert a nonexistent or invalid `done/...` target still warns, preventing the
     archive exception from becoming warning suppression.

4. Update `docs/task-status-hooks.md` to document the two scopes precisely: active
   scanning still skips `done/`, while exact archive targets in current or previous
   Pomodoro ledgers may be loaded read-only so collection-repaired terminal references
   can be normalized without false warnings. Preserve the documented unresolved behavior
   for dependency links into the archive.

## Acceptance criteria

- The supplied `bob task-status-hooks --dry-run --format json` scenario reports no
  unresolved reference for `done/dev_done#^lower-athena-disk-use`, and the normal
  command no longer emits either of the two shown warnings.
- Completed/canceled archive references retain the structural behavior they had before
  collection moved their target, including retirement/canonicalization or
  canceled-subtree removal where applicable.
- No file below `done/` is ever planned for a status write, archived tasks do not
  participate in active dependency propagation or derived Blocked state, and active
  scan/report counts retain their meaning.
- Genuine missing, ambiguous, conflicting, and dependency-to-archive cases continue to
  produce actionable warnings.
- Current and previous daily structural/read-only guarantees remain intact and repeated
  successful runs are idempotent.

## Validation

1. Run focused task-status hook tests, including the new archived-reference cases:
   `cargo test --test cli task_status_hooks`.
2. Run the complete project checks: `just all`.
3. Run the built command read-only against the reported vault state with
   `bob task-status-hooks --dry-run --format json` (or the equivalent locally built
   binary), confirm both archived-target warnings are absent, and confirm the JSON has
   no matching `unresolved_references` entry and no planned changes.
