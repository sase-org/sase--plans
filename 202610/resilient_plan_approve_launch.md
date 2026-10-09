---
tier: tale
title: Resilient coder launch for sase plan approve
goal:
  sase plan approve fails to launch a coder only when launching the plan on this machine
  is essentially impossible.
size: medium
decisions:
  relocation_scope:
    ask:
      Should every session-attach launch relocate off an occupied pinned workspace, not
      just plan coders?
    choices:
      plan_coders:
        Only plan-approve coders relocate; other session launches still fail, now naming
        the occupant
      all_session_launches:
        Every pinned session launch relocates; changes the 2026-08 single-shot pin
        contract
    default: plan_coders
    why: Generic session members may need the parent's checkout; plan coders never do
    answer: plan_coders
proposed_by: bbugyi200.athena.0z3
decided_by: auto
create_time: 2026-10-09 14:29:16
status: wip
---

# Resilient coder launch for `sase plan approve`

## Problem

`sase plan approve ~/.sase/plans/202610/ref_task_locator.md` failed twice for the same
plan: first on the original direct approval, then on the coder relaunch:

```
✗ Coder launch failed: Failed to claim an available workspace for gh_bobs-org__bob-cli after 6 attempts: Failed to claim workspace #11: workspace #11 is already claimed
  Launch it yourself:
    sase run '+bob-cli %model:@medium %id(code, session=bbugyi200.athena.bob-cli-5y.5) #coder(plan:202610/ref_task_locator.md)'
```

Root cause, verified on the host:

1. The direct and recovery paths build the coder prompt with
   `%id(code, session=<planner>)` (`compose_coder_prompt` in
   `src/sase/main/plan_direct_approval_prompt.py`). This happens whenever
   `resolve_placement` (`src/sase/main/plan_direct_approval_placement.py`) can attach to
   the planner's session.
2. At launch, `prepare_agent_session_attach_launch`
   (`src/sase/agent/_agent_session_attach_launch.py`) handles the case where the planner
   is not running: it pins the child to the planner's recorded workspace by setting
   `use_preallocated_workspace=True` with the planner's workspace number (#11 here).
3. The planner had released #11, and an unrelated live agent (`bob-cli-5x.4`, a
   `%w`-deferred epic phase worker) then claimed it.
4. `spawn_slot_with_workspace_retry` (`src/sase/agent/launch_executor_workspace.py`)
   catches the `WorkspaceClaimError` raised by the spawn claim callback
   (`src/sase/agent/launch_spawn.py`, "Failed to claim workspace #N"). It then retries
   the **same** pinned slot, because `_resolve_slot_workspace` returns the same
   `(num, dir)` every time. That is six identical, doomed attempts.
5. `_launch_coder` (`src/sase/main/plan_direct_approval_run.py`) has no fallback. The
   printed "Launch it yourself" command carries the same `%id(session=...)` pin, so it
   fails identically.

The live-gate path does not have this problem. When a PlanApproval gate turn launches
the coder, `launch_turn_followup` (`src/sase/turns/followup.py`, used by
`src/sase/gate_turn/followup.py`) walks a ladder: transfer the claim, take a fresh
claim, claim a pool workspace, then fall back to #0. The direct and recovery routes of
`sase plan approve` have no equivalent.

Goal: `sase plan approve` should fail to launch a coder only when launching the plan on
this machine is essentially impossible. Examples: the owner identity is not configured,
the project tag is unknown or disabled, the plan's model is blocked by a hard-disabled
provider, or the process cannot be spawned. The legitimate refusals stay unchanged:
invalid plan, coder already live, plan already implemented, run from inside an agent,
and epic guards.

## Design

### 1. Pinned workspace relocation in the launch layer

Add the opt-in env key `SASE_AGENT_PINNED_WORKSPACE_FALLBACK` with value `pool`. Define
the name and an `is_pinned_workspace_fallback_requested(env)` helper in
`src/sase/agent/launch_executor_workspace.py` and export both.

The key uses the `SASE_AGENT_` prefix on purpose. `scrub_agent_identity_env`
(`src/sase/agent/env_hygiene.py`) strips it from inherited env, so it reaches a child
only when that launch supplies it in `extra_env`, and a coder's own nested launches
never inherit it.

**Parent side** (`spawn_slot_with_workspace_retry`), when
`context.use_preallocated_workspace` is true:

- The first attempt claims the pinned slot exactly as today.
- On a `WorkspaceClaimError` for that slot (`exc.workspace_num` is the pinned number or
  `None`) **and** the fallback is requested in `extra_env`:
  - Do not sleep and do not retry the pin.
  - Switch the remaining attempts to the pool path: `_preclaim_axe_workspace`, the same
    path non-pinned launches use. Keep the existing pool retry/backoff and the
    preclaim-release `finally`.
  - Before spawning in the pool workspace, rewrite the session-attach payload in the
    request's env so `parent_workspace_dir` and `parent_workspace_num` name the
    relocated workspace. Load it with `load_agent_session_attach_plan_from_env`, apply
    `dataclasses.replace`, then re-encode with `agent_session_attach_env`. This mirrors
    the existing precedent in `spawn_agent_session_successor`
    (`src/sase/agent/detached_child.py`, which sets
    `parent_workspace_dir=workspace_dir`) and in `src/sase/monitor/followup.py`.
  - Record the relocation on the result. Add an optional field to `AgentLaunchResult`
    (`src/sase/agent/launch_types.py`): `workspace_relocation: str | None = None`. Set
    it to text like `pinned workspace #11 is claimed by <occupant>; launched in #13`.
- If the fallback is **not** requested, keep today's attempt count and backoff, so a pin
  released within a few seconds still succeeds. The final `WorkspaceClaimError` message
  must name the pinned slot and its occupant instead of the generic "Failed to claim an
  available workspace" text.

**Occupant description:** move `_describe_workspace_occupant`,
`_format_workspace_occupant`, and `_pid_is_alive` out of
`src/sase/axe/run_agent_phases.py` into a public
`describe_workspace_occupant(project_file, workspace_num, *, checkout_dir=None) -> str | None`
exported from `sase.running_field`. Keep the current format, which
`tests/test_axe_run_agent_runner_deferred_workspace_claim.py` asserts on. When
`checkout_dir` is given and `read_occupant_record(checkout_dir)`
(`src/sase/workspace_provider/occupant.py`) returns a record with a matching pid and an
`agent_name`, lead with that agent name (for example
`bob-cli-5x.4 (pid 105010, live, …)`). `run_agent_phases.py` then calls the public
helper. Do not import private names across modules, because Symvision flags private
misuse.

**Child side** (`claim_deferred_workspace` in `src/sase/axe/run_agent_phases.py`). This
covers a pinned target with a running parent; recovery can launch one while the planner
is still alive.

- When `_claim_pinned_deferred_workspace` fails and
  `is_pinned_workspace_fallback_requested(os.environ)` is true, fall back to
  `_claim_next_deferred_workspace` with the normal attempt limit.
- Print one runner-log line, for example
  `Pinned workspace #N is claimed by <occupant>; relocating to a pool workspace`.
- Without the key, keep the single-shot exit-1 behavior and its test unchanged.

> [!decision] relocation_scope = all_session_launches Drop the env key and the opt-in
> helper. Relocation becomes the default for every pinned session-attach launch, on both
> the parent side and the deferred child side. Update
> `test_pinned_target_held_by_live_pid_fails_with_occupant` to expect relocation. Also
> update the docstrings in `run_agent_phases.py` and the
> `fix(workspace): claim slots before materializing checkouts` contract they describe.
> The non-opted branch and its occupant-naming error then apply only to failures where
> relocation itself fails.

### 2. Coder launch ladder for direct approval and recovery

`src/sase/main/plan_direct_approval_run.py` is already 573 lines, so put the ladder in a
new module, `src/sase/main/plan_direct_approval_launch.py`. Both
`execute_direct_approval` and `execute_coder_recovery` call it in place of
`_launch_coder`.

- `launch_coder_once(prompt, local_plan) -> AgentLaunchResult` is the existing
  `_launch_coder` body. It calls `launch_agents_from_cwd` with
  `extra_env={"SASE_PLAN": str(local_plan), SASE_AGENT_PINNED_WORKSPACE_FALLBACK: "pool"}`
  and `origin="generated"`. Under `all_session_launches`, only `SASE_PLAN` is passed.
- `launch_coder_with_fallbacks(plan, local_plan, plan_argument, *, launch=launch_coder_once) -> CoderLaunch`
  returns a dataclass with these fields:
  - `coder: AgentLaunchResult | None`
  - `error: str | None`
  - `prompt: str`: the successful prompt, or the last one attempted
  - `placement: CoderPlacement`: the effective placement
  - `notes: tuple[str, ...]`

Ladder, in order:

1. **Precheck.** Call `require_agent_owner_identity()` (`src/sase/config/_owner.py`). If
   it fails, return a fatal error immediately. That is the only cheap way to tell "this
   machine cannot launch agents" apart from a placement problem.
2. **Planned placement.** Compose the prompt exactly as today: `compose_coder_prompt`
   with `plan.placement`, `plan.bead`, `plan.model_directive`,
   `plan.request.coder_prompt`, and `plan.request.wait`. Launch it. The env key above
   makes an occupied pin relocate instead of failing. On success, copy any
   `coder.workspace_relocation` into `notes`.
3. **Standalone.** This step runs only when step 2 used `mode == "session"` and failed
   with a non-fatal error. Build
   `CoderPlacement(mode="standalone", parent=plan.placement.parent, reason=f"agent session launch failed: {err}")`.
   Recompute the bead with `resolve_bead(plan.source_path, standalone, plan.project)`
   from `plan_direct_approval_placement.py`, recompose the prompt (no `%id(session=)`),
   and launch. Add a note:
   `could not join agent session <session> (<err>); launched standalone`.
4. **One transient retry.** If the last error is transient, sleep about 2 seconds and
   re-run the last placement once. Transient means `WorkspaceClaimError`,
   `TimeoutError`, `BlockingIOError`, `InterruptedError`, or a `GateError` whose `code`
   is `lock_timeout`.

**Fatal errors** stop the ladder at once, because another placement cannot help. They
are:

- the owner-identity precheck
- `ProjectTagError` (`src/sase/project_tags/tags.py`)
- `DisabledProviderLaunchError` (`src/sase/agent/launch_guard.py`)
- `TypedAdmissionRequiredError` (`src/sase/agent/launch_request_types.py`)
- `ProjectProviderMismatchError` (`src/sase/workspace_provider/utils.py`)

Import these types lazily. Every other `Exception` moves on to the next step. Never swap
the model: a hard-disabled provider is a deliberate host policy, and size aliases
already span providers.

`coder_error` is the last error's message. Also record each failed step's error in
`notes` (`session attempt failed: …`) so the card shows the whole ladder.

### 3. Outcome, receipt, and bookkeeping hardening

- `DirectApprovalOutcome` gains `placement: CoderPlacement | None = None`, the effective
  placement. `coder_prompt` becomes the ladder's `prompt`, and the ladder `notes` are
  appended to `warnings`.
- `_write_receipt` and `_write_recovery_receipt` derive `route` and `agent_session` from
  the effective placement. A standalone fallback records `route="standalone"` and
  `agent_session=None`.
- Bookkeeping must never turn a launched coder into a failure:
  - The post-launch receipt rewrite (both paths) catches `Exception` and appends a
    warning: `approval receipt could not be updated: …; the coder is running`.
  - `_record_planner_metadata` already catches errors; keep it that way.
- The pre-launch receipt writes (step 4 of `execute_direct_approval` and the first
  `_write_recovery_receipt` call) catch `OSError` and append a warning instead of
  aborting. The post-launch rewrite then retries the write.
- `execute_coder_recovery` holds the receipt `file_lock(..., timeout=30.0)`. Convert its
  `GateError` (`code == "lock_timeout"`) into
  `PlanApprovalActionError("approval_in_progress", plan.name, ...)`. The message should
  say that another `sase plan approve` for this plan holds the lock and to retry when it
  finishes. Today this case prints a raw traceback.
- In `_archive_plan`, wrap any exception from `archive_approved_plan` other than
  `PlanAlreadyArchivedError` and existing `PlanApprovalActionError`s as
  `PlanApprovalActionError("plan_archive_failed", ...)`. Nothing was launched or
  receipted at that point. The message keeps the original error and adds:
  `to launch the coder without archiving the plan: sase plan approve <path> -k approve`.
  Leave the credential preflight's `git_credential_denied` text as is, but append the
  same `-k approve` hint for the direct route.

### 4. Rendering (`src/sase/main/plan_approve_render.py`)

- `render_direct_approval` and `render_coder_recovery` take the route, the
  `no agent session: …` line, and the agent-session name from
  `outcome.placement or plan.placement`.
- The recovery card header becomes `✗ Coder relaunch failed · <name>` when `coder_error`
  is set. Today it prints `↻ Coder relaunched` even on failure.
- Relocation and fallback notes print as the existing yellow `! …` warning lines.
- Replace the failure footer (both cards) with:

  ```
  ✗ Coder launch failed: <error>
    Retry (re-runs every fallback and records the receipt):
      sase plan approve <local plan path>
    Or launch it yourself:
      sase run '<last attempted prompt>'
  ```

  Keep `_print_recovery_command` for the `sase run` line. Shell-quote the plan path in
  the retry line. Re-approval already takes the recovery route for a receipt with a
  coder error, so the retry command is valid.

- Exit codes are unchanged: 0 on success, 1 when `coder_error` is set.

### 5. Rust boundary

No sase-core changes. The pinned-slot pick, the executor retry loop, the session-attach
pin decision, and the gate follow-up ladder this mirrors are all Python today. The Rust
claim (`workspace_claims.rs`) keeps rejecting any existing claim. Relocation changes
which slot Python asks Rust to claim, not the claim semantics.

### 6. Docs

- `docs/cli.md`: in the `sase plan approve` paragraph (around the "starts a `#coder` in
  the planner's agent session" text), describe the launch ladder. Cover relocation when
  the planner's workspace is taken, the standalone fallback when the session cannot be
  joined, the single transient retry, and that only unlaunchable-machine errors fail the
  launch. Mention the retry command printed on failure.
- `docs/workspace.md`: next to the deferred/session workspace text (around "When a
  session follow-up's composed prompt..."), document pinned session workspaces. Cover
  `SASE_AGENT_PINNED_WORKSPACE_FALLBACK=pool` (or the always-on relocation under
  `all_session_launches`), the occupant-naming error, and that relocated children record
  the real workspace.

## Tests

- `tests/test_agent_launch_executor.py`:
  - **Pinned with opt-in:** the first spawn raises
    `WorkspaceClaimError(..., workspace_num=11)` for a `use_preallocated_workspace=True`
    context. Assert:
    - the second request uses the pool workspace (patch `claim_next_axe_workspace` and
      `get_workspace_directory_for_num`)
    - there is no backoff sleep before relocation
    - the session-attach env payload names the new workspace
    - `result.workspace_relocation` is set
  - **Pinned without opt-in:** today's attempt count is kept and the final error names
    the occupant (patch `describe_workspace_occupant`).
  - The existing pool-retry tests still pass.
- `tests/test_axe_run_agent_runner_deferred_workspace_claim.py`: an occupied pinned
  target with the env key claims a pool workspace and prints the relocation line. The
  existing occupant-failure test stays as is under `plan_coders`.
- A new `tests/test_plan_direct_approval_launch.py`, which patches
  `sase.agent.launch_cwd.launch_agents_from_cwd`:
  - session failure then standalone success: two calls, the second prompt has no
    `session=`, the effective placement is standalone, and there is a note
  - a fatal `ProjectTagError` makes exactly one call
  - an owner-identity precheck failure makes zero calls
  - a transient `WorkspaceClaimError` on the last step makes exactly one extra attempt
    (patch the sleep)
  - every attempt passes `SASE_PLAN` and the fallback env key
  - a relocation note is copied from the result
- `tests/test_plan_direct_approval_recovery_execute.py` and
  `tests/test_plan_direct_approval_run.py`:
  - Update `test_executor_launch_failure_records_error` and
    `test_coder_failure_records_error` for the ladder: a session placement now tries
    twice, and the receipt records the last error.
  - Add: a standalone fallback writes receipt `route == "standalone"` and
    `agent_session is None`.
  - Add: a post-launch receipt write failure keeps `outcome.coder` and adds a warning.
  - Add: a lock timeout raises the `approval_in_progress` action error.
  - Add: a non-refusal archive failure raises `plan_archive_failed` with the
    `-k approve` hint.
- `tests/test_plan_approve_render_recovery.py` and the direct-render tests: the failure
  header, the retry and `sase run` footer, the effective standalone route line, and the
  warning lines for relocation and fallback notes.
  `tests/test_plan_approve_render_handler.py` patches
  `sase.main.plan_direct_approval_run._launch_coder`; repoint it at the new seam and
  keep the exit-code assertions.

## Verification

Follow the `lint_and_test` memory note and run `just check`. Then do a manual smoke test
with fakes, not a real launch from inside an agent. With the launch seam patched to
raise the pinned `WorkspaceClaimError` once, render the recovery card through
`handle_plan_approve_command` to confirm the relocation and fallback warnings and the
new failure footer look right.
