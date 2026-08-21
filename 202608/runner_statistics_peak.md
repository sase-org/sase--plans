---
tier: tale
title: Correct historical runner occupancy peaks
goal:
  Make Admin Center runner statistics reflect only real runner-slot occupancy and remain
  invariant across equivalent query windows.
size: medium
proposed_by: bbugyi200.athena.093
create_time: 2026-08-21 08:28:28
status: wip
---

# Plan: Correct historical runner occupancy peaks

## Diagnosis

The Statistics pane is faithfully rendering schema-v6 data from
`agent_stats_query_runs`; the configured limit is independently loaded only as today's
reference value. The invalid peak is created in the Rust historical occupancy builder,
not in the TUI.

The regression was introduced when monitor handoffs were added to historical family
occupancy. A durable `monitor_id` is copied onto both the actual monitor family member
and agents that start or resume through a monitor, but the Rust occupancy code currently
uses `monitor_id` alone as its monitor predicate. It consequently applies monitor-only
start and gap-fill behavior to ordinary root/code agents. It also interprets the
timezone-free artifact-directory timestamp as UTC and chooses it ahead of the explicit
UTC `run_started_at`. On this host, normal monitor records move roughly four hours
earlier, while pre-created plan/follow-up artifacts move by even more.

The production index proves the failure is query-dependent: the same interval reports a
peak of 31 inside the seven-day query and only 8 when queried directly. Reconstructing
the current algorithm from the indexed records reproduces 31 exactly. The canonical
monitor model already documents the violated contract: `monitor_id` is shared and a real
monitor member is identified by `agent_family_role == "monitor"` together with a
non-empty monitor id.

## Implementation

1. Open the linked `sase-core` repository through `sase repo open` and centralize the
   durable real-monitor predicate near the shared agent-runtime behavior. Use the same
   explicit role-plus-id contract already used by monitor indexing; do not infer a
   monitor merely from an inherited `monitor_id`.
2. Apply that predicate consistently to live Rust runner-slot occupancy and historical
   family contributions. Only a real monitor member may use the PID-only started rule or
   trigger monitor handoff gap filling. Ordinary roots, planners, coders, and follow-ups
   with an inherited monitor id must use ordinary agent semantics.
3. Stop manufacturing a historical start by parsing the timezone-free artifact name as
   UTC. Use the authoritative offset-aware `run_started_at` for recorded intervals, and
   preserve continuous monitor handoff occupancy by merging the true monitor member with
   its preceding serial family interval. A transient live monitor that has a PID but has
   not yet recorded `run_started_at` may count in the live snapshot, but must not create
   a fabricated historical interval from an ambiguous wall-clock stamp.
4. Add focused Rust regressions in the agent-runtime and agent-statistics suites:
   distinguish a true monitor member from a root that inherited `monitor_id`; retain
   one-slot continuity across a real starter/monitor/follow-up handoff; prove a
   pre-created or local-time artifact stamp cannot move a later agent backward; and
   assert that a broad query and a nested query agree on occupancy over their shared
   interval. Include an integration fixture shaped like the reported case, with several
   later monitor-associated agents that previously inflated an earlier peak.
5. In the primary `sase` repository, align the Python live occupancy predicate and its
   documentation with the same real-monitor contract so admission/display snapshots do
   not drift from Rust. Update runner-slot unit tests and the Rust/Python historical
   parity fixtures to carry explicit monitor roles, plus a negative case for an ordinary
   agent with an inherited monitor id.
6. Rebuild/install the linked Rust binding through the repository's supported
   development workflow, then run its focused runtime/statistics tests and full required
   checks. Run `just install` before verification in the primary repository, exercise
   the focused runner-slot, binding-smoke, statistics-view, TUI runner, and parity
   tests, and finish with the required `just check` (escalating to the
   repository-prescribed full check through `/sase_monitor` if selection broadens).
7. Re-query the real local index for the seven-day range and a nested interval. Confirm
   the fabricated 31-runner peak disappears, the overlapping-window results agree, and
   the pane still shows today's configured limit separately from the corrected
   historical peak. Do not clamp statistics to 10: a genuine historical peak may exceed
   today's setting if the past limit/temporary override differed, and the UI already
   labels that distinction.

## Acceptance criteria

- An inherited `monitor_id` never changes an ordinary agent's start time or family-gap
  semantics.
- A true monitor handoff remains one continuously occupied family slot without counting
  overlapping serial members twice.
- Historical occupancy for an interval is independent of whether that interval is
  queried alone or as part of a wider range.
- The reported production data no longer yields the fabricated peak of 31, and the
  Python live counter agrees with the Rust historical definition on equivalent fixture
  snapshots.
- Focused tests and each repository's required verification gates pass without changing
  unrelated statistics presentation or clamping observed data to current configuration.
