---
tier: tale
title: Refresh the finalizer completion snapshot
goal:
  The checked-in CLI completion contract includes the complete finalizer command
  surface.
size: xsmall
proposed_by: bbugyi200.athena.sase-rn.land
bead: sase-rn
create_time: 2026-08-20 19:49:30
status: wip
---

- **PARENT:** [202608/pluggable_finalizers.md](pluggable_finalizers.md)
- **BEAD:**
  [sase-rn](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rn/README.md)

# Refresh the finalizer completion snapshot

## Goal

Bring the checked-in CLI completion contract into sync with the completed pluggable
finalizer command surface.

## Context

Epic `sase-rn` phase 5 refreshed `tests/completion/snapshots/cli_spec.json` after adding
`sase final list`, `show`, and `doctor`. Phase 4 subsequently added `sase final context`
and `sase final submit`, so the final combined tree now fails
`tests/completion/test_snapshot.py::{test_checked_in_snapshot_has_no_drift,test_current_structural_view_matches_checked_in_snapshot}`.
The structural diff contains only the two missing command paths and the resulting
`final` description digest.

## Implementation

1. Regenerate `tests/completion/snapshots/cli_spec.json` with the repository's
   `just sync-completion-spec` command.
2. Review the diff and confirm it adds only `final context`, `final submit`, and the
   corresponding `final` group digest change.
3. Run `tests/completion/test_snapshot.py` and the focused finalizer suites to confirm
   the combined command surface remains green.
4. Run the repository-required verification appropriate to the one generated snapshot
   change, recording any unrelated pre-existing gate failures rather than expanding this
   repair.

## Acceptance criteria

- Both completion snapshot drift tests pass.
- The regenerated snapshot accurately describes `sase final context` and
  `sase final submit` without unrelated structural changes.
- Focused finalizer tests remain green.
