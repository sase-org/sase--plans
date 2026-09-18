---
tier: tale
title: Cut bounded Tier 1 loader decode amplification
goal:
  Restore bounded Agents-list load cost by filtering non-projectable index records
  before JSON hydration while preserving loader parity.
size: medium
proposed_by: bbugyi200.athena.sase-132.3
bead: sase-132.3
status: done
---

- **PARENT:**
  [202609/tui_startup_regression.md](https://github.com/sase-org/sase--plans/blob/main/202609/tui_startup_regression.md)
- **BEAD:**
  [sase-132.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-132/sase-132.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-132.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-132.3.md)
- **COMMITS:**
  - [13a8efb](https://github.com/sase-org/sase/commit/13a8efbb4ad4adc7a1694238b27a2613cf78553f)
    — feat(tui): project Agents-list index rows before JSON hydration

# Cut bounded Tier 1 loader decode amplification

## Objective

Complete phase `sase-132.3` by making the shared Rust artifact-index query return only
records that can contribute an Agents-list row while preserving Python loader parity,
then prove that the production bounded path decodes at most three records per returned
row on a production-shaped synthetic archive and returns to the phase's real-archive
latency target when host conditions permit.

The baseline capture at 11,851 artifacts measured `production_bounded` at 1,759.6 ms and
decoded 1,447 record JSON payloads to return 100 rows. The bounded Rust query limits
completed candidates, but currently decodes every SQL-active candidate first. In the
live index, hundreds of old `ace-run` records are waiting/question marker-only records:
they may provide enrichment/context for a separately loaded ProjectSpec RUNNING claim,
but `load_done_agents_from_snapshot`, `load_running_home_agents_from_snapshot`, and the
workflow loaders cannot materialize them as base rows. They must not consume the JSON
decode window.

## Implementation

1. Extend the shared artifact-index query contract in `sase-core` with an explicit,
   default-off Agents-list projection mode. Mirror the field through the PyO3 wire and
   the Python `AgentArtifactIndexQueryWire` serializer. Keep generic index callers
   unchanged; enable the mode only in the TUI tiered loader query so shared backend
   behavior remains authoritative and other index consumers retain their existing record
   sets.

2. In the Rust cached window/full-history candidate selection, apply a scalar/index-side
   predicate before `record_json` hydration. A candidate is loader-projectable when it
   can produce one of the Python loader's artifact-backed base rows: a supported done
   record with a done marker, a home `ace-run` record with a running marker, or a
   workflow-state record accepted by the workflow loader. Preserve query candidate
   filters, active/completed ordering, dismissal visibility, family-relative expansion,
   truncation metadata, and the existing list record shape.

3. Preserve context-only behavior without restoring JSON amplification. Marker-only
   records associated with ProjectSpec RUNNING claims can carry clan/tribe/summary
   context even though they are not base rows, so derive any represented clan keys and
   clan context needed by the projected result from indexed scalar columns. Add parity
   coverage for a running claim whose waiting-only artifact supplies clan context, as
   well as done, home-running, workflow, family-relative, dismissed, hidden, and noop
   cases. The optimized bounded and full-history production paths must match the
   authoritative source path (`missing_count=0` and no visible extras where the tier is
   complete).

4. Make the regression measurable. Expand the synthetic tiering fixture with a
   production-shaped population of marker-only active-looking records, expose a derived
   decoded-records-per-returned-row value in the benchmark report, and add a
   deterministic assertion that `production_bounded` stays at or below 3.0x
   amplification while still returning the requested window. Keep wall-clock latency as
   reported evidence rather than a flaky unit-test assertion.

5. Verify both repositories and the phase acceptance surface:
   - run focused Rust core/index and PyO3 binding tests, then `just check` in
     `sase-core`;
   - run the Python wire, loader-window/parity, and tiering benchmark smoke tests, then
     the SASE repository's required check workflow from `sase/memory/lint_and_test.md`;
   - run the synthetic archive benchmark and record its amplification, parity, and p50;
   - on a quiet athena host if available, run the documented real-archive
     `bench_agent_load_tiering.py --sase-home ~/.sase` recipe, record deployed SHAs and
     host state, and verify `production_bounded` p50 <= 1.1 s at >=11.5k artifacts. If
     host load prevents a valid quiet capture, retain the deterministic amplification
     and parity proof and note the unavailable environmental acceptance evidence rather
     than weakening the target.

6. Before closing only `sase-132.3`, run `sase bead epic-symbols sase-132.3` and resolve
   every remaining symbol or re-key it to an open later phase/parent. Record any
   out-of-scope retention/vacuum finding as a `PROPOSED FOLLOW-UP:` note on this phase,
   then close it with a note naming the checks, benchmark results, parity result, and
   real-host evidence actually verified. Do not close `sase-132` or another phase.

## Boundaries

- Do not move shared filtering or decoding behavior into Python; the Rust core owns the
  query semantics and the Python layer only opts into and consumes the wire contract.
- Do not redesign artifact-index retention/vacuum policy in this phase.
- Do not conflate startup-window contention or the separately observed multi-second
  dismissed-bundle snapshot read with the standalone `production_bounded` acceptance
  metric; preserve those findings for the startup-sequence phase/epic land triage.
- Do not change historical startup telemetry field meanings or weaken full-history
  reconciliation semantics.

## Acceptance

- The production bounded synthetic path decodes no more than 3.0 records per returned
  row on the marker-heavy fixture and returns the requested viewport.
- Full-history and bounded parity checks retain their documented zero-missing semantics.
- The generic artifact-index query remains backward compatible because the new
  projection mode defaults off.
- A valid quiet-host athena capture reaches `production_bounded` p50 <= 1.1 s at
  > =11.5k artifacts, or the phase note explicitly records why only deterministic and
  > busy-host evidence was obtainable without changing the target.
