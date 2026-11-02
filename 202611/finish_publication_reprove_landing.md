---
tier: tale
size: medium
title:
  Measure and audit the installed publication re-prove, then land sase-1pt.9 and
  sase-1pt
goal:
  The 73-run installed re-prove sample is measured against the pre-epic baseline, and
  every sampled run's sidecar obligations are audited and reported. The stale Justfile
  comment is gone. Epic sase-1pt.9 and its parent sase-1pt are closed, with their plan
  files marked done.
proposed_by: bbugyi200.athena.sase-1pt.9.land
proposed_with: claude/opus
bead: sase-1pt.9
create_time: 2026-11-02 16:22:20
status: wip
---

- **BEAD:** sase-1pt.9

# Measure and audit the installed publication re-prove, then land sase-1pt.9 and sase-1pt

## Why this tale exists

The land agent for epic `sase-1pt.9` (plan
`plan:202610/commit_publication_landing_repairs.md`) verified it against master
`84e22d98be` and recorded the result as a note on `sase-1pt.9`. Two pieces of epic work
remain. This tale finishes them and then performs the whole landing, because no other
land agent will resume after it.

1. **The installed re-prove was never measured.** Phase `sase-1pt.9.2` confirmed the
   install, then started a read-only collector. A human closed the phase after
   collection. Its last checkpoint (`file:explicit:7016ae2d8528704b800b8bff`, read with
   `sase artifact read`) still lists measure, audit, and report as remaining work, and
   no report artifact exists. The collector stopped at 2026-11-01T11:22Z. It holds 73
   qualifying no-conflict runs, which is above the plan's accepted minimum of 30 and
   below its 100-run target. Its exclusions are 2 runs booted before the install and 4
   conflict repairs. Accept this sample and state the count, the truncation, and the
   install time (2026-10-30T13:06:35Z, host commit `207ded297a`, which contains repair
   commit `6d615680b8`).
2. **One cleanup item was missed.** Phase `sase-1pt.9.1` did not delete the stale
   `Justfile` comment
   `# sase-1pt.6 provides publisher_is_live for the enqueue-only caller in sase-1pt.7.`

Already settled. Do not redo these:

- The publisher-repairs code is on master and its 29 regression tests pass.
- Later commits touching the publisher files do not conflict with it.
- Both `PROPOSED FOLLOW-UP:` notes from `sase-1pt.9.1` were declined as already fixed.
  The outcomes are on `sase-1pt.9`.
- `sase bead epic-symbols` currently lists nothing for `sase-1pt.9` or `sase-1pt`.

## Constraints carried from the epics

- Reviewer decisions are final: `convert_backlog = yes` and `decision_record = no`. Do
  not author a decision record. Do not edit any memory note.
- Queries over production state are read-only. Never install globally
  (`decisions:global-install-is-human-only`), never commit manually, and never run
  `check-full`. Hand long commands to `/sase_monitor`.
- Open sidecar or linked repositories only through `/sase_repo`. Read artifacts only
  through `sase artifact read`.
- No bead `--force` closes. Never use `--force` merely to make a close succeed.

## Step 1: Measure

Evidence lives in
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/30/20261030091257/installed-reprove/`.
It contains:

- `candidates.json`: `selected`, 73 runs with `run_id`, `agent`, `artifacts_dir`,
  `booted_at`, `host_commit`, and `finished_at`; and `excluded`, 6 runs with a reason.
- `runs/<run_id>/`: copies of each run's `done.json`, `finalizer_result.json`,
  `commit_results.json`, `final_submission_host.json`, `agent_meta.json`, and its
  `finalizers/` journal.
- `queues.jsonl`, `heartbeats.jsonl`, `drains.jsonl`, and `retention.jsonl` time series.
- `initial-*.json` backlog snapshots.
- `collect.py`, which records how each series was sampled.

The live `~/.sase/logs/agent_publication_drains.jsonl` and
`~/.sase/agent_publication/heartbeat.json` can fill gaps.

Check the selection independently. Re-derive the 73 runs from the copied evidence and
confirm three things for each: `host_commit` contains `6d615680b8`, the run committed,
and no conflict repair occurred. Report any disagreement. Never drop a run silently.

Compute these with a read-only script kept outside the evidence directory:

- median and p95 time from `provider_done` to `done.json`;
- finalizer-phase p95;
- per-substep `publication:` timings;
- publisher queue age p50/p95, from the heartbeat and queue series and from
  enqueue-to-ack times where they can be derived (state the 30-second sampling limit);
- the share of commits that took the background branch rather than the inline fallback.
  The commit log prints `Commit publication selected ...`.

Compare against the October pre-epic baseline from `sase-1pt.8`. Read its report with
`sase artifact read file:explicit:9ee3e3420293f5b9d4e45d18 "<reason>"`. Baseline on the
same host:

- n = 54;
- median 346.5 s and p95 562.1 s from model done to `done.json`;
- finalizer-phase p95 802.9 s;
- publication substep median 255.4 s.

Targets: finalizer-phase p95 under 60 s, median under 60 s, queue-age p95 under 10
minutes.

## Step 2: Audit every sampled run

A gap is a defect to diagnose, not a statistic. For each of the 73 runs, check:

- each declared repository pushed;
- its hood row was acknowledged, not left active or quarantined;
- its prompt archive is on the agents-sidecar remote and lists the run;
- its hood page shows terminal state;
- no eligible link-outbox rows remain for its `(run_id, agent)`;
- any planned prompt `cites` were published.

Note whether each obligation was met by the background publisher, by the inline
fallback, or later by a backstop such as hourly backfill.

Also check:

- the fenced Referenced By backlog (77 quarantined, 0 active at install) was converted
  to migrated `cites` events and acknowledged, or why not;
- the publisher stays idle between commits, judged by pass cadence while no work is
  queued;
- the outcome of any pin-only `sase-core-revision.txt` conflict and whether the
  revision-pin resolver handled it;
- the retention stamp cadence;
- whether the roughly 3,531-row quarantined hood backlog (`sase-1nf.11`) still confounds
  drain timings.

## Step 3: Act on the findings

- **A missed latency or queue target** is a measured residual. Record it in the report.
  For each cause, run `/sase_new_task` with details identifying proposing bead
  `sase-1pt.9`. That skill corroborates a duplicate, attaches the cause to a causally
  related active epic, or creates a sized task.
- **An audit gap or misbehavior caused by `sase-1pt` code** (refby-events,
  deferred-parity, publisher, enqueue-only, or publisher-repairs) is epic work.
  - When the fix fits this tale, fix it with a regression test that fails on the old
    behavior. Two examples: the publisher busy-loops while idle, or a commit takes the
    inline fallback while the publisher is live.
  - When it is too large to fix here, do not close anything. Add a note on `sase-1pt.9`
    describing the blocker, and report it in the final response.
- **A gap whose root cause is outside `sase-1pt`** goes through `/sase_new_task` naming
  `sase-1pt.9` as the proposer.
- Add a note on `sase-1pt.9` recording every triage outcome, including why any proposal
  was declined.

## Step 4: Cleanup and verify

- Delete the stale `Justfile` comment line
  `# sase-1pt.6 provides publisher_is_live for the enqueue-only caller in sase-1pt.7.`.
  Leave the surrounding epic-symbol comment block otherwise unchanged.
- Read the `lint_and_test` memory. Run `just fix`, then verify with
  `sase tool run check`, routing it through `/sase_monitor` if it may outrun the turn.
  Document any failure that reproduces identically on the clean base, with its
  reproduction. Such a failure does not block landing.

## Step 5: Report

- Write a Markdown report containing the sample definition and count, the truncation,
  each metric against its baseline and target, and the per-run audit table with every
  gap and its diagnosis. Also include the backlog, idle, pin-resolver, retention, and
  `sase-1nf.11` findings, plus each follow-up outcome.
- Store it with `sase artifact create --bead sase-1pt.9`.
- Add the headline numbers as a note on `sase-1pt.9`.

## Step 6: Close out epic `sase-1pt.9`

Only proceed when Step 3 left no unresolved epic-caused blocker.

1. Run `sase bead epic-symbols sase-1pt.9`. Resolve every listed entry under the
   Symvision epic-whitelist policy: wire the symbol up, privatize it, add a non-test
   pragma, or delete it. Re-key an entry only to a still-open later bead that needs it.
2. Run
   `sase bead close sase-1pt.9 --note "<verification: publisher-repairs on master and tested; installed re-prove measured and audited (report artifact ref, n, headline numbers); Justfile comment removed; follow-up outcomes>"`.
   If the close is rejected, fix the cause and close again, without `--force`.
3. Run `just symvision` (through `sase tool run` if it is guarded) and confirm the
   whitelist is clean.
4. Set `status: done` in the frontmatter of the plan file at the PLAN path shown by
   `sase bead read sase-1pt.9`, which is
   `plan:202610/commit_publication_landing_repairs.md`.

## Step 7: Resume and close parent epic `sase-1pt`

`sase-1pt` is a plan bead, and `sase-1pt.9` is its last open child.

1. Run `sase bead read sase-1pt -r "<reason>"`. Review its landing note #1, which
   triaged follow-ups (1) through (8). Confirm items (1), (4), and (5) are still owned
   by their recorded active epics or are fixed. Do not duplicate them.
2. Review every descendant (`sase-1pt.1` through `sase-1pt.9`, all of which must be
   closed) and their notes.
3. Review the linked plan `plan:202610/commit_stage_async_publication.md` and the
   post-child drift on master since `sase-1pt.9`'s landing.
4. Treat the parent as still complete only if both hold:
   - the measurement shows the background publication path actually working, meaning
     commits take the background branch and sidecar obligations are met;
   - any missed latency target has a recorded residual and follow-ups, as that plan's
     prove phase prescribes.
5. If complete:
   - Retire every entry from `sase bead epic-symbols sase-1pt` as in Step 6.
   - Run `sase bead close sase-1pt --note "<what you rechecked>"`.
   - Run `just symvision`.
   - Set `status: done` in `plan:202610/commit_stage_async_publication.md`'s
     frontmatter.
   - `sase-1pt` has no parent bead, so stop there.
6. If the parent is incomplete or ambiguous, add a note on `sase-1pt` describing the
   blocker, leave it open, and report it in the final response.

Finish through `/sase_final`.
