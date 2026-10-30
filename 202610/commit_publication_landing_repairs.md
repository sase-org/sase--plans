---
tier: epic
title: Finish the durable-publication-enqueue landing (publisher repairs and an installed
  re-prove)
goal: 'The background publisher from epic sase-1pt idles when it has no work. It converts
  the fenced Referenced By backlog itself and never holds a hood acknowledgment hostage
  to an unrelated drain. The epic''s latency and sidecar reliability claims are then
  measured on an athena install that actually contains the epic.

  '
parent_bead: sase-1pt
phases:
- id: publisher-repairs
  title: Stop publisher busy-loops and keep hood acknowledgment request-local
  depends_on: []
  size: medium
  description: 'publisher-repairs: stop keeping hints and re-ringing for link rows
    the drain cannot publish, convert the fenced Referenced By backlog in the publisher,
    and move the project-wide Referenced By drain out of the shared prompt-archive
    side-effect helper.'
- id: installed-reprove
  title: Re-prove commit-stage latency and sidecar obligations on the new install
  depends_on:
  - publisher-repairs
  size: medium
  description: 'installed-reprove: get a human-approved athena reinstall that contains
    publisher-repairs, then measure commit-stage latency and audit every sampled run''s
    sidecar obligations end to end.'
proposed_by: bbugyi200.athena.sase-1pt.land
proposed_with: claude/opus
create_time: 2026-10-30 03:24:13
status: wip
bead_id: sase-1pt.9
---

- **PROMPT:** [prompts/202610/commit_publication_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/commit_publication_landing_repairs.md)
- **PARENT:** [202610/commit_stage_async_publication.md](https://github.com/sase-org/sase--plans/blob/main/202610/commit_stage_async_publication.md)
- **BEAD:** [sase-1pt.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pt/sase-1pt.9.md)

# Finish the durable-publication-enqueue landing

## Why this child epic exists

The land agent for epic `sase-1pt` (plan
`plan:202610/commit_stage_async_publication.md`) verified every phase against master
`3266724c91` and found two kinds of remaining epic work.

1. Three defects in the new background publication path. All three are caused by
   `sase-1pt` and are described below.
2. The `prove` phase (`sase-1pt.8`) never measured the epic's code. Its own report
   (`file:explicit:9ee3e3420293f5b9d4e45d18`) states this.
   - Athena's `sase` is a uv editable install whose code directory is the primary
     checkout. That checkout is still at `5437e0baaa`, which lacks refby-events,
     deferred-parity, publisher, and enqueue-only.
   - The `sase_job_agent_publication` console script does not exist in the uv tool
     `bin/` directory, because a new `pyproject.toml` entry point needs a reinstall.
   - `~/.sase/agent_publication/heartbeat.json` has never been written. Every commit
     therefore still takes the inline fallback.
   - The prove agent closed its phase with "Measured, not yet proven" and did not
     request the reinstall.

The proof has to measure code that includes the repairs, so it depends on the repair
phase's commit. That is why this is a two-phase child epic and not a tale. Its
`parent_bead` link lets this epic's land agent resume and finish the `sase-1pt` landing.

Carry these parent constraints into both phases:

- The parent reviewer chose `convert_backlog = yes`.
- The parent reviewer chose `decision_record = no`. Do not edit any memory note,
  including `decisions`.
- Phase workers record discoveries as `PROPOSED FOLLOW-UP:` notes on their own phase
  bead and never create beads.

## Common implementation rules

- Read the `lint_and_test` memory before finishing a sase change, and verify with
  `sase tool run check`. Hand long commands to `/sase_monitor`.
- Never run `check-full`, raw guarded recipes outside `sase tool run`, manual commits,
  branches, or global installs (`decisions:global-install-is-human-only`). Propose
  installs through `/sase_gate`. Finish through `/sase_final`.
- Keep on-disk formats unchanged. No outbox, hint, or heartbeat schema change is needed.
- Every behavior change gets a regression test that fails on the old behavior. Do not
  weaken existing tests.
- Known pre-existing check failures on this tree are already tracked on other active
  epics. A check failure that reproduces identically on the clean base is documented
  with its reproduction and does not block the phase. The known failures are:
  - the stale `tests/contract_manifest.txt` (`sase-1pj`);
  - the `tests/test_suite_gate_scoped_integration.py` fixture (`sase-1pu`);
  - `test_sase_sudo_skill_teaches_reviewed_request_contract` (`sase-1pj.4.1`).
- Before closing a phase, run `sase bead epic-symbols <phase>` and resolve or re-key any
  entry.

## Phase publisher-repairs

Work in:

- `src/sase/agents_sync/publication_job.py`;
- `src/sase/agents_sync/prompt_archive/publish.py`;
- `src/sase/agents_sync/_commit_publication_transaction_steps.py`.

### 1. Do not keep hints for link rows the drain cannot publish

`_drain_project` sets `preserve_hints = True` (outcome `link_pending`) whenever
`drain_artifact_link_outbox(...).selected_retained` is non-zero. `_run_publication_job`
then re-rings the doorbell every pass. Hint claims are all-or-nothing, so one stuck
agent also keeps every other claimed hint alive.

Selection is by agent name only. Rows from another run of the same agent name that has
no release evidence are therefore retained until the stale-terminal cutoff, which is
`artifacts.retention.max_age_days` (default 90 days). Event-only and unauthorized rows
are retained the same way.

This is common on athena. `gh_sase-org__sase/artifact-link-outbox.jsonl` holds 11,066
rows, and hinted names such as `sase-1pn.land` (76 rows across two run IDs, neither with
release evidence) would pin the publisher to a 15-second wake cycle for months.

- Keep hints (and re-ring) for the link step only when the link drain was deferred by
  the deadline or raised.
- Rows that a completed drain attempted but retained stay durable for the hourly
  `artifact_link_backfill` drain, as they did after the pre-epic inline drain. This
  includes ineligible, event-only, unauthorized, and publish-failed rows.
- Report the retained count in the per-project outcome or diagnostics so the heartbeat
  still shows it. Then finish the claim.
- Tests:
  - A hinted agent with one eligible row and one row from another run without release
    evidence drains the eligible row and finishes the claim. It does not re-ring.
  - A deadline-deferred link drain still keeps the claim and re-rings.
  - A raised link drain still keeps the claim.

### 2. Convert the fenced Referenced By backlog in the publisher

`_drain_project` calls `drain_referenced_by_requests` only when
`legacy_referenced_by_writes_fenced(target)` is false. `_run_publication_job` re-rings
whenever `list_referenced_by_requests(..., include_quarantined=False)` is non-empty,
fenced or not.

Athena's sase project is fenced and holds 82 active plus 77 quarantined legacy rows. In
background mode their only conversion path today is the hood side-effect helper (item 3
removes it) or a manual `sase agent sync`. Until then, every pass re-rings.

- Call `drain_referenced_by_requests` whenever active requests exist.
  - In fenced projects it already dispatches to `convert_fenced_referenced_by_backlog`,
    which appends migrated `cites` events through the trusted machine lane. It takes no
    lock, pull, or snapshot, and it acknowledges each request after its append. This is
    `convert_backlog = yes`.
  - Unfenced projects keep the legacy drain unchanged.
- Count drained requests the same way for both branches.
- Tests:
  - A fenced fixture with active legacy requests converts and acknowledges them in one
    pass. It never takes the plans lock or pulls, and it does not re-ring for Referenced
    By afterwards.
  - An unfenced fixture still runs the legacy drain.
  - A conversion failure charges attempts. A request at the threshold quarantines, after
    which the publisher stops re-ringing for it.

### 3. Keep hood acknowledgment request-local

`record_prompt_archive_side_effects` (`prompt_archive/publish.py`) ends by calling
`_drain_referenced_by_after_prompt_publish`, a project-wide drain. The deferred path
(`acknowledge_completed` in `_commit_publication_transaction_steps.py`) calls that
helper for each verified request. Any non-empty combined error then charges the request
an attempt and skips its acknowledgment, even though its hood and archive are already
pushed. The drain also runs inside the hood drain's 120-second alarm.

So a failure in an unrelated backlog row can charge, and eventually quarantine, a hood
row that already published. In unfenced projects, the legacy drain's plans-sidecar lock,
pull, and full snapshot (about 50 s in the research report) can push the hood
transaction past its alarm on every pass. Timeouts are not charged, so it retries
forever.

The parent plan's `deferred-parity` step 3 listed only `upsert_ref_file_versions` and
the `cites` emission or legacy enqueue for the shared helper. The drain belongs to the
publisher (item 2) and to `sase agent sync`.

- Make the shared helper do only the per-request side effects:
  - `upsert_ref_file_versions`;
  - `_record_prompt_cites`.
- In the synchronous `_publish_prompt_archive` path, keep calling
  `_drain_referenced_by_after_prompt_publish` right after the helper and keep combining
  its error into the outcome. The inline path stays byte-for-byte equivalent.
- The deferred path no longer drains Referenced By. It charges a request only for its
  own `cites` persist failure.
- Tests:
  - A deferred acknowledgment acks and charges nothing when an unrelated Referenced By
    drain would fail. Make the drain fail through the drain function, not the helper.
  - A deferred request whose own `cites` persist fails is charged and not acknowledged.
  - The synchronous path still drains Referenced By after a prompt publish.
  - Side effects still run exactly once per acknowledged request. Keep the existing
    parity tests green.

### 4. Cleanup and docs

- Delete the stale `Justfile` comment
  `# sase-1pt.6 provides publisher_is_live for the enqueue-only caller in sase-1pt.7.`
  Its `--epic-symbol` row was already removed by `e69360f49d`.
- Update the publisher paragraphs in `docs/reference/agents/hood-sync.md`,
  `docs/reference/artifact-references.md`, and
  `docs/reference/automation/scheduler-housekeeping.md` where they describe what keeps
  hints, what re-rings, or who drains Referenced By. Read `docs_writing` first.
- Update the `agent_publication` job description in `src/sase/default_config.yml` if its
  wording changes, then regenerate references through `just fix`.

## Phase installed-reprove

This phase is the parent plan's `prove` phase, run against code that really contains the
epic and the repairs. Follow the parent plan's "Phase prove" section for the metrics and
targets. Read the earlier report with
`sase artifact read file:explicit:9ee3e3420293f5b9d4e45d18 "<reason>"`.

### 1. Install

- Confirm the installed sase contains the `publisher-repairs` commit:
  - `sase version` shows the code directory of the editable install. Run
    `git merge-base --is-ancestor <commit> HEAD` in that checkout.
  - The uv tool `bin/` directory has `sase_job_agent_publication`.
  - A fresh `~/.sase/agent_publication/heartbeat.json` appears within a few minutes of
    the scheduler running.
- If any of those fail, request the fix through `/sase_gate`, with a follow-up that
  continues this phase. Never install yourself. The fix usually means updating that
  checkout to master, `just install`, and restarting the scheduler if the `publication`
  routine does not start.
- Creating the gate ends your turn by design (`decisions:gates-never-block`). That is
  expected. Do not close the phase before measuring.

### 2. Wait

Under `/sase_monitor`, wait until at least 100 no-conflict committing runs have finished
on the new install, or 72 hours have passed. Accept a minimum of 30 runs. Report the
count and the install time.

### 3. Measure

Use read-only queries over run artifacts, `finalizers/` journals,
`~/.sase/logs/agent_publication_drains.jsonl`, and the publisher heartbeat.

- Median and p95 time from `provider_done` to `done.json`.
- Finalizer-phase p95.
- Per-substep `publication:` timings.
- Publisher queue age p50/p95.
- The share of commits that took the background branch rather than the inline fallback.
  The commit log prints `Commit publication selected ...`.

Compare against the October baseline and `sase-1pt.8`'s pre-epic numbers on the same
host:

- n = 54;
- median 346.5 s and p95 562.1 s from model done to `done.json`;
- finalizer-phase p95 802.9 s;
- publication substep median 255.4 s.

Targets: finalizer-phase p95 under 60 s, median under 60 s, queue-age p95 under 10
minutes.

### 4. Audit

Audit every sampled run. A gap is a defect to diagnose, not a statistic.

- Each declared repository pushed.
- Its hood row was acknowledged.
- Its prompt archive is on the agents-sidecar remote.
- Its hood page shows terminal state.
- No eligible link-outbox rows remain for its `(run_id, agent)`.
- Any planned `cites` were published.

Also check:

- The fenced Referenced By backlog was converted to migrated `cites` events and
  acknowledged.
- The publisher stays idle between commits. Look at the pass cadence while no work is
  queued.
- The outcome of any pin-only `sase-core-revision.txt` conflict, and whether the
  revision-pin resolver handled it.
- The retention stamp cadence.
- Whether the 3,527-row quarantined hood backlog (`sase-1nf.11`) still confounds drain
  timings.

### 5. Report

Store the report with `sase artifact create --bead <this phase>` and attach the numbers
to the phase bead. For every missed target, record the measured residual and a
`PROPOSED FOLLOW-UP:` for each cause. Do not author a decision record.

## Out of scope

- Ringing the doorbell from monitor, gate-turn, or staged done-marker writers. Their
  rows publish through the five-minute backstop, and `installed-reprove` measures
  whether that is fast enough.
- Everything in the parent plan's "Out of scope" list.
