---
tier: epic
title: Race-free plan-lane epic clan summaries
goal: 'Epic clan panels reliably render the PLAN-lane-style rich summary through the
  generic %clan summary_script machinery — never the bare "EPIC <id>" identity fallback
  in normal operation — because the launch flow hands the summary script a race-free
  plan source, the runner re-resolves the summary once the prepared workspace exists,
  and every residual failure leaves durable, attempt-labeled diagnostics instead of
  silently disappearing.

  '
phases:
- id: diagnostics
  title: Durable attempt-labeled clan summary diagnostics
  depends_on: []
  size: small
  description: '''Durable attempt-labeled clan summary diagnostics'' section: capture
    each summary-script run''s stderr and outcome into a per-launch artifact file
    so fallbacks are never silent again, working around the runner-stdout overwrite
    that destroyed prior diagnostics.'
- id: snapshot
  title: Race-free epic plan snapshot at launch creation
  depends_on:
  - diagnostics
  size: medium
  description: '''Race-free epic plan snapshot at launch creation'' section: have
    the epic work launch flow copy the approved plan file to a durable per-project
    snapshot and export its absolute path to launch segments, and teach the epic summary
    script to use it as a guaranteed-local plan candidate.'
- id: reresolve
  title: Post-preparation summary re-resolution in the runner
  depends_on:
  - snapshot
  size: medium
  description: '''Post-preparation summary re-resolution in the runner'' section:
    re-run the declaring member''s summary script after workspace and sidecar preparation
    with a reconstructed launch environment, persist the fresher result last-success-wins,
    document the multi-run script contract, and pin the end state with an end-to-end
    race-shaped launch test.'
create_time: 2026-07-21 10:39:27
status: wip
---

# Plan: Race-free plan-lane epic clan summaries

## Context

Selecting an epic agent clan row in `sase ace` renders the declaring member's persisted `clan_summary` markup in the
clan detail panel (`build_clan_detail_text` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan.py`). Epic
launches declare `%clan(<epic>, tribe=epic, summary_script=sase_clan_summary_epic)` (built in
`src/sase/bead/work.py::_clan_identity_directives`), and two prior epics already tried to make that summary match the
rich PLAN lane that epic lander agents show in the SASE CONTEXT panel:

- **sase-85** added a bead-store refresh retry, stderr diagnostics on fallback, a 20s script timeout, and a rich
  bead-store rendering.
- **sase-8d** extracted the PLAN lane's rendering into the shared `src/sase/sdd/plan_display.py` module, added
  `summary_script=` argv support plus the generic `sase_clan_summary_plan` script, and rewrote
  `src/sase/scripts/sase_clan_summary_epic.py` to render the authored plan file first via `SASE_EPIC_PLAN_REF`, with
  multi-root candidate probing.

Both epics closed, yet a live epic clan launched afterwards still showed the literal fallback `EPIC <id>` in the panel.
The rendering itself is not the problem: running the current `sase_clan_summary_epic` by hand against that epic's
committed plan file produces exactly the desired PLAN-lane document (Title/Goal/Path rows, numbered `◆` phase blocks
with dependency metadata, size chips, and full descriptions). What keeps failing is _when and where_ the script runs.

## Root-cause analysis (verified against the live failed launch)

The complete evidence chain from the most recent failure, reconstructed from the declaring member's persisted
`agent_meta.json`, its runner output file, and the plans-sidecar reflogs:

1. **The summary script runs at the worst possible moment.** `extract_directives_and_write_meta`
   (`src/sase/axe/run_agent_directives.py`) resolves the clan summary during directive extraction, which
   `src/sase/axe/run_agent_runner.py` performs _before_ dependency waits, runner-slot admission, and
   `prepare_workspace_if_needed`. The subprocess runs with the claimed — but not yet prepared — workspace as its cwd.
2. **For a freshly approved epic, no probe-able checkout contains the plan yet.** The approval flow archives the plan
   into the plans sidecar and creates the epic beads seconds before `sase bead work` launches the clan. At extraction
   time in the failed run:
   - the claimed workspace's plans sidecar did not exist at all — the runner log shows it was _freshly cloned_ during
     workspace preparation, roughly fifteen seconds after the script ran;
   - the primary checkout's sidecar missed the plan by two seconds: the reflog shows the summary script's own
     `refresh_current_bead_store()` rebased it onto a fetch tip committed at 10:14:36, while the plan-archive commit
     landed at 10:14:38 — the refresh raced the approval flow's push and lost;
   - the bead-store retry raced the same push, so `project.show(<epic>)` failed too. Every candidate in
     `_plan_reference_candidates` plus the bead fallback missed, and the identity fallback was persisted permanently.
     This race is structural: epic launches _always_ happen moments after the artifacts they need were committed, so
     tightening the script's search (what both prior epics did) can never close the window.
3. **The diagnostics sase-85 added were destroyed, which is why the failures kept looking silent.**
   `resolve_clan_summary_script` (`src/sase/axe/clan_summary_script.py`) appends script stderr to the runner output file
   through a separate `"ab"` file descriptor. The runner's own stdout is a block-buffered, non-append descriptor
   positioned earlier in the same file; when its buffer later flushes, it overwrites the appended bytes. The failed
   run's meta contains the fallback, the env demonstrably held `SASE_EPIC_PLAN_REF` (its popped value was persisted as
   `epic_plan_ref`), the current plan-first script code was in effect — yet the output file contains no trace of the
   mandatory "Unable to load epic clan summary" stderr block. Diagnosis of prior attempts was impossible for exactly
   this reason.

## Design decisions

- **Keep the `summary_script=` architecture.** No `%clan` syntax changes, no hard-coded epic logic in the TUI, no
  literal-summary embedding (forks re-run the declaring member's directives and must re-resolve). The fix attacks the
  timing race at both ends instead.
- **Fix the source first (snapshot), then the execution point (re-resolution).** The launch flow that builds the epic
  segments has the approved plan file in hand — it just committed it. Handing its content to the summary script through
  a durable local snapshot removes every dependency on git fetch/push timing. Independently, re-running the script once
  the prepared workspace exists makes the machinery self-healing for _any_ clan summary script whose data lives in the
  workspace, and refreshes summaries after long dependency waits.
- **Never silent again.** Summary diagnostics move to a durable per-launch artifact file; the output-file append path
  stays but is no longer the only record. Runner stdio semantics (owned by the Rust spawn layer) are explicitly not
  touched.
- **No sase-core changes, no TUI changes, no golden churn.** `clan_summary` already flows through the agent-scan wire
  and re-renders on the panel's periodic refresh, and the shared plan renderer is untouched, so the existing clan-panel
  PNG goldens must keep passing unchanged. Documentation updates go to `docs/` only; `sase/memory/` and generated
  instruction shims are out of scope.

## Durable attempt-labeled clan summary diagnostics

All in the sase repo:

1. `src/sase/axe/clan_summary_script.py`: capture the summary subprocess's stderr to a private temporary file instead of
   handing it the shared agent-log descriptor. After the run completes (success, failure, timeout, or discovery miss),
   append the existing behavior's copy to the agent log _and_ write an attempt record to a new `clan_summary_stderr.log`
   inside the launch's artifacts directory (new `artifacts_dir` parameter threaded from the caller in
   `src/sase/axe/run_agent_directives.py`). Each record carries a header — timestamp, attempt label (see the
   re-resolution phase), resolved script argv, outcome (ok / not-found / timeout / exit-code / empty-output), and which
   `SASE_CLAN_*` / `SASE_EPIC_*` env keys were present (names only, never values) — followed by the captured stderr.
   Successful runs with empty stderr may skip the record to keep the artifact meaningful.
2. Include the stderr tail in the existing `log.warning` calls so axe logs show the cause inline.
3. Tests (`tests/test_axe_smoke_clan_summary.py` and the focused clan-summary runner suites): artifact written with
   outcome and stderr on script failure/timeout/fallback; no artifact (or an empty-run record policy, whichever is
   chosen) on clean success; env-key names listed without values; existing append-to-agent-log assertions updated to the
   new capture flow.

## Race-free epic plan snapshot at launch creation

All in the sase repo:

1. `src/sase/bead/cli_work_handler.py` (around the `epic_work_segment_env(plan, plan_ref=issue.design)` call): resolve
   the approved plan file from the bead project's own plans checkout — the same store the launch flow just read and
   committed — and copy it to a durable per-project location that workspace recycling can never clean, e.g.
   `~/.sase/projects/<key>/artifacts/epic-plans/<epic_id>.md` (implementer picks the exact layout; it must be stable
   across respawns and overwritten on each new launch of the same epic so resumes stay fresh). Snapshot failure is
   non-fatal: log a warning and launch without it.
2. `src/sase/bead/work.py`: `_bead_env` / `epic_work_segment_env` gain the snapshot's absolute path as a new env var
   (suggested `SASE_EPIC_PLAN_SNAPSHOT`), exported alongside `SASE_EPIC_PLAN_REF` to every segment.
3. `src/sase/axe/run_agent_directive_metadata.py`: `epic_work_metadata_from_env` consumes the new variable into agent
   metadata (e.g. `epic_plan_snapshot`), and `preserved_agent_metadata` preserves it across runner re-execs, matching
   the existing `epic_plan_ref` handling.
4. `src/sase/scripts/sase_clan_summary_epic.py`: `_plan_reference_candidates` appends the snapshot path (from its env
   var) after the existing cwd and primary-checkout candidates — live checkout copies win when present; the snapshot
   guarantees a hit when they race. The displayed `Path:` value remains the original `SASE_EPIC_PLAN_REF` so the panel
   shows the real repo-relative reference.
5. Tests: unit coverage for snapshot creation and overwrite-on-relaunch in the work handler; script tests where every
   checkout candidate is absent and only the snapshot exists (the real race shape) asserting the full PLAN-lane
   rendering; launch smoke coverage (`tests/test_bead/test_cli_work_epic_launch.py`,
   `tests/test_axe_smoke_clan_summary.py`) asserting the snapshot env var reaches summary subprocesses and survives
   metadata consumption; snapshot missing/unreadable falls through to the existing candidate chain.
6. Docs: extend the summary-script environment contract in `docs/agent_families.md` and `docs/xprompt.md` with the new
   variable.

## Post-preparation summary re-resolution in the runner

All in the sase repo:

1. Carry the re-resolution inputs out of extraction: `AgentInfo` (`src/sase/axe/run_agent_directives.py`) gains a small
   optional carrier (script value, clan name, generation, tribe) populated only for a declaring member with a
   `summary_script=`.
2. Environment reconstruction: a helper co-located with `epic_work_metadata_from_env`
   (`src/sase/axe/run_agent_directive_metadata.py`) rebuilds the popped epic env vars (`SASE_EPIC_PLAN_REF`,
   `SASE_EPIC_BEAD_ID`, `SASE_PHASE_BEAD_ID`, `SASE_EPIC_CLAN_TRIBE`, plus the new snapshot variable) from persisted
   agent metadata via the same name mapping, so the reverse mapping cannot drift. The re-run environment is current
   `os.environ` overlaid with these reconstructed values; this keeps the refreshed (re-exec) runner pass working even
   though the original launch env was consumed before the re-exec.
3. `src/sase/axe/run_agent_runner.py`: after `prepare_workspace_if_needed` and the linked-repo / sidecar preparation
   blocks — and before `claim_bead_for_agent_launch` / `capture_sdd_base_sha` — re-run `resolve_clan_summary_script` for
   the carrier with the prepared workspace as cwd and the attempt label distinguishing it from the extraction-time run.
   Last-success-wins: a non-empty result replaces `clan_summary` in the in-memory `agent_meta` _and_ on disk through the
   read-merge-write pattern `record_run_started_at` already uses (`src/sase/axe/run_agent_markers.py`); a `None` result
   keeps the extraction-time value. Updating the in-memory dict is mandatory so later `write_agent_meta` calls (e.g. the
   `sdd_base_sha` write) cannot resurrect the stale summary. The panel picks the change up on its normal scan refresh;
   no wire or TUI work.
4. Contract documentation (`docs/agent_families.md`, `docs/xprompt.md`): a clan summary script may now run more than
   once per launch — once at directive extraction for immediate display and once after workspace preparation — with the
   last successful output winning; scripts must remain read-only and idempotent, and both runs carry the same env
   contract and 20-second timeout.
5. End-to-end race-shaped coverage: a fakey-provider epic launch (`tests/test_bead/test_cli_work_epic_launch.py` /
   `tests/test_axe_smoke_clan_summary.py`) arranged so no plan or bead source is visible at extraction time but the
   prepared workspace (or snapshot) provides it afterwards, asserting: the finally persisted `clan_summary` contains the
   plan title, goal fragment, and every phase title (never the identity fallback); the diagnostics artifact recorded the
   failed first attempt with its label; and a control launch with everything available at extraction still resolves on
   the first attempt. Re-exec/refresh coverage: a runner pass with only preserved metadata still re-resolves correctly
   through the reconstructed environment.

## Testing and verification

- Every phase runs `just install` then the mandatory `just check` before finishing; no clan-panel PNG golden changes are
  expected anywhere (the renderer is untouched), so any golden diff is a regression to fix, not accept.
- Targeted suites per phase are listed above; the epic's land agent additionally re-runs the full end-to-end race-shaped
  tests to confirm the combined behavior.
- Post-land spot check (manual, for the user): launch any fresh epic via plan approval and confirm the clan panel shows
  the PLAN-lane document within one panel refresh after the declaring member's workspace preparation, and that
  `clan_summary_stderr.log` in the launch artifacts explains any fallback that ever appears again.

## Risks and constraints

- **Double execution of user summary scripts.** The contract change is documented; scripts are decorative and read-only
  by contract, and both runs stay bounded by the existing timeout and non-fatal guarantees. The extraction-time run is
  deliberately kept so waiting clans still show an early summary.
- **Snapshot staleness.** A snapshot taken at launch reflects the approved plan at that moment; the candidate order
  prefers live checkout copies, and clan summaries are launch-time-stable by design, so drift is bounded and invisible
  in practice.
- **Meta write interleaving.** The post-prep update must use the established read-merge-write helper and update the
  in-memory dict; the phase includes an explicit test that a later meta write does not restore the stale summary.
- **Runner stdio overwrite is worked around, not fixed.** Reordering or re-opening the Rust-owned output descriptor is
  out of scope; the artifact file is the durable diagnostic channel, and the agent-log append remains best-effort.
- **No memory-file edits.** `sase/memory/*.md`, `AGENTS.md`, and provider shims stay untouched; all documentation lands
  under `docs/`.
