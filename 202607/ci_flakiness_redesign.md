---
tier: epic
title: CI flakiness redesign
goal: 'Every started master CI run completes with trustworthy results: no starvation-by-cancellation,
  one Rust build per run, no duplicated or drift-prone lanes, and no loss of meaningful
  coverage.

  '
phases:
- id: ci-signal-restore
  title: Restore completed-run signal and unbreak lint
  depends_on: []
  size: small
  description: 'ci-signal-restore: stop cancelling in-flight master runs, skip CI
    on the release-please branch, disable matrix fail-fast, add the missing beads
    sidecar to the lint environment, pin keep-sorted, and file beads for the two observed
    flaky tests.'
- id: core-wheel-once
  title: Build the Rust core once per run
  depends_on:
  - ci-signal-restore
  size: medium
  description: 'core-wheel-once: add a build-core root job that builds one abi3 sase_core_rs
    wheel per run, teach the Justfile a SASE_CORE_WHEEL install path, fan the wheel
    out via artifact and a setup-sase composite action, and drop the duplicated sase-core
    rust-check from bead-backend.'
- id: lane-consolidation
  title: Consolidate lanes without losing coverage
  depends_on:
  - core-wheel-once
  size: medium
  description: 'lane-consolidation: merge the three perf-floor jobs into one, delete
    the redundant install-smoke/bead-backend/build/fmt-md-check jobs after folding
    their unique steps into neighbors, run the visual suite exactly once per run,
    build docs once per event, and serialize docs deploys.'
- id: config-driven-sidecars
  title: Derive the CI sidecar environment from configuration
  depends_on:
  - lane-consolidation
  size: small
  description: 'config-driven-sidecars: replace the hand-written sidecar checkouts
    and sdd-store heredoc with a tools/ci_bootstrap_sidecars script generated from
    sase/sase.yml, with unit tests locking the store shape.'
create_time: 2026-07-28 18:05:46
status: done
bead_id: sase-am
---

- **BEAD:** [sase-am](https://github.com/sase-org/sase--beads/blob/main/pages/sase-am/README.md)
- **PROMPT:** [202607/prompts/ci_flakiness_redesign.md](prompts/ci_flakiness_redesign.md)
- **AGENTS:**
  - [bbugyi200.athena.nm](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.nm/README.md)
  - [bbugyi200.athena.sase-am.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-am.1/README.md)
  - [bbugyi200.athena.sase-am.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-am.2/README.md)
  - [bbugyi200.athena.sase-am.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-am.3/README.md)
  - [bbugyi200.athena.sase-am.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-am.4/README.md)
  - [bbugyi200.athena.sase-am.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-am.land/README.md)

# Redesign GitHub Actions CI to Eliminate Unnecessary Flakiness

## Problem

The sase repo's CI is currently producing almost no usable signal, and the signal it does produce is diluted by several
structural flakiness sources. Evidence gathered from the live repo (2026-07-28):

- **Zero successful CI runs in the last 200.** `gh run list` shows 192 cancelled, 6 failed, 2 in progress, 0 successful.
  The `ci-${{ github.ref }}` concurrency group with `cancel-in-progress: true` cancels the in-flight master run on every
  push. With ~50–90 minute wall time per run and a push cadence faster than that, master CI (and the constantly
  force-updated `release-please--branches--master` PR) never completes. `just workflow-status` has no completed workflow
  set to report on.
- **`lint` is failing on every master run that survives long enough to report.** `sase validate` runs
  `sase init repo --check`, which now requires the `beads` sidecar configured in `sase/sase.yml`, but
  `.github/workflows/ci.yml` hand-checks-out only the `plans` and `research` sidecars and hand-writes a two-sidecar
  `.sase/sdd-store.json`. The CI environment is a manual mirror of `sase/sase.yml` and drifts every time the sidecar
  config changes.
- **~30 runner slots consumed per master push.** Each push triggers 14 CI jobs for the master ref, another 14 for the
  force-updated release-please PR branch (which only ever carries version/changelog bumps and whose runs are always
  cancelled by the next push), plus docs-deploy and the publish release job. This is the org-wide runner queue pressure
  that motivated `cancel-in-progress: true` in the first place — the cure became the disease.
- **Eight jobs independently check out sase-core master HEAD and compile `sase_core_rs` from scratch.** Jobs in the same
  run can build different sase-core commits if sase-core receives a push mid-run. Every job pays the rustup toolchain
  download, crates.io dependency downloads (the Justfile already carries `CARGO_NET_RETRY` /
  `CARGO_HTTP_MULTIPLEXING=false` workarounds for observed HTTP2 framing flakes), and a full release build. Observed
  logs also show `Failed to restore: Cache service responded with 400` cache flakes.
- **`bead-backend` duplicates sase-core's own CI.** `just rust-check` re-runs `cargo fmt --check`,
  `cargo clippy -D warnings`, and `cargo test --workspace` against the sase-core checkout. sase-core's own CI
  (`.github/workflows/ci.yml` in sase-org/sase-core, job `rust-checks`) already runs exactly these with
  `Swatinem/rust-cache`. A new stable Rust release can fail sase CI on clippy lints for another repo's code.
- **The visual snapshot suite runs twice per push**: once inside the `test` matrix 3.12 leg (`test-cov` includes visual)
  and again in the dedicated `visual-test` job. Two chances to flake for one signal, and the visual tests slow the
  longest matrix leg — the 3.12 coverage leg has been observed hitting its 90-minute ceiling.
- **Matrix `fail-fast` (default true) destroys sibling-leg signal.** Observed: a 3.14-only test failure cancelled the
  in-flight 3.12 (coverage) and 3.13 legs, so one flaky leg erases the coverage gate and the other version signals.
- **`install-smoke` adds no unique coverage.** Its editable `just install` + `sase core health --json` is subsumed by
  every other job that installs the package and runs tests against the Rust binding. (The _published-wheel_ smokes —
  `published-core-minimum-smoke` in CI and `install-smoke` in publish.yml — do carry unique coverage and stay.)
- **Three perf-floor jobs (`phase7-perf-floor`, `launch-perf-floor`, `view-hints-perf-floor`) each pay a full setup
  (~5–10 min of checkouts, toolchains, Rust build, dependency install) for ~1–2 minutes of measurement.** Three
  shared-runner warm-ups triple the noisy-neighbor exposure of ratio-based latency gates.
- **Docs are built twice per master push** (`docs-build` in ci.yml and the identical steps in docs-deploy.yml),
  `docs-deploy` has no concurrency group (two concurrent deploys can finish out of order, deploying a stale commit
  last), and both lanes re-download the Playwright Chromium browser on every run.
- **Tool version drift**: CI installs `keep-sorted@latest` (cached under a key derived from `ci.yml`'s hash) while the
  Justfile pins `v0.8.0`. A cache eviction can silently change lint behavior with no repo change.

Not in scope (test-level, not CI-design): the observed flaky tests
`tests/ace/tui/test_artifacts_plans_filtering.py::test_deep_archive_typing_burst_fetches_once_and_becomes_exact`
(typing-burst race, one extra fetch) and
`tests/test_bead/test_cli_work_epic_lifecycle.py::test_work_missing_bead_json_error_is_one_envelope` (pytest-sandbox
isolation guard). File beads for these during Phase 1 so they are tracked separately.

Also verified: the repo has **no branch protection and no rulesets**, so renaming, merging, or deleting jobs cannot
break required status checks. CI here is an informational/trailing signal (pushes land directly on master; the real
pre-push gate is the local `just check` / vcs hooks), which makes _completed-run reliability_ the top priority.

## Goals

1. Every started master CI run completes (success or genuine failure) — no more starvation-by-cancellation.
2. One Rust compile per CI run; no per-job crates.io/rustup exposure; all jobs in a run test the same sase-core commit.
3. Every retained check keeps its coverage; redundant executions (not redundant coverage) are removed.
4. The CI sidecar environment is derived from `sase/sase.yml` instead of hand-mirrored.
5. Fewer runner slots per push (~30 → ~10) to relieve the org-wide queue that motivated the aggressive cancellation in
   the first place.

## Non-goals

- Fixing individual flaky tests (beads filed instead).
- Gating releases on CI (publish.yml semantics unchanged; there is no branch protection by design).
- Switching CI dependency resolution to `uv.lock`-enforced installs (`uv pip install -e .[dev]` resolves fresh from PyPI
  today — a real drift class, but changing the install strategy affects local dev too; left as a noted follow-up).
- Changing pr-title.yml or publish.yml (publish's release-please retry ladder and `skip-existing` already handle its
  known flake classes).

## Phases

### Phase 1 — Restore completed-run signal and unbreak lint

- **Slug**: `ci-signal-restore`
- **Depends on**: nothing

Smallest set of edits to `.github/workflows/ci.yml` that makes CI produce trustworthy completed runs again:

1. **Stop cancelling in-flight master runs.** Change the concurrency block to
   `cancel-in-progress: ${{ github.ref != 'refs/heads/master' }}`. GitHub's concurrency grouping already bounds the
   queue to one running + one pending run per ref (newer pushes replace the _pending_ slot), so the original motivation
   (unbounded org-queue growth) stays solved while every started master run now completes. PR refs keep latest-wins
   cancellation. Update the explanatory comment above the block accordingly.
2. **Skip all ci.yml jobs on the release-please PR branch.** Add to every job:
   `if: github.event_name != 'pull_request' || github.event.pull_request.head.ref != 'release-please--branches--master'`.
   That branch only carries version/changelog bumps, its runs have never completed (always superseded), and its merge
   triggers a full master-push CI run anyway. This alone halves the CI slots consumed per master push. (Phase 2
   centralizes this condition on the new root job; keep it simple per-job here.)
3. **`strategy: fail-fast: false`** on the `test` matrix so a single flaky leg no longer cancels the 3.12 coverage leg
   and the other version signals.
4. **Unbreak `sase validate` in lint**: add a checkout step for the beads sidecar (repo `sase-org/sase--beads` — verify
   the exact repo name with `gh repo view` before wiring; pattern per existing sidecars is `<project>--<name>`) at
   `sase/repos/beads` with `token: ${{ secrets.SASE_RELEASE_TOKEN }}` and `persist-credentials: false`, and add the
   matching `beads` entry to the `.sase/sdd-store.json` heredoc. Confirm locally-reproduced `sase init repo --check`
   behavior: the `agents` sidecar currently only produces a warning ("skipped agents sidecar planning") and needs no
   checkout; do not add it unless `--check` fails without it.
5. **Pin keep-sorted in CI to the Justfile's version** (`go install github.com/google/keep-sorted@v0.8.0`) and key the
   Go-bin cache on that version string (e.g. `go-bin-keep-sorted-v0.8.0-linux-amd64`) instead of
   `hashFiles('.github/workflows/ci.yml')`, so workflow edits stop reinstalling a floating `@latest`.
6. **File beads** for the two flaky tests listed in the Problem section (test-level fixes, tracked separately).

**Acceptance**: the next master push's CI run reaches a terminal conclusion that is not `cancelled`; the `lint` job
passes `just validate`; a release-please PR sync shows all ci.yml jobs skipped; `gh run list` over the following days
shows completed (not cancelled) master runs.

### Phase 2 — Build the Rust core once per run

- **Slug**: `core-wheel-once`
- **Depends on**: `ci-signal-restore`

Replace eight independent sase-core checkouts + compiles with one wheel build fanned out via artifact:

1. **New root job `build-core`** in ci.yml:
   - Checks out sase and sase-core (sase-core once per run; record/echo the resolved sase-core SHA so every downstream
     job provably tests the same core commit).
   - `dtolnay/rust-toolchain` + `Swatinem/rust-cache` keyed on the sase-core commit (mirroring sase-core's own CI, which
     builds its abi3 wheel with `maturin` in `crates/sase_core_py`).
   - Builds the release abi3 wheel (`maturin build --release --out dist` in `sase-core/crates/sase_core_py`, e.g. via
     `uvx maturin` or `PyO3/maturin-action@v1` as sase-core CI does) and uploads it as a `sase-core-wheel` artifact.
     abi3 makes one wheel valid for all three matrix Pythons.
   - Carries the release-please-branch skip condition from Phase 1; downstream jobs use `needs: build-core`, so they
     skip automatically and the per-job `if` lines from Phase 1 can be dropped.
2. **Justfile support for installing from a prebuilt wheel.** Add a `SASE_CORE_WHEEL` environment variable honored by
   `install`, `install-visual`, `_setup`, `_setup-visual` (and `_setup-terminal-smoke` for uniformity): when set to a
   wheel path, skip the `rust-install` source build and instead `uv pip install --python {{ venv_bin }}/python <wheel>`
   before editable resolution, passing the existing version-window overrides file (reuse the `_core-overrides-arg`
   mechanism, extended to also trigger when `SASE_CORE_WHEEL` is set) so the `sase-core-rs>=X,<Y` window in
   pyproject.toml cannot downgrade or replace the wheel during `-e ".[dev]"` resolution. Behavior with the variable
   unset is unchanged (local dev keeps building from the linked checkout). This must be testable locally: build a wheel
   from the linked sase-core checkout, then run `SASE_CORE_WHEEL=... just install` and `sase core health --json`.
3. **Composite action** `.github/actions/setup-sase` to end the 6-way step duplication: inputs for Python version and
   install recipe (`install` / `install-visual`); steps: pinned `just` download with
   `curl --retry 5 --retry-all-errors` + existing sha256 verification, `astral-sh/setup-uv` with cache,
   `uv python install <version>`, download of the `sase-core-wheel` artifact, and
   `SASE_CORE_WHEEL=<path> just <recipe>`.
4. **Convert downstream jobs** (`lint`, `test` matrix, `visual-test`, `build`, `bead-backend`, the perf-floor jobs until
   Phase 3 merges them): `needs: build-core`, use the composite action, and delete their sase-core checkout, Rust
   toolchain, and `SASE_CORE_DIR` env. `published-core-minimum-smoke` is intentionally untouched (it must keep resolving
   the published minimum from PyPI with no local core).
5. **Drop `just rust-check` from `bead-backend`.** sase-core's own CI owns cargo fmt/clippy/test for sase-core
   (verified: job `rust-checks` in sase-org/sase-core ci.yml runs the identical three commands with rust-cache). The job
   keeps its Python bead parity/CLI pytest subset and `bead-perf-smoke` until Phase 3.
6. Verify no remaining test in the retained jobs needs the sase-core _source tree_ (grep tests for references to the
   checkout path / `SASE_CORE_DIR`); the binding wheel plus sase sources must be sufficient.

**Acceptance**: a full CI run is green with exactly one cargo/maturin build across all jobs (verify in logs: no
"Building sase_core_rs" or crates.io downloads outside `build-core`); wall time of the slowest job drops measurably; all
jobs in one run print the same sase-core SHA.

### Phase 3 — Consolidate lanes without losing coverage

- **Slug**: `lane-consolidation`
- **Depends on**: `core-wheel-once`

Remove duplicate executions and fold tiny jobs into neighbors. Every check retained, fewer runner slots and fewer
independent flake opportunities:

1. **One `perf-floors` job** (needs `build-core`) replacing `phase7-perf-floor`, `launch-perf-floor`, and
   `view-hints-perf-floor`: after one setup, run `.venv/bin/sase core health --json` (absorbing the only unique
   assertion of `install-smoke`), then `just phase7-perf-check`, `just launch-perf-check`, `just view-hints-perf-check`,
   and `just bead-perf-smoke` sequentially; upload all four artifacts with `if: always()`. Structure the steps so a
   failing floor does not mask the later floors (e.g. `continue-on-error: false` per step but independent step-level
   artifact uploads, or a final aggregation step) — the job must still fail if any floor fails.
2. **Delete `install-smoke`** (unique assertion moved into `perf-floors`).
3. **Delete `bead-backend`.** Its pytest subset (`tests/test_bead`, `tests/test_core_facade/test_bead_read.py`,
   `tests/test_core_facade/test_bead_mutation.py`) already runs inside every `test` matrix leg via `test-cov` (verify
   before deleting: confirm those paths carry no marker excluding them from the fast suite), its `rust-check` left in
   Phase 2, and `bead-perf-smoke` moves to `perf-floors`. The job's original purpose — catching stale locally-rebuilt
   bindings before the matrix obscures the cause — is obsolete once all jobs install the same freshly built wheel.
4. **Fold `fmt-md-check` into `lint`** as a step (`just fmt-md-check` already bootstraps repo-local Prettier via
   `npm ci`); add an `actions/cache` entry for `node_modules` keyed on `hashFiles('package-lock.json')` so the npm
   network fetch stops being a per-run flake point.
5. **Fold `build-check` into `lint`** as a final step (`just build-check`) and delete the `build` job — same venv, ~1–2
   extra minutes, one fewer slot. (Release packaging keeps its own independent build in publish.yml.)
6. **Run the visual suite exactly once per run**: set `SASE_PYTEST_EXCLUDE_VISUAL: "true"` on _all_ `test` matrix legs;
   `visual-test` (with its failure-report tooling) becomes the sole visual lane. Before landing, verify the 50% coverage
   gate still passes without visual tests contributing (run the cov suite once with the exclusion; if coverage lands
   below the gate, keep visual in the 3.12 leg and note it — do not lower the gate).
7. **`docs-build` runs only on `pull_request` events** (`if: github.event_name == 'pull_request'`): master pushes get
   the identical `docs-check` / `docs-pdf-check` / `docs-deploy-artifact-check` steps from docs-deploy.yml, so pushes
   currently build docs twice.
8. **docs-deploy.yml hardening**: add `concurrency: { group: docs-deploy, cancel-in-progress: false }` so concurrent
   deploys cannot finish out of order (stale-commit-deployed-last race), and cache the Playwright browser directory
   (`~/.cache/ms-playwright`) in both docs lanes to remove the per-run Chromium download.
9. **Revisit `timeout-minutes`** with post-Phase-2 durations (e.g. `test` 90 → a value with ~2× observed headroom); keep
   every job's explicit ceiling.

**Acceptance**: per-master-push CI job count drops to ~8 (`build-core`, `lint`, `test` ×3, `visual-test`, `perf-floors`,
`published-core-minimum-smoke`); all retained checks still execute (verify by enumerating the old jobs' commands against
the new layout); a full run is green; visual tests execute exactly once per run.

### Phase 4 — Derive the CI sidecar environment from configuration

- **Slug**: `config-driven-sidecars`
- **Depends on**: `lane-consolidation`

Eliminate the drift class that broke lint (Problem #2) instead of just patching it:

1. **New script `tools/ci_bootstrap_sidecars`** (follow the tools/ directory conventions enforced by the pyscripts
   lint): reads the `repos.sidecar` list from `sase/sase.yml`, and for each sidecar required by `sase init repo --check`
   in CI (plans, beads, research today; skip the hidden `agents` sidecar while the check merely warns), shallow-clones
   `sase-org/<project>--<name>` to `sase/repos/<name>` using a token from the environment, with bounded retries, and
   writes the complete `.sase/sdd-store.json` (schema_version 2, `storage: sidecar_repos`) covering every cloned
   sidecar. Fail with an actionable message naming any sidecar repo that cannot be cloned.
2. **Replace** the lint job's hand-written sidecar checkout steps and sdd-store heredoc with a single step invoking the
   script (pass `SASE_RELEASE_TOKEN` via env). The `plans` checkout used by `just validate-committed-plans` must keep
   landing at the same path.
3. **Unit-test the script's pure parts** (config parsing, store-record generation) so the store shape is locked by tests
   rather than by a YAML heredoc in a workflow file.

**Acceptance**: lint is green; adding or removing a sidecar in `sase/sase.yml` requires no ci.yml edit (dry-run
demonstration: the script's generated store matches what `sase init repo --check` expects for the current config); the
heredoc is gone from ci.yml.

## Risks and mitigations

- **`cancel-in-progress: false` on master increases concurrent-run overlap** (one running + one pending): the slot
  reductions in Phases 1–3 (~30 → ~10 per push) more than compensate; if org-queue pressure reappears, the pending slot
  is still bounded at one.
- **Wheel-based installs diverge from the editable-checkout dev path**: the `SASE_CORE_WHEEL` path reuses the existing
  overrides mechanism and is verified locally in Phase 2 (`sase core health --json` + full test run);
  `published-core-minimum-smoke` continues to cover the published-wheel resolution path unchanged.
- **Coverage gate sensitivity to removing visual tests from the cov leg** (Phase 3 step 6) has an explicit verify-first
  fallback.
- **Perf floors remain shared-runner-noise-sensitive**: unchanged thresholds and margins; consolidation only removes
  redundant warm-ups. If floors show noise-flakes after consolidation, that is a threshold question for a separate
  change, not this epic.
- **Deleting `bead-backend`/`install-smoke`/`build`/`fmt-md-check` cannot break required status checks**: the repo has
  no branch protection or rulesets (verified via API).

## Follow-ups (explicitly not in this epic)

- Fix the two flaky tests (beads filed in Phase 1).
- Consider lockfile-enforced CI installs (`uv.lock`) to close the fresh-resolution drift class.
- Consider a scheduled (cron) full-matrix + perf run if push-time CI cost needs to shrink further.
