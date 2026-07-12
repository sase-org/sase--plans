---
create_time: 2026-07-12 16:22:16
status: done
prompt: 202607/prompts/hold_failed_agent_workspaces.md
tier: tale
---
# Hold Failed-Agent Workspaces Until Dismissal

## Problem

When an axe agent fails, the runner still releases its workspace claim during shutdown. A failed agent frequently has
finished (or nearly finished) its work without committing, so its uncommitted file changes live only in the workspace
checkout. Because the claim is released immediately, the next agent launch can claim the same workspace number, and
workspace preparation (`prepare_workspace` in `src/sase/axe/runner_utils.py:147`, via `run_sase_hg_clean`) discards the
previous run's changes. Recovering that work is difficult (only the best-effort backup diff written by
`run_sase_hg_clean` remains). The `revert_agent_workspace.py` module docstring already calls out this hazard: "that
directory can be reclaimed by other agents and accumulate unrelated changes."

**Goal:** when an agent fails, keep its workspace claim (a "workspace hold") so no other agent can claim/clear the
workspace. Release the claim only when the user dismisses that failed agent from the Agents tab (`x` keymap and its bulk
variants).

## Current behavior (verified)

- **Unconditional release at shutdown.** `finalize_runner_shutdown`
  (`src/sase/axe/run_agent_runner_lifecycle.py:100-112`) is called from the `finally` block of `run_agent_runner.main()`
  and releases the workspace for every outcome — success, failure, and kill — guarded only by `is_home_mode`.
- **Failure classification.** `state.success` comes from `classify_exec_success`
  (`src/sase/axe/run_agent_runner_finalize.py`); genuine failures write a failed `done.json` (`write_error_done_marker`,
  outcome `"failed"`) plus an error report. Distinct non-failure outcomes that also reach finalize: `killed`,
  `failed_retried` (claim already transferred to the retry child via `transfer_workspace_claim`, which rewrites
  pid/workflow/timestamp on the claim row), and plan/questions handoffs (`plan_rejected` counts as success; SIGTERM
  handling already preserves the claim during pending handoffs).
- **Claim model.** Claims are rows in the project file's RUNNING field
  (`#N | PID | WORKFLOW | CL_NAME | TIMESTAMP [| PINNED]`), parsed/planned by the Rust core (`sase_core_rs`:
  `list_workspace_claims_from_content`, `plan_claim_workspace_from_content`, `release_workspace_from_content`, ...)
  behind `src/sase/running_field/_operations.py`. The `pinned` flag already exists end-to-end (wire, Rust round-trip,
  Python model).
- **Stale-claim reapers.** Dead-PID claims are released by:
  - `cleanup_stale_running_entries` (`src/sase/ace/scheduler/stale_running_cleanup.py:34`) — **skips pinned**;
  - Agents-tab loader `_release_stale_running_claim` (`src/sase/ace/tui/models/_loaders/_running_loaders.py:61-73`) —
    **skips pinned**, and never renders a row for a dead-PID claim (no phantom "running" entry);
  - `orphan_cleanup.py` (Reverted ChangeSpecs) — does **not** check pinned;
  - the kill flow's stale-state branch (`src/sase/agent/running.py:467-484`).
- **Dismissal today releases nothing for run-type agents.** The `x` action plans cleanup via the Rust
  `plan_agent_cleanup`; its dismiss path emits `workspace_release_requests` **only for workflow-type agents** (host
  executor: `persist_cleanup_side_effect_intents` in `src/sase/ace/tui/actions/agents/_dismiss_persistence.py:57-81`;
  Python fallback `_release_workspace_for` is likewise workflow-only). That was correct because the runner always
  released its own claim at exit — the assumption this plan changes.
- **Compatibility probe (empirical, against the installed `sase_core_rs`):**
  - A claim row with an **unknown trailing token (e.g. `FAILED`) is silently dropped** by the Rust parser — so
    introducing a new claim-state token would make held claims invisible to allocation for any component running an
    older core, causing exactly the double-claim we're preventing.
  - `PINNED` rows round-trip correctly; `release_workspace_from_content` removes pinned rows; composing release +
    re-claim(pinned=True) in memory yields a correct `... | PINNED` row.
- **Only current pinned setter** is `sase workspace git` setup (`src/sase/scripts/git_setup.py:53,69`); those claims
  carry no `artifacts_timestamp`, so they cannot be confused with agent holds.
- **Checkout GC safety.** `workspace_handler_maintenance.py` excludes claimed workspace numbers from checkout-dir GC, so
  a retained claim also protects the directory itself.

## Design

**Concept: a "workspace hold".** A hold is the failed agent's existing RUNNING-field claim kept in place and marked
`PINNED` (same workspace number, workflow, cl_name, pid, and artifacts timestamp). Pinning is what makes every existing
stale reaper leave it alone, and every existing allocator (Rust `allocate_and_claim_workspace_from_content`) already
treats any claim row as "taken". Reusing `PINNED` — rather than adding a new claim-state token — is deliberate: it is
understood by every deployed parser today, whereas unknown tokens are dropped (see probe above).

**Invariant:** a workspace hold exists if and only if the Agents tab shows a dismissable FAILED row for that run
(excluding `FAILED (RETRIED)`, whose claim was transferred to the retry child). Every hold therefore has exactly one
user-visible release path — dismissing that agent. All other outcomes (success, kill, retry handoff, plan/questions
handoff, auto-dismiss, home mode, hard crash) keep today's behavior.

**Identification:** a hold is recognizable as a claim row whose `artifacts_timestamp` (+ `cl_name`) matches a finished
agent run and whose PID is dead. No sidecar file is required; `done.json` already records
`workspace_num`/`workspace_dir` for the failed run.

## Implementation

### Phase 1 — `hold_workspace_claim` operation (this repo)

Add `hold_workspace_claim(project_file, workspace_num, workflow, cl_name, artifacts_timestamp)` to
`src/sase/running_field/_operations.py`: under one `changespec_lock`, compose the existing Rust content transforms —
`release_workspace_from_content` then `plan_claim_workspace_from_content` with `pinned=True` (preserving
pid/workflow/cl_name/timestamp) — and perform a single `write_changespec_atomic`. This is a thin adapter over existing
core primitives (verified to round-trip). Do not double-count claim/release telemetry; add a small dedicated metric
(e.g. `WORKSPACE_HOLDS`) if useful. Export from `sase.running_field`.

Optional core follow-up (not blocking): a dedicated `plan_hold_workspace_claim_from_content` in `sase-core` so the hold
is one plan instead of a composition.

### Phase 2 — Runner: hold instead of release on genuine failure

In `finalize_runner_shutdown` (`src/sase/axe/run_agent_runner_lifecycle.py`):

- Introduce an explicit predicate (e.g. `should_hold_workspace(state)`), true only for a genuine failure:
  `not state.success` and `state.exec_outcome` not in the kill/retry/handoff set (`killed`, `failed_retried`,
  `plan_rejected`, `plan_committed`, question-handoff analogs — verify the exact outcome tokens against `finalize_loop`
  during implementation), and `deps.was_killed()` is false. The predicate must align with "this run writes a failed
  `done.json` without a `retried_as_timestamp`" so the invariant above holds.
- When `SASE_AGENT_AUTO_DISMISS` is set, never hold (auto-dismiss is an immediate dismissal, so release as today).
- On hold: call `hold_workspace_claim(...)` instead of `release_workspace(...)`; log
  `Workspace #N held (failed run) — dismiss the agent in ace to release it` to the run output.
- Surface the hold to the user: include the held workspace number/directory in the error report (`write_error_report`
  already receives `workspace_dir`) and in the failure completion notification, with a one-line recovery hint
  (inspect/commit from the workspace, then dismiss).
- Leave the SIGTERM release handler (`run_agent_runner_signals.py`) unchanged (kills release; pending plan/questions
  handoffs already keep the claim).

### Phase 3 — Dismissal releases the hold

Per the Rust-core backend boundary rule, dismissal semantics live in the core cleanup planner, so the primary change is
in the sibling `sase-core` repo:

- **`sase-core`** (`agent_cleanup/planner.rs` + cleanup wire): for dismiss-mode items of run type (and workflow parents,
  in addition to the existing workflow release), emit a `workspace_release_requests` entry with a new lookup flavor —
  "resolve by artifacts timestamp" (mirroring the existing `lookup_workflow` pattern; the wire gains e.g.
  `lookup_timestamp: bool` plus the timestamp value). Additive change; release a new `sase-core-rs` 0.3.x and bump the
  floor in `pyproject.toml` (`sase-core-rs>=0.3.2,<0.4.0` today).
- **This repo, host executor** (`persist_cleanup_side_effect_intents` in `_dismiss_persistence.py`): resolve the new
  lookup flavor host-side — find claims in the RUNNING field where `artifacts_timestamp` matches the dismissed run (and
  `cl_name` matches when present) **and the claim PID is dead** — then `release_workspace(...)`. The dead-PID guard
  makes the release a no-op for live runs; the timestamp match protects retry children (transfer rewrites the claim's
  timestamp) and unrelated pinned claims (e.g. `workspace git` claims have no timestamp).
- **Fallback parity:** update the Python planner mirror (`_build_cleanup_side_effects` in
  `src/sase/ace/tui/actions/agent_cleanup_facade.py`) and the non-plan fallbacks (`persist_dismiss_side_effects` /
  `persist_bulk_dismiss_side_effects`) with the same release-by-timestamp step, factored into one shared helper (e.g.
  `release_held_workspace_claims(project_file, artifacts_timestamp, cl_name)`).
- All Agents-tab dismissal surfaces route through these persistence helpers (single `x`, bulk dismiss-all, kill-all's
  dismiss half, cleanup-panel modal, `%x` relaunch), so they inherit the release. Ordering inside the executor already
  releases workspaces before deleting artifact dirs — keep that ordering.

If decoupling from a `sase-core` release proves necessary, the same helper can ship host-side-only first (invoked per
dismissed agent alongside intent execution) with the planner extension as a fast-follow; the helper is written so it can
migrate into intent resolution without behavior change.

### Phase 4 — Reaper audit and leak backstop

- `orphan_cleanup.py`: skip pinned claims (consistency with the other reapers; a Reverted ChangeSpec should not silently
  destroy a hold).
- Verify no change needed (pinned already skipped): `stale_running_cleanup.py`,
  `_running_loaders._release_stale_running_claim`, and kill-flow stale cleanup (kill of a done agent returns "already
  completed" before touching claims).
- **Backstop against permanent leaks:** in `cleanup_stale_running_entries`, release a pinned dead-PID claim when it
  carries an `artifacts_timestamp` but its artifacts directory no longer exists (i.e. the agent was dismissed/deleted
  through a path that missed the release — mobile notification actions, older flows). Claims without a timestamp
  (workspace-git pins) are never touched.

### Phase 5 — Visibility polish (small, optional)

- The ChangeSpec detail panel already renders RUNNING claims (`changespec_detail.py:368-373`), so held claims stay
  visible with their `PINNED` marker.
- Agents tab: consider a small "holds workspace #N" hint on FAILED rows (workspace number is in `done.json`), and update
  the `?` help popup text for kill/dismiss to mention that dismissing a failed agent releases its held workspace.

### Explicitly out of scope

- Workflow-runner parity (CRS/fix-hook workflows release on failure in `run_workflow_runner.py:245` and
  `workflows_runner/completer.py`): same hazard, but dismissal release for workflows matches by workflow name today and
  needs timestamp-matched claims to avoid cross-run ambiguity. Propose as a follow-up ChangeSpec once this lands.
- Automatic recovery/commit of the held changes (the existing backup-diff behavior in `run_sase_hg_clean` remains the
  fallback for pre-existing losses).
- Hard runner crashes (no `finally`): claim is left unpinned with a dead PID and is reaped as today; making crash-path
  holds work would require pinning at claim time, which would hide live claims from stale cleanup.

## Test plan

- `running_field`: `hold_workspace_claim` round-trips (row keeps number/pid/workflow/cl/timestamp, gains PINNED; single
  atomic write; RUNNING field formatting preserved).
- Runner finalize: holds on genuine failure; releases on success, kill, `failed_retried`, handoff outcomes,
  auto-dismiss, and home mode; failure output/error report mentions the held workspace.
- Reapers: stale cleanup and Agents-tab loader leave holds alone and render no phantom rows; orphan cleanup skips
  pinned; backstop releases a hold whose artifacts dir is gone and never touches timestamp-less pinned claims.
- Dismissal: dismissing a FAILED agent releases exactly its hold (single, bulk, kill-all, cleanup-panel paths);
  dismissing a `FAILED (RETRIED)` parent does not release the child's claim; dismissing while another agent's live claim
  shares the ChangeSpec releases nothing else; allocation cannot grab a held workspace in between.
- End-to-end: fail an agent with uncommitted changes → verify a new launch claims a different workspace and the changes
  survive → dismiss → verify the claim is gone and the workspace is claimable again.

## Risks

- **Workspace-pool pressure:** many undismissed failed agents accumulate holds. Acceptable: the unified pool is ~990
  slots, the ChangeSpec panel shows holds, and dismissal (bulk `x` included) is the natural relief valve; the
  notification hint teaches the release path.
- **Missed-release leaks** via dismissal surfaces that bypass the shared helpers — mitigated by the Phase 4 backstop.
- **Version skew:** an older `sase ace` (without Phase 3) dismissing a failed agent won't release the hold; the backstop
  then reaps it once artifacts are deleted. No format change means no parse-level skew risk.
