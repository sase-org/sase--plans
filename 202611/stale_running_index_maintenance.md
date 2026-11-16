---
tier: tale
title: Terminalize stale running index rows with verified lifecycle evidence
goal:
  Drain witnessed stale running rows through fair bounded Rust maintenance while
  preserving live and uncertain agents and the Agents live predicate.
size: medium
proposed_by: bbugyi200.athena.sase-1t3.12
proposed_with: codex/gpt-6-astra
bead: sase-1t3.12
create_time: 2026-11-16 08:20:53
status: wip
---

- **PARENT:**
  [202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/tui_storm_trust_heap_recovery.md)
- **BEAD:**
  [sase-1t3.12](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.12.md)

# Terminalize stale running index rows with verified lifecycle evidence

Complete phase **sase-1t3.12**, `stale-index-rows`, of
`plan:202611/tui_storm_trust_heap_recovery.md`. Keep the Agents live predicate
unchanged. Repair the materialized index in bounded, coalesced background maintenance,
preserving genuinely live and uncertain rows.

## Scope and sizing

This is one bounded coding task: extend the existing Rust terminalization pass, carry
its diagnostics through its existing binding, and verify the Python maintenance
integration. No earlier worker notes or partial implementation were recorded on the
phase. The existing maintenance API, off-thread scheduler, wire types, and
terminalization tests supply the implementation framework; a child epic is unnecessary.

All shared decisions belong in linked `sase-core`; Python remains a thin caller. Use
`/sase_repo` to open it and follow its `AGENTS.md`. Preserve the parent epic's final
decisions: `hotfix_reinstall=yes`, `trust_ack_compat=cutover`, and `tui_perf_memory=no`.
The first two belong to other phases. Edit no memory files, change no trust or hijack
behavior, and do not reinstall globally or restart the user's TUI. Do not introduce an
archive deletion, a new daemon, extra worker threads, or a full-archive scan on startup,
query, render, or keystroke paths.

## Verified starting point

The planner inspected sase `e27b302028` and sase-core `50a6d972`; both checkouts were
clean. Recheck the relevant history before coding so concurrent phase work is retained.

An athena read-only projection on 2026-11-16, through
`load_agent_artifact_index_projection`, returned schema 37 and:

| Measure                                |         Rows |
| -------------------------------------- | -----------: |
| Total index                            |       23,293 |
| `status='running'` without done marker |       12,398 |
| `has_running_marker=1`                 |            0 |
| Current terminalization SQL candidates |       13,012 |
| Running / starting candidates          | 12,302 / 710 |

The candidate timestamps ranged from `20260706063245` to `20261116075923`. These are
observations, not a classification of every candidate as dead. The parent plan reports
an earlier baseline of 12,474 running/no-done rows out of 23,133. Its referenced
consolidated research artifact was unavailable through the audited read during planning;
the counts above were independently remeasured.

Relevant implementation facts:

- `src/sase/axe/run_agent_runner_setup_meta.py::write_home_running_marker` writes
  `running.json` only for home-mode runs. Runner metadata also records PID and
  `process_identity`. Workspace runs use RUNNING claims.
- Core `agent_scan/index/record_summary.rs::RecordSummary::from_record` derives
  `running` from `run_started_at` / `wait_completed_at` when no terminal/waiting state
  intervenes. It does not require `running.json`.
- `agent_scan/index/refresh.rs::select_terminalization_candidates` selects marker-free
  rows oldest-first with a fixed LIMIT and no progress cursor. Skipped rows can occupy
  the same prefix on every pass.
- `record_has_live_workspace_claim` treats any same workspace number, or matching
  workflow/change-list pair, as protection, without checking claim PID liveness or
  distinguishing a later occupant. The terminalization pass never checks the candidate's
  own PID/process identity.
- `agent_scan/index/maintenance.rs::terminalize_stale_active_agent_artifact_index_rows`
  already synthesizes hidden `abandoned` terminal records in SQLite, performs retention,
  and checkpoints the WAL. It does not write source done markers.
- Python `agent_artifact_index_lifecycle.py` supplies a 24-hour grace and a 10,000-row
  limit. Startup maintenance already runs off-thread; the coalescing scheduler in
  `ace/tui/actions/agents/_index_maintenance.py` rate-limits later terminalization to
  hourly passes. Preserve these ownership boundaries.

The overbroad claim protection and repeated prefix are demonstrated code defects;
measure their respective contribution to the live backlog before claiming either
explains all 12,398 rows.

## Implementation

### 1. Classify candidates from fresh witnesses in Rust

Factor the terminalization policy and witness collection out of `refresh.rs` into a
focused index module if needed. Use injectable filesystem/process observations for
deterministic tests. Keep the public maintenance entry point and its existing defaults;
extend its result with defaulted, additive diagnostics when needed.

For each bounded candidate:

1. Revalidate source marker signatures. A valid on-disk done marker repairs the
   projection using its actual outcome, visibility, and finish time. Never replace it
   with synthetic `abandoned`. Missing, unreadable, and malformed evidence are distinct
   outcomes. A freshly created marker must win over the stale SQL snapshot.
2. Preserve waiting/question states, held claims, and nonterminal workflow states unless
   an authoritative terminal marker resolves them. This phase must not invent new
   waiting, gate, monitor, or workflow lifecycle rules. Include stale running-marker
   rows in inspection so a leftover marker is not an eternal veto.
3. For an abandoned candidate, require the existing age grace, positive evidence that
   its recorded local process has ended or changed identity, and no matching live or
   uncertain ownership claim. Age, missing `running.json`, missing claim, or a `running`
   status alone is insufficient. Missing or invalid PIDs remain unknown unless another
   authoritative terminal witness exists.
4. Compare recorded process identity with the current process as well as PID existence:
   distinguish missing/zombie, matching live, reused PID/previous boot, and unknown.
   Match the persisted `boot_id:start_ticks` convention from
   `src/sase/core/process_identity.py`; use existing Rust host-liveness primitives where
   their semantics fit. Do not treat permission errors, malformed procfs, unsupported
   identity observation, or foreign-machine PIDs as proof of death. On non-procfs
   platforms, use positive absence evidence where available and otherwise preserve the
   row. Do not change fleet-wide liveness policy to obtain a maintenance-specific
   decision.
5. Match claims to the run using project and artifact timestamp plus available PID
   identity. A known different timestamp/occupant cannot protect an old run merely
   because its workspace number or change list matches. Legacy claims without sufficient
   identity, held/non-process claims, and unreadable claim state remain conservative. A
   matching live claim protects the row even during runner handoffs. Observe claims once
   per project per batch, with fresh confirmation of mutations that depend on absence or
   mismatch.

Gather expensive evidence outside write transactions. Immediately before applying each
repair, check that relevant source signatures, claim evidence, and stored row still
agree; skip races rather than overwriting a newer lifecycle update. Apply accepted
changes in a short transaction. Preserve existing hidden-abandoned finish time behavior
and archive contents. Repeated maintenance and ordinary revalidation must converge; a
full rebuild may recreate the source projection, after which the same witnessed repair
must converge again.

### 2. Make bounded passes advance past skipped rows

Replace the repeatedly selected prefix with a persisted keyset cursor over
`(timestamp, artifact_dir)` in index metadata, scoped to the applicable project root.
Advance after every examined row, including protected, fresh, corrupt, and uncertain
rows. Wrap after the end of the range; reset safely after rebuild, root change, or
malformed cursor metadata. Rows inserted before the cursor must be visited on the next
cycle, and deletions must not break continuation.

Keep `max_rows` a bound on rows examined, not only rows mutated. Handle zero and
unbounded limits explicitly. Avoid OFFSET pagination and whole-archive record
materialization. Persist cursor progress with successful batch updates so an interrupted
pass is safely retryable. Preserve index lock/busy handling, retention, and best-effort
WAL checkpoint behavior. One bad row must not permanently block later eligible rows.

Include enough per-pass diagnostics to distinguish examined rows, source-done repairs,
synthesized terminal rows, and skipped-live/claim/fresh/unknown/race reasons, along with
whether the bounded traversal wrapped. Keep existing update fields backward compatible.
Carry any added fields through Rust wire tests, `sase_core_py/src/agent_scan`, and
Python wire conversion/reporting; do not merely count all repaired rows as new abandoned
rows.

### 3. Keep the existing background integration and prove the result

Route the improved pass through the existing lifecycle facade and coalescing scheduler.
Preserve hourly scheduling, first-paint independence, navigation gates, and
last-request-wins. Emit compact batch diagnostics through the existing maintenance
logging/trace path, off the event loop. No per-row telemetry, no hot query repair
expansion, and no new public CLI are needed.

Before changing the production index, run the corrected pass on a consistent SQLite
backup in workspace scratch storage, reading the real sources without mutating them.
Compare the old and corrected candidate classifications on the same snapshot. After
focused tests pass, run bounded maintenance against the normal index using the validated
local core build, through the existing facade; do not hand-edit SQLite. Record counts
and skipped reasons before and after a full cursor cycle, and compare equivalent
revalidate queries using elapsed time and existing scan statistics
(candidate/repair/signature work). Use `/sase_monitor` before starting a run that
exceeds the foreground budget.

The goal is a running set close to verified live runs plus explicitly explained
fresh/held/unknown rows. Report remaining categories honestly; do not weaken evidence
checks to hit a count. Retain a small fixture benchmark with a skipped prefix larger
than the batch and eligible rows beyond it so the regression is reproducible without the
live archive.

## Verification and completion

Add focused tests beside the existing core terminalization suite for:

- Done marker appearing after indexing; real outcome/visibility preserved.
- Old live metadata-only process without `running.json`; matching live/held claim.
- Dead process, reused PID identity, previous boot, and stale running marker.
- Same workspace reused by a different run, exact matching run, legacy ambiguous claim,
  and source/claim/process probe uncertainty.
- Waiting/question/workflow, remote ownership, and fresh-run preservation.
- Marker, row, or claim changing between selection and write.
- More skipped candidates than `max_rows`, progress across multiple calls/reopen,
  wraparound, insert/delete during traversal, zero limit, corrupt rows/cursor,
  idempotence, and query/rebuild convergence.

Use injected process/procfs fixtures, not arbitrary real PIDs. Test binding round trips
for any new result fields and retain tests in `tests/agent_artifact_index_lifecycle`,
`tests/ace/tui/test_artifact_index_maintenance_scheduler.py`, and
`tests/ace/tui/test_apply_path_index_maintenance_offthread.py` that prove coalescing,
cadence, and off-thread execution. Include a live-predicate regression showing that a
legitimate workspace agent with no running marker remains visible.

Run targeted core tests with `just test -p sase_core <filter>` and binding tests with
the repository's supported recipes; never bare cargo. Then run `sase tool run check` in
both changed repositories. Read `lint_and_test.md` and use `/sase_monitor` for long
verification; do not run `just check-full`. Build/test the actual edited extension, not
an unrelated installed wheel. Declare both changed repos through `/sase_final`; the host
commits core first and advances `sase-core-revision.txt` to that commit before the sase
commit. Preserve a newer concurrent pin rather than moving it backward.

Record investigation, before/after measurements, unresolved conservative cases, and
verification evidence with `sase bead note sase-1t3.12`. Record out-of-scope work only
as `PROPOSED FOLLOW-UP:` notes on this phase; never create task beads. An identical
clean-base check failure needs a cited witness and a follow-up note, not an indefinitely
open phase.

When the scope is implemented and verified, run `sase bead epic-symbols sase-1t3.12`,
resolve any entries or re-key them to an appropriate open bead, and close **only**
`sase-1t3.12` with a note stating what was verified. The planner found no epic-symbol
entries, but recheck at completion. Do not change its status by hand or close `sase-1t3`
or any ancestor.
