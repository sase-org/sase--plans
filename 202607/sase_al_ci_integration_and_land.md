---
tier: tale
title: Finish sase-al CI integration and land the epic
goal:
  SASE master CI is fully green with a valid split-sidecar record, then epic sase-al is closed cleanly and its plan is
  marked done.
bead: sase-al
create_time: 2026-07-28 18:57:00
status: done
---

- **PROMPT:** [202607/prompts/sase_al_ci_integration_and_land.md](prompts/sase_al_ci_integration_and_land.md)
- **PARENT:**
  [202607/fix_ci_core_clippy_and_minimum.md](https://github.com/sase-org/sase--plans/blob/main/202607/fix_ci_core_clippy_and_minimum.md)
- **BEAD:** [sase-al](https://github.com/sase-org/sase--beads/blob/main/pages/sase-al/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-al.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-al.land.md#member-code)
  - [bbugyi200.athena.sase-al.land--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-al.land.md#member-plan)
- **COMMITS:**
  - [41a01b3](https://github.com/sase-org/sase/commit/41a01b397c79303acad241f2a44822193b3aeb32) — ci: emit valid split
    SDD store record
  - [887999f](https://github.com/sase-org/sase/commit/887999fb5d0c7acd0ca0a232e9a98f33d1fcc182) — fix(ci): stabilize
    full matrix isolation
  - [14b30c4](https://github.com/sase-org/sase/commit/14b30c411fe4ab371048b9b38d28dfad9bca3c06) — test: make task help
    aliases version-neutral

# Finish sase-al CI integration and land the epic

## Goal

Restore a fully green `sase` master CI run after the `sase-al.2` published-core minimum bump, then close epic bead
`sase-al`, run the required post-close Symvision cleanup, and mark its durable epic plan done.

## Context

- Epic `sase-al` and both children have been audited. `sase-al.1` and `sase-al.2` are closed with resolution `done`.
- The core work is present in `sase-core` commit `461c7f1b410c1c3a979ef7fbc21a64db30451a91`: the two `clone_on_copy`
  calls are gone and `close_issues_with_note` plus the PyO3 `py_bead_close` binding carry the targeted
  `too_many_arguments` allowances.
- `sase-core` release commit `a7a3121fa4cf2bf15efd7842b9e0997a907e1bd5` is tagged `v0.12.5`. Core master CI run
  `30402012749`, release-PR CI run `30402752940`, and release run `30403013933` all succeeded. PyPI reports
  `sase-core-rs 0.12.5` as latest with four wheels and one sdist.
- The exact published `0.12.5` wheel passes `tools/smoke_sase_core_rs_telemetry`, `tools/check_sase_core_rs_bindings`
  (all 202 current bindings), `tools/smoke_sase_core_rs_bead_resolution`, and `tools/smoke_sase_core_rs_plan_header`
  (schema 2).
- The SASE floor bump is master commit `ab6f07a68c63a7a8438942980ca20e133748dc90`; `pyproject.toml` requires
  `sase-core-rs>=0.12.5,<0.13.0`, `uv.lock` pins `0.12.5`, and the matching smoke-tool unit expectation is `0.12.5`.
- Master CI run `30405720692` proved the new `published-core-minimum-smoke`, shared core-wheel build, bead backend,
  install smoke, build, and several performance lanes green, but its lint job failed at `just validate`. The failure is
  not a core-wheel problem: `.github/workflows/ci.yml` writes a split SDD record with a `beads` sidecar and
  `"schema_version": 2`; `src/sase/sdd/_store_records.py` deliberately rejects beads sidecars below schema 3. The
  invalid record also makes plan-link validation fall back to nonexistent `.sase/sdd`.
- Git blame shows the beads mapping was added by intervening commit `4d55dabc17152d033c195fcebdf21df4e16b2170` while the
  schema-version line remained stale. This is the integration work the epic land audit uncovered.
- The durable epic plan is `plans:202607/fix_ci_core_clippy_and_minimum.md` in the `plans` sidecar; open that repository
  with the `/sase_repo` skill before reading or changing it.

## Phase 1: Correct and prove the CI sidecar record

1. In `.github/workflows/ci.yml`, change the generated split SDD record to `"schema_version": 3`, matching the existing
   three-way plans/research/beads payload and `SddStoreRecord` contract.
2. Strengthen the focused workflow regression test in `tests/test_justfile_lint.py` so
   `test_ci_lint_job_validates_split_sdd_sidecars` asserts that the embedded record uses schema 3 and includes the
   plans, research, and beads checkouts. Prefer a semantic YAML/record assertion if it stays small and clear; otherwise
   an exact focused text assertion is sufficient.
3. Run `just install` first, then focused tests for the changed workflow contract. Run `just check` as required for SASE
   source/test changes and resolve any attributable failures.
4. Commit the integration through the SASE commit workflow, wait for the resulting `sase` master CI run, and require the
   entire CI workflow—not only `published-core-minimum-smoke`—to finish successfully. In particular confirm lint's
   `SASE validation` and `Validate committed plans` steps are green. If CI reveals another attributable integration
   issue, fix, revalidate, and repeat rather than closing the epic against a red master.

## Phase 2: Land epic sase-al

Perform this phase only after Phase 1 has a fully green master CI run.

1. Re-read `sase bead show sase-al` and its two children to ensure no bead was reopened and all descendants remain
   closed with resolution `done`.
2. Close the epic without force:

   `sase bead close sase-al --note "<concise audit covering the verified core fixes/release, exact-wheel smoke, 0.12.5 floor, intervening schema-3 CI integration, and final green master CI run>"`

   If close is rejected, resolve or deliberately reopen the named unfinished work; do not force a `done` result.

3. After the close, run `just symvision`. If it reports expired `sase-al` whitelist entries or newly unused code, use
   the `/sase_memory_read` skill for `sase/memory/symvision.md` before editing, remove only the stale entries and
   genuinely unused code, and rerun Symvision plus proportionate tests and `just check`. Commit any required post-close
   source cleanup through the SASE commit workflow.
4. Open the `plans` sidecar through `/sase_repo`, set only the epic plan frontmatter field from `status: wip` to
   `status: done`, and verify the plan remains valid/readable. Confirm `sase bead show sase-al` reports `status: closed`
   and `resolution: done`.

## Completion criteria

- `sase-core` 0.12.5 release evidence and exact-wheel behavior remain verified.
- `sase` requires and locks `sase-core-rs 0.12.5`.
- The CI split-sidecar record uses schema 3 and has regression coverage.
- A post-fix `sase` master CI workflow is fully green.
- Epic `sase-al` is closed with resolution `done` without force.
- Post-close Symvision is clean.
- `plans:202607/fix_ci_core_clippy_and_minimum.md` has `status: done`.
