---
tier: tale
title: Make agent-session forced reuse wipe every owner of the session reservation
goal:
  Forcing reuse of an agent-session name succeeds in one pass, or fails before deleting
  anything and names the file that blocked it.
size: medium
decisions:
  older_generations:
    ask: Should forced reuse also wipe older or unselected generations of the session?
    choices:
      wipe:
        Wipe every owner that still carries the session name, so the launch succeeds in
        one pass
      refuse:
        Refuse before deleting anything and name the extra owners, leaving cleanup to
        the user
    default: wipe
    why:
      Any such owner blocks reuse of the name anyway, and the user asked for forced
      reuse.
    answer: wipe
proposed_by: bbugyi200.athena.1dy
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 09:53:33
status: wip
---

# Make agent-session forced reuse wipe every owner of the session reservation

## Problem

Relaunching an agent-session root with forced reuse (for example
`%id(!land, clan=sase-1tb, bead=sase-1tb)`, or the bead-work relaunch path) sometimes
fails with:

```text
Agent name reuse failed: forced reuse cleanup left agent session 'sase-1tb.land'
reservations after rebuild: sase-1tb.land
```

This has happened 4 times in about 28 hours (`sase-1t3.5`, `sase-1t3.1`,
`sase-1tb.land`, `sase-1t3.10.1.3`; see `~/.sase/logs/launch_failures.log`). Each
failure is also **destructive and partial**. In the `sase-1tb.land` and
`sase-1t3.10.1.3` cases:

- the `--code` successor, the plan gate, and the `#fork:` monitor-continuation members
  were deleted;
- the session root (`<session>--plan`) artifact and its dismissed bundles survived;
- the surviving root re-derived the session container reservation, so the residual check
  raised.

A retry then succeeds, because the root is now the only owner left and becomes the
"newest generation" (`sase-1t3.10.1.3` was relaunched about 4 minutes later). A dry run
against the current `sase-1tb.land` state confirms that a retry would now wipe the root
artifact plus 5 bundles cleanly.

## Root cause

`src/sase/agent/names/_forced_reuse.py::_wipe_agent_sessions_for_forced_reuse` takes its
target set from one source and checks its result against another:

- **What it deletes:** the members returned by `find_agent_session(name)`. That is the
  artifact index's `session_latest` selection in sase-core
  (`agent_scan/index/group_selection.rs::select_latest_session`). It picks the newest
  parentless hit as root, then follows `parent_timestamp` descendants. Without a
  parentless hit it falls back to the candidate set.
- **What re-derives the reservation:** the registry rebuild in
  `_registry_scan_collectors.py`. `collect_artifact_entries` and
  `collect_dismissed_bundle_entries` call `add_owner_agent_session` for _every_ artifact
  `agent_meta.json` and every dismissed bundle whose `agent_session_from_payload(...)`
  names the session.

When these two views disagree, the generation wipe deletes part of the session, and the
remaining owner re-derives `<session>` in the rebuild. The residual check then raises
after the deletion has already happened. Possible causes of disagreement include:

- the index row for the root lagging (for example around a dismissal);
- an older generation still on disk;
- a member whose parent chain doesn't reach the selected root;
- an owner that exists only as a dismissed bundle.

The cleanup deleted the evidence of which of these hit the root today, so the fix must
not depend on the index view being right. The fix: build the target set from the same
derivation the registry uses, and wipe it in one batch.

## Changes

### 1. Enumerate every owner that derives a session reservation

Add a helper that returns, for a set of session names, every artifact and
dismissed-bundle owner whose payload derives that session's container reservation.

- Match **exactly** as the registry scan does:
  - `agent_session_from_payload` (which already ignores `agent_session_parallel`
    payloads);
  - localized through `localize_payload_name`;
  - compared against `current_owner_agent_name_lookup_candidates(name)`.
- Prefer extending the existing wipe catalog over adding a second full filesystem scan:
  - Add a `sessions` field to `ArtifactRecord` and `BundleRecord` in
    `src/sase/agent/names/_wipe_scan.py`, populated at scan time.
  - Let callers pass a preloaded `WipeCatalog` into `wipe_agent_names_for_reuse`
    (optional keyword, default unchanged), so session cleanup scans the filesystem once.
- Keep new code in a small module (for example `_session_owners.py`) if
  `_forced_reuse.py` would otherwise grow past a comfortable size.
- Return owner mappings in the shape `wipe_agent_names_for_reuse` already accepts:
  - `name`: the owner's own concrete name from `owner_identity_names`, falling back to
    the session name when the payload has no own name;
  - `artifacts_dir` and/or `bundle_path`;
  - `raw_suffix`;
  - no `container_kind`, so the owner is wiped rather than skipped.

### 2. Wipe the full closure in one batch, before checking residuals

In `_wipe_agent_sessions_for_forced_reuse`:

- Keep `find_agent_session` for the newest-generation member names: the residual check
  and error messages use them, and they preserve today's behavior when the index is
  right.
- Union those member names with the session-owner mappings from step 1, deduplicated by
  resolved artifact dir and bundle path. Any owner that the generation selection didn't
  cover is added to the **same** batch (`older_generations = wipe`).
- Run one `_wipe_names_for_reuse_batch` call over the union. This preserves the existing
  ordering guarantees:
  - `execute_wipe_plan` stops live processes and returns before deleting anything if a
    stop fails;
  - the session-root guard (`_wipe_guard.session_root_removal_refusal`) still sees the
    root as a requested target.
- Stale-container handling stays as is, with one change. A session with no members from
  `find_agent_session` but with derived owners in the catalog is **not** stale; its
  owners go into the batch.
- Do not touch owners of other sessions:
  - names that merely share a prefix, such as a nested session `epic.land--3` or a
    sibling `epic.land2`, must not match;
  - unrelated agents and any enclosing clan must not be touched.

> [!decision] older_generations = refuse Compute the same union, but if it contains
> owners outside the `find_agent_session` generation, raise `ForcedReuseCleanupError`
> **before any deletion**. The error lists those owners (name plus artifact dir or
> bundle path) and says to dismiss or wipe them, then retry. The newest generation is
> only wiped when no such owners exist.

### 3. Make residual failures self-explanatory

When the post-rebuild residual check still finds `<session>` or a member reserved, build
the error message from the registry entry for each residual name. Include:

- `source` (`artifact` or `dismissed_bundle`);
- `artifacts_dir` and/or `bundle_path`;
- the number of `collision_owners`.

The goal is that the next failure names the file that kept the reservation alive,
instead of only `sase-1tb.land`. Keep the `ForcedReuseCleanupError` type, and keep the
existing message prefix so the launch-failure log stays greppable.

### 4. Update docstrings

The docstrings that promise "replaces its newest generation" go stale with this change.
Update them to say forced reuse replaces every owner of the session reservation, with
the newest generation found through the index and any other owners found through the
registry's own derivation:

- the module docstring of `_forced_reuse.py`;
- `wipe_force_reuse_owner`;
- `wipe_names_for_forced_reuse` in `src/sase/agent/launch_validation.py`.

### Scope notes

- **Rust boundary:** the forced-reuse cleanup policy and wipe pipeline live in Python
  today (`src/sase/agent/names/`). This fix stays there and needs no sase-core change:
  the Rust `session_latest` selector is used only to name members, and its semantics are
  unchanged. Porting the wipe pipeline into sase-core is out of scope.
- **Bead-work relaunch:** this path (`src/sase/bead/cli_work_cleanup_apply.py`) uses the
  same `wipe_force_reuse_owners` primitive, so it picks up the fix automatically. Its
  existing tests must keep passing.

## Tests

Add the regression tests next to the existing ones in
`tests/test_agent_names_forced_reuse.py` and/or
`tests/test_agent_name_wipe_sessions.py`. Seed a real registry with the existing helpers
(`tests/_agent_name_wipe_helpers.py`: `make_wipe_artifact`, `make_wipe_bundle`,
day-sharded layout) under a patched `Path.home`.

1. **Index view omits the root (the observed failure).**
   - Seed session `epic.land` with:
     - root `--plan` (parentless, `agent_session_role: root`, `plan_chain_root: true`);
     - `--gate` and `--code` (parent root);
     - `--mon` (parent `--code`);
     - forks `--1` and `--2` (role `root`, parent root);
     - `--mon-0` (parent `--1`);
     - a dismissed bundle plus a `__c0` child bundle for the root.
   - Patch `sase.agent.names.find_agent_session` to return a generation without the
     root.
   - Assert that `wipe_force_reuse_owners(["epic.land"], allow_container_skip=False)`
     succeeds, every artifact and bundle is gone, and no `epic.land*` name stays
     reserved.
   - Add a variant that runs without the patch, to pin the normal path.
2. **Two generations.** An older parentless root generation and a newer one, both
   `agent_session: epic.land`. Forced reuse succeeds and removes both (under `refuse`:
   raises before deleting anything, and both generations are still on disk).
3. **Bundle-only owner.** A dismissed bundle carrying `agent_session: epic.land` whose
   artifact dir is already gone. It is wiped, and the container is released.
4. **Isolation.** These must all survive the cleanup:
   - a nested session `epic.land--3` (its own `agent_session`);
   - a sibling `epic.land2` session;
   - an unrelated agent;
   - the enclosing clan.
5. **Diagnostics.** Force a residual (for example, patch the batch wipe to skip one
   owner) and assert that the error names that owner's artifact dir or bundle path.
6. Existing forced-reuse, session-wipe, session-root-guard, and bead-work cleanup tests
   pass unchanged:
   - `tests/test_agent_names_forced_reuse.py`
   - `tests/test_agent_name_wipe_sessions.py`
   - `tests/test_force_reuse_launch_seam_registry.py`
   - `tests/test_bead/test_cli_work_cleanup_*.py`

## Verification

- Run the targeted tests above, then the repo's standard agent check through the tool
  runner. Follow the `lint_and_test` reference memory.
- Do not run any destructive reproduction against the real `~/.sase` state. Use
  `tmp_path` homes only.
