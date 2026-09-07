---
tier: tale
title: Fix sase-core Clippy CI failures
goal: Restore sase-core master CI and unblock Release-plz without changing fleet-attention
  behavior or weakening lint enforcement.
size: small
proposed_by: bbugyi200.athena.053
status: done
---

# Plan: Fix sase-core Clippy CI failures

## Goal

Restore the `sase-org/sase-core` master CI and unblock Release-plz by fixing the two
`clippy::cloned_ref_to_slice_refs` errors without changing fleet-attention behavior or
weakening lint enforcement.

## Diagnosis

`actstat --repo sase-org/sase-core -n 5 --no-active` shows that commits `b19c603` and
`eacd178` both fail the `cargo fmt + clippy + test` CI job at its Clippy step. The
Release-plz failures are downstream: its merge job waits for the generated release PR's
checks, whose matching Clippy job fails for the same reason.

The failed logs report Rust Clippy 1.98's `cloned_ref_to_slice_refs` lint, promoted to
an error by `-D warnings`, at the two single-element input slices in
`crates/sase_core/src/fleet_attention.rs`'s
`notice_dedupe_suppresses_reconnect_and_announces_new_revision` test. Both expressions
were introduced by `b19c603`; `eacd178` did not touch this file and simply inherited the
already-red master state. `decide_attention_notices` only borrows its input slice, so
cloning `entry` to construct either temporary slice is unnecessary.

## Implementation

1. In `crates/sase_core/src/fleet_attention.rs`, replace both singleton
   `&[entry.clone()]` arguments in
   `notice_dedupe_suppresses_reconnect_and_announces_new_revision` with
   `std::slice::from_ref(&entry)`. Keep the later `entry.clone()` used to create the
   independently mutated `superseded` revision; that clone is semantically necessary.
2. Do not add a lint suppression or alter the GitHub Actions/Release-plz workflows: the
   existing `-D warnings` policy correctly identified the avoidable clones, and the
   release failure will clear when its underlying CI check clears.

## Validation

1. Run `just check` from the `sase-core` repository root, as required by its
   `AGENTS.md`. This executes formatting checks, workspace/all-targets Clippy with
   warnings denied, and the full workspace test suite including the Python bindings.
2. Inspect the final diff and repository status to confirm the change is limited to the
   two test inputs and introduces no version edits or unrelated files.

## Acceptance criteria

- Workspace Clippy no longer reports `cloned_ref_to_slice_refs` in `fleet_attention.rs`.
- The notice-deduplication test retains its original three behaviors: first delivery
  announces, an identical reconnect is suppressed, and a superseding revision announces.
- `just check` exits successfully for the full `sase-core` workspace.
- The diff contains no lint allowances, workflow changes, or release/version edits.
