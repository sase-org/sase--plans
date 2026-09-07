---
tier: epic
title: Keep machine artifact-link mutations out of primary sidecar clones
goal: 'SASE background link maintenance never dirties or commits into the sidecar
  clones nested under a project''s primary (human) workspace checkout: rename-repair
  deletions are committable, every background writer authorizes before mutating with
  an honest machine origin, machine writes land in hidden host-owned sidecar clones
  that push to the remote, the primary''s clones converge via pull-based auto-sync
  only, and the currently stranded deletions are healed with a doctor guardrail against
  recurrence.

  '
phases:
- id: stage-removed-link-indexes
  title: Make removed link indexes committable
  depends_on: []
  size: small
  description: 'stage-removed-link-indexes: validate deleted per-artifact link-index
    paths by canonical location instead of on-disk content in _group_valid_indexes
    so rename repairs commit their deletions alongside their rewrites, with regression
    tests proving one clean commit and no leftover worktree dirt.'
- id: authorize-before-mutate
  title: Authorize before mutating, with honest machine origin
  depends_on:
  - stage-removed-link-indexes
  size: medium
  description: 'authorize-before-mutate: gate every background link-maintenance writer
    (rename repair, backfill sweep, outbox drain, referenced-by refresh) on machine
    writability of each sidecar root before any worktree write, skip unauthorized
    roots with diagnostics, and replace defaulted user mutation origins with explicit
    machine origin on background commit paths.'
- id: hidden-clone-machine-writes
  title: Machine writes move to hidden host-owned sidecar clones
  depends_on:
  - stage-removed-link-indexes
  - authorize-before-mutate
  size: large
  description: 'hidden-clone-machine-writes: add a machine-context artifact-link store
    resolution rooted at hidden host-owned sidecar clones following the agents-sidecar
    precedent, bless those clones for machine mutation in the ownership contract without
    weakening primary-#0 refusal, switch the artifact_link_backfill chop and agents-sync
    referenced-by drain to it, and verify primary clones converge via pull-based auto-sync.'
- id: heal-and-guard
  title: Heal stranded deletions and add a doctor guardrail
  depends_on:
  - hidden-clone-machine-writes
  size: small
  description: 'heal-and-guard: add a doctor check for dirty machine-managed sidecar
    clones under a primary checkout with a restore-based user-invoked fix, propose
    the one-time restore of the six stranded research link-index deletions through
    a gate, and file task beads proposing a decisions-web record plus any unpublished-commit
    retry gap.'
proposed_by: bbugyi200.athena.04n
create_time: 2026-09-07 14:58:55
status: wip
bead_id: sase-y3
---

- **BEAD:** [sase-y3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-y3/README.md)

# Keep Machine Artifact-Link Mutations Out Of The Primary Workspace's Sidecar Clones

## Problem

The user found unexplained dirty files in the research sidecar clone nested inside the
sase project's **primary (human) workspace checkout**: six unstaged deletions of
per-artifact link indexes (`links/202609/*.md.json`, e.g.
`tailnet_agent_fleet_v2.md.json`, `provider_neutral_remote_dispatch__a.md.json`). The
user made none of these changes. Because `sync_primary_sidecar_role`
(`src/sase/_sidecar_auto_sync.py`) skips dirty clones, the dirty state also silently
blocks that clone from ever syncing again.

The user's expectation, which this epic adopts as the target invariant: **a project's
primary workspace directory (including the sidecar clones nested under its `repos/`
directory) is for human work and sanctioned pull-based sync only; SASE machine processes
must never leave dirty state there and must do their write work in a location owned by
the machinery that made the change.**

## Root Cause (diagnosed, with evidence)

The suspicion is confirmed: SASE background processes write into the primary checkout's
sidecar clones. Four cooperating defects produce the observed dirt:

1. **Background link maintenance anchors its store at the primary checkout.** The hourly
   `artifact_link_backfill` chop (housekeeping lumberjack;
   `src/sase/scripts/sase_chop_artifact_link_backfill.py`) resolves
   `resolve_artifact_link_store(cwd=Path(record.workspace_dir))` per enabled project,
   and a project record's `workspace_dir` is its primary checkout. The agents-sync
   referenced-by drain (`src/sase/agents_sync/referenced_by_publication.py`,
   `_resolve_store`) likewise resolves from `target.primary_checkout`. Both therefore
   get sidecar roots pointing at `<primary>/repos/<role>` (via `resolve_sdd_store` ->
   `resolve_sidecar_clone_root` -> `sidecar_repo_clone_dir`).

2. **Worktree mutation happens before authorization.** The chop's reconcile/repair job
   (`reconcile_and_repair_artifact_links` in `src/sase/sdd/artifact_link_backfill.py` ->
   `repair_historical_artifact_renames` in `src/sase/sdd/_artifact_link_renames.py`)
   rewrites rename-successor indexes with `atomic_write_json` and `unlink()`s the stale
   ones, and only then tries to commit with `mutation_origin="machine"`. The ownership
   gate (`authorize_store_mutation` in
   `src/sase/workspace_provider/_ownership_authorize.py`) correctly refuses machine
   mutations that resolve to primary workspace #0 — but only the commit is gated, so the
   refusal strands the deletions as dirty worktree state. Log evidence (housekeeping
   lumberjack log, 2026-09-06 12:39:30):
   `sase: reconcile/repair failed: machine mutation refused at ...`.

3. **Mutation origins are inconsistent, and the default bypasses the gate.** The chop's
   backfill sweep persists via `persist_derived_link_candidates`
   (`src/sase/sdd/artifact_link_derivation.py`) ->
   `persist_artifact_link_graph_mutation` -> `commit_artifact_link_indexes` with no
   `mutation_origin`, which defaults to `"user"`; `authorize_store_mutation` returns
   immediately for user origin with no context, so these commits **succeed** at the
   primary (the `chore(artifact-links): persist link indexes` commits in the research
   sidecar history, e.g. 9a1dcaf / c5b5aee on 2026-09-06, `SASE_TYPE=sdd`). The
   referenced-by refresh (`refresh_artifact_links_locked` in
   `src/sase/sdd/_artifact_link_refresh.py` -> `commit_sdd_store_files`) also defaults
   to user origin. Meanwhile the outbox drain (`drain_artifact_link_outbox`) and the
   reconcile/repair job honestly pass `"machine"` and get refused. So the same chop both
   commits at the primary (dishonest origin) and strands worktree changes there (honest
   origin, gated too late).

4. **Removed link indexes can never be committed at all.** `_group_valid_indexes` in
   `src/sase/sdd/_artifact_link_commit.py` validates every candidate path with
   `is_canonical_artifact_link_index` (`src/sase/sdd/_artifact_link_files.py`), which
   requires `path.is_file()` plus a valid schema-v2 payload. A just-deleted index fails
   and is silently dropped, so even in a fully authorized context a rename repair's
   deletions are excluded from the commit while its rewrites land — exactly the observed
   "committed successors + stranded deletions" split.

Design-intent references that support the target invariant:
`canonical_sidecar_dir_for_project` (`src/sase/bead/store_locator.py`) documents the
primary's sidecar root as canonical for _readers_ ("Automated mutation must go through a
writable operation context"); `AccessKind.PRIMARY_SIDECAR_SYNC` plus
`require_separate_sidecar_clone` (`src/sase/workspace_provider/_ownership_paths.py`)
define the only sanctioned machine lane into the primary's sidecar clones (pull-style
sync of a separate clone); and the agents sidecar already lives in a hidden host-owned
clone via `hidden_sidecar_clone_dir` (`src/sase/_linked_repo_paths.py`,
`~/.sase/projects/<key>/repos/<role>`), which is the precedent this epic extends.

Note: direct agent CLI writes are _not_ offenders — `resolve_artifact_link_store`
anchors at the calling checkout root (`resolve_checkout_anchor`), so an agent running
`sase artifact` from a numbered workspace mutates its own workspace's sidecar clone and
publishes via push. Only host-side background jobs anchored at the primary are broken.

## Direction

- Fix the deletion-staging defect so rename repairs are committable wherever they are
  authorized (Phase 1).
- Enforce authorize-before-mutate with honest `machine` origin across every background
  link-maintenance writer, so an unauthorized root is skipped with a diagnostic instead
  of half-mutated or dishonestly committed (Phase 2). Until Phase 3 lands, this
  deliberately turns background link maintenance into a clean, diagnosed no-op for
  primary-anchored stores.
- Route machine link maintenance to hidden host-owned sidecar clones
  (`~/.sase/projects/<key>/repos/<role>`, the agents-sidecar precedent), commit and push
  there, and let the primary's nested clones converge through the existing pull-based
  sidecar auto-sync (Phase 3).
- Heal the currently-stranded deletions and add a doctor guardrail so machine dirt in
  primary sidecar clones is surfaced instead of lingering (Phase 4).

## Phase: stage-removed-link-indexes

- slug: stage-removed-link-indexes
- size: small
- depends: []

Make removed per-artifact link indexes committable.

Steps:

1. In `src/sase/sdd/_artifact_link_files.py`, add a location-based validator (e.g.
   `is_canonical_artifact_link_index_location`) that answers whether a path — existing
   or not — is a canonical per-artifact link-index location under an owning root
   (`links/**/*.json`, no symlink games, canonical relative shape), without reading the
   file. Reuse the existing classification helpers where possible.
2. In `src/sase/sdd/_artifact_link_commit.py` `_group_valid_indexes`, keep a candidate
   path that no longer exists on disk when it satisfies the location-based validator for
   its owning root; keep the current content validation for paths that do exist. The
   downstream machinery already stages deletions once paths survive grouping
   (`changed_sdd_files` uses `git ls-files --modified --others --deleted`, and
   `git add -- <path>` stages a deletion), so no change is needed there.
3. Add regression tests: a rename repair (`repair_historical_artifact_renames` or
   `_apply_artifact_renames`-level fixture) in a writable sidecar clone must produce a
   single commit containing both the rewritten successor index and the deletion of the
   stale index, leaving the worktree clean. Also cover `commit_artifact_link_indexes`
   called directly with a mix of existing and deleted index paths.

Verification: `just install` (fresh workspace), then `just check`.

## Phase: authorize-before-mutate

- slug: authorize-before-mutate
- size: medium
- depends: [stage-removed-link-indexes]

No background link-maintenance job may touch a sidecar worktree it is not authorized to
commit in, and none may commit with a dishonest mutation origin.

Steps:

1. Add a small helper (suggested home: `src/sase/sdd/_artifact_link_commit.py` or a
   sibling) that answers "is this sidecar root machine-writable?" by calling
   `authorize_store_mutation(root, mutation_origin="machine")` and translating
   `WorkspaceOwnershipError` into a boolean plus diagnostic, without mutating anything.
2. `repair_historical_artifact_renames` / `_apply_artifact_renames`
   (`src/sase/sdd/_artifact_link_renames.py`): before rewriting or unlinking anything
   under a sidecar root, check machine-writability of that root; skip unauthorized roots
   entirely and surface a per-root diagnostic in the report (the chop already logs
   report warnings). The aggregate rewrite (machine-local
   `~/.sase/projects/<key>/artifact-links.json`) is not a git mutation and stays as-is.
3. `reconcile_and_repair_artifact_links` (`src/sase/sdd/artifact_link_backfill.py`):
   propagate the new skip diagnostics into the chop warning stream so a skipped project
   says _why_ (e.g. "research root not machine-writable: resolves to primary #0").
4. Fix the dishonest origins: `persist_artifact_link_graph_mutation`
   (`src/sase/sdd/_artifact_link_commit.py`) must accept and forward a
   `mutation_origin`, and its background callers — `persist_derived_link_candidates`
   (`src/sase/sdd/artifact_link_derivation.py`) and any other chop/drain-reached call
   site — must pass `"machine"`. Interactive CLI callers (e.g. `sase artifact link add`,
   the plan-links inlet) keep user origin. Same for the referenced-by refresh commit
   (`_commit_changes` in `src/sase/sdd/_artifact_link_refresh.py` /
   `commit_sdd_store_files`) when driven by the agents-sync drain.
5. Apply the same pre-authorization gate before the sweep's and drains' _worktree
   writes_ (the `store._upsert_sidecar` path and `refresh_artifact_links_locked`'s
   index/document writes): when the target sidecar root is not machine-writable, skip
   that root's work with a diagnostic instead of writing files a refused commit will
   strand.
6. Tests: with a store whose sidecar roots resolve to a primary-#0-owned clone, each
   background job (sweep persist, outbox drain, rename repair, referenced-by refresh)
   must leave the worktree byte-identical, create no commit, and report the skip
   diagnostic. With a machine-writable root, behavior is unchanged from Phase 1.

Note the intended interim regression: on hosts where these stores anchor at the primary,
background link maintenance becomes a diagnosed no-op until Phase 3 relocates the
writes. That is strictly better than dirtying the primary.

Verification: `just install`, then `just check`.

## Phase: hidden-clone-machine-writes

- slug: hidden-clone-machine-writes
- size: large
- depends: [stage-removed-link-indexes, authorize-before-mutate]

Give machine link maintenance a home of its own: hidden host-owned sidecar clones.

Design outline (the phase worker plans the details):

1. **Machine store resolution.** Add a machine-context resolution mode for the
   artifact-link store (e.g. `resolve_artifact_link_store(..., machine=True)` or a
   dedicated `resolve_machine_artifact_link_store(project_key)`) whose document sidecar
   roots live at `hidden_sidecar_clone_dir(project_key, role)`
   (`~/.sase/projects/<key>/repos/<role>`), following the existing agents-sidecar
   pattern. Materialize a missing hidden clone on demand from the recorded sidecar
   remote URL (see `ensure_sidecar_sdd_clone` / `ensure_beads_sidecar_clone` in
   `src/sase/sdd/_store_workspace.py`, including the `reference_repo` trick for cheap
   objects), and integrate fresh before mutating so repairs run against current remote
   state.
2. **Ownership blessing.** Teach the ownership contract to authorize machine mutations
   at hidden host-owned sidecar clones. Today `_infer_machine_context`
   (`src/sase/workspace_provider/_ownership_authorize.py`) refuses them ("missing
   checkout marker or registry evidence") because they are outside any registered
   checkout. Options for the phase worker to weigh: a new `AccessKind` for hidden
   sidecar clones with an explicit `OperationContext` constructed by the machine store
   resolver, or registry/marker evidence for hidden clones. Fail-closed semantics for
   everything else must not weaken; add tests asserting primary #0 refusal is unchanged.
3. **Switch the two background anchors.** The `artifact_link_backfill` chop
   (`_run_project` in `src/sase/scripts/sase_chop_artifact_link_backfill.py`) and the
   agents-sync referenced-by drain (`_resolve_store` in
   `src/sase/agents_sync/referenced_by_publication.py`) resolve the machine store
   instead of the primary-anchored one. All four chop jobs (sweep, outbox drain,
   aggregate reconcile, rename repair) and the drain then write, commit
   (`mutation_origin="machine"`), and push (`push_after_commit="async"`, keeping the
   existing publication verification) in the hidden clones.
4. **Convergence.** The primary's nested `repos/<role>` clones receive these changes
   exclusively through the existing pull-based auto-sync (`sync_primary_sidecar_role`).
   Verify the auto-sync cadence covers the research and plans roles and document (in
   code comments where the machine resolver lives) that the primary's clones are human +
   pull-sync only.
5. **Tests.** End-to-end: a rename in a document sidecar leads to the hidden clone
   committing rewrite + deletion and pushing; a primary-anchored clone of the same
   remote fast-forwards cleanly via the auto-sync path with zero worktree dirt. Plus
   ownership-contract unit tests for the new machine-writable location.

Verification: `just install`, then `just check`; this phase also warrants a
`just check-full` run through a monitor before the epic's combined tree lands.

## Phase: heal-and-guard

- slug: heal-and-guard
- size: small
- depends: [hidden-clone-machine-writes]

Heal existing dirt and keep it from silently returning.

Steps:

1. **Doctor guardrail.** Extend the artifact-links doctor checks
   (`src/sase/doctor/checks_artifact_links.py`) with a check that flags a dirty
   machine-managed sidecar clone nested under a primary checkout (uncommitted changes
   under `links/`, or any uncommitted machine-pattern dirt), explaining that dirt blocks
   sidecar auto-sync. Offer a fix path (doctor `--fix` or a printed exact command) that
   **restores** stranded link-index deletions (`git checkout -- <paths>` or
   `git restore`) rather than committing them — the durable deletion must land via the
   machine lane (hidden clone commit + push) and reach the primary through auto-sync.
   The fix runs as a user-origin action from the human's own CLI, which the ownership
   contract already permits.
2. **One-time healing of the current dirt.** The six stranded deletions in the sase
   project's primary research sidecar clone (`links/202609/`:
   `cross_machine_agent_control_plane.md.json`, `legacy_backcompat_removal.md.json`,
   `legacy_compatibility_retirement.md.json`,
   `provider_neutral_remote_dispatch__a.md.json`,
   `remote_dispatch_plugin_architecture__b.md.json`, `tailnet_agent_fleet_v2.md.json`)
   are semantically correct deletions that were stranded by defect 2/4. After Phase 3 is
   live, the repair re-derives and lands them through the hidden clone, so the healing
   action is simply the doctor fix above (restore, then let auto-sync fast-forward).
   Propose the concrete restore command to the user through a gate (`/sase_gate`) rather
   than mutating the primary's clone directly from an agent.
3. **Decision record proposal.** File a `memory` task bead (through `/sase_new_task`)
   proposing a decisions-web record capturing the invariant this epic establishes:
   machine link mutations never target a primary checkout's nested sidecar clones;
   hidden host-owned clones are the machine write lane; the primary converges via
   pull-based auto-sync. Do not write the memory note directly — the bead is the
   sanctioned channel.
4. **Push-failure follow-up.** The 2026-09-06 12:39:30 log also shows
   `persist link indexes was committed locally but NOT published`. If Phase 3's
   publication verification does not already cover retry/aging of unpublished hidden
   clone commits, file a task bead (through `/sase_new_task`) describing the gap.

Verification: `just install`, then `just check`.
