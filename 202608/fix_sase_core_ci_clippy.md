---
tier: tale
title: Repair sase-core Clippy CI failure
goal:
  Restore sase-core CI and unblock Release-plz without changing the public Python API.
size: small
proposed_by: bbugyi200.athena.0bg
create_time: 2026-08-23 11:44:42
status: wip
---

# Repair sase-core Clippy CI failure

## Diagnosis

`actstat` reports that `sase-org/sase-core` commit `b39dfbf` fails its CI workflow in
the `cargo fmt + clippy + test` job, specifically the
`cargo clippy --workspace --all-targets -- -D warnings` step. The failed log identifies
`crates/sase_core_py/src/lib.rs::py_sanitized_proc_env` as an eight-argument function,
one over Clippy's default threshold. The binding was introduced by `92a4fc4`; the later
`b39dfbf` commit did not touch it and inherited the same failure.

The Release-plz `Merge release PR` failure is secondary: its `Wait for checks to pass`
step observed the same Clippy check failing on release PR 166. Existing PyO3 bindings in
this crate intentionally retain their public Python call signatures and place a narrow
`#[allow(clippy::too_many_arguments)]` on wrappers that exceed the Rust lint's
threshold, with comments explaining why bundling parameters would distort the exported
Python API.

## Implementation

1. Open the linked `sase-core` repository through `/sase_repo` and confirm the checkout
   is still based on the failing `master` revision before editing.
2. In `crates/sase_core_py/src/lib.rs`, add an explanatory comment and a function-scoped
   `#[allow(clippy::too_many_arguments)]` to `py_sanitized_proc_env`. Preserve its seven
   public Python parameters, defaults, PyO3 signature, forwarding behavior, and return
   shape; do not suppress the lint at crate or workspace scope.
3. Review the diff to ensure the change is limited to the binding's lint rationale and
   attribute, with no release-version edits or unrelated formatting churn.

## Verification

1. Run `just check` from the `sase-core` repository root, as required by its
   `AGENTS.md`. This exercises the same formatting, workspace-wide Clippy, and complete
   workspace test gates as CI, including the `sase_core_py` binding tests.
2. If any gate fails, diagnose and repair only failures caused by this change, then
   rerun `just check` until it passes. Report unrelated pre-existing failures without
   broadening this repair.
3. Reinspect `git diff` and `git status` after verification and report the exact CI root
   cause, the secondary Release-plz consequence, the changed file, and the successful
   verification result.
