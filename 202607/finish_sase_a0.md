---
tier: epic
title: Finish and land sase-a0 after typed-resolution integration
goal: Current master passes every original sase-a0 CI gate and the later typed close-resolution
  feature against the exact published sase-core-rs minimum, then sase-a0 is closed,
  post-close symvision cleanup is completed, and its linked plan is marked done.
phases:
- id: core-release
  title: Publish the typed-resolution core
  depends_on: []
  size: small
  description: 'core-release: publish and verify the next sase-core-rs patch containing
    the typed close-resolution bindings and compatible bead_close signature.'
- id: minimum-integration
  title: Raise and exercise the published minimum
  depends_on:
  - core-release
  size: medium
  description: 'minimum-integration: raise SASE''s core floor, add a semantic typed-resolution
    exact-minimum smoke, and reverify every original epic fix.'
- id: verify-and-land
  title: Settle CI, close, clean, and finalize
  depends_on:
  - minimum-integration
  size: medium
  description: 'verify-and-land: require a settled green CI run, close sase-a0, perform
    post-close symvision cleanup, and mark the linked canonical plan done.'
parent_bead: sase-a0
create_time: 2026-07-27 13:13:35
status: wip
bead_id: sase-a0.5
---

- **PROMPT:** [202607/prompts/finish_sase_a0.md](prompts/finish_sase_a0.md)
- **PARENT:** [202607/fix_ci_failures.md](https://github.com/sase-org/sase--plans/blob/main/202607/fix_ci_failures.md)
- **BEAD:** [sase-a0.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-a0/sase-a0.5.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-a0.5.2--1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-a0.5.2.md#member-1)

# Finish and land `sase-a0` after the typed-resolution integration

## Goal

Restore the guarantee promised by epic bead `sase-a0`: current `master` must pass `published-core-minimum-smoke`, the
original lint and performance fixes must remain effective, the two Python 3.14 failures must stay fixed without
ambient-environment assumptions, and the epic must be closed only after a settled green CI run. Finish with the exact
landing sequence requested for the epic: close `sase-a0`, run symvision after closure, remove expired epic-symbol
entries and newly exposed dead code, and set the linked plan's frontmatter status to `done`.

## Verified baseline

- `sase bead show sase-a0` resolves the plan as `plans:202607/fix_ci_failures.md`. All four phase beads are closed:
  `sase-a0.1`, `sase-a0.2`, `sase-a0.3`, and `sase-a0.4`.
- The phase implementations are present in source and history:
  - `55a2b0321` adds the documented 3.0x `notification_store_5k_mark_all_read` per-anchor variance factor.
  - `26ead3f39` adds read-only plain-checkout sidecar bead-store resolution, rejects mutations and `bead work`, prevents
    auto-commit, and covers those cases in regression tests.
  - `f90108a46` raises the published core floor to 0.11.3 after release `sase-core` `v0.11.3` and removes the
    then-expired `sase-9z` symvision whitelist/dead wrappers.
  - `921ca80f7` removes the two Python 3.14 tests' ambient dependencies: the statistics-pane soak waits for the observed
    tick count under runner starvation, and the deferred-fork test disables prettier explicitly and checks the
    whitespace-insensitive contract.
- The non-epic commits after the first `sase-a0` commit were audited. `a947469ee` only adjusts the visual-contention
  recipe/reporting, and its Justfile changes coexist with the lint fix. `f90108a46` contains the `sase-a0.2` floor work
  as part of the `sase-9z` land. The later `d1b02a69f` / `sase-core` `815e2e1` typed-close-resolution feature is the
  remaining integration defect.
- A clean Python 3.12 environment containing exactly `sase-core-rs==0.11.3` passes `tools/smoke_sase_core_rs_telemetry`,
  but `tools/check_sase_core_rs_bindings` reports:

  ```
  sase_core_rs 0.11.3 is missing 2 of 194 required binding(s):
    bead_needs_resolution_migration
    bead_resolution_migration_sql
  ```

- The incompatibility is not limited to missing names. The 0.11.3 wheel exposes
  `bead_close(beads_dir, issue_ids, reason=None, now=None)`, while current `src/sase/core/bead_mutation_facade.py`
  passes `(beads_dir, issue_ids, reason, resolution, now)`.
- The new bindings and five-argument `bead_close` implementation are present on `sase-core` `master`, but no release-plz
  PR is open yet and `Cargo.toml` remains at 0.11.3. Therefore SASE cannot safely raise its minimum until the next core
  patch is published.
- No CI run containing all epic commits has settled green. Pushes through `921ca80f7` were superseded, and the current
  `d1b02a69f` run began after the incompatible published-minimum change and is expected to fail that lane.

## Phase dependencies

| Phase                 | Depends on            | Why                                                                                          |
| --------------------- | --------------------- | -------------------------------------------------------------------------------------------- |
| `core-release`        | none                  | The compatible wheel must exist before SASE can declare it as a minimum.                     |
| `minimum-integration` | `core-release`        | The dependency floor, lockfile, and exact-minimum checks must target an installable release. |
| `verify-and-land`     | `minimum-integration` | The epic can close only after the integrated minimum and all original fixes pass together.   |

## Phase `core-release`: publish the typed-resolution core

Work in `sase-core`, opened through the repository skill, and preserve release-plz ownership of version fields.

1. Wait for or diagnose the `sase-core` CI/release-plz run for commit `815e2e1`. Confirm the release input includes both
   new migration bindings, the resolution field in bead wire/event records, and the new
   `bead_close(..., resolution=None, now=None)` signature.
2. Let release-plz create and refresh the next patch release PR (expected 0.11.4 because
   `features_always_increment_minor = false`). Do not hand-edit the release-plz-managed Cargo versions. If another
   release lands first, use the actual next version containing `815e2e1`.
3. Merge the release PR and wait for the guarded release workflow to tag and publish all wheels plus the sdist. Use the
   workflow's guarded manual `publish_pypi` / `expected_version` recovery only if automatic publishing fails.
4. In a fresh scratch Python 3.12 environment, install the exact published version from PyPI and verify:
   - `bead_needs_resolution_migration` and `bead_resolution_migration_sql` exist;
   - `inspect.signature(sase_core_rs.bead_close)` includes `resolution` between `reason` and `now`;
   - an explicit `done`, `canceled`, or `superseded` close round-trips through the binding, rather than merely checking
     that the function is present;
   - the existing telemetry smoke still passes.

Do not start the SASE floor bump until the exact released artifact is installable and these checks pass.

## Phase `minimum-integration`: raise and exercise the published minimum

After `core-release`, update the SASE repository to use the actual published core version containing typed close
resolutions.

1. Raise the inclusive `sase-core-rs` floor in `pyproject.toml`, update the asserted floor in
   `tests/test_sase_core_rs_telemetry_smoke_tool.py`, and refresh `uv.lock`. Keep the existing `<0.12.0` upper bound if
   release-plz produced the expected 0.11.x patch.
2. Add a focused published-wheel compatibility smoke for the typed-resolution bead path. It must run in the same
   exact-minimum environment as `published-core-minimum-smoke`, exercise an explicit non-default resolution through
   `bead_close`, and assert the returned/persisted resolution. Keep `tools/check_sase_core_rs_bindings` as the
   exhaustive static name gate; the new smoke covers the semantic/signature compatibility that name presence alone
   cannot prove. Add normal unit coverage for the smoke helper.
3. Run the CI-equivalent sequence in a fresh environment holding the exact declared minimum:
   - `tools/smoke_sase_core_rs_telemetry`;
   - `tools/check_sase_core_rs_bindings`;
   - the new typed-resolution bead smoke.
4. Run `just install` before repository checks, then `just check`. Also run the focused plain-checkout resolution tests,
   Phase 7 regression-floor tests, the two repaired Python 3.14 tests, and the typed-resolution tests so the original
   phases and the new integration are verified together.
5. Reproduce the CI plain-checkout layout with a checkout-local `.sase/sdd-store.json`, no workspace registry or marker,
   and a plans sidecar at `sase/repos/plans`. Confirm `sase bead show sase-a0` succeeds, exposes current resolution
   metadata, and a mutation remains refused.

Before handing off, re-audit commits added after `d1b02a69f`; integrate any new caller that should use the read-only
store resolution or the newly published typed-resolution API, and remove any duplication or conflict.

## Phase `verify-and-land`: settle CI, close, clean, and finalize

This is the final phase and must preserve the order below.

1. Wait for a full SASE `master` CI run containing `minimum-integration` to settle. Do not treat a concurrency
   cancellation as success. Confirm at minimum that `lint`, `published-core-minimum-smoke`, `phase7-perf-floor`, and
   `test (3.14)` all succeed. If a failure reproduces, fix it or create a tracked bead with evidence; do not rerun it
   until green without diagnosis.
2. Re-run `sase bead show sase-a0` and each child. Confirm all dependencies are closed, every phase description/note is
   satisfied by current source and the published artifacts, and no later commit invalidates the audit.
3. Close the epic with `sase bead close sase-a0` using the normal `done` resolution.
4. Only after closure, run `just symvision` if the recipe is available. Remove every expired `sase-a0` `--epic-symbol`
   entry and any unused code symvision exposes. Run `just check` again after any source or Justfile cleanup.
5. Open the plans sidecar through the repository skill and change only the linked canonical plan
   `plans:202607/fix_ci_failures.md` frontmatter from `status: wip` to `status: done`.
6. Verify the final bead shows `status: closed` with resolution `done`, the linked plan shows `status: done`, symvision
   has no expired epic whitelist or unused-symbol findings, and both repositories are clean except for the intended
   landing changes.

If closing the epic or editing its plan exposes an unexpected store or concurrency failure, stop before papering over
it: preserve the exact command output, repair the underlying issue, then repeat the final verification.
