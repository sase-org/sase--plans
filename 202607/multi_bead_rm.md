---
tier: epic
title: Remove multiple beads atomically
goal: '`sase bead rm` accepts one or more bead IDs, removes the requested beads and
  their descendant cascades in one atomic operation, and behaves consistently through
  the Rust fast path and Python compatibility path.

  '
phases:
- id: core-batch-remove
  title: Atomic batch removal in sase-core
  depends_on: []
  size: medium
  description: '''Atomic batch removal in sase-core'' section: add the shared batch
    mutation, event, binding, and fast-path contracts with deterministic all-or-nothing
    behavior.'
- id: cli-contract
  title: Python CLI contract, documentation, and end-to-end coverage
  depends_on:
  - core-batch-remove
  size: medium
  description: '''Python CLI contract, documentation, and end-to-end coverage'' section:
    expose one-or-more IDs in argparse, route the slow path through the batch API,
    align commit/output/docs contracts, and run the full integrated checks.'
create_time: 2026-07-24 14:32:58
status: done
bead_id: sase-8x
---

- **PROMPT:** [prompts/202607/multi_bead_rm.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/multi_bead_rm.md)
- **BEAD:** [sase-8x](https://github.com/sase-org/sase--beads/blob/main/pages/sase-8x/README.md)

# Plan: Remove multiple beads atomically

## Context and intended behavior

`sase bead close` already accepts `ids [ids ...]`, but `sase bead rm` currently accepts exactly one `id`. Removal is
implemented in both the Rust fast path and the Python compatibility path; the underlying Rust mutation currently loads
and saves the store once per single target. Changing only argparse or looping over the existing single-item mutation
would therefore either miss the fast path or permit a later invalid ID to fail after earlier beads have already been
irreversibly deleted.

Make the public contract:

```text
sase bead rm <id> [<id2> ...]
```

The command must retain today's single-ID behavior. For multiple IDs it must validate every requested ID against the
same pre-mutation store snapshot, compute the union of every requested bead and each plan bead's recursive descendants,
remove that union once, clean dependencies that reference any removed bead, append durable removal events, and save
once. If any requested ID is missing, nothing is removed and no removal event is appended.

Return and print each actually removed bead once. Preserve the existing children-first order inside a selected plan's
cascade and process independent requested roots in argument order. Overlapping selections (for example, a plan and one
of its descendants) and duplicate arguments must not duplicate output or mutation records. The mutation summary and
automatic commit message should record the requested IDs in CLI argument order, while the mutation outcome's
removed-issue list describes the unique expanded deletion set.

Do not add confirmation flags or change removal's existing irreversible semantics. Do not broaden `sase bead open`,
`show`, or `update` to multiple IDs. Do not edit release-plz-owned crate versions manually.

## Atomic batch removal in sase-core

Work in the linked `sase-core` repository because batch mutation and event-store semantics are shared backend behavior.

1. In `crates/sase_core/src/bead/mutation.rs`, introduce a batch removal operation that accepts a non-empty slice of
   requested IDs, loads `MutableStore` once, and validates all requested IDs before changing the store or its event
   streams. Keep the existing single-bead Rust API as a thin compatibility wrapper if external callers use it.
2. Expand each unique request using the existing recursive, children-first descendant traversal, deduplicate the
   combined removal set deterministically, remove all matching issues, and strip dependencies whose source or target is
   in that set. Append a removal event for each unique requested root using one batch timestamp and enough cascade
   metadata for event replay to reconstruct the same final store. Handle requested roots that overlap without duplicate
   returned issues or reducer failures, then persist the event store, configuration, and compatibility projection once.
3. Preserve the outcome distinction between requested roots and expanded removals: the domain outcome returns the unique
   removed issues/IDs in deterministic display order, while callers that own the CLI retain the original requested-root
   order for summaries.
4. In `crates/sase_core_py/src/lib.rs`, expose a list-taking PyO3 binding for the batch operation while retaining the
   existing single-ID binding for compatibility. Export the new binding from the module without changing
   release-plz-managed version fields.
5. In `crates/sase_core/src/bead/cli.rs`, let the Rust fast path handle one or more `rm` arguments through the batch
   API. Render every unique removed issue once, report missing IDs using the existing CLI error style, and put the
   requested IDs (not the expanded descendants) into `BeadCliMutationSummaryWire`.
6. Add focused Rust tests covering:
   - one target, preserving the current child-first cascade and output;
   - two independent requested beads;
   - a plan plus an explicitly requested descendant, in both argument orders;
   - duplicate requested IDs;
   - a missing later ID proving the store, projection, dependencies, and event streams remain unchanged;
   - event reduction/reload after a successful batch;
   - fast-path stdout, exit status, and requested-ID mutation summaries.
7. Run `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and
   `cargo test --workspace` in `sase-core` (or the equivalent top-level `just rust-check` integration after installing
   the host environment).

## Python CLI contract, documentation, and end-to-end coverage

This phase depends on `core-batch-remove` so the product shell can require and exercise the new binding rather than
reimplementing shared batch behavior.

1. In `src/sase/core/bead_mutation_facade.py` and `src/sase/bead/project.py`, add list-taking batch-removal adapters
   that call the new Rust binding and refresh the compatibility database once. Preserve the existing single-ID
   `remove()` API as a thin wrapper for internal callers such as plan-launch rollback.
2. In `src/sase/main/parser_bead.py`, change the `rm` positional to `ids` with `nargs="+"`, plural help text, and help
   output consistent with `sase bead close`. In `src/sase/bead/cli_crud.py`, invoke the batch adapter, print each unique
   removed issue returned by the core, retain the existing missing-issue error format, and create one automatic commit
   message that joins all requested IDs in argument order.
3. In `src/sase/main/bead_fast_path.py`, make `rm` automatic commit messages join every requested ID from the Rust
   mutation summary, matching the slow path exactly.
4. Ensure the host's `sase-core-rs` dependency floor is advanced to the first published core version that contains the
   new batch binding before this CLI change is released; update lock/metadata artifacts through the repository's normal
   dependency workflow. Do not guess or manually edit the Rust crate release version. Confirm
   `tools/check_sase_core_rs_bindings` succeeds against the declared published minimum.
5. Update `docs/beads.md` and the bead command table in `docs/configuration.md` to document `<id> [<id2> ...]`,
   recursive union deletion, uniqueness for overlapping selections, and all-or-nothing validation. Keep the irreversible
   warning explicit.
6. Extend Python coverage at the relevant boundaries:
   - parser/help assertions for required one-or-more `ids`;
   - facade/project tests for multi-root removal, overlap deduplication, and missing-ID atomicity;
   - CLI golden coverage showing multiple requested removals while retaining the existing single-target golden contract;
   - slow-path automatic commit coverage for all requested IDs;
   - fast-path side-effect coverage proving multiple IDs reach Rust and produce the same commit message as the slow
     path;
   - Rust/Python mutation parity coverage where applicable.
7. From the SASE repository, run `just install` first so the local PyO3 extension is rebuilt from the linked
   `sase-core`, then run targeted bead, parser, facade, and fast-path tests. Finish with `just rust-check` and the
   required `just check`. Re-run `sase bead rm --help` and a temporary-store smoke exercise for one ID, multiple
   independent IDs, an overlapping parent/descendant request, and a request containing a missing ID.

## Acceptance criteria

- `sase bead rm a` remains compatible with current behavior.
- `sase bead rm a b ...` works through both execution paths, removes the unique union of requested beads and recursive
  descendants, and prints each removed bead once in deterministic order.
- A missing requested ID causes a non-zero exit without any partial removal, event append, dependency rewrite,
  compatibility-projection change, or automatic commit.
- Rust mutation outcomes, fast-path summaries, slow-path commit messages, CLI help, manuals, and tests agree on
  requested IDs versus expanded descendants.
- The published SASE dependency range cannot install a `sase-core-rs` version that lacks the binding required by the new
  CLI.
- All Rust checks and the SASE repository's required `just check` pass.
