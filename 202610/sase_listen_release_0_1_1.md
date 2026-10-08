---
tier: tale
title: Fix sase-listen release PR CI and publish v0.1.1 to PyPI
goal: 'sase-listen release PR #2 is green and merged, master CI is green, and sase-listen
  0.1.1 (wheel and sdist) is installable from PyPI.'
size: medium
proposed_by: bbugyi200.athena.0y7
status: done
---

# Fix sase-listen release PR #2 CI and ship v0.1.1 to PyPI

Repair the failing CI on the release-please PR
https://github.com/sase-org/sase-listen/pull/2 at its source, commit the fix to
`sase-listen` master with `/sase_git_commit`, wait for green CI on master and on the
regenerated release PR, merge the PR, and then verify that `sase-listen` 0.1.1 is live
on PyPI (https://pypi.org/project/sase-listen/ currently returns 404 because the package
has never been published).

## Root cause (verified during planning)

1. **Stale `uv.lock` in the release PR.** All four CI matrix jobs on PR #2 (head
   `c22982b8`, branch `release-please--branches--master--components--sase-listen`) fail
   at `just install` (`uv sync --locked --all-groups`) with:
   `error: The lockfile at uv.lock needs to be updated, but --locked was provided.` (The
   3.13 and macOS jobs show CANCELLED only because fail-fast stopped them; both
   "Conventional PR title" checks pass.) release-please's `python` strategy bumps
   `pyproject.toml`, `src/sase_listen/__init__.py`, and `.release-please-manifest.json`
   to 0.1.1, but `uv.lock` still records the editable `sase-listen` package at
   `version = "0.1.0"`. The same bug hit 0.1.0 and was patched by hand with a one-line
   lock commit pushed to the release branch
   (`3e35a63 chore(master): refresh uv.lock for 0.1.0 version bump`). Master CI at
   `f64bc6c` is green; only the release PR is affected.
2. **release-please will not refresh the PR for a config-only commit.**
   `Manifest.maybeUpdateExistingPullRequest` (release-please `src/manifest.ts`) returns
   early with "PR ... remained the same" when the computed PR **body** is unchanged. A
   `build`/`ci`/`chore` commit is hidden from the changelog, so the body does not change
   and PR #2 would keep its stale `uv.lock`, even after the config is fixed. The root
   config option `"always-update": true` (added in release-please v16.15.0;
   `googleapis/release-please-action@v5` bundles v17) makes release-please always
   rebuild the existing release PR.
3. **First-ever PyPI publish.** Every earlier `Publish` run either failed in the
   `release` job (before the `SASE_RELEASE_TOKEN` secret existed: "GitHub Actions is not
   permitted to create or approve pull requests") or skipped
   `build`/`install-smoke`/`publish` because no release was created. The v0.1.0 GitHub
   release was created out-of-band and never published. So merging PR #2 runs the
   `build → install-smoke → publish` chain for the first time. The `publish` job uses
   PyPI trusted publishing (`environment: pypi`, `id-token: write`, no API token). A
   brand-new PyPI project therefore needs a **pending trusted publisher** registered on
   PyPI.

## Prerequisite for the user (ideally before approving this plan)

Register a pending trusted publisher at https://pypi.org/manage/account/publishing/ with
these values:

- PyPI project name: `sase-listen`
- Owner: `sase-org`
- Repository name: `sase-listen`
- Workflow name: `publish.yml`
- Environment name: `pypi`

The agent cannot check this from the CLI. If it is missing, the `publish` job fails with
an `invalid-publisher` / trusted-publishing token-exchange error (see Failure handling).
The GitHub `pypi` environment already exists with no protection rules, so it needs no
manual approval.

## Already verified while planning (do not redo unless something changed)

- Replicated release-please's `GenericToml` updater (`src/updaters/generic-toml.ts` and
  `src/util/toml-edit.ts`, using `@iarna/toml@3` and `jsonpath-plus@10`) against the
  current `uv.lock`. JSONPath `$.package[?(@.name.value=='sase-listen')].version`
  matched exactly one pointer (`/package/86/version`), and the output diff is the single
  line `version = "0.1.0"` → `version = "0.1.1"` under `name = "sase-listen"`. The
  `.value` segment is required: release-please's tagged TOML parser wraps every scalar
  as `{start, end, value}`.
- With `pyproject.toml` at 0.1.1, `uv lock --check` (uv 0.12.23, CI's version) fails
  without that lock edit, reproducing the CI error, and passes with it.
  `uv sync --locked --all-groups --dry-run` also passes.
- Built 0.1.1 sdist and wheel with `uv build`, installed the wheel into a fresh Python
  3.12 venv with an empty environment (no system ffmpeg on PATH), and ran the publish
  workflow's `install-smoke` steps. `--version` printed `sase-listen 0.1.1`,
  `doctor --json` returned ok via bundled imageio-ffmpeg, and the tone render plus
  chapter/ID3 assertions passed.

## Steps

### 1. Open the repo

- `sase repo open sase-listen -r "Fix release-please uv.lock drift breaking PR #2 CI and ship v0.1.1"`.
  Use only the printed path. Read its `AGENTS.md`.
- Confirm `master` is clean and current: `git status --short --branch`,
  `git pull --ff-only`. If master moved past `f64bc6c`, re-check PR #2's current state
  before continuing.

### 2. Fix `release-please-config.json`

Add two root options and keep every existing key unchanged. The resulting file:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "bootstrap-sha": "e70e585abac3d0cbe856cfa04092055ba8b0513d",
  "release-type": "python",
  "initial-version": "0.1.0",
  "include-v-in-tag": true,
  "include-component-in-tag": false,
  "bump-minor-pre-major": true,
  "bump-patch-for-minor-pre-major": true,
  "always-update": true,
  "extra-files": [
    {
      "type": "toml",
      "path": "uv.lock",
      "jsonpath": "$.package[?(@.name.value=='sase-listen')].version"
    }
  ],
  "packages": {
    ".": {
      "component": "sase-listen"
    }
  }
}
```

- Do **not** edit `uv.lock`, `pyproject.toml`, `.release-please-manifest.json`,
  `CHANGELOG.md`, or any workflow on master. Master is correctly at 0.1.0, and
  release-please owns the bump.
- No docs change is needed. `docs/field-notes.md` "no PyPI release" lines are dated
  historical field notes; leave them.

### 3. Verify locally

- `python3 -m json.tool release-please-config.json > /dev/null` (valid JSON).
- If `.venv` is missing, run `just install` first. Then run `sase tool run check` (the
  guarded lint + test gate; never bare `just check`). It must pass.
- `uv lock --check` still passes on master (sanity check; nothing lock-related changed).

### 4. Commit with `/sase_git_commit`

- Invoke the `/sase_git_commit` skill (the user explicitly asked for it) and follow it
  exactly for the `sase-listen` checkout. Write the message file at
  `.sase/commit_message.md` inside the sase-listen repo.
- Use a hidden-from-changelog conventional type so release notes and the 0.1.1 version
  stay unchanged. Suggested message:

  ```
  build(release): keep uv.lock in step with release-please version bumps

  release-please bumps pyproject.toml but not uv.lock, so every release PR
  failed `uv sync --locked` in CI (0.1.0 needed a hand-pushed lock fix).
  Update the editable sase-listen entry in uv.lock as a TOML extra-file, and
  enable always-update so an existing release PR is rebuilt even when its
  changelog body is unchanged.
  ```

- Afterwards `git status --short --branch` must show a clean tree that is not ahead of
  `origin/master`. Record the pushed SHA (`FIX_SHA`).

### 5. Wait for green master CI and the regenerated release PR

Wait in the foreground with explicit Bash timeouts. CI takes about 1–2 minutes per run.
Poll until runs exist, e.g.
`gh run list --repo sase-org/sase-listen --commit "$FIX_SHA" --json databaseId,workflowName,status,conclusion`.
Then run `gh run watch <id> --repo sase-org/sase-listen --exit-status` for each run.
Hand off to `/sase_monitor` only if a wait is expected to exceed about 30 minutes, for
example because runners are queued.

- For `FIX_SHA`, the **CI** (4 matrix jobs), **Docs**, and **Publish** runs must all
  conclude `success`. In Publish, `release` succeeds and
  `build`/`install-smoke`/`publish` are skipped; that is expected.
- Confirm release-please rebuilt PR #2:
  `gh pr view 2 --repo sase-org/sase-listen --json headRefOid,title,files,body`.
  - `headRefOid` is no longer `c22982b8f43d8cac0e767c7db6d63164478fdfe1`.
  - `files` now include `uv.lock` alongside `.release-please-manifest.json`,
    `CHANGELOG.md`, `pyproject.toml`, and `src/sase_listen/__init__.py`.
  - `gh pr diff 2 --repo sase-org/sase-listen -- uv.lock` (or the full diff) shows only
    the `sase-listen` entry's `version` line moving from 0.1.0 to 0.1.1.
  - Title is still `chore(master): release 0.1.1`, and the changelog body has no new
    entry for the build commit.
- If PR #2 did not change, read the Publish run's `release` job log
  (`gh run view <id> --log --repo sase-org/sase-listen`):
  - "remained the same" means `always-update` was not honored. Re-check the key's
    spelling and placement at the config root.
  - "No entries modified in" means the JSONPath missed. Re-check it against the verified
    expression above. Fix the config on master with another `/sase_git_commit` and
    repeat this step. If it still will not regenerate, stop and report to the user with
    the log excerpt rather than hand-pushing to the release-please branch.

### 6. Wait for green CI on PR #2

- `gh pr checks 2 --repo sase-org/sase-listen --watch` (foreground, explicit timeout).
  Then confirm, for the **new** head SHA, that every check concluded `success`:
  `check (ubuntu-latest, 3.12, true, true)` (lint, tests, docs build, `uv build` and
  `twine check --strict`), `check (ubuntu-latest, 3.13)`, `check (ubuntu-latest, 3.14)`,
  `check (macos-latest, 3.12)`, and the `Conventional PR title` check(s). Make sure no
  stale result from the old head is being read.
- If a check fails for a reason other than the lockfile, investigate the log, fix it on
  master via `/sase_git_commit` with the honest conventional type, and repeat steps 5–6.
  Never merge with red or pending checks, and never use `--admin`.

### 7. Submit (merge) PR #2

- `gh pr merge 2 --repo sase-org/sase-listen --squash`. Squash is the repo convention:
  `pr-title.yml` enforces conventional titles "so squash merges produce useful release
  metadata". The squash subject becomes `chore(master): release 0.1.1 (#2)`.
- Record the merge commit
  (`gh pr view 2 --repo sase-org/sase-listen --json mergeCommit,state`) as `MERGE_SHA`,
  and confirm `state` is `MERGED`.

### 8. Wait for green master CI and the publish chain on the merge commit

- For `MERGE_SHA`, the **CI**, **Docs**, and **Publish** runs must all conclude
  `success`. In Publish, all four jobs must succeed: `release` (release-please creates
  tag `v0.1.1` and the GitHub release, output `release_created == 'true'`), `build`,
  `install-smoke`, and `publish` (`pypa/gh-action-pypi-publish`).
- Verify the GitHub side:
  - `gh release view v0.1.1 --repo sase-org/sase-listen` exists.
  - `git ls-remote --tags origin v0.1.1` points at `MERGE_SHA`.
  - PR #2 now carries the `autorelease: tagged` label instead of `autorelease: pending`.
  - Optional: on its next run, release-please should not open a new release PR unless
    new releasable commits have landed.

### 9. Verify the PyPI publication

Allow a few minutes for PyPI's CDN. Retry with sleeps for up to about 10 minutes before
declaring failure.

- `curl -fsS https://pypi.org/pypi/sase-listen/json` must show:
  - `info.version == "0.1.1"`.
  - `releases["0.1.1"]` containing both `sase_listen-0.1.1-py3-none-any.whl` and
    `sase_listen-0.1.1.tar.gz`.
  - `info.requires_python == ">=3.12"`, MIT license metadata, and project URLs matching
    `pyproject.toml`.
- `curl -s -o /dev/null -w '%{http_code}' https://pypi.org/project/sase-listen/` and
  `https://pypi.org/project/sase-listen/0.1.1/` must both return `200`.
- Fresh install from PyPI, run from an empty temp dir so no local project is picked up:
  `uvx --refresh --default-index https://pypi.org/simple --from 'sase-listen==0.1.1' sase-listen --version`
  must print `sase-listen 0.1.1`. Optionally add `... sase-listen doctor --json`, which
  must report `"ok": true`.

## Failure handling

- **`publish` fails with `invalid-publisher` / trusted-publishing exchange error.** The
  pending publisher is not registered. Do not retry blindly. Use `/sase_questions` to
  ask the user to register it with the exact values from the Prerequisite section. Once
  they confirm, rerun only the failed job with
  `gh run rerun <publish-run-id> --repo sase-org/sase-listen --failed`; it reuses the
  run's `dist` artifact. If the artifact is unavailable, and only while master's
  `pyproject.toml` version is still `0.1.1`, use
  `gh workflow run publish.yml --repo sase-org/sase-listen --ref master -f publish_existing=true`.
  Then redo step 9.
- **PyPI rejects the upload for another reason** (name policy, metadata, file already
  exists). Stop and report the exact error. Never try to re-upload different files under
  the same version.
- **`build` or `install-smoke` fails after the v0.1.1 tag exists.** Stop and report with
  logs. Do not delete tags or releases, and do not hand-publish, without asking the
  user.
- **Waits that would exceed a foreground timeout.** Rerun the wait with a larger
  explicit timeout, or hand it to `/sase_monitor`. Never end the turn promising to check
  later.

## Out of scope

- Changing the Publish trigger (sase's main repo moved release-please to a cron with a
  `uv lock` sync job; that is unnecessary here).
- Sibling repos such as `sase-telegram` and `sase-research-artifacts`. Their CI does not
  install with `--locked`, so they do not hit this failure.
- Rewriting historical field notes or the 0.1.1 changelog.

## Final report to the user

Include:

- The fix commit SHA.
- The CI/Docs/Publish run URLs for `FIX_SHA`.
- PR #2's regenerated head SHA, its green check list, and the confirmed `uv.lock` diff.
- `MERGE_SHA` and the merge-commit CI/Docs/Publish run URLs.
- The GitHub release URL.
- PyPI evidence: JSON version, file list, HTTP 200s, and the `uvx` install output.
- Any user action that was needed, such as registering the pending publisher.
