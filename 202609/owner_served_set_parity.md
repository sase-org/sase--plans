---
tier: tale
title: Owner served-set parity
goal:
  The gateway serves exactly the agent rows the owner presents, excluding dead
  protected, dismissed, zombie, and recycled-PID records.
size: medium
proposed_by: bbugyi200.athena.sase-133.1
bead: sase-133.1
create_time: 2026-09-18 17:07:34
status: wip
---

- **PARENT:**
  [202609/remote_dispatch_agents_tab_parity.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_dispatch_agents_tab_parity.md)
- **BEAD:**
  [sase-133.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/sase-133.1.md)

# Plan: Owner served-set parity

Complete phase bead `sase-133.1` so the gateway presentation catalog contains the same
owner-visible agent records as the owner's local Agents tab. Keep this phase limited to
served-set selection and owner-side observations; the later epic phases own wire
presentation fields and viewer rendering.

## Established baseline

- `crates/sase_core/src/fleet_presentation.rs` already owns the pure, fact-driven
  `decide_fleet_presentation()` policy and its unit tests.
- `crates/sase_gateway/src/fleet_reads.rs` queries the agent artifact index, resolves
  owner observations, calls that policy, and builds catalog/history snapshots only from
  selected identities.
- The current policy deliberately keeps every protected waiting/question row current
  before considering dead liveness, and it considers dismissal only after protected and
  live/unknown cases. Those precedence rules are the direct parity defects for this
  phase.
- Gateway liveness still uses bare `kill(pid, 0)`. A stronger Rust implementation
  already exists in the runner-stats path: it rejects zombies and non-SASE/Python
  command lines and, where applicable, verifies the record against its home marker or
  project workspace claim. Consolidate or reuse that logic instead of creating a second
  subtly different process classifier.
- The dismissal-lineage index API already resolves exact active dismissals and dismissed
  family roots; its `seed_definitively_dead` compatibility path also handles killed
  workflow records dismissed under their run identity. Preserve those semantics and pass
  the result into the pure policy for every candidate.

## Implementation

1. Open the linked `sase-core` repository with the repository-access workflow and
   inspect the current tree before editing, because other epic phases may have touched
   `fleet_reads.rs` or adjacent contract code. Preserve unrelated and phase-2 changes.

2. Make owner process observation robust and testable.
   - Extract or expose one shared Rust host-liveness helper from the existing
     runner-stats implementation so both runner accounting and gateway snapshot
     construction use the same non-zombie, SASE/Python-process checks. Preserve portable
     behavior when `/proc` is unavailable.
   - Retain the gateway's `OwnerLivenessWire` distinction: completed records are `Dead`,
     missing process evidence is `Unknown`, invalid PIDs are `NotProcess`, and a
     positive PID is `Alive` only after the strong probe succeeds.
   - Use record identity evidence where available: project records must still match the
     current workspace claim (PID plus artifact timestamp/workspace identity), while
     home-mode records must match their own running marker. Ensure workflow records
     without those claim shapes still receive zombie and command-line identity
     validation rather than being rejected solely because they are shaped differently.
   - Introduce a narrow observation/probe seam so unit and gateway snapshot tests can
     deterministically model alive, dead, zombie, wrong-command/recycled-PID, and
     claim-mismatch cases without relying on whichever PIDs happen to exist on the test
     host.

3. Correct the pure served-set precedence in
   `crates/sase_core/src/fleet_presentation.rs`.
   - A resolved owner dismissal excludes the candidate regardless of protection or
     apparent liveness; this prevents stale protected or recycled-PID rows from
     bypassing owner dismissal state.
   - Waiting/question protection keeps only non-definitively-dead candidates current. A
     protected `Dead` or `NotProcess` candidate must retire through the same bounded
     terminal path as another dead active-tier leftover, so old rows fall outside the
     seven-day/200-row window while recent rows retain the existing deterministic
     terminal treatment.
   - Preserve `Unknown` conservatism, current/recent/excluded partitioning, stable
     terminal ordering, terminal family-member suppression, and history safety behavior
     unless a focused parity fixture proves a different owner rule.
   - Update comments and field documentation so protection and dismissal semantics match
     the new precedence and no longer claim unconditional visibility.

4. Supply the owner facts consistently in gateway presentation and history builds.
   - Resolve the strong liveness observation once per indexed record and reuse it for
     dismissal-lineage seeding, presentation candidates, and projected details.
   - Continue resolving dismissal state for every candidate before selection; do not
     gate that lookup on liveness or protection.
   - Apply the same dismissal and strong-liveness safety rules to explicit history,
     while keeping presentation-only age/count bounds out of history.
   - Do not add or alter the phase-2 resolved-summary presentation fields or schema
     version unless compilation reveals a direct compatibility requirement.

5. Add regression and parity coverage.
   - In `sase_core`, replace the tests that pin the old behavior and add table-style
     coverage for dead protected rows, dismissed protected rows, dismissed rows with
     apparently alive/unknown liveness, and unaffected live/unknown undismissed rows.
     Assert every identity lands in exactly one decision bucket.
   - Cover the shared host-liveness helper with deterministic tests for zombie,
     wrong-command/recycled PID, valid agent process, stale project claim, matching
     project claim, and matching/mismatching home markers. Keep OS-specific parsing
     behind injected observations where necessary.
   - In `sase_gateway`, build one fixture index containing an ordinary live row, a
     dead-PID protected waiting row, a dismissed row, and an apparent-live row whose
     process identity does not match. Assert the presentation catalog contains exactly
     the owner-visible identity set and that history does not resurrect the dismissed or
     recycled-PID rows. This fixture is the phase's owner/gateway served-set parity
     gate.
   - Retain coverage that recent undismissed dead leftovers follow the existing
     recent-terminal window and that current active family members needed to materialize
     a missing root remain visible.

6. Verify the linked Rust repository with focused tests first, then its canonical checks
   (`just check`, which includes formatting, Clippy, and the workspace test suite). If
   this phase requires any tracked change in the primary `sase` repo, consult the
   project's lint/test reference memory and run the corresponding `just check` there as
   well; otherwise leave the Python/TUI tree unchanged.

## Phase handoff and closure

- Add a note to `sase-133.1` summarizing the rule choices: dismissal is evaluated before
  protection/liveness; dead protected rows use the bounded terminal path; zombie/command
  plus marker-or-claim observations supply liveness to the pure core selector. Record
  any out-of-scope discovery only as a `PROPOSED FOLLOW-UP:` note on this phase bead; do
  not create a bead.
- Run `sase bead epic-symbols sase-133.1`. Resolve every remaining symbol or re-key its
  Justfile annotation to the parent epic or a still-open later phase.
- Close only `sase-133.1` with
  `sase bead close sase-133.1 --note "<focused tests and canonical checks verified>"`.
  Do not close `sase-133` or any ancestor plan bead.
