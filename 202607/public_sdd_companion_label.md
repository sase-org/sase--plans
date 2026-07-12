---
create_time: 2026-07-08 21:56:03
status: done
prompt: 202607/prompts/public_sdd_companion_label.md
tier: tale
---
# Plan: Public SDD companion repos and idempotent `sase--sdd` GitHub label

## Goal

Follow up on automated GitHub SDD companion repository creation with two policy improvements:

- Newly-created SDD companion repositories should be **public**, not private.
- The selected companion repository should have a GitHub label named **`sase--sdd`**. This label should be checked every
  time an explicit create/verify flow runs, including when the `<project>--sdd` repository already exists or a
  materialized store record already exists locally. If the label was deleted or never created, the flow should recreate
  it.

Assumption: "this repo" means the SDD companion repository that SASE creates/connects for `separate_repo` storage
(`owner/<project>--sdd` by default, or the configured/fallback companion repo), not the source project repository.

## Current grounding

- Main repo separate-repo init runs through `src/sase/main/sdd_handler.py::_run_separate_repo_sdd_init`, which calls
  `sase.sdd.store.create_and_materialize_sdd_store(project_root, 1)`.
- `src/sase/sdd/store.py::create_and_materialize_sdd_store` currently has an early fast path for an existing
  materialized `.sase/sdd-store.json` record. That path refreshes/clones/bootstrap-checks the local SDD store but does
  **not** call the provider again.
- GitHub-specific naming, probing, creation, and cloning live in the linked `sase-github` plugin, specifically
  `src/sase_github/workspace_plugin.py`.
- The plugin already creates missing companion repos with:
  `gh repo create <owner>/<repo> --private --description "SDD companion repository for <source-owner>/<source-repo>"`.
- The plugin's create hook, `ws_create_sdd_remote`, already verifies existing repos and creates missing repos; it is
  reached from explicit init/migration flows, not from broad setup-time `ws_materialize_sdd_store`.
- Local `gh label create --help` confirms the GitHub CLI supports idempotent create/update via
  `gh label create <name> --force --description ... --color ... --repo OWNER/REPO`.

## Design decisions

1. **Creation visibility changes to public.** Replace `--private` with `--public` for new GitHub SDD companion repos. Do
   not automatically convert existing private companion repos to public in this change; changing visibility on an
   existing repository is a stronger privacy mutation than "created as public" and should remain explicit unless the
   product decision changes.

2. **Label belongs to the companion repo SASE actually selects.** Add/refresh `sase--sdd` on the repo returned by
   `ws_create_sdd_remote`: the default `<owner>/<project>--sdd`, the `sdd.repo.name` override, or the legacy/fallback
   `<owner>/sdd` repo if that is the chosen companion.

3. **Label enforcement is explicit-command behavior.** `ws_create_sdd_remote` should ensure the label on every
   successful `found` or `created` result. Setup-time `ws_materialize_sdd_store` should stay discovery/clone-only and
   should not mutate GitHub from background agent startup.

4. **Existing materialized records still get checked.** Change the main repo's `create_and_materialize_sdd_store`
   existing-record fast path so explicit `sase sdd init` / `sase init` still asks the provider to verify the recorded
   companion repo and refresh provider metadata, which now includes the label. This is the piece that satisfies "even if
   the `<project>--sdd` repo already exists" after the project has already been initialized once.

5. **Label failures should be user-visible in explicit flows.** If `gh label create --force` fails due to auth, network,
   missing `gh`, or permission errors, `sase sdd init` should fail with an actionable message. If the repo was just
   created before the label step failed, rerunning `sase sdd init` should adopt the existing repo and retry the label
   check.

## Implementation

### A. `sase-github` plugin

File: linked repo `sase-github`, `src/sase_github/workspace_plugin.py`. Open it with:

```bash
sase workspace open -p sase-github -r "public SDD companion repos and label enforcement" <workspace_num>
```

1. Add small constants near the SDD companion constants:
   - `_SDD_COMPANION_LABEL = "sase--sdd"`
   - `_SDD_COMPANION_LABEL_DESCRIPTION = "SASE SDD companion repository"`
   - `_SDD_COMPANION_LABEL_COLOR = "<chosen 6-char hex>"`

2. Change `_create_github_sdd_repo(...)` from `--private` to `--public`. Keep the description argument and the existing
   "already exists, re-probe, then adopt" behavior.

3. Add `_ensure_github_sdd_label(host, repo_full_name) -> None`:
   - Run `gh label create sase--sdd --repo <repo_full_name> --description ... --color ... --force`.
   - Use the same non-interactive `gh` environment pattern as repo probing/creation, including `GH_HOST`.
   - Classify `FileNotFoundError`, timeout/network, auth, and generic non-zero failures into actionable `RuntimeError`
     messages.
   - Include repo name in generic failures and mention that write/label-management permissions may be missing.

4. Call `_ensure_github_sdd_label(...)` in `ws_create_sdd_remote` after choosing the target repo and before returning a
   successful record:
   - Existing repo found through candidate discovery.
   - Missing repo created successfully.
   - Name-taken race/pre-existing repo adopted after `gh repo create` reports an existing name.
   - Explicit target repo path added in Workstream B below.

5. Teach `ws_create_sdd_remote` to honor an optional exact repo target supplied by the main repo for existing
   materialized records, for example `options["sdd_repo"] = "owner/repo"`.
   - When this option is present, probe that exact repo instead of rediscovering candidates. This prevents a project
     with an existing fallback/override record from accidentally switching to a different candidate during a label
     refresh.
   - If the exact repo is found, ensure the label and return `created=False`.
   - If it is missing and `options["create"]` is true, create it public, ensure the label, and return `created=True`.
   - If it is unavailable, use the same actionable create-path errors.

### B. Main repo create-aware init path

Files: `src/sase/sdd/store.py` and focused tests.

1. Extend the private dispatch helper for provider create/verify so it can pass an optional existing store record to
   plugins, e.g. by adding `sdd_repo`, `sdd_host`, and/or `sdd_remote_url` to the `options` mapping when an existing
   materialized record is present.

2. Update `create_and_materialize_sdd_store(...)`:
   - If a materialized record already exists, call the provider create/verify dispatch with the existing record before
     finalizing the local store.
   - If the provider returns a record, finalize using that record.
   - If no provider claims the workspace, preserve the existing fast-path behavior for non-GitHub or older provider
     scenarios.
   - If the GitHub provider raises an error while verifying or ensuring the label, surface it as
     `SddMaterializationError`, as current create failures do.

3. Keep `materialize_sdd_store(...)` unchanged for setup-time materialization. It should not create repos or labels.

### C. Tests

In the linked `sase-github` repo:

- Rename/update the existing create test from "private" to "public" and assert `gh repo create ... --public`.
- Assert `--private` is no longer present in create calls.
- Add/extend tests that `gh label create sase--sdd --force ... --repo acme/widget--sdd` is called when:
  - The repo already exists.
  - The repo is newly created.
  - `gh repo create` reports "already exists" and the follow-up probe adopts it.
  - An exact `options["sdd_repo"]` target is provided.
- Add error-path coverage for label creation failures, including auth/network/tool-missing or a generic permission
  error.
- Confirm `ws_materialize_sdd_store` still does **not** call `gh label create`.

In the main repo:

- Add/adjust `tests/sdd_store/test_materialize.py` coverage so an existing materialized store record still dispatches
  provider create/verify with the recorded repo metadata during `create_and_materialize_sdd_store`.
- Add/adjust `tests/main/test_init_sdd_plan.py` or handler-level tests only if needed to verify the explicit init path
  keeps surfacing provider failures cleanly; planner behavior should not need network/label awareness.

### D. Docs

Update user-facing docs where companion repo creation is described:

- `docs/sdd_storage.md`
- `docs/init.md`
- `docs/sdd.md`
- `docs/configuration.md`

Docs should say GitHub SDD companion repos are created public by default and SASE ensures a `sase--sdd` GitHub label on
the selected companion repo during explicit init/migration verification. Keep wording clear that existing private repos
are not automatically made public by this change.

## Validation

- In the linked `sase-github` repo:
  - Run focused tests around `tests/test_workspace_plugin.py` companion repo behavior.
  - Run the plugin's full pytest suite if the environment is usable; if the plugin editable install remains blocked by
    the known `sase>=0.11.0` / workspace `sase==0.10.2` mismatch, run tests through the main repo virtualenv as in the
    previous change.
  - Run plugin `ruff` and `mypy`.

- In the main repo:
  - Run focused SDD store/init tests touched by the change.
  - Run `just install` before repository checks if the workspace has not been freshly bootstrapped.
  - Run required `just check` because source/docs/tests in the main repo will change.

- In both repos:
  - Run `git diff --check`.
  - Confirm worktrees contain only the implementation/docs/test changes intended by this plan.

## Risks and recovery

- **Public by default is intentional but visible.** New SDD companion repositories may contain SDD planning artifacts.
  The implementation should only change newly-created companion repo visibility, and docs should make this explicit.
- **Existing private repos remain private.** This avoids surprising visibility changes but means old repos are not
  automatically normalized. If later desired, add an explicit migration/repair command rather than hiding it in init.
- **Label creation can fail after repo creation.** This is recoverable: the next explicit `sase sdd init` run will adopt
  the existing public repo and retry the label ensure step.
- **Exact-record verification matters.** Without passing the existing record's repo to the provider, rerunning init
  could rediscover a different candidate. The implementation should avoid that drift.
