---
tier: tale
title: Close the deferred SASE eval-pilot phase
goal:
  Close only sase-1jo.3 under eval_pilot=skip and record the full pilot spec for the v2
  report.
size: xsmall
proposed_by: bbugyi200.apollo.sase-1jo.3
bead: sase-1jo.3
create_time: 2026-10-10 15:51:01
status: wip
---

- **PARENT:**
  [202610/databricks_followups.md](https://github.com/sase-org/sase--plans/blob/main/202610/databricks_followups.md)
- **BEAD:**
  [sase-1jo.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jo/sase-1jo.3.md)

# Complete sase-1jo.3 under the approved eval_pilot = skip decision

## Outcome and scope

Close only phase `sase-1jo.3` with an auditable note that hands the full deferred
agent-eval pilot specification to phase `sase-1jo.4` (the v2 report).

The parent epic's final reviewer decision is `eval_pilot = skip`. Its design explicitly
says: "Close the phase with no changes. Note in the bead that the v2 report must carry
the full pilot spec." The harness and harness_and_runs branches are excluded. No
harness, task manifest, tests, model calls, research-file changes, or memory edits are
required. Do not reopen this decision or close the parent epic or any ancestor plan
bead. Do not create beads or hand-edit statuses.

There were no earlier work notes on this phase at planning time, and
`sase bead epic-symbols sase-1jo.3` reported no entries. This is a fixed administrative
closure that one coding agent can complete; a child epic is unnecessary.

## Authoritative context

- Read `sase-1jo.3` with
  `sase bead read sase-1jo.3 -r "Confirm final skip decision and phase notes before closure"`.
- Read the design through
  `sase artifact read plan:202610/databricks_followups.md "Confirm evalpilot skip-branch acceptance criteria"`.
- The original source is
  `research:202610/databricks_nyc_cv_and_role_pitches/databricks_nyc_cv_and_role_pitches.md`,
  section "Application plan and timeline", item 4. Both it and the epic design were read
  through audited artifact reads when preparing this plan.
- Use `/sase_beads` and `/sase_memory_read` for the applicable bead workflow.

## Work

1. Confirm the phase remains assigned/in progress and the final decision is still
   `skip`. Capture `git status --short` before work so existing changes are not confused
   with this phase's work.
2. Append a durable note with `sase bead note sase-1jo.3 '<text>'`. The note must
   identify the final `eval_pilot = skip` decision, state that no pilot harness or real
   runs were produced, and tell `sase-1jo.4` to carry this full deferred spec:
   - An optional pilot between applying and the onsite, without delaying applications.
   - 8-12 coding tasks from real, already-merged history. Freeze each start at the
     selected commit's parent and use the commit's own tests as the oracle in a
     declarative task manifest.
   - Explicit success checks for passing tests, staying within the task's scope, and
     producing required artifacts. Keep timeouts and failed runs in the denominator.
   - Record harness, model, version, and exact prompt for every run. Repeat a subset and
     report variance and pass@k with the per-task outcomes.
   - Log runs as MLflow traces. Use `mlflow.genai.evaluate` scorers only when they
     clarify results; localize and report one regression from a trace.
   - Keep `fakey` conformance results separate from real-agent task-success scores.
   - For a future approved implementation, place dev tooling in
     `tools/agent_eval_pilot/`, first reading `tools/AGENTS.md`; keep MLflow optional
     (for example `uv run --with mlflow`) and add no runtime dependency.
   - Consider `sase tool run` for success-check records and triage evidence. Unit tests
     use a stub launcher; plumbing smoke uses explicitly labeled `fakey`. Real launches
     from an agent require `/sase_run` LaunchApproval and waiting uses `/sase_monitor`.
     The future README should supply one real-pilot command and explain the outcome
     table and pass@k. Include both authoritative artifact references in the note. These
     are deferred requirements, not completed capabilities or measured results.
3. Immediately before closure, run `sase bead epic-symbols sase-1jo.3`. If entries have
   appeared, resolve them or re-key them to the still-open parent epic or a later phase
   before closing. Do not add exemptions for this administrative work.
4. Close only this phase with
   `sase bead close sase-1jo.3 --note "Verified final eval_pilot=skip branch; full deferred pilot spec recorded for sase-1jo.4; no implementation changes or model runs; no remaining phase epic-symbol entries."`

## Verification and completion

- Read the phase again through `sase bead read` to verify the note and closed status.
- Confirm repository status shows no source changes introduced by this phase. The design
  requires `just check` only when evalpilot changes the sase repo; the selected skip
  branch makes no implementation changes, so no test run is needed.
- If unrelated work is discovered, record `PROPOSED FOLLOW-UP:` on this phase rather
  than creating a bead. Leave the parent and sibling phases for their own agents.
- End through `/sase_final` as required by SASE. Report that the phase was closed under
  the approved skip decision and the pilot remains deferred to the v2 report.
