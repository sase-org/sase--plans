---
tier: tale
title: Stop unmanaged projects from tripping agent_publication and artifact retention
goal:
  An enabled project that is not sase-managed no longer produces agent_publication
  check_errors or a doorbell hot loop, and no longer blocks artifact retention.
  Fail-closed behavior for managed-but-unconnected projects and unreadable configs stays
  unchanged.
size: medium
proposed_by: bbugyi200.athena.1d9
proposed_with: claude/opus
create_time: 2026-11-17 06:53:11
status: wip
---

# Plan: Stop Unmanaged Projects From Tripping `agent_publication` And Artifact Retention

## Problem

An enabled project whose repository is **not sase-managed** (no `is_sase_managed: true`
in its `sase.yml`) is a supported configuration. It launches agents, but it never gets
SDD sidecars: `auto_connect_sdd_store` returns `False` for it, and the
`sidecar_auto_sync` connect leg classifies it as `not_applicable`. Two scheduler jobs
still treat such a project as broken, and one of them does so in a hot loop.

Observed on a machine where a fresh `gh` project (`nut`: a one-commit repo with only a
`README.md` and no `sase.yml`) was registered and enabled. Its axe error digest held 44
errors in 12 minutes:

1. **`publication/agent_publication`**, 43 times, about every 15–18 seconds:

   ```
   [agent_publication] gh_bbugyi200__nut: no usable agents publication target
   agent_publication: projects=9 ... errors=1
   ```

   `sase agent sync --check -p nut` reports the project as `disabled` with
   `agents sidecar is disabled or unavailable`. That is a **skip**, not an error. The
   project has zero queued publication rows.

2. **`housekeeping/artifact_run_prune`**, hourly:

   ```
   artifact_run_prune protection source unavailable: nut:beads (sidecars not connected on this machine)
   artifact_run_prune protection source unavailable: nut:plans (sidecars not connected on this machine)
   artifact_run_prune: candidates=7698 selected=0 ... protected=7698 unavailable=2 ... reason=protection_unavailable
   ```

   Retention on that machine is fully blocked, and it stays blocked: the project can
   never connect sidecars, so the fail-closed gap never closes. `artifact_retention`
   (artifact-file retention) uses the same required-sidecar rule through
   `collect_protected_artifact_ids` and is blocked the same way.

The user-side workaround (run `sase repo init` in the project to make it managed, or
disable the project) clears both errors. The code is still wrong, though: any enabled
unmanaged project produces permanent axe noise and silently disables retention.

## Root Causes

### A. `agent_publication` turns a skip into an error and rings its own doorbell

`src/sase/agents_sync/publication_job.py::_run_publication_job`:

- `resolve_sync_targets` returns no `ProjectTarget` for the project. Its `SyncOutcome`
  carries only `skip_reason="agents sidecar is disabled or unavailable"`; the same
  happens for `"agents sidecar has not been created"`. The job counts only
  `outcome.error` while it reads the selection, which is correct.
- The per-project loop then sees `target is None` and **always** does
  `summary.errors += 1`, appends `no usable agents publication target`, and calls
  `_safe_ring(project_key)`.
- The routine's `fs` trigger watches `projects/*/agent_publication/doorbell` (see
  `src/sase/default_config.yml`, the `agent_publication` job). The job touches that
  doorbell while it runs, so the scheduler fires the job again on its next 15-second
  interval. This is a self-sustaining loop: on the observed machine the doorbell mtime
  advanced on every pass, and every pass reported `check_error`.
- When the selection outcome carries a real `error`, the project is counted **twice**:
  once while the selection is read and again in the loop.

### B. Retention requires beads/plans sidecars from projects that can never have them

`src/sase/core/agent_artifact_run_protection.py::_collect_project_sidecar_refs` and
`src/sase/core/artifact_file_protection.py::collect_protected_artifact_ids` add an
unavailable source for every inventory primary whose `beads` or `plans` sidecar root is
missing. `src/sase/core/_sidecar_protection_label.py::unavailable_sidecar_label` only
improves the label (`(sidecars not connected on this machine)`); it never distinguishes
"not connected yet, the connect leg will fix it" from "not sase-managed, nothing to
connect".

Skipping such a project is safe. When an unmanaged project has no materialized SDD store
record on this machine, it has no beads or plans corpus here, and no remote sidecar can
exist for it, because remote creation requires management. Agents launched here on a
managed project auto-connect and write a store record, so this skip never covers a
project whose agents wrote beads. Every other protection source (artifact file index,
`.sase` project files, state refs, hijack holds, links, consumption) still applies to
the project's run dirs.

## Changes

### 1. Shared "SDD sidecars not applicable here" predicate

In `src/sase/sdd/store.py`, next to `sdd_store_connected`, add
`sdd_store_not_applicable(primary_workspace_dir) -> bool`. It returns `True` only when
**both** of these hold:

- `sdd_store_connected(primary)` is `False`: no store record. A record that is present
  but unreadable (`None`) does **not** qualify.
- `repo_project_management(primary)` (from `sase.main._repo_init_config`) returns a
  status with `error is None` and `is_sase_managed is False`. A missing `sase.yml` is a
  valid unmanaged status. Invalid YAML, a non-bool marker, or a layout collision does
  **not** qualify.

Import `sase.main._repo_init_config` lazily inside the function, as `_auto_init.py`
already does, to avoid import cycles. The predicate must never touch the network. Export
it in the module's `__all__` if the module defines one.

Do **not** add a provider-policy check (`in_tree` / `local` / non-remote-backed). That
case was not observed, and its stores may live in-tree. Leave its fail-closed behavior
unchanged.

Optionally, but preferably, make `src/sase/sdd/_auto_init.py::auto_connect_sdd_store`
reuse the same management check, so the connect leg's `not_applicable` and retention's
skip cannot drift apart. Keep its current return values and exceptions exactly.

### 2. Retention protection skips not-applicable projects

In `src/sase/core/_sidecar_protection_label.py`, add a helper alongside
`unavailable_sidecar_label`, for example
`sidecar_protection_not_applicable(primary: RepoRecord) -> bool`. It returns
`bool(primary.path) and sdd_store_not_applicable(primary.path)`. Update the module
docstring to describe the third case: the project is not sase-managed, so it has no
required sidecar sources.

In both scanners:

- `agent_artifact_run_protection.py::_collect_project_sidecar_refs`
- `artifact_file_protection.py::collect_protected_artifact_ids`, inside the
  `_REQUIRED_SIDECAR_ROLES` loop

keep today's order: a live sidecar root is always scanned. Only when the role's live
root is missing, check the helper. If it says not applicable, skip the role without
adding an unavailable source. Otherwise, add `unavailable_sidecar_label(...)` as today.
Compute the predicate at most once per project (for example, lazily on the first missing
role), not once per role.

Opportunistic roles (`research`) are unchanged.

### 3. `agent_publication` treats skipped targets as skips and stops self-ringing

In `src/sase/agents_sync/publication_job.py::_run_publication_job`:

- Keep a `dict[str, SyncOutcome]` of `selection.outcomes` by `project_key`. Continue to
  count each outcome's `error` once, at selection time, as today.
- In the `target is None` branch:
  - Never call `_safe_ring(project_key)`. A missing target cannot be drained, and the
    trigger's `max_quiet: "5m"` backstop already re-runs the job. Queued hints stay in
    the hints file because no claim is taken, so nothing is lost when the target comes
    back.
  - Compute `missing_active` and `missing_parked` as today. Add parked rows to
    `summary.parked`, and also add `len(missing_active)` to `summary.active_remaining`
    so stuck rows stay visible in the summary.
  - Error accounting:
    - If the selection outcome had an `error`, it is already counted. Do not count it
      again or append a second diagnostic.
    - Else, if the project has **active publication rows**, count one error with a
      diagnostic that names the skip reason. For example:
      `<key>: N queued publication request(s) cannot drain: <skip_reason or "no usable agents publication target">`.
      Queued work that cannot drain is a real problem.
    - Else (skip reason and no rows, the observed case), count **no** error and log
      nothing at warning level.
  - Heartbeat `last_outcome`:
    - Use `"target_skipped"` for a skip with no queued rows.
    - Keep `"target_unavailable"` for an error or for a skip with queued rows.
- The diverged-replay marker read in that branch existed only to gate the ring. Drop it
  along with the ring.

Leave the other `_safe_ring` call sites (budget deferral, drain exception, post-drain
pending work) unchanged.

Check `src/sase/agents_sync/publication_doorbell.py`'s heartbeat writer and any
heartbeat consumers (`rg "last_outcome" src/sase`) to confirm that a new
`"target_skipped"` value needs no schema or reader change. If a reader enumerates
outcomes, add it there.

### 4. Docs and config text

Read `sase/memory/docs_writing.md` through `/sase_memory_read` before editing docs.

- `docs/contribute/sdd-storage.md`, in the paragraph near "artifact-run retention needs
  every enabled project's beads and plans sidecars": say that the requirement applies
  only to sase-managed projects. An unconnected project that is definitively not
  sase-managed has no required sidecar sources. An unreadable `sase.yml` or store record
  still fails closed.
- `docs/reference/automation/scheduler-default-routines.md`:
  - In the `sidecar_sync` connect-leg paragraph, next to "An unmanaged or
    non-remote-backed project is rechecked every 5 minutes", note that retention does
    not wait on unmanaged projects.
  - Where the `agent_publication` job is described, if it is, add one sentence: a
    project without a usable agents sidecar is skipped without an error unless it has
    queued publication rows, and the job never rings the doorbell for it.
- `src/sase/default_config.yml`, the `agent_publication` description: update it only if
  the docs routine reference is generated from it. Keep `default_config.yml` and the
  docs consistent.

No SASE memory note states the old behavior, so no memory change is planned.

### 5. Boundary note

The publication job and both protection scanners are Python today. This change only
corrects their existing logic, so no `sase-core` change or `sase-core-revision.txt` bump
is needed. Do not port them to Rust as part of this tale.

## Tests

Follow the existing layouts. Publication job tests live in
`tests/agents_sync/_publication_job_*.py` helper modules and are re-exported by name
from `tests/agents_sync/test_publication_job.py`. Add a new helper module, for example
`_publication_job_targets.py`, and import its tests in the facade.

1. **Predicate** (`tests/sdd_store/`, beside the existing `test_auto_init.py` coverage):
   - Unconnected primary with no `sase.yml` → `True`.
   - `is_sase_managed: false` → `True`.
   - `is_sase_managed: true` → `False`.
   - Invalid YAML, or a non-bool marker → `False`.
   - Materialized store record → `False`, even when unmanaged.
   - Unreadable store record → `False`.
2. **Run-retention protection**
   (`tests/core/test_agent_artifact_run_protection_unconnected.py`):
   - An unmanaged, unconnected project adds no `sources_unavailable` entry.
   - A managed, unconnected project still adds both
     `<project>:<role> (sidecars not connected on this machine)` labels.
   - An unmanaged project that still has a live beads clone is scanned.
   - End to end, `plan_ace_run_retention` with only an unmanaged project is no longer
     `protection_unavailable`.
3. **File-retention protection** (`tests/test_artifact_protection_scan.py`): the same
   three-way matrix for `collect_protected_artifact_ids`.
4. **Publication job** (new helper module). Monkeypatch `resolve_sync_targets` the same
   way the existing helpers do:
   - Skip outcome, no rows: `errors == 0`, the doorbell file is not created or touched,
     `last_outcome == "target_skipped"`, and no diagnostic.
   - Skip outcome with one active row: `errors == 1`, the diagnostic names the skip
     reason, `active_remaining` includes the row, and there is no ring.
   - Error outcome: `errors == 1`, not 2; one diagnostic; no ring.
   - A sibling project with a valid target still drains in the same pass.
5. **Chop status**: `artifact_run_prune` with only an unmanaged project ends with no
   `check_error` from protection. `agent_publication` with only a skipped project emits
   no `check_error`. Use the existing chop tests near `sase_chop_artifact_run_prune` and
   `sase_chop_agent_publication`, if they exist; otherwise the job-level tests above are
   enough.

Before finishing, read `sase/memory/lint_and_test.md` through `/sase_memory_read` and
run the verification it prescribes.

## Out Of Scope

- Making unmanaged projects managed, or auto-running `sase repo init`.
- Non-remote-backed (`in_tree` / `local`) SDD policies in retention.
- The two long-parked `awaiting_done_marker` publication rows in other projects that the
  same summary line shows (`parked=2`). They are not errors and are unrelated.
