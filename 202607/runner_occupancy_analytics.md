---
tier: tale
title: Core runner-occupancy analytics
goal: The shared Rust statistics API reports exact, overlap-aware runner occupancy
  summaries, distributions, and bounded trends for any selected time and project scope
  while preserving compatibility and query performance.
bead: sase-8j.1
parent: sase/repos/plans/202607/runners_statistics.md
create_time: 2026-07-21 16:36:45
status: wip
---

- **PROMPT:** [202607/prompts/runner_occupancy_analytics.md](prompts/runner_occupancy_analytics.md)

# Plan: Core runner-occupancy analytics

## Context and scope

Implement phase `sase-8j.1` in the linked `sase-core` repository. The existing agent-run statistics query is
launch-window oriented; this phase extends that same read-only artifact-index query with historical runner-slot
occupancy so all frontends receive one authoritative concurrency calculation. The work remains in Rust, retains the
existing launch/run aggregates, and does not add UI or Python presentation models owned by later phases.

Runner eligibility will mirror user-agent slot admission: include `ace-run` root agents and explicitly parallel family
members, honor `workflow_state.appears_as_agent`, and exclude serial family bookkeeping, non-agent workflow steps,
hidden records, hooks, and Axe runners. Historical terminal records remain eligible even though the live admission
predicate only counts currently active records.

## Wire contract and compatibility

Extend the agent statistics wire module with an optional/defaulted nested runner record and bump the statistics schema
version. The record will expose the effective analysis bounds, peak and peak duration, time-weighted average, busy
duration/share, integrated runner-seconds, an ordered zero-through-peak occupancy distribution, bounded contiguous trend
slices, and explicit skipped data diagnostics. Each trend slice will carry exact start/end bounds, average, peak, busy
seconds, and runner-seconds.

Keep serde defaults so older or partial payloads can deserialize without the new runner field, and represent an all-time
query with no valid runner coverage as no runner record. A fixed empty range will still return a populated zero- runner
result. Export every new wire type through `agent_stats` and the crate root; the existing dictionary-based PyO3
conversion should remain thin and receive focused smoke assertions for the new nested shape and schema version.

## Shared active-interval semantics

Refactor the existing clan/family runtime helper to expose one internal active- interval derivation used by both clan
runtime aggregation and runner analytics. It will use `run_started_at`, choose the earliest valid `stopped_at` or
`finished_at`, cap live records at the exclusive query end, subtract plan and question human-wait windows, and reject
malformed, reversed, or zero-length segments without disturbing valid records. This keeps stop precedence, follow-up
metadata handling, and yielded-slot behavior consistent across runtime consumers.

Add a record-level runner eligibility helper near these shared semantics so the historical query and future core callers
have a single definition aligned with the current admission cases.

## Overlap query and aggregation

Refactor the run-statistics scan so launch aggregates still consume only rows launched inside `[start_ts, end_ts)`,
while runner aggregation also sees agents that began earlier and overlap the range. Use cached workflow, launch, finish,
hidden, and project columns to reject clearly irrelevant terminal rows before decoding `record_json`; fully decode open
records, boundary candidates, and rows whose cached fields cannot safely prove non-overlap. Preserve existing
malformed-row behavior for launch statistics and separately report malformed or impossible runner candidates.

Clip every derived active segment to the effective analysis window and perform a deterministic event sweep that combines
all transitions at the same timestamp before measuring duration. Retain idle segments, derive the dense
zero-through-peak distribution and summary values from the swept occupancy segments, and enforce the conservation
identities: distribution seconds equal the analysis span, weighted distribution seconds equal runner-seconds, and busy
plus idle seconds equal the span.

For all-time requests using the existing epoch sentinel, begin coverage at the earliest valid active runner segment and
retain subsequent idle gaps. For fixed ranges, preserve the requested bounds even when no runner appears. Build
contiguous trend slices from the same occupancy segments, using the caller's bucket granularity when it fits and
coarsening it as needed behind a strict backend bucket-count limit; include a partial final slice rather than extending
past the analysis end.

## Verification

Add Rust unit and integration coverage through the real scanner/index path for overlapping and disjoint agents,
simultaneous boundaries, carry-in and exact- boundary clipping, live records, earliest stop/finish selection, human-wait
exclusions, root/serial/parallel family eligibility, `appears_as_agent`, hidden records, project filters, malformed
JSON/timestamps and impossible intervals, an observed peak above ten, wholly idle fixed windows, all-time effective
coverage/no-data behavior, partial final buckets, and strict long-range trend bounds.

Assert distribution/share tolerances and all duration/integral conservation identities, plus deterministic bucket
summary values. Update the PyO3 binding round-trip test to verify the schema bump and runner dictionary without moving
domain logic into the binding. Run `cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`, then review the linked repository
diff for accidental version, generated, or unrelated changes. After verification, close only `sase-8j.1`; leave parent
epic `sase-8j` open and create no beads.
