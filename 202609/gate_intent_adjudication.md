---
tier: tale
size: medium
title: Gate-creation intent marker and host adjudication
goal:
  An agent-side gate creation that dies before handing off leaves a durable intent
  marker, and the host fails the run loudly, naming the lost gate kind and request id,
  instead of reporting a silent SUCCESS.
proposed_by: bbugyi200.athena.sase-11t.1
bead: sase-11t.1
create_time: 2026-09-16 10:58:18
status: wip
---

- **PARENT:**
  [202609/sudo_gate_crash_safe_handoff.md](https://github.com/sase-org/sase--plans/blob/main/202609/sudo_gate_crash_safe_handoff.md)
- **BEAD:**
  [sase-11t.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11t/sase-11t.1.md)

# Plan: Gate-Creation Intent Marker and Host Adjudication (sase-11t.1)

Phase `gate-intent-adjudication` of epic `sase-11t` (plan
`plan:202609/sudo_gate_crash_safe_handoff.md`). Sibling phases `sase-11t.2` (codex
turn-integrity, `src/sase/llm_provider/codex.py`) and `sase-11t.3` (sudo skill template)
are running in parallel and touch different files. Do not edit their files.

## Problem

An in-agent gate creation (`sase sudo request`, `sase gate create --shell`, agent-side
LaunchApproval, agent-side workflow HITL) leaves nothing on disk until
`create_gate_shell` (`src/sase/gate_shell/transaction.py`) calls
`create_gate_shell_member`. If the provider harness kills the CLI before that write, or
at any point before `maybe_handoff_gate_from_agent` writes `.sase_gate_pending`, the
host cannot tell that a gate was ever intended. The provider turn ends normally,
finalizers run, and the run is published as SUCCESS with no gate, no bundle, and no
notification. That is what happened to run `ace-run/202609/16/20260916100310`: codex
killed the `sase sudo request` process at teardown after 17s.

## Goal

Every agent-side gate creation writes a durable **intent marker** before any slow work.
The marker is cleared on every clean exit. If the provider turn ends and a marker is
still there, with no pending handoff to replace it, the host fails the run. It shows the
reason in the workflow output, in `done.json`, and in the failed-run notification (which
opens the error report and holds the workspace), and names the lost gate kind and
request id. The retry path never retries the run.

No feature flag: this fixes a bug. A silent false SUCCESS becomes a loud failure, and
normal runs behave exactly as before. Nothing here crosses the Rust core boundary. It is
host orchestration only, and no other frontend needs to share it.

## Design

### 1. Marker I/O module: `src/sase/agent/gate_intent.py` (new)

Put this module next to `pending_handoff.py` / `pending_handoff_write.py`. It may import
only the stdlib plus `sase.core.process_identity`, so it stays cheap for both the CLI
and the runner. Put it under `sase.agent`, not `sase.gate_shell`: `sase.llm_provider`
already imports `sase.agent.pending_handoff` at module top, so this location adds no
import cycle.

- **One file per creating process:** `<artifacts_dir>/.sase_gate_intent.<pid>.json`. Two
  concurrent gate commands from the same agent then never overwrite each other's marker.
  Export the prefix constant and a glob helper.
- **Payload:** `kind`, `request_id` (may be `null` until known), `source` (a front-door
  label such as `"sase sudo request"`, default `"create_gate_shell"`), `pid`,
  `process_identity` (`process_identity_token(pid)`), and `timestamp` (`time.time()`).
  Write it atomically with fsync, like `_atomic_write_json` in
  `pending_handoff_write.py`. Reuse or share that helper rather than copying it if that
  stays clean.
- **Where the marker is written:** `intent_artifacts_dir(env=None) -> str | None`
  returns the directory only when `SASE_AGENT` is truthy, `SASE_ARTIFACTS_DIR` is set,
  and the current process is **not** the runner. A process is the runner when
  `agent_meta.json["pid"] == os.getpid()`; if the meta can't be read, the process counts
  as agent-side.
  - Why the pid check: the runner process itself has `SASE_AGENT=1` and
    `SASE_ARTIFACTS_DIR` set. It creates plan/question gate shells host-side
    (`run_agent_exec_plan.py`, `run_agent_exec_questions.py` via `plan_shell/create.py`
    and `question_shell/create.py`) and never calls `maybe_handoff_gate_from_agent`.
    Writing a marker there would cause a false failure.
- **API.** Exact names are flexible; keep the behavior.
  - `begin_gate_intent(kind, *, request_id=None, source=None) -> Path | None`: writes or
    re-stamps this process's marker. Re-stamping is how the sudo front door's early
    marker (request id still `null`) picks up the request id once `create_gate_shell`
    knows it. Returns `None` and is a no-op when `intent_artifacts_dir()` is `None`.
    Write failures (`OSError`) are logged at debug level and swallowed: the marker is a
    safety net and must never block gate creation.
  - `clear_gate_intent(artifacts_dir=None) -> None`: removes **this process's** marker
    only. Idempotent, ignores missing files.
  - `list_gate_intents(artifacts_dir) -> list[GateIntent]`: tolerant of corrupt JSON. A
    corrupt file still counts as an intent with unknown fields.
  - `discard_gate_intents(artifacts_dir) -> None`: removes all markers (host side).

### 2. Agent-side lifecycle (write early, clear on every clean exit)

`create_gate_shell` (`src/sase/gate_shell/transaction.py`) is the shared path. Every
gate kind is covered there once.

- **Write:** call `begin_gate_intent(spec.kind, request_id=spec.request_id)` right after
  `_spec_from_request` and the existing shell/finalizer-owned-turn guards. The finalizer
  guard must still raise before any marker is written. The write must come **before**
  `_resolve_project_name`, `_resolve_creator`, the lane lock, and
  `create_gate_shell_member`. `spec.request_id` is always set by then.
- **Clean error:** if the body raises an `Exception` (`GateShellError`,
  `GateShellLaneError`, `GateError`, `OSError`, and so on), clear the marker and
  re-raise. The CLI reports the error to the agent, and the run continues normally. Do
  **not** clear on non-`Exception` `BaseException`s (such as `KeyboardInterrupt`): an
  interrupted creation must stay visible to the host.
- **Terminal result:** if the returned creation is terminal (auto-resolved, or a
  replayed terminal record), no handoff follows, so clear before returning.
- **Non-terminal result:** keep the marker. Ownership passes to the handoff.

`maybe_handoff_gate_from_agent` (`src/sase/gate_shell/agent_handoff.py`) settles the
marker:

- **Successful handoff:** clear the marker after `.sase_gate_pending` is durably written
  and **before** `kill_agent_runner_group`. Add an optional callback such as
  `on_marker_written: Callable[[], None] | None` to `maybe_handoff_shell_from_agent`
  (`src/sase/shells/handoff.py`), or split the write and the kill inside
  `maybe_handoff_gate_from_agent`. Either way, a crash between the two writes leaves
  both files, and the host treats the pending marker as superseding the intent.
- **`ShellHandoffError`:** clear the marker on both branches, whether or not
  `_compensate_failed_handoff` runs. This is a clean error that is reported to the
  caller.

Early write at the sudo front door (`_request` in `src/sase/sudo/cli.py`, the incident
path):

- Call `begin_gate_intent("sudo", source="sase sudo request")` before
  `_read_stdin_object()`, `build_sudo_gate_request`, and the lazy `sase.gate_shell`
  import. This also covers the stdin, spec, and import window.
- Wrap the body so any `Exception` (invalid JSON, `GateError`, and so on) clears the
  marker and re-raises. `create_gate_shell` then re-stamps the same per-pid file with
  the request id.
- The `SystemExit` from a successful handoff needs no extra handling, because the
  handoff already cleared the marker.
- Leave the other front doors (`src/sase/main/gate_handler.py`,
  `src/sase/agent/launch_request.py`, `src/sase/xprompt/workflow_hitl_gate.py`) alone:
  they are covered by `create_gate_shell` plus the handoff.

What deliberately leaves the marker in place: a SIGKILL or default-SIGTERM death, a
`KeyboardInterrupt`, or any exit after a successful non-terminal creation that never
reaches the handoff (for example, a `BrokenPipeError` while printing the descriptor, or
the LaunchApproval `invalid_state` checks). The host should flag each of these loudly,
because a gate shell may exist whose creator never handed off.

### 3. Host adjudication module: `src/sase/llm_provider/gate_intent_guard.py` (new)

- **Error type:** `GateIntentLostError(LLMInvocationError)` carries `kind`,
  `request_id`, `pid`, `source`, and the evidence path. Its message leads with a stable
  `gate intent lost:` prefix and names the gate kind and request id (`"(unassigned)"`
  when `null`). It says the creating process exited before the gate was created or
  handed off, and tells the user to check `sase gate list --all` (or
  `sase sudo list --all` for sudo) and re-run the request in the foreground. When the
  provider raised, the message also quotes that error (`provider error: ...`), so
  usage-limit detection in `_invoke`/`handle_workflow_error` still sees the original
  text.
  - Optional best-effort enrichment: look up
    `find_gate_shell_by_gate_id(None, request_id)` and report whether a member record
    exists and its state. Import it lazily and swallow every error.
- **Main entry:** `raise_if_gate_intent_lost(artifacts_dir, *, cause=None) -> None`.
  1. Return if `artifacts_dir` is falsy, no intents exist, or the runner has been
     signalled (`sase.axe.runner_signals.was_killed()`). A user kill or a real handoff
     SIGTERM is handled by the killed-iteration path. Checking first also avoids a grace
     wait on a user kill: Claude runs Bash tools in their own process group, so a `sase`
     CLI can outlive a runner-group SIGTERM.
  2. If `has_pending_handoff(artifacts_dir)`, discard the intents (they are superseded)
     and return.
  3. For each intent whose pid is still alive (`os.kill(pid, 0)` plus
     `process_identity_matches`), poll every 0.2s for up to `GATE_INTENT_GRACE_SECONDS`
     (module constant, 60.0). Stop early when that pid's marker disappears, the pid
     exits, or a pending handoff marker appears. Then go back to step 2. Tests
     monkeypatch the constant.
  4. Move each remaining marker to evidence `gate_intent_lost.json` in `artifacts_dir`
     (a list of payloads plus `adjudicated_at` and `pid_alive`), so a later check or
     retry never trips on it again. Then raise `GateIntentLostError` for the first
     (oldest) one, chained `from cause` when given. A pid still alive after the grace
     period is also lost; say so in the message.
- **Error-path helper:**
  `gate_intent_lost_error_for(exc, artifacts_dir) -> BaseException`. It returns `exc`
  unchanged when nothing is lost; otherwise it returns the chained
  `GateIntentLostError`. It lets the runner loop convert a provider or workflow failure
  without raising from inside the helper.
- **Chain helper:** `find_gate_intent_lost_error(exc) -> GateIntentLostError | None`
  walks `__cause__`/`__context__`. It is needed because
  `WorkflowExecutionError("Step '...' failed: ...")` wraps the original exception.

### 4. Host wiring

- **`src/sase/llm_provider/_invoke.py`:**
  - Import the guard at module top, not lazily. It must already be loaded before the
    turn so a mid-run `sase dev update` can't tear the import (see
    `preload_post_gate_modules`).
  - Call `raise_if_gate_intent_lost(artifacts_dir)` right after `provider.invoke(...)`
    returns and **before** `run_finalizers(...)`. A lost intent then never reaches
    declaration recovery, commit finalizers, `postprocess_success`, or reply
    publication.
  - The raised `GateIntentLostError` (an `LLMInvocationError` subclass) goes through the
    existing `except LLMInvocationError` branch: metrics, `postprocess_error` (so the
    reason appears in the workflow output and the error chat), and a bare re-raise.
    Leave the provider-raised path alone here; the runner loop handles it (below).
- **`src/sase/axe/run_agent_exec.py`, `_run_execution_loop_bound`:**
  - **Provider/workflow raised, runner not killed:** first replace `wf_exc` with
    `gate_intent_lost_error_for(wf_exc, state.current_artifacts_dir)`. Then call
    `handle_workflow_error`. On `"raise"`, raise the (possibly converted) exception so
    the chain is kept (`raise converted from wf_exc` only when it differs; otherwise
    keep the bare `raise`).
  - **Loop ended without a kill (backstop):** call
    `raise_if_gate_intent_lost(state.current_artifacts_dir)` before `_finalize_loop`.
    This catches gate commands run by non-`_invoke` workflow steps. Because it is
    outside the `try`, it propagates as a runner error.
  - **`_handle_killed_iteration`:** call `discard_gate_intents(...)` on both branches
    (user kill, and adopting or ignoring pending markers). A handoff supersedes the
    intent, and a kill ends the run anyway.
- **`src/sase/axe/run_agent_exec_retry.py`, `handle_workflow_error`:** after the
  existing continuation-delta persist and usage-limit second writer, and **before**
  retry classification: if `find_gate_intent_lost_error(exc)` is set, snapshot the
  attempt as `"raised"` with reason `"gate intent lost; retry skipped"` (only when that
  fits `snapshot_attempt`'s current contract; otherwise skip the snapshot) and return
  `"raise"`. A lost gate intent is never retried or sent to the fallback model, even
  when the provider text matches a retry pattern.
  - Rationale: retrying a turn whose gate creation was lost can hide the loss or create
    a second gate. The epic's goal is a loud failure. `sase-11t.2`'s aborted-turn retry
    still applies to aborted turns that had no gate intent.
- **`src/sase/axe/run_agent_runner.py`, `main`:** in the `except Exception` branch, when
  `find_gate_intent_lost_error(e)` is set, pass `error_summary=str(lost)` to
  `record_runner_error`. The failed `done.json` `error`, the error report, and the
  failed completion notification's notes then lead with the gate kind and request id
  instead of the `WorkflowExecutionError: Step ... failed:` wrapper. Everything else
  already works:
  - `success=False` gives a FAILED status line in the output log.
  - `exec_outcome` stays `""`, so `_should_hold_workspace` holds the workspace (a
    visible failed run).
  - `write_error_report` runs, and the notification uses the `ViewErrorReport` action.

### 5. Docs

In `docs/notifications.md` under "Gate shells and continuation", add a short paragraph.
It should say that agent-side gate creation writes a per-process intent marker, which
the handoff or a clean error clears. It should also say that a creation command that
dies before handing off makes the host fail the run with a `gate intent lost:` error
naming the kind and request id, and that such a run is never retried. Run `just fmt` for
Markdown wrapping.

## Tests

Follow the existing patterns in `tests/gate_shell/test_agent_handoff.py`,
`tests/gate_shell/test_transaction_finalizer_owned_turn.py`, `tests/test_sudo_gate.py`,
`tests/test_llm_provider_invoke.py` (+ `tests/_llm_provider_invoke_helpers.py`),
`tests/test_axe_run_agent_exec_retry.py` (+ helpers),
`tests/test_run_agent_gate_handoff.py`, and `tests/test_run_agent_runner_lifecycle.py`.
New test files are fine; keep each under the toobig limits.

1. **Marker I/O** (`tests/test_agent_gate_intent.py`):
   - A marker is written only with `SASE_AGENT` + `SASE_ARTIFACTS_DIR` set and when
     `agent_meta.json["pid"]` differs from the current pid; it is not written when they
     match (the runner case).
   - The file is per-pid, and re-stamping updates `request_id`.
   - Clear removes only this process's file; discard removes all; corrupt JSON is
     tolerated.
2. **`create_gate_shell` lifecycle**
   (`tests/gate_shell/test_transaction_gate_intent.py`), covering each outcome in the
   epic's test list:
   - **Killed before first write:** patch `_resolve_creator` to raise
     `KeyboardInterrupt`. The marker (kind + request id) remains, and no member exists.
   - **Clean CLI error:** `_resolve_creator` raises `GateShellLaneError`, or
     `create_gate` raises `GateError`. The marker is cleared.
   - **Auto-resolved gate:** the marker is cleared.
   - **Non-terminal creation:** the marker is kept.
   - **Finalizer-owned turn:** no marker is ever written.
3. **Handoff** (extend `tests/gate_shell/test_agent_handoff.py`):
   - **Normal handoff:** patch `kill_agent_runner_group` to assert that, at kill time,
     `.sase_gate_pending` exists and the intent is gone.
   - **`ShellHandoffError`:** the marker is cleared.
4. **Sudo front door:**
   - `_request` writes the early marker before reading stdin (assert from a patched
     `_read_stdin_object`).
   - Invalid stdin raises `GateError`, and the marker is cleared.
5. **Guard** (`tests/test_llm_provider_gate_intent_guard.py`):
   - No intent: no-op.
   - Pending handoff present: intents discarded, no raise.
   - Dead pid: raises `GateIntentLostError`. The message contains `gate intent lost:`,
     the kind, and the request id; evidence `gate_intent_lost.json` is written; the
     marker is removed.
   - **Real kill:** a `subprocess` Python child with `SASE_AGENT`/`SASE_ARTIFACTS_DIR`
     set calls `begin_gate_intent("sudo", request_id=...)`, then
     `os.kill(os.getpid(), signal.SIGKILL)`. Adjudication then raises. This is the
     killed-CLI-before-first-write case end to end.
   - **Live pid within the grace period:** a live pid whose marker is cleared during the
     wait means no raise.
   - **Live pid beyond the grace period** (grace monkeypatched small): raises.
   - `was_killed()` true: no-op.
   - The `cause` chaining and the provider-error text are kept.
6. **`_invoke`:**
   - A provider returns normally while a lost intent exists: `run_finalizers` and
     `postprocess_success` are not called, `postprocess_error` is called, and
     `GateIntentLostError` propagates.
   - A normal turn with no intent is unchanged.
7. **Retry:**
   - `handle_workflow_error` returns `"raise"` for a `WorkflowExecutionError` whose
     cause is `GateIntentLostError`, even when the text matches a configured retry
     pattern and a fallback model is configured.
   - A provider error plus a lost intent is converted by the loop and not retried.
8. **Runner loop and summary:**
   - The killed iteration discards intents while still adopting `.sase_gate_pending` (a
     normal handoff still ends `"gated"`).
   - The backstop raises after a non-killed loop with a lost intent.
   - The runner `main` error summary leads with the `GateIntentLostError` message, so
     the failed `done.json` `error` and the notification notes name the kind and request
     id.
   - A normal non-gate run still finishes SUCCESS with no spurious failure.

## Verification

- Run `just install` if the workspace venv is stale, then `just fix`, then `just check`.
  Hand `just check` to a `/sase_monitor` verify monitor (`TESTING`/`TESTED`) if it runs
  long.
- Fix Symvision findings per the `symvision` memory. Do not delete new public helpers
  that tests or other modules use.
- Before closing, run `sase bead epic-symbols sase-11t.1`. It currently reports no
  entries; if any appear, resolve them or re-key them to the parent epic `sase-11t`.

## Completion

- Close **only** `sase-11t.1`:
  `sase bead close sase-11t.1 --note "<what was verified>"`. Never close `sase-11t` or
  any ancestor.
- Do not create beads. Record discovered work with
  `sase bead note sase-11t.1 'PROPOSED FOLLOW-UP: <summary — detail>'`. Two likely
  candidates, to be filed only if confirmed while implementing:
  - A creation killed after `create_gate_shell_member` and the claim move leaves a
    pending gate-shell member with no bundle, whose workspace claim was already
    retitled. Check whether `gate_shell/reclaim.py` settles it, and whether the failed
    creator's workspace hold then errors.
  - The other runner handoff commands (`sase plan propose`, `sase questions`,
    `sase pipe`, monitors) have the same crash-before-marker window and could reuse the
    intent pattern.
