---
tier: epic
title: Fix GitHub Actions failures (sase-core clippy + published-core minimum)
goal: 'sase-core master CI and sase master CI are both fully green: the clippy lints
  from the close-note change are fixed, sase-core-rs 0.12.5 (plan-header wire schema
  2) is published to PyPI, and the sase repo requires it as its published-core minimum.

  '
phases:
- id: fix-core-clippy-and-release
  title: Fix sase-core clippy lints and release 0.12.5
  depends_on: []
  size: small
  description: 'fix-core-clippy-and-release: allow too_many_arguments on close_issues_with_note,
    drop two clone-on-Copy calls in tests, land the fix on sase-core master, then
    merge the release-plz PR and verify sase-core-rs 0.12.5 reaches PyPI with plan-header
    schema 2.'
- id: bump-published-core-minimum
  title: Bump the sase published-core minimum to 0.12.5
  depends_on:
  - fix-core-clippy-and-release
  size: small
  description: 'bump-published-core-minimum: raise the sase-core-rs floor in pyproject.toml
    to 0.12.5, regenerate uv.lock, rerun the published-core smoke gates locally, and
    land the bump so master CI goes green.'
create_time: 2026-07-28 17:36:55
status: wip
bead_id: sase-al
---

- **BEAD:** [sase-al](https://github.com/sase-org/sase--beads/blob/main/pages/sase-al/README.md)
- **PROMPT:** [202607/prompts/fix_ci_core_clippy_and_minimum.md](prompts/fix_ci_core_clippy_and_minimum.md)
- **AGENTS:**
  - [bbugyi200.athena.nj](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.nj/README.md)
  - [bbugyi200.athena.sase-al.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-al.1/README.md)
  - [bbugyi200.athena.sase-al.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-al.2/README.md)
  - [bbugyi200.athena.sase-al.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-al.land/README.md)

# Fix GitHub Actions failures: sase-core clippy lints + published-core minimum smoke

## Context / Diagnosis

GitHub Actions is red on `sase-org/sase` master (commit d67de4ca, CI run #11850) and on `sase-org/sase-core` master
(commit e098a1a, CI run #720). There are two independent root causes:

1. **Clippy violations in sase-core**, introduced by sase-core commit e098a1a ("feat(beads): support atomic close
   notes"):
   - `crates/sase_core/src/bead/mutation.rs:748` — `pub fn close_issues_with_note` has 8 arguments
     (`clippy::too_many_arguments`, limit 7).
   - `crates/sase_core/src/bead/mutation.rs:3667` (test code) — `.map(|event| event.operation.clone())` where
     `BeadEventOperationWire` is `Copy` (`clippy::clone_on_copy`).
   - `crates/sase_core/src/bead/cli.rs:2828` (test code) — identical `clone_on_copy` violation.

   These fail sase-core's "cargo fmt + clippy + test" job AND the sase repo's `bead-backend` job, which checks out
   sase-core master and runs `just rust-clippy` (clippy with `-D warnings`).

2. **Plan-header wire schema mismatch in the sase repo's `published-core-minimum-smoke` job**:
   `tools/smoke_sase_core_rs_plan_header` expects `WIRE_SCHEMA_VERSION = 2` (bumped by sase commit ab1c36040,
   "feat(plan): project bead links into plan headers (sase-ai.8)"), but the job installs the _exact published minimum_
   parsed from `pyproject.toml` (`sase-core-rs>=0.12.4,<0.13.0` → installs `0.12.4` from PyPI), and published 0.12.4
   still reports schema version 1. Schema v2 lives on sase-core master (`crates/sase_core/src/plan/artifact_link.rs:16`,
   landed in c81b144 _after_ the v0.12.4 release) and has not been published yet. sase-core has an open release-plz PR
   ("chore: release v0.12.5", PR #42) that will publish it.

The other red jobs in sase run #11850 (test 3.12/3.13/3.14, lint, visual-test, view-hints-perf-floor) were collateral
cancellations of the same run, not independent failures.

**Ordering constraint:** the clippy fix must land first (sase-core CI must be green for a clean release), then
`sase-core-rs 0.12.5` must be published to PyPI, and only then can the sase repo bump its published-core minimum — the
smoke job installs the exact minimum from PyPI, and regenerating `uv.lock` also requires the version to exist on PyPI.

## Phase 1: fix-core-clippy-and-release

Dependencies: none.

All file changes in this phase are in the **sase-core** repo. Open it with the `/sase_repo` skill
(`sase repo open sase-core -r "<reason>"`) and work only in the path it prints.

1. In `crates/sase_core/src/bead/mutation.rs`, add `#[allow(clippy::too_many_arguments)]` directly above
   `pub fn close_issues_with_note(` (~line 748). This follows existing repo precedent (e.g.
   `crates/sase_core/src/plan/search.rs:84`, `crates/sase_core/src/content_layout.rs:510`). Do NOT refactor the
   signature: `close_issues` delegates to this function, and the pyo3 binding (`crates/sase_core_py/src/lib.rs:3192`)
   and bead CLI (`crates/sase_core/src/bead/cli.rs:932`) both pass all 8 arguments positionally.
2. In `crates/sase_core/src/bead/mutation.rs` (~line 3667, inside the close-note test), change
   `.map(|event| event.operation.clone())` to `.map(|event| event.operation)`.
3. In `crates/sase_core/src/bead/cli.rs` (~line 2828, inside the close-note CLI test), make the identical change.
4. Validate from the sase-core checkout: `cargo fmt --all -- --check`, then
   `cargo clippy --workspace --all-targets -- -D warnings`, then `cargo test --workspace`. (Equivalently, run
   `just rust-check` from the sase workspace, which wraps all three against the linked checkout.)
5. Commit to sase-core master via the sase commit workflow (suggested message:
   `fix(beads): resolve clippy lints in close-note support`) and wait for sase-core CI on master to go green.
6. release-plz automatically refreshes the open release PR ("chore: release v0.12.5") to include the fix. Once the
   refreshed release PR's CI is green, merge it and wait for the release workflow to publish `sase-core-rs 0.12.5` to
   PyPI. Verify with `curl -s https://pypi.org/pypi/sase-core-rs/json` that the latest version is `0.12.5`, then confirm
   the published wheel reports plan-header schema 2: install `sase-core-rs==0.12.5` into a throwaway venv and run the
   sase repo's `tools/smoke_sase_core_rs_plan_header` with that interpreter (must exit 0).

Completion criteria: sase-core master CI green, and `sase-core-rs 0.12.5` live on PyPI reporting plan-header wire schema
version 2.

## Phase 2: bump-published-core-minimum

Dependencies: fix-core-clippy-and-release.

All file changes in this phase are in the **sase** repo (the primary workspace repo).

1. In `pyproject.toml`, change the runtime dependency `"sase-core-rs>=0.12.4,<0.13.0"` to
   `"sase-core-rs>=0.12.5,<0.13.0"`.
2. Regenerate the lock entry: `uv lock --upgrade-package sase-core-rs` (`uv.lock` currently pins 0.12.4 with hashes),
   then run `just install` to sync the venv.
3. Replicate the failing CI job locally before committing: create a venv with the exact new minimum
   (`uv venv /tmp/published-core-smoke && uv pip install --python /tmp/published-core-smoke/bin/python "sase-core-rs==0.12.5"`)
   and run all four smoke gates with that interpreter — `tools/smoke_sase_core_rs_telemetry`,
   `tools/check_sase_core_rs_bindings`, `tools/smoke_sase_core_rs_bead_resolution`, and
   `tools/smoke_sase_core_rs_plan_header`. All must exit 0.
4. Run `just check` (mandatory for file changes in this repo).
5. Commit via the sase commit workflow (suggested message: `build(deps): require sase-core-rs 0.12.5`, matching prior
   bumps such as 702f1aece).
6. After the commit lands on master, watch the master CI run (`actstat` or `gh run watch`) and confirm it goes fully
   green — in particular the `bead-backend` and `published-core-minimum-smoke` jobs.

Completion criteria: sase master CI fully green.
