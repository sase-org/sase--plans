---
tier: tale
title: Make bead event identities and stream merges replay-stable
goal: Concurrent bead writers converge through multi-commit rebase replay without
  renumbering recorded events or rejecting prior semantic merge results.
bead: sase-9x.1
parent: sase/repos/plans/202607/bead_merge_replay_stability.md
create_time: 2026-07-27 06:42:12
status: wip
---

- **PROMPT:** [202607/prompts/replay_stable_bead_event_merge.md](prompts/replay_stable_bead_event_merge.md)

# Plan: Make bead event identities and stream merges replay-stable

## Goal

Complete phase `merge` of epic `sase-9x` so a semantic bead-stream merge never rewrites an event that was already
recorded, concurrent writers mint distinct stable identities, and every later commit in a multi-commit rebase accepts
the prior merge result as an order-preserving extension of its base.

## Context and decisions

Canonical event semantics belong in `sase-core`; Python remains a thin caller through the existing
`bead_merge_event_streams(base, ours, theirs)` binding. The binding signature does not need to change.

Existing event IDs remain valid and are never migrated. Newly materialized events will keep the readable
`<stream>:<creation-ordinal>:<operation>:<issue>` prefix and append a full SHA-256 digest over the immutable event
content (schema version, timestamp, actor, operation, issue ID, and payload). The ordinal remains a human hint, while
uniqueness no longer depends on stream position. Exact duplicate mutations produce the same identity and distinct
concurrent mutations at the same ordinal produce different identities without a schema-version bump.

Merge validation will treat the base as an ordered subsequence of each branch: additions may appear before or between
base events, but a missing base record or a record with the same identity and edited content remains a specific
validation error. After extracting records not already in the base, merge will deduplicate exact records and sort the
union by timestamp, operation priority, event ID, and serialized record as the final deterministic tie-breaker. The base
stays unchanged at the front and additions are appended verbatim, so stable IDs make repeated and regrouped merges
converge.

Because the Rust/Python function signature stays unchanged, no duplicated Python invariant logic or facade change is
appropriate. The host package's published `sase-core-rs` floor can only advance after release-plz publishes the new core
patch; this phase will keep that release-owned versioning boundary intact and verify the existing binding end to end
locally.

## Implementation

1. In `sase-core/crates/sase_core/src/bead/events.rs`, add one canonical event ID minting helper backed by the existing
   `sha2` and `hex` dependencies. Route both legacy-snapshot materialization and mutable-store appends through it while
   preserving all IDs read from existing streams.
2. Replace positional prefix validation with ordered-subsequence containment. Report the exact missing or rewritten
   base-event ordinal, and return the matched base records so additions can be identified across the whole branch rather
   than with `skip(base.events.len())`.
3. Simplify `merge_bead_event_streams` to form the deterministic union of non-base events, append the original records
   without renumbering, and remove the obsolete renumbering and merge-order machinery.
4. Extend Rust parity and mutation coverage for:
   - distinct stable IDs for concurrent same-ordinal events and deterministic IDs for exact duplicate content;
   - preservation of pre-existing legacy IDs;
   - additions interleaved around base records;
   - precise rejection of deleted and edited base records;
   - commutativity, associativity, and idempotence;
   - sequential replay of several local commit states against timestamp- interleaved upstream events, with validation at
     every step and every event present exactly once in the final stream.
5. Exercise the unchanged PyO3 merge binding as part of the workspace suite. Do not change release-plz-owned crate
   versions or advertise an unpublished Python dependency floor.

## Validation

From `sase-core`, run:

1. `cargo fmt --all -- --check`
2. `cargo test -p sase_core bead_event`
3. `cargo test --workspace`
4. `cargo clippy --workspace --all-targets -- -D warnings`

If this SASE checkout changes, first run `just install`, then run `just check` as required by its project instructions.
Finally inspect both worktrees, confirm only phase-scoped files changed, close `sase-9x.1`, and verify the parent
`sase-9x` remains open.
