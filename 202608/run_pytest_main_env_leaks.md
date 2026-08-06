---
tier: tale
title: Isolate every process global run_pytest.main() writes, and stop one leak from poisoning a worker
goal:
  test_commit_completion_rows_match_shared_inventory_and_resolve passes on all three test legs of a master CI run,
  because no test in the tools/run_pytest family can leave TMPDIR, the commit-workflow env keys, or the working
  directory mutated past its own teardown, and because a leak that does slip through is contained at the test that
  caused it instead of cascading across its xdist worker.
proposed_by: bbugyi200.athena.tw.f1
create_time: 2026-08-06 09:42:21
status: done
---

# Plan: Isolate every process global `run_pytest.main()` writes, and stop one leak from poisoning a worker

## Context

This continues phase `scratch-fix` (bead sase-fq.8.2) of the epic plan
[202608/artifact_ref_scratch_failure.md](202608/artifact_ref_scratch_failure.md). The previous plan for this phase,
[202608/scratch_tmpdir_leak_fix.md](202608/scratch_tmpdir_leak_fix.md), landed as commit `e0acf80` on PR #278.

That commit's diagnosis was correct and its guard works. Its fix was incomplete: it repaired **one** of **eleven** tests
that leak `TMPDIR`, and the ten it missed reproduce the original epic failure exactly.

### What CI reported

Run [31104590294](https://github.com/sase-org/sase/actions/runs/31104590294), legs `test (3.13)` and `test (3.14)`:

```
ERROR tests/test_run_pytest_scoped.py::test_scoped_escalation_runs_the_governed_fast_lane - Failed:
  ... left TMPDIR: '.../test_scoped_mode_reports_a_fai0/scratch'
             -> '.../test_scoped_escalation_runs_th0/scratch' changed after its own teardown.
= 3 failed, 25831 passed, 18 skipped, 69 warnings, 10 errors in 742.34s =
```

Read the two paths in that message carefully. The **baseline** is already a leaked value — it is the `tmp_path` of a
_different_ test, `test_scoped_mode_reports_a_failing_selection_in_the_manifest`. The guard is not reporting one leak;
it is reporting a link in a chain.

### The ten offenders, reproduced locally

`tools/run_pytest`'s `main()` calls `_prepare_pytest_tmpdir()` (`tools/run_pytest:678`), which writes `TMPDIR` and
`SASE_PYTEST_TMP_REDIRECTED` straight to `os.environ` (`tools/run_pytest:408-409`). Every in-process test that drives
`main()` far enough to reach that line leaks. `install_scoped_selection()` routes `SASE_PYTEST_TMPDIR` through
`monkeypatch`, but nothing restores what `main()` itself writes.

```bash
uv run pytest -p no:randomly -q --no-header \
  tests/test_run_pytest_scoped.py tests/test_run_pytest_main.py \
  tests/test_run_pytest_health.py tests/test_run_pytest_tmpdir.py
```

`30 passed, 10 errors in 2.78s` — and the ten are exactly CI's ten:

| Module                      | Test                                                                  |
| --------------------------- | --------------------------------------------------------------------- |
| `test_run_pytest_scoped.py` | `test_scoped_mode_runs_the_selection_serially_and_never_acquires`     |
| `test_run_pytest_scoped.py` | `test_scoped_mode_reports_a_failing_selection_in_the_manifest`        |
| `test_run_pytest_scoped.py` | `test_scoped_escalation_runs_the_governed_fast_lane`                  |
| `test_run_pytest_main.py`   | `test_main_prepares_governed_environment_and_descriptors_before_exec` |
| `test_run_pytest_main.py`   | `test_main_serial_snapshot_mode_never_acquires`                       |
| `test_run_pytest_main.py`   | `test_main_terminal_smoke_mode_redirects_and_never_acquires`          |
| `test_run_pytest_health.py` | `test_scoped_run_lands_in_the_durable_health_store`                   |
| `test_run_pytest_health.py` | `test_escalated_scoped_run_is_recorded_before_the_handoff`            |
| `test_run_pytest_health.py` | `test_full_lane_arms_the_failure_recorder`                            |
| `test_run_pytest_health.py` | `test_health_recording_can_be_switched_off`                           |

The remaining `tests/test_run_pytest_*.py` modules (`_command`, `_workers`) never call `main()` and do not leak today.

### The epic's original failure is still live

This is the part that matters. The guard names the offender but does **not** undo the leak, so `TMPDIR` stays pointed at
a deleted `tmp_path` for the rest of that worker's session:

```bash
uv run pytest -p no:randomly -q --no-header \
  "tests/test_run_pytest_health.py::test_scoped_run_lands_in_the_durable_health_store" \
  "tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py::test_commit_completion_rows_match_shared_inventory_and_resolve"
# 1 failed, 1 passed, 1 error
```

The catalog test fails with the identical empty-inventory shape epic sase-fq started with, and the probe confirms why:

```
artifact-ref commit inventory: skipping repository sase at ...: could not open a scratch file for `git log` output
scratch-probe: resource state at empty commit inventory
  TMPDIR env: /tmp/pytest-of-bryan/pytest-27/test_scoped_run_lands_in_the_d0/scratch
  tempfile.gettempdir(): /tmp
```

So sase-fq.8.2's actual goal — the parity test passing — is **not** met by `e0acf80`. It only removed one of eleven ways
to reach the same broken state. CI's "3 failed" almost certainly includes the catalog test; the run's `test (3.12)` leg
was still in progress when this plan was written, so the job log could not be retrieved to confirm the other two, which
are expected to be the two failures the previous plan already routed elsewhere (see Out of scope).

### Two defects, not one

1. **The tests are not isolated.** Ten tests call a function whose documented job is to mutate process globals, without
   pinning those globals first.
2. **The guard reports but does not contain.** One leak cascades: later offenders report a corrupted baseline (as the CI
   message above shows), and innocent tests anywhere on the worker fail with a symptom that points nowhere near the
   cause. That cascade is exactly what cost this epic two diagnostic rounds.

Fixing only (1) leaves the next leak just as expensive to diagnose. Fixing only (2) leaves the parity test broken.

### Why a shared fixture instead of ten inline fixes

`e0acf80` fixed its one test with an inline `monkeypatch.setenv` preamble. Repeating that ten more times would repair
today's failure and leave the eleventh author to rediscover it. The tests that need isolation are exactly the tests that
load `tools/run_pytest`, and they all already import from one helper module — so the isolation can live there and be
imported once per module rather than restated once per test.

This repo already uses that pattern: `tests/test_commit_workflow_hooks.py:13` imports
`commit_artifacts_dir,  # noqa: F401 (registers artifacts_dir fixture)` from `tests/_commit_workflow_fixtures.py`.

## Steps

### 1. Pin every process global `main()` writes, in one shared autouse fixture

Add to `tests/_run_pytest_fixtures.py` an autouse fixture that pre-registers each global `main()` mutates, so
`monkeypatch`'s teardown restores it. Registering a value that is currently **absent** needs care: a bare
`monkeypatch.delenv(name, raising=False)` records nothing when the name is missing, so a later `os.environ[name] = ...`
would still survive teardown. Set-then-delete registers both halves and undoes in the right order:

```python
current = os.environ.get(name)
monkeypatch.setenv(name, "" if current is None else current)
if current is None:
    monkeypatch.delenv(name, raising=False)
```

Cover the full set `main()` touches on its way to `execv`, not just the two that caused this failure:

- `TMPDIR` and `SASE_PYTEST_TMP_REDIRECTED` — written by `_prepare_pytest_tmpdir()`.
- The four keys in `run_pytest.PYTEST_ENV_UNSET_KEYS` (`SASE_COMMIT_METHOD`, `SASE_COMMIT_METHOD_ALLOW_OVERRIDE`,
  `SASE_PR_NAME`, `SASE_PR_STATUS`) — popped by `_sanitize_pytest_environment()`. These are not causing a failure today
  (under `just test` the outer runner has already popped them, so the pop is a no-op), but they are the same defect and
  cost nothing to close. Being complete here is the whole lesson of this plan.
- The working directory — `main()` calls `os.chdir(REPO_ROOT)`. Pin it with `monkeypatch.chdir(os.getcwd())`. Also a
  no-op in practice, because `REPO_ROOT` is the directory pytest already runs from; pin it so it stays one.

Import the fixture into every module that calls `load_run_pytest()` — `test_run_pytest_main.py`,
`test_run_pytest_scoped.py`, `test_run_pytest_health.py`, `test_run_pytest_tmpdir.py`, `test_run_pytest_workers.py`,
`test_run_pytest_command.py` — with the `# noqa: F401` comment form used at `tests/test_commit_workflow_hooks.py:13`.
Include `_workers` and `_command` even though they do not leak today: the rule "every `run_pytest` test module imports
the isolation fixture" is greppable and uniform, and the per-test cost is a handful of `os.environ.get` calls.

Do not derive the key list by calling `load_run_pytest()` from the fixture — that re-execs the module on every test.
Duplicate the tuple, and add a contract test asserting `set(run_pytest.PYTEST_ENV_UNSET_KEYS)` is a subset of the
fixture's list so the two cannot drift apart silently.

**Do not change `tools/run_pytest`.** Mutating `os.environ` and the CWD is `main()`'s contract, exercised for real by
every `just test`; the defect is tests calling it without isolation. This constraint is inherited from the previous plan
and still holds.

**Keep the explicit preamble in `test_prepare_pytest_tmpdir_honors_override`**
(`tests/test_run_pytest_tmpdir.py:46-47`). It looks redundant once the fixture exists, but it is not: it forces a
_distinct_ pre-state (`TMPDIR` to a placeholder, `SASE_PYTEST_TMP_REDIRECTED` to `"0"`) so the assertions on lines 53-55
actually prove the write happened. Under `just test` the ambient values are already the real scratch root and `"1"`, so
without the preamble those assertions would pass vacuously. Reword its comment: restoration is now the fixture's job,
and the preamble's job is establishing a pre-state.

### 2. Make the guard contain the leak it reports

In `tests/_tmp_leak_guard.py`, `check_tmp_env_leak_guard()` currently reports and returns, leaving the poisoned value in
place. Restore the baseline before calling `pytest.fail()`.

The offending test still fails loudly — this is containment, not suppression. What changes is that the blast radius
stops at the test that caused it:

- Each offender reports its own true baseline instead of the previous offender's leaked value, so N leaks read as N
  independent, correctly attributed errors rather than a chain.
- No innocent test anywhere else on the worker fails with a symptom that points nowhere near the cause.

Say so in the failure message — the reader should know the guard already repaired the environment and that the required
action is still to route the mutation through `monkeypatch`.

Extend `tests/test_tmp_leak_guard.py` to cover restoration: a test that leaks a watched var must both error _and_ leave
the variable at its pre-test value for the next test.

### 3. Extend the ordering regression test to the `main()`-driven class

`tests/test_scratch_tmpdir_leak_regression.py` pins exactly one node ID pair, which is why it passed while ten sibling
leaks shipped. Add a `main()`-driven offender —
`test_run_pytest_health.py::test_scoped_run_lands_in_the_durable_health_store` is representative — to the same nested
`runpytest_subprocess` run and assert `passed=3`.

Note what the assertion now rests on. Once step 2 restores the baseline, a regression no longer shows up as the
downstream catalog test failing; it shows up as an **error** on the offender. `assert_outcomes()` checks `errors == 0`
by default, so the test still catches it — but update the docstring, because the mechanism it pins has changed.

Do not add a nested run of the whole `tests/test_run_pytest_*.py` family. The guard already checks this property on
every test of every run; a meta-test that re-runs fifty tests to assert the same thing is redundant and slow.

### 4. Adopt the released sase-core diagnostic (still blocked)

Unchanged and still blocked from the previous plan. PyPI's latest `sase-core-rs` is `0.18.3` as of this writing, and
sase-core's release-please PR [#88](https://github.com/sase-org/sase-core/pull/88) (`chore: release v0.18.4`) is open
and unmerged. `0.18.4` carries `CommitLogIoCause`, which would have reported this failure as `ENOENT` on its first
occurrence instead of the guessed "check that TMPDIR exists and is writable" that misdirected two diagnostic rounds.

Re-check before starting; if `0.18.4` has published by then, raise the floor in `pyproject.toml` (line 46, currently
`sase-core-rs>=0.18.3,<0.19.0`), refresh `uv.lock`, and update the declared-minimum assertion in
`tests/test_sase_core_rs_telemetry_smoke_tool.py::test_declared_minimum_tracks_pyproject_dependency`. Use
`sase repo open sase-core -r "<reason>"` and only the path it prints.

If it is still unpublished, **do not merge PR #88 to unblock this** — that is a cross-repo release action for the
project owner to take. Record the blocked status in the phase notes and on bead sase-fq.8.2 and finish everything else.
This plan's exit criterion does not depend on it.

### 5. Correct the record on bead sase-fq.8.2

The bead's current note claims the root cause was "Fixed in sase-fq.8.2". That is half true and will mislead the next
reader. Add a note recording that the mechanism was correctly identified but the fix covered one of eleven affected
tests, that `main()` — not just `_prepare_pytest_tmpdir()` — is the shared source, and that the parity test was still
failing after `e0acf80` for exactly the same reason as before.

## Verification

Run all of these to completion. The previous session committed while `just check-full` was still running in the
background and never reported; that is how ten reproducible-in-2.8-seconds failures reached CI. Do not repeat it.

1. The ten-offender reproduction is clean:

   ```bash
   uv run pytest -p no:randomly -q --no-header tests/test_run_pytest_*.py
   ```

   Expect zero errors.

2. The epic's parity test survives a `main()`-driven neighbour:

   ```bash
   uv run pytest -p no:randomly -q --no-header \
     "tests/test_run_pytest_health.py::test_scoped_run_lands_in_the_durable_health_store" \
     "tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py::test_commit_completion_rows_match_shared_inventory_and_resolve"
   ```

   Expect `2 passed`.

3. `just check-full`, not `just check` — this touches `tests/conftest.py`'s guard module and lands into an epic's tree.
   Wait for it to finish and report its real result.

4. The phase's actual exit criterion, inherited from the previous plan: confirm on a **master** CI run that
   `test_commit_completion_rows_match_shared_inventory_and_resolve` passes on all three `test` legs, and that no
   `left TMPDIR ... changed after its own teardown` error appears on any leg. Name the run in the phase notes. A single
   green leg is not sufficient evidence — which worker an offender lands on is what made this look runner-specific.

## Out of scope

- **Do not weaken the parity assertion** in `test_artifact_ref_completion_catalog.py`, and do not add suppression
  patterns to `tests/_tmp_leak_guard.py`. Inherited from sase-fq.
- **Do not add a suppression or allow-list to the env-leak guard.** A test that legitimately needs to redirect `TMPDIR`
  uses `monkeypatch`.
- **Do not broaden the guard's watched-variable list.** Step 1 pins the four `PYTEST_ENV_UNSET_KEYS` at the source,
  which is where it belongs; `TMPDIR` is the one whose corruption silently poisons unrelated tests, and it is already
  watched. Adding the others to the guard would trade a real signal for noise.
- `tests/test_contract_manifest.py::test_contract_set_serial_runtime_stays_within_budget` — routed to epic sase-fp by
  the parent plan.
- `tests/ace/tui/modals/test_artifact_files_modal_copy.py::test_artifact_file_modal_copy_anchors_pdf_markdown_source_path`
  — a clipboard-content mismatch unrelated to `TMPDIR`, already tracked on bead `sase-ct` (corroborated there during the
  previous session rather than filed as a duplicate).
- Any further change to the commit-log budget.
