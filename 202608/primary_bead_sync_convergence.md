---
tier: epic
title: Converge bead-store refresh on the single primary-sidecar sync policy
goal: "The only automated writer that touches a project's primary bead sidecar is the
  conservative fetch/fast-forward auto-sync policy, periodic bead chops hold one lease
  per project per tick, and the ownership invariant audit covers the launch and archive
  workflows plus every directory operation the epic introduced.

  "
phases:
  - id: waiter-sync-hints
    title: Retire the competing canonical bead-store refresh path
    depends_on: []
    size: medium
    description:
      "waiter-sync-hints: replace machine-initiated integrate_sdd_repository against
      canonical primary bead sidecars with sync hints consumed by sidecar_auto_sync."
  - id: chop-lease-batching
    title: One lease and one publication per project per claim-check tick
    depends_on: []
    size: medium
    description:
      "chop-lease-batching: fuse the bead_claim_checks read snapshot and reconcile batch
      into a single operational lease per project."
  - id: audit-gaps
    title: Close the ownership epic's own audit gaps
    depends_on: []
    size: small
    description:
      "audit-gaps: review reset_replay._clear_owned_paths in the artifact directory
      audit, register the writable-store import boundary as a source-tree audit, and
      document the remaining user-directed primary bead writers."
  - id: launch-invariant-coverage
    title: Extend the ownership invariant audit to launch and archive workflows
    depends_on:
      - waiter-sync-hints
      - chop-lease-batching
      - audit-gaps
    size: medium
    description:
      "launch-invariant-coverage: assert primary worktree/index/HEAD/ref stability
      across plan approval and archive, epic launch, and task launch."
parent_bead: sase-mq
proposed_by: bbugyi200.athena.sase-mq.land
create_time: 2026-08-16 04:51:30
status: wip
---

- **PROMPT:**
  [prompts/202608/primary_bead_sync_convergence.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/primary_bead_sync_convergence.md)
- **PARENT:**
  [202608/primary_workspace_ownership.md](https://github.com/sase-org/sase--plans/blob/main/202608/primary_workspace_ownership.md)

# Plan: Converge bead-store refresh on the single primary-sidecar sync policy

## Context

Epic `sase-mq` (`plan:202608/primary_workspace_ownership.md`) established a workspace
ownership contract, operational leases, reset-and-replay recovery, leased launches,
leased background bead writers, and a generic opt-in primary-sidecar auto-sync. Its
seven phases closed and its own test suites pass (242 tests across
`tests/workspace_provider/`, `tests/test_sidecar_auto_sync.py`,
`tests/test_sidecar_sync_hints.py`, `tests/test_axe_chop_sidecar_auto_sync.py`, and
`tests/test_bead/test_background_store.py`).

Landing verification found four pieces of that epic's declared scope still open. Two of
them are explicit requirements the epic's own plan states but no phase implemented; one
is a red test the epic introduced; one is an integration gap against a registry that
landed while the epic was in flight.

## Findings

### 1. Two synchronizers still compete for the primary bead sidecar

The ownership contract's rule 4 says a primary sidecar may only be fetched and
fast-forwarded when it is clean, attached, and non-diverged, and that SASE "never
rebases, commits, hard-resets, or removes user state" there. Phase 6 built exactly that
(`src/sase/_sidecar_auto_sync.py` plus the `sidecar_auto_sync` chop) and its plan text
closes with: "Retire or narrow the bead-only primary refresh path so there is one
scheduling policy rather than competing synchronizers." Phase `sase-mq.6` deliberately
deferred this, recording a `PROPOSED FOLLOW-UP` note that it was unsafe to retire before
`sase-mq.5` moved background writers off the primary. `sase-mq.5` has since closed, and
the deferral was never revisited.

Two machine-initiated paths still call `sase.bead.sync.refresh_bead_store()` against
`canonical_beads_dir_for_project(...)`, i.e. the user's primary bead sidecar clone:

- `src/sase/scripts/sase_chop_bead_store_refresh.py:260-272` — the `bead_store_refresh`
  chop, registered in the `waits` lane at a 30s `run_every` with a 2m timeout
  (`src/sase/default_config.yml:695-703`, console script at `pyproject.toml:133`).
- `src/sase/axe/run_agent_wait_deps.py:21-35` — `refresh_bead_wait_store()`, the waiting
  runner's coarser outage backstop.

`refresh_bead_store()` is not conservative. Via `src/sase/bead/_sync_refresh.py:38-96`
it takes a `mutates_worktree=True` store write lock and calls
`integrate_sdd_repository()`, which commits, rebases/pulls, pushes, and then runs
`repair_event_manifest_after_integration()`
(`src/sase/sdd/_repository_integration.py:307-311`).

The epic bead's own `DISCOVERED ISSUE` note records the damage this caused in
production: the canonical primary bead-store clone diverged 1 local / 10 remote commits
and accumulated 53 consecutive failed managed-sync integrations, 41 of them
`dirty-worktree refusal`, plus 2 retained recovery refs and 2 recovery stashes. Because
the primary clone was 10 commits behind, waiting phase agents polling it could not
observe phase closes that had already landed, stalling epics `sase-mi`, `sase-mj`, and
`sase-m6.6.1` on dependencies that were in fact satisfied. Separately,
`events/manifest.json` `stream_count` flip-flopped 851 → 852 → 851 → 853 across
consecutive `repair event manifest` commits (`d1ee870f`, `420fd244`, `6c734f5c`) — two
clones repairing each other, which is precisely the competing-synchronizer failure mode.

### 2. `bead_claim_checks` takes two leases per project per tick

Phase 5's plan text: "Chops should batch one project's changes under one lease and one
publication transaction so periodic work does not churn claims."
`src/sase/scripts/sase_chop_bead_claim_checks.py` acquires one operational lease in
`_read_claimed_issues()` (line 254) to read a refreshed snapshot and a second in
`_reconcile_project_claims()` (line 82) to write the batch, for every project on every
tick. Phase `sase-mq.5` recorded this as a `PROPOSED FOLLOW-UP` against its own
requirement.

### 3. The epic left a red audit and an unregistered one

`tests/test_agent_artifact_directory_operation_audit.py` fails on a clean tree:

```
Extra items in the left set:
'src/sase/workspace_provider/reset_replay.py:_clear_owned_paths'
```

`_clear_owned_paths` (`src/sase/workspace_provider/reset_replay.py:167-180`) calls
`shutil.rmtree` and was added by phase 3 (commit `985aae20c`). Phases `sase-mq.4` and
`sase-mq.5` both filed it as a `PROPOSED FOLLOW-UP`; phase `sase-mq.7` did not review
it. This is the epic's own new code failing the epic's own class of gate.

Separately, commit `298cea966`
(`test(selection-health): register the proc-submission audit as a source-tree audit`,
landed 2026-08-16 00:24, while this epic was in flight) widened
`_SOURCE_AUDIT_SCAN_ROOTS` in `tests/_test_selection_health_correlation.py` to cover
invariant-style audits that walk a source tree, so their dirty-tree failures are
attributed to the agent's own edit instead of counted as flake evidence. Phase 7's
`tests/workspace_provider/test_primary_writable_store_import_boundary.py` (landed 04:18)
is exactly that shape — it `rglob`s `src/` and asserts a fixed invariant — and was never
registered.

### 4. Stale deferral comments on user-directed primary writers

`src/sase/bead/task_launch.py:52-58` documents `_resolve_task_launch_cwd` as "Used only
for background bead-store mutations (close/snooze) that still resolve a primary cwd; see
sase-mq.5 for moving those off the canonical primary clone." `sase-mq.5` closed without
moving them, because those call sites — `src/sase/bead/_task_gate_actions.py:159-170`
and `src/sase/bead/snooze_gate.py:593-604` — apply a user's answer to a task-triage or
snooze gate and commit with the default `user` mutation origin. That is a foreground
user action under ownership rule 3, not a background writer. Phase 7's plan text
requires any remaining primary writer to be "explicitly document[ed] why it is a
foreground user action"; a pointer to a closed phase is not that.

## Phase 1: Retire the competing canonical bead-store refresh path

Make the live-bead-waiter signal a hint source for the one auto-sync policy instead of a
second synchronizer.

- Delete the `bead_store_refresh` chop:
  `src/sase/scripts/sase_chop_bead_store_refresh.py`, its `pyproject.toml`
  console-script entry, its `waits`-lane block in `src/sase/default_config.yml`, and
  `tests/test_axe_chop_bead_store_refresh.py`. Fold what
  `tests/test_axe_chop_wait_checks_beads.py` still covers about waiter-driven refresh
  into the replacement path rather than deleting the coverage.
- Move the "projects with a live, unresolved bead waiter" scan (today
  `_projects_with_live_bead_waits`, an `ace-run` artifact scan filtered by
  `waiting.wait_for_beads`, absent `ready.json`, and a live owner process) into
  `sase_chop_sidecar_auto_sync`. For each such project, mark the `beads` role hint via
  `sase._sidecar_sync_hints.mark_sidecar_sync_hint` so the existing hinted-role path
  syncs it on the next 30s tick instead of waiting for the 5-minute backstop.
- Preserve today's unblock guarantee exactly: a project with a live bead waiter must
  have its `beads` role converged even when that role has not set `auto_sync: true`.
  Honor an explicit `disabled` sidecar entry and keep `HIDDEN_SIDECAR_ROLES` excluded.
  Document in the code why bead-wait liveness implies coverage: the sync performed is
  the same conservative fetch/fast-forward, so implicit coverage is strictly less
  invasive than the managed integration it replaces.
- Keep `sdd.bead_refresh.mode: off` disabling the waiter-driven path, matching what
  `docs/configuration.md` already promises for that setting.
- Replace `refresh_bead_wait_store()` in `src/sase/axe/run_agent_wait_deps.py` with hint
  marking. The runner must not call `refresh_bead_store()` on a canonical store; it may
  mark the hint and continue. Update its callers and docstring accordingly.
- Verify no machine-initiated caller of `refresh_bead_store()` resolves
  `canonical_beads_dir_for_project` afterwards. The surviving callers must all pass a
  leased store (`src/sase/bead/claims.py:211`,
  `src/sase/scripts/sase_chop_bead_claim_checks.py:89,262`,
  `src/sase/external_mirror/issues.py:326`) or be user-directed CLI
  (`src/sase/bead/cli_common.py`, `src/sase/main/bead_fast_path.py`,
  `src/sase/scripts/sase_clan_summary_epic.py`).
- Update `docs/axe.md` (the `waits` lane chop table and the `wait_for_beads` paragraph
  describing "`bead_store_refresh` integrates their projects' canonical stores every 30
  seconds, with the waiting runner providing a coarser outage backstop") and
  `docs/configuration.md` (the `waits` lane example and the `sdd.bead_refresh.mode` row
  that names the retired chop).
- Tests: a project with a live bead waiter gets a `beads` hint and a clean, strictly
  behind primary bead sidecar fast-forwards on the next tick; a dirty or diverged
  primary bead sidecar is left byte-for-byte untouched and reported instead of
  integrated; `bead_refresh_mode() == "off"` suppresses the waiter-driven hint; a waiter
  whose owner process is dead produces no hint; and no code path reachable from a chop
  or the runner calls `integrate_sdd_repository` against a canonical bead store.

## Phase 2: One lease and one publication per project per claim-check tick

Restructure `src/sase/scripts/sase_chop_bead_claim_checks.py` so each project's read
snapshot and reconcile batch share a single operational lease and a single publication.

- Fuse `_read_claimed_issues` and `_reconcile_project_claims` into one
  `writable_bead_store_for_machine` block per project per tick: refresh once, list
  `CLAIMED` issues from that same refreshed leased store, compute releases and
  acquisitions, apply them under one `bead_store_write_lock`, commit once, publish once,
  and mark the sidecar convergence hint once.
- Preserve existing semantics exactly: the cheap pre-pass that avoids touching stores
  when no unpromoted record needs repair; tombstoning dead records so later ticks skip
  them; the rule that an empty snapshot only proves "nothing left to release" because
  the store was integrated first; per-project failure isolation so one project's error
  cannot stop the chop; and the distinction between a release error and a successful
  release in `_ProjectClaimReconciliation`.
- Tests: assert one lease acquisition per project per tick (not two) while release,
  acquisition, tombstoning, and publication outcomes stay unchanged; keep the existing
  `tests/test_axe_chop_bead_claim_checks*.py` coverage green.

## Phase 3: Close the ownership epic's own audit gaps

- Review `src/sase/workspace_provider/reset_replay.py:_clear_owned_paths` and add the
  correct `DirOpReview` entry to `_REVIEWED_DIR_OPERATION_CONTEXTS` in
  `tests/test_agent_artifact_directory_operation_audit.py`. Decide between an
  `exemption` and `lifecycle_calls`/`batched_by` from what the code actually does: it
  `shutil.rmtree`s only lease-owned generated paths, refusing any path not relative to
  the leased checkout. If agent marker files can live under a cleared path, wire the
  marker lifecycle call instead of exempting it.
- Register `tests/workspace_provider/test_primary_writable_store_import_boundary.py`
  with scan root `("src/sase/",)` in `_SOURCE_AUDIT_SCAN_ROOTS`
  (`tests/_test_selection_health_correlation.py`). The registry's own guard test
  (`tests/test_test_selection_health_correlation.py`) already asserts every registered
  path resolves to a file, so no new gate is needed.
- Replace the stale `sase-mq.5` deferral pointers in
  `src/sase/bead/task_launch.py:_resolve_task_launch_cwd`,
  `src/sase/bead/_task_gate_actions.py:_resolve_task_triage_project_cwd`, and
  `src/sase/bead/snooze_gate.py:_resolve_bead_snooze_project_cwd` with the actual
  ownership justification: these apply a user's gate answer, commit with the default
  `user` mutation origin, and are therefore foreground user actions under ownership rule
  3 rather than background writers awaiting a lease.
- Verify `tests/test_agent_artifact_directory_operation_audit.py` passes on a clean
  tree; it currently fails.

## Phase 4: Extend the ownership invariant audit to launch and archive workflows

`tests/workspace_provider/test_ownership_invariant_audit.py` proves primary immutability
for `authorize_store_mutation`, `reset_and_replay`, and sidecar auto-sync. Phase 7's
plan text also named plan approval/archive, epic launch, and task launch, and those are
not covered.

- Extend the audit with scenarios that snapshot the primary repo's worktree, index,
  `HEAD`, local refs, and operation markers, exercise approval-time plan archiving
  (`src/sase/_plan_archive_approval.py`), approved epic launch
  (`src/sase/bead/epic_launch.py`), and task-bead launch
  (`src/sase/bead/task_launch.py:submit_task_launch_task`), then assert byte-for-byte
  and oid stability of that primary state.
- Assert the negative path too: when lease acquisition, materialization, or claim
  transfer fails, the launch surfaces `OperationalLeaseError` and never falls back to
  the primary checkout.
- Reuse the existing `tests/workspace_lease_helpers.py` fixtures rather than adding a
  parallel harness.
- Because this phase lands after phases 1 and 2, also assert that a full
  `sidecar_auto_sync` tick driven by a bead-waiter hint leaves the primary repository
  itself untouched while converging only the opted-in sidecar clone.

## Verification

Every phase runs `just install` first (ephemeral workspaces drift), then `just check`.
Phases 1, 2, and 4 change core identity files or the chop registry and will escalate the
scoped selector, so hand `just check-full` to `/sase_monitor` with a `--next` action
rather than running it inline. Phase 3 changes only tests and comments and can land on
`just check`.
