---
tier: tale
title: Complete Fleet performance and failure-table hardening
goal:
  Fleet navigation stays responsive under remote faults and its remaining reconciliation
  and fencing contracts are covered end to end.
size: medium
proposed_by: bbugyi200.athena.sase-xe.16.9
bead: sase-xe.16.9
create_time: 2026-09-08 11:13:56
status: wip
---

- **PARENT:** [202609/remote_dispatch_completion.md](remote_dispatch_completion.md)
- **BEAD:**
  [sase-xe.16.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.9.md)

# Plan: Complete Fleet performance and failure-table hardening

## Context

Phase `sase-xe.16.9` closes the remaining acceptance-hardening gap for remote Fleet
refreshes. The reusable offline Fleet substrate exists, but the `j`/`k` benchmark suite
does not exercise a hung host, reconnect churn, or an event burst; the performance
runbook has no capture recipe for those scenarios; two failure-table boundaries lack
explicit regression coverage; and reconciliation normalizes the follow store without
deriving explicit-follow singleton-to-family promotions from authoritative followed
batch results.

The binding contracts remain unchanged: remote work stays off Textual's event loop and
serial message pump, navigation p95 is strictly below 16 ms, a timed-out host cannot
erase a healthy host's result, and lifecycle mutations carry the selected row's exact
instance locator and revision rather than relying on a reused name or PID.

## Implementation

1. Extend the offline Fleet test substrate with the smallest deterministic helpers
   needed to synthesize multiple hosts and scripted deadline/reconnect/event-burst
   behavior without network access or a real federation worker.
2. Add Fleet fault scenarios to the historical Agents `j`/`k` benchmark entrypoint. Warm
   the rendered Fleet list, drive navigation while each fault is active, print
   per-scenario p50/p95/max summaries, assert every measured `j`/`k` p95 is below 16 ms,
   and reject recorded TUI stalls.
3. Add the two missing failure-table regressions at their production boundaries: verify
   a facade read observes its deadline while retaining an independent healthy host
   result, and project a remote row into a lifecycle mutation whose old exact instance
   is rejected after the same logical agent/name is replaced by a new run.
4. Derive safe family promotions after followed-batch hydration for active explicit
   singleton follows only. Match the authoritative family locator to the same origin,
   project, and agent identity; pass those promotions through the existing follow-store
   reconciliation API; then use the reconciled snapshot for Focus/Fleet projection.
   Cover successful promotion plus ambiguous, already-family, dispatch-created, and
   tombstone-preserving cases without adding work to the empty-store path.
5. Document an exact offline capture command and interpretation checklist in the TUI
   performance runbook, including the `< 16 ms` gate and the three fault scenarios.

## Verification

- Run focused follow-store, Fleet refresh, federation facade, remote mutation, and Fleet
  model tests.
- Run the slow Fleet `j`/`k` fault benchmark with perf logging enabled and confirm all
  three scenarios satisfy p95 < 16 ms with no stall rows.
- Run `just check` after the workspace is installed for this ephemeral clone.
- Immediately before closing, run `sase bead epic-symbols sase-xe.16.9`; resolve or
  re-key any remaining symbols, then close only `sase-xe.16.9` with the verified
  evidence.
