---
tier: tale
title: Keep live family roots in ACE runner-slot accounting
goal:
  ACE reports the same occupied runner slots that admission enforces when family phases
  share one runner PID.
create_time: 2026-09-09 19:53:22
status: wip
---

# Plan: Keep live family roots in ACE runner-slot accounting

## Outcome

Make the ACE Agents tab report the same runner-slot occupancy that admission enforces
when one runner process has produced multiple family-phase artifacts. A queued agent
must show `10/10`, not `9/10`, when a live family root still owns the tenth slot and a
completed serial follow-up shares its PID.

## Confirmed root cause

The scheduler is working correctly in the reported incident. At the screenshot time,
admission saw ten live slot holders, so the implicit waiter with threshold nine remained
parked. It started at `11:14:09`, two seconds after
`toobig-i.split_file.tests.ace.test_update_receipt.d06e4eac` completed at `11:14:07` and
released a real slot.

ACE understated occupancy because its loaded row set had discarded holder `hc.f0.f0--0`
(PID 1729466). The runner's workspace claim produced a live `RUNNING` root row for
artifact `20260721155935`; a completed serial family follow-up produced a `WORKFLOW` row
for artifact `20260721160507` with the same PID and a parent link to the root. In
`src/sase/ace/tui/models/_dedup.py`, `dedup_by_pid()` applies `WORKFLOW > RUNNING`
before recognizing that the two rows have distinct artifact suffixes. It removes the
root and merges `runner_is_live` into the terminal child. Later,
`refresh_runner_slot_context()` correctly excludes that serial child from runner-slot
participation, leaving no row that represents the live root's slot. The scheduler and
`sase agent list -j`, which count from admission records, both retain the root and
report the true occupancy.

## Implementation

1. Correct the final PID safety-net deduplication in
   `src/sase/ace/tui/models/_dedup.py`.
   - Treat two non-VCS rows with present, different artifact suffixes as distinct
     artifacts before applying the generic `WORKFLOW > RUNNING` preference. This
     generalizes the function's existing same-type exceptions for distinct WORKFLOW
     phases and distinct RUNNING runs to the missing cross-type case.
   - Continue collapsing RUNNING/WORKFLOW projections of the same artifact when suffixes
     match, and retain the legacy fallback when either suffix is absent.
   - Keep the higher-priority VCS workspace-claim deduplication unchanged so
     provider/workspace duplicates cannot leak back into the Agents tab.
   - Update the function documentation and small helper structure as needed so the
     governing rule is artifact identity, with PID serving only as a safety-net
     candidate key.

2. Add focused loader/dedup regression coverage in
   `tests/test_agent_loader_dedup_pid.py`.
   - Reproduce the production shape: a live RUNNING family root and a completed serial
     WORKFLOW child share a PID, have different artifact suffixes, and are linked
     through `parent_timestamp`/family metadata.
   - Assert normalization preserves the root as well as the legitimate child/family
     projection, rather than transferring the root's liveness onto the child and
     deleting the root.
   - Preserve explicit cases proving that same-suffix RUNNING/WORKFLOW duplicates and
     VCS duplicates still collapse, and that missing-suffix legacy rows retain the
     established fallback behavior.

3. Lock the cross-layer runner-capacity invariant in the ACE test suite.
   - Feed the normalized production-shaped family rows through
     `refresh_runner_slot_context()` and assert exactly one occupied slot: the root
     counts, while its serial child remains excluded.
   - Include an implicit waiter and assert its attached `runner_slots_in_use`,
     eligibility/queue context, and returned `RunnerCapacitySnapshot` reflect the true
     occupied count.
   - Where practical, compare the fixture's expected occupancy to the shared admission
     semantics so future loader or family-projection changes cannot silently recreate a
     scheduler/TUI disagreement.

## Performance and architecture constraints

- Keep the correction inside the existing off-thread load/normalization pass. Do not add
  filesystem scans, process probes, JSON reads, or recomputation to Textual render, key,
  timer, or message-pump paths.
- Continue deriving runner context in O(loaded rows) through the existing refresh fast
  path; preserving one legitimate row must be the only runtime cost.
- Do not change admission thresholds, FIFO/priority policy, serial-child exemption,
  question yielding, or the global cap. The observed queue delay was correct; only ACE's
  projection was wrong.
- Do not modify historical artifacts or workspace claims as part of the fix.

## Validation

1. Run the focused PID-dedup and runner-slot context tests, including the new
   production-shaped regression.
2. Exercise a loader fixture through normalization/family attachment and verify the live
   root survives, the serial child remains available for family history, and the
   capacity snapshot counts one holder rather than zero.
3. Run `just install`, then the required full `just check` gate. Inspect any visual
   snapshot output if the preserved row changes a fixture, but no intentional golden
   update is expected for this data-correction-only change.
4. Review the final diff to confirm it contains no admission changes, new UI-thread
   work, memory/generated-instruction edits, historical artifact mutations, or unrelated
   formatting.

## Acceptance criteria

- In the reported live-family shape, ACE and `sase agent list -j` agree with
  `running_root_agent_count()` about the number of occupied slots.
- The Agents header and a waiting agent's detail/row context show the full cap while the
  family root is alive; when a holder exits, the count falls and the eligible waiter
  starts under the existing two-second poll behavior.
- Legitimate serial family history remains rendered without double-counting the child as
  a slot holder.
- True duplicate projections remain deduplicated and all focused plus full checks pass.
