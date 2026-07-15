---
tier: tale
goal: 'Scheduler-launched fix-hook and CRS agent runs succeed end-to-end (the commit
  finalizer runs, a proposal is created, and the run reports success), and any run
  that still fails records a specific, human-readable failure reason that the ACE
  TUI and notifications can display.

  '
create_time: 2026-07-15 13:21:48
status: wip
prompt: 202607/prompts/fix_hook_finalizer_env.md
---

# Plan: Make scheduler-launched fix-hook/CRS runs commit their work and report real failure reasons

## Background: the investigated failure and its true root cause

The user observed SASE agents "failing for some unknown reason" with no useful failure message in the TUI, and that
relaunching a failed agent from the TUI always succeeds. The suspicion was that agents intentionally fail when they
detect a sase version different from the launcher's version.

The investigation **denies that suspicion as a mechanism**: no code path makes an agent fail on a launcher-vs-runtime
sase version mismatch. The only update-awareness mechanism (`src/sase/axe/run_agent_runner_refresh.py`) re-execs the
runner after a dependency wait; it never fails a run. Two real, distinct bugs explain everything that was observed:

### Bug A (root trigger — already fixed on master, no work in this plan)

Commit `350c2a359` (2026-06-25) added `tools/validate_sase_core_rs_version` to the Justfile `_setup` recipe. In
hook-runner environments (spawned by the ACE scheduler without the `SASE_LINKED_REPO_SASE_CORE_DIR`-style env vars that
SASE-launched agents get), `sase_core_dir` fell back to `../sase-core`, which is no longer a checkout — it is now a
workspace-store container directory (holding `sase-core_10`, `sase-core_11`, …) with no `Cargo.toml`. Every `just lint`
/ `just test` hook therefore failed in 0–1 seconds with
`[validate_sase_core_rs_version] missing TOML file: ../sase-core/Cargo.toml`, which is what kept spawning fix-hook
agents in pairs (one per hook). This was fixed by `3b54a7bd1` (2026-07-12, treat a placeholder checkout as absent) and
hardened by `1114961b4` (2026-07-15, prefer the workspace-local checkout). Hook outputs under `~/.sase/hooks/202607/`
confirm hooks pass again since 2026-07-12 17:28. **No changes are planned for Bug A.**

### Bug B (still live — the subject of this plan)

Every scheduler-launched fix-hook run on record (39/39 between 2026-07-06 and 2026-07-12) exited 1 and was shown in the
TUI as a failed agent with **no error message**, even when the agent correctly fixed the underlying problem. The chain:

1. `src/sase/axe/fix_hook_runner.py` calls `invoke_agent(...)` without publishing the agent phase environment. In
   particular `SASE_AGENT_TIMESTAMP` is never set in that process.
2. The provider-neutral commit finalizer (`src/sase/llm_provider/commit_finalizer.py`) hard-skips when
   `SASE_AGENT_TIMESTAMP` is unset, writing `commit_finalizer_result.json` with
   `status=skipped, reason=outside_sase_agent`. Failed fix-hook artifact directories all contain exactly this marker.
3. Because the finalizer never runs, the agent is never instructed to commit (the fix_hook prompt explicitly forbids
   committing unless the finalizer asks), so no `commit_result.json` is written to `SASE_ARTIFACTS_DIR`.
4. The `#propose` post-step (`src/sase/xprompts/propose.yml`) then either reports `success=false` or is skipped
   entirely, so no proposal entry is appended, `proposal_id` stays `None`, and the runner exits 1.
5. Since no exception occurred, `error_summary` is `None`, so `done.json` gets `outcome=failed` with `error=null` — the
   TUI has literally nothing to display. This is the "no good failure message" symptom.

Relaunching from the TUI always works because relaunch re-submits the raw `#fix_hook(...)` xprompt through the normal
agent launch path (`src/sase/axe/run_agent_runner.py` → `run_execution_loop`), which calls `publish_phase_env`
(`src/sase/axe/run_agent_exec_markers.py`) before invoking the provider. With
`SASE_AGENT_TIMESTAMP`/`SASE_ARTIFACTS_DIR` published, the finalizer runs, the agent commits via `/sase_git_commit` →
`sase commit` (honoring `SASE_COMMIT_METHOD=create_proposal`), a proposal is created, and the run succeeds. Recent
successful ace-run artifacts show `commit_finalizer_result.json` with `status=finalized`/`clean`, confirming the two
paths differ only in this environment publication.

`src/sase/axe/crs_runner.py` + `src/sase/workflows/crs.py` (CrsWorkflow) share the exact same defect class:
`invoke_agent` without phase env, proposal extraction from the `#propose` step, exit 1 without `proposal_id`, and
`SASE_COMMIT_METHOD` only being exported after the agent has already run.

### Decision: no auto-relaunch

The user asked whether failed agents should be relaunched automatically since relaunching "works every time".
Relaunching only works because the relaunch path publishes the agent environment that the scheduler path forgot — it
re-runs an expensive agent to paper over a missing env var. This plan fixes the root cause instead, making the first
scheduler-launched run behave identically to a relaunch. Auto-relaunch machinery is explicitly out of scope.

## Design

### 1. Publish the agent phase environment in standalone axe runners

In `src/sase/axe/fix_hook_runner.py` and the CRS path (`src/sase/axe/crs_runner.py` / `src/sase/workflows/crs.py`):

- Before `invoke_agent` runs, publish the same phase environment the run-agent exec loop publishes: `SASE_ARTIFACTS_DIR`
  and `SASE_AGENT_TIMESTAMP` (reuse `publish_phase_env` from `src/sase/axe/run_agent_exec_markers.py` or a thin shared
  helper rather than duplicating the normalization logic).
- Move the commit-related exports that the finalizer and commit skill need — `SASE_COMMIT_METHOD`, `SASE_AGENT_CL_NAME`,
  `SASE_AGENT_PROJECT_FILE` — to before the `invoke_agent` call. Today `fix_hook_runner.py` exports them only after the
  agent returns (they were only ever consumed by post-steps), and `CrsWorkflow` exports `SASE_COMMIT_METHOD` after its
  invoke as well. Verify whether `#propose`'s `environment:` block already exports `SASE_COMMIT_METHOD` at expansion
  time; keep exactly one authoritative place and make the ordering explicit.
- Audit the behaviors gated on `SASE_AGENT_TIMESTAMP` (`src/sase/workflows/commit/checkpoint.py`,
  `src/sase/workflows/commit/commit_tracking.py`, `src/sase/llm_provider/_plan_utils.py`,
  `src/sase/axe/run_agent_helpers_questions.py`) and confirm each is correct or benign when the var is now set inside
  fix-hook/CRS runner processes. These runs are real agent runs with real artifact directories, so being treated as
  "inside a SASE agent" is the intended semantic.
- The finalizer resolves its workspace via `SASE_GIT_WORKSPACE_DIR` (set by the embedded `gh` VCS workflow's pre-steps)
  before falling back to cwd. The fix-hook runner process runs with cwd=`~`, so add a regression test asserting the
  finalizer inspects the claimed workspace, not the home directory. (The `gh` workflow lives in the sase-github plugin;
  open it via the `/sase_repo` skill if tracing is needed — do not read it another way.)
- `summarize_hook_runner.py` needs no commit and is intentionally untouched.

### 2. Record a real failure reason when no proposal is created

Even with the env fix, a fix-hook/CRS run can legitimately end without a proposal (agent made no changes, commit
rejected, propose step failed). Today that surfaces as `done.json` `outcome=failed, error=null` — invisible in the TUI.
Change both runners so that when the run completes without an exception but without a proposal:

- Synthesize a specific `error_summary`, incorporating the finalizer verdict from `commit_finalizer_result.json` (e.g.
  "agent completed but no proposal was created — commit finalizer: skipped (outside_sase_agent)" or "… finalizer: clean
  (no_changes)") and, when available, the propose step's outcome.
- Pass that summary through the existing `write_done_marker` / `write_error_report` / `notify_workflow_complete`
  plumbing so the Agents tab, error report action, and notifications all show the reason. No new UI is needed — the TUI
  already renders `error` from `done.json` and the ViewErrorReport action when an error report exists.

### 3. Tests

- Unit tests for the new env publication in both runners: with a stubbed provider/invoke, assert `SASE_AGENT_TIMESTAMP`,
  `SASE_ARTIFACTS_DIR`, `SASE_COMMIT_METHOD`, `SASE_AGENT_CL_NAME`, and `SASE_AGENT_PROJECT_FILE` are all set before the
  provider is invoked, and that the finalizer gate (`outside_sase_agent`) no longer triggers.
- A test that a completed-but-proposal-less run writes a `done.json` whose `error` field contains the synthesized reason
  (including the finalizer status), and writes an error report.
- A regression test covering finalizer workspace resolution from a runner whose cwd is not the workspace.
- Keep runtime-uniform: no provider-specific branches (all agent CLIs share the same finalizer/commit workflow).

## Verification

- `just check` (mandatory) and the focused test files for the axe runners.
- Manual end-to-end: seed a ChangeSpec with a deliberately failing hook (or invoke `fix_hook_runner.py` directly with a
  captured failing-hook output file), let the scheduler launch fix-hook, and confirm: the finalizer result is
  `finalized`, a proposal entry appears on the ChangeSpec, `done.json` has `outcome=completed`, and the Agents tab shows
  success. Then repeat with an agent that makes no changes and confirm the TUI shows the synthesized failure reason
  instead of a blank error.

## Risks and notes

- Setting `SASE_AGENT_TIMESTAMP` in these runners flips every "am I inside a SASE agent" gate to true for the whole
  process lifetime; the audit in Design §1 must confirm no gated behavior misfires (checkpointing, commit tracking,
  question routing).
- The finalizer may now prompt the agent for follow-up commit passes, lengthening fix-hook runs slightly; this is the
  intended behavior and matches what relaunched agents already do.
- This is Python frontend/runner glue, not shared domain logic, so it stays in this repo per the Rust core backend
  boundary; no `sase-core` changes.
- Bug A (hook 0-second failures) is already fixed on master; ChangeSpec branches created between 2026-06-25 and
  2026-07-12 still carry the broken Justfile until rebased, which can still spawn fix-hook agents — after this plan
  those agents will at least report precisely why they fail and will successfully propose whatever fix they make.
