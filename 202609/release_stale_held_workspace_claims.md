---
tier: tale
title: Release stale held workspace claims and fix the leaks that strand them
goal:
  Orphaned dead-PID workspace claims in the sase project are released, unclaimed
  ephemeral checkouts are deleted, and the runner/sweep are fixed so failed runs can no
  longer hold workspaces invisibly or accumulate holds without bound.
size: medium
proposed_by: bbugyi200.athena.06s
create_time: 2026-09-07 18:53:39
status: wip
---

# Plan: Release Stale Held Workspace Claims And Fix The Leaks That Strand Them

## Problem

`sase workspace list` for the sase project shows ~30 claimed workspaces while only 9
agents are running. Investigation (2026-09-07) found three distinct causes:

1. **Held (pinned) claims from undismissed failed runs — by design, but accumulating
   without bound.** A failed run (or a completed run with a deferred commit) pins its
   workspace claim via `hold_workspace_claim` (see `_should_hold_workspace` in
   `src/sase/axe/run_agent_runner_lifecycle.py`) so the user keeps a "visible dismissal
   path" in ACE; dismissing the run releases the claim. The scheduler sweep
   `cleanup_stale_running_entries` (`src/sase/ace/scheduler/stale_running_cleanup.py`)
   intentionally skips a pinned dead-PID claim while the run's artifacts directory still
   exists. Roughly 20 numbered workspaces plus 8 placeholder rows on workspace `#0`
   currently carry dead pinned claims for failed runs dating 2026-08-13 through
   2026-09-07 that were never dismissed.

2. **Bug: a run that crashes during finalize can hold a workspace while never writing
   `done.json`.** At least three current holds have an `error_report.md` but no
   `done.json` in their artifacts directory (artifact timestamps `20260815193021` →
   workspace #10, `20260907170541` → #32, `20260907151610` → #34). With a dead PID and
   no done marker, the ACE running-claims loader skips them (dead PID ⇒ no row) and the
   done-marker loaders skip them (no `done.json`), so they are invisible everywhere —
   there is _no_ dismissal path, and the sweep never reaps them because the artifacts
   directory still exists. These leak forever (#10 leaked since Aug 15). The observed
   crash causes were transient (a stale `.git/index.lock` in another checkout broke a
   plan-archive lease; a stale `sase_core_rs` wheel missing a binding; a commit
   finalizer failure) and have all since cleared — the durable defect is the marker gap,
   not those causes.

3. **Unclaimed checkout directories are not removed.** `sase workspace cleanup` requires
   `-s/--stale` and honors `workspace.cleanup_ttl_days` (14), so recently-touched
   unclaimed checkouts stay on disk indefinitely.

Additionally, workspace #23 is claimed by workflow `ace-gate` with a dead PID — that is
a _pending_ plan gate shell (`0h6--gate`, gate list shows it pending). A pending gate
intentionally keeps its dead creator PID in the RUNNING row; it must NOT be released. It
settles when the user answers or dismisses the gate.

Claims live in the `RUNNING:` field of
`~/.sase/projects/gh_sase-org__sase/ gh_sase-org__sase.sase`. Every mutation is audited
in `~/.sase/logs/workspace_claims.jsonl` (see
`src/sase/logs/workspace_claim_ledger.py`).

## Goals

1. One-time: release every orphaned (dead-PID, non-gate, non-monitor-pending) claim in
   the sase project, including pinned holds, and delete every unclaimed ephemeral
   checkout under the managed workspace root.
2. Durable fix A: a run that holds its workspace can never be invisible — guarantee a
   `done.json` exists whenever a hold is placed.
3. Durable fix B: the stale sweep reaps pinned dead claims that have no dismissal path
   (no `done.json`).
4. Durable fix C: pinned dead claims older than a configurable TTL are released
   automatically (artifacts kept), so ignored failed runs stop accumulating workspace
   holds without bound.

## Non-Goals / Out Of Scope

- Do not release or touch the pending-gate claim (`ace-gate` workflow) or any claim
  whose PID is alive. Do not touch monitor (`ace-monitor`) claims unless their monitor
  marker is terminal — reuse the existing `gate_claim_is_releasable` /
  `_monitor_claim_is_releasable` guards.
- Do not delete any run's artifacts directory; failure history stays inspectable via
  `sase agent show` and dismissible in ACE later (dismissal's workspace-release step
  becomes a no-op once the claim is gone).
- No new `sase workspace release`/`prune` CLI subcommand in this tale. If, while
  implementing, a supported operator CLI for this looks clearly worthwhile, file a task
  bead through the `/sase_new_task` skill instead of growing this change.
- Do not modify the primary checkout (workspace `#0`) or any `role=share` registry
  entry.

## Step 1 — One-time remediation: release orphaned claims

Work against the sase project file
`~/.sase/projects/gh_sase-org__sase/ gh_sase-org__sase.sase`. Enumerate claims at
execution time (state drifts constantly; the tables in this plan are a reference
snapshot, not the target list) using `sase.running_field.get_claimed_workspaces`.

Release a claim only when ALL of the following hold at execution time:

- its PID is not alive (`os.kill(pid, 0)` raising `ProcessLookupError`), AND
- it is not an `ace-gate` claim that `sase.gate_shell.claims.gate_claim_is_releasable`
  refuses, AND
- it is not an `ace-monitor` claim that
  `sase.ace.scheduler.stale_running_cleanup._monitor_claim_is_releasable` refuses.

Release via
`sase.running_field.release_workspace(project_file, num, workflow, cl_name, caller_tag="manual-stale-hold-cleanup")`
so the claim ledger audits each release. This covers pinned holds (including the `#0`
placeholder-row holds) that the sweep refuses by design. Expected scale from the
snapshot: ~20 numbered workspaces (#10–#28, #32–#34 range, excluding live ones) plus ~8
rows on `#0`; expect the pending-gate claim on #23 to remain.

After releasing, run `sase workspace list` and confirm the only remaining claims belong
to live PIDs plus the pending gate.

## Step 2 — One-time remediation: delete unclaimed ephemeral checkouts

For each numbered entry in `sase workspace list` for the sase project with `role=claim`,
an existing checkout directory, and no remaining claim:

1. Guard against a concurrent launcher claiming the number mid-deletion: first claim the
   workspace number yourself (the claim helpers in `src/sase/running_field/_claim.py`,
   own PID, a clearly-labeled workflow string such as `manual-cleanup`), and skip the
   workspace if the claim is refused.
2. Delete the checkout directory (`shutil.rmtree`).
3. Release your own claim.
4. Never delete the checkout the implementing agent itself is running in (its own claim
   is alive, so the live-PID rule already excludes it — keep it excluded even if tooling
   is confused about the current workspace number).

Finish with `sase workspace repair` (drops registry entries whose checkout is missing
and unclaimed) and verify `sase workspace list` shows only: the primary `#0`, checkouts
with live claims, and the pending-gate workspace #23.

## Step 3 — Fix A: a held workspace always has a done marker

In the runner finalize path (`finalize_runner_shutdown` /
`src/sase/axe/run_agent_runner_lifecycle.py`): when a hold is about to be placed via
`hold_workspace_claim` (and equally when `write_error_report` runs for a failed run),
check whether `<artifacts_dir>/done.json` exists; if not, write a failed done marker
using the existing `write_error_done_marker` / `build_done_marker` machinery
(`src/sase/axe/run_agent_runner_finalize.py`) with the error summary available in the
shutdown state. Today the marker is only written by `record_runner_error` for exceptions
caught inside `main()`'s inner try in `src/sase/axe/run_agent_runner.py`; exceptions
surfacing later (completion recording, gate settlement, finalize itself) produce an
`error_report.md` but no marker — that is exactly the invisible-hold leak. Note
`write_error_done_marker` swallows its own failures; keep the guarantee best-effort but
log loudly on failure.

Tests: extend `tests/test_run_agent_runner_lifecycle.py` — a shutdown that holds a
workspace with no pre-existing `done.json` must end with a failed `done.json` present.

## Step 4 — Fix B: the sweep reaps holds that have no dismissal path

In `cleanup_stale_running_entries` (`src/sase/ace/scheduler/stale_running_cleanup.py`),
the pinned branch currently skips the claim whenever `_held_agent_artifacts_exist(...)`.
Tighten it: a pinned dead-PID claim whose artifacts directory exists but contains no
`done.json` is releasable — without a done marker no UI surface can ever offer
dismissal, so the "conservative" skip preserves nothing. There is no finalizer race to
worry about: the only process that would still write the marker is the claim's own dead
PID.

Tests: extend `tests/test_stale_running_cleanup.py` for pinned-dead claims with
artifacts-with-marker (kept), artifacts-without-marker (released), and no-artifacts
(released, existing behavior).

## Step 5 — Fix C: TTL for held claims

Add config field `workspace.held_claim_ttl_days` (int, default `14`, `0` disables) next
to `cleanup_ttl_days` in `src/sase/workspace_provider/store.py`, with entries in
`src/sase/default_config.yml` and `src/sase/config/sase.schema.json`. In the sweep's
pinned branch, when the claim's PID is dead and its `artifacts_timestamp` parses to a
launch time older than the TTL, release the claim (artifacts are NOT deleted — the
failed run stays visible and dismissible in ACE; only the workspace hold is dropped,
meaning the failed run's dirty checkout may be recycled by the next claimant). Pinned
claims without a parseable `artifacts_timestamp` keep today's never-release behavior.

This is a permanent operator-tunable policy, so it is a config field, not a feature flag
(per the flags memory: values users may choose forever are config, not flags).

Tests: TTL boundary cases in `tests/test_stale_running_cleanup.py` (younger than TTL
kept, older released, `0` disables, missing timestamp kept).

## Step 6 — Verification and reporting

- Run `just install` first (ephemeral workspace clones may have stale venvs), then
  `just check` for the code changes; escalate to `just check-full` through the
  `/sase_monitor` skill per the lint-and-test memory if the scoped run escalates.
- Re-run `sase workspace list` and include before/after claim counts in the final
  response.
- Tell the user explicitly that workspace #23 stays claimed by the pending plan gate
  `0h6--gate` (~1 day old) until they answer or dismiss it (`sase gate list` / ACE).

## Reference snapshot (2026-09-07 ~18:40 ET)

Dead-PID pinned holds on numbered workspaces at investigation time: #10, #11, #13, #14,
#15, #16, #17, #18, #19, #20, #21, #22, #24, #25, #26, #27, #28, #32, #33, #34 (all
`ace(run)-*` workflows; launch stamps 2026-08-13 → 2026-09-07; sampled `done.json`
outcomes all `failed`; #10/#32/#34 have no `done.json` at all). Dead pinned rows on
`#0`: 8. Pending gate claim: #23 (`ace-gate`, stamp `20260906183859`). Unclaimed
existing checkouts at the time: #36, #41, #42, #43 (more will become unclaimed after
Step 1). Live claims at the time: #12, #29, #30, #31, #35, #37, #38, #39, #40.
