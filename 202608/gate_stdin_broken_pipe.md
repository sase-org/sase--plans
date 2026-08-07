---
tier: tale
title: Stop gate command execution from failing on a broken stdin pipe
goal:
  'Answering a notification gate never fails with the spurious error "cannot start
  command: [Errno 32] Broken pipe" when a gate command exits without reading its stdin
  payload, and a deterministic regression test pins the behavior.'
proposed_by: bbugyi200.athena.v4
create_time: 2026-08-07 17:21:13
status: done
---

- **PROMPT:**
  [prompts/202608/gate_stdin_broken_pipe.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gate_stdin_broken_pipe.md)

# Plan: Stop gate command execution from failing on a broken stdin pipe

## Problem

`just test` fails intermittently on:

```
FAILED tests/ace/tui/test_notification_custom_gate.py::test_tracked_executor_reports_terminal_and_extra_commands_live
AssertionError: assert False is True
 +  where False = _GateTaskOutcome(message='cannot start command: [Errno 32] Broken pipe', success=False, severity='error').success
```

The test is not at fault. It exposes a real race in the streaming gate command executor,
which is the code path the ACE TUI uses to answer every notification gate.

## Root Cause

`_run_command_streaming` in `src/sase/notification_gates/executor.py` writes the
option's JSON input payload to the command's stdin like this (current lines 489-499):

```python
try:
    try:
        process.stdin.write(input_data)
        process.stdin.flush()
    except BrokenPipeError:
        pass
    finally:
        process.stdin.close()
    returncode = process.wait()
    stdout_thread.join()
    stderr_thread.join()
finally:
    ...
```

The `except BrokenPipeError` clause guards only the `write()`/`flush()` pair. It does
**not** guard the sibling `finally: process.stdin.close()`.

That matters because `process.stdin` is a `BufferedWriter`. When the child has already
exited, `flush()` raises `BrokenPipeError` (swallowed) but the unwritten payload stays
in the buffer, so `close()` re-flushes and raises `BrokenPipeError` **again** — this
time with nothing to catch it. This is the exact CPython behavior that
`subprocess.Popen._stdin_write` guards against, and the reason `communicate()` wraps
both the write and the close.

Verified locally: writing to a `Popen` whose child has exited raises from `flush()` and
then raises again from `close()` in 200/200 trials.

The escaping `BrokenPipeError` is an `OSError`, so `_run_owned_command`'s
`except OSError` (line 429) catches it and mislabels it:

```python
raise GateError("command_start_failed", command_argv[0], f"cannot start command: {exc}")
```

Hence the misleading "cannot start command" message for a command that started fine and
exited 0.

### Why it is a race, and why the failing test triggers it

The failing test selects two options. The second, `commands/audit`, is
`#!/bin/sh\nprintf '{"audited": true}\n'` — it never reads stdin and exits immediately.
Whether `flush()` beats the child's exit is pure timing, which is why the test passes
standalone but fails ~5 runs in 8 when the module runs in order.

Confirmed by instrumentation: the failing command is always `('commands/audit',)`, and
the traceback is
`executor.py:496 in _run_command_streaming -> process.stdin.close() -> BrokenPipeError: [Errno 32] Broken pipe`.

### Why this is a production bug, not just a test bug

- Nothing in the gate contract requires a command to read stdin. `input_schema` is
  optional context, and `docs/notifications.md` documents no such obligation.
- The **non-streaming** path is already immune, because it uses `subprocess.run` →
  `communicate()` → `_stdin_write`, which guards the close. So `sase gate` CLI execution
  works while the TUI intermittently fails on the same bundle.
- `_run_command_streaming` is reached only when `on_output_line` is set, and the only
  caller that sets it is
  `src/sase/ace/tui/actions/agents/_notification_gate_execution.py`. So the blast radius
  is exactly "answering a gate from the ACE TUI", which surfaces a spurious red
  `cannot start command` toast and leaves the gate unanswered.

### Secondary defect in the same function

When `close()` raises, control leaves the block before `process.wait()` and the two
`join()` calls. The child is never reaped and the drain threads are never joined, so the
escape also leaks a zombie plus two daemon threads per failure. Any future exception
between `Popen` and `wait()` would leak the same way.

## Scope

Only `src/sase/notification_gates/executor.py` and one test module change. No wire
format, schema, CLI surface, keymap, or config change. No Rust core change: the core
crate only _reads_ gate bundles (`crates/sase_core/src/agent_stats/gate_bundles.rs`); it
does not execute gate commands, so this stays Python-owned and does not cross the Rust
core backend boundary.

## Implementation

### 1. Guard the whole stdin handoff in `src/sase/notification_gates/executor.py`

Add a module-level private helper next to `_run_command_streaming` that mirrors
`subprocess.Popen._stdin_write` semantics, and note in its docstring _why_ it exists so
the guard is not "simplified" away later:

```python
def _ignorable_stdin_error(exc: OSError) -> bool:
    """Report whether one stdin error just means the command ignored its input."""
    return isinstance(exc, BrokenPipeError) or exc.errno == errno.EINVAL


def _write_command_stdin(process: subprocess.Popen[bytes], input_data: bytes) -> None:
    """Hand one payload to a command that is free to never read it.

    A command may exit before draining stdin. The buffered writer then raises
    BrokenPipeError from ``flush()`` and, because the payload stays buffered,
    raises it a second time from ``close()``. Both have to be tolerated, exactly
    as ``subprocess.Popen._stdin_write`` does for ``communicate()``.
    """
    assert process.stdin is not None
    try:
        try:
            process.stdin.write(input_data)
            process.stdin.flush()
        except OSError as exc:
            if not _ignorable_stdin_error(exc):
                raise
    finally:
        try:
            process.stdin.close()
        except OSError as exc:
            if not _ignorable_stdin_error(exc):
                raise
```

Add `import errno` to the existing stdlib import block (alphabetically before
`hashlib`).

### 2. Always reap the child and join the drain threads

Restructure the body of `_run_command_streaming` so the wait and joins cannot be
skipped:

```python
    try:
        try:
            _write_command_stdin(process, input_data)
        except BaseException:
            process.kill()
            raise
        finally:
            returncode = process.wait()
            stdout_thread.join()
            stderr_thread.join()
    finally:
        if on_process_state is not None:
            on_process_state(process, False)
```

`process.kill()` on the exceptional path keeps the `wait()` bounded: stdin is already
closed by then, but a genuinely broken handoff must not let the wait block on a
long-running child. On the normal path nothing is killed and behavior is unchanged.

Keep the existing `subprocess.CompletedProcess(...)` return as-is.

### 3. Do not touch `_run_owned_command`'s `except OSError`

Leave the `command_start_failed` mapping alone. Once the stdin handoff is guarded, the
only OSError sources left in that `try` are the actual start (`Popen`) and `wait()`, so
the label is no longer reachable from a post-start pipe condition. Narrowing the
`except` further is churn without a demonstrated failure.

## Tests

### Deterministic regression test

Add to `tests/test_notification_gate_execution.py` (which already imports
`execute_gate_selection`, `create_gate`, and `gate_spec` from
`tests._notification_gates_fixtures`, and uses the `gate_home` fixture).

A plain integration test is **not** a valid regression test here: measured over 30 runs,
an unassisted gate whose command ignores stdin passed 30/30 even against the buggy code.
The ordering must be forced. Monkeypatch the executor's `subprocess.Popen` with a
subclass that does not return until the child has exited, which makes the broken pipe
certain:

```python
class _ExitedPopen(subprocess.Popen):  # type: ignore[type-arg]
    """Popen that returns only once the command has exited."""

    def __init__(self, *args: Any, **kwargs: Any) -> None:
        super().__init__(*args, **kwargs)
        # Gate command output is a single short line, so this cannot deadlock.
        while self.poll() is None:
            pass


def test_gate_command_may_exit_without_reading_its_stdin_payload(
    gate_home: Path,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    created = create_gate(
        gate_spec(
            request_id="stdin-ignored",
            command='#!/bin/sh\nprintf \'{"status": "ok"}\\n\'\n',
        )
    )
    monkeypatch.setattr(
        "sase.notification_gates.executor.subprocess.Popen", _ExitedPopen
    )
    lines: list[tuple[str, str]] = []

    execution = execute_gate_selection(
        created.bundle_path,
        ["accept"],
        on_output_line=lambda _kind, _id, stream, line: lines.append((stream, line)),
    )

    assert execution.response["option_results"] == [
        {"id": "accept", "result": {"status": "ok"}}
    ]
    assert lines == [("stdout", '{"status": "ok"}')]
    assert created.response_path.is_file()
```

Asserting on `lines` matters: it proves the drain threads were still joined and their
output still reached the caller, which covers the secondary defect. Confirm the
implementer checks both directions — this test must fail on the pre-fix tree (measured
30/30 `GateError: cannot start command: [Errno 32] Broken pipe`) and pass after
(measured 30/30).

Use `monkeypatch.setattr` on the dotted path rather than assigning the module attribute
by hand, so the patch is reverted even if the test fails. Import `subprocess` and
`typing.Any` in the test module as needed.

### Confirm the originally failing test is stable

`tests/ace/tui/test_notification_custom_gate.py::test_tracked_executor_reports_terminal_and_extra_commands_live`
needs no edit. Its `commands/audit` script legitimately ignores stdin, which is the
behavior the fix now supports. Run the whole module repeatedly (at least 10 times) and
require 10/10 green; before the fix it failed roughly 5 times in 8.

```bash
for i in $(seq 1 10); do
  .venv/bin/python -m pytest tests/ace/tui/test_notification_custom_gate.py -q | tail -1
done
```

## Verification

```bash
just install
just check-full
```

`just check-full` rather than `just check`, because a shared subprocess execution
primitive is exactly the kind of change whose scoped-selection closure is easy to
under-approximate, and because the fix must be validated against the full suite that was
red.

Also run the gate-focused modules directly:

```bash
.venv/bin/python -m pytest tests/test_notification_gate_execution.py \
  tests/test_notification_gates.py tests/test_gate_e2e_smoke.py \
  tests/test_gate_option_query.py tests/test_plan_gates_execution.py \
  tests/ace/tui/test_notification_gate_pane.py \
  tests/ace/tui/test_notification_custom_gate.py -q
```

## Out Of Scope

`src/sase/llm_provider/claude.py:339`, `codex.py:478`, `qwen.py:301`, and `fakey.py:293`
each do a bare `process.stdin.write(prompt)` + `process.stdin.close()` with no
`BrokenPipeError` guard, so an agent CLI that dies during startup would raise a raw
`BrokenPipeError` instead of a diagnosable provider error. That is the same class of
defect but a different code path with a different failure mode and no observed failure,
so it stays out of this change. File it as a separate task bead rather than widening
this fix.
