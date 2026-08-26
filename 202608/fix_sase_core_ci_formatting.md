---
tier: tale
title: Restore sase-core CI formatting compliance
goal:
  Apply the canonical Rust formatter to the unformatted bead-link changes, verify the
  complete workspace gate, and allow both CI and release automation to recover without
  changing release-owned versions.
size: small
proposed_by: bbugyi200.athena.0e0
---

- **AGENTS:**
  - [bbugyi200.athena.0e0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0e0.md)
- **COMMITS:**
  - [f232814](https://github.com/sase-org/sase-core/commit/f232814e4cd974fddf8891193a9b112b15890ade)
    — fix: restore sase-core check gate

# Context

`actstat --repo sase-org/sase-core -n 4 --only-failures` reports the same primary
failure on master commits `4b1f2d6` and `2664e20`: the `cargo fmt + clippy + test` job
stops at `cargo fmt --all -- --check`. The `maturin build + import smoke` job passes.
The two Release-plz failures are consequential: their `Merge release PR` job finds
release PR #182, then exits while watching that PR's failed formatter check.

The failure reproduces locally with `cargo fmt --all -- --check`. Rustfmt requests only
layout changes in these files introduced by the bead-link commit `4b1f2d6`:

- `crates/sase_core/src/bead/read.rs`
- `crates/sase_core/src/bead/search.rs`
- `crates/sase_core/src/lib.rs`
- `crates/sase_core/tests/bead_read_parity.rs`

The later plan-validation commit `2664e20` does not introduce another formatter diff; it
inherited the already-red master tree.

# Implementation

1. Reconfirm that the linked `sase-core` working tree contains no unrelated user
   changes. Preserve any unexpected changes and stop rather than overwriting them.
2. Run the repository's canonical formatter through `just fmt` (which delegates to
   `./scripts/check.sh fmt` and `cargo fmt --all`).
3. Inspect the resulting diff. Keep only rustfmt's mechanical layout changes in the four
   reproduced files; do not alter behavior, workflow definitions, Cargo versions, or
   release-plz's generated release branch.
4. Run `just check`, the repository-required full local gate. It must pass formatting,
   clippy for the whole workspace and all targets with warnings denied, and workspace
   tests including the PyO3 bindings. Diagnose and correct any failure caused by this
   repair, then rerun the complete gate until it succeeds.
5. Recheck the final diff and working-tree status so the handoff contains only the
   intended formatting repair. A subsequent push to master will trigger CI and
   Release-plz; the release workflow owns refreshing/reconciling release PR #182, so no
   manual version edit or release-branch mutation is appropriate.

# Acceptance criteria

- `cargo fmt --all -- --check` is clean as part of `just check`.
- Workspace-wide clippy and tests pass through `just check`.
- The source diff is formatting-only and limited to the four files reproduced above.
- No release-owned Cargo version, GitHub Actions workflow, or release PR branch is
  edited manually.
- The root CI failure is removed, allowing the Release-plz wait job to recover when its
  release PR checks rerun against the corrected master history.
