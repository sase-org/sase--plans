---
tier: tale
title: Restore sase-core GitHub Actions after the atomic close-note change
goal:
  The sase-core workspace passes its formatting, warnings-as-errors Clippy, and full
  test gates without changing the new close-note behavior or public API.
create_time: 2026-09-09 19:53:13
status: wip
---

# Restore sase-core GitHub Actions after the atomic close-note change

## Problem and evidence

GitHub Actions run `30399429666` for `sase-org/sase-core` commit `e098a1a`
(`feat(beads): support atomic close notes`) fails only in the
`cargo fmt + clippy + test` job, at the
`cargo clippy --workspace --all-targets -- -D warnings` step. `actstat` shows that the
preceding commits passed and that the concurrent Release-plz workflow for `e098a1a`
succeeded.

The failing Clippy command reproduces locally with the repository's pinned stable Rust
1.97 toolchain and reports exactly three diagnostics introduced by the close-note
commit:

- `close_issues_with_note` in `crates/sase_core/src/bead/mutation.rs` has eight
  arguments, exceeding Clippy's default seven-argument threshold.
- A new CLI test in `crates/sase_core/src/bead/cli.rs` calls `.clone()` on
  `BeadEventOperationWire`, although that enum implements `Copy`.
- A new mutation test in `crates/sase_core/src/bead/mutation.rs` repeats the same
  unnecessary `.clone()`.

The first diagnostic is an intentional consequence of extending the existing flat
close-operation compatibility API with optional note and author fields. This repository
already uses narrowly scoped `#[allow(clippy::too_many_arguments)]` annotations for
deliberately flat public/binding-style APIs. Replacing the API with a request object
would expand this CI repair into an unnecessary public API redesign and require
coordinated caller changes. The two `clone_on_copy` diagnostics have direct,
behavior-free fixes.

## Implementation

1. In `crates/sase_core/src/bead/mutation.rs`, add a narrowly scoped
   `#[allow(clippy::too_many_arguments)]` annotation to `close_issues_with_note`, with a
   short comment documenting that the flat signature intentionally preserves the
   close-operation compatibility and binding boundary. Do not alter the existing API,
   mutation ordering, note attribution, or atomic save behavior.
2. In the new close-note assertions in `crates/sase_core/src/bead/cli.rs` and
   `crates/sase_core/src/bead/mutation.rs`, read `event.operation` directly instead of
   cloning the `Copy` enum.
3. Keep the change limited to these Clippy regressions. Do not edit Cargo versions or
   dependency version pins; release-plz owns them.

## Validation

Run the same checks and ordering used by the failing GitHub Actions job:

1. `cargo fmt --all -- --check`
2. `cargo clippy --workspace --all-targets -- -D warnings`
3. `cargo test --workspace`

If any command fails, diagnose and correct the implementation, then rerun the full
sequence from formatting through tests. Confirm the final diff contains only the scoped
lint repair and no release-owned version changes.

After the fix is integrated and GitHub Actions has run, use
`actstat --repo sase-org/sase-core` to verify that the latest `sase-core` commit settles
successfully.
