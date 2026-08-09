---
tier: tale
title: Fix ACE `,U` update crashing with "No such file or directory - 'just'"
goal:
  The ACE comprehensive update (`,U`) rebuilds Rust dev artifacts successfully again,
  because every dev-update reconcile step receives a complete child environment
  regardless of which command runner executes it, and an unlaunchable step command
  becomes a labeled step failure instead of an opaque aggregate exception.
proposed_by: bbugyi200.athena.wp
create_time: 2026-08-09 14:59:34
status: wip
---

# Plan: Restore a complete child environment for dev-update reconcile steps

## Symptom

Pressing `,U` (leader mode `update_sase`, `src/sase/default_config.yml:595`) in ACE
raises an error toast:

```
SASE, core & plugins: [Errno 2] No such file or directory: 'just'; Agent CLIs: no
captured work; Cached agents: no cached agent hoods
```

`just` is installed (`~/.cargo/bin/just`) and `~/.cargo/bin` **is** on the `PATH` of the
running `sase ace` process, so this is not a user environment problem.

## Root cause

`DevReconcileStep.env` (`src/sase/dev_update/models.py:133`) is an **overlay** — the
planner only ever puts one key in it:

- `src/sase/dev_update/plan.py:326-335` — the `rust_dev_install` step runs
  `("just", "rust-dev-install-uv-tool")` with `env=_rust_dev_install_env()`, which
  returns `{"SASE_RUST_DEV_PROFILE": <profile>}`
  (`src/sase/dev_update/plan.py:409-411`).
- `src/sase/dev_update/plan.py:448-467` — the `rust_prebuild_install` step carries the
  same single-key `env`.

But that overlay is handed to the injected runner **raw**
(`src/sase/dev_update/execute.py:518-532`):

```python
if env is None:
    result = run(argv, cwd=cwd)
else:
    result = run(argv, cwd=cwd, env=env)
```

Two runners implement `DevCommandRunner` (`src/sase/dev_update/models.py:174-183`), and
they disagree about what `env` means:

| Runner                                                                                                | Used by                                                   | Treats `env` as                                                                                                                                                |
| ----------------------------------------------------------------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `run_dev_update_command` (`src/sase/dev_update/execute.py:156-200`)                                   | `sase update` CLI (`src/sase/main/update_handler.py:339`) | overlay — merges over `os.environ` via `_merged_subprocess_env` (`execute.py:558-563`)                                                                         |
| `dev_update_reporter_runner` (`src/sase/ace/tui/modals/plugins_browser_sase_update_tasks.py:332-345`) | every ACE update path, including `,U`                     | **full replacement** — forwards to `TaskReporter.run`, which reaches `subprocess.Popen(..., env=dict(env) ...)` at `src/sase/ace/tui/task_subprocess.py:48-57` |

So in the TUI the `just` step is spawned with an environment containing exactly one
variable and no `PATH`. `subprocess.Popen` then resolves a bare executable name against
`os.get_exec_path(env)`, which falls back to `os.defpath` (`:/bin:/usr/bin`) — `just` is
not there. Verified reproduction:

```console
$ python3 -c "import subprocess; subprocess.Popen(['just','--version'], env={'SASE_RUST_DEV_PROFILE':'dev-update'})"
FileNotFoundError: [Errno 2] No such file or directory: 'just'
```

That is byte-for-byte the string in the toast.

`TaskReporter.run` does not translate the `FileNotFoundError` into a result, so it
escapes through `dev_update_reporter_runner` and `execute_dev_update` into the broad
`except` in `_execute_comprehensive_sase_leg`
(`src/sase/ace/tui/modals/plugins_browser_comprehensive_update_execution.py:166-177`),
which renders it via `error_text(exc)` under the `SASE, core & plugins:` prefix (same
file, line 269). That is why the toast has no step label and no command context.

### Why it started failing today

`env=` was first attached to the `rust_dev_install` step by `2bb7ce463` (_perf: use
dev-update profile for rust updates_, 2026-08-09 12:49) and a second env-bearing step
was added by `9bce277c9` (_feat: add Rust prebuild cache_, 13:51). The 14:44:46 `,U`
succeeded because the process was still running pre-`2bb7ce463` code; it installed those
commits and reloaded, so the 14:50:21 attempt was the first to execute the new plan
under the TUI runner.

### Second, quieter symptom

The `rust_prebuild_install` step runs an absolute interpreter path, so it launches even
with the truncated environment — but with no `HOME`, `PATH`, or `CARGO_HOME`. It is
therefore unreliable and can report a spurious cache miss, meaning the prebuild cache
added in `9bce277c9` has effectively never worked from the TUI. Fix 1 repairs this too.

### Blast radius

TUI-only. `sase update` on the CLI is unaffected because `run_dev_update_command`
already merges. No `sase-core` (Rust) change is needed: this is Python-side update
plumbing and presentation glue, not shared backend behavior, so the Rust core backend
boundary rule does not apply.

## Fix

### Fix 1 (root cause) — merge the overlay at the single choke point

In `src/sase/dev_update/execute.py`, make `_run` resolve the overlay before it reaches
any runner, so the `DevCommandRunner` contract becomes "`env`, when not `None`, is the
complete child environment":

```python
    if env is None:
        result = run(argv, cwd=cwd)
    else:
        result = run(argv, cwd=cwd, env=_merged_subprocess_env(env))
```

`_merged_subprocess_env` already exists at `src/sase/dev_update/execute.py:558-563` and
returns `dict(os.environ)` updated with the overlay. Because it is defined below `_run`
in the same module, no import change is needed.

Also update the two docstrings that encode the contract, so the next runner author
cannot repeat the mistake:

- `DevReconcileStep.env` (`src/sase/dev_update/models.py:125-137`): state that it is an
  overlay applied by the executor over the parent environment.
- `DevCommandRunner.__call__` (`src/sase/dev_update/models.py:174-183`): state that a
  non-`None` `env` is already complete and must be passed through to the child as-is.

Keep the existing merge inside `run_dev_update_command`. It becomes idempotent for the
`_run` path and still protects direct callers of that public helper.

### Fix 2 (fault isolation) — an unlaunchable command is a step failure, not a crash

Make `dev_update_reporter_runner`
(`src/sase/ace/tui/modals/plugins_browser_sase_update_tasks.py:332-345`) mirror the
error mapping that `run_dev_update_command` already performs
(`src/sase/dev_update/execute.py:186-195`):

- `FileNotFoundError` → `DevCommandResult(returncode=127, stderr=str(exc))`
- `subprocess.TimeoutExpired` →
  `DevCommandResult(returncode=124, stderr="command timed out")`
- `OSError` → `DevCommandResult(returncode=1, stderr=str(exc))`

This is worth doing independently of Fix 1. With the exception escaping today, ACE loses
the step label, discards the detail of legs that already succeeded, and — because
`_run_reconcile_steps` treats a failing rust build step that has a later health check as
a _pending_ failure (`src/sase/dev_update/execute.py:360-384`) — skips the
health-check/repair recovery path in `_run_rust_health_check_step`
(`src/sase/dev_update/execute.py:436-501`) entirely. After the fix, a genuinely missing
`just` yields a labeled failure that still runs the `sase_core_rs` import health check.

Do **not** change `TaskReporter.run` / `_stream_subprocess` semantics — see rejected
alternatives.

## Tests

1. **Update**
   `test_execute_dev_update_runs_unified_rust_install_before_core_health_check` in
   `tests/dev_update/test_execute_reconcile.py:97-138`. Its assertion that the runner
   saw exactly `{"SASE_RUST_DEV_PROFILE": "release"}` encodes the buggy contract. With
   `monkeypatch.setenv("PATH", "/sentinel/bin")`, assert instead that the captured env
   contains both `SASE_RUST_DEV_PROFILE == "release"` and `PATH == "/sentinel/bin"`.
   (The test currently takes no fixtures; add `monkeypatch: pytest.MonkeyPatch`.)

2. **Add** a focused regression test in `tests/dev_update/test_execute_reconcile.py`,
   e.g. `test_execute_dev_update_gives_runner_a_complete_environment`: a plan with one
   env-bearing reconcile step, a `FakeRunner`
   (`tests/dev_update/_execute_helpers.py:88-107` — it already records `env_calls`), a
   monkeypatched sentinel `PATH`, and assertions that the runner received the sentinel
   `PATH` plus the overlay key. Also assert a step with `env=None` still gets `env=None`
   (inherit), so the "no overlay" path is pinned.

3. **Add** a test for the reporter runner — a new
   `tests/ace/tui/test_dev_update_reporter_runner.py` is the cleanest home. Build a
   `TaskReporter` over a real `TaskInfo` (see the existing task tests in
   `tests/ace/tui/test_task_queue.py` for construction) or a small stub with `phase` and
   `run`; make `run` raise
   `FileNotFoundError("[Errno 2] No such file or directory: 'just'")` and assert the
   runner returns `DevCommandResult(returncode=127, ...)` with `just` in `stderr` rather
   than propagating. Cover the `TimeoutExpired` and generic `OSError` branches too.

No PNG snapshot goldens are affected.

## Verification

```bash
just install           # ephemeral workspace: dependencies may be stale
just check
```

`src/sase/dev_update/execute.py` is on the shared update path, so if `just check`
reports an unusual or escalated selection, follow up with `just check-full`.

Manual confirmation (needs a real pending update, so it is best done by the project
owner rather than blocking the implementer):

1. From ACE, press `,U` and confirm the comprehensive update.
2. Open the tracked task log for `sase update` and confirm the
   `Rebuild Rust dev artifacts into the uv-tool venv` phase actually runs
   `just rust-dev-install-uv-tool` and exits 0, and that
   `Install prebuilt Rust dev artifacts into the uv-tool venv` reports a real cache
   hit/miss rather than `stamp-missing`.
3. Confirm the completion toast reports the update instead of the `[Errno 2]` error.

Note for whoever verifies: the currently running ACE instance is on the broken code, so
it must be updated (via `sase update` on the CLI, which is unaffected) before `,U` can
be exercised against the fix.

## Considered and rejected

- **Make `_stream_subprocess` / `TaskReporter.run` treat `env` as an overlay.**
  Rejected: full replacement is the standard `subprocess` contract, and
  `sase.agent_clis.runner.run_command` (`src/sase/agent_clis/runner.py:84-95`)
  deliberately hands it a complete env it has already composed (including a managed
  `TMPDIR`). Making the TUI helper silently overlay would prevent any future caller from
  intentionally scrubbing a variable.
- **Merge only inside `dev_update_reporter_runner`.** Rejected: it fixes the one runner
  that exists today and leaves the identical trap set for the next injected runner.
- **Guard the step at plan time with `shutil.which("just")`** (as `_stale_core_plan`
  already does for `cargo`, `src/sase/dev_update/plan.py:380-385`). Not needed once Fix
  1 lands, and Fix 2 already produces a labeled, actionable failure when `just` is
  genuinely absent. Reasonable as a separate follow-up, not part of this fix.

## Acceptance criteria

- Every `DevReconcileStep` command executed through `execute_dev_update` is spawned with
  the parent environment plus the step's overlay, under both the CLI runner and the ACE
  tracked-task runner.
- A launch failure inside the ACE dev-update runner surfaces as a step-labeled failure
  with a non-zero return code, not as an escaped exception.
- New and updated tests above pass, and `just check` is clean.
