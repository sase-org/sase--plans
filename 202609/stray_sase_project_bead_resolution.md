---
tier: tale
title: Fix bead list shadowed by an auto-created stray sase project
goal: sase bead list works from the primary sase checkout, and
size: medium
proposed_by: bbugyi200.athena.c
status: done
---

# Fix `sase bead list` finding no beads from the primary `sase` checkout

## Problem

Running `sase bead list` from the primary checkout `~/projects/github/sase-org/sase`
prints `No issues found.` Running it from any managed `sase_<N>` workspace lists the
beads normally, but prints
`Ignoring conflicting PROJECT_NAME 'sase' for project 'gh_sase-org__sase'` warnings.

## Root cause (verified)

1. **A stray bare-git project named `sase` was auto-created on 2026-09-12 17:36:01 local
   time.** These all have that exact creation time:
   - `~/.sase/projects/sase/sase.sase` contains only
     `BARE_REPO_DIR: ~/.sase/repos/sase.git` and `WORKSPACE_DIR: ~/projects/git/sase/`.
   - `~/.sase/repos/sase.git` holds a single `Initial commit` by
     `sase <sase@localhost>`, whose only content is the SDD scaffold.
   - `~/projects/git/sase` is a clone of that bare repo.
   - `~/.local/state/sase/workspaces/sase-03e92f89/` holds a claim registry and runner
     workspaces `sase_10`, `sase_15`, `sase_17`, `sase_22` and `sase_32`.

   These are the default paths and default identity used by `init_bare_git_project()` in
   `src/sase/workspace_provider/plugins/bare_git_init.py`. The creation path is
   `resolve_git_ref("sase")` in `src/sase/workspace_provider/plugins/bare_git_ref.py`:
   - Mode 1 only checks whether the directory `~/.sase/projects/sase/` exists.
   - It never consults the project alias/display-name registry, where `sase` is the
     `PROJECT_NAME` of the real project `gh_sase-org__sase`.
   - So a `#git:sase` reference falls through to Mode 3, `_init_missing_project_ref()`,
     which silently creates a brand-new project that shadows the real one. Mode 4, the
     bare path, has the same gap.
   - The run that issued the `#git:sase` reference was not recorded in
     `~/.sase/logs/runs.jsonl`; project resolution fails before a run record is written.
     The mechanism is certain from the artifacts above.

2. **The shadow project hijacks project-name resolution.** Once a real project directory
   is named `sase`:
   - `load_project_alias_map` (`src/sase/project_alias_records.py`) drops
     `gh_sase-org__sase`'s `PROJECT_NAME: sase`, which is what prints the "Ignoring
     conflicting PROJECT_NAME" warning.
   - `resolve_project_alias_ref("sase")` now returns the stray `sase`.
   - Consequences already observed:
     - Audit logs from real `gh_sase-org__sase` agents land in
       `~/.sase/projects/sase/*.jsonl`.
     - Sidecar paths for real workspaces resolve under the stray `sase-03e92f89` key.

3. **SDD store resolution disagrees with bead-location resolution in the primary
   checkout.** `resolve_beads_location()` in `src/sase/bead/cli_location.py` correctly
   computes this context via `scan_projects_for_cwd`:
   `primary=~/projects/github/sase-org/sase`, project `gh_sase-org__sase`. It then calls
   `resolve_sdd_store(context.root, 1)`, which re-derives the primary on its own.

   `get_primary_workspace_dir()` → `resolve_primary_from_project()` in
   `src/sase/sdd/_paths.py` works like this:
   - It asks the workspace-name hook for a name, and the hook returns the bare basename
     `sase`.
   - It opens `~/.sase/projects/sase/sase.sase` and returns its `WORKSPACE_DIR`,
     `~/projects/git/sase`, without checking that this primary owns the cwd at all.
   - That directory has no `.sase/sdd-store.json` record, so storage falls back to the
     GitHub provider policy `separate_repo`.
   - That selects `<checkout>/.sase/sdd/beads`. This is a leftover git repo from
     2026-08-06 with no commits, no remote and an empty `issues.jsonl`.

   Managed `sase_<N>` workspaces are unaffected only because their `.sase/checkout.json`
   marker short-circuits `get_primary_workspace_dir()`.

Verified by simulation from the primary checkout. Patching
`resolve_primary_from_project` to return `None` makes `resolve_beads_location()` return
the `sase/repos/beads` sidecar (`storage=sidecar_repos`), with 303 open beads.

## Changes

### 1. Stop `#git:` refs and bare-git init from shadowing existing projects

In `src/sase/workspace_provider/plugins/bare_git_ref.py`:

- In `resolve_git_ref()`, Mode 1, before concluding a shorthand ref is missing, check
  whether another project claims the ref as its `PROJECT_NAME` or alias. Use the
  existing `find_project_ref_owner()` / `resolve_project_alias_ref()` helpers in
  `src/sase/project_aliases.py`; confirm their exact semantics first.
  - If a claiming project exists and is bare-git (it has `BARE_REPO_DIR`), resolve it as
    Mode 1 against that canonical project.
  - Otherwise raise `build_provider_mismatch_error(<canonical>, <its spec path>)`.
  - A claimed ref must never reach Mode 2 or Mode 3 initialization.
- Apply the same claimed-name guard to the project name derived in Mode 4.

In `src/sase/workspace_provider/plugins/bare_git_init.py`, `init_bare_git_project()`:

- Add a defensive guard next to the existing provider-mismatch guard, before any
  filesystem or git work.
- Refuse to initialize a project whose name is already claimed as another project's
  `PROJECT_NAME` or alias.
- The error must name the owning project key and suggest its VCS tag.
- This also protects every other caller of `init_bare_git_project`; grep for them.

### 2. Make SDD primary resolution agree with the bead-location context

- In `src/sase/sdd/_paths.py`, `resolve_primary_from_project()`:
  - Canonicalize the hook-derived name through the alias registry.
  - Only return the configured `WORKSPACE_DIR` when that primary actually owns
    `workspace_dir`. Reuse the ownership rule already used by `scan_projects_for_cwd`
    (`_cwd_matches_project_workspace` in `src/sase/bead/project_name.py`) so that
    adjacent `<primary>_<N>` variants keep working. Promote it to a non-private helper
    if Symvision requires that.
  - When the configured primary does not own the cwd, try the `scan_projects_for_cwd`
    primary next, then the existing suffix-stripping fallback.
- Thread an optional `primary_workspace_resolver` keyword through the public wrappers
  `resolve_sdd_store()` and `materialize_sdd_store()` in `src/sase/sdd/store.py`, and
  through `src/sase/sdd/_store_materialization.py` if it re-derives the primary. Default
  to `get_primary_workspace_dir` so existing callers are unchanged.
- In `resolve_beads_location()` (`src/sase/bead/cli_location.py`), pin that resolver to
  `context.primary`. Bead store resolution must then use the same primary the context
  already computed, and cannot diverge again.

### 3. Surface the collision in `sase doctor`

- Expose the dropped/conflicting refs from the alias-map build as data, not only log
  warnings: a small helper in `src/sase/project_alias_records.py` or
  `src/sase/project_aliases.py`.
- Add a check in `src/sase/doctor/checks_project.py`, registered in
  `project_check_specs`, that warns when a project directory key collides with another
  project's `PROJECT_NAME` or alias. Report:
  - both project keys;
  - both `WORKSPACE_DIR`s;
  - whether the key-owner is a bare-git project whose `WORKSPACE_DIR` sits under
    `~/projects/git/`, the auto-init signature.
- Next step text: quarantine the accidental project.
- If project-record or alias logic already lives in `../sase-core`, respect the Rust
  core boundary. Add the conflict data there and keep only the Python rendering here.

### 4. Tests

All tests must use sandboxed HOME / project roots, never the real `~/.sase`.

- `resolve_git_ref` tests (e.g. `tests/test_bare_git_workspace.py`; grep for existing
  `resolve_git_ref` coverage):
  - Setup: a GitHub-style project `gh_org__sase` with `PROJECT_NAME: sase`.
  - `resolve_git_ref("sase")` raises a provider-mismatch error naming the canonical
    project.
  - It creates no `projects/sase/`, no bare repo and no clone.
  - A bare-git project referenced by its alias resolves to the canonical project.
- `tests/test_bare_git_init.py`: `init_bare_git_project("sase")` is refused when `sase`
  is another project's `PROJECT_NAME` or alias, and leaves no filesystem residue.
- `tests/test_sdd_paths.py`:
  - A stale project spec named like the workspace-name hook result, whose
    `WORKSPACE_DIR` points elsewhere, no longer wins.
  - `get_primary_workspace_dir(<primary checkout>, 1)` returns the owning checkout.
  - Adjacent `_<N>` variants still resolve.
- `tests/test_bead/test_cli_resolution.py`, end-to-end reproduction:
  - Setup: a canonical project with a `sidecar_repos` store record and a split beads
    sidecar at `<primary>/sase/repos/beads`.
  - Setup: a conflicting stray `sase` bare-git project spec pointing elsewhere.
  - Setup: an empty `<primary>/.sase/sdd/beads` directory.
  - From the primary checkout, `resolve_beads_location(require_existing=True)` returns
    the sidecar beads dir with `storage == "sidecar_repos"`.
- `tests/doctor/test_checks_project.py`: the new collision check warns for the
  conflicting setup and passes for a clean one.

### 5. Repair the host state

Do this after the code changes. Move things aside; do not delete.

Put everything under one backup directory, e.g.
`~/.sase/trash/stray-sase-project-20260913/`, and record what was moved in a
`README.txt` there.

1. Safety checks:
   - `pgrep -af 'sase-03e92f89|projects/git/sase( |/|$)'` returns nothing.
   - No running agent holds a claim in
     `~/.local/state/sase/workspaces/sase-03e92f89/registry.json`; check
     `sase agent list`.
   - `git -C ~/.sase/repos/sase.git log --all` shows only the SASE `Initial commit`.
   - `git -C ~/projects/git/sase status --porcelain` is clean.
   - If any check fails, stop and report instead of moving anything.
2. Preserve audit history. For each of `artifact_reads.jsonl`, `memory_reads.jsonl`,
   `repo_opens.jsonl` and `skill_uses.jsonl` in `~/.sase/projects/sase/`, append its
   lines to the same-named file in `~/.sase/projects/gh_sase-org__sase/`. Hold the
   sibling `<name>.lock` with `flock` while appending. These are append-only audit logs,
   and every entry was written by real `gh_sase-org__sase` agents.
3. Move into the backup directory:
   - `~/.sase/projects/sase/`
   - `~/.sase/repos/sase.git`
   - `~/projects/git/sase`
   - `~/.local/state/sase/workspaces/sase-03e92f89/`
4. In `~/projects/github/sase-org/sase/.sase/sdd`, verify all of the following, then
   move that directory into the backup as `primary-checkout-dot-sase-sdd`:
   - `git log` reports no commits;
   - `git remote -v` is empty;
   - `beads/issues.jsonl` is empty.

   Leave `.sase/sdd.post-split-legacy` and `.sase/sdd-store.json` untouched.

## Verification

1. Read the `lint_and_test` reference memory with `sase memory read` and follow it;
   `just check` must pass.
2. From `~/projects/github/sase-org/sase`, all of these must hold:
   - `sase bead list` lists the open beads, not `No issues found.`, and prints no
     "Ignoring conflicting PROJECT_NAME" warning.
   - `sase project list` shows a single `sase` entry, `gh_sase-org__sase`.
   - `sase project current` reports `gh_sase-org__sase`.
   - `sase doctor` reports no project-name collision.
3. From a managed `sase_<N>` workspace:
   - `sase bead list` still works, with no warning.
   - `resolve_sdd_store(...)` sidecar dirs (e.g. `agents`) resolve under the
     `gh_sase-org__sase` project key, not `~/.sase/projects/sase/`.

## Out of scope

- Identifying the exact caller that issued `#git:sase`. The guards above make any such
  caller fail loudly instead of creating a shadow project.
- Linked-repo ambiguity for `sase-core` between `~/projects/git/sase-core` and
  `~/projects/github/sase-org/sase-core`, seen in an agent's `sase repo open` output. If
  it reproduces after this fix, file it as a separate bead.
