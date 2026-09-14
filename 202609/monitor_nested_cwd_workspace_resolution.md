---
tier: tale
title: Fix monitor workspace resolution for cwds nested inside managed checkouts
goal:
  "A monitor started from a directory nested inside a managed workspace checkout (e.g.
  an opened external repo clone) keeps its workspace identity, inherits the lane
  workspace claim, and its follow-up relaunches in that workspace instead of degrading
  to workspace #0."
size: medium
proposed_by: bbugyi200.apollo.x
create_time: 2026-09-13 21:02:58
status: wip
---

# Fix Monitor Workspace Resolution For Cwds Nested Inside Managed Checkouts

## Problem

Monitor `89dp49ttb35n` (lane `sase-xe.16.11.7.15.3`) ran `just check` with its cwd
inside an external repo clone opened with `sase repo open` from a numbered workspace — a
path of the shape `<workspace checkout>/sase/repos/external/gh/sase-org/sase-core`. The
monitored command's failure was a pre-existing upstream rustfmt breakage (since fixed on
sase-core master), but the monitor machinery _also_ misbehaved, and that is what this
plan fixes:

1. At monitor start, the member metadata recorded `workspace_num=0` and
   `workspace_dir=<raw nested cwd>` even though the cwd sat inside the lane's own
   claimed numbered workspace. Because the workspace identity was lost, the lane's
   workspace claim was not inherited/transferred to the monitor.
2. When the monitor settled, the follow-up repair also failed to map the nested
   directory back to its workspace, so the follow-up agent launched **degraded in
   workspace #0** with a "do not assume the monitored command's workspace files are
   present" note (`followup_outcome=launched-degraded` on the monitor record). The
   successor lost access to the in-flight uncommitted work and had to redo it from
   scratch in another clone.

Running a repo gate from an opened external/linked repo clone is a standard SASE agent
workflow, so any monitor started this way currently loses its workspace identity.

Evidence: `sase monitor show 89dp49ttb35n` (fields `followup_outcome`,
`followup_degraded_reason`, `cwd`), diagnostics ref
`file:monitor-diagnostic-manifest:89dp49ttb35n`.

## Root Cause

Directory-to-workspace resolution matches registry checkout directories by **exact path
only**, in three cooperating places:

- `src/sase/workspace_provider/lookup.py` — `resolve_workspace_num_for_dir()` compares
  the target against each registry entry's `checkout_dir` with exact normalized-path
  equality. A directory _nested inside_ a managed checkout resolves to `None`.
- `src/sase/monitor/start.py` — `_resolve_lane_start()` computes `cwd_matches_lane` with
  exact `_same_path(request.cwd, workspace_dir)`, so a cwd nested under the lane
  workspace does not count as "in" the lane workspace: the lane's `workspace_num` is not
  reused, claim inheritance (`transfer_from_pid`) is skipped, and
  `_resolve_monitor_workspace_num()` falls through the exact registry lookup to `0`.
  `member_meta["workspace_dir"]` is then overwritten with the raw nested cwd.
- `src/sase/shells/followup.py` — `launch_shell_followup()` repairs a falsy
  `meta_workspace_num` via `resolve_consistent_workspace_pair()`
  (`src/sase/workspace_provider/lookup.py`), which reuses the same exact-match lookup,
  gets `None`, and takes the `meta_pairing_reason` degraded path into workspace #0.

Precedent: `src/sase/workspace_provider/_ownership_identity.py` already implements the
correct semantics for another caller — `_registry_owner_for_path()` picks the **deepest
registry checkout that contains the path** (longest-prefix containment). The monitor
start and follow-up repair paths simply do not use containment.

The workspace registry and this resolution logic are host-side Python only (no sase-core
Rust counterpart), so no Rust/core change is involved.

## Changes

### 1. Add a public containment resolver to `src/sase/workspace_provider/lookup.py`

Add
`resolve_workspace_owner_for_path(primary_workspace_dir, directory, *, config=None, env=None) -> tuple[int, str] | None`
returning `(workspace_num, checkout_dir)` for the registry entry whose `checkout_dir`
contains `directory` (exact match included), with the deepest (longest-prefix) entry
winning, and the same seeded-primary / `WorkspaceStore.resolve(0)` fallback semantics as
`resolve_workspace_num_for_dir`. Use the existing `_normalize_checkout_path`
symlink-resolving normalization for both sides of the containment test.

Refactor `_registry_owner_for_path()` in
`src/sase/workspace_provider/_ownership_identity.py` to delegate to the new function
(preserving its current return types and `normalize_workspace_num` behavior) so the
containment logic has one home. Export the new function from
`src/sase/workspace_provider/__init__.py` alongside the existing lookup helpers.

Keep `resolve_workspace_num_for_dir()`'s exact-match semantics unchanged for existing
callers; its docstring's "never guessed from the directory basename" contract still
holds for the new function — containment consults only registry `checkout_dir` values.

### 2. Repair nested directories in `resolve_consistent_workspace_pair()`

In `src/sase/workspace_provider/lookup.py`, when `workspace_num` is falsy and
`workspace_dir` is neither empty, the primary checkout, nor an exact registry match,
fall back to `resolve_workspace_owner_for_path()`. On a containment hit, return the
**owning pair** — `(owning_checkout_dir, owning_num)` — not the nested directory,
because callers use the returned directory as the successor launch/workspace directory.
A directory nested under the primary checkout repairs to `(primary_workspace_dir, 0)`,
which preserves the invariant from plan `202608/workspace_claim_invariant.md` that
`workspace_num == 0` only ever pairs with the primary checkout. Update the docstring
accordingly.

Knock-on (intended) behavior changes to verify, not suppress:

- `src/sase/shells/followup.py::launch_shell_followup` — a monitor/shell member whose
  metadata carries a nested dir with num `0` now repairs to its owning workspace and
  launches the follow-up there un-degraded (claim transfer path), instead of taking
  `meta_pairing_reason` into workspace #0.
- `src/sase/agent/_family_attach_launch.py` — a family-attach plan naming a nested
  directory with no workspace number now repairs to the owning checkout root instead of
  raising `FamilyAttachError`. This still never launches an unclaimed occupant of a
  numbered checkout: the repaired pair names the registry checkout root.

### 3. Containment-aware monitor start in `src/sase/monitor/start.py`

In `_resolve_lane_start()`:

- Compute `cwd_matches_lane` as "exact match **or** `request.cwd` is nested within the
  lane's recorded `workspace_dir`" (add a `_path_contains`-style helper next to
  `_same_path`, or reuse an existing one such as `path_is_within` used by
  `_ownership_identity`). This restores lane `workspace_num` reuse and claim inheritance
  (`transfer_from_pid`) for monitors started from a subdirectory of their lane workspace
  — the fix that keeps the workspace claimed while the monitor runs.
- In `_resolve_monitor_workspace_num()`, replace the exact
  `_lookup_workspace_num_for_dir` fallback with the containment resolver (exact matches
  still resolve identically through it).
- Record `member_meta["workspace_dir"]` as the **owning checkout root** (the lane's
  `workspace_dir` when `cwd_matches_lane`, else the containment owner's `checkout_dir`)
  instead of the raw `request.cwd`, falling back to the current raw-cwd behavior only
  when the cwd is not inside any managed checkout. The monitored command's raw cwd must
  remain what the monitor record itself stores and displays (the `cwd` field on the
  monitor request/proc row — verify `sase monitor show` output is unchanged in that
  respect).
- Audit the member-meta `workspace_dir` consumers for compatibility with
  root-not-raw-cwd values: `src/sase/monitor/followup.py` (lines ~196/301/434),
  `reconcile.py`, `supervise.py`, `host_completion_state.py`, `resume.py`,
  `proc_adapter.py`. These treat the field as workspace identity / successor launch dir,
  which is exactly what the checkout root provides.

### 4. Tests

- `tests/workspace_provider/test_workspace_lookup.py`:
  - `resolve_workspace_owner_for_path`: path nested in a numbered workspace returns that
    `(num, root)`; the workspace root itself returns `(num, root)`; path nested under
    the primary checkout returns `(0, primary)`; deepest entry wins when one registry
    checkout contains another; unmanaged path returns `None`.
  - `resolve_consistent_workspace_pair`: falsy num + nested dir repairs to the owning
    `(root, num)`; falsy num + dir nested under primary repairs to `(primary, 0)`;
    unmanaged dir still returns `None`; truthy num remains returned unchanged.
- `tests/monitor/` (start): starting a monitor with cwd nested inside the lane's
  workspace records the lane's `workspace_num` and the checkout root in member meta and
  sets `transfer_from_pid`; cwd nested inside a _different_ managed workspace records
  that workspace's number without lane claim inheritance; unmanaged cwd still records
  `0` + raw cwd.
- `tests/monitor/test_monitor_followup.py`: member meta carrying `workspace_num=0` with
  a nested managed dir launches the follow-up in the owning workspace with no degraded
  reason; the existing unmanaged-dir case still degrades to workspace #0 with
  `meta_pairing_reason`.
- `tests/test_dynamic_agent_family_attach_resolution.py`: nested managed dir with no
  number now resolves to the owning pair instead of raising; unmanaged dir still raises
  `FamilyAttachError`.

## Non-Goals

- No change to the monitored `just check` failure itself: the rustfmt breakage was
  introduced upstream in sase-core commit `3fa0a54` and already fixed on sase-core
  master by `d0ec62c`.
- No change to the degraded-fallback ladder for genuinely unmanaged directories
  (fresh-claim → pool → workspace #0); those messages and semantics stay.
- No workspace-claim lifecycle changes beyond restoring the existing inheritance path
  for nested cwds.

## Acceptance Criteria

1. A monitor started with cwd `<workspace checkout>/sase/repos/external/<...>` records
   the owning workspace number and checkout root in its member metadata, inherits the
   lane workspace claim, and its follow-up launches back in that workspace with
   `followup_outcome=launched` and no `followup_degraded_reason`.
2. Monitors started from genuinely unmanaged directories behave exactly as today.
3. `just check` passes.
