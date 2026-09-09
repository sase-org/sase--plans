---
tier: epic
title: Restore plugin git fallback and publish bugyi-chops 0.7.0
goal: "Catalog plugin installs automatically fall back to the repository only when
  public PyPI definitively lacks the distribution, while bugyi-chops has a green,
  trusted-publishing release path and its first PyPI release is verifiably published as
  0.7.0.

  "
phases:
  - id: git_fallback
    title: Definitive index-to-git fallback
    depends_on: []
    description:
      "git_fallback: add typed public-index availability probing and apply the
      conservative fallback consistently across CLI, ACE, batch, and required-plugin
      install planning."
    size: medium
  - id: chops_release_readiness
    title: bugyi-chops release readiness
    depends_on: []
    description:
      "chops_release_readiness: repair the red typed-launch integration tests, harden
      the trusted-publishing artifact handoff, and correct installation documentation
      without changing the unreleased 0.7.0 version."
    size: small
  - id: integrated_verification
    title: Cross-repository release gate
    depends_on:
      - git_fallback
      - chops_release_readiness
    description:
      "integrated_verification: run the full SASE and bugyi-chops verification lanes,
      confirm fallback behavior at every install surface, and require a green finalized
      bugyi-chops default-branch commit before tagging."
    size: xsmall
  - id: pypi_release
    title: Publish and verify bugyi-chops 0.7.0
    depends_on:
      - integrated_verification
    description:
      "pypi_release: tag the verified bugyi-chops default-branch commit as v0.7.0,
      monitor trusted publishing to completion, and prove the immutable PyPI artifacts
      install and report version 0.7.0."
    size: small
proposed_by: bbugyi200.athena.0dm
bead_id: sase-to
create_time: 2026-09-09 19:50:36
status: wip
---

- **PROMPT:**
  [prompts/202608/git_fallback_and_bugyi_chops_release.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/git_fallback_and_bugyi_chops_release.md)
- **BEAD:**
  [sase-to](https://github.com/sase-org/sase--beads/blob/main/pages/sase-to/README.md)

# Plan: Restore plugin git fallback and publish `bugyi-chops` 0.7.0

## Problem and verified evidence

The research sidecar report
`202608/retiring_git_plugin_installs/retiring_git_plugin_installs.md` found that the
original plugin-install design promised two catalog resolution paths: use the
distribution name by default, but use `git+<repository>` when the catalog distribution
is not published to an index. Only the explicit `-g|--git` half was implemented.
`resolve_install_spec()` and `_spec_from_entry()` currently choose the distribution for
every unforced catalog hit, so an unpublished plugin reaches uv and fails resolution.

This is not a request to retire git support. Existing uv receipts may contain git rows,
raw `git+...` arguments are a supported passthrough, and `--git` remains the explicit
force control. The missing behavior is a conservative automatic fallback for catalog
entries that are definitively absent from the public index.

The release investigation also established the current `bugyi-chops` state:

- PyPI returns 404 for `bugyi-chops`; no version has ever been published.
- The repository version is already `0.7.0`, `v0.7.0` does not exist, and the only tag
  is `v0.3.1`.
- The sole publish run, for `v0.3.1`, built successfully and failed in the PyPI action
  with `invalid-publisher`. Its OIDC claims were repository `bbugyi200/bugyi-chops`,
  workflow `.github/workflows/publish.yml`, and environment `pypi`, exactly matching the
  publisher the user has now configured.
- The workflow already has the necessary `environment: pypi` and `id-token: write`. The
  registration was the historical publishing failure; do not replace trusted publishing
  with an API token.
- Current `master` CI is red in both Python 3.12 and 3.13. Three `toobig_split`
  integration tests invoke `%if` planning or directive parsing outside their existing
  `override_flags(typed_launch_units=True)` scopes. The publish workflow runs the same
  `just check`, so tagging now would fail before the publish job.

The release target is therefore the current unreleased version, `0.7.0`. Do not rerun
the obsolete `v0.3.1` workflow: that would publish old package contents as the first
public release.

## Intended fallback contract

Catalog resolution must follow this matrix:

| Input and probe state                                                              | Planned source                                             |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Raw requirement, URL, git URL, or durable local path                               | Existing passthrough; no index probe                       |
| Catalog hit with explicit `--git`                                                  | Repository git URL; no index probe                         |
| Catalog hit, public PyPI definitively has the distribution                         | Distribution name from the index                           |
| Catalog hit, public PyPI definitively returns not found                            | Automatic repository git fallback                          |
| Catalog hit, probe is offline, times out, is malformed, or otherwise indeterminate | Preserve the index plan; never infer absence from an error |
| Catalog miss                                                                       | Existing suggestions and nonzero result                    |

Only a definitive not-found response permits automatic fallback. Today
`fetch_latest_version()` returns `None` for 404, timeout, transport failure, malformed
JSON, and missing metadata; using it directly would silently turn a PyPI outage into a
source-policy change. Introduce a typed result that separates `available`, `missing`,
and `unavailable`, while retaining the existing latest-version facade for callers that
only need an optional version.

The public PyPI endpoint remains the source of truth, consistent with SASE's current
latest-version behavior. Unknown/offline status intentionally preserves historical index
resolution so uv configuration, caches, or custom indexes still have a chance to
succeed. The automatic decision must be visible as `source: "git"` in JSON and as "from
git" in previews; a previewed automatic fallback must execute the same git plan even if
availability changes before ACE starts its durable operation.

## Phase: `git_fallback`

Implement the missing fallback in the `sase` repository without changing receipt-level
git tolerance or removing `--git`.

1. Refactor `src/sase/plugins/pypi_source.py` around an injectable typed project probe.
   A valid project response is `available` and may carry its latest version; HTTP 404 is
   `missing`; timeouts, other HTTP errors, decode/schema errors, and offline operation
   are `unavailable`. Keep `fetch_latest_version()` backward-compatible by adapting the
   typed result to `str | None` for catalog latest-version consumers.
2. Thread the probe through the console-free install planning boundary in
   `src/sase/plugins/_operations_common.py` and
   `src/sase/plugins/_operations_install.py`. Probe only catalog hits that are neither
   raw passthroughs nor explicitly forced to git. Construct `git+<entry.url>` only for
   `missing`; use the distribution for `available` and `unavailable`.
3. Give batch planning the same semantics without multiplying the timeout by the number
   of marked/required plugins. Resolve the relevant catalog entries through a bounded
   availability batch (or an equivalently bounded shared probe map), then build one
   reconstructed uv argv that may contain both index and git requirements. Continue to
   preserve existing receipt rows exactly.
4. Update the ACE single-install preview in
   `src/sase/ace/tui/modals/plugins_browser_install.py`. Its primary plan is no longer
   guaranteed to be an index plan: label it from `plan.spec.source`, do not offer a
   duplicate forced-git variant when the default already fell back, and ensure the
   durable CLI argv pins the source the user confirmed. Marked-set installs should list
   each resolved source correctly.
5. Let the existing shared planners carry the behavior into `sase plugin install`, the
   ACE marked-set path, and the `PluginsRequired` notification gate. Do not add a second
   fallback implementation in a frontend. Update parser/help text, `docs/plugins.md`,
   and other existing install documentation to explain index-first resolution,
   definitive git fallback, and `--git` as a force option.
6. Add deterministic tests for 200/404/timeout/other-HTTP/malformed responses, forced
   git, raw passthrough, offline/unknown behavior, mixed-source batch argv, dry-run
   JSON, ACE preview variants and durable argv, and required-plugin gate planning. Tests
   must inject probes and never contact PyPI.

No feature flag is needed: the behavior lands complete across every current install
surface, preserves the old index choice under uncertainty, and implements the already
documented contract rather than exposing an unfinished branch.

## Phase: `chops_release_readiness`

Use `/sase_repo` to open `gh:bbugyi200/bugyi-chops`; do not locate, clone, or fetch its
contents another way.

1. Repair the three red `toobig_split` integration tests at the test abstraction rather
   than enabling the beta globally in CI. Any helper that plans or parses `%if`
   directives must enter `override_flags(typed_launch_units=True)` for the entire SASE
   operation, including assertions before and after `launch_chop_proposals()`. Retain
   coverage that the flag is normally off outside those explicit scopes.
2. Keep the trusted-publisher identity exactly aligned with the configured tuple:
   repository `bbugyi200/bugyi-chops`, workflow `publish.yml`, environment `pypi`.
   Preserve least-privilege `id-token: write`. Harden the artifact handoff by making a
   missing `dist/` upload fail and by making the publish action's `dist/` input
   explicit; use the canonical `https://pypi.org/project/bugyi-chops/` environment URL.
3. Correct the README's claim that `sase plugin install bugyi-chops -g` is a development
   install. The normal user command is the published index install. Describe `-g`
   truthfully as a built VCS snapshot/force option and direct repository development to
   the existing editable `just install` workflow instead of inventing a per-plugin
   editable command.
4. Keep `project.version = "0.7.0"`: it is current, unreleased, and absent from PyPI.
   Build wheel and sdist from a clean tree, run `twine check`, inspect their metadata
   and file lists, and prove neither `.deps` nor test/build debris is packaged.
5. Run the repository's complete `just check` against the same SASE/core development
   setup used by Actions. The focused typed-launch tests must pass with the user/home
   flag disabled, demonstrating that test correctness no longer depends on machine
   state.

## Phase: `integrated_verification`

Gate the irreversible release on both repositories being finalized and green.

1. In `sase`, run `just install` followed by the required `just check`. Because this is
   an epic's combined tree, run `just check-full` only through `/sase_monitor` with the
   required `TESTING`/`TESTED` statuses and a concrete follow-up that inspects the
   result. Also run the focused PyPI probe, install resolution, CLI, ACE preview/batch,
   and PluginsRequired gate tests explicitly so source-policy regressions are easy to
   audit.
2. Exercise the fallback matrix with injected HTTP results and a disposable uv receipt:
   published catalog entries stay index-based, definitive 404 entries produce a `git+`
   requirement, transient failures stay index-based, mixed batches preserve all existing
   requirements, and forced/raw git behavior is unchanged. Do not mutate the operator's
   live SASE uv-tool receipt.
3. In `bugyi-chops`, rerun `just check`, the focused `toobig_split` tests, a clean
   wheel/sdist build, `twine check`, and an install/import smoke from the built wheel in
   a temporary environment. Confirm both supported CI Python versions are covered.
4. Require the host-finalized release-readiness commit to be the clean current
   `origin/master` commit and require the Python 3.12/3.13 GitHub Actions matrix for
   that exact SHA to be green. The release phase must not tag an unfinalized local-only
   commit, a red commit, or a commit that has moved behind the remote default branch.
5. Recheck immediately before handoff that PyPI still has no `bugyi-chops` project and
   that `v0.7.0` is absent both locally and remotely. Record the exact commit SHA and
   expected wheel/sdist names to be used by the release phase.

## Phase: `pypi_release`

Publish the first package version only after every gate above succeeds. This external
mutation is explicitly authorized by the user's release request, but package versions
and published tags are immutable, so keep the target exact.

1. Fetch and revalidate the recorded `bugyi-chops` commit, clean tree, green Actions
   status, absent tag, absent PyPI project, package version `0.7.0`, and tag/version
   guard. Create annotated tag `v0.7.0` at that exact `origin/master` commit and push
   only that tag; do not move or recreate an existing tag.
2. The tag must trigger `.github/workflows/publish.yml`. Use `/sase_monitor` for the
   long-running Actions wait rather than blocking the agent turn. Inspect both jobs and
   require the build artifact checks and trusted-publishing job to succeed.
3. Verify through PyPI's JSON/project endpoints that `info.version` is `0.7.0` and that
   both the expected wheel and sdist are present. Confirm the GitHub deployment used
   environment `pypi` and that the publish run's repository/workflow/environment claims
   match the configured trusted publisher.
4. Install `bugyi-chops==0.7.0` from public PyPI into a fresh temporary Python 3.12
   environment with dependencies, import `bugyi_chops`, enumerate the two console
   scripts, and verify installed metadata/version. Then confirm SASE's catalog resolver
   now chooses the index for `bugyi-chops` rather than the fallback.
5. If the tag workflow fails, inspect PyPI before retrying. A transient failure before
   upload may be rerun for the same immutable tag; a workflow/package defect requires a
   normal patch-version fix and a new tag, never deletion or movement of `v0.7.0`. If
   PyPI already contains 0.7.0, treat it as published and do not attempt a duplicate
   upload.

## Non-goals and boundaries

- Do not remove or deprecate `-g|--git`, reject raw `git+` specs, or delete
  `Requirement.git`/receipt parsing. SASE must permanently preserve git rows it did not
  create.
- Do not include the report's separate source-classification, receipt-owned inventory,
  per-plugin editable switching, local-path, or live-environment migration proposals.
- Do not modify the research sidecar report or any SASE memory file.
- Do not publish the stale `v0.3.1` contents, change the package version speculatively,
  use a long-lived PyPI token, or bypass the green default-branch release gate.
- Do not uninstall or change the operator's currently installed git-sourced
  `bugyi-chops`; this work fixes future/default resolution and publishes the package.

## Acceptance criteria

- Every catalog install surface plans git automatically only after a definitive public
  PyPI not-found response; unavailable/offline probes retain index resolution.
- Explicit `--git`, raw `git+` passthrough, existing git receipt reconstruction, and
  mixed-source batch installs remain correct and covered.
- CLI JSON, human output, ACE confirmation variants, durable execution, marked-set
  installs, and PluginsRequired gates agree on the resolved source.
- SASE documentation describes the implemented fallback contract, and `just check` plus
  the monitored epic `just check-full` are green.
- `bugyi-chops` tests are independent of the user's saved feature flags, CI is green on
  Python 3.12 and 3.13 for the tagged default-branch commit, and built artifacts pass
  metadata/file-list/install checks.
- GitHub Actions publishes `v0.7.0` through the configured `pypi` environment and OIDC
  trusted publisher; PyPI exposes `bugyi-chops` 0.7.0 with wheel and sdist, and a clean
  install succeeds.
