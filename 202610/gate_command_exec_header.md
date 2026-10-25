---
tier: tale
title: Reject gate commands that cannot be exec'd without a shell
goal:
  A gate command with no shebang fails at sase gate create, never after a reviewer
  accepts it.
size: small
decisions:
  unstartable_recovery:
    ask:
      Should a gate command that can never start (ENOEXEC) get its own failure code
      whose recovery offers only cancel?
    choices:
      narrow:
        New `command_not_executable` code; the recovery notification and TUI modal offer
        only cancel
      keep:
        Keep `command_start_failed` with resume/restart/cancel; only the error message
        gets clearer
    default: narrow
    why:
      resume/restart re-run the same hash-locked bytes, so they are certain to fail
      again
    answer: narrow
proposed_by: bbugyi200.athena.14d
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 15:45:47
status: wip
---

# Reject gate commands that cannot be exec'd without a shell

## Problem

Phase agent `bob-cli-an.6.6.6.5.6.5.5.6.6.6.1` (project bob-cli) is stalled. It handed
its remaining decision to custom gate `custom-bcd7a8e0-d70b-463b-bc09-88186ade243e`
("Repair live vault reference ids (bob-cli-b8)"). Bryan accepted `repair` from the TUI,
and execution failed before the command ran:

```
command_start_failed: cannot start command: [Errno 8] Exec format error: '/proc/self/fd/5'
```

The gate's `commands/repair`, `commands/tidy`, and `commands/show_diagnostics` scripts
start with `set -eu`. They have no `#!` line (`commands/reject` has `#!/bin/sh`).
`run_owned_command` in `src/sase/notification_gates/command_runner.py` runs every gate
command with `shell=False`. Where `/proc` exists, it runs the hash-verified descriptor
as `/proc/self/fd/N` (`_exec_target`). Without a shell there is no `/bin/sh` fallback on
ENOEXEC, so a script without a shebang can never start. This also holds on the macOS
path-exec branch, because Python's subprocess calls `execve` directly.

Nothing checks for this when the gate is created. `GateResource.from_mapping`
(`model_request.py`), `validate_gate_spec` (`validation.py`), and `materialize_resource`
(`durability.py`) accept any bytes for an executable command resource. So the gate was
created, reviewed, and accepted, and it failed only at execution. The lifecycle policy
then classifies the gate as `accepted_failed`. `src/sase/gate_turn/reclaim.py` defers
that state on purpose, so the gate turn sits in `settling` with a workspace claim, and
the clan's `.land` agent stays WAITING. The recovery notification offers
`resume,restart,cancel` (`_recovery_actions` in `failure_notifications.py`). Resume and
restart re-run the same reviewed bytes and are guaranteed to fail again.

The `sase_gate` skill (`src/sase/macros/skills/sase_gate.md`) says command argv runs
"without a shell", and every example has `#!/bin/sh`. It never says the interpreter line
is required, so the authoring agent left it out.

## Changes

### 1. Reject unstartable command resources at creation

In `src/sase/notification_gates/durability.py` `materialize_resource`, after computing
`content` and before opening the target, reject an executable resource whose bytes start
with neither `b"#!"` nor the ELF magic `b"\x7fELF"`. Raise:

```python
GateError(
    "invalid_command",
    resource.path,
    "command resources run without a shell and must start with a '#!' "
    "interpreter line (for example '#!/bin/sh') or be a native executable",
)
```

Put the header test in a small named helper, for example `_has_exec_header(content)`.
This check covers both `content` and `source` resources, because `source` bytes are read
in the same function. Creation already wraps materialization in `_start_gate_creation`'s
`try`/`except BaseException` (`service_creation.py`), which removes the partial bundle.
No other cleanup is needed. Command resources cannot be rewritten after creation:
`edit_file` and action `targets` must be `editable` (`validation.py`). So this one check
point is enough.

Before landing, confirm that every built-in command producer already emits a header.
These producers go through `python_gate_command_script` in `entrypoints.py` (which emits
`#!/bin/sh`) or hand-written content: `plan_gate.py`, `user_question_actions.py`,
`macro/workflow_hitl_gate.py`, `sudo/gate.py`, `agent/launch_request_gate.py`,
`bead/_task_gate_spec.py`, `bead/_stale_cleanup_gate_spec.py`,
`bead/_flag_gate_spec.py`, `bead/_snooze_gate_spec.py`, and
`plugins/_required_gate_spec.py`. Fix any that do not.

### 2. Classify ENOEXEC as a failure no retry can fix

> [!decision] unstartable_recovery = narrow
>
> In `run_owned_command`'s `except OSError` branch, map `exc.errno == errno.ENOEXEC` to
> `GateError("command_not_executable", command_argv[0], ...)`. The message should name
> the cause: no `#!` interpreter line, and not a native executable. Other `OSError`s
> keep `command_start_failed`. In `failure_notifications.py`, make `_recovery_actions`
> also take the failure code. For `command_not_executable` it returns `("cancel",)`.
> Update both call sites (`publish_gate_execution_failed` and
> `gate_failure_action_data`). The TUI modal
> (`src/sase/ace/tui/actions/agents/_notification_gate_recovery.py`) already honors the
> comma-joined `recovery_actions` in action data, so it only needs a test. Check that
> nothing else, Python or the `sase_core` lifecycle wire, switches on the literal
> `command_start_failed` in a way that drops the new code. Grep `src/` for the code and
> for `_FIXED_SUMMARY_CODES`-style sets. The Rust lifecycle policy receives failure
> facts as opaque strings today. If you find it does enumerate codes, stop and record a
> `PROPOSED FOLLOW-UP:` instead of editing sase-core in this tale.
>
> With `keep`, leave the code and the recovery actions as they are, and only add the
> ENOEXEC hint to the `command_start_failed` message.

### 3. Document the requirement

- `src/sase/macros/skills/sase_gate.md`: in the paragraph that starts "Command `argv`
  arrays are executed without a shell", state that every command resource must start
  with a `#!` interpreter line (for example `#!/bin/sh`), or be a native executable. Say
  that `sase gate create` rejects other content with `invalid_command`. Mention it again
  next to the `source` guidance ("For larger commands, set a resource's `source`...").
  This is a generated skill source. Do not edit or deploy chezmoi copies. Deploying
  happens only from a landed tree, per the generated-skills memory note.
- `docs/reference/automation/notification-gates.md`: add the same rule beside "Commands
  are argv arrays executed without a shell" (around line 21). Read the `docs_writing`
  reference memory before editing `docs/`.

### 4. Tests

- `tests/test_notification_gates.py` (or the creation-focused gate test module): a
  `content` command without a header is rejected with `invalid_command`, and no bundle
  directory is left behind. The same holds for a `source` command. Bundles are created
  for `#!/bin/sh` content and for ELF-magic content (`b"\x7fELF..."` written via
  `source`).
- `tests/fakey/test_gate_capacity_custom_e2e.py` around line 443 creates a valid gate
  with `"content": "not a script\n"` and then strips the executable bit to force a start
  failure. Change its content to `"#!/bin/sh\nexit 0\n"` so creation still succeeds. The
  chmod-based start failure still exercises the same path.
- With `unstartable_recovery = narrow`, add a test in
  `tests/test_notification_gate_execution.py`. Build a valid gate, then replace the
  command file in place with headerless bytes and re-record the resource hash in the
  envelope, the way existing tests tamper with bundles. Accepting it must record code
  `command_not_executable`, and the failure notification's
  `action_data["recovery_actions"]` must equal `"cancel"`. Leave the existing
  `"resume,restart,cancel"` assertions for other failures (line ~218 and
  `tests/test_plan_approval_actions_archive.py:316`) unchanged.
- Run `just check` per the `lint_and_test` memory note.

## Out of scope

- Unblocking the live gate is a reviewer action, not code. Bryan cancels it
  (`sase gate cancel --kind custom --id custom-bcd7a8e0-d70b-463b-bc09-88186ade243e`, or
  cancel in the recovery notification). That triggers its authored `stopped` branch,
  whose follow-up records the failure on bead `bob-cli-an.6.6.6.5.6.5.5.6.6.6.1` and
  re-creates the gate. Nothing ran, so the live vault is unchanged.
- Falling back to `/bin/sh` on ENOEXEC at execution time. That would run reviewed bytes
  under an interpreter the reviewer never saw declared, which weakens the hash-verified
  trust model.
