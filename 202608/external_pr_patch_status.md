---
tier: tale
title: Refresh adopted external Patches from pull-request state
goal:
  Adopted external Patches follow their remote pull request into merged or closed
  terminal status without mutating SASE-owned Patch lifecycles.
size: medium
proposed_by: bbugyi200.athena.sase-k2.5
bead: sase-k2.5
create_time: 2026-08-12 12:49:25
status: done
---

- **PROMPT:**
  [prompts/202608/external_pr_patch_status.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/external_pr_patch_status.md)
- **PARENT:**
  [202608/external_mirror_refinement.md](https://github.com/sase-org/sase--plans/blob/main/202608/external_mirror_refinement.md)
- **BEAD:**
  [sase-k2.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-k2/sase-k2.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-k2.5](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-k2.5.md)
- **COMMITS:**
  - [0567ce0](https://github.com/sase-org/sase/commit/0567ce03be8450a991ec296494dbb8d185804d96)
    — feat(external-mirror): refresh adopted external Patches from PR state

# Plan: Refresh adopted external Patches from pull-request state

## Goal

Make the external pull-request mirror revisit Patches it previously adopted with
`PR_ORIGIN: external`, update their recorded status when the remote pull request changes
state, and move newly terminal Patches from the active ProjectSpec into the archive.
Patches owned by SASE's tracked lifecycle must remain untouched.

## Implementation

1. Extend the external-PR reconciliation wire and classifier in `sase-core` with a
   schema-version bump and a `refresh` action. Emit it only for an already-owned Patch
   whose `pr_origin` is `external` and whose local status or active/archive placement
   differs from the remote PR's mapped status and destination; preserve `skip` for
   unchanged external Patches and every SASE-owned or unknown-origin Patch. Add Rust
   unit coverage for open, draft, merged, closed-unmerged, unchanged, and guarded
   ownership cases.
2. Keep the Python wire model and golden classifier in exact lockstep with the Rust
   contract, including the version, action/reason constants, and parity cases that
   exercise refresh and non-refresh decisions through the compiled binding.
3. Teach the Python ProjectSpec importer to apply a refresh under the existing
   active-before-archive locks. Revalidate that the canonical PR is still owned by the
   planned external Patch so a race cannot mutate a SASE-owned Patch, rewrite STATUS
   while preserving external origin and existing descriptive fields, and move terminal
   active records into the archive. Return a distinct `refreshed` outcome rather than
   overloading origin repair.
4. Thread `refreshed` through the mirror report, dry-run and real mutation accounting,
   AXE chop counters, no-op reasoning, and the `sase patch sync-external` output table.
   Treat refreshes as budgeted mutations so budget/deadline exhaustion prevents cursor
   advancement and the daily full scan can retry deferred transitions.
5. Add focused importer and sync regression tests proving an adopted open external Patch
   becomes Submitted/Archived after merge, a closed-unmerged PR maps to Archived,
   dry-run reports without writing, the mutation budget applies, and SASE-owned Patches
   are never refreshed.

## Verification

- In `sase-core`, run its full `just check` gate so Rust classifier and PyO3 binding
  coverage compile and pass together.
- In the SASE workspace, run `just install`, the focused external-PR classifier,
  importer, sync, chop, and CLI tests, then the required `just check` whole-repo lint
  and diff-scoped test gate.
- Inspect both repositories' diffs and status, then close only `sase-k2.5` with a note
  summarizing the verified refresh, guard, budgeting, and archive behavior.
