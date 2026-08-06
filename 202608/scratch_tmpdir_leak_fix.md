---
tier: tale
title: Stop a contract test from leaking TMPDIR into every later test on its xdist worker
goal:
  test_commit_completion_rows_match_shared_inventory_and_resolve passes on all three test legs of a master CI run,
  because no test can silently redirect the process TMPDIR at a directory pytest is about to delete, and sase-core's
  commit-inventory diagnostic reports the real errno instead of guessing at TMPDIR.
proposed_by: bbugyi200.athena.tw
create_time: 2026-08-06 08:46:37
status: done
---

- **PROMPT:**
  [prompts/202608/scratch_tmpdir_leak_fix.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/scratch_tmpdir_leak_fix.md)

# Plan: Stop a contract test from leaking TMPDIR into every later test on its xdist worker

## Context

This is phase `scratch-fix` (bead sase-fq.8.2) of the epic plan
[202608/artifact_ref_scratch_failure.md](202608/artifact_ref_scratch_failure.md).

Phase `scratch-probe` (sase-fq.8.1) landed a resource probe on PR #278 and CI answered the question on the first run.
The answer is **none of the three candidates** the parent plan predicted. It is not `EMFILE`, not `ENOSPC`, and not an
`O_TMPFILE` quirk. It is test pollution, and it is reproducible locally in under a second.

### What the probe reported

From run [31097887770](https://github.com/sase-org/sase/actions/runs/31097887770), on all three `test` legs:

```
scratch-probe: resource state at empty commit inventory
  TMPDIR env: /var/tmp/sase-d1260045/pytest-of-runner/pytest-0/popen-gw2/test_prepare_pytest_tmpdir_hon0/pytest scratch
  tempfile.gettempdir(): /var/tmp/sase-d1260045
  tmpdir state: exists=True is_dir=True writable=True executable=True
  RLIMIT_NOFILE: soft=65536 hard=65536
  open descriptors: 20
  tempfile.TemporaryFile(): ok (fd=20)
  os.dup() of a scratch fd: ok
```

Two lines settle it:

- `TMPDIR` is **not** the managed scratch root. It points inside the `tmp_path` of a completely different test,
  `test_prepare_pytest_tmpdir_honors_override`, whose directory name (`pytest scratch`) is unmistakable.
- Descriptors and free space are healthy, and Python's own `tempfile.TemporaryFile()` and `os.dup()` both succeed.

The leg-to-leg variation is only _which worker_ the leaking test landed on: `popen-gw2` on 3.12 and 3.13, `popen-gw1` on
3.14. The catalog test fails whenever it draws the same worker, which is why this looked runner-specific.

### The mechanism

`tests/test_run_pytest_tmpdir.py::test_prepare_pytest_tmpdir_honors_override` calls the real `_prepare_pytest_tmpdir()`,
and that function's job is to mutate the process environment (`tools/run_pytest:405-407`):

```python
os.environ["TMPDIR"] = str(scratch_root)
os.environ[PYTEST_TMP_REDIRECTED_ENV] = "1"
```

The test monkeypatches `SASE_PYTEST_TMPDIR`, but never `TMPDIR` or `SASE_PYTEST_TMP_REDIRECTED`. `monkeypatch` only
restores variables it set itself, so both writes survive teardown for the rest of that worker's session.
`pyproject.toml` sets `tmp_path_retention_policy = "failed"`, so pytest then deletes the passing test's `tmp_path` — and
`TMPDIR` is left pointing at a directory that no longer exists.

Python never notices, because `tempfile.gettempdir()` caches its answer on first use and had already resolved the real
scratch root. That is exactly why the probe's own `tempfile.TemporaryFile()` came back `ok`, and why every earlier
inspection of `TMPDIR`'s health looked fine. Rust does notice: `tempfile::tempfile()` re-reads `$TMPDIR` on every call,
gets `ENOENT`, and `commit_log_output` turns that into `CommitLogFailure::Scratch` — whose message discards the errno
and guesses "check that TMPDIR exists and is writable". The guess sent this investigation down the wrong path for
several rounds.

The parent plan's "What is already ruled out" section is wrong on exactly this point. It ruled out a bad `TMPDIR` on the
grounds that "no conftest fixture rewrites or deletes it" — true, and irrelevant. A _test_ rewrote it.

### Local reproduction

Deterministic, no xdist, 0.9 seconds:

```bash
uv run pytest -p no:randomly -x -q \
  "tests/test_run_pytest_tmpdir.py::test_prepare_pytest_tmpdir_honors_override" \
  "tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py::test_commit_completion_rows_match_shared_inventory_and_resolve"
```

The second test fails with the identical `assert () == (...)` shape and the identical sase-core stderr message seen on
CI. Routing `TMPDIR` and `SASE_PYTEST_TMP_REDIRECTED` through `monkeypatch.setenv` before the `_prepare_pytest_tmpdir()`
call makes both tests pass; that fix shape is confirmed, not hypothesized.

### Why this is worth a guard, not just a two-line fix

The leak is invisible by construction. It corrupts only _later_ tests, only on the same worker, only through non-Python
code that re-reads `$TMPDIR`, and it took two CI-diagnostic rounds to find. Nothing in the suite would stop the next
test that calls a real environment-mutating helper from reintroducing it. `tests/_tmp_leak_guard.py` watches for leaked
temp _entries_ and would not have caught this.

There is also a second, quieter symptom worth closing off: the same test leaks `SASE_PYTEST_TMP_REDIRECTED=1`, which is
the flag that arms `_tmp_leak_guard`. A plain `pytest` run that was meant to leave the guard inert can have it switched
on mid-session by an unrelated test.

## Steps

### 1. Fix the leak at its source

In `tests/test_run_pytest_tmpdir.py::test_prepare_pytest_tmpdir_honors_override`, route both variables
`_prepare_pytest_tmpdir()` writes through `monkeypatch.setenv` _before_ calling it, so teardown restores the real
values. Keep the existing assertions on lines 46-49 — they are what pins the redirect behavior and they must still pass.

Do not change `tools/run_pytest`. Mutating `os.environ` is `_prepare_pytest_tmpdir()`'s contract, exercised for real by
every `just test` invocation; the defect is a test calling it without isolation.

### 2. Add a suite-wide guard that fails a leak instead of hiding it

Add an autouse fixture (or an equivalent `pytest_runtest_*` hook alongside the existing guards in `tests/conftest.py`)
that snapshots `TMPDIR` and `SASE_PYTEST_TMP_REDIRECTED` before each test and fails the test that leaves either one
changed. Name the offending test and both values in the failure message.

Requirements:

- It must fail the **leaking** test, not some innocent later one. A session-finish check would name the wrong culprit.
- Keep it cheap. This runs 25k+ times per leg; two `os.environ.get` calls and a comparison, no filesystem work.
- Do not add a suppression list. If a test legitimately needs to redirect `TMPDIR`, it can use `monkeypatch`.

Cover the guard itself with a test proving it fails a deliberately leaking test (pytester, or the harness already used
for suite-level guards in this repo).

### 3. Add the ordering regression test

Turn the local reproduction into a regression test: the catalog parity test must still pass when
`test_prepare_pytest_tmpdir_honors_override` has already run in the same process. An in-process `pytester` run of the
two node IDs in sequence is the honest shape — asserting on `os.environ["TMPDIR"]` after the fact only re-tests step 1.

Note that the reproduction's value does not depend on the catalog test specifically. Any test that reaches non-Python
code through `TMPDIR` would do; the catalog test is simply the one that exposed it.

### 4. Adopt the released sase-core diagnostic

The sase-core half of this phase already landed: `7b28c3e` ("fix(editor): report the OS error behind a dropped
commit-log repository") is on sase-core master, carrying `CommitLogIoCause` so `Scratch` reports the real errno and
names `open` versus `dup` apart. It is **not yet released** — PyPI's latest `sase-core-rs` is `0.18.3`, and sase-core's
release-please PR [#88](https://github.com/sase-org/sase-core/pull/88) (`chore: release v0.18.4`) is open and unmerged.

So: merge sase-core PR #88, wait for `0.18.4` to publish, then in this repo raise the floor in `pyproject.toml`, refresh
`uv.lock`, and update the declared-minimum assertion in
`tests/test_sase_core_rs_telemetry_smoke_tool.py::test_declared_minimum_tracks_pyproject_dependency` (currently pinned
to `0.18.3`). Record the released version in the phase notes.

Use `sase repo open sase-core -r "<reason>"` and only the path it prints. Note that master already moved the floor to
`0.18.3` for an unrelated reason (sase-fr.9.3, commit `6b0976bcb`), so this is a `0.18.3` → `0.18.4` bump.

With the errno carried, this failure would have read as `ENOENT` on the first CI occurrence instead of costing two
diagnostic rounds. That is the value being banked here, independent of the leak fix.

### 5. Decide the probe's fate deliberately

`tests/_scratch_resource_probe.py` and `tests/ace/tui/widgets/test_artifact_ref_scratch_probe.py` were built to answer a
question that is now answered. Keeping them is defensible — the report is cheap, fires only on an already-failing
assertion, and would speed up the next scratch-shaped failure. Removing them is also defensible now that step 2's guard
catches the specific cause and step 4 makes sase-core report the errno itself.

Make the call explicitly and say why in the commit message. Do not leave it undecided. If the probe stays, its module
docstring must be corrected: it currently asserts that "TMPDIR demonstrably exists and is writable", which is precisely
the wrong conclusion this phase overturned.

### 6. Correct the record

The `PROPOSED FOLLOW-UP` / notes for this phase should state the real cause plainly, so the next reader does not
re-derive it. Also add a note on bead `sase-fq.8.2` recording that the parent plan's `EMFILE`/`ENOSPC`/`O_TMPFILE`
candidate list was wrong and why the "TMPDIR is fine" reasoning failed (cached `gettempdir()` masked it from every
Python-side check).

## Verification

- The local reproduction command above passes.
- `just check-full`, not `just check`: this lands into an epic's tree and touches `tests/conftest.py`.
- Confirm on a **master** CI run that `test_commit_completion_rows_match_shared_inventory_and_resolve` passes on all
  three `test` legs. Name the run in the phase notes. This is the phase's actual exit criterion — the failure is
  worker-assignment dependent, so a single green run on one leg is not sufficient evidence.

## Out of scope

- **Do not weaken the parity assertion** in `test_artifact_ref_completion_catalog.py`, and do not add suppression
  patterns to `tests/_tmp_leak_guard.py`. Both constraints are inherited from sase-fq.
- `tests/test_contract_manifest.py::test_contract_set_serial_runtime_stays_within_budget` (failed the 3.12 leg at 32.7s
  against a 30s budget) — already routed to epic sase-fp by the parent plan.
- `tests/ace/tui/modals/test_artifact_files_modal_copy.py::test_artifact_file_modal_copy_anchors_pdf_markdown_source_path`
  — also failed the 3.12 leg, on `gw0`, with a clipboard-content mismatch unrelated to `TMPDIR` (the leak was on `gw2`
  in that job). Not caused by this work and not fixed by it. It will block sase-fq.8.3's "every job green on master"
  criterion, so file a task bead through `/sase_new_task` rather than absorbing it here.
- Any further change to the commit-log budget. 30s with an override is fine and was never what failed.
