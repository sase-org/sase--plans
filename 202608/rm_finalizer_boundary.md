---
tier: tale
title: Repair the finalizer declaration module boundary
goal:
  Production finalizer consumers use an explicit public declaration API, restoring the
  Symvision gate without changing finalizer behavior.
size: small
proposed_by: bbugyi200.athena.sase-rm.land
bead: sase-rm
create_time: 2026-08-21 13:27:07
status: done
---

- **PARENT:** [202608/task_backlog_closeout.md](task_backlog_closeout.md)
- **BEAD:**
  [sase-rm](https://github.com/sase-org/sase--beads/blob/main/pages/sase-rm/README.md)

# Repair the finalizer declaration module boundary

## Context

Epic `sase-rm` landed broad finalizer/Symvision integration in commit `72f93fb1f`, but
the current combined tree still fails Symvision because `src/sase/finalizers/commit.py`
imports eight private helpers from `src/sase/finalizers/declaration.py`. These are real
production consumers, so the cross-file contract must be explicit rather than hidden
with a whitelist or synthetic static reference.

The affected helpers load the accepted plan/context/submission, normalize and validate
the submission envelope, resolve the artifacts directory, and compute repository
obligation/state identities. Their behavior is already covered by the finalizer
declaration and built-in commit suites; this work changes ownership visibility, not the
wire or persistence format.

## Implementation

1. Promote the eight shared helpers to deliberately named public functions in
   `src/sase/finalizers/declaration.py`: remove the leading underscores, update all
   in-module callers, and export them through `__all__`. Keep signatures, exceptions,
   locking, serialization, digest construction, and validation behavior unchanged.
2. Update `src/sase/finalizers/commit.py` to call/import those public names. Remove the
   function-local private import and do not add a Symvision pragma, epic-symbol entry,
   or `_symvision_static_refs.py` workaround: the production imports are the real
   consumers.
3. Update focused tests only where they intentionally patch or reference the renamed
   declaration API. Add a narrow public-boundary assertion if existing tests do not
   prove that every promoted name is exported and consumed without a private import.

## Verification

- Run focused finalizer declaration and built-in commit tests covering accepted context,
  submission loading/normalization/validation, repository obligation IDs, and state
  digests.
- Run `just symvision` to confirm the eight private-import findings are gone without a
  waiver.
- Run `just check` for the required combined-tree verification. Preserve unrelated user
  changes and report any genuinely independent failure back to the interrupted `sase-rm`
  landing rather than expanding this tale.
