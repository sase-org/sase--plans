---
create_time: 2026-07-09 01:01:59
status: done
prompt: 202607/prompts/ci_sdd_companion_clone.md
tier: tale
---
# Fix CI `lint` failure: clone the SDD companion repo in the CI environment

## Problem

GitHub Actions has been red on `master` for the `sase` repo. `actstat` and the GitHub API show the **`CI` workflow's
`lint` job** failing consistently across every recent commit (the `Deploy Docs` "Smoke deployed PDF" failure seen on one
older commit was a transient deployment-propagation 404 and has since gone green on its own — it is **not** in scope
here).

The failing step is `just lint`, at its final sub-step `just validate` → `sase validate`. The CI log at the true
`master` HEAD (which already includes PR #216 "fix: avoid false init memory drift in lint") shows:

```
SASE validation
  fail   init --check
  fail   sdd validate

init --check failed (exit 1)
  ...
  Needs attention:
    run  init sdd   create or connect GitHub companion SDD repository and
                    create SDD README files and directory map
      - create .sase/sdd  create or connect the GitHub companion SDD repository
      - create .sase/sdd/README.md ...

sdd validate failed (exit 1)
  error: .../.sase/sdd: SDD path does not exist or is not a directory (invalid-root)
```

## Root cause (single, confirmed)

`sase validate` runs two checks — `init --check` and `sdd validate` — and both fail for the **same** reason: the
project's SDD content lives in a **separate private companion repository, `sase-org/sdd`**, which the SASE repo
materializes into the git-ignored path `.sase/sdd`. CI never clones or materializes that companion, so `.sase/sdd` is
absent, and:

- `sase sdd validate` resolves the SDD root to `.sase/sdd` (because `sase.yml` declares `sdd.storage: separate_repo`),
  finds no directory, and returns an `invalid-root` error (exit 1).
- `sase init --check`'s `init sdd` sub-check plans a "create or connect GitHub companion SDD repository" action plus
  README/directory-map creation actions, because there is no companion clone and no store record — so it reports "needs
  attention" (exit 1).

Why this surfaced only recently: for many prior commits `just lint` failed earlier, at the `pyvision` sub-step (private
symbols were being imported across modules), which short-circuited `just lint` before it ever reached `sase validate`.
Once the `sase-5n` epic (a) made those symbols public and (b) landed PR #216 to fix the `init memory` drift, `pyvision`
and the `init memory`/`init skills` checks began passing — exposing the underlying missing-companion failure in
`init sdd` and `sdd validate`.

Confirmations gathered during investigation:

- `.sase/` is git-ignored, so the companion content is never in the CI checkout.
- The CI `lint` job checks out `sase-org/sase` and the public `sase-org/sase-core` Rust core, but has **no** step for
  the private `sase-org/sdd`.
- `sase-org/sdd` is **private** (unlike the public `sase-core`, which is why the core is checked out with no token); the
  companion holds the real SDD tree (`epics/`, `legends/`, `myths/`, `tales/`, `prompts/`, `research/`, `assets/`,
  READMEs, directory map).
- `sase sdd validate` does **not** honor the `SASE_SDD_DIR` env var, so the content must physically live at `.sase/sdd`;
  pointing elsewhere is not an option.
- `init sdd`'s companion action is suppressed only when a store record exists with `discovery != "not_found"` (verified
  in isolation: a `discovery: "found"` `separate_repo` record makes the planner return `None`).
- Neither `sase validate` nor the CI `Initialize SASE home` step (`sase init memory --no-commit`) performs any network
  SDD clone, so a pre-placed clone + record are consumed entirely offline — there is no SSH-auth clone risk inside these
  commands.

## Goal

Make the `CI` → `lint` job green by ensuring the `sase-org/sdd` companion repo is cloned into `.sase/sdd` in the CI
environment, and that `sase validate` (both `init --check`'s `init sdd` and `sdd validate`) passes there — satisfying
the requirement that "the companion sdd repo should be cloned in the CI environment."

## Approach (high-level)

Scope the change to `.github/workflows/ci.yml`, in the **`lint` job** (the only job that runs `sase validate`; `test`,
`build`, `visual-test`, `docs-build`, etc. do not need the companion and are already green). Two additions, placed after
the existing repo checkouts and before the `Lint` step:

1. **Authenticated checkout of the companion repo into `.sase/sdd`.** Add an `actions/checkout` step for
   `repository: sase-org/sdd` with `path: .sase/sdd`, authenticated with a token that has read access to the private
   companion (candidate: the existing `SASE_RELEASE_TOKEN` secret, which is already used for cross-repo release
   automation). This gives both `sdd validate` its tree and `init sdd` its READMEs/directory-map content.

2. **Write the SDD store record `.sase/sdd-store.json`.** Write a minimal `separate_repo` record with
   `discovery: "found"` (matching the real record's shape: `provider: github`, `host: github.com`, `repo: sase-org/sdd`,
   `remote_url: git@github.com:sase-org/sdd.git`, `storage: separate_repo`, `schema_version: 1`). This makes
   `init sdd`'s "create or connect companion repository" action disappear so `init --check` passes. `sdd validate` does
   not read this record; `init --check` does.

Ordering matters: both steps must run **after** the primary `actions/checkout` of `sase-org/sase` (so the workspace
isn't cleaned over them) and before the `Lint` step. Placing them immediately after the "Check out Rust core" step is
the natural spot.

### Auth / access considerations (must verify during implementation)

- Confirm the chosen token (`SASE_RELEASE_TOKEN` or a purpose-built read token / deploy key) actually grants read to
  `sase-org/sdd`. If it does not, provision a dedicated secret (fine-grained PAT with `contents: read` on
  `sase-org/sdd`, or a read deploy key) and use that instead.
- CI runs on `push` to `master` and `pull_request` to `master`. Same-repo PRs and pushes have secret access; **fork PRs
  do not** and would fail the companion checkout. Given this is a single-maintainer org repo where PRs come from
  same-repo branches, this is acceptable — but note it explicitly, and if fork PRs ever matter, guard/skip the companion
  steps when the secret is absent.

### Alternative considered (documented, not chosen)

Instead of writing the store record in CI, the `init sdd` planner (`_plan_sdd_companion_repo_action`) could be changed
to treat an existing, remote-matching `.sase/sdd` git clone as "connected" even without a record file. That would make
cloning alone sufficient in any environment. It is a cleaner, more general behavior but is a **product code change**
(with tests) rather than a CI-config change, and the stated requirement points specifically at cloning the companion in
CI. Prefer the CI-config fix; revisit the code-level change later if the store-record step proves brittle.

## Files to change

- `.github/workflows/ci.yml` — add the companion checkout + store-record steps to the `lint` job. (Optionally factor the
  two steps into a small composite action or a tracked JSON template under `.github/` if reuse across jobs is later
  desired; not required for this fix.)

No changes to `Justfile`, `sase validate`, or the SDD Python code are required for the chosen approach.

## Verification

1. **Local sanity (non-CI portion):** on a checkout that includes PR #216 with the companion present at `.sase/sdd`,
   `just lint` completes green (`sase validate` reports `ok init --check` and `ok sdd validate`). This confirms the
   validation logic passes once the companion is in place.
2. **CI (authoritative):** after the workflow change lands (on the fix branch's PR run and again on the post-merge
   `master` run), confirm via `actstat` / `gh run view` that the `CI` → `lint` job reaches and passes the
   `Running SASE validation` step, and that the overall run is green.
3. Confirm no regression in the other `CI` jobs and that `Deploy Docs` / `Publish` remain green.

## Out of scope

- The transient `Deploy Docs` "Smoke deployed PDF" 404 (already self-resolved). If it recurs, a separate hardening (more
  retry attempts / longer backoff on the deployed-URL smoke poll) can be considered, but it is not part of this fix.
- Any change to how the SDD content itself is authored or kept in sync with the SASE renderers (the companion's
  committed READMEs/directory map are currently up to date; keeping them current is ongoing maintenance, not this fix).
