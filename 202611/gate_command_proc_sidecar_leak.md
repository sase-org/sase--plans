---
tier: tale
title: Stop gate commands from inheriting the gate.answer proc's operation sidecars
goal:
  A gate option or action command can run any sidecar-aware sase subcommand (for example
  sase project create) and succeed when the gate is answered through the detached
  gate.answer proc.
size: small
decisions:
  ops_guard:
    ask: Also make load_request ignore an inherited sidecar for a different operation?
    default: true
    why: A mismatched env sidecar already always fails, so this closes the leak class
    answer: true
proposed_by: bbugyi200.athena.1d1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 14:18:15
status: wip
---

# Stop gate commands from inheriting the gate.answer proc's operation sidecars

## Problem

Agent `tailnet-x.1` on apollo created a custom gate (`custom-245d90e9-…`) whose `create`
option runs a one-line script:

```sh
sase project create gh:bbugyi200/nut --visibility public --description "…" --yes
```

Both answer attempts failed with `command_failed`:

```
operation sidecar ~/.sase/procs/runtime/<proc>/operation-request.json has operation
'gate.answer', expected 'project.create'
```

The agent authored the gate correctly. The failure is a SASE host bug.

## Root cause

1. A gate-turn-backed answer runs as a detached, supervised `gate.answer` proc by
   default (`notification_gates/cli_answer_submit.py::effective_detach`). This covers
   the TUI, mobile, Telegram, and plain `sase gate answer`.
   `procs/submission.py::_with_operation_paths` exports the proc's operation sidecar
   contract into its environment: `SASE_PROC_REQUEST_PATH`, `SASE_PROC_RESULT_PATH`, and
   `SASE_PROC_OPERATION` (constants `REQUEST_ENV`, `RESULT_ENV`, and `OPERATION_ENV` in
   `src/sase/ops/models.py`).
2. `src/sase/notification_gates/command_runner.py::run_owned_command` starts every gate
   option and action command with `command_env = {**os.environ, GATE_COMMAND_ENV: "1"}`.
   That passes those three variables straight to the reviewed command.
3. The nested `sase project create` reaches
   `main/project_handler_add.py: if resolve_request_path(args) is None`. Through
   `ops/cli.py::resolve_request_path`, the inherited `SASE_PROC_REQUEST_PATH` counts as
   "a sidecar was supplied", so it calls
   `load_request("project.create", …, required=True)`. That loads the parent's
   `gate.answer` request and raises the mismatch.

The same leak breaks any gate command that runs a sidecar-aware `sase` subcommand
(`project add/create`, `run`, `notify`, `agent drain`, `gate answer`/`act`, and others
that call `load_request`). Through `emit_operation_result`, such a child can also write
its own result into the parent's `operation-result.json`, labeled with the parent's
operation. The agent-launch boundary already handles this case
(`agent/launch_spawn.py::_remove_inherited_proc_operation_env`); the gate command
boundary was missed.

## Changes

### 1. Scrub the operation-sidecar trio at the gate command boundary (the fix)

- In `src/sase/agent/env_hygiene.py`, add `scrub_proc_operation_sidecar_env(env)`. It
  pops exactly `SASE_PROC_REQUEST_PATH`, `SASE_PROC_RESULT_PATH`, and
  `SASE_PROC_OPERATION`. Import the names from `sase.ops.models` rather than re-spelling
  them, and make sure the import does not create a cycle; a local import inside the
  function is fine. Its docstring should say why it is narrower than
  `scrub_proc_operation_env`: a gate command runs inside the answering proc's lifetime
  and process tree, so `SASE_PROC_ID`, `SASE_PROC_LOG_PATH`, and `SASE_PROC_SESSION_ID`
  stay accurate. Tool-run ownership (`tool/ownership.py`, `tool/adopt.py`) and
  `notification_gates/execution_owner.py` keep attributing to that proc. Only the
  sidecar contract belongs to the proc's own command.
- In `run_owned_command`, build the env as a copy of `os.environ`, call the new
  scrubber, then set `GATE_COMMAND_ENV` and the optional `GATE_WORKSPACE_ENV`. Both the
  `subprocess.run` path and `_run_command_streaming` already receive `command_env`, so
  one change covers options and repeatable actions (`attempts.py` and `operations.py`
  both route through `run_owned_command`). Extend the module docstring's trust-model
  paragraph with one sentence on this env boundary.

### 2. Defense in depth in `sase.ops.cli` (decision `ops_guard`)

> [!decision] ops_guard
>
> In `src/sase/ops/cli.py`, give `resolve_request_path` an optional keyword
> `operation: str | None = None`. When the path comes from `REQUEST_ENV` rather than the
> explicit `-Q/--request-path` flag, `operation` is given, and `OPERATION_ENV` is set
> and non-empty but differs from `operation`, return `None`: the sidecar belongs to an
> ancestor proc. `load_request` passes its `operation` through, so an inherited foreign
> sidecar behaves as "no sidecar". `required=True` callers still fail loudly, with
> `missing` instead of `mismatched`. Change
> `main/project_handler_add.py: resolve_request_path(args)` to pass its `operation`.
>
> An explicit `-Q` path must stay strict: keep the `expected_operation` mismatch error
> in `read_operation_request`. Do not change `resolve_result_path` or
> `emit_operation_result`. Several emitters intentionally prefer the proc's
> `OPERATION_ENV` label, so a result-side guard would need a producer-inventory audit.
> That is out of scope here.
>
> This changes no working path. Every proc producer writes the sidecar's `operation` and
> `OPERATION_ENV` from the same `request.operation`, so an env/expected mismatch already
> guaranteed a hard `mismatched` failure.

> [!decision] ops_guard = no
>
> Skip section 2 entirely. Only the gate command boundary changes.

## Tests

- `tests/test_gate_command_history_marker.py` has the pattern to follow: a tiny
  hash-verified bundle command that echoes env. Add a test, either in a new
  `tests/test_gate_command_proc_env.py` or alongside the existing ones. It monkeypatches
  all six `SASE_PROC_*` variables, runs the command through both the `subprocess.run`
  path and the streaming path, and asserts that:
  - the request, result, and operation variables are absent in the child;
  - `SASE_PROC_ID` (and the log and session vars) are still present;
  - `SASE_GATE_COMMAND` is still `1`.
- `tests/test_agent_env_hygiene.py`: unit test that `scrub_proc_operation_sidecar_env`
  removes only the trio.
- If `ops_guard` is true, extend `tests/test_proc_env_isolation.py`, reusing its
  `_seed_live_proc_env`-style setup and its intentional-override pattern. Cover four
  cases:
  - `load_request("project.create")` with an env sidecar for `gate.answer` returns an
    empty payload when `required=False`, and raises `OperationIOError` with code
    `missing` when `required=True`;
  - the matching-operation env path still loads;
  - an explicit `-Q` path for the wrong operation still raises `mismatched`;
  - `OPERATION_ENV` unset keeps today's behavior.
- Optional end-to-end regression, if cheap with existing gate test fixtures: a custom
  gate whose command runs a sidecar-aware `sase` subcommand that succeeds offline (one
  that calls `load_request`), answered through the detached proc path, settles
  `CONFIRMED`. Skip this if no fixture makes it hermetic; the unit tests above pin the
  boundary.

## Verification

- `just check` (per the lint-and-test memory). Do not run `just check-full`.
- No memory note describes this boundary, so no memory update is needed. Nothing here
  touches the Rust core: the env boundary is Python host glue in this repo.

## Recovery for the failed apollo gate (operator note, not implementation work)

The gate (`custom/custom-245d90e9-bd94-45d9-8082-41cb87248a89`, 24 h timeout from
2026-11-16 14:06 ET) is still recoverable. Use either route:

- after this fix ships and is installed on apollo, press **resume** on the
  `GateExecutionFailed` notification; or
- right now, from a plain apollo shell (outside any SASE proc), run
  `sase gate answer --kind custom --id custom-245d90e9-bd94-45d9-8082-41cb87248a89 --option create --resume --no-detach`.
  `--no-detach` keeps the answer in-process, so no `gate.answer` sidecar env exists to
  leak.
