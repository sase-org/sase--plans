---
tier: tale
title: Stabilize the notification snooze round-trip gate
goal:
  Make the trailing-zero snooze timestamp contract deterministic and stop pre-fix core
  records from blocking the live flake gate.
size: medium
proposed_by: bbugyi200.athena.sase-me
bead: sase-me
create_time: 2026-08-15 17:53:24
status: done
---

- **PROMPT:**
  [prompts/202608/stabilize_mark_snoozed_round_trip.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/stabilize_mark_snoozed_round_trip.md)
- **BEAD:**
  [sase-me](https://github.com/sase-org/sase--beads/blob/main/pages/sase-me/README.md)

# Plan: Stabilize the notification snooze round-trip gate

## Context

`tests/notification_store/test_mute_snooze.py::TestMarkSnoozed::test_round_trip` failed
in three independent full-run records on 2026-08-11, 2026-08-12, and 2026-08-15. The
first two SASE revisions required `sase-core-rs` 0.24.x and the third required 0.27.5;
both core lines normalized snooze deadlines with Chrono's `to_rfc3339()`, which shortens
fractional seconds such as Python's `.296000` to `.296`. The Python assertion compared
exact `datetime.isoformat()` strings, so the test failed whenever the sampled wall-clock
microsecond ended in trailing zeros. Core commit `1ecbc8c` replaced that formatter with
Python-compatible precision preservation and added a `.296000` parity regression. The
same SASE head that failed at 2026-08-15T17:12:28Z passed after that core fix at
2026-08-15T17:38:38Z, and the current SASE dependency floor is 0.27.7, which contains
the fix.

The selection-health gate still treats the pre-fix records as a live flake because its
committed `effective-after` timestamp predates all three failures. The remedy should
make the sensitive Python boundary test deterministic and advance the evidence window
only past the exact backend fix, without adding a known-fixed node to the flake
baseline.

## Implementation

1. Change the mark-snoozed round-trip test to use an explicit future deadline whose
   microsecond field has trailing zeros, and name/document the test around preservation
   of the six-digit Python ISO representation. Keep assertions for both the mute state
   and exact UTC deadline so the boundary regression becomes deterministic instead of
   clock-sampled.
2. Advance `tests/reproducible_flake_baseline.txt`'s `effective-after` marker to the UTC
   timestamp of core fix `1ecbc8c` (2026-08-15T17:22:27Z). Do not add the notification
   node to the baseline. This excludes only records created against the known-broken
   formatter and leaves post-fix evidence eligible.
3. Verify the revised node repeatedly and run the notification-store test module. Run
   the selection-health gate with explanation/JSON as needed to confirm the old node is
   no longer a live above-baseline flake and that eligible post-fix records remain,
   making the cutoff non-vacuous.
4. Run the repository-required verification, escalating to monitored `just check-full`
   if the baseline change is treated as broadening or scoped selection escalates. Close
   `sase-me` only after the focused contract, live flake gate, and required repository
   checks all pass, recording those facts in the close note.

## Acceptance criteria

- The snooze round-trip test deterministically exercises a trailing-zero microsecond
  timestamp and preserves the exact six-digit UTC ISO string.
- The committed flake baseline starts after core fix `1ecbc8c`, contains no new entry
  for this fixed node, and still evaluates eligible post-fix full-run records.
- Focused notification-store tests, the flake-baseline gate, and the required repository
  verification pass.
- No unrelated bead or baseline debt is changed; genuinely distinct discoveries are
  routed through `/sase_new_task` with `sase-me` identified as the source.
