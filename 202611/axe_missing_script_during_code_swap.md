---
tier: tale
title: Defer axe script jobs whose script vanishes mid code swap
goal:
  A script job whose entry point is briefly missing while a live code-swap writer (sase
  dev update / just install / just install-dev) holds the lock is recorded as a
  non-advancing skipped run and retried, instead of a missing_script error that reaches
  the axe error digest; a missing script with no active swap still fails loudly.
size: small
proposed_by: bbugyi200.athena.1db
proposed_with: claude/opus
create_time: 2026-11-17 07:23:18
status: wip
---

# Defer axe script jobs whose script vanishes mid code swap

## Problem

The axe error digest keeps reporting one-off failures like:

```
Routine:    waits
Job:        agent_auto_restart
Error:      Job script not found: sase_job_agent_auto_restart (job: agent_auto_restart)
```

These are races with a code swap, not real configuration errors:

- 2026-11-17 07:15:51 (`agent_auto_restart`): a `sase dev update` was reinstalling the
  uv tool environment. The tool venv's `bin/sase_job_agent_auto_restart` was rewritten
  at 07:15:51.61, its `~/.local/bin` symlink at 07:15:51.70, and the uv receipt at
  07:15:51.74. The axe orchestrator was then restarted at 07:16:19.
- 2026-11-16 10:21:51 (`tg_outbound`, a sase-telegram plugin script): this overlapped
  the `just install-dev` run logged as
  `~/.sase/logs/install/install-20261116T152146Z-*`, which started at 10:21:46 local
  time.

`agent_auto_restart` is especially exposed because its fs trigger watches
`locks/code-swap-v2.lock` (`src/sase/default_config.yml`, the `agent_auto_restart` job
under the `waits` routine). The job wakes up exactly when a swap takes the writer lock,
which is the window when uv has removed the old entry-point scripts and not yet written
the new ones. The earlier mitigation in `discover_chop_script`
(`src/sase/axe/chop_script_runner.py`) only helps when the venv's own `bin/` still has
the script. It cannot help while uv is reinstalling the distribution that owns that
script.

Today `run_script_chop_once` (`src/sase/axe/chop_runner_script.py`) turns any
`discover_chop_script(...) is None` result into a `missing_script` outcome. The routine
(`src/sase/axe/lumberjack.py`, the `outcome.status == "missing_script"` branch) records
that as an error, so it shows up in `~/.sase/axe/recent_errors.json` and the digest.

## Fix

When a job's script can't be found **and a live code-swap writer holds the lock**,
record the run as a policy `skipped` run that does not advance cadence, instead of
`missing_script`. The job retries on its next tick or trigger. Releasing the lock
truncates the lock file, which also re-fires `agent_auto_restart`'s fs trigger. A
missing script with no active swap keeps its current `missing_script` behavior, so real
misconfiguration stays loud.

### 1. Public writer-holder probe in `src/sase/dev_update/code_swap_lock.py`

Add a public, non-blocking helper next to `code_swap_readers_active()`:

```python
def code_swap_writer_holder() -> str | None:
    """Return a description of a live code-swap writer, or None."""
```

- Return `None` when `_lock_disabled()`.
- Read the holder payload with the existing `_read_writer_holder()`. Both writers record
  their identity in the lock file only while they hold the exclusive `flock`, and clear
  it before unlocking: `code_swap_writer_lock()` (op `dev.update`) and
  `tools/_sase_install_run.py`'s `acquire_swap_lock` / `SwapLock.release` (ops
  `install.dev` / `install.pypi`). Readers never write the lock file. A non-empty dict
  payload therefore means a writer.
- Require a live pid: `_holder_pid(holder)` is not `None` and `is_process_running(pid)`.
  This makes a payload left behind by a crashed writer read as "no swap", so the probe
  fails toward the existing loud error.
- On success return `_format_writer_holder(holder)`. Optionally extend `_display_op` so
  `install.dev` / `install.pypi` render as `just install-dev` / `just install`.
- Do **not** take any `flock` in the probe, not even a non-blocking shared one. A brief
  shared lock could make a concurrent non-blocking `sase dev update` writer defer
  spuriously.
- Never raise: treat `OSError` / malformed JSON as `None`. `_read_writer_holder` already
  does this for the read.

Leave `prebuild.py`'s private `_code_swap_writer_active()` alone. It deliberately
matches only `dev.update`.

### 2. Defer instead of failing in `src/sase/axe/chop_runner_script.py`

- Add an injectable seam to `run_script_chop_once`, matching the existing
  `discover_chop_script_fn` style:
  `code_swap_writer_holder_fn: Callable[[], str | None] = code_swap_writer_holder`.
  Import it from `sase.dev_update.code_swap_lock`; axe already imports that module in
  `run_agent_runner_launch.py`.
- In the `if script is None:` branch, call the probe first and wrap the call in
  `try/except Exception` so a probe failure falls back to `missing_script`. If it
  returns a holder description:
  - Write a `ChopRunEntry` with `status="skipped"`, `finished_at`/`duration_ms` set, no
    `error`/`traceback`, and
    `reason=f"job script {script_name} is unavailable while a code swap is in progress ({holder}); retrying after the swap"`.
    Pass the same text as `output` so the run log explains itself. Ignore `OSError` the
    same way the existing branch does.
  - Keep the checkpoint bookkeeping the same as the missing-script path:
    `record_checkpoint_best_effort(..., "action_failed")`. A trigger checkpoint observed
    during the swap then stays retryable instead of being acknowledged.
  - Return
    `ChopRunOutcome(status="skipped", run_id=run_id, reason=<text>, advances_cadence=False, skip_reason="inhibited", chop_verbose=chop_verbose)`.
- Otherwise keep the existing `missing_script` behavior byte-for-byte.

Check that no consumer has to change. The routine's `skipped` branch (`lumberjack.py`)
already returns `success=True` with no error and only advances `run_every` cadence when
`advances_cadence` is true. The manual CLI (`src/sase/axe/cli.py`) and the TUI chop-run
action (`src/sase/ace/tui/actions/axe_chop_run.py`) already handle `skipped`. Grep for
any other `outcome.status == "skipped"` consumer that assumes a preflight `decision`,
and confirm `reason` alone is enough.

### 3. Tests

- `tests/dev_update/test_code_swap_lock.py`: cover `code_swap_writer_holder()`.
  - It returns a description while `code_swap_writer_lock()` is held, and `None` after
    release.
  - It returns `None` for an empty lock file, for a payload whose pid is dead
    (monkeypatch `is_process_running`), for malformed JSON, and when
    `SASE_DISABLE_CODE_SWAP_LOCK=1`.
  - It recognizes an installer-shaped payload (`op: install.dev`, live pid).
- `tests/test_axe_chop_runner_script.py`, next to
  `test_run_configured_chop_once_missing_script`:
  - Absent script plus an active writer gives `outcome.status == "skipped"`,
    `advances_cadence is False`, `skip_reason == "inhibited"`, and a persisted entry
    with `status == "skipped"`, no `error`, and a `reason` naming the script and the
    swap holder. Either write a live writer payload into the temp state's
    `locks/code-swap-v2.lock`, or patch the probe at its import site in
    `sase.axe.chop_runner_script`.
  - Absent script with the probe raising still gives `missing_script`.
  - The existing missing-script test (no lock file) stays unchanged and green.
- A routine-level test (`tests/test_axe_lumberjack_tick.py` or similar) shows that a
  deferred-for-swap outcome adds nothing to recent errors / the error digest input. Add
  it only if a cheap seam exists; otherwise the runner-level tests are enough.

### 4. Docs

In `docs/reference/automation/scheduler-jobs.md`, in the run-status paragraph that lists
`missing_script` (around "Exit-code-only scripts end as ..."), add one sentence: a
script that is missing while a live code-swap writer (`sase dev update`, `just install`,
`just install-dev`) holds the lock is recorded as `skipped` with a reason and retried
after the swap, instead of `missing_script`. Before editing, read the `docs_writing`
reference memory with `/sase_memory_read`.

## Out of scope

- Changing `agent_auto_restart`'s fs trigger paths. Watching the code-swap lock is
  intentional: it catches update-adjacent agent deaths.
- Making the running orchestrator robust to torn imports mid-swap.
- A blocking wait/retry loop for discovery. A swap can take minutes, and the deferral
  above is deterministic.

## Verification

Before finishing, read the `lint_and_test` reference memory with `/sase_memory_read` and
run the verification it prescribes. Then run
`pytest tests/test_axe_chop_runner_script.py tests/dev_update/test_code_swap_lock.py`.
