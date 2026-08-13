---
tier: tale
title: Release family waits after handled monitor failures
goal:
  A monitor failure handed to a successful continuation no longer leaves the
  agent-family waiter stuck.
size: medium
proposed_by: bbugyi200.athena.002
create_time: 2026-08-13 17:50:28
status: wip
---

# Release family waits after a monitor hands off to a successful continuation

## Goal

Make a wait on an agent-family lane complete when a terminal monitor member has
successfully launched its recorded `--next` continuation and that continuation later
completes successfully, without weakening the fail-closed behavior for monitor failures
that have no valid continuation. Then reconcile the existing `sase-l1.land` waiter so
the completed epic can proceed.

## Root cause and evidence

`sase-l1.land` has both agent waits and bead waits for phases `sase-l1.1` through
`sase-l1.6`. All six beads are closed, but `dependency_resolution_status()` reports
`blocked_on=('sase-l1.6',)`.

Phase 6 became the sequential family rooted at `sase-l1.6--plan` when it started monitor
`sase-l1.6--mon` (`9yeer0htvj79`). The monitor ran `just test`, reached its intentional
six-minute timeout with real pytest output, and persisted `monitor_state: timeout`,
`monitor_followup_outcome: launched`, and `monitor_followup_agent: sase-l1.6--1`. That
continuation completed successfully and closed bead `sase-l1.6`.

The wait index nevertheless computes the family as `is_resolved=False` and
`is_failed=True`: `_family_entity()` requires every historical family member's raw wait
outcome to resolve, and `effective_done_outcome()` correctly classifies the timed-out
monitor itself as failed. The aggregation does not distinguish an unhandled monitor
failure from a terminal monitor that explicitly transferred responsibility to a real,
later family continuation. Consequently, the earlier `--mon` failure permanently poisons
the lane even after `--1` succeeds.

## Implementation

1. Represent a monitor's durable follow-up handoff in the wait-dependency artifact
   model. Derive it only from terminal monitor metadata/markers whose follow-up outcome
   is a successful launch (`launched` or `launched-degraded`) and whose recorded
   follow-up agent name is non-empty. Keep the monitor artifact's own effective outcome
   failed for direct named-monitor waits and diagnostics.

2. Teach sequential-family aggregation to treat that failed monitor member as a
   delegated handoff only when the recorded successor artifact actually exists in the
   same family generation. The successor remains part of the aggregate, so an absent,
   queued, waiting, running, failed, or killed continuation keeps the family blocked;
   only a successfully completed continuation releases it. Exclude a successfully
   delegated monitor from the family's effective failure/terminal-blocker set so
   identity waits and blocker logging agree with name-based family waits. Do not change
   standalone monitor or clan semantics, and preserve the existing rule that ordinary
   failed/killed family members remain blockers even if another sibling succeeds.

3. Add focused resolver and `wait_checks` regressions reproducing the real topology: a
   completed `--plan` root, a timed-out `--mon` child with a persisted launched
   follow-up, and a `--1` continuation attached to the same root. Cover successful and
   degraded launches followed by success; missing and still-running successors; failed
   successors; unsuccessful monitors without a launched follow-up; direct waits on the
   monitor member; family identity waits; and terminal-blocker reporting. Assert that
   only the handled-success case writes the waiter's `ready.json`.

4. Run `just install`, the focused wait-dependency test modules, and `just check`. If
   scoped selection escalates or reports an unusual selection, run `just check-full`
   through `/sase_monitor` with a continuation, as required by the repository workflow.

5. After the implementation passes, use the workspace's fixed SASE executable to run one
   forced `wait_checks` reconciliation against the live artifacts. Verify that the
   existing `sase-l1.land` gets `ready.json`, leaves `WAITING`, and starts (or reaches a
   later terminal state). Do not hand-edit its markers or remove either its agent waits
   or bead waits.

## Acceptance criteria

- A failed/timed-out monitor with no successfully launched continuation remains a
  terminal blocker, including for direct waits on the monitor member.
- A family remains blocked while a recorded monitor follow-up is absent or nonterminal,
  and remains blocked if that follow-up fails.
- Once the recorded continuation completes successfully, the monitor handoff no longer
  poisons the family; name and identity waits resolve consistently and diagnostics do
  not report the delegated monitor as the active blocker.
- Ordinary failed family members retain their existing fail-closed behavior.
- The focused tests and `just check` pass.
- The durable `sase-l1.land` waiter is reconciled through normal `wait_checks` behavior
  and is no longer stuck on `sase-l1.6`.

<!-- sase:referenced-by:start -->

## Referenced By

| Agent                        | Project | Reference                                    | Published  | Uses |
| ---------------------------- | ------- | -------------------------------------------- | ---------- | ---: |
| [bbugyi200.athena.002--1][1] | sase    | plan:202608/monitor_followup_wait_release.md | 2026-08-13 |    1 |

[1]: https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.002.md

<!-- sase:referenced-by:end -->
