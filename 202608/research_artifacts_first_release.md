---
tier: epic
title: Publish the first sase-research-artifacts release from CI
goal: "sase-research-artifacts has a real GitHub release and a matching distribution on
  PyPI, published end to end by the repo's own Publish workflow, with the
  SASE_RELEASE_TOKEN and trusted-publishing prerequisites proven rather than assumed.

  "
phases:
  - id: preflight
    title: Prove the prerequisites and rehearse the unexercised release path
    depends_on: []
    size: medium
    description: "preflight: settle the two irreversible decisions with the user (PyPI
      pending publisher, first-release version), create the missing `pypi` GitHub
      environment, rehearse the never-run install-smoke job locally via `just
      test-wheel`, and delete the stale pre-rename release-please branch.

      "
  - id: release_pr
    title: Exercise SASE_RELEASE_TOKEN and open the release PR
    depends_on:
      - preflight
    size: small
    description: 'release_pr: push one trigger commit to master so the release job runs
      under the new token, then confirm release-please opens the repo''s first release
      PR instead of failing with the "GitHub Actions is not permitted to create or
      approve pull requests" error.

      '
  - id: publish
    title: Merge the release PR and drive the publish pipeline green
    depends_on:
      - release_pr
    size: medium
    description: "publish: merge the release PR, then shepherd the resulting Publish run
      through the release, build, install-smoke, and publish jobs -- the last three of
      which have never executed -- until the GitHub release and the PyPI upload both
      exist.

      "
  - id: verify
    title: Verify the published artifact and record the install blocker
    depends_on:
      - publish
    size: small
    description:
      "verify: confirm the tag, GitHub release, and PyPI files match what CI built,
      prove the wheel's entry points from the real PyPI artifact, and file the follow-up
      for the unresolvable `sase>=0.17.0` floor."
proposed_by: bbugyi200.athena.064
status: done
bead_id: sase-pt
create_time: 2026-09-09 19:51:25
---

- **PROMPT:**
  [prompts/202608/research_artifacts_first_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/research_artifacts_first_release.md)
- **BEAD:**
  [sase-pt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-pt/README.md)

# Plan: Publish the first sase-research-artifacts release from CI

## Context

`sase-org/sase-research-artifacts` has never produced a release. Every `Publish`
workflow run since the repo was created has failed, and neither a git tag, a GitHub
release, nor a PyPI project exists yet.

Verified state as of 2026-08-18 (all facts below were checked directly, not assumed):

| Fact                                                                                                                                           | Evidence                                                                                       |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `SASE_RELEASE_TOKEN` exists as a repo secret, updated `2026-08-18T14:25:10Z`                                                                   | `gh secret list`                                                                               |
| That update is _newer_ than the last Publish run (`2026-08-16T18:45:50Z`), so the token has never been exercised                               | `gh run list`                                                                                  |
| All 7 Publish runs failed at the same point                                                                                                    | `gh run list`                                                                                  |
| Root cause: `release-please failed: GitHub Actions is not permitted to create or approve pull requests.`                                       | `gh run view 31965555233 --log-failed`                                                         |
| That setting is off at both repo and org scope (`can_approve_pull_request_reviews: false`)                                                     | `gh api .../actions/permissions/workflow`, `gh api orgs/sase-org/actions/permissions/workflow` |
| No tags, no releases, no PRs have ever existed in the repo                                                                                     | `git tag -l`, `gh release list`, `gh pr list --state all`                                      |
| `sase-research-artifacts` does not exist on PyPI (HTTP 404)                                                                                    | `curl https://pypi.org/pypi/sase-research-artifacts/json`                                      |
| The repo has **no** GitHub environments, but `publish.yml` declares `environment: pypi`                                                        | `gh api .../environments` -> `total_count: 0`                                                  |
| `sase-org/sase`, `sase-org/sase-core`, and this repo are all public                                                                            | `gh repo view --json visibility`                                                               |
| No branch protection and no rulesets on `master`                                                                                               | `gh api .../branches/master/protection` -> 404, `gh api .../rulesets` -> `[]`                  |
| Release-please has already staged branch `release-please--branches--master--components--sase-research-artifacts` at `3b82a0d`                  | `git ls-remote --heads origin`                                                                 |
| That staged branch bumps to **0.2.0**, not 0.1.0                                                                                               | `git diff master origin/rp-current`                                                            |
| A stale pre-rename branch `release-please--branches--master--components--sase-research` still exists at `4f9d5ae`                              | `git ls-remote --heads origin`                                                                 |
| `sase-github` is a known-good reference: same workflow shape, same token secret, a `pypi` environment, and a successful `v0.2.5` release today | `gh release list -R sase-org/sase-github`, `gh api .../environments`                           |

### Why the token is the right fix

Release-please got all the way to pushing its branch and only failed on
`POST /repos/.../pulls`. The org-level "Allow GitHub Actions to create and approve pull
requests" toggle is off, and `secrets.GITHUB_TOKEN` therefore cannot open the release
PR. `publish.yml` already prefers a PAT:

```yaml
token: ${{ secrets.SASE_RELEASE_TOKEN || secrets.GITHUB_TOKEN }}
```

A PAT opens the PR as a _user_, which the org toggle does not restrict. `sase-github`
publishes successfully with exactly this arrangement, so the design is proven; only this
repo's copy of the secret is unproven. The token's value cannot be read back, so the
only real verification is to make the workflow use it -- that is what `release_pr` does.

### The two things that are still missing

1. **PyPI has no publisher for this project name.** The `publish` job uses
   `pypa/gh-action-pypi-publish` with `id-token: write` and no password, i.e. Trusted
   Publishing. For a project name that does not yet exist on PyPI, a **pending
   publisher** must be registered by hand at
   <https://pypi.org/manage/account/publishing/>. No API exposes this, and an agent
   cannot log into PyPI, so this is a user action and a hard gate on the whole plan.

2. **The `pypi` GitHub environment does not exist on this repo.** `sase-github` has one.
   The environment name is part of the OIDC claim PyPI checks, so it must exist on the
   GitHub side and match whatever the pending publisher records.

### Decision: what version is the first release?

`.release-please-manifest.json` currently says `{".":"0.1.0"}`. Release-please reads
that as _already released_
(`No latest release found ... but a previous version (0.1.0) was specified in the manifest`)
and, because history contains a `feat!:` commit under `bump-minor-pre-major: true`,
computes the next version as **0.2.0**. So the first published version will be `v0.2.0`
unless the manifest is changed first. The staged changelog also carries a
`compare/v0.1.0...v0.2.0` link to a tag that will never exist, plus a stray trailing
`## Changelog` heading.

Options, to be confirmed with the user in `preflight` because **a PyPI version number
can never be reused or reclaimed**:

- **A (recommended): accept `v0.2.0`.** Zero config churn, the release branch is already
  staged and correct, and the pre-release identity rename genuinely was a breaking
  change. Cost: the changelog's first compare link 404s -- cosmetic, fixable later.
- **B: make the first release `v0.1.0`.** Set the manifest to `{".":"0.0.0"}` and let
  the same breaking change bump it to `0.1.0`. Matches the `initial-version: "0.1.0"`
  already in `release-please-config.json` and the current `pyproject.toml` version.
  Cost: one extra commit, and release-please must recompute its staged branch.
- **C: accept `v0.2.0` and also push a `v0.1.0` tag at the bootstrap SHA** so the
  manifest's claim and the compare link both become true. Cost: a tag with no release
  and no artifact behind it, which is its own kind of confusing. Not recommended.

### Risks this plan is built around

- **`build`, `install-smoke`, and `publish` have literally never run.** They are gated
  on `release_created == 'true'`, which has never been true. `install-smoke` is the most
  involved job in the repo: it installs the built wheel with a `--overrides` file, runs
  `just install-source-sase`, maturin-builds `sase-core-rs`, and then asserts the real
  provider registry assembles with `registry.diagnostics == ()`. `preflight` rehearses
  this locally with `just test-wheel` rather than discovering a failure mid-release.
- **The release PR will be this repo's first PR ever.** It triggers `ci.yml`
  (`pull_request`) and `pr-title.yml`. The title `chore(master): release <version>` does
  satisfy `pr-title.yml`'s regex (`chore` is an allowed type and `master` matches the
  scope character class), but neither workflow has been exercised on a PR here.
- **The published wheel will not be installable from PyPI yet.** `pyproject.toml` pins
  `sase>=0.17.0` while PyPI's `sase` is at `0.16.0`. CI hides this with a `--overrides`
  file pointed at a source checkout. This does not block publishing, and it is
  deliberate per the comment in `pyproject.toml`, but it must be recorded rather than
  silently shipped -- handled in `verify`.
- **PyPI uploads are permanent.** If `publish` half-succeeds, a naive re-run fails with
  "file already exists". Remediation is documented in that phase.

### Ground rules for every phase

- Only ever touch the repo through the path printed by
  `sase repo open sase-research-artifacts -r "<reason>"`. Never clone or web-fetch it.
- Every commit goes through the `/sase_git_commit` skill (`sase stitch create`). This
  repo has no PRs and no branch protection; existing history is pushed directly to
  `master`, so the trigger commit follows that same convention.
- Wait on CI with the `/sase_monitor` skill, never with an inline blocking loop.
- Do not edit `.github/workflows/*.yml` casually: `tests/test_ci_install_contract.py`
  makes static assertions about both `ci.yml` and `publish.yml` (checkout counts, the
  rust toolchain step, `--overrides /tmp/sase-overrides.txt dist/*.whl`, and the
  ordering of `dist/*.whl` before `install-source-sase`). Any workflow edit must keep
  those assertions true and must be re-verified with `just test`.

## Phase: Prove the prerequisites and rehearse the unexercised release path

Settle everything that must be true _before_ anything irreversible happens.

1. **Ask the user, and block on the answer** (use `/sase_questions`; both answers are
   needed before proceeding):
   - Has a **pending publisher** been created on PyPI for this project? Report the exact
     values it must have and ask the user to confirm or create them:
     - PyPI Project Name: `sase-research-artifacts`
     - Owner: `sase-org`
     - Repository name: `sase-research-artifacts`
     - Workflow name: `publish.yml`
     - Environment name: `pypi`
   - Which version option -- A (`v0.2.0`, recommended), B (`v0.1.0`), or C -- should the
     first release use?
2. **Create the missing GitHub environment** so the OIDC claim carries it and the repo
   matches `sase-github`:
   ```bash
   gh api -X PUT repos/sase-org/sase-research-artifacts/environments/pypi
   gh api repos/sase-org/sase-research-artifacts/environments   # expect total_count: 1
   ```
   Leave it with no required reviewers and no branch restrictions, matching
   `sase-github`.
3. **Rehearse `install-smoke` locally.** From the repo checkout, `just install`, then
   run the wheel-contract suite that mirrors the CI job:
   ```bash
   just test-wheel
   ```
   This builds a real sdist/wheel and installs it plus a maturin build of `sase-core-rs`
   into a throwaway venv. It takes minutes -- run it through `/sase_monitor`, not
   inline. If it fails, fix the cause here; a failure in CI's `install-smoke` would
   strand a tagged release with no artifact on PyPI. Also confirm
   `sase.artifact_providers.assemble_artifact_provider_registry` still exists on `sase`
   master (it does today, at `src/sase/artifact_providers/registry.py:57`), since the
   smoke job imports it and asserts `registry.diagnostics == ()`.
4. **If the user chose option B**, edit `.release-please-manifest.json` to
   `{".":"0.0.0"}` and commit it via `/sase_git_commit` with a `chore:`-typed message.
   That commit doubles as the `release_pr` trigger, so note it and skip the separate
   trigger commit in the next phase.
5. **Delete the stale pre-rename release branch** left over from the
   `feat!: rename research plugin identity` commit:
   ```bash
   git push origin --delete release-please--branches--master--components--sase-research
   ```
   Leave `release-please--branches--master--components--sase-research-artifacts` alone;
   release-please will reuse and update it.
6. Do **not** push anything else to `master` in this phase.

Exit criteria: the user has confirmed the PyPI pending publisher and the version choice,
the `pypi` environment exists, `just test-wheel` passed locally, and the stale branch is
gone.

## Phase: Exercise SASE_RELEASE_TOKEN and open the release PR

This phase exists purely to answer "was the token added correctly?", because the secret
cannot be read back and `workflow_dispatch` skips the release job (it is gated on
`if: github.event_name == 'push'`). Only a push to `master` exercises it.

1. **Push one trigger commit to `master`** via `/sase_git_commit`. If phase `preflight`
   already produced the manifest commit (option B), that push is the trigger -- do not
   add a second one. Otherwise make a small, real, non-version-bumping change and use a
   `chore:` or `docs:` subject; those types do not themselves force a release, and a
   release is already pending from the existing `feat` history either way.
2. **Watch the `Publish` run** via `/sase_monitor`:
   ```bash
   gh run list --workflow Publish --limit 3
   gh run watch <run-id> --exit-status
   ```
3. **Interpret the `release` job:**
   - _Success, PR opened_ -> the token is good. Record the PR number and the version it
     proposes; confirm the version matches the option chosen in `preflight`.
   - _Still `GitHub Actions is not permitted to create or approve pull requests`_ -> the
     workflow fell back to `GITHUB_TOKEN`, meaning the secret name is wrong or empty.
     Compare against `sase-github`'s secret; the name must be exactly
     `SASE_RELEASE_TOKEN` at repo scope.
   - _`Resource not accessible by integration` / 403 / 404 on the PR or label calls_ ->
     the token exists but is under-scoped. A classic PAT needs `repo` + `workflow`; a
     fine-grained PAT needs Contents: read/write, Pull requests: read/write, and Issues:
     read/write on this repo, and must not be expired. Report the precise missing grant
     to the user rather than guessing.
4. **Confirm the PR's own checks.** This is the repo's first PR, so verify `CI` and
   `PR Title` both pass on it. If `pr-title.yml` rejects the release PR title, fix the
   pattern in that workflow rather than renaming the PR, since release-please will
   regenerate the title.

Exit criteria: a release PR is open, its version matches the agreed choice, and its CI
and PR-title checks are green. Do not merge in this phase.

## Phase: Merge the release PR and drive the publish pipeline green

1. **Optionally clean the changelog first.** If option A was chosen, the staged
   `CHANGELOG.md` contains a `compare/v0.1.0...v0.2.0` link to a nonexistent tag and a
   stray trailing `## Changelog` heading. Either fix them with a commit onto the release
   branch before merging, or leave them and clean up afterwards -- state which was done.
   Do not let this block the release.
2. **Merge the release PR.** Squash merge, keeping the
   `chore(master): release <version>` title so release-please recognises it:
   ```bash
   gh pr merge <number> --squash
   ```
3. **Watch the `Publish` run triggered by the merge** with `/sase_monitor`. Expect four
   jobs in sequence, three of which have never run before:
   - `release`: creates tag `v<version>` and the GitHub release, and sets
     `release_created=true`.
   - `build`: `uv build`, uploads the `dist` artifact.
   - `install-smoke`: the job rehearsed in `preflight`.
   - `publish`: OIDC upload to PyPI under `environment: pypi`.
4. **If `release_created` comes back `false`** even though the tag was created, the
   downstream jobs will skip. `sase-github` proves the unprefixed `release_created`
   output works with this exact config, so first re-read the `release` job log before
   changing anything; the manifest may simply have judged there was nothing to release.
5. **If `publish` fails with a PyPI 403 / "not a valid publisher"**, the pending
   publisher does not match. Report the exact mismatch from the error (owner, repo,
   workflow filename, environment) to the user, have them correct it on PyPI, then retry
   _without_ a new release:
   ```bash
   gh workflow run Publish -f publish_existing=true
   ```
   That path skips the `release` job and rebuilds the same version from `master`.
6. **If `publish` fails after a partial upload**, do not blindly re-run: PyPI refuses
   re-uploads of an existing file. Check which files landed on PyPI first, and if only
   some did, add `skip-existing: true` to the `pypa/gh-action-pypi-publish` step for the
   retry.
7. **Never bump the version to work around a failed upload** without saying so
   explicitly; a burned version number is permanent.

Exit criteria: the `Publish` run is green end to end, tag `v<version>` and a GitHub
release exist, and `https://pypi.org/project/sase-research-artifacts/` resolves.

## Phase: Verify the published artifact and record the install blocker

1. **Confirm the release surfaces agree:**
   ```bash
   gh release list -R sase-org/sase-research-artifacts
   git ls-remote --tags origin
   curl -s https://pypi.org/pypi/sase-research-artifacts/json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['info']['version'], [f['filename'] for f in d['urls']])"
   ```
   The PyPI version, the tag, the GitHub release, `pyproject.toml` on `master`, and
   `.release-please-manifest.json` must all name the same version.
2. **Prove the real PyPI artifact, not just the CI-built one.** Download the published
   wheel and check its metadata and entry points in a throwaway venv, using the same
   `--overrides` trick CI uses so the unsatisfiable `sase>=0.17.0` floor does not block
   the check. Assert the four entry-point groups declared in `pyproject.toml`
   (`sase_artifact_refs`, `sase_file_hooks`, `sase_xprompts`, `sase_config`) and that
   the old `sase-research` distribution name is absent.
3. **File the install blocker as a task bead** using `/sase_new_task` (check for
   duplicates first, as that skill requires): the published distribution requires
   `sase>=0.17.0`, but PyPI's `sase` is at `0.16.0`, so
   `pip install sase-research-artifacts` cannot resolve for an outside user until `sase`
   0.17.0 ships. Record that the floor and the README/`docs/configuration.md` notes
   should be revisited when that happens.
4. **Report to the user**, in plain terms: which version shipped, that the token is now
   proven, what the PyPI page shows, and the one remaining caveat from step 3.

Exit criteria: the published artifact is verified from PyPI itself, the follow-up bead
exists, and the user has a clear summary.
