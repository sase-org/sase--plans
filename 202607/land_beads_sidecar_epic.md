---
tier: epic
title: Finish and land the dedicated beads sidecar
goal: 'Every code path that locates a bead store''s owning repository handles the
  dedicated beads sidecar''s repo-root layout, so unpushed bead commits survive workspace
  preparation, the ACE Plans tab resolves and commits bead state correctly on migrated
  projects, and epic sase-a8 is closed with its plan file marked done.

  '
phases:
- id: prepare
  title: Restore unpushed bead-commit protection for root-layout stores
  depends_on: []
  size: medium
  description: 'prepare: teach the workspace-preparation preflight that a repository
    can itself be the bead store, so local-only bead commits in the dedicated beads
    clone are published or rescued before a clean and hard checkout discards them.'
- id: flatroot
  title: Stop a README-only plans subdirectory from shadowing a flat plans root
  depends_on: []
  size: small
  description: 'flatroot: require a nested plans directory to hold month directories
    before it disqualifies a flat plans sidecar root, so plan search and SDD file
    listing stop returning nothing for a plans clone that carries a generated directory
    README.'
- id: plansroot
  title: Resolve the ACE Plans tab's plans root through the store
  depends_on:
  - flatroot
  size: medium
  description: 'plansroot: resolve each project''s plans root through the SDD store
    instead of walking up from the bead directory, so linked plan documents, cache
    keys, and the archive search all target the real plans clone.'
- id: acecommit
  title: Commit ACE scoped bead edits against the bead store's own repository
  depends_on: []
  size: small
  description: 'acecommit: resolve the owning git root from the bead directory instead
    of assuming its parent, so bead updates submitted from the Plans tab are actually
    committed and pushed rather than silently dropped.'
- id: land
  title: Close the epic, sweep symbols, and mark the plan done
  depends_on:
  - prepare
  - flatroot
  - plansroot
  - acecommit
  size: small
  description: 'land: re-verify the split end to end, close bead sase-a8 without forcing,
    clear whatever symvision reports once the epic whitelist expires, and mark both
    plan files done.'
create_time: 2026-07-28 07:35:58
status: done
bead_id: sase-ab
---

- **PROMPT:** [202607/prompts/land_beads_sidecar_epic.md](prompts/land_beads_sidecar_epic.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-a8.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-a8.land/README.md)
  - [bbugyi200.athena.sase-ab.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ab.1/README.md)
  - [bbugyi200.athena.sase-ab.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ab.2/README.md)
  - [bbugyi200.athena.sase-ab.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ab.3/README.md)
  - [bbugyi200.athena.sase-ab.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ab.5/README.md)
  - [bbugyi200.athena.sase-ab.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ab.land/README.md)

# Plan: Finish and land the dedicated beads sidecar (sase-a8)

## Motivation

Epic `sase-a8` split bead state out of the plans sidecar into a dedicated `<project>--beads` repository whose bead store
lives **at the repository root** (`BEADS_DIRNAME_ROOT == "."`, clone path `<workspace>/sase/repos/beads`). All ten
phases are closed, the schema-3 store record is written, and all three enabled projects (`sase`, `actstat`, `bob-cli`)
are migrated: `sase repo path beads` resolves, the beads clones are healthy and pushed, and the plans clones no longer
carry `beads/`. `just check` passes every lint, symvision, `sase validate`, and committed-plan stage; the only failures
in a full run are three known xdist-contention flakes that pass in isolation.

The landing audit nevertheless found **three confirmed regressions** that the epic introduced and that must be fixed
before the epic can be closed. Every one of them has the same shape: code that derived a repository root or a sibling
kind root by walking **up from the bead directory**, which was correct while the bead store was a `beads/` subdirectory
of the plans clone and is wrong now that the bead store _is_ a clone root.

1. **Unpushed bead commits are no longer protected before a workspace reset (data loss).**
   `_protect_unpushed_sidecar_bead_commits` in `src/sase/axe/runner_workspace.py` guards `prepare_workspace` against
   discarding local-only bead commits. It locates the store through `_top_level_beads_dir(repo_root)`, which returns
   `repo_root / "beads"` or `None`. For the dedicated beads clone that path does not exist, so the guard returns
   immediately and `prepare_workspace` proceeds to `run_sase_hg_clean` plus a hard checkout of the default parent
   revision. The beads sidecar is reported with `auto_clone: true` once the store record names it, so
   `prepare_linked_repo_workspaces_if_needed` runs `prepare_workspace` against that clone **on every agent launch**, and
   `prepare_opened_checkout` runs it for `sase repo open beads`. Unpushed bead commits are discarded with no recovery
   ref and no warning. This protection was added by `sase-9x.3` and its only regression test
   (`tests/test_bead/test_sync_workspace_prepare_regressions.py`) exercises `beads_dirname="beads"` only, so nothing
   caught the loss.

2. **The ACE Artifacts -> Plans tab resolves `plans:` references against the wrong root.**
   `src/sase/ace/tui/widgets/artifacts/plans_data.py` derives `plans_root = beads_dir.parent`. Post-split that is
   `<primary>/sase/repos`, not the plans clone, so `plans:202607/<file>.md` resolves to
   `<primary>/sase/repos/202607/<file>.md`, which does not exist. Every epic and phase row renders "Linked plan
   unavailable: file not found."

3. **ACE scoped bead edits are never committed.** `_commit_scoped_bead_store` in
   `src/sase/ace/tui/actions/artifacts_plans.py` computes `root = beads_dir.parent` and returns early when
   `root / ".git"` is absent. Post-split `root` is `<primary>/sase/repos`, which is not a git repository, while
   `<primary>/sase/repos/beads/.git` is. A bead update submitted from the Plans tab (`_update_scoped_bead`) writes to
   disk and reports success, but is never committed and never pushed.

Fixing (2) exposes a fourth, **pre-existing** defect that must be handled in the same epic or the fix regresses a
different surface. `plans_data.py` feeds the same value to `plan_search` as `repo_root`, and `plan_search`'s
`_is_flat_plans_root` treats any root with a `plans/` subdirectory as non-flat. The plans sidecar clone carries a legacy
README-only `plans/` directory (committed long before this epic, by
`docs(sdd): document structured development layout`), so handing `plan_search` the real plans clone yields **zero**
archive matches, while today's accidental `<primary>/sase/repos` value yields 51. Setting `plans_root` correctly without
first fixing that shadowing would take the Plans-tab archive from 51 rows to 0.

Everything in this plan was confirmed by direct probe against the migrated `sase` project, not by reading alone:
`_top_level_beads_dir` returns `None` for a real root-layout store while returning the store for the legacy layout;
reference resolution returns the missing path for the derived root and the existing path for the plans clone;
`<primary>/sase/repos/.git` does not exist while `<primary>/sase/repos/beads/.git` does; and relaxing
`_is_flat_plans_root` restores 51 archive matches from the plans clone.

## Design

### The invariant these fixes restore

After `sase-a8`, a bead directory has three possible relationships to its owning git repository:

| Layout                  | `beads_dir`                       | Owning repo root   |
| ----------------------- | --------------------------------- | ------------------ |
| dedicated beads sidecar | `<workspace>/sase/repos/beads`    | the same path      |
| legacy plans-embedded   | `<plans clone>/beads`             | `beads_dir.parent` |
| in-tree / local         | `<repo>/sdd/beads`, `.sase/sdd/…` | further up         |

No caller may assume the second row. Each phase below replaces one such assumption with either an explicit root-layout
arm or a resolution through `SddStore`/`resolve_sdd_kind_dir`, which are already correct for all three.

### Phase ordering

`flatroot` must land before `plansroot`, because `plansroot` starts feeding `plan_search` the real plans clone and that
only returns results once the flat-root probe stops being shadowed. `prepare` and `acecommit` are independent of both
and of each other. `land` runs last and only after every other phase is closed.

## Phases

### Restore unpushed bead-commit protection for root-layout stores

Teach `src/sase/axe/runner_workspace.py` that a repository can itself be a bead store.

- Replace `_top_level_beads_dir(repo_root) -> Path | None` so that it returns:
  - `repo_root / "beads"` when that is a directory (unchanged legacy plans-embedded behavior), else
  - `repo_root` when `repo_root` itself holds bead state, else `None`.
- Detect "holds bead state" with the same marker set the adoption transaction already trusts: a `config.json` file or an
  `events/` directory directly under the candidate. Do **not** use the presence of `.git` alone, and do not treat a bare
  `issues.jsonl` as sufficient on its own if `config.json` is absent — mirror `src/sase/sdd/_bead_adoption.py`'s
  existing "already contains bead state" probe so the two cannot drift. Import or factor out that predicate rather than
  writing a third copy of it.
- Leave `_beads_dir_belongs_to_repo` semantics intact. For the root layout `bead_store_git_root(beads_dir)` resolves to
  `repo_root` itself, so the equality check still passes.
- `unpushed_bead_commit_count(repo_root, beads_dir)` already tolerates `beads_dir == repo_root`: `relative_pathspec`
  yields `"."` and the pathspec becomes `"./"`, which is every path in the repository. That is the correct semantics for
  a dedicated bead repository. Add a test that pins it rather than changing it.
- `push_bead_work_launch(beads_dir)` is already root-layout-safe: `_semantic_beads_dir_for_sync` maps a
  `requested == repo_root` call through `resolve_beads_dir`. No change needed; add coverage.

Tests. Extend `tests/test_bead/test_sync_workspace_prepare_regressions.py`. The existing
`test_prepare_workspace_rescues_unpushed_bead_commits_before_sidecar_reset` covers `beads_dirname="beads"`; add the
root-layout twin that builds its store with `BeadProject.init(root, beads_dirname=BEADS_DIRNAME_ROOT)` against a local
bare remote, commits a bead claim, injects a failing publish, and asserts that `prepare_workspace` still (a) attempts
the managed sync against the clone root, (b) retains a `refs/sase/recovery/` ref pointing at the pre-reset HEAD, and (c)
survives `safe_reap_sdd_recovery_snapshots`. Prefer parameterizing the existing test over duplicating it if the fixtures
allow. Add a direct unit test for `_top_level_beads_dir` covering all three layouts plus a plain repository with no bead
state (must stay `None`).

### Stop a README-only `plans/` subdirectory from shadowing a flat plans root

`src/sase/plan_search/facade.py:_is_flat_plans_root` returns `False` as soon as `(path / "plans").is_dir()`. A plans
sidecar clone keeps its `<YYYYMM>/` directories at its root _and_ carries a generated `plans/README.md` directory
README, so a real flat sidecar root is misclassified as a nested SDD root and searches nothing.

- Make the `plans/` subdirectory only disqualify a flat root when it actually looks like a plans root — i.e. when it
  contains at least one six-digit `<YYYYMM>` directory. A `plans/` directory holding only `README.md` must not
  disqualify a root whose own children include month directories.
- `src/sase/sdd/_link_files.py:list_sdd_files` has the identical
  `plans_root = root / "plans"; if not plans_root.is_dir(): plans_root = root` shadowing. Apply the same "has month
  directories" refinement there so the two probes cannot disagree. Factor the predicate into one shared helper rather
  than duplicating the glob.
- Leave `src/sase/sdd/_paths.py:sdd_kind_roots` alone. It already appends `base_dir` itself when `_has_month_dirs`, so
  it is not shadowed.
- This is a behavior fix to a pre-existing defect, deliberately scoped to the shadowing case. Do not otherwise change
  how nested SDD roots or in-tree layouts resolve.

Tests. Add `tests/test_plan_search_facade.py` coverage for a flat plans sidecar root that also contains a README-only
`plans/` subdirectory: it must be classified flat and must return the month-directory plans. Pin the converse too — a
root whose `plans/` subdirectory _does_ contain month directories must still be treated as nested. Add the mirrored
cases for `list_sdd_files` in `tests/sdd_store/test_link_files.py`.

### Resolve the ACE Plans tab's plans root through the store

Depends on `flatroot`.

`src/sase/ace/tui/widgets/artifacts/plans_data.py` currently does
`plans_root = None if beads_dir is None else beads_dir.parent`.

- Add a `project_plans_root(project) -> Path | None` next to `project_beads_dir` in
  `src/sase/ace/tui/widgets/artifacts/plans_data_sources.py`, resolving through
  `sase.sdd.store.resolve_sdd_kind_dir(<project workspace dir>, 1, "plans")` so every storage mode (sidecar_repos,
  separate_repo, local, in-tree) resolves correctly. Resolve the workspace directory the same way `project_beads_dir`
  already resolves its primary, and return `None` on failure so a project with no resolvable store degrades exactly as
  it does today rather than raising on the worker thread.
- Use it for `plans_by_project`, which flows into `_load_epic_linked_plan_documents` (reference resolution),
  `store_mtime_key` (cache invalidation), `load_project_archive`, and the `PlansSnapshot.plans_roots` map consumed by
  `plans_deep_archive.py`.
- Keep the existing `beads_dir is None` / `plans_root is None` guards and the "No bead store is available for this
  project." error path intact; a project that resolves a bead store but not a plans root must degrade gracefully, not
  crash the pane.

Verification, on a migrated project. `plans:<YYYYMM>/<file>.md` epic references must resolve to the file inside the
plans clone and render their linked plan document instead of "Linked plan unavailable: file not found.", and the
Plans-tab archive row count must not drop — with `flatroot` landed it should match or exceed the count observed before
this phase.

Tests. Add `plans_data` coverage that builds a split-layout fixture (a beads clone at `<workspace>/sase/repos/beads` and
a plans clone at `<workspace>/sase/repos/plans` with a schema-3 store record) and asserts the resolved plans root is the
plans clone and that an epic's `plans:` design reference loads its document. Pin the legacy plans-embedded layout too,
so both resolve correctly.

### Commit ACE scoped bead edits against the bead store's own repository

`_commit_scoped_bead_store(beads_dir, message)` in `src/sase/ace/tui/actions/artifacts_plans.py` derives
`root = beads_dir.parent` and constructs an `SddStore(storage=SDD_STORAGE_SEPARATE_REPO, sdd_dir=root, repo_root=root)`.
For the dedicated beads sidecar the owning repository is `beads_dir` itself, so the `(root / ".git").exists()` guard
sees no repository and the function returns without committing — silently, while `_update_scoped_bead` reports success
to the user.

- Resolve the owning repository from the bead directory instead of assuming its parent. Reuse
  `sase.bead.sync.bead_store_git_root(beads_dir)` (a thin wrapper over the shared `find_git_root`) rather than adding
  another parent-walking heuristic, and keep the "no repository -> in-tree store, committed with its owning code change"
  early return for the case where no git root is found.
- Construct the `SddStore` with `sdd_dir` and `repo_root` set to that resolved root so `commit_sdd_store_files` targets
  the right worktree, and keep `paths=[beads_dir]` so the commit stays scoped to bead state.
- The commit must reach the beads repository for a migrated project and must keep landing in the plans repository for a
  project that has not been migrated.

Tests. Add coverage in the ACE artifacts-plans test module that exercises `_commit_scoped_bead_store` against (a) a
root-layout beads clone, asserting a commit lands in that repository, (b) a legacy `<plans clone>/beads` layout,
asserting unchanged behavior, and (c) an in-tree store with no git root, asserting the early return. The existing
interaction tests patch `_update_scoped_bead` wholesale, so this needs new direct coverage of the commit helper.

### Close the epic, sweep symbols, and mark the plan done

Depends on `prepare`, `flatroot`, `plansroot`, and `acecommit`.

Run only after every other phase in this plan is closed and `just install && just check` passes. Treat the three known
xdist-contention flakes (`tests/ace/tui/test_notification_custom_gate.py`, `tests/test_suite_gate_integration.py`,
`tests/test_bead/test_cli_work_from_plan_concurrency.py`) as acceptable only after confirming each passes when rerun in
isolation.

1. Re-verify the epic end to end on a migrated project: `sase bead list` and `sase bead show` succeed; a bead write
   commits and pushes to the beads clone; deleting `sase/repos/beads` and rerunning `sase bead list` re-clones it;
   `sase repo path beads` prints the clone; `.sase/sdd-store.json` is `schema_version: 3` with a `beads` sidecar.
2. `sase bead close sase-a8`. Do not pass `--force` to make the command succeed. If the close is rejected, the named
   phases were not actually complete — finish or reopen them, or record the outcome deliberately with
   `--force --reason ... --resolution canceled|superseded`.
3. `just symvision`. Epic-symbol whitelist entries for `sase-a8` expire at close; remove whatever stale entries and
   now-unused code it reports. As of this plan there are no `sase-a8` pragmas anywhere in `src/`, `tests/`, `tools/`, or
   `docs/`, so a clean run is the expected result — investigate rather than assume if it reports otherwise.
4. Set `status: done` in the frontmatter of the epic's plan file, `plans:202607/beads_sidecar_repo.md`.
5. Set `status: done` in the frontmatter of this plan file.

## Follow-ups deliberately excluded from this epic

These are recorded so the landing agent reports rather than silently drops them.

- **Publish `sase-core` and raise the floor.** Phase `sase-a8.1` landed as `sase-core` commit
  `feat(bead): recognize the beads sidecar root in path heuristics (sase-a8.1)`, which is **after** the `v0.12.1` tag;
  release PR `sase-org/sase-core#39` (`chore: release v0.12.2`) is open and unmerged. This repository still pins
  `sase-core-rs>=0.12.1,<0.13.0`, so the installed binding lacks the `["beads","repos","sase"]` arm. Probing the
  installed 0.12.1 against a real root-layout store showed `plan(<path>)` design references still canonicalizing to
  `plans:<YYYYMM>/<file>.md` from the workspace root and from nested working directories, so nothing is observably
  broken today. Raise the floor after #39 merges, following the `sase-a0.5.2` pattern. This is a separate bead, not a
  phase here, because it is gated on an external release.
- **Two memory files still describe the pre-split layout.** `sase/memory/glossary.md` defines an SDD sidecar repo as
  `<project>--plans` or `<project>--research` and omits `<project>--beads`; `sase/memory/build_and_run.md` still points
  bead changes at `sdd/beads/`. The `sase-a8` plan's docs phase deliberately deferred both, and `CLAUDE.md` forbids
  editing `sase/memory/*.md` without explicit user permission in the conversation that makes the edit. **Do not edit
  them as part of this epic.** Report them to the user and let them decide.
