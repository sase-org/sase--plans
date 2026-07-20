---
tier: tale
title: Atomic bead launch claim
goal: Add a concurrency-safe single-bead launch claim in the Rust core and expose
  it consistently through the Python binding and bead APIs.
create_time: 2026-07-20 15:53:26
status: wip
prompt: 202607/prompts/atomic_bead_launch_claim.md
---

# Plan: Atomic bead launch claim

## Context and scope

The just-in-time launch design needs one reusable domain operation that claims exactly one bead for the agent that is
about to execute. The operation must work for both phase beads and the parent epic used by a future land agent, accept
an open bead or reassign an existing non-closed in-progress bead, and reject missing, closed, or invalid requests before
model execution can begin. This tale implements that storage and API foundation only. Parsing `%id(bead=...)`, invoking
the claim from the runner, and removing eager epic-work preclaims belong to later beads and will not change here.

The existing batch `preclaim_epic_work` API remains exported and behaviorally compatible for current callers until the
epic migration phase removes it from the launch path.

## Rust-core mutation and locking

Add `claim_for_agent_launch` to the bead mutation domain with a bead-store path, bead ID, resolved agent name, and an
optional deterministic timestamp. Validate that the agent name is non-blank and that the target exists and is not
closed. Both open and already-in-progress targets succeed; every success sets `status=in_progress`, replaces the
assignee with the supplied agent, updates the timestamp, and returns the updated issue in the standard mutation outcome.
Use distinct, actionable error kinds/messages for missing targets, closed targets, invalid agent names, lock acquisition
failures, and durable read/write failures.

Introduce a bounded exclusive bead-mutation lock using the core's existing `fs2` dependency and timeout/retry style.
Acquire it before loading canonical events or the compatibility projection, retain it through validation, event append,
event-store/config/projection persistence, and release it after the whole read-modify-write transaction completes. This
prevents sibling workers claiming beads in the same store—and especially the same epic event stream—from loading the
same snapshot and overwriting each other's events or projection updates. Keep lock cleanup exception-safe, and surface
timeout or unlock failures rather than silently continuing.

Represent the successful claim as the existing `issue_updated` event with both status and assignee fields. This keeps
old event readers compatible and makes event reduction reproduce the claimed issue without introducing a launch-only
event variant. Preserve the batch preclaim event contract for its legacy API.

Export the new operation and any required wire type through the bead module and the top-level `sase_core` crate API.

## PyO3 binding and Python APIs

Expose the core operation as a `bead_claim_for_agent_launch` PyO3 function that releases the GIL around the bounded
filesystem transaction and serializes the standard mutation outcome. Follow the existing bead error conversion so typed
core errors retain their kind and actionable message at the Python boundary.

Add `claim_for_agent_launch` to the Python bead mutation facade, returning the converted `Issue` together with the raw
outcome. Add the matching `BeadProject` adapter method, supply the project's UTC timestamp, refresh the legacy SQLite
compatibility state after success, and leave it untouched on failure. Maintain the public facade export list and the
existing Python exception mapping for missing beads while allowing the new validation/closed/lock errors to remain
specific `ValueError` failures for runner code to contextualize later.

## Verification

Add Rust mutation tests covering an open target, an already-in-progress target reassigned to a new agent, deterministic
timestamps, normal update-event reduction, regenerated JSONL projection, missing target, closed target, and empty or
blank agent names. Add a same-store concurrency regression that starts two claims against sibling phase beads and then
proves both claims, both update events, and both projected assignees survive; include a bounded-lock failure assertion
so contention cannot hang indefinitely.

Add Python binding/facade parity tests for the outcome and `Issue` conversion, open and reassignment behavior, missing
and closed failures, and `BeadProject` delegation/compatibility refresh. Confirm the legacy batch preclaim tests still
pass unchanged.

Validate the linked core with targeted bead tests, then `cargo fmt --all -- --check`, `cargo test --workspace`, and
`cargo clippy --workspace --all-targets -- -D warnings`. In the main repository, run `just install` so the local PyO3
extension is rebuilt from the linked core, run the focused mutation facade and project-delegation tests, and finish with
the required `just check`.
