---
create_time: 2026-07-09 02:39:14
status: done
prompt: 202607/prompts/sdd_repo_rename.md
tier: tale
---
# Rename SDD Companion Repo to `sase--sdd`

## Goal

Rename the GitHub companion SDD repository from `sase-org/sdd` to `sase-org/sase--sdd`, update local state and hardcoded
references, and remove the GitHub provider's implicit fallback that looks for an org-level `sdd` companion repository by
default.

## Current Findings

- `gh repo view sase-org/sdd` succeeds. It is private, uses `master`, and has SSH URL `git@github.com:sase-org/sdd.git`.
- `gh repo view sase-org/sase--sdd` currently fails with "Could not resolve to a Repository".
- The primary SASE checkout and this numbered workspace both have `.sase/sdd-store.json` records pointing at
  `sase-org/sdd`.
- The primary and numbered `.sase/sdd` companion clones are clean on `master` and have `origin` set to
  `git@github.com:sase-org/sdd.git`.
- There is no obvious standalone `/home/bryan/projects/github/sase-org/sdd` dev checkout to rename; the relevant local
  checkout is the SASE store path `.sase/sdd`.
- Main repo CI hardcodes `sase-org/sdd` in `.github/workflows/ci.yml`.
- The GitHub workspace provider in the linked `sase-github` repo implements the default candidate order as
  `<owner>/<repo>--sdd`, then `<owner>/sdd`, and tests explicitly assert the fallback.
- SASE's storage layout currently uses `.sase/sdd` as the stable SDD store mount for both local and separate-repo
  storage. I will keep that path stable unless you explicitly want a broader storage-layout migration; the repo identity
  will become `sase-org/sase--sdd`.

## Plan

1. Preflight and protect state.
   - Re-check that `sase-org/sdd` exists and `sase-org/sase--sdd` does not.
   - Re-check that primary and current `.sase/sdd` clones are clean and can see their current origin.
   - If a standalone local clone named `sdd` appears before implementation, rename that directory to `sase--sdd`;
     otherwise do not rename SASE's `.sase/sdd` storage mount because current code and docs treat it as the API path.

2. Rename the GitHub repository.
   - Run `gh repo rename -R sase-org/sdd sase--sdd --yes`.
   - Verify `gh repo view sase-org/sase--sdd --json nameWithOwner,sshUrl,url,isPrivate,defaultBranchRef`.
   - Check how GitHub reports the old name after rename so stale references can be handled intentionally rather than
     relying on redirects.

3. Update local SDD companion state.
   - Update the primary and current workspace `.sase/sdd` remotes to `git@github.com:sase-org/sase--sdd.git`.
   - Update the primary and current workspace `.sase/sdd-store.json` records to `"repo": "sase-org/sase--sdd"` and
     `"remote_url": "git@github.com:sase-org/sase--sdd.git"`.
   - Pull or fetch after the remote update to verify the renamed repository is reachable.
   - Avoid changing generated memory/provider instruction files unless explicitly approved.

4. Stop default discovery from looking for `owner/sdd`.
   - In the linked `sase-github` repo, change `_companion_sdd_candidates()` so the no-override default returns only
     `(owner, f"{repo}--sdd")`.
   - Preserve `sdd.repo.name` override support so projects can still explicitly choose `owner/sdd` or any other
     companion repo.
   - Preserve existing materialized store-record behavior: if `.sase/sdd-store.json` already names a repo,
     `create_and_materialize_sdd_store()` should continue verifying that exact recorded repo through
     `options["sdd_repo"]`.
   - Update or replace tests that currently assert the legacy fallback:
     - default candidates should be only `widget--sdd`;
     - materialization should not probe `acme/sdd`;
     - create-with-missing should create `acme/widget--sdd` instead of adopting `acme/sdd`;
     - explicit `sdd.repo.name: sdd` or `sdd.repo.name: acme/sdd` should still probe/use `acme/sdd`.

5. Update SASE main repo references.
   - Change `.github/workflows/ci.yml` to check out `sase-org/sase--sdd` and write that repo/remote into
     `.sase/sdd-store.json`.
   - Update exact `sase-org/sdd` test fixtures to `sase-org/sase--sdd` where they model this repo's companion metadata.
   - Update `src/sase/default_config.yml` comments and SDD docs that currently describe the GitHub default order as
     `<owner>/<repo>--sdd`, then `<owner>/sdd`; the default should be only `<owner>/<repo>--sdd`, with `sdd.repo.name`
     documented as the explicit way to choose another repo.
   - Review docs that mention companion examples such as `owner/sdd` and keep them only where they are clearly examples
     of explicit override or existing materialized records, not default discovery.

6. Verify both repos.
   - In `sase-github`: run `just install` if needed, then `just check`.
   - In `sase`: run `just install` first, then `just check` because repo instructions require it after file changes.
   - Run targeted tests if full checks reveal unrelated failures, especially SDD store/materialization tests and the
     exact metadata fixture tests touched by the rename.
   - Re-run read-only operational checks:
     - `gh repo view sase-org/sase--sdd`;
     - `git -C <primary>/.sase/sdd remote -v`;
     - `git -C <workspace>/.sase/sdd remote -v`;
     - `sase sdd path` and an SDD command that reads the materialized store.

## Risks and Compatibility

- GitHub may redirect `sase-org/sdd` after the rename, but the code should not rely on that; CI, local records, and
  tests should use `sase-org/sase--sdd`.
- Removing the default `owner/sdd` fallback is a behavior change for any GitHub project that depended on an org-level
  shared SDD repo without explicit config. The intended compatibility path is to set `sdd.repo.name: sdd` or
  `sdd.repo.name: owner/sdd`.
- Renaming `.sase/sdd` itself would be a larger storage layout change affecting `SASE_SDD_DIR`, prompt references, bead
  paths, docs, and many tests. This plan keeps `.sase/sdd` as the stable storage root and renames the repository
  identity/remotes instead.
