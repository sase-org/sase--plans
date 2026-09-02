---
tier: tale
title: Auto-connect the SDD store on a project's first agent launch per machine
goal:
  First launches of sase-managed projects on a new machine succeed by auto-connecting to
  existing SDD sidecar repositories, and any residual SDD materialization failure tells
  the user to run `sase repo init`.
size: medium
proposed_by: bbugyi200.kellys_mbp.6.f0
---

- **AGENTS:**
  - [bbugyi200.kellys_mbp.6.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.kellys_mbp.6.f0.md)
- **COMMITS:**
  - [58d3ed7](https://github.com/sase-org/sase/commit/58d3ed746e37f1f02c19f0d4e4db95f8ef7dece9)
    — feat(sdd): auto-connect existing sidecar stores on first launch

# Plan: Auto-Connect The SDD Store When A Project Launches Its First Agent On A Machine

## Problem

Launching the first sase agent against a sase-managed project that has never been
initialized on the current machine fails hard during workspace prep:

```
SddMaterializationError: could not create workspace SDD sidecar clone at
<workspace>/.sase/sdd
```

Reproduced today with `#gh:bobs-org/bob-cli`: the `#gh:owner/repo` first-use flow
(sase-github `_resolve_repo_path_ref`) cloned the primary checkout to
`~/projects/github/bobs-org/bob-cli/` and registered the project, but nothing
initialized the SDD store. The user's suspicion is confirmed: running `sase repo init`
in the primary checkout first would have prevented the failure, and `sase repo init` has
no programmatic callers — it is only ever run by hand — so every sase-managed project
hits this on its first launch per machine.

## Root Cause

1. `prepare_launch_workspace_repos` (`src/sase/axe/runner_workspace.py`) calls
   `ensure_workspace_sdd_clone(workspace_dir, workspace_num, strict=workspace_num > 1)`
   for numbered workspaces. This path only _syncs from an existing record_; it never
   materializes a store.
2. With no record at the primary, `resolve_sdd_store`
   (`src/sase/sdd/_store_resolution.py`) falls back to the workspace provider's
   `sdd_storage_policy` (`separate_repo` for GitHub) with `remote_url=None`.
3. `ensure_workspace_sdd_clone` (`src/sase/sdd/_store_link.py`) then finds no primary
   clone at `<primary>/.sase/sdd` to clone from and no remote URL to fall back to, and
   strict mode raises the opaque `SddMaterializationError` above.

In the bob-cli case the GitHub org already hosted `bob-cli--plans`, `bob-cli--beads`,
`bob-cli--agents`, and `bob-cli--research` (initialized from another machine in July
2026), and the project's `sase/sase.yml` declares `is_sase_managed: true` plus those
builtin/custom sidecars. Pure read-only _discovery_ followed by connect would have
succeeded — no remote creation was needed.

## Fix Overview

Two changes, both host-side (the axe runner runs before the agent starts, so this does
not conflict with single-turn-agent or host-owned-completion rules):

1. **Connect-only auto-init at launch.** When launch workspace prep finds a sase-managed
   project with no materialized SDD record at the primary, run the same
   discovery+connect flow `sase repo init` uses — but strictly non-interactive and
   _never creating remote repositories_. If every required sidecar already exists
   remotely (the "existing project, new machine" case), the launch proceeds exactly as
   if `sase repo init` had been run. If a required sidecar repo does not exist, fail
   with an actionable error instead of the opaque one.
2. **Actionable strict-failure message.** When the strict workspace SDD clone still
   fails because the primary has no materialized record, the error must name the real
   problem ("this project's SDD store has never been initialized on this machine") and
   the remedy (`sase repo init` in the primary checkout; `is_sase_managed: true` when
   the repo is unmanaged).

No feature flag: the replaced behavior is a hard launch failure, so no backward-compat
branch must stay reachable. No Rust-core crossing: the SDD store subsystem and the axe
runner are both Python in this repo today.

## Implementation Steps

### 1. Extract a reusable connect-only auto-init helper

Add a helper (suggested: `src/sase/sdd/_auto_init.py`, exported through
`sase.sdd.store`) with roughly this contract:

```python
def auto_connect_sdd_store(workspace_dir: str | Path, workspace_num: int) -> bool:
    """Connect this machine to the project's existing SDD store, creating nothing.

    Returns True when a store record is materialized (pre-existing or newly
    connected). Returns False (no-op) when the project is not sase-managed, is not
    a project directory, or the provider policy is not remote-backed. Raises
    SddMaterializationError with a `sase repo init` remedy when a required sidecar
    repository does not exist remotely.
    """
```

Behavior, mirroring `run_repo_init` (`src/sase/main/repo_init_handler.py`) minus every
interactive and project-repo-mutating step:

- Resolve the primary via `get_primary_workspace_dir`. If
  `is_materialized_record(read_sdd_store_record(primary))` → return True (no-op).
- If the primary is not a project directory, or `project_management_status` says the
  repo is not sase-managed, or the provider SDD policy is not remote-backed
  (`separate_repo` / `sidecar_repos`) → return False (leave current behavior).
- Otherwise run the configured-sidecars flow against the _primary_:
  - `configured_sidecar_specs(primary)` (today in `src/sase/main/_repo_init_config.py`;
    either import it from there — `axe` already imports `sase.main` elsewhere — or move
    it into `sase/sdd/_sidecar_init.py` and re-export; prefer the move only if it does
    not create an import cycle).
  - `preflight_sidecars(primary, 1, specs)` (read-only discovery).
  - Any preflight `unavailable`/invalid → raise `SddMaterializationError` (same as
    `run_configured_sidecars` in `src/sase/main/_repo_init_sidecars.py`).
  - Roles with status `not_found`:
    - the agents role (`AGENTS_SIDECAR_ROLE`): drop it and log/print a warning that
      `sase repo init` must be run interactively to create it (mirrors the existing
      non-interactive repo-init behavior).
    - any other role: raise `SddMaterializationError` telling the user to run
      `sase repo init` in `<primary>` to create the missing `<provider>` sidecar
      repository `<repo>`. Never create remote repos from the launch path.
  - `initialize_sidecars(primary, 1, selected_specs, creation_authorized={}, publish_sidecar_changes=True)`
    — connect-only; `creation_authorized={}` guarantees nothing is created even if a
    provider misreports. `publish_sidecar_changes=True` matches `sase repo init`
    defaults and only pushes seeded guide files to the _sidecar_ repos, never the
    project repo.
- Explicitly out of scope for the helper (these stay `sase repo init`-only): writing
  `repos.sidecar` config into `sase/sase.yml`, updating the project `.gitignore`, and
  committing anything to the project repository.
- Concurrency: `initialize_sidecars` already serializes on
  `materialization_lock(primary)`; re-check the record after acquiring it so two
  simultaneous first launches settle to one connect and one no-op.

### 2. Call the helper from launch workspace prep

In `prepare_launch_workspace_repos` (`src/sase/axe/runner_workspace.py`, currently
around the `ensure_workspace_sdd_clone` call at line ~437): before the strict
`ensure_workspace_sdd_clone`, call
`auto_connect_sdd_store(workspace_dir, workspace_num)`. When it connects a store, print
a short status line consistent with the surrounding `=== ... ===` progress output (e.g.
"Connected existing SDD sidecars for first use on this machine"). Let its
`SddMaterializationError` propagate — it is now actionable.

### 3. Make the residual strict failure actionable

In `ensure_workspace_sdd_clone` (`src/sase/sdd/_store_link.py:198-213`) — or, if
cleaner, wrapped at the `_store_workspace.py` / runner call site — when the strict clone
fails _and_ the primary has no materialized record, replace the message "could not
create workspace SDD sidecar clone at <path>" with one that states:

- the project has no materialized SDD store on this machine,
- the primary checkout path,
- the remedy: run `sase repo init` in that primary checkout, and
- when the repo is not sase-managed, that `is_sase_managed: true` must be set in the
  target repository's `sase/sase.yml` first.

Keep the existing message for genuine clone failures (record present but clone/remote
operations failed).

### 4. Tests

Use the existing fake workspace-provider test infrastructure (see the sidecar-init and
store-materialization tests around `src/sase/sdd/` / `tests/` introduced with provider
SDD materialization and sidecar init):

- Helper unit tests:
  - materialized record already present → True, no provider calls.
  - unmanaged repo / non-project dir / non-remote-backed policy → False, no-op.
  - no record, all configured sidecars `found` → record written, sidecars connected, no
    creation call made.
  - no record, non-agents role `not_found` → `SddMaterializationError` naming
    `sase repo init` and the primary path; nothing created.
  - no record, only agents role `not_found` → agents skipped with warning, remaining
    roles connected, record written.
- Launch-path test: `prepare_launch_workspace_repos` on a numbered workspace whose
  primary lacks a record triggers the helper before the strict clone (e.g. monkeypatch
  the helper and assert ordering, or drive the fake provider end-to-end).
- Error-message test: strict failure with no primary record produces the new actionable
  message; strict failure with a record keeps the old message.

## Verification

- `just install` first if the workspace's virtualenv is stale, then run `just check`
  inline (whole-repo lint gates + diff-scoped tests) and fix everything it reports.
- `just check-full` is the landing gate and is handled by the host/monitor flow; do not
  run it inline.

## Risks / Notes

- Projects whose remote hosts only a legacy `<repo>--sdd` repository (never migrated to
  sidecar repos) will preflight `not_found` for `plans` and get the actionable
  `sase repo init` error — identical to what interactive `sase repo init` would require
  today, so this is acceptable and strictly better than the opaque failure.
- Auto-connect publishes nothing to the project repository and creates no remote
  repositories, so a first launch against a clone of someone else's sase-managed repo
  cannot create repos in the user's namespace; it either connects to existing sidecars
  the user can read or fails with the actionable message.
- `configured_sidecar_specs` injects default builtin sidecars, so the helper works for
  managed projects even when `sase/sase.yml` omits an explicit `repos.sidecar` block.
