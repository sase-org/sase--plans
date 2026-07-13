---
create_time: 2026-07-13 07:03:06
status: wip
prompt: 202607/prompts/fix_silent_failed_agent_workspace_leak.md
tier: tale
---
# Fix: Silent Failed-Agent Workspace Holds Leak Claims Forever

## Problem

Since 2026-07-12, sase workspace claims appear to never be released: a freshly launched agent ("7e") claimed workspace
#27 even though only three agents are running on the sase project. Workspace numbers keep climbing because the RUNNING
field of the project file accumulates dead claims.

## Root cause (diagnosed, with evidence)

Commit `4518dc19d` ("feat: hold failed agent workspaces until dismissal", 2026-07-12 16:43) is the proximate cause, but
the hold feature itself is not wrong — it has a visibility gap that a concurrently-broken hourly chop turned into an
unbounded leak. Three interacting facts:

1. **The hold predicate ignores whether the failure is user-visible.** `finalize_runner_shutdown`
   (`src/sase/axe/run_agent_runner_lifecycle.py`) now pins (`PINNED`) the workspace claim of every genuinely failed run
   (`_should_hold_workspace`: `not success`, outcome not in the kill/retry/handoff set, not killed, no auto-dismiss).
   The design invariant in the original plan (`~/.sase/plans/202607/hold_failed_agent_workspaces.md`) was _"every hold
   has exactly one user-visible release path — dismissing that agent"_, with the failure notification teaching that
   path. The predicate never checks that the notification will actually be sent.

2. **Scheduled maintenance chops fail silently.** The hourly axe chops (`pylimit_split`, `fix_just`, `refresh_docs`,
   `audit_recent_bugs` in `xprompts/`) consist entirely of `hidden: true` steps. When such a run fails,
   `all_steps_hidden()` is true, so `finalize_runner_shutdown` **suppresses the completion/failure notification
   entirely** (the `not deps.all_steps_hidden(...)` guard). The user is never told a hold exists, so the sole release
   path (dismissal in ace) is never exercised.

3. **The leak backstop cannot fire.** `cleanup_stale_running_entries`
   (`src/sase/ace/scheduler/stale_running_cleanup.py`) releases a pinned dead-PID claim only when its artifacts
   directory no longer exists. Artifacts are deleted at dismissal — which never happens for silent failures. Net effect:
   one pinned claim leaks permanently per silent failure.

**The amplifier:** the hourly `sase_pylimit_split` chop has failed on _every_ run since the pylimit→toobig migration
(`a66dc398a`), because the deployed sase uv tool environment lacks the `toobig` executable (`pyproject.toml` declares
`toobig>=0.1.0,<0.2.0`, but the tool env was never resynced — `.../uv/tools/sase/bin/toobig` does not exist). One pinned
workspace leaked per hour.

Evidence gathered:

- RUNNING field of `~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase.sase`: 15 `PINNED` claims (#12–#26), all with
  dead PIDs, at hourly cadence starting 260712_1648 — minutes after `4518dc19d` landed. The 5 non-pinned claims with
  live PIDs match the actually-running agents.
- `done.json` of each held run: `outcome: "failed"`, error `FileNotFoundError: ... /uv/tools/sase/bin/toobig` (14 ×
  `sase_pylimit_split-*`, 1 × `sase_fix_just`).
- `workflow_state.json` of the held runs: every executed step has `hidden: true` → `all_steps_hidden` → notification
  suppressed → nobody ever dismissed them → backstop never fires.

## Fix design

**Principle (restores the original invariant):** hold a failed run's workspace _only when the failure is actually
surfaced to the user_ — i.e. only when the failure completion notification will be sent. Silent failures (all steps
hidden, or notification suppressed) release the claim exactly as before `4518dc19d`. Automated hidden chop work has
nothing worth preserving that justifies an invisible, permanent claim; the run's backup-diff fallback still applies.

### 1. Gate the hold on failure visibility (`src/sase/axe/run_agent_runner_lifecycle.py`)

- In `finalize_runner_shutdown`, evaluate `steps_hidden = deps.all_steps_hidden(state.current_artifacts_dir)` once,
  _before_ the workspace claim decision (it is currently evaluated only later, in the notification guard — reuse the
  single computed value there so notification behavior is unchanged).
- Extend `_should_hold_workspace` so it returns False when `steps_hidden` is true or
  `state.suppress_completion_notification` is true. These are precisely the conditions under which no failure
  notification is delivered (`killed` and auto-dismiss are already excluded).
- Deliberate scope choice: do **not** gate on `agent_hidden` (`%hide` directive). Hidden-by-directive agents still
  deliver a (silent) inbox notification with the held-workspace recovery note and have a dismissable row (revealed with
  `.`), so their holds retain a user-visible release path.
- Update the hold log line/comments to reflect the new rule.

### 2. Tests (`tests/test_run_agent_runner_lifecycle.py`)

- New: failed run with all-hidden steps → workspace **released**, no hold.
- New: failed run with `suppress_completion_notification=True` → released.
- New: failed run with visible steps → still held (existing behavior).
- Existing success/kill/retry/auto-dismiss/home-mode cases stay green.

### 3. One-off remediation of the live leak (operational step, no code)

Release the leaked pinned claims in `~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase.sase`: every claim that is
`PINNED` **and** carries an `artifacts_timestamp` **and** whose PID is dead. The dead-PID guard protects live claims;
the timestamp guard protects intentional timestamp-less pins created by `sase workspace git` setup. Use
`sase.running_field.release_workspace` (takes the changespec lock; idempotent) from a short one-off script. Expected
result: #12–#26 released; the remaining claims all have live PIDs. Leave the failed runs' artifact directories in place
(the rows stay inspectable in ace; dismissing them later is harmless once the claims are gone). Unpinned dead-PID claims
(e.g. #11) can be left to the regular stale reaper.

### 4. Stop the hourly chop failure itself (environment, coordinate with user)

Even after 1–3, the `sase_pylimit_split` chop still fails every hour (now without leaking). The deployed uv tool env
must gain the `toobig` entry point: run `uv tool upgrade sase` (or reinstall the tool install) — ideally while no agents
are mid-run since the runner processes use that interpreter — then verify `.../uv/tools/sase/bin/toobig` exists and the
next scheduled chop tick succeeds. This is a deployment action, not a repo change; flag it to the user rather than
performing it mid-implementation if agents are active.

### Out of scope (possible follow-ups)

- Supersede semantics for recurring chops (releasing the previous run's hold when the same chop launches again) — would
  bound accumulation from _visible_ recurring failures too.
- Surfacing repeated silent chop failures (e.g. via the existing chop error digest script) so a broken hourly chop can't
  go unnoticed for a day again.

## Verification

- `just install`, then `just check` (lint + tests) before finishing, per repo policy.
- Targeted: `pytest tests/test_run_agent_runner_lifecycle.py tests/test_stale_running_cleanup.py`.
- After remediation: RUNNING field contains only live-PID claims (plus intentional pins); launching a new agent claims a
  low workspace number again instead of #29+.
