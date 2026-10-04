---
tier: tale
title: Unstick agent-session waits after monitor host-completion recovery
goal:
  Monitor host completion succeeds in the scrubbed supervisor environment, recovery
  follow-ups record their launch outcome so agent-session name waits resolve, and the
  currently stranded waiters (sase-1fv.4, sase-1eq.5.1.4,
  toobig-6y.test_xprompt_completion_spacer.0) start.
size: medium
proposed_by: bbugyi200.athena.0wb
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0wb](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0wb.md)
- **COMMITS:**
  - [56b6ee3](https://github.com/sase-org/sase/commit/56b6ee35d892c0ec059c890555e4c144ed9f517b)
    — fix(monitor): persist recovery follow-up outcome before finalization

# Unstick agent-session waits after monitor host-completion recovery

## Problem

Phase agent `sase-1fv.4` has sat in `WAITING` for hours although both phases it depends
on (`sase-1fv.1`, `sase-1fv.2`) are closed and landed. Its prompt carries both a name
wait (`%w:sase-1fv.1,sase-1fv.2`) and bead waits. The bead waits resolve, but the name
wait on the `sase-1fv.2` agent session never does, so the agent never starts. The same
defect currently strands two more chains:

- `sase-1fv.4` → `sase-1fv.5` → `sase-1fv.6` → `sase-1fv.land` (blocked on session
  `sase-1fv.2`, whose last member `sase-1fv.2--4` completed and landed `2608a2439e`)
- `sase-1eq.5.1.4` → `.5` → `.6` → `sase-1eq.5.1.land` (blocked on session
  `sase-1eq.5.1.3`)
- `toobig-6y.test_xprompt_completion_spacer.0` → `test_detach_scope.0` →
  `test_macro_terminology.0` (blocked on session
  `toobig-6y.test_ace_png_snapshots_memory_pane_history_states.0`)

### Root cause 1: recovery follow-ups never record their launch outcome

The wait index (`src/sase/core/wait_dependency_resolution/_index_entities.py`,
`_agent_session_members_after_shell_handoffs`) drops a failed turn member from a
session's effective generation only when:

- (a) its `turn_followup_agent` is present in the generation, or
- (b) a newer member of the same turn kind supersedes it
  (`_is_superseded_terminal_turn_member`).

`turn_followup_handoff_agent`
(`src/sase/core/wait_dependency_resolution/_artifact_state.py`) returns the follow-up
agent only when `monitor_followup_outcome` is in `SUCCESSFUL_TURN_FOLLOWUP_OUTCOMES`
(`launched`, `launched-degraded`).

The normal settlement path `settle_turn_claim_and_followup`
(`src/sase/turns/settlement.py`) records that outcome after launching a follow-up. The
host-completion recovery path does not. `recover_host_completion`
(`src/sase/monitor/host_completion_complete.py`) calls `launch_recovery` directly. Back
in `settle_claim_and_followup` (`src/sase/monitor/settlement.py`), only a
`host-completed` outcome is backfilled. So a recovered monitor member ends with
`monitor_followup_agent` set (written by `record_followup_launched`, which runs only on
a successful launch) and no `monitor_followup_outcome`.

Earlier recovered monitors are hidden by rule (b). The newest one is not, so it stays in
the effective generation with `is_resolved=False`. That session can never resolve, and
every `%w:<session>` waiter parks forever. Every recovery launch goes through the
`launch_and_capture` closure in `settle_claim_and_followup`. So fixing it there covers
every `recover_host_completion` call site in `host_completion_run.py` and
`host_completion_execute.py`.

### Root cause 2: host completion can never succeed in production

Every host-completion attempt in recorded history has failed: 38 attempts since
2026-09-24, zero successes. Each failed with
`sase final context requires active finalizer turn metadata: SASE_AGENT_TIMESTAMP` and
fell back to a recovery LLM turn, which is what triggers root cause 1.

- The monitor supervisor runs with a scrubbed environment: `_supervisor_env` in
  `src/sase/monitor/spawn.py` calls `scrub_agent_identity_env`, which removes every
  `SASE_AGENT_*` variable. That scrub is deliberate and must stay.
- `snapshot_execution_context` (`src/sase/monitor/host_completion_state.py`) calls
  `publish_final_context`. That reaches `_run_identity`
  (`src/sase/finalizers/declaration.py`), which requires `SASE_AGENT_TIMESTAMP`.
- Host completion binds that variable only later, in the submit path
  (`os.environ.setdefault("SASE_AGENT_TIMESTAMP", Path(artifacts_dir).name)` just before
  `submit_final_manifest`). The first snapshot therefore always fails.
- `SASE_AGENT_NAME` is not a problem: it falls back to the `agent_meta.json` name. The
  turn nonce is minted by `mint_finalizer_turn_nonce`.
- `tests/monitor/test_monitor_host_completion.py` hides the bug: its autouse
  `_sandbox_home` fixture sets `SASE_AGENT_TIMESTAMP` and `SASE_AGENT_NAME`, which
  production never has.

## Changes

### 1. Record the follow-up outcome for host-completion recovery launches

In `settle_claim_and_followup` (`src/sase/monitor/settlement.py`), after
`settle_host_completion` returns a non-`None` settlement, record the recovery launch's
disposition the same way `settle_turn_claim_and_followup` does for ordinary follow-ups,
whenever `monitor_followup_outcome` is not already set:

- If the launch result is `launched` and not `host_completed`:
  - record `launched`;
  - or record `launched-degraded` plus `monitor_followup_degraded_reason` when the
    result carries a `degraded_reason`.
- If a launch was attempted and failed (`launched=False`, not host-completed): record
  `not-launchable`. Without it, the monitor lane reads as permanently "Running" (it has
  a next action and no outcome). Keep the existing error handling, and do not overwrite
  an error that is already recorded.
- Keep the existing `host-completed` backfill unchanged.

Reuse the existing recorder rather than hand-writing fields. Promote
`_record_followup_outcome` in `src/sase/turns/settlement.py` to a public helper (for
example `record_turn_followup_outcome`), keeping one implementation for both paths. Call
it with `_MONITOR_SETTLEMENT_CONFIG` and `update_meta_field`, so both `meta` and
`agent_meta.json` are updated. `supervise.py` already copies `monitor_followup_outcome`
and `monitor_followup_degraded_reason` from `meta` into the done marker, so nothing else
needs to change there. Confirm this by test.

### 2. Bind finalizer turn identity before host completion's first finalizer call

In `src/sase/monitor/host_completion_state.py`, add one small helper that binds the host
finalizer's process identity for a monitor member:

- `os.environ.setdefault("SASE_AGENT_TIMESTAMP", Path(artifacts_dir).name)`
- `os.environ["SASE_ARTIFACTS_DIR"] = artifacts_dir`

Call it at the start of `snapshot_execution_context`, before `publish_final_context`.
Replace the inline pair in the submit path with the same helper, so the run id published
in the final context and the run id in the submitted envelope always agree.

Do not loosen `_run_identity`'s requirements, and do not stop scrubbing the supervisor
environment. Ambient identity must not leak across processes; that was the reason for
`292e9db152`. After this change, check `controller_context.py` and `controller_run.py`:
they also gate on `SASE_AGENT_TIMESTAMP`. Confirm that host completion now gets through
them, and not merely past the first snapshot.

### 3. Tests

- **Host completion with a scrubbed environment**
  (`tests/monitor/test_monitor_host_completion.py`): make the autouse fixture mirror the
  production supervisor. Delete `SASE_AGENT_TIMESTAMP` and `SASE_AGENT_NAME`
  (`monkeypatch.delenv(..., raising=False)`) instead of setting them, and fix any test
  that truly needs an explicit value by setting it locally. The existing `run-1` case
  already does this. Add or adjust an end-to-end eligible completion test asserting
  that:
  - host completion reaches the completed-by-host status rather than `recovery`;
  - `SASE_AGENT_TIMESTAMP` ends up bound to the monitor member's artifacts directory
    name.
- **Recovery outcome recording** (`tests/monitor/`, next to the existing settlement and
  host-completion tests): drive `settle_claim_and_followup` through a forced recovery
  and assert that `monitor_followup_outcome` is persisted to `agent_meta.json`:
  - `launched` for a plain launch;
  - `launched-degraded` plus the degraded reason for a degraded launch;
  - `not-launchable` for a failed launch. Also assert that the existing host-completed
    path still records `host-completed`.
- **Wait-index regression** (extend
  `tests/test_monitor_wait_dependency_supersession.py`, reusing
  `tests/_monitor_wait_dependency_helpers.py`): build an agent session in which:
  - the root member completed;
  - the newest monitor member failed, carries
    `monitor_host_completion_status: recovery`, `monitor_followup_agent` naming the next
    member, and `monitor_followup_outcome: launched`;
  - that follow-up member completed.

  Assert that a name wait on the session resolves. Add the paired case without
  `monitor_followup_outcome`, asserting it stays blocked. This documents why the
  write-side fix is required.

### 4. One-time repair of already-written records (host-local, not committed)

Data written before this fix will never gain the field, so repair it once. Use a short
throwaway Python snippet run with the repo's environment. This is deliberately not a
read-side compatibility branch, which would need a sunset flag.

1. Scan every artifacts directory under `~/.sase/projects/*/artifacts/ace-run/*/*/*/`.
2. Select each one whose `agent_meta.json` has:
   - `turn_kind == "monitor"`;
   - `monitor_host_completion_status == "recovery"`;
   - a non-empty `monitor_followup_agent`;
   - no `monitor_followup_outcome`.
3. For each selected directory:
   - set `monitor_followup_outcome` with `update_meta_field`
     (`sase.axe.run_agent_helpers_artifacts`). Use `launched-degraded` when
     `monitor_followup_degraded_reason` is present, otherwise `launched`;
   - load `done.json`, add the same key, and rewrite it with
     `write_done_marker_and_update_index` (`sase.axe.run_agent_exec_markers`), so the
     artifact index refreshes.
4. Print every repaired directory.

The repaired set should include `sase-1fv.2--mon-2`, `sase-1eq.5.1.3--mon-2`, and
`toobig-6y.test_ace_png_snapshots_memory_pane_history_states.0--mon-b`, plus older,
already-superseded recovery monitors (harmless to repair). The repair is data-only and
works with the code already loaded in the parked runners. Run it as soon as you start,
so the stranded chains resume while you work.

Then verify: within about two minutes (the runner's 60-second fallback or the
`wait_checks` chop), `sase agent list -j` should show `sase-1fv.4`, `sase-1eq.5.1.4`,
and `toobig-6y.test_xprompt_completion_spacer.0` leaving `WAITING`. `sase-1fv.land` and
`sase-1eq.5.1.land` stay parked only on their unfinished phases. Report the before and
after states in your final response. If any of them still waits, re-run the wait
diagnosis and report the remaining blocker instead of forcing it.

## Out of scope

- Gate-turn host completion and gate follow-up settlement. Only monitors use the
  host-completion recovery path.
- `sase-1fu.land` (blocked on `sase-1fu.4`): its newest member is still unfinished,
  which is a different cause.
- `sase-1eq.10` (blocked on bead `sase-1eq.5`, which is still open): working as
  intended.

## Verification

Run `just check` (inside `sase tool run`, per the guarded-recipe rule). It must pass,
apart from failures already known on master.
