---
tier: epic
title: Restore master CI to green after the sase-core 0.18 skew and the parallelism
  restoration
goal: 'Every job in the sase CI workflow passes on master again, each of the six independent
  root causes behind the current failure is fixed at its source rather than suppressed,
  and CI regains the guarantee that source lanes actually test the sase-core wheel
  built from sase-core master.

  '
phases:
- id: core-window
  title: Bump the published sase-core-rs window to 0.18.1
  depends_on: []
  size: small
  description: 'core-window: raise the pyproject sase-core-rs constraint (and uv.lock)
    from >=0.17.15,<0.18.0 to >=0.18.1,<0.19.0 so the bead relocation binding sase
    already calls is guaranteed present.'
- id: symvision-import
  title: Give progress_fingerprint an import symvision can see
  depends_on: []
  size: small
  description: 'symvision-import: make commit_finalizer.py import progress_fingerprint
    explicitly instead of reaching it through a module alias, so the symvision lint
    stage stops reporting it as an unused public symbol.'
- id: sidecar-git-identity
  title: Configure a git identity on the sidecar clone in the git-sync fixtures
  depends_on: []
  size: small
  description: 'sidecar-git-identity: set user.name/user.email on the sidecar clone
    built by setup_repo so tests that commit there stop failing with exit 128 on runners
    where git cannot auto-detect an identity.'
- id: uv-harness-tmpdir
  title: Stop the real-uv harness leaking lock files into the watched temp root
  depends_on: []
  size: small
  description: 'uv-harness-tmpdir: give the uv_env fixture its own TMPDIR under tmp_path
    so real uv subprocesses stop dropping uv-setuptools-*.lock into the managed SASE
    temp root and tripping the session temp-leak guard.'
- id: ci-wheel-pin
  title: Keep CI's prebuilt core wheel installed for every just recipe in a job
  depends_on: []
  size: medium
  description: 'ci-wheel-pin: stop later just recipes from silently re-resolving sase-core-rs
    back to a published wheel, so source lanes really do test the sase-core commit
    that build-core built, and add CI-shape coverage that locks the behavior in.'
- id: core-commit-budget
  title: Fix the silent 2s commit-log budget in sase-core
  depends_on: []
  size: medium
  description: 'core-commit-budget: replace the hard, silently-empty two-second git
    log budget in the artifact-ref commit inventory with a generous and overridable
    one, land it in sase-core, and get a release published.'
- id: commit-budget-adopt
  title: Adopt the released commit-budget fix and stabilize the parity test
  depends_on:
  - core-window
  - core-commit-budget
  size: small
  description: 'commit-budget-adopt: raise the sase-core-rs floor to the release carrying
    the commit-budget fix and confirm the commit-completion parity test is stable
    under CI-like load.'
proposed_by: bbugyi200.athena.tq
create_time: 2026-08-05 21:05:31
status: done
bead_id: sase-fq
---

- **PROMPT:** [prompts/202608/ci_master_red_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/ci_master_red_recovery.md)
- **BEAD:** [sase-fq](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fq/README.md)

# Plan: Restore master CI to green after the sase-core 0.18 skew and the parallelism restoration

## Context

The CI workflow for `sase-org/sase` is failing on master at commit `01398f5af` ("fix(ace): stop the beads detail pane
from oscillating between two layouts").

Failing run: <https://github.com/sase-org/sase/actions/runs/31057603842>

Six jobs fail:

| Job                            | Failing step                                                 |
| ------------------------------ | ------------------------------------------------------------ |
| `published-core-minimum-smoke` | Check every required binding exists in the published minimum |
| `lint`                         | Lint                                                         |
| `test (3.12)`                  | Run tests (coverage leg)                                     |
| `test (3.13)`                  | Run tests                                                    |
| `test (3.14)`                  | Run tests                                                    |
| `perf-floors`                  | Run slow tests                                               |

All three `test` legs fail with the **same five tests**, on every Python version, so none of this is flaky:

```
FAILED tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py::test_commit_completion_rows_match_shared_inventory_and_resolve
FAILED tests/test_bead/test_sync_conflict_recovery.py::test_concurrently_minted_bead_id_relocates_instead_of_wedging_sync
FAILED tests/test_check_sase_core_rs_bindings_tool.py::test_dev_extension_exposes_every_collected_name
FAILED tests/agents_sync/test_publication_repair_git.py::test_repair_commits_and_pushes_a_resigned_snapshot
FAILED tests/agents_sync/test_publication_repair_git.py::test_repair_is_a_noop_when_nothing_has_drifted
```

Investigation found **six independent root causes**. They are unrelated to each other and to the commit CI happened to
stop on; three of them are latent defects that two recent master commits exposed. Nothing here is a real regression in
`01398f5af`.

### Root cause summary

| #   | Cause                                                                                                             | Jobs it breaks                                            |
| --- | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| R1  | `pyproject.toml` still pins `sase-core-rs>=0.17.15,<0.18.0`, but sase calls a binding first published in `0.18.1` | `published-core-minimum-smoke`, all `test` legs (2 tests) |
| R2  | `just test` re-resolves and **downgrades** the core wheel CI just built                                           | all `test` legs (mechanism for R1's test-lane failures)   |
| R3  | symvision cannot see `progress_fingerprint`'s only consumer                                                       | `lint`                                                    |
| R4  | The real-uv harness leaks a lock file into the watched temp root                                                  | `perf-floors`                                             |
| R5  | The git-sync fixtures never give the sidecar clone a git identity                                                 | all `test` legs (2 tests)                                 |
| R6  | sase-core's artifact-ref commit inventory has a hard, silently-empty 2s `git log` budget                          | all `test` legs (1 test)                                  |

### Evidence for R1 — stale sase-core-rs window

`src/sase/core/bead_conflict_facade.py:25` calls `require_rust_binding("bead_merge_event_streams_with_relocation")`.
That binding was added to sase-core in `1370830` ("fix(bead): relocate duplicate bead ids instead of failing the merge")
and first published in **sase-core-rs 0.18.1**.

`pyproject.toml:46` still says:

```
"sase-core-rs>=0.17.15,<0.18.0",
```

So:

- `published-core-minimum-smoke` installs the exact declared minimum (`0.17.15`) and `tools/check_sase_core_rs_bindings`
  correctly reports
  `sase_core_rs 0.17.15 is missing 1 of 248 required binding(s): bead_merge_event_streams_with_relocation`.
- `test_dev_extension_exposes_every_collected_name` fails the same way.
- `test_concurrently_minted_bead_id_relocates_instead_of_wedging_sync` fails because the facade's
  `except AttributeError` fallback silently degrades to the legacy plain merge, which reproduces the historical
  duplicate-creation error:
  `semantic bead conflict resolution failed: validation: duplicate issue_created event for seed-2`.

This was confirmed by downgrading a working checkout's `sase-core-rs` to `0.17.16` and re-running the three tests: the
two binding-dependent ones reproduce exactly, and restoring `0.18.1` clears them.

Bumping the window is safe. sase-core `0.18.0` was a breaking release (`feat!: remove prompt xprompt core bindings`,
removing `prompt_xprompt_records_parse`, `prompt_xprompt_records_select`, and `prompt_xprompt_rewrite_links`). A
repo-wide grep over `src/`, `tools/`, and `tests/` finds **zero** references to any of those three names, so nothing in
sase depends on what `0.18.0` removed.

### Evidence for R2 — CI silently discards the wheel it built

`.github/actions/setup-sase/action.yml` sets `SASE_CORE_WHEEL` **only for its own "Install dependencies" step**. Every
later step in the job runs another `just` recipe, and every `just` recipe depends on `_setup`. On that second entry
`SASE_CORE_WHEEL` is unset and there is no `../sase-core` checkout, so `_core-overrides-arg` (Justfile:55) emits
nothing, and `_setup`'s final `uv pip install --no-sources ... -e ".[dev]"` re-resolves `sase-core-rs` strictly inside
the `pyproject.toml` window. The run log proves the downgrade:

```
test (3.13)  Install dependencies  [install] Installing prebuilt sase_core_rs wheel from .../sase_core_rs-0.18.1-cp312-abi3-manylinux_2_39_x86_64.whl.
test (3.13)  Install dependencies   + sase-core-rs==0.18.1 (from file:///.../sase_core_rs-0.18.1-...whl)
test (3.13)  Run tests              - sase-core-rs==0.18.1 (from file:///.../sase_core_rs-0.18.1-...whl)
test (3.13)  Run tests              + sase-core-rs==0.17.16
```

`build-core` builds the wheel from sase-core master specifically so source lanes test that commit — the job even records
and echoes the sase-core SHA. That guarantee is currently void for every source lane (`test`, `lint`, `perf-floors`,
`visual-test`). Fixing R1 alone would hide this: the lane would then resolve to the _published_ `0.18.1` instead of the
built wheel, so the symptom disappears while the defect remains and sase-core master-only changes stay untested.

### Evidence for R3 — symvision and the module-alias call

`lint` fails at the symvision stage:

```
Unused public functions/classes. ...
  progress_fingerprint in src/sase/llm_provider/commit_finalizer_git.py
```

`progress_fingerprint` (added by `840cdff10`, "fix(commit-finalizer): break async-wait deadlock in finalizer passes")
does have a real consumer — `src/sase/llm_provider/commit_finalizer.py:258` and `:300` call
`finalizer_git.progress_fingerprint(dirty_state)`, reached through the
`from . import commit_finalizer_git as finalizer_git` alias on line 16. symvision counts imports, not alias attribute
calls at a call site, so it sees no consumer.

The sibling `normalize_path` is _also_ reached through the same alias (`commit_finalizer.py:63`) and is _not_ flagged,
because other modules (`src/sase/_linked_repo_paths.py` and friends) import it by name. That contrast confirms the
mechanism.

This reproduces locally, so it is not CI-specific, and it also failed in the previous master run (`4330fd0d5`).

### Evidence for R4 — the real-uv harness temp leak

`perf-floors` fails its `just test-slow` step even though every test passes (`9 passed, 3 skipped`). The session
temp-leak guard fails the run:

```
============================= system temp leakage ==============================
The test suite left new entries in a watched temp directory:
  /var/tmp/sase-d1260045 (1 new):
    - uv-setuptools-50532de7f8ab28ae.lock
```

`tests/uv_tool/test_real_uv_harness.py`'s `uv_env` fixture builds its child environment with `env = dict(os.environ)`
and overrides only `UV_TOOL_DIR`, `UV_TOOL_BIN_DIR`, and `UV_LINK_MODE`. It therefore inherits the `TMPDIR` that
`tools/run_pytest` redirected to the managed SASE temp root, and real `uv` subprocesses drop their build-backend lock
straight into the directory `tests/_tmp_leak_guard.py` watches.

This is a **latent** leak, newly exposed: `9672c5602` ("fix(tests): stop CI worker collapse and drop visual from default
lane") added the `just test-slow` step to `perf-floors`, and its own commit message notes "the slow lane previously ran
in no CI job at all".

Confirmed by running `just test-slow` locally: every test passes and the recipe still exits 1, having left two such
locks behind.

```
The test suite left new entries in a watched temp directory:
  /var/tmp/sase-75285096 (2 new):
    - uv-setuptools-77f0e93df0d45cbb.lock
    - uv-setuptools-dcaba95409a85f48.lock
...
10 passed, 2 skipped in 60.63s (0:01:00)
error: recipe `test-slow` failed on line 339 with exit code 1
```

The only tests in that lane that shell out to real `uv` are the `tests/uv_tool/test_real_uv_harness.py` ones, which pins
the source.

### Evidence for R5 — the sidecar clone has no git identity

`tests/agents_sync/git_sync_fixtures.py::setup_repo` configures `user.name` and `user.email` on the **seed** repo only
(lines 38-39). The `sidecar` is produced by `git clone` (line 51), and `git clone` does not copy the source repo's local
config, so the sidecar has no identity at all.

`tests/agents_sync/test_publication_repair_git.py` (added yesterday by `2a9627bc0`, "fix(agents-sync): repair stale
hood-snapshot digests and add drift check") is the first test to commit directly in the sidecar. On a GitHub runner git
cannot auto-detect an identity, so the commit dies with exit 128. The failure first appears in run `31054860115` (commit
`4330fd0d5`), the first completed master run after `2a9627bc0`.

Reproduced locally by denying git its identity guess:

```bash
HOME=$(mktemp -d) GIT_CONFIG_GLOBAL=/dev/null GIT_CONFIG_SYSTEM=/dev/null \
  GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=user.useConfigOnly GIT_CONFIG_VALUE_0=true \
  .venv/bin/python -m pytest tests/agents_sync/test_publication_repair_git.py -q
# 2 failed — both with `git commit` exit status 128
```

### Evidence for R6 — the 2s commit-log budget (hypothesis, must be confirmed)

`test_commit_completion_rows_match_shared_inventory_and_resolve` fails with:

```
AssertionError: assert () == ('@commit:sase-core@16bc0cefac7c', '@commit:sase@6e833b27e730')
```

Both the raw LSP inventory and the prompt completion rows come back **empty**.

This one is _not_ a core-version problem: the test passes locally against both `sase-core-rs` `0.17.16` and `0.18.1`.

Both sequences originate in the Rust binding `artifact_ref_payload_inventory`
(`src/sase/ace/tui/widgets/prompt_commit_inventory.py:56`). In sase-core, `append_commit_candidates`
(`crates/sase_core/src/editor/completion.rs`) skips a repository only when it is an SDD sidecar, has no checkout path,
has no `.git` entry, or when `commit_log_output` returns `None`. The test's three repositories are freshly `git init`-ed
with one commit each and the test's own `git` invocations all succeed with `check=True`, so the first three conditions
cannot apply to all of them.

`commit_log_output` spawns `git log` and enforces a hard wall-clock budget:

```rust
const ARTIFACT_REF_COMMIT_TIMEOUT: Duration = Duration::from_secs(2);
...
Ok(None) | Err(_) => {
    let _ = child.kill();
    let _ = child.wait();
    return None;
}
```

On expiry (or on a failed `tempfile::tempfile()` in the same `TMPDIR`) it returns `None` and that repository silently
contributes **zero** rows — which is exactly the observed `()`.

The timing correlation is strong. `9672c5602` fixed the suite gate's flat 4-CPU reserve, which had been collapsing a
4-vCPU runner to a single worker. Run `31057603842` is the first _completed_ master run carrying that fix, and it is the
first run in which this test fails. The wall-clock evidence for the parallelism jump:

| Run                            | 3.12 leg duration |
| ------------------------------ | ----------------- |
| `31054860115` (before the fix) | 3282s             |
| `31057603842` (after the fix)  | 1405s             |

A ~2.3x speed-up from restored parallelism on a 4-vCPU runner means heavy oversubscription, and a cold `git` fork/exec
can exceed a 2s budget under it.

**This remains a hypothesis.** An attempt to reproduce locally by pinning the test and twelve CPU hogs to two cores
slowed the test from 0.86s to ~4.9s but did not make it fail. The `core-commit-budget` phase must confirm the mechanism
before changing behavior, and must not simply retry-loop the assertion.

## Phases

### Bump the published sase-core-rs window to 0.18.1

Raise the constraint in `pyproject.toml:46` from `sase-core-rs>=0.17.15,<0.18.0` to `sase-core-rs>=0.18.1,<0.19.0`, then
refresh `uv.lock` (it currently records both the old specifier at line 2009 and a resolved `sase-core-rs` `0.17.15` at
line 2038).

Before landing, re-confirm the breaking-change audit for sase-core `0.18.0`: grep the whole repo for
`prompt_xprompt_records_parse`, `prompt_xprompt_records_select`, and `prompt_xprompt_rewrite_links` and verify there are
still no hits.

Verify:

- `just install && just check`
- `tools/check_sase_core_rs_bindings` against a venv holding exactly the new minimum, mirroring the
  `published-core-minimum-smoke` job:

  ```bash
  core_minimum="$(python3 tools/smoke_sase_core_rs_telemetry --print-minimum pyproject.toml)"
  uv venv --python 3.12 /tmp/published-core-smoke
  uv pip install --python /tmp/published-core-smoke/bin/python "sase-core-rs==${core_minimum}"
  /tmp/published-core-smoke/bin/python tools/smoke_sase_core_rs_telemetry
  /tmp/published-core-smoke/bin/python tools/check_sase_core_rs_bindings
  ```

- `tests/test_check_sase_core_rs_bindings_tool.py` and `tests/test_bead/test_sync_conflict_recovery.py` pass.

If a newer sase-core release exists by the time this phase runs, pin to the newest `0.18.x` that is actually published
rather than inventing a version.

### Give progress_fingerprint an import symvision can see

In `src/sase/llm_provider/commit_finalizer.py`, add `progress_fingerprint` to the existing explicit
`from .commit_finalizer_git import (...)` block (line 25) and call it directly at lines 258 and 300 instead of through
the `finalizer_git.` alias. Keep the import list sorted the way the surrounding block already is.

Do not add a symvision pragma and do not make the symbol private: the consumer is a real non-test Python file in this
repo, so the fix per the symvision guidance is to give symvision an import it can resolve. Do not touch the installed
symvision package.

Verify with `just _lint-symvision` (it must print nothing and exit 0), then `just check`.

### Configure a git identity on the sidecar clone in the git-sync fixtures

In `tests/agents_sync/git_sync_fixtures.py::setup_repo`, configure `user.name` and `user.email` on the `sidecar` clone
after line 51, the same way the seed repo already does. Prefer fixing it once in the shared fixture over patching
individual tests, since every git-sync suite builds its sidecar through this helper.

While here, check whether any other repository this module creates can be committed to without an identity and give it
the same treatment.

Verify that the tests fail before the change and pass after it, under an environment where git refuses to guess an
identity:

```bash
HOME=$(mktemp -d) GIT_CONFIG_GLOBAL=/dev/null GIT_CONFIG_SYSTEM=/dev/null \
  GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=user.useConfigOnly GIT_CONFIG_VALUE_0=true \
  .venv/bin/python -m pytest tests/agents_sync -q
```

Then run `just check`.

### Stop the real-uv harness leaking lock files into the watched temp root

In `tests/uv_tool/test_real_uv_harness.py`, give the `uv_env` fixture's child environment its own temp directory under
`tmp_path` (create it, then set `TMPDIR` — and any other temp variable `uv` honours on this platform — in the returned
`env`) so that real `uv` subprocesses can no longer write into the managed SASE temp root.

Check whether pointing `UV_CACHE_DIR` at `tmp_path` as well is warranted; note the existing fixture comment explaining
that `UV_LINK_MODE=copy` is set precisely because `tmp_path` and the uv cache usually sit on different filesystems, so
weigh the runtime cost of a cold per-test cache before moving it.

Do not silence this by adding `uv-setuptools-*` to `FOREIGN_ENTRY_PATTERNS` in `tests/_tmp_leak_guard.py` or by setting
`SASE_TMP_LEAK_GUARD_DISABLED`; the guard is reporting a genuine leak.

Verify with `just test-slow` — it must finish with no "system temp leakage" section — then `just check`.

### Keep CI's prebuilt core wheel installed for every just recipe in a job

Make the wheel `build-core` produced stay installed for the whole job, not just the install step. The minimal change
consistent with the existing design is for `.github/actions/setup-sase/action.yml` to export `SASE_CORE_WHEEL` to
`$GITHUB_ENV` rather than scoping it to one step, so every later `just` recipe re-enters `_setup` with the variable set.
`_setup` then reinstalls that exact wheel and `_core-overrides-arg` (Justfile:55) emits the `--overrides` file that
lifts the published version window, so the editable install can no longer resolve `sase-core-rs` backwards.

Confirm the reinstall on each recipe entry is cheap enough for the lanes that invoke `just` many times (`lint` runs at
least five recipes); if it is not, make `_setup` skip the reinstall when the installed distribution already matches the
wheel, but do not drop the overrides file — the overrides are what actually prevent the downgrade.

Add coverage in `tests/test_github_actions_ci.py` (which already asserts CI workflow shape) locking in that the core
wheel stays pinned for the whole job, so this silent substitution cannot come back.

Verify by re-reading a subsequent CI run's `Run tests` step log and confirming it no longer contains a
`+ sase-core-rs==<published version>` line replacing the file-URL wheel, plus `just check` locally.

Note the interaction with `core-window`: these two phases fix different halves of the same story and neither subsumes
the other. `core-window` makes the declared floor honest for published installs; this phase makes source lanes test the
wheel built from sase-core master. Land both.

### Fix the silent 2s commit-log budget in sase-core

This phase crosses the Rust core backend boundary. Open the sase-core checkout with
`sase repo open sase-core -r "<reason>"` and use only the path it prints.

First **confirm the R6 mechanism** before changing anything. The local stress attempt described above did not reproduce
the failure, so establish the cause directly — for example by instrumenting `commit_log_output` to report why it
returned `None` (timeout versus `tempfile` failure versus non-zero `git` exit) and exercising it under a CPU- and
IO-constrained environment closer to a 4-vCPU runner, or by adding a sase-core test that drives
`append_commit_candidates` against an artificially slow `git`. If the evidence points somewhere else entirely, stop and
report that instead of proceeding.

Assuming the budget is confirmed, fix it in `crates/sase_core/src/editor/completion.rs`:

- The failure mode is worse than a slow completion — an expired budget silently yields an empty inventory, so on a
  loaded machine a user typing `@commit:` gets no completions and no indication why. Whatever design is chosen must not
  keep that silent-empty behaviour as the ordinary outcome under load.
- Raise the default budget to something that survives a heavily oversubscribed host, and make it overridable (an
  environment variable is the cheapest lever for both CI and the test suite).
- Add sase-core test coverage for the new behaviour and for the override.

Land the change in sase-core through its normal review flow and get a release published (release-please cuts sase-core
releases on merge to master). Record the released version in the phase notes — `commit-budget-adopt` needs it.

Do not work around this on the sase side by loosening the assertion in
`tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py` or by retrying until the inventory is non-empty. The
parity assertion is the point of that test.

### Adopt the released commit-budget fix and stabilize the parity test

Once `core-commit-budget` has a published release, raise the `sase-core-rs` floor in `pyproject.toml` to that version
(keeping the `<0.19.0` ceiling unless the release crossed a major line) and refresh `uv.lock`. This replaces the floor
`core-window` set; only one bump ships if both phases are ready at the same time.

If the sase-core fix introduced an environment override for the commit-log budget, decide whether the sase test suite
should set it explicitly — an explicit generous budget in the fixture is preferable to relying on a default that happens
to be large enough.

Verify `tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py` passes under CI-like contention, not just on an
idle machine: run it repeatedly with the process pinned to a small CPU set while background load saturates those cores,
and with the full suite running in parallel.

Then run `just check` and confirm on the next master CI run that all six previously failing jobs are green.

## Out of scope

- The `RuntimeWarning: coroutine 'Timer._run_timer' was never awaited` and "changed the process working directory"
  warnings visible in the test logs. They are pre-existing warnings, not failures, and none of the six failing jobs
  depends on them.
- Any change to the beads detail pane work in `01398f5af`. That commit is where CI happened to stop; it did not cause
  any of these failures.
