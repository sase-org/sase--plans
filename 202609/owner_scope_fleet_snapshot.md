---
tier: tale
title: Owner-side presentable fleet snapshots and honest freshness
goal:
  The gateway serves a bounded, liveness-aware owner snapshot whose family metadata,
  freshness, and authoritative counts stay honest over time.
size: medium
proposed_by: bbugyi200.athena.sase-xe.16.11.7.14.1
bead: sase-xe.16.11.7.14.1
create_time: 2026-09-10 13:52:29
status: wip
---

- **PARENT:**
  [202609/fleet_stale_remote_rows.md](https://github.com/sase-org/sase--plans/blob/main/202609/fleet_stale_remote_rows.md)
- **BEAD:**
  [sase-xe.16.11.7.14.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.11.7.14.1.md)

# Owner-side presentable fleet snapshots and honest freshness

## Objective

Complete phase `sase-xe.16.11.7.14.1` in the `sase-core` repository by making the
gateway's authoritative fleet snapshot a bounded owner-side presentation rather than a
dump of every visible artifact-index row. Preserve every record whose liveness could
still be live, expose enough safe family metadata for the later viewer phase to fold
members, age snapshot and row freshness honestly, and ensure dead observations never
inflate running buckets or authoritative counts.

## Contract and policy

- Add a transport-independent fleet-presentation policy to `sase_core`. Its input must
  separate record facts from owner observations: artifact lifecycle/timestamps and
  family lineage come from the index, while the gateway supplies resolved liveness. Its
  deterministic decision must distinguish current presentation rows, recent terminal
  presentation rows, and excluded rows.
- Treat `Dead` and `NotProcess` active-tier records as terminal for presentation only.
  Preserve their recorded lifecycle/status so a viewer can render "was running" rather
  than fabricating a completion, but remove current-instance/action capabilities and
  place them in a non-running status bucket. Never demote or exclude an `Alive` or
  `Unknown` row, or a record protected by a waiting/question marker.
- Bound recent terminal presentation by both age and count. Use a seven-day horizon and
  a 200-row newest-first cap, matching the local listing's 200-row bounded tier and the
  local date model's definition of the recent week. Rank by trustworthy completion time
  (`done.finished_at`, then stopped/record timestamp fallback) with a stable identity
  tie-breaker. Apply one combined cap after actual terminal records and dead-active
  demotions are merged, so dead active leftovers cannot evade the window.
- Resolve dismissed-family ancestry in core from indexed `parent_timestamp` and family
  metadata plus the index's dismissed identities. A definitively dead member whose
  family root is dismissed must not escape as a standalone row. Do not hide a live or
  unknown member merely because its family lineage looks dismissed. Keep artifact files
  untouched; this phase is serving policy, not reconciliation or deletion.
- Extend the safe fleet summary surface with the normalized family role/parent linkage
  needed to distinguish roots, ordinary members, monitors, gates, procs, and historical
  shells. Continue to derive the logical family ID and row kind from canonical artifact
  metadata, keep path/PID fields out of serialized responses, and update the checked
  fleet contract fixture/schema and exports when the wire shape changes.

## Gateway integration

- In `crates/sase_gateway/src/fleet_reads.rs`, replace the flat 512-completion query and
  unconditional record projection with the core policy. Revalidate the artifact index,
  resolve owner liveness once per candidate, obtain dismissal-lineage facts through a
  bounded core index API, select the presentable set, then build details, content
  handles, summaries, and counts only from that set. Keep the work inside the existing
  blocking snapshot build and retain the four-second timeout.
- Use one build timestamp for every row's `observed_at_unix` and for the envelope's
  `refreshed_at_unix`. Store a monotonic build instant internally. On every cached read,
  derive both row and envelope `ObservationFreshnessWire` values through
  `classify_cache_freshness` instead of returning the original `Fresh` stamp. Use a
  short fresh interval and an aging interval, and require a rebuild once the stale
  threshold is reached (60 seconds); coalesce concurrent rebuilds under the existing
  refresh lock. A failed or timed-out rebuild must still return the previous snapshot
  marked stale/partial with the safe error code, preserving the established
  retain-previous behavior.
- Make the core projection's status bucket liveness-aware: lifecycle `Running`,
  `Starting`, or `Unknown` with definitive `Dead`/`NotProcess` liveness must bucket as
  stopped, never running/starting. Authoritative running counts must require compatible
  live owner facts rather than trusting a lifecycle label. Keep waiting and attention
  accounting intact and compute counts once over the complete presentable snapshot so
  catalog page size and terminal filtering cannot change them.

## Tests and verification

- Add focused `sase_core` unit tests for the presentation selector: live and unknown
  candidates survive; dead active rows demote into the shared recent window; records
  outside seven days and rows beyond the 200-item cap are excluded; stable ordering is
  deterministic; dead orphan members of a dismissed family are excluded while live or
  unknown members are preserved.
- Extend fleet projection/count tests to prove family role/kind/parent metadata is safe
  and accurate, a dead running/starting observation has a stopped bucket and contributes
  zero to `running`, and waiting/attention semantics remain unchanged.
- Extend gateway tests with realistic indexed fixtures proving dead active leftovers and
  dismissed-family orphans are absent, recent completions are jointly age/count bounded,
  counts are authoritative and page-independent, row/envelope freshness ages from the
  build instant, an aged cache triggers exactly one coalesced rebuild, and a failed
  rebuild retains a stale partial prior snapshot. Avoid wall-clock sleeps by exposing a
  test-only policy/clock seam or directly aging private cached state in the module test.
- Run targeted Rust tests while iterating, then run the repository's full `just check`
  gate (including the PyO3 binding tests). Before declaring completion, inspect the
  working diff for accidental version edits, run
  `sase bead epic-symbols sase-xe.16.11.7.14.1`, resolve or re-key every reported
  symbol, and close only `sase-xe.16.11.7.14.1` with a note naming the successful full
  check and the behavioral cases verified. Do not close any ancestor bead.
