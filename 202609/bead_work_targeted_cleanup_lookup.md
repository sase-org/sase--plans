---
tier: tale
title: Fix bead-work cleanup TypeError with a targeted owner lookup
goal:
  All three bead-work cleanup fast paths call the post-635c5a31b classify signatures
  through a bounded owner lookup, so epic retries like `sase bead work x7 -Y` complete
  without a TypeError while keeping the no-full-rescan perf contract and the
  live-retry-descendant PRESERVE semantics.
size: medium
proposed_by: bbugyi200.athena.01d.f2
status: done
---

# Fix bead-work cleanup TypeError: give targeted paths a real owner lookup

## Problem

`sase bead work x7 -Y` crashes while retrying an epic whose failed phase owner needs
forced-reuse cleanup:

```
File ".../src/sase/bead/cli_work_cleanup_apply.py", line 212, in _fresh_cleanup_target
    return classify_artifact_record(
        ...
        identity=AgentIdentitySnapshot.current(),
    )
TypeError: classify_artifact_record() got an unexpected keyword argument 'identity'
```

Master is currently broken for every bead-work retry path that reaches these calls, and
the existing test suite is red on master:

- `tests/test_bead/test_cli_work_cleanup_verify.py::test_cleanup_verify_does_not_rescan_history_per_target`
  fails with this exact TypeError.
- `tests/test_bead/test_cli_work_epic_already_running.py` tests fail with the same
  TypeError raised from `cli_work_cleanup_selection.py:287`.
- `mypy` on `src/sase/bead/cli_work_cleanup_apply.py` and
  `src/sase/bead/cli_work_cleanup_selection.py` reports 3 `[call-arg]` errors.

## Root cause

Two commits landed concurrently, three minutes apart, and were each verified only
against a tree that did not contain the other:

1. `635c5a31b` —
   `fix(bead): unblock bead work relaunch when assignee is a stale retry descendant` —
   changed `_classify_artifact_record`'s keyword-only parameter
   `identity: AgentIdentitySnapshot` to `view: _AgentOwnerView` and added a required
   `view:` parameter to `_classify_stale_registry_owner` (both in
   `src/sase/bead/cli_work_cleanup_targets.py`). The new `_resolve_assignee_conflict`
   needs `view.records_by_name_key` to find live records for a retry-descendant
   assignee, and `view.identity` for name-key normalization.
2. `fd2cebac4` — `perf(beads): bound epic-launch history work and prove scale` —
   authored against the pre-635c tree. It renamed the two functions public
   (`classify_artifact_record`, `classify_stale_registry_owner`) and added three new
   call sites written against the OLD signatures:
   - `src/sase/bead/cli_work_cleanup_apply.py:212` — `_fresh_cleanup_target` passes
     `identity=AgentIdentitySnapshot.current()`.
   - `src/sase/bead/cli_work_cleanup_apply.py:231` — `_fresh_cleanup_target` calls
     `classify_stale_registry_owner(...)` without the now-required `view=`.
   - `src/sase/bead/cli_work_cleanup_selection.py:287` —
     `_select_preserved_slots_from_registry` passes `identity=identity`.

The merge was textually clean (the hunks did not overlap), so nothing forced a rebase
conflict, and neither branch's `just check-full` ever saw the combined tree.

## Why the fix is not just renaming the kwarg

All three broken call sites live on deliberately _targeted_ fast paths whose entire
purpose (per `fd2cebac4`) is to avoid the full `scan_agent_artifacts` history scan that
`load_agent_owner_view()` performs. Two structural tests enforce this with
`scan_calls == 0` assertions (`test_cleanup_verify_does_not_rescan_history_per_target`,
`test_all_active_retry_does_not_scan_unrelated_history`). So the fix must NOT call
`load_agent_owner_view()` from these paths.

But `classify_artifact_record` genuinely needs more than `identity` now: when a bead's
assignee is a lineage descendant of the relaunch owner, `_resolve_assignee_conflict`
must look up the assignee's agent records to decide between "descendant is dead →
compatible, proceed destructive" and "descendant is live → PRESERVE". A view built from
only the owner's own record cannot answer that, because the descendant's record lives in
a different artifact directory.

## Design: a narrow owner-lookup seam with a targeted implementation

All work is in Python CLI glue (`src/sase/bead/`); no Rust core boundary is crossed.

### 1. Introduce the lookup seam in `src/sase/bead/cli_work_cleanup_targets.py`

- Define a small structural protocol (name suggestion: `OwnerRecordLookup`; it may stay
  module-private if only used in annotations) with:
  - `identity: AgentIdentitySnapshot` (attribute or property)
  - `records_for_agent_name(name: str) -> tuple[AgentArtifactRecordWire, ...]`
- Add `records_for_agent_name` to `_AgentOwnerView`: normalize `name` with
  `current_owner_agent_name_key(name, self.identity)` and return
  `self.records_by_name_key.get(key, ())`.
- Change `_resolve_assignee_conflict` to call `view.records_for_agent_name(assignee)`
  instead of reaching into `view.records_by_name_key` directly.
- Re-type the `view:` parameter of `classify_artifact_record`,
  `classify_stale_registry_owner`, and `_resolve_assignee_conflict` as the protocol.

### 2. Add a bounded `TargetedOwnerLookup` (public, importable cross-module)

A frozen dataclass holding `identity: AgentIdentitySnapshot`, living in
`cli_work_cleanup_targets.py` (at 522 lines the file has headroom under the 700-line
`toobig` floor; split a sibling module only if a gate complains).

`records_for_agent_name(name)`:

1. Resolve the name to a registry entry WITHOUT a registry rebuild:
   `read_registry(registry_path())`, then try
   `current_owner_agent_name_lookup_candidates(name, self.identity)` against
   `data["entries"]` (same approach as `_registry_entry_without_rebuild` in
   `cli_work_cleanup_apply.py` — prefer extracting one shared public helper so the logic
   is not duplicated in three places).
2. If no entry, no `artifacts_dir`, or a container entry: return `()`.
3. Otherwise scan just that one directory with
   `scan_agent_artifact_dirs(sase_projects_dir(), [artifacts_dir], AgentArtifactScanOptionsWire(include_prompt_step_markers=False))`
   and return `tuple(snapshot.records)`.

This keeps the perf contract: no full history scan, at most one targeted single-dir
read, and only in the rare descendant-assignee case (no assignee, or assignee == owner /
launch name, never reaches the lookup).

### 3. Fix the three call sites

- `cli_work_cleanup_apply.py` / `_fresh_cleanup_target`: build one
  `TargetedOwnerLookup(identity=AgentIdentitySnapshot.current())` and pass it as `view=`
  to BOTH `classify_artifact_record` (replacing the `identity=` kwarg) and
  `classify_stale_registry_owner` (adding the missing kwarg). Everything else
  (membership derivation, the `except ForcedReuseCleanupError: return None` guard, the
  registry branch's own owner lookup) stays as is.
- `cli_work_cleanup_selection.py` / `_select_preserved_slots_from_registry`: replace
  `identity=identity` with `view=TargetedOwnerLookup(identity=identity)` (one instance
  built outside the loop).

### Semantics this restores/improves (intended, not incidental)

- The revalidation path (`prepare_selected_bead_work_force_reuse` →
  `_verify_cleanup_target_still_selected`) can now see a retry descendant that went LIVE
  between preview and wipe: `classify_artifact_record` returns PRESERVE, the target is
  no longer destructive, and the guard raises the existing "no longer eligible for
  destructive cleanup" error instead of wiping under a live retry. Pre-crash `fd2cebac4`
  could never do this (it passed no name index at all).
- A descendant with a registry entry but a dead/missing record remains compatible, and
  foreign/ancestor assignees keep the exact `ForcedReuseCleanupError` message from
  `635c5a31b`. Do not change any user-facing error strings.
- A descendant with NO registry entry resolves to `()` and is treated as dead — same as
  the existing "missing record is compatible" rule.

## Tests

Existing tests that must go green again (currently failing on master):

- `tests/test_bead/test_cli_work_cleanup_verify.py` (all three tests, including the
  `scan_calls == 0` structural guard).
- `tests/test_bead/test_cli_work_epic_already_running.py` (both tests, including its
  `scan_calls == 0` guard).
- `tests/test_bead/test_cli_work_cleanup_assignees.py` (all seven) must stay green.

New tests (suggested home: extend `tests/test_bead/test_cli_work_cleanup_verify.py`
and/or `test_cli_work_cleanup_assignees.py`, reusing `write_bead_agent_meta`,
`claim_registered_name`, and the `_force_reuse_query` pattern):

1. **Crash regression, artifact branch:** owner with a FAILED artifact record and a
   registry claim, `bead_assignees` mapping the bead to a dead retry descendant
   (descendant record written with `done=True`). `select_bead_work_launch` then
   `prepare_selected_bead_work_force_reuse` completes without error and calls the
   (monkeypatched) `wipe_force_reuse_owners` with the owner name. This is the exact user
   scenario that raised the TypeError.
2. **Live descendant blocks the wipe:** same setup, but after the selection is taken,
   write a LIVE descendant record and claim its registry name; then
   `prepare_selected_bead_work_force_reuse` raises `ForcedReuseCleanupError` matching
   "no longer eligible" and the monkeypatched `wipe_force_reuse_owners` is never called
   (`pytest.fail` sentinel). This proves `TargetedOwnerLookup` actually resolves
   assignee records rather than silently returning `()`.
3. **Crash regression, registry branch:** a destructive target whose registry entry has
   no `artifacts_dir` (stale stored owner) plus a dead-descendant assignee; revalidation
   must classify through `classify_stale_registry_owner` without a TypeError.
4. **Fast-path regression:** keep/extend an already-running-style test so
   `_select_preserved_slots_from_registry` classifies with the targeted lookup (the two
   existing tests in `test_cli_work_epic_already_running.py` already cover the crash; an
   extra assignee-variant case is optional if the helpers make seeding the bead assignee
   straightforward).

## Verification

1. `just install` (ephemeral workspace prerequisite), `just fmt`.
2. Targeted first: `.venv/bin/mypy src/sase/bead/` is clean;
   `.venv/bin/pytest tests/test_bead/test_cli_work_cleanup_verify.py tests/test_bead/test_cli_work_epic_already_running.py tests/test_bead/test_cli_work_cleanup_assignees.py`
   passes.
3. `just check`. If it escalates or reports an unusual selection, run `just check-full`
   through the `/sase_monitor` skill (never inline) before declaring done.

## Notes for the reviewer

- This is a fix-forward: master's per-SHA fast CI gate should already be red at
  `fd2cebac4`..HEAD for these mypy/test failures; no separate CI bead is filed because
  this plan IS the fix.
- The root cause (two branches individually green, semantically conflicting merge) is
  the accepted trade-off recorded in the `ci-two-speed-split` decision; no process
  change is proposed here.

<!-- sase:referenced-by:start -->

## Referenced By

| Relation | Artifact                                 | Why                                                                | Uses |
| -------- | ---------------------------------------- | ------------------------------------------------------------------ | ---: |
| cited-by | [agent:bbugyi200.athena.01d.f2--code][1] | prompt reference @plan:202609/bead_work_targeted_cleanup_lookup.md |    1 |
| read-by  | [agent:01d.f2--code][1]                  | Need the approved plan to implement targeted cleanup lookup        |    1 |

[1]:
  https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.01d.f2.md

<!-- sase:referenced-by:end -->
