---
tier: tale
title: Unblock the sase-research-artifacts Publish workflow
goal:
  The Publish workflow on sase-org/sase-research-artifacts succeeds on master,
  release-please opens its release PR, and the repo can cut its first release.
size: small
proposed_by: bbugyi200.athena.00q
create_time: 2026-08-14 08:23:10
status: wip
---

# Plan: Unblock the sase-research-artifacts Publish workflow

## Diagnosis

`actstat --repo sase-org/sase-research-artifacts -n 3` surfaced two distinct failures.
Only one of them is still live.

### Failure 1 — CI `check (3.13)` / "Run tests" (ALREADY FIXED, no work required)

On commit `a7d9e04` ("feat: Add `wait` argument to #research_swarm"), the CI workflow
failed:

```
tests/test_xprompt_loading.py::test_research_swarm_declares_typed_input FAILED
E  AssertionError: assert [('prompt', '...ait', 'word')] == [('prompt', 'text')]
E    Left contains one more item: ('wait', 'word')
```

The commit added a `wait` input to the `research_swarm` xprompt without updating the
test that pins the xprompt's declared inputs. The sibling `check (3.12)` job shows `⊘`
at "Install dependencies" only because matrix fail-fast cancelled it; it is not an
independent failure.

The follow-up commit `189841d` ("fix: document research swarm wait argument") already
updated `tests/test_xprompt_loading.py` to expect
`[("prompt", "text"), ("wait", "word")]`. CI is **green at master HEAD**:

```
$ gh run list --repo sase-org/sase-research-artifacts --commit 189841d... \
    --json workflowName,conclusion
CI      -> success
Publish -> failure
```

**No action is required for this failure.** It is documented here so the implementing
agent does not chase an already-resolved test.

### Failure 2 — Publish / `release` job (LIVE, the actual work)

Every Publish run in the repo's history has failed — all six, from the first
release-capable commit `f499469` through HEAD `189841d`. The job reaches the very last
step of release-please and then dies:

```
✔ Successfully updated reference release-please--branches--master--components--sase-research-artifacts
##[error]release-please failed: GitHub Actions is not permitted to create or
         approve pull requests.
```

release-please does all its real work successfully (finds 17 commits, computes the
version, builds the tree, pushes the release branch) and fails only when it calls the
"create pull request" API.

**Root cause: the repo is missing the `SASE_RELEASE_TOKEN` secret.**

`.github/workflows/publish.yml` resolves its token as:

```yaml
token: ${{ secrets.SASE_RELEASE_TOKEN || secrets.GITHUB_TOKEN }}
```

`sase-research-artifacts` has **no repository secrets at all**
(`gh secret list --repo sase-org/sase-research-artifacts` returns nothing), so the
expression falls back to `secrets.GITHUB_TOKEN`. GitHub then blocks the PR creation
because both the org and the repo have Actions PR creation disabled:

```
$ gh api repos/sase-org/sase-research-artifacts/actions/permissions/workflow
{"default_workflow_permissions":"read","can_approve_pull_request_reviews":false}
```

(`can_approve_pull_request_reviews` is the API name for the UI checkbox "Allow GitHub
Actions to create and approve pull requests".)

This was confirmed by differential comparison against the sibling plugin repos whose
Publish workflows succeed:

| repo                      | `release` job in publish.yml | `SASE_RELEASE_TOKEN` secret | Publish result |
| ------------------------- | ---------------------------- | --------------------------- | -------------- |
| `sase-github`             | byte-identical               | present (2026-06-08)        | success        |
| `sase-telegram`           | byte-identical               | present (2026-06-08)        | success        |
| `sase-research-artifacts` | byte-identical               | **absent**                  | failure (6/6)  |

All three repos are public and all three have `can_approve_pull_request_reviews: false`.
The workflow file, `release-please-config.json`, and `.release-please-manifest.json` are
all correct — the only difference is the missing PAT. `sase-research-artifacts` is a
newer repo, so the secret-provisioning step that `sase-github` and `sase-telegram`
received on 2026-06-08 was simply never performed for it.

**Consequences of the failure:** zero PRs have ever been opened on the repo, zero GitHub
releases exist, and nothing has been published to PyPI. Each failed run also left the
pushed release branch behind, so the remote now carries two orphaned branches —
including one under the pre-rename component name:

```
refs/heads/release-please--branches--master--components--sase-research
refs/heads/release-please--branches--master--components--sase-research-artifacts
```

## Approach

The primary fix is a credential/settings change, not a code change; the workflow file is
already correct and must not be "fixed". Two options were considered:

- **Option A (recommended): provision `SASE_RELEASE_TOKEN`.** Matches the sibling repos
  exactly and is clearly what the workflow's `||` fallback was written to expect.
  Requires the user to designate which PAT to store.
- **Option B (fallback): enable "Allow GitHub Actions to create and approve pull
  requests" on the repo.** Needs no secret and can be done entirely with `gh api`, but
  it diverges from the sibling repos and has a real downside: pull requests created by
  `GITHUB_TOKEN` do **not** trigger other workflows, so the release PR would never get a
  CI run before merge.

This plan implements **Option A**, with Option B recorded as the documented fallback if
the user declines to provision a token.

Because step 1 writes a credential, the implementing agent MUST confirm the exact token
with the user (via the `/sase_gate` skill) before setting the secret. Do not reuse,
copy, or infer a token value without that confirmation — in particular, do not assume
the local `gh auth token` is the intended `SASE_RELEASE_TOKEN`.

## Implementation Steps

1. **Provision the release token.** Ask the user which PAT to use for
   `SASE_RELEASE_TOKEN` (it should be the same PAT already stored in `sase-github` and
   `sase-telegram`; secret values cannot be read back from GitHub, so the user must
   supply it). Confirm through `/sase_gate`, then set it:

   ```bash
   gh secret set SASE_RELEASE_TOKEN --repo sase-org/sase-research-artifacts
   ```

   Prefer reading the value from stdin or an env var rather than passing it as a literal
   argument, and never echo it into the transcript or a file.

   Ask the user whether to instead set it once as an **organization** secret visible to
   all `sase-org` repos:

   ```bash
   gh secret set SASE_RELEASE_TOKEN --org sase-org --visibility all
   ```

   That variant fixes this repo and immunizes every future plugin repo against the same
   bootstrap gap. It is the better structural fix, but it is a broader credential-scope
   change, so it is the user's call.

   If the user declines to provision any token, fall back to Option B instead:

   ```bash
   gh api -X PUT repos/sase-org/sase-research-artifacts/actions/permissions/workflow \
     -F default_workflow_permissions=read \
     -F can_approve_pull_request_reviews=true
   ```

   and record in the final report that the release PR will not receive CI runs.

2. **Delete the two orphaned release-please branches.** No PR references either branch
   (`gh pr list --state all` returns `[]`), and the `--sase-research` one is dead weight
   from the `807e209` component rename. Removing both lets release-please recreate a
   clean branch under the current component name:

   ```bash
   gh api -X DELETE repos/sase-org/sase-research-artifacts/git/refs/heads/release-please--branches--master--components--sase-research
   gh api -X DELETE repos/sase-org/sase-research-artifacts/git/refs/heads/release-please--branches--master--components--sase-research-artifacts
   ```

3. **Re-run Publish and verify the release PR appears.** Re-run the failed run for
   master HEAD rather than pushing an empty commit:

   ```bash
   gh run rerun 31750586877 --repo sase-org/sase-research-artifacts --failed
   ```

   Wait for it through the `/sase_monitor` skill, then verify:
   - the `release` job concludes `success`;
   - `gh pr list --repo sase-org/sase-research-artifacts` shows an open
     `chore(main): release 0.1.0`-style PR targeting `master`;
   - the release PR bumps `version` in `pyproject.toml` and
     `.release-please-manifest.json` to `0.1.0` and writes `CHANGELOG.md`;
   - if the token was provisioned (Option A), CI also runs **on the release PR** — this
     is the observable difference from Option B.

   The `build`, `install-smoke`, and `publish` jobs are gated on
   `release_created == 'true'`, which is only true once the release PR is _merged_. They
   are expected to be skipped on this run; that is correct behavior, not a new failure.

4. **Report, do not merge.** Merging the release PR cuts `v0.1.0` and publishes
   `sase-research-artifacts` to PyPI through the `pypi` environment. That is a real,
   externally visible release, so stop here and report status. Ask the user before
   merging.

## Verification

- `actstat --repo sase-org/sase-research-artifacts -n 1` reports the master HEAD commit
  as passing, with no `✘ Publish` entry.
- An open release PR exists on the repo.
- `git ls-remote --heads origin` on the repo shows `master` plus exactly one freshly
  created `release-please--branches--master--components--sase-research-artifacts`
  branch, with no stale `--sase-research` branch.

## Notes and Constraints

- **No files in `sase-research-artifacts` need to change.** If verification suggests
  otherwise, re-read the diagnosis above before editing `.github/workflows/publish.yml`
  — the workflow is byte-identical to the two repos that publish successfully.
- Use the `/sase_repo` skill to obtain the checkout path for `sase-research-artifacts`
  before reading or writing any of its files.
- Treat the token as a secret at all times: no plaintext in the plan, the transcript,
  commit messages, shell history, or any file.

### Out of scope (discovered, do not fix here)

The primary `sase-org/sase` repo's Publish workflow is **also** failing on its last four
master commits (`9df15db`, `d1e8815`, `2b64c55`, `800033d`). That is a separate repo
with a different workflow — its publish.yml references `secrets.SASE_RELEASE_TOKEN` with
no `GITHUB_TOKEN` fallback — and it was not diagnosed as part of this plan. It should be
filed as its own task bead via `/sase_new_task` rather than folded into this one.
