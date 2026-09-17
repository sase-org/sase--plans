---
tier: tale
title: Stop machine artifact-link bead writes from dirtying the primary beads clone
goal:
  The artifact_link_backfill machine lane publishes derived bead links through a hidden
  host-owned beads clone (or refuses cleanly before writing), so the primary checkout's
  beads sidecar clone stays clean, sidecar auto-sync keeps fast-forwarding it, and
  agents waiting on closed beads stop stalling.
size: medium
proposed_by: bbugyi200.athena.0m7
create_time: 2026-09-17 08:42:32
status: wip
---

# Stop Machine Artifact-Link Bead Writes From Dirtying The Primary Beads Clone

## Problem

Agents waiting on closed beads stall because the `wait_checks` chop reads closed-bead
state from the canonical primary beads clone
(`~/projects/github/sase-org/sase/sase/repos/beads`, `closed_bead_ids_for_project` in
`src/sase/bead/store_locator.py`), and that clone keeps falling behind its remote: the
pull-only `sidecar_auto_sync` chop (`sync_primary_sidecar_role` in
`src/sase/_sidecar_auto_sync.py`) deliberately refuses to fast-forward a dirty clone,
and the clone keeps being dirtied by uncommitted machine-generated files.

## Confirmed Root Cause

The hourly `artifact_link_backfill` housekeeping chop
(`src/sase/scripts/sase_chop_artifact_link_backfill.py`) derives `plan implements bead`
links from plan `bead_id:` frontmatter and drains them through the artifact-link outbox.
The publication path (`publish_artifact_link_events` -> `_apply_events_to_beads` ->
`set_bead_endpoint_projection`) then:

1. **Writes first, authorizes later.** `apply_events_to_beads`
   (`src/sase/sdd/_artifact_link_event_project.py`) appends `link_added` events to bead
   event streams and rewrites `issues.jsonl` `links` fields via the Rust bead facade
   with no ownership check, then calls `commit_bead_link_events`.
2. **The commit is refused.** `commit_sdd_files` (`src/sase/sdd/_commit_store.py`) calls
   `authorize_store_mutation(mutation_origin="machine")`
   (`src/sase/workspace_provider/_ownership_authorize.py`), which fails closed on
   primary workspace #0. The housekeeping log confirms this every tick:

   ```
   sase: artifact-link bead event publication failed: machine mutation refused at
         /home/bryan/projects/github/sase-org/sase/sase/repos/beads: path...
   sase: artifact-link bead projection is not committed
   ```

3. **The dirty worktree is left behind.** The refusal is swallowed as a best-effort
   diagnostic; the pre-commit worktree writes stay uncommitted in the shared primary
   beads clone. Auto-sync then reports `dirty` and skips, so the clone stops converging
   and bead waiters hang. Bead-sync self-healing
   (`src/sase/sdd/_repository_recovery*.py`) periodically stashes the foreign changes
   (the `sase recovery refs/sase/recovery/...` stashes visible in that clone), after
   which the next chop tick re-dirties it, because the failed operations stay queued in
   the outbox and the swept-refs checkpoint never marks errored documents done.

Why the machine store's beads root is the primary clone at all: commit `2edf986b6`
(phase sase-y3.3, "route machine artifact-link writes to hidden host-owned clones",
landed 2026-09-08) moved only the _document_ sidecar roles (plans, research) to hidden
host-owned clones under `~/.sase/projects/<project_key>/repos/<role>`;
`resolve_machine_artifact_link_store` (`src/sase/sdd/_artifact_link_machine_store.py`)
explicitly leaves `beads_dir` at the primary's nested clone "as read context". That
assumption was true when sase-y3 landed, and was then invalidated one day later: the
bead endpoint projection leg (`publish_artifact_link_events` ->
`_apply_events_to_beads`) arrived with sase-yy.4 (commit `37ab56bd9`, 2026-09-09,
immutable link events v2) and writes beads without inheriting sase-y3's
authorize-before-mutate invariant ("skip that root's work with a diagnostic instead of
writing files a refused commit will strand" — epic plan
`plan:202609/machine_link_mutations_off_primary.md`, authorize-before-mutate step 5).
Additionally, the outbox drain's machine-writability probe
(`_partition_machine_writable_entries` in `src/sase/sdd/_artifact_link_outbox_drain.py`)
never covers the beads root, because `sidecar_root_for` returns `None` for `bead:` refs
(`bead` is in `NON_SIDECAR_KINDS`).

The stashed changes match exactly: `link_added` events with `origin: "derived"`,
description ``derived from the plan's `bead_id:` frontmatter field``, and the stable
epoch `created_at` (`1970-01-01T00:00:00Z`) produced by
`artifact_link_stable_fact_created_at()`.

## Constraints From Prior Fixes (Do Not Regress)

- **sase-y3.3 hidden-clone routing** (commit `2edf986b6`): background link maintenance
  must never dirty or commit into clones nested under a human workspace. This plan
  _completes_ that work for beads; it must not move any document role back.
- **Ownership contract**: do not weaken `authorize_store_mutation`. Machine origin must
  keep failing closed on primary #0, unclaimed checkouts, and read-only canonical
  locations. `_hidden_sidecar_machine_context` already recognizes
  `~/.sase/projects/<key>/repos/<role>` as machine-writable (`HOST_OWNED_SIDECAR`) for
  any role recorded in the primary's SDD store record — beads included — so no ownership
  changes are needed or wanted.
- **Primary stays pull-only**: `mark_bead_wait_sync_hint`
  (`src/sase/axe/run_agent_wait_deps.py`) documents that the runner no longer integrates
  the canonical primary bead sidecar directly; the primary converges only via
  `sync_primary_sidecar_role`. Keep it that way; do not "fix" this by committing or
  cleaning the primary clone from machine code.
- **Auto-sync conservatism**: `sync_primary_sidecar_role` must keep skipping dirty /
  detached / diverged clones.
- **Rust core boundary**: the acknowledgment policy (`artifact_link_publication_receipt`
  in sase-core) is correct and unchanged — a bead-owner event still needs a real bead
  receipt. No sase-core changes.
- **Coordinate with the in-flight sase-yy.8.6 landing**: that landing audit owns
  remaining durable-truth defects (publication retry after failed push, duplicate
  immutable operations, `_install_local_receipt_events` semantics). This tale must not
  change receipt/acknowledgment semantics or operation-id derivation; it only changes
  _where_ the machine lane's bead writes land and _whether_ they are attempted. Both
  discovered issues from this diagnosis (this one, and the bob-cli
  `operation_id ... reused for different events` store wedge) are recorded as DISCOVERED
  ISSUE notes on bead `sase-yy.8.6`; rebase over its landing commits if they touch
  `_artifact_link_event_publish.py` / `_artifact_link_outbox_drain.py`.
- **Related, not in scope**: task bead `sase-10y` (dirty _hidden plans clone_ wedges the
  shared write lane) asks for self-healing of foreign dirt in the hidden document lane.
  Step 2 here applies the same fail-closed principle to the bead leg but does not
  implement hidden-clone quarantine/self-healing; leave that to sase-10y's worker.

## Implementation

### 1. Route the machine store's beads root to a hidden host-owned clone

In `src/sase/sdd/_artifact_link_machine_store.py`:

- Extend the hidden-store resolution (`_hidden_document_store`, called from
  `resolve_machine_artifact_link_store`) to also resolve the beads role
  (`BEADS_SIDECAR_ROLE`) to `hidden_sidecar_clone_dir(project_key, "beads")` via the
  existing `_ensure_hidden_document_root` machinery (`fresh=True`, identity checks,
  `_primary_reference_repo` for cheap first materialization), when
  `store.remote_url_for_kind("beads")` is configured.
- On success, `replace(store, beads_dir=<hidden beads dir>, ...)` and update
  `sidecar_dirs` / `sidecar_remote_urls` for the beads role so
  `ArtifactLinkStore.from_sdd_store` and `commit_bead_link_events` both see the hidden
  clone.
- On failure (hidden clone unavailable, identity mismatch, deadline), keep the primary
  beads path as read context, record the diagnostic (mirroring the existing
  `unresolved_sidecars` handling without breaking `remote_url_for_kind`), and rely on
  step 2 to refuse the write cleanly.
- Update the module docstring: beads is no longer blanket "read context"; machine bead
  writes go to the hidden beads clone, and the primary still converges only via
  pull-based auto-sync.

The commit-and-publish path needs no changes: `commit_bead_link_events` already commits
`beads_dir`'s repo with the caller's `mutation_origin` and then runs
`ensure_bead_mutation_published`, which pushes the hidden clone's commit to the beads
remote. The primary then converges through the existing auto-sync pull, and
`wait_checks` sees closed beads again.

### 2. Fail closed before writing bead projections

In `apply_events_to_beads` (`src/sase/sdd/_artifact_link_event_project.py`): before the
first `set_bead_endpoint_projection` call, authorize the beads repository for this
mutation — same target derivation as `commit_bead_link_events` (`beads_dir` when it
contains `.git`, else its parent), via
`authorize_store_mutation(repo, mutation_origin=mutation_origin)`. On
`WorkspaceOwnershipError`, return
`_ArtifactLinkBeadProjectionResult(changed=False, receipt=False, diagnostic=...)`
without touching the worktree.

This is the invariant the whole incident violated: never mutate a bead worktree the
commit step will refuse. User origin with no context still returns immediately from
`authorize_store_mutation`, so foreground CLI and agent-workspace flows are unchanged.

### 3. Probe the beads root in the outbox drain

In `_partition_machine_writable_entries`
(`src/sase/sdd/_artifact_link_outbox_drain.py`): for entries whose `_sidecar_refs`
include a `bead:` ref, additionally probe `store.beads_dir` (when set) with
`probe_machine_writable_sidecar_root`, retaining the entry with the standard
`sidecar_root_not_machine_writable_message` diagnostic when it fails. This keeps the
drain honest — unpublishable bead-leg entries are retained up front instead of writing
document-root event objects for an operation whose bead receipt can never arrive.

### 4. Tests

- `tests/sdd/test_artifact_link_machine_store.py`: the machine store's `beads_dir`
  resolves to the hidden beads clone; when the hidden beads clone cannot be prepared,
  the store keeps the primary path and surfaces a diagnostic.
- `tests/sdd/test_artifact_link_hidden_clone_e2e.py`: a derived `plan implements bead`
  candidate drained through the machine store lands as a committed `link_added` event in
  the hidden beads clone, is published (pushed), and the primary's nested beads clone
  worktree stays byte-for-byte clean.
- New regression for step 2: with a beads root that fails machine authorization,
  `apply_events_to_beads` returns `receipt=False` with a diagnostic and the beads
  worktree is unmodified (compare `git status --porcelain` before/after).
- Drain regression for step 3: a queued derived event targeting a bead is retained with
  a skip diagnostic when the beads root is not machine-writable, and drained once it is.

### 5. Verification

- `just check` (two-speed default; `just check-full` stays the landing gate).
- After landing on the host: one `artifact_link_backfill` tick logs no
  `machine mutation refused at .../repos/beads` and no
  `artifact-link bead projection is not committed` for the sase project;
  `~/projects/github/sase-org/sase/sase/repos/beads` stays clean (`git status`), and
  `sidecar_auto_sync` reports `refreshed`/`up_to_date` for the beads role. The
  previously queued outbox entries drain (`outbox_drained > 0`) and `sweep_remaining`
  stops re-scanning the same errored documents.
- Re-check the
  `sase: artifact-link aggregate projection failed: artifact-link legacy/event overlap rejected before event cutover`
  warning after the queue drains; it is expected to clear once publication completes. If
  it persists, file a task bead rather than expanding this change.

## Aftercare (Host, No Code)

- The `sase recovery refs/sase/recovery/...` stashes in the primary beads clone are
  managed by the recovery reaper and need no manual action.
- The manual `WIP on main` stash there contains only machine-generated derived-link
  noise; the fixed machine lane republishes the same facts with stable operation ids, so
  the stash is safe to drop. Leave that to the user.

## Out Of Scope

- The `bob-cli` project's recurring drain/reconcile failure
  (``operation_id `de29d2e25c1cfb4381f223c44d576f8c` was reused for different events``)
  is a distinct defect in the sase-yy durable-truth scope, recorded as a DISCOVERED
  ISSUE note on bead `sase-yy.8.6`.
- Bead publication-retry sweeps for the hidden beads clone:
  `ensure_bead_mutation_published` plus bead sync hints already own bead push retries;
  do not fold beads into `machine_document_sidecar_roots`.
