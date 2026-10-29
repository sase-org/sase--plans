---
tier: tale
title: Finish landing the staged plan-handoff foundation
goal:
  A held staged handoff tells the user which handoff was held and why, staged stranded
  facts are proven against the pinned Rust reducer, and epic sase-1pj.1.1 and its parent
  phase sase-1pj.1 are closed with verified evidence.
size: small
proposed_by: bbugyi200.athena.sase-1pj.1.1.land
proposed_with: claude/opus
bead: sase-1pj.1.1
status: done
---

- **PARENT:**
  [202610/staged_plan_foundation.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_plan_foundation.md)
- **BEAD:**
  [sase-1pj.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.1.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1pj.1.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1pj.1.1.land.md)
- **COMMITS:**
  - [71cfd8e](https://github.com/sase-org/sase--plans/commit/71cfd8e5c53269447ee862449716f8ac48106c47)
    — docs(plans): mark staged_plan_foundation.md done after sase-1pj.1.1 landing

# Plan: Finish landing the staged plan-handoff foundation

## Context

Epic `sase-1pj.1.1` ("Stage plan handoffs after host finalization") is a child epic of
phase `sase-1pj.1` in epic `sase-1pj`. All five of its phases are closed. Their work is
on sase master in commits `06cd630543`, `dd59c28f44`, `0f73623591`, `9f88875b98`, and
`cc32e69dbb`. The sase-core commits `cd62c83d` (staged-handoff contract) and `751bba5f`
(stranded ownership of staged facts) are both inside the current
`sase-core-revision.txt` pin. The `staged_handoffs` flag is beta and off by default; its
flag bead is `sase-1pk`.

The land agent verified the epic against its plan
(`plan:202610/staged_plan_foundation.md`) and the code. The 158 focused staged-handoff
tests pass on master. The land agent also triaged every `PROPOSED FOLLOW-UP:` and
recorded the outcomes on `sase-1pj.1.1`. Two pieces of epic work remain, listed below,
followed by the closeout. Do not reopen anything the triage note settled.

Read `sase bead read sase-1pj.1.1 -r "<why>"` first. Its LAND TRIAGE note and the
HANDOFF API note on `sase-1pj.1` summarize what was verified.

## 1. Name the held handoff in the failure notification

The plan requires this for failure-boundaries, and the parent plan's failure table
repeats it: when adoption is refused, the run fails with `handoff_not_adopted` and the
notification must name the held kind, the staged ID, and the concrete reason. Today the
receipt is written (`_write_held_receipt` in `src/sase/axe/run_agent_exec_staged.py`).
The runner then breaks out of the loop with `loop_outcome = "failed"` and no exception,
so `error_summary` stays `None`. The completion notification therefore says only
`<agent> failed: <workflow>`.

Change `src/sase/axe/run_agent_runner_finalize.py`:

- Add a private helper `_held_staged_handoff_note(current_artifacts_dir)` next to
  `_sdd_publication_error`. Lazy-import `StagedHandoffError`, `read_adoption_receipt`,
  and `read_staged_handoff` from `sase.agent.staged_handoff`, as the other runner
  readers do.
  - When the receipt exists with `outcome == "handoff_not_adopted"`, return
    `Staged <kind> handoff <staged_id> was not adopted: <reason>. Its staged record and archived proposal are kept for re-adoption.`
  - `<kind>` comes from the staged record's `kind`. Use `malformed` when reading the
    record raises `StagedHandoffError`, and `unknown` when the record is absent. Use
    `unknown reason` when the receipt has no `reason`.
  - When reading the receipt raises `StagedHandoffError`, return a note saying the
    staged handoff state is malformed and held. Malformed staged state is a held
    failure, never absence.
  - Return `None` when there is no artifacts dir, no receipt, or an `adopted` receipt.
  - Return `None` on any other exception. Notification building must never raise.
- In `send_completion_notification`, when `not success`, append that note to `notes`
  right after the `error_summary` note. This covers both held paths: the plain held
  dispatch, and an adopter exception that re-raises after writing the held receipt (that
  path also carries an `error_summary`).

Add tests to `tests/test_run_agent_runner_notifications.py`, using its `base_kwargs`
fixture and `test_failed_notification_includes_held_workspace_recovery` as the pattern.
Write the record into `base_kwargs["current_artifacts_dir"]` with
`build_plan_staged_record(plan_file=..., original_file=..., dirty_obligations_at_stage=[])`,
dumped to `STAGED_HANDOFF_FILENAME`. Write the receipt with
`write_adoption_receipt(..., artifacts_dir=...)`. Cover these cases:

- A held receipt plus a plan record produces a note with `plan`, the staged ID, and the
  reason.
- An `adopted` receipt produces no held note.
- No staged files produces no held note.
- A malformed receipt file produces the malformed note, and the notification is still
  sent.

## 2. Prove staged stranded facts against the pinned reducer

Phase 3 (`sase-1pj.1.1.3`, note #2) deferred a sase-side integration test until the pin
contained `751bba5f`. The pin now contains it. `src/sase/bead/stranded_facts.py` still
carries the stale comment in `_member_facts`, which says the staged fields wait for the
pin and are omitted so an older core accepts the member.

- In `_member_facts`, put `"staged_handoff": staged_kind` and
  `"staged_receipt_outcome": staged_outcome` directly in the facts dict, as the other
  optional fields are. The Rust wire declares both as
  `#[serde(default)] Option<String>`, so `None` is accepted. Delete the conditional
  assignments and the stale comment. If an existing test compares a whole member dict,
  update it.
- Add collector tests, either in `tests/test_stranded_collector.py` or in a new
  `tests/test_stranded_staged_handoff.py` that reuses
  `tests/_stranded_collector_helpers.py`. Model them on
  `test_pending_handoff_marker_owns_the_bead`: a `done` record that is not live, under
  `phase("sase-1.2", assignee="sase-1.2")`. Assert both the collected member facts and
  the real reducer verdict from `load_stranded_report(PROJECT, now=_NOW)`; see
  `tests/test_stranded_single_turn.py` for how that is used with `wire_collector`.
  - A valid staged plan record gives `staged_handoff == "plan"`, verdict `owned`, reason
    `handoff`.
  - Only a `handoff_not_adopted` receipt gives
    `staged_receipt_outcome == "handoff_not_adopted"`, verdict `owned`, reason
    `handoff`.
  - A malformed staged file (for example `{}`) gives `staged_handoff == "malformed"`,
    verdict `owned`, reason `handoff`.
  - Only an `adopted` receipt is not owned through `handoff`.

## 3. Verify

Read `lint_and_test.md` with `/sase_memory_read`. Run the new and touched tests, plus
`tests/test_axe_run_agent_exec_staged.py`, `tests/test_staged_failure_boundaries.py`,
and `tests/test_staged_handoff_acceptance_matrix.py`. Then run `just fmt` and
`sase tool run check`. Do not run `check-full`.

Three failures already exist on clean master and are owned elsewhere. Do not fix them
here:

- `tests/monitor/test_monitor_start_supervisor.py::test_start_monitor_scrubs_agent_identity_from_the_supervisor_env`
  and
  `tests/monitor/test_monitor_start_ack.py::test_start_monitor_kills_a_supervisor_that_never_writes_the_ack_marker`
  belong to task `sase-1pg`.
- `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`
  (`tests/test_ratchet_core_window_tool_ref.py` is missing from the manifest) is a
  DISCOVERED ISSUE on epic `sase-1pi`.

Treat any other failure as yours.

## 4. Close out the epic and its parent phase

This tale is the epic's landing. Do every step in this turn; none of them waits for this
work's commit.

1. Run `sase bead epic-symbols sase-1pj.1.1` and `sase bead epic-symbols sase-1pj.1`.
   Both were empty at planning time. If either lists entries, resolve each symbol per
   the Symvision epic-whitelist policy (read `symvision.md` with `/sase_memory_read`):
   wire it up, privatize it, add a non-test pragma, or delete it. Re-key a Justfile line
   only to a still-open later bead that really needs the exemption.
2. Close the epic. Fill in the verification evidence from steps 1–3:

   ```bash
   sase bead close sase-1pj.1.1 --note "Land verified: all five phases closed with commits 06cd630543, dd59c28f44, 0f73623591, 9f88875b98, cc32e69dbb; sase-core cd62c83d and 751bba5f are in the pin; staged_handoffs is beta and off by default (flag bead sase-1pk); 158 focused staged tests passed on cc32e69dbb; the post-start drift (including the e056c6b8c0 invoke split) preserved the staged paths. Closeout tale: held staged handoffs now name kind, staged ID, and reason in the failure notification; staged stranded facts are proven against the pinned reducer, and the stale comment is gone. <tests and sase tool run check result>. Follow-up triage is recorded in this bead's LAND TRIAGE note."
   ```

   Never use `--force` to make the close succeed. If the close is refused for leftover
   epic symbols, finish step 1 and close again.

3. Run `just symvision`. If the recipe guard refuses the raw run, use
   `sase tool run symvision`. Confirm the whitelist is clean.
4. Set `status: done` in the epic's plan-file frontmatter. That file is
   `plan:202610/staged_plan_foundation.md`; edit the path that
   `sase bead read sase-1pj.1.1 -r "<why>"` prints under EPIC PLAN. Change only the
   `status` field.
5. The epic's `parent_bead` is phase `sase-1pj.1` ("Staged-handoff foundation and plan
   proposals"). Check that its plan-staging scope in `plan:202610/staged_handoffs.md` is
   complete: flag, record and guards, adoption epilogue, real final response, finalizer
   interplay, failure semantics, marker-reader audit, `sase plan propose` staging,
   dual-state skills plus `macros.md`, and both-flag-state tests. One item stays open by
   design: the `memory-sase.template.md` core-note rewrite. It is recorded as
   `PROPOSED FOLLOW-UP:` #1 on `sase-1pj.1` for the `sase-1pj` land agent, as is the
   PR-mode boundary (#2). Then close only that phase:

   ```bash
   sase bead close sase-1pj.1 --note "Plan-staging is complete through child epic sase-1pj.1.1 (closed): <one-line summary of the verification above>. The public staged/adopter API and flag ID are recorded in this bead's HANDOFF API note for handoff-wire, questions-pipe, gate-staging, monitor-staging, and adoption-recovery. PROPOSED FOLLOW-UP #1 (core-note authorization) and #2 (PR mode) remain for the sase-1pj land agent."
   ```

   Do not close `sase-1pj`. Its land agent is already waiting.
