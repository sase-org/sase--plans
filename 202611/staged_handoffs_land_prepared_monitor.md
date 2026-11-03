---
tier: tale
size: medium
title: Finish the staged-handoffs landing with prepared-monitor completion fixes
goal:
  A prepared (-f) monitor started by a bare root agent completes commit-on-green under
  staged handoffs, the four lost task-backlog plan statuses read done, and epic sase-1pj
  is closed with a clean Symvision whitelist and its plan marked done.
proposed_by: bbugyi200.athena.sase-1pj.land
proposed_with: claude/opus
bead: sase-1pj
status: done
---

- **PARENT:**
  [202610/staged_handoffs.md](https://github.com/sase-org/sase--plans/blob/main/202610/staged_handoffs.md)
- **BEAD:**
  [sase-1pj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1pj.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1pj.land.md)
- **COMMITS:**
  - [e44819a](https://github.com/sase-org/sase--plans/commit/e44819a02664d7d46d032683a5163c59edd7e3a9)
    — docs(plans): mark staged handoffs complete

# Finish the staged-handoffs landing with prepared-monitor completion fixes

## Context

Epic **sase-1pj** ("Staged handoffs end through host completion",
`plan:202610/staged_handoffs.md`) is fully implemented. Its nine phases and all
descendant beads are closed, and `sase bead epic-symbols sase-1pj` is empty. Its land
agent verified the phases, notes, commits, and post-start integration. That triage is
recorded verbatim in the `LAND TRIAGE (sase-1pj.land, …)` note on sase-1pj. Read it with
`sase bead read sase-1pj -r "<why>"`; do not redo it.

Two pieces of remaining **epic-caused** work block the close, plus a small repair of
historical work loss. This tale finishes them, then closes the epic. Nothing else
resumes this landing, so the final step must be the closeout.

Do not refile the follow-ups already triaged. The PR-mode follow-up is task `sase-1r3`.
`sase-1qm` (`stale_worktree_fingerprint` refusal) is a related but distinct ready bug:
do not fix it here. If your investigation shows it shares a root cause with the defects
below, add evidence to it with `sase bead note sase-1qm "..."`.

## Defect A — prepared starter coverage breaks after stage-time root promotion

Epic note #5 records a reproduction on athena on 2026-11-03 (agent sase-1pj.8.1.7,
monitor `n7exmbcn38dr`). The failure chain:

1. `sase final prepare` seals `seal.creator.agent_name = context.agent_id`
   (`src/sase/finalizers/_prepare_completion.py`, around line 66). The starter is a bare
   root `X`, so the sealed creator is `X`.
2. `sase monitor start -f` stages. For a root that is not yet an agent-session member,
   `resolve_lane_start` (`src/sase/monitor/start_lane.py`, around line 93) calls
   `promote_agent_to_agent_session`. That renames `agent_meta.json` `name` to `X--0`, or
   to the plan suffix for plan roles (the athena case became `sase-1pj.8.1.7--plan`).
3. The staged protocol then requires `/sase_final`. `build_live_context`
   (`src/sase/finalizers/_declaration_context.py`) resolves `agent_id` metadata-first
   through `resolve_local_agent_name`, so it gets `X--0`. It passes that value as
   `starter_agent_id` to `evaluate_prepared_starter_coverage_for_artifacts`
   (`src/sase/finalizers/prepared_coverage.py`).
4. sase-core's `evaluate_prepared_starter_coverage`
   (`crates/sase_core/src/continuation/prepared_coverage.rs`) compares
   `intent.seal.creator.agent_name != request.starter_agent_id`. It refuses with
   `prepared_creator_agent_mismatch` even though the run id, turn nonce, obligation
   digest, and plan digest are unchanged. The starter must then commit, which defeats
   commit-on-green.

Before this epic, `-f` killed the runner and no `/sase_final` followed the start, so the
mismatch could not occur.

**Fix (Python adapter; no Rust or wire change):**

- Extract the exact-root-promotion rule that `_runner_ctx_from_monitor_metadata`
  (`src/sase/axe/run_agent_exec_staged_adopters.py`) already enforces into one shared
  helper. Put it next to the promotion code (for example in
  `src/sase/agent/_agent_session_promotion.py`, or `sase.plan_chain` if that fits
  better). Rewrite `_runner_ctx_from_monitor_metadata` to call it. The rule: the
  persisted name is `<base>--0` or `<base>` + `PLAN_CHAIN_PLAN_SUFFIX` with a canonical
  suffix, and `agent_session_base(name) == base`, `agent_session_value(meta) == base`,
  `workflow_name == base`, `agent_session_role == "root"`, and `role_suffix == suffix`.
- In `evaluate_prepared_starter_coverage_for_artifacts`, present the sealed creator name
  as `starter_agent_id` only when both of these hold:
  - the intent's `seal.creator.run_id` equals the current `run_id`;
  - the current artifacts' `agent_meta.json` is the exact root promotion of
    `seal.creator.agent_name` under the shared helper.

  Otherwise pass the resolved `agent_id` unchanged. The Rust predicate stays the
  authority for every other check (run, workspace, command, reservation or binding, plan
  digest, worktree fingerprint, coverage). Any other rename must still refuse.

- Why Python and not a new Rust field: the agent-session promotion rules already live in
  Python (`_agent_session_promotion.py`, `plan_chain`, the adopter check). The Rust
  coverage predicate keeps its contract, so no core pin move is needed.

**Tests:**

- Positive: a bare root seals a prepared intent. Then drive the real `-f` staging
  promotion, preferably through the staged-start fixtures in
  `tests/monitor/test_monitor_start_stage_prepared.py`, or else by calling
  `promote_agent_to_agent_session` exactly as `resolve_lane_start` does. Then
  `publish_final_context` / `build_live_context` reports prepared coverage eligible, and
  the covered repositories need no starter submission.
- Negative: a rename that is not the exact root promotion (for example `X--1`, a
  different base, or a non-root role) still refuses with
  `prepared_creator_agent_mismatch`. A different run id still refuses with
  `prepared_creator_run_mismatch`.
- `_runner_ctx_from_monitor_metadata` behaviour is unchanged. The existing staged
  adopter promotion tests still pass.

## Defect B — staged-spawned host completion lacks a finalizer turn nonce

When the monitor finishes, host completion refuses with
`sase final context requires active finalizer turn metadata: SASE_FINAL_TURN_NONCE`
(raised by `_run_identity` in `src/sase/finalizers/_declaration_context.py`). This also
reproduced on athena for monitor `n7exmbcn38dr` after all 19 `just check` stages passed.

The chain:

1. The monitor adopter now spawns the supervisor from the runner after finalizers
   settle.
2. By then `_invoke` (`src/sase/llm_provider/_invoke.py`, around line 472) has restored
   the pre-turn value of `SASE_FINAL_TURN_NONCE`, which means it popped it.
3. `_supervisor_env` (`src/sase/monitor/spawn.py`) copies the runner env, so the
   supervisor has no nonce.
4. `run_host_completion` (`src/sase/monitor/host_completion_run.py`, around line 178)
   calls `snapshot_execution_context` → `publish_final_context` → `_run_identity` before
   `execute_host_completion` mints its own nonce
   (`src/sase/monitor/host_completion_execute.py`, around line 136).

Kill-era starts ran the spawn from the agent's shell, so the supervisor inherited the
starter's nonce. `scrub_agent_identity_env` never removes it. Existing unit tests patch
`_snapshot_execution_context` / `publish_final_context` or set the env var, which masks
the gap.

**Fix:** host completion is its own no-model finalizer turn, so it must have a nonce
before its first context publication. In `_bind_finalizer_process_identity`
(`src/sase/monitor/host_completion_state.py`), mint one with
`mint_finalizer_turn_nonce()` when `SASE_FINAL_TURN_NONCE` is unset. Never overwrite an
existing nonce, and keep the fresh mint before `pre_exec` in `execute_host_completion`.

Then confirm that every host-completion and recovery path that publishes a context
reaches this binding first: `run_host_completion`, `_publication_for_prepared_install`,
`recover_host_completion`, and `host_completion_settle`. `execution_context_drifted`
compares only plan digest, observation fingerprint, and obligation ids, so a nonce
change between snapshots cannot register as drift. Keep it that way.

**Tests:**

- With `SASE_FINAL_TURN_NONCE` removed (`monkeypatch.delenv`) and without patching
  `_snapshot_execution_context` or `publish_final_context`, a green prepared intent
  completes through `run_host_completion` / `settle`. Reuse the
  `tests/monitor/ _host_completion.py` helpers and the
  `tests/test_finalizers_prepared_bridge.py` fixtures.
- A pre-set nonce is preserved, not overwritten, by the binding.

## Combined regression

Add one chained regression in `tests/test_finalizers_prepared_bridge.py` or a sibling
module. A bare starter prepares, stages `-f` with real promotion, and its `/sase_final`
context shows prepared coverage. The starter's commit finalizer leaves the covered repos
uncommitted. Monitor adoption binds the intent, and host completion then runs in a
nonce-free environment and commits on green with no model call. A unit-level chain that
stubs only process spawning is acceptable. Do not weaken any existing assertion.

## Historical plan-status repair (epic note #4)

Under the retired kill path, four landers' `status: done` edits to their epic plan files
were lost. Their epics are closed, but the plan files in the plans repo still say
`status: wip`. Resolve the plans checkout with `sase repo path plans` (or
`$SASE_SDD_PLANS_DIR`). For each file below, first confirm the bead is closed with
`sase bead read <id> -r "<why>"`. Then change only the frontmatter `status: wip` to
`status: done`:

- `202610/task_backlog_epic_1.md` (bead `sase-1pm`)
- `202610/task_backlog_epic_2.md` (bead `sase-1pn`)
- `202610/task_backlog_epic_3.md` (bead `sase-1pu`)
- `202611/task_backlog_epic_4.md` (bead `sase-1qn`)

Do not try to reconstruct other lost edits; `sase-1qv` owns re-landing the sase-1pn
repairs.

## Verification

1. Read `lint_and_test.md` with `sase memory read lint_and_test.md -r "<why>"` and
   follow it.
2. Run the focused suites while you iterate:
   - `tests/test_finalizers_prepared_bridge.py`
   - `tests/monitor/test_monitor_host_completion*.py`
   - `tests/monitor/test_monitor_start_stage_prepared.py`
   - the staged-adopter promotion tests that cover `_runner_ctx_from_monitor_metadata`
3. Run `just check` as `sase tool run check`. Do not run `just check-full`. Record the
   ToolRun id.

## Final step — close epic sase-1pj (must be the last step)

1. Run `sase bead epic-symbols sase-1pj`. It is empty today. For any entry that appears,
   resolve the symbol under the Symvision epic-whitelist policy (wire it, privatize it,
   add a non-test pragma, or delete it). Re-key a Justfile row only to a still-open
   later bead that needs it.
2. Close the epic normally. Never use `--force`. Keep the note's substance and fill in
   the placeholders:

   ```bash
   sase bead close sase-1pj --note "Landed. Verified all 9 phases and 51 descendants closed with every note addressed (see LAND TRIAGE note): plan/questions/pipe/gate/monitor staging, carry wire, adoption recovery, carry hardening, kill-path and flag retirement (sase-1pk closed), docs/memory per DECISIONS. Integration with post-start commits (agent-hijack, lost-gate withdrawal, flag-removal gates, readout/runner/monitor splits) was clean. Follow-ups: PR mode -> sase-1r3; core-note authorization resolved by 09a2273dbb; epic notes 1-3 resolved on master; note 4 kill path retired, sase-1qv owns sase-1pn re-landing, and task_backlog_epic_1..4.md statuses were repaired; note 5 fixed here: prepared starter coverage accepts the exact stage-time root promotion, and host completion mints its own turn nonce before its first context. sase-1qm stays distinct. Check: <ToolRun id>. epic-symbols: none."
   ```

   If the close is refused for leftover epic-symbol rows, clean them up and close again.
   If it is refused for any other reason, fix the cause rather than forcing.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file
   `202610/ staged_handoffs.md` in the plans repo. This is the PLAN path that
   `sase bead read sase-1pj` shows.
5. sase-1pj has no `parent_bead`, so nothing above it needs closing.
