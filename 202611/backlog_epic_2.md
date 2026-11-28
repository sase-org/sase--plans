---
tier: epic
title: 'Backlog Epic 2: leak gate, federation freshness, bench integrity, and load
  flakes (final round)'
goal: 'The five most important remaining open task beads are fixed and closed. The
  land agent verifies, closes this final epic, and stops: the user capped the backlog
  chain at two rounds, so no Backlog Epic 3 is planned.'
phases:
- id: leak-gate
  title: Fix the leak-detector thread poison on the CI Telemetry gate (sase-1vg)
  depends_on: []
  size: large
  description: 'leak-gate: fix the root cause of sase-1vg. Read sase-1vg and all of
    its notes first. ACE pilot tests intermittently leave a sase-config-token-refresh
    thread alive past the leak-detector snapshot, poisoning global sase.config.core._live_config_token_refresh_threads
    on the CI Telemetry test-cost gate (run 46071761552: 63,676 passed in 2:03 inside
    the 180m cap, failing only this gate). Roughly 1 in 3 recent Telemetry runs. Fix
    it at its root; do not exempt the name from the detector. Verify with the cost
    lane and just check through sase tool run check, then close sase-1vg and the phase
    bead.'
- id: federation-freshness
  title: Federation cache-only reads must not report stale snapshots as ok (sase-1aw)
  depends_on:
  - leak-gate
  size: medium
  description: 'federation-freshness: fix sase-1aw. Read sase-1aw and all of its notes
    first. Cache-only federation reads ignore the last network failure and label the
    stale cache ok; the bug body carries the reproduction and proposed fix. Make cache-only
    reads surface the failure instead of a stale ok. Add or update a regression test
    for the stale-after-failure shape. Run sase tool run check, then close sase-1aw
    and the phase bead.'
- id: bench-integrity
  title: Axe j/k benchmark must not capture Agents-tab samples under load (sase-199)
  depends_on:
  - federation-freshness
  size: large
  description: 'bench-integrity: fix sase-199. Read sase-199 and all of its notes
    first. The Axe j/k bench collects samples from tabs other than the measured one
    under load (4 corroborations). Scope the sample window to the measured tab so
    bench results stay meaningful. Cover with a regression test that fails on the
    old cross-tab capture. Run sase tool run check, then close sase-199 and the phase
    bead.'
- id: index-lock-flake
  title: Stale index-lock recovery flake under loaded scoped check (sase-1kr)
  depends_on:
  - bench-integrity
  size: large
  description: 'index-lock-flake: fix sase-1kr. Read sase-1kr and all of its notes
    first (7 corroborations; pairs with sase-1s4, a test that leaves a stale lock
    in a real checkout). Make test_run_workspace_git_command_recovers_stale_index_lock
    robust under the loaded scoped lane, or fix the product recovery path it guards
    if the flake exposes a real race. Do not close on a single green rerun; show the
    node green across repeated loaded runs. Run sase tool run check, then close sase-1kr
    and the phase bead.'
- id: admission-overlap-flake
  title: 'Tool-admission overlap flake: rule out a real limit violation (sase-1vw)'
  depends_on:
  - index-lock-flake
  size: large
  description: 'admission-overlap-flake: fix sase-1vw. Read sase-1vw and all of its
    notes first (related sase-1ve tracks a different node in the same module). test_limit_two_with_four_runs_never_overlaps_more_than_two
    failed once under host load with no admission change in the finding diff, then
    passed 5/5 isolated plus a green whole-module run. Determine whether the overlap
    measurement needs a semantic wait or the failure is a real admission-limit violation
    with more than two concurrent runs at limit two; fix whichever it is. Do not close
    on a single green rerun. Run sase tool run check, then close sase-1vw and the
    phase bead.'
proposed_by: bbugyi200.athena.sase-1vn.land
proposed_with: muse/muse-spark-1.3-contributor
create_time: 2026-11-28 03:14:39
status: wip
bead_id: sase-1vy
---

- **PROMPT:** [prompts/202611/backlog_epic_2.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/backlog_epic_2.md)
- **BEAD:** [sase-1vy](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1vy/README.md)

# Backlog Epic 2: Leak Gate, Federation Freshness, Bench Integrity, and Load Flakes (Final Round)

This is the second and final backlog epic. The user capped the chain at two rounds
(sase-1vn note #1), so this epic closes the chain: its land agent verifies, closes this
epic, and stops. No Backlog Epic 3 is planned, and no child plan of this epic may plan
one either.

picks: sase-1vg (the only red on the CI Telemetry test-cost gate and `just check-full`),
sase-1aw (federation reports stale snapshots as ok after network failure), sase-199 (Axe
j/k bench cross-tab sample contamination, +4), sase-1kr (stale index-lock recovery
flake, +7), sase-1vw (tool-admission overlap flake that may hide a real limit violation,
new since triage).

Governing decisions: `decisions:check-full-is-explicit` (verify with
`sase tool run check`, never `just check-full`),
`decisions:handoffs-end-through-host-completion`, and `decisions:host-owned-completion`.

## Land: Verify, Close This Final Epic, and Stop

The land agent does its normal verification, integration, and follow-up triage, then
closes this epic. It does not plan another epic.

Order matters only in that this plan was proposed after sase-1vn closed, so it stands
alone as a top-level epic.

Steps after all phases close:

1. **Verify.** Run `sase tool run check` (and `sase tool run check` in sase-core if any
   phase touched it). Treat UNKNOWN triage as yours. Confirm `sase bead epic-symbols`
   for this epic is empty, resolving any entry per `symvision.md` (read it with
   `/sase_memory_read`).
2. **Integrate.** Review non-epic commits since this epic started for drift that
   duplicates or conflicts with what the epic added; integrate anything needed.
3. **Follow-up triage.** Collect each phase's `PROPOSED FOLLOW-UP:` notes and either
   file them as task beads (via `/sase_new_task`) or record a declined-with-reason note
   on the owning bead. Record newly discovered issues on the causally linked epic when
   one exists.
4. **Close.** Run `sase bead close <epic> --note "<verification>"` with the check run
   id, drift outcome, and triage outcome. Never use `--force`. Then run `just symvision`
   and confirm it is clean.

This epic is the last round. Its land agent finishes normally after closing the epic. If
its landing needs a child plan, that child plan inherits the same stop: it finishes
after closing this epic and plans nothing further.

Candidates the user may pick up by hand (not in this epic):

- bead-event actor attribution: sase-159, sase-15a, sase-1ri;
- bead-scale performance: sase-1iu, sase-1iv, sase-1iw, sase-1ix, sase-1j5;
- stale memory notes, batched by note: tui_perf.md (sase-1f2, sase-1fd, sase-1fq,
  sase-1jr), macros.md (sase-1hu, sase-1jq), lint_and_test.md (sase-1dk, sase-1hv), and
  glossary strands (sase-1bx, sase-1c9, sase-1cw, sase-1kc);
- other triaged queue beads not picked here: sase-1em, sase-1oo/sase-1on/sase-1pr
  broad-lane isolation cluster (with sase-1tp, sase-1to, sase-1tq, sase-1tr, sase-1ts,
  sase-1sg), sase-r2, sase-1f9, sase-1v2, sase-1q8 (with sase-1tn evidence; fakey
  runner-slot cluster with sase-1u0, sase-1u1, sase-n6, sase-v0), sase-1j2, sase-1nl,
  sase-1fw, sase-1u6, and the newer flakes sase-1vv and sase-1vx.
