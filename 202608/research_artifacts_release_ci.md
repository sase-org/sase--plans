---
tier: tale
title: Repair the sase-research-artifacts release pipeline
goal:
  Every push to sase-research-artifacts master finishes with a green Publish workflow,
  release-please opens its release PR again, and the first real PyPI publish cannot fail
  silently or ship an uninstallable wheel.
size: medium
proposed_by: bbugyi200.athena.00s
create_time: 2026-08-14 08:43:37
status: wip
---

# Plan: Repair the sase-research-artifacts release pipeline

Repair the `Publish` workflow in the `sase-research-artifacts` repository, which has
failed on **every** push to `master` since the repo was created. Align its release
configuration with the way `sase` and `sase-core` handle releases, and close the two
latent failures that will surface the moment the release PR is merged.

Open the repository with `sase repo open sase-research-artifacts` and use the printed
path for all reads and writes. Do not clone or web-fetch it any other way.

## Diagnosis

`actstat -n 5 --repo sase-org/sase-research-artifacts` shows two distinct failure
classes across the last five commits on `master`.

### Failure A -- `Publish` / `release` (live, fails on every push)

Every `Publish` run dies after ~15-25s in
`step 2: Run googleapis/release-please-action@v5`. The job log ends with:

```
✔ Successfully updated reference release-please--branches--master--components--sase-research-artifacts to a58907d…
##[error]release-please failed: GitHub Actions is not permitted to create or approve pull requests.
```

Root cause, confirmed against the live GitHub configuration:

1. `.github/workflows/publish.yml` requests
   `token: ${{ secrets.SASE_RELEASE_TOKEN || secrets.GITHUB_TOKEN }}`.
2. `gh secret list --repo sase-org/sase-research-artifacts` returns **nothing** -- the
   repo has no secrets at all -- so that expression silently degrades to `GITHUB_TOKEN`.
3. `gh api orgs/sase-org/actions/permissions/workflow` and the same endpoint for every
   repo return
   `{"default_workflow_permissions":"read","can_approve_pull_request_reviews":false}`.
   With that policy the built-in `GITHUB_TOKEN` may push branches but may **not** open
   pull requests.

release-please therefore gets far enough to create and update its release branch
(`contents: write` is granted and works) and then fails at PR creation, leaving the repo
in a half-applied state on every single push.

**Correction to the premise in the request.** `sase` and `sase-core` do _not_ release
without a secret -- they both carry a repo-level `SASE_RELEASE_TOKEN`:

```
$ gh secret list --repo sase-org/sase        →  CLOUDFLARE_API_TOKEN, SASE_RELEASE_TOKEN (2026-06-08)
$ gh secret list --repo sase-org/sase-core   →  SASE_RELEASE_TOKEN (2026-06-08)
$ gh secret list --repo sase-org/sase-research-artifacts → (empty)
$ gh secret list --org  sase-org             →  (empty; there are no org-level secrets)
```

`sase/.github/workflows/publish.yml` passes `token: ${{ secrets.SASE_RELEASE_TOKEN }}`
with **no** `GITHUB_TOKEN` fallback, and `sase-core/.github/workflows/release-plz.yml`
uses `GITHUB_TOKEN: ${{ secrets.SASE_RELEASE_TOKEN || secrets.GITHUB_TOKEN }}` -- whose
fallback never engages because the secret is present. A PAT is the only mechanism
currently in use anywhere in `sase-org` that can open a release PR. There is no
secret-free release path to copy; `sase-research-artifacts` is simply the one repo that
never had the secret provisioned.

### Failure B -- `CI` / `check` (historical, already resolved; no fix required)

`CI` failed on `a7d9e04` and `379b362`:

```
tests/test_xprompt_loading.py::test_research_swarm_declares_typed_input FAILED
E  AssertionError: assert [('prompt', '...ait', 'word')] == [('prompt', 'text')]
```

`a7d9e04 feat: Add \`wait\` argument to
#research_swarm`added the`wait`input without updating the assertion;`189841d fix:
document research swarm wait argument`fixed it. **CI is green at`master` HEAD** (`gh run
list --commit 189841d`→`CI:
success`), so there is no test to repair. The only residue worth addressing is that the matrix defaults to `fail-fast:
true`, so the 3.13 failure cancelled the 3.12 job (`⊘ check (3.12)`) and hid whether
3.12 was also broken.

### Failure C -- `Publish` / `publish` (latent; fires as soon as Failure A is fixed)

The `publish` job is currently skipped (`release_created == 'false'`), so it has never
run. Once release-please can open a PR and that PR is merged, it will run -- and fail:

- `curl https://pypi.org/pypi/sase-research-artifacts/json` → **404**. The PyPI project
  does not exist, and there is no pending trusted publisher, so
  `pypa/gh-action-pypi-publish` (which uses `id-token: write` OIDC against
  `environment: pypi`) has nothing to authenticate against.
- `gh api repos/sase-org/sase-research-artifacts/environments` → **empty**, whereas
  `sase` has a `pypi` environment. GitHub auto-creates an environment referenced by a
  job, but the matching PyPI-side trusted publisher must name it exactly.
- Worse, the wheel would be **uninstallable**. `pyproject.toml` pins
  `dependencies = ["sase>=0.17.0"]`, and PyPI's latest `sase` is **0.16.0**. The in-repo
  comment and `README.md:108` both say 0.17.0 "has not reached PyPI" yet. CI and the
  install-smoke job only pass because they install `sase` from a source checkout via
  `--overrides`. A real PyPI consumer would hit an unsatisfiable resolution.

### Residual state to clean up

`gh api repos/sase-org/sase-research-artifacts/branches` lists three branches and
`gh pr list --state all` lists **zero** PRs:

- `master`
- `release-please--branches--master--components--sase-research-artifacts` -- the live
  release branch; release-please will reuse it and open the PR once the token works.
- `release-please--branches--master--components--sase-research` -- **orphaned** by
  `807e209 feat!: rename research plugin identity`; nothing will ever reference it
  again.

## Decision point (resolve before implementing)

Fixing Failure A requires one of these. **Recommend Option A** -- it is what `sase` and
`sase-core` actually do, and it is the only option that does not change policy for every
repo in the org.

**Option A (recommended): provision `SASE_RELEASE_TOKEN` on `sase-research-artifacts`.**
The project owner runs, from a machine holding the same PAT already installed on `sase`
and `sase-core`:

```bash
gh secret set SASE_RELEASE_TOKEN --repo sase-org/sase-research-artifacts
```

Secret _values_ are never readable through the API, so the implementing agent cannot
copy the token from `sase`; the owner must supply it. If the PAT is fine-grained, its
repository-access list must be extended to include `sase-research-artifacts` with
`Contents: read/write` and `Pull requests: read/write`.

**Option B (fallback, only if the owner declines a PAT): allow Actions to open PRs.**
Because the _organization_ setting is `can_approve_pull_request_reviews: false`, and the
org setting caps what a repo may enable, this must be turned on at the org level:

```bash
gh api -X PUT orgs/sase-org/actions/permissions/workflow \
  -f default_workflow_permissions=read -F can_approve_pull_request_reviews=true
```

Trade-offs to weigh before choosing B: it loosens policy for **every** `sase-org` repo,
and pull requests opened with `GITHUB_TOKEN` do not trigger `on: pull_request` workflows
-- so the release PR would run neither `CI` nor the `PR Title` check. Verify the setting
actually took effect at repo scope afterwards
(`gh api repos/sase-org/sase-research-artifacts/actions/permissions/workflow`); if the
org cap does not propagate as expected, Option A is the only remaining path.

Implement Option A below. If the owner picks Option B, keep the existing
`|| secrets.GITHUB_TOKEN` fallback and skip the preflight guard in step 2, but apply
every other step unchanged.

## Implementation steps

All paths are relative to the `sase-research-artifacts` checkout.

### 1. Provision the release token (owner action, unblocks everything else)

Confirm the secret landed before touching workflow files, because steps 2 and 6 assume
it exists:

```bash
gh secret list --repo sase-org/sase-research-artifacts   # expect SASE_RELEASE_TOKEN
```

### 2. Make the release step demand the real token, and retry like `sase`

In `.github/workflows/publish.yml`, rework the `release` job to mirror
`sase/.github/workflows/publish.yml`:

- Add a preflight step that fails fast with an actionable message when the secret is
  absent, so a missing token can never again masquerade as a permissions error:

  ```yaml
  - name: Require a release token
    if: ${{ github.event_name == 'push' }}
    env:
      RELEASE_TOKEN: ${{ secrets.SASE_RELEASE_TOKEN }}
    run: |
      if [ -z "${RELEASE_TOKEN}" ]; then
        echo "SASE_RELEASE_TOKEN is not set on this repository." >&2
        echo "The built-in GITHUB_TOKEN cannot open pull requests while sase-org sets" >&2
        echo "can_approve_pull_request_reviews=false, so release-please cannot proceed." >&2
        echo "Fix: gh secret set SASE_RELEASE_TOKEN --repo ${{ github.repository }}" >&2
        exit 1
      fi
  ```

- Change the action's token to `token: ${{ secrets.SASE_RELEASE_TOKEN }}` -- drop the
  `|| secrets.GITHUB_TOKEN` fallback. `sase` deliberately has no fallback; silent
  degradation is what made this failure take five pushes to notice.
- Add the same three-attempt retry ladder `sase` uses: `release1` / `release2` /
  `release3` with `continue-on-error: true` on the first two, a `sleep 60` back-off
  between attempts gated on the previous step's `outcome == 'failure'`, and
  `release_created` computed as
  `${{ steps.release3.outputs.release_created || steps.release2.outputs.release_created || steps.release1.outputs.release_created || 'false' }}`.
  Keep every attempt gated on `github.event_name == 'push'`.

Leave the job's existing `permissions:` block (`contents: write`, `issues: write`,
`pull-requests: write`) exactly as is -- it is already correct.

### 3. Add a `concurrency` group to `Publish`

`sase` guards its publish workflow with

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false
```

`sase-research-artifacts` has none, so two quick pushes can race release-please into the
same release branch. Add the same block at workflow top level.

### 4. Guard the publish job against an unresolvable `sase` floor

Before `pypa/gh-action-pypi-publish` in the `publish` job, add a step that reads the
`sase` floor out of `pyproject.toml`, queries `https://pypi.org/pypi/sase/json`, and
exits non-zero with an explicit message when no released `sase` version satisfies the
floor. This turns Failure C from a confusing OIDC/upload error into one line naming the
real blocker, and prevents publishing a wheel nobody can install.

Do **not** relax the `sase>=0.17.0` floor to make this pass -- the floor is correct; the
dependency simply is not on PyPI yet. The guard is expected to fail (loudly, and only at
publish time) until `sase` 0.17.0 ships.

### 5. Stop the CI matrix from hiding a second failure

In `.github/workflows/ci.yml`, add `fail-fast: false` under `strategy:` so a 3.13
failure no longer cancels the 3.12 job. No other CI change is needed -- `master` HEAD is
already green.

### 6. Clean up the orphaned release branch

Delete the branch stranded by the plugin rename, so future `gh api .../branches` output
reflects reality:

```bash
gh api -X DELETE repos/sase-org/sase-research-artifacts/git/refs/heads/release-please--branches--master--components--sase-research
```

Leave `release-please--branches--master--components--sase-research-artifacts` in place;
release-please will reuse it and open the PR on the next push.

### 7. Record the owner prerequisites for the first publish

Add a short "Releasing" section to `README.md` (or `docs/configuration.md`, whichever
fits the repo's existing structure) documenting what must be true before the first
release can reach PyPI:

- `SASE_RELEASE_TOKEN` must exist as a repo secret, and why `GITHUB_TOKEN` is not
  sufficient under the org's Actions policy.
- A PyPI **pending trusted publisher** must be created for project
  `sase-research-artifacts`, owner `sase-org`, repository `sase-research-artifacts`,
  workflow `publish.yml`, environment `pypi`.
- The `sase>=0.17.0` floor must be satisfiable from PyPI before publishing.

## Deliberate non-changes

- **Keep `"component": "sase-research-artifacts"` in `release-please-config.json`.**
  `sase` uses a bare `"packages": {".": {}}`, but with `include-component-in-tag: false`
  the component only affects branch and PR naming. Removing it now would orphan a
  _second_ release branch and force another cleanup for zero functional gain.
- **Keep `bootstrap-sha: 6b63282…`** -- it points at `chore: First Commit`, so
  release-please correctly considers the full history.
- **Do not add `sase`'s `sync-release-metadata` job.** It reconciles `uv.lock` and the
  `sase-core-rs` floor, neither of which exists in this repo.
- **Do not touch `tests/test_xprompt_loading.py`** -- Failure B is already fixed on
  `master`.

## Verification

1. `gh secret list --repo sase-org/sase-research-artifacts` lists `SASE_RELEASE_TOKEN`.
2. Lint the edited workflows locally (`just lint` if it covers YAML, otherwise
   `python -c "import yaml,sys; [yaml.safe_load(open(p)) for p in sys.argv[1:]]" .github/workflows/*.yml`).
3. Run the repo's own gate: `just install && just test` (and `just lint`) in the
   `sase-research-artifacts` checkout, so the workflow edits do not regress the `tests/`
   suite that asserts repo structure.
4. Push to `master`, then confirm with
   `actstat -n 1 --repo sase-org/sase-research-artifacts` that `Publish` is **green**.
5. Confirm release-please actually opened its PR:
   `gh pr list --repo sase-org/sase-research-artifacts` should show one
   `chore(master): release …` PR (currently zero).
6. Confirm the orphaned branch is gone:
   `gh api repos/sase-org/sase-research-artifacts/branches --jq '.[].name'` should list
   only `master` and
   `release-please--branches--master--components--sase-research-artifacts`.
7. Do **not** merge the release PR as part of this work. Merging triggers `build`,
   `install-smoke`, and `publish`; publishing is blocked on the PyPI trusted publisher
   and the `sase` 0.17.0 floor from step 7's checklist. Report the release PR as ready
   and leave the merge decision to the project owner.

## Risks

- **The PAT may not cover this repo.** If `SASE_RELEASE_TOKEN` is fine-grained and
  scoped to `sase` and `sase-core` only, step 1 will appear to succeed while
  release-please still fails -- with a 403 on the _repository_, not the "not permitted
  to create pull requests" message. Distinguish the two by reading the new failure text
  before assuming the fix did not land.
- **release-please may reuse a stale branch body.** The existing release branch was
  built across the `feat!` rename. Sanity-check the generated `CHANGELOG.md` and the
  proposed version in the PR (expect `0.2.0`: `bump-minor-pre-major` with a `feat!` at
  `0.1.0`) before the owner merges.
