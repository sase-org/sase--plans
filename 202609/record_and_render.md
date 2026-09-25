---
tier: tale
title: Record and render ToolRun failure triage
goal:
  Settled named ToolRuns persist failure triage and expose it in the flagged footer and
  ungated show views.
size: medium
proposed_by: bbugyi200.athena.sase-18j.6
bead: sase-18j.6
status: done
---

- **PARENT:**
  [202609/tool_e3_failure_triage.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e3_failure_triage.md)
- **BEAD:**
  [sase-18j.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-18j/sase-18j.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-18j.6](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-18j.6.md)
- **COMMITS:**
  - [71b25fb](https://github.com/sase-org/sase/commit/71b25fbf4eb20f3e184f25104307e20cc5488515)
    — feat(tool): render settled failure triage

# Record and render ToolRun failure triage (sase-18j.6)

## Goal

Complete only epic phase `sase-18j.6` from `plan:202609/tool_e3_failure_triage.md`.
Every settled named ToolRun records bounded, fail-open triage, and the new
`tool_failure_triage` beta flag gates only the footer. `sase tool show RUN` and
`show RUN -j` expose stored triage without the flag and after logs are reaped. Preserve
process exit codes and flag-off output. Do not close the parent epic or any ancestor.
Record unrelated discovered work as a `PROPOSED FOLLOW-UP:` note on this phase, not a
new task bead.

## Implementation

1. Read the phase design and current Rust binding wire shapes. Inspect
   `src/sase/tool/executor.py`, `stage_protocol.py`, `executor_display.py`, `query.py`,
   `triage_inputs.py`, `tools/run_silent`, `tools/_run_silent_record.py`, the ToolRun
   adapter, and the focused tests. Use only the pinned core binding for classification
   and persistence. Treat the existing precision backtest results from phase 5 as the
   prerequisite; note the known possible-owner overmatch on `sase-18j.6` and present
   owners only as suggestions.
2. Capture each failed `run_silent` stage's bounded output in its retained run
   directory, recording its path in the event protocol so the shared executor can find
   it after stage ingestion. Keep recording and log retention safe for missing or
   truncated output; test core retention of stage-output files. Preserve the
   byte-identical no-handshake behavior for human `run_silent` invocations.
3. In `run_recorded_body`, after `tool_run_finish`, assemble the failed-stage outputs,
   the bounded 256 KiB tail of the output of record for `stages: none` or failed runs
   without a failed stage, authoritative continuation records, and the bounded gatherer
   inputs. Call `tool_run_triage_settle` for recorded named runs, including adopted
   workers. Do not triage ad-hoc runs. Enforce the five-second total budget with
   per-gatherer limits. On any missing input, exception, timeout, or unwritable store,
   retain the original exit status and record or display a diagnostic where possible. Do
   not call any bead mutation entry point.
4. Create the planned beta flag through `sase flag new tool_failure_triage` using the
   three exact enabled/disabled/removal sentences in the epic design, and paste its
   printed registry entry. Recording and `tool show` stay ungated. Implement a shared
   presentation formatter for the footer and `tool show` so ids, labels, evidence, and
   owner wording agree. With the flag on, render per-failed-stage class counts and
   continued/stopped state; in compact mode include the capped NEW and UNKNOWN items
   first, up to three KNOWN and FLAKY examples, repeat marker, existing show pointer,
   and verdict last. Non-compact mode prints stage lines and verdict. Successful runs
   have no verdict. Flag-off footer output is unchanged, including `-T` behavior.
5. Add an ungated TRIAGE section to human `sase tool show RUN` and a versioned `triage`
   object to `show -j` using stored core data. Include kind, verdict, extraction status,
   continuation decision, item class, stage, evidence, and possible owner. Support both
   ordinary and `-F` final show paths, and legacy runs with missing triage tables or
   rows. Keep read paths read-only.
6. Remove this phase's consumed `--epic-symbol` entries from the Justfile; keep symbols
   reserved for phases 7 and 8 assigned to an open bead. Run
   `sase bead epic-symbols sase-18j.6` immediately before close and resolve or re-key
   every leftover entry. Close only `sase-18j.6` with a verification note.

## Verification

- Focused tests use an isolated `SASE_HOME` and fixture named-tool catalogs for a
  foreground run and adopted worker, seeded evidence runs, failed-stage capture, stage
  decisions, `stages: none`, and an ad-hoc exclusion. Compare stored rows, footer, and
  `show -j` ids and labels. Reap logs and verify that `show -j` retains triage and
  existing spawn, ingestion, and log-write diagnostics.
- Verify flag-on compact and non-compact output, flag-off golden parity, successful-run
  silence, `-T`, 5-second budget, slow gatherer, unwritable store, no bead mutations,
  linked-repo isolation, and unchanged child exit codes.
- Run `just install` if the workspace needs it, `just fix`, focused tests, then
  `sase tool run check` (the required `just check` recipe). If a failure is identical on
  a clean base tree, record it as a `PROPOSED FOLLOW-UP:` note with any existing task
  bead and close the phase. Never run `check-full` here.
