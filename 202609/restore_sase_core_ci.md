---
tier: tale
title: Restore and verify sase-core GitHub Actions
goal: "Confirm the formatting defect behind the red sase-core workflows is corrected on
  master, make only any residual fix proven necessary by current checks, and leave the
  local and GitHub CI/release gates green.

  "
size: small
proposed_by: bbugyi200.athena.0hc
create_time: 2026-09-09 09:08:35
status: wip
---

# Plan: Restore and verify sase-core GitHub Actions

## Context and constraints

- Work in the `sase-core` repository opened through `sase repo open`; do not use or
  record an ephemeral numbered-workspace path.
- `actstat` identified the red master run for commit `7af2640` and the blocked
  Release-plz workflow. The primary CI job stopped at `./scripts/check.sh fmt-check`,
  before clippy or tests could run.
- The failed logs from the master run and both failed release-PR refreshes show the same
  `rustfmt` diff at `crates/sase_xprompt_lsp/src/server.rs`: the long async test
  function `model_alias_shortcut_text_edit_covers_every_trailing_whitespace_case` needed
  its parameter list split across lines.
- Current master already contains the exact formatting correction in commit `c2161c7`.
  Its replacement master CI run has passed both `cargo fmt + clippy + test` and
  `maturin build + import smoke`; the refreshed release branch has also passed the
  previously failing Rust gate. Preserve that correction and do not manufacture a
  redundant source change if the remaining workflows complete successfully.
- Follow `sase-core/AGENTS.md`: release-plz owns all versions, and the required local
  verification is the full `just check` suite rather than a crate-only test command.

## 1. Reconfirm the live failure and repository baseline

Run `actstat` and inspect the current GitHub Actions run/job conclusions for `master`
and the active release-plz branch. Confirm that every historical red job traces to the
single rustfmt failure described above, while current `master` contains `c2161c7` and
the working tree begins clean. If a current job is red for a different reason, inspect
its failed step and logs before changing code and narrow the implementation to that
newly demonstrated defect.

## 2. Retain or apply the minimal corrective change

Keep the rustfmt-produced signature now present in
`crates/sase_xprompt_lsp/src/server.rs`. If the implementation checkout no longer
contains it, run the repository formatter and retain only the formatter's change to that
signature. Do not alter behavior, tests, workflows, dependencies, Cargo versions, or
release metadata for the known failure.

If step 1 discovers an additional current failure, fix only its verified root cause,
preserving unrelated work and documenting why the extra edit is necessary. If no current
failure remains, make no source edit: the correct implementation is to verify the
already-landed fix rather than duplicate it.

## 3. Validate the complete local gate

From the `sase-core` repository root, run `just check` exactly as required by
`AGENTS.md`. Resolve any reproducible failure attributable to the current tree, then
rerun the full command until it exits successfully. Confirm afterward that the working
tree contains no generated or unrelated tracked changes; if a source correction was
required, review its focused diff and rerun `just check` once more on the final tree.

## 4. Revalidate GitHub Actions and release recovery

Use `actstat` plus the individual GitHub run/job conclusions to verify that the current
master CI run is successful, including both the Rust gate and the PyO3/maturin smoke
job. Then follow the refreshed release-plz CI and parent Release-plz workflow to
terminal success, confirming that the former `Wait for checks to pass` failure clears
once the release branch checks are green.

If an in-progress run reveals a new actionable repository failure, return to steps 1–3,
apply the smallest evidence-backed fix, and revalidate locally and remotely. Treat
cancelled/superseded historical runs as history, not as a reason to change working code.
Completion requires a clean `just check`, green current CI, and no Release-plz failure
caused by the original formatting defect.
