---
create_time: 2026-07-09 03:10:56
status: done
prompt: 202607/prompts/sdd_artifact_repo_migration.md
tier: tale
---
# Plan: Move bob-cli and actstat SDD Artifacts to Companion Repos

## Goal

Migrate the SASE SDD artifacts for `bob-cli` and `actstat` out of tracked in-tree `sdd/` directories and into
project-specific GitHub companion repositories:

- `bobs-org/bob-cli--sdd`
- `bbugyi200/actstat--sdd`

After migration, each primary repo should use:

```yaml
sdd:
  storage: separate_repo
```

The companion repos should contain the SDD artifacts, generated SDD guide files, and bead event/config files that belong
under separate-repo storage. Local `.sase/sdd-store.json` records should point at the project-specific companion repos,
not at the old org-level `owner/sdd` fallback.

## Current State From Read-Only Discovery

- Both primary repos are clean on `master`.
- Both primary repos currently have `sase.yml` with `sdd.version_controlled: true`.
- Both primary repos have tracked in-tree `sdd/` artifacts.
- Neither desired companion repo currently exists on GitHub.
- The authenticated GitHub CLI has repo scope and can create/push the companions.
- `actstat` has a simple migration shape: tracked in-tree `sdd/`, no existing `.sase/sdd` local repo.
- `bob-cli` has a mixed migration shape:
  - 562 tracked files under in-tree `sdd/`.
  - An ignored local `.sase/sdd` git repo with two unique historical artifacts for `bob_cli_migration_1` and
    `bob_cli_migration_2`.
  - A stale negative `.sase/sdd-store.json` record pointing at `bobs-org/sdd`.
  - Overlapping bead files differ between in-tree `sdd/` and `.sase/sdd`; the in-tree bead config is newer and should
    win for overlapping bead metadata.
- Read-only `sase init --check` currently plans separate-repo SDD setup, but in `bob-cli` it displays the stale cached
  target `bobs-org/sdd`. That is misleading now that GitHub defaults to `<owner>/<repo>--sdd`.
- Plain SDD init is a bootstrap path. Existing in-tree SDD artifacts must be migrated, not stranded behind a new empty
  `.sase/sdd` store.

## SASE Fixes Before Migrating Repos

1. Make SDD init planning stop reusing stale `not_found` store-record repo names.
   - The run path already deletes negative records before creating/materializing a companion.
   - The read-only plan should match that behavior and either show the current provider-derived target or a generic
     "GitHub companion SDD repository" action.
   - Add a regression test using a cached negative record such as `acme/sdd` and assert the plan does not promise that
     stale target.

2. Make the separate-repo init path migration-aware for existing in-tree SDD content.
   - When effective storage is `separate_repo` and the project still has existing in-tree SDD artifacts, `sase init sdd`
     should preserve those artifacts by reusing the migration helper instead of only creating fresh guide files in
     `.sase/sdd`.
   - Keep primary-repo cleanup explicit. `sase init sdd` may copy/adopt artifacts into the companion, but removing
     tracked `sdd/` from the primary repo should remain part of the explicit migration workflow.
   - Add tests for the transition from legacy `sdd.version_controlled: true` to `sdd.storage: separate_repo` with
     existing in-tree files.

3. Harden bead-store ignore handling for separate/local SDD stores.
   - Ensure `.gitignore` for `.sase/sdd` includes SQLite sidecars as well as the database itself: `beads/beads.db`,
     `beads/beads.db-shm`, and `beads/beads.db-wal`.
   - Update existing helper paths that create or refresh the non-version-controlled bead store so old `.gitignore` files
     are amended rather than left stale.
   - Add focused tests around materialization/migration so SHM/WAL files do not become tracked again.

4. Verification for SASE changes:
   - Run `just install` first, per repo instructions.
   - Run focused SDD init/migration/bead tests.
   - Run `just check` in the SASE repo before moving on to the target repos.

## Migration Procedure

1. Use the refreshed SASE development environment for the migration commands.
   - Do not rely on a stale globally installed `sase` entry point until it is known to include the SASE and
     `sase-github` fixes above.
   - Verify `sase version -j` sees the updated GitHub workspace provider before touching target repo files.

2. Migrate `actstat`.
   - Run the SDD migration path with companion creation enabled.
   - Expected companion: `bbugyi200/actstat--sdd`.
   - Verify `.sase/sdd-store.json` records `repo: bbugyi200/actstat--sdd` and the SSH remote points at that repo.
   - Verify `.sase/sdd` contains all prior in-tree SDD markdown/artifact files except intentionally ignored bead
     database files.
   - Remove tracked `sdd/` from the primary repo after the companion content is verified.
   - Keep `sase.yml` with `sdd.storage: separate_repo`.

3. Migrate `bob-cli`.
   - Snapshot the in-tree-vs-local SDD file comparison before running the migration.
   - Run the SDD migration path with companion creation enabled.
   - Expected companion: `bobs-org/bob-cli--sdd`.
   - Preserve the union of content:
     - In-tree `sdd/` wins for overlapping bead metadata.
     - Unique ignored local `.sase/sdd` artifacts for `bob_cli_migration_1` and `bob_cli_migration_2` are retained in
       the companion repo.
   - Verify the stale `bobs-org/sdd` negative record is gone and the new store record points at `bobs-org/bob-cli--sdd`.
   - Remove tracked `sdd/` from the primary repo after the companion content is verified.
   - Keep `sase.yml` with `sdd.storage: separate_repo`.

4. Primary repo commit boundaries.
   - Companion SDD repos will need their migration commits pushed as part of becoming usable separate stores.
   - Primary repo changes should be easy to review: `sase.yml` update plus removal of tracked `sdd/`.
   - If commit authorization is provided after plan approval, use the SASE commit workflow for primary repo commits.
     Otherwise leave primary repo changes uncommitted with clear status output.

## Validation

For SASE:

- `just install`
- focused tests covering SDD init planning, migration, and non-version-controlled bead store ignore rules
- `just check`

For each target repo:

- `gh repo view <owner>/<repo>--sdd`
- `git -C .sase/sdd remote -v`
- `git -C .sase/sdd status --short`
- `sase sdd path`
- `sase sdd list --kind all`
- `sase sdd validate`, reporting any pre-existing historical warnings/errors rather than hiding them
- primary repo `git status --short`

Rust project checks:

- `bob-cli`: `just all`, plus `just check-scripts` if `just all` does not cover script syntax checks
- `actstat`: `just check`

## Rollback And Failure Handling

- If companion creation succeeds but later clone/bootstrap/push fails, rerun the migration after fixing the reported
  issue. The migration path is designed to adopt existing materialized records and remotes.
- Do not remove primary `sdd/` until the companion repo content, remote, and store record have been verified.
- For `bob-cli`, do not delete the ignored local `.sase/sdd` history until the unique local artifacts are visible in the
  companion repo.
- If validation reveals historical SDD link problems unrelated to this storage migration, report them separately instead
  of mixing link repairs into this change.
