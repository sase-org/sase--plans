---
tier: tale
title: Fast per-SHA master gate
goal:
  Every push to sase-org/sase master gets its own never-cancelled CI run that verifies
  the whole fast test suite plus the lint gate in balanced shards, reaches a terminal
  conclusion in single-digit minutes, and gets its Rust core from a SHA-keyed wheel
  cache instead of a nine-minute source build.
size: medium
proposed_by: bbugyi200.athena.sase-um.1
bead: sase-um.1
create_time: 2026-08-26 19:28:04
status: wip
---

- **PARENT:** [202608/release_gate_liveness.md](release_gate_liveness.md)
- **BEAD:**
  [sase-um.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.1.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                  | Why                                                                                                                          |
| ------------ | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| derives-from | [plan:202608/release_gate_liveness.md][1] | Parent epic. This tale implements its `gate` phase (bead sase-um.1) and records two measured deviations from the phase text. |

[1]: https://github.com/sase-org/sase--plans/blob/main/202608/release_gate_liveness.md

<!-- sase:links:end -->

# Plan: Fast per-SHA master gate

## 1. Problem

`sase-org/sase` has not cut a release in ~19 days. The parent epic's research
established three independent blockers; this tale removes the first one, **liveness**.

Master CI (`ci.yml`) takes ~107 minutes. Master commits land every ~11 minutes. The
`ci-refs/heads/master` concurrency group holds one running plus one pending run, so each
push cancels the pending one: of the last 60 master runs, **44 were cancelled, 14
failed, 0 succeeded**. Only ~13% of commits ever get a completed run, so `ci_watch`'s
"is the tip of master settled and green?" release condition is essentially never
satisfiable.

Removing the cancellations does not help. Deleting the concurrency block or setting
`queue: max` leaves the tip at the back of a growing FIFO at a 10x arrival-to-service
ratio. The fix is to apply this project's own `decisions:two-speed-verification` to CI:
add a **fast, per-SHA, never-cancelled** gate whose result is attributable to exactly
one commit, and (in later phases of the epic) move the exhaustive lane onto a schedule.

**Scope boundary.** This tale is deliberately _additive_. It adds one workflow and the
tooling it needs. It does **not** touch `ci.yml`'s triggers, jobs, or concurrency, does
not add `full.yml`, and does not change `ci_watch`. Those belong to the epic's `heavy`
and `chop` phases. The temporary overlap is nearly free: master `ci.yml` runs keep being
cancelled by the existing group, and a run cancelled while pending starts no jobs.

## 2. Deliverable

A new `.github/workflows/master-gate.yml` (`name: Master Gate`) plus the in-repo tooling
that makes its test leg a _partition_ of the fast suite rather than a selection, its
tests, and its README badge.

| File                                           | Change                                                 |
| ---------------------------------------------- | ------------------------------------------------------ |
| `.github/workflows/master-gate.yml`            | new: `core-wheel`, `lint`, sharded `test`              |
| `tests/_test_shards.py`                        | new: discovery + deterministic shard assignment        |
| `tests/shard_timings.json`                     | new: committed per-test-file duration table            |
| `tools/refresh_shard_timings`                  | new: regenerates the table from the local timing store |
| `tools/run_pytest`                             | `SASE_TEST_SHARD` support in `fast` mode               |
| `Justfile`                                     | `refresh-shard-timings` recipe                         |
| `tests/_run_pytest_fixtures.py`                | isolate `SASE_TEST_SHARD`                              |
| `tests/test_test_shards.py`                    | new: partition, balance, and staleness contracts       |
| `tests/test_run_pytest_shards.py`              | new: runner-side shard contracts                       |
| `tests/test_github_actions_ci.py`              | gate invariants; extend the anti-`test-scoped` test    |
| `README.md`                                    | fast-mode badge                                        |
| `docs/development.md`, `docs/configuration.md` | document the lane and the env var                      |

## 3. Two measured deviations from the phase text

The parent epic's `gate` phase text prescribes two specifics that the measurements below
refute. Both deviations are deliberate; do not silently revert to the phase text.

### 3.1 Start at six shards, not four

The phase says "start at N=4 ... raise N only if p50 exceeds 8 min", but its own
arithmetic already predicts the miss: it budgets ~7.3 min of test time at N=4 _plus_ ~2
min of per-shard setup, which is 9.3 min against an 8-minute acceptance criterion.
Per-shard setup does not shrink with N; only test time does.

|     N |    test time |  + setup | + core-wheel (hit) |         gate wall | job-min/commit |
| ----: | -----------: | -------: | -----------------: | ----------------: | -------------: |
|     4 |     ~6.5 min |     ~8.5 |               ~9.2 |          **miss** |            ~41 |
| **6** | **~4.3 min** | **~6.3** |           **~7.0** | **meets ≤ 8 min** |        **~49** |
|     8 |     ~3.3 min |     ~5.3 |               ~6.0 |             meets |            ~57 |

(Test time is `ci.yml`'s measured 29-minute `test (3.14)` leg less ~3 min of install,
divided by N.) N=6 meets the epic's acceptance criterion #2 on the first measurement and
stays under the epic's ≤ 60 job-min/commit guardrail: ~49 job-min × ~74 commits/day is
~3,600 of the account's 28,800 job-min/day, about 12%.

N is one workflow-level `SHARD_COUNT` value plus the matrix list, so re-tuning is a
two-line change; the tests derive N from the workflow rather than hardcoding it.

### 3.2 The duration table is committed, not fetched from the heavy lane

The phase's first-choice shard split has the heavy lane publish
`tests/_test_selection_timings` as a run artifact for the gate to download. That is not
available to this phase: the heavy lane (`full.yml`) does not exist yet and its phase
_depends on_ this one, so the download path would be dead code shipping with only its
fallback live.

Worse, a per-run fetch is a correctness hazard. Each shard job would have to resolve
"the newest table" independently; if two shards resolve different tables they compute
different partitions, and tests fall between the shards with nothing reporting it.
Avoiding that needs a separate planner job, an artifact hand-off, and `actions: read`.

**Instead**: commit the table, exactly as this repo already commits
`tests/contract_manifest.txt` and refreshes it with `tools/refresh_contract_manifest`.
The table is generated _from the same machinery the phase names_ — a merged
`tests._test_selection_timings.load_timing_table()` snapshot — so it is not a second
timing source. Every shard computes its partition from the checked-out commit, so the
partition is identical by construction with no coordination, no extra job, and no API
surface. Phase `heavy` can later merge a published artifact on top of the committed
table without touching the split algorithm.

Measured on the local table (3,399 files, 4,549 serial seconds, worst file 70.4 s), the
slowest shard relative to a perfect split:

| split                          |    N=4 |        N=6 |    N=8 |
| ------------------------------ | -----: | ---------: | -----: |
| stable hash mod N              | 1.128x |     1.240x | 1.224x |
| committed table, top 800 files | 1.002x | **1.004x** | 1.007x |

At N=6 the table is worth ~1 minute of gate wall on every commit, for a ~60 KB file.

## 4. The shard split

### 4.1 `tests/_test_shards.py`

A new module beside the existing `tests/_test_selection*.py` support modules. Public
API:

- `discover_test_files(root: Path) -> list[str]` — every `tests/**/test_*.py` path,
  repo-relative POSIX, sorted, skipping path components that start with `.` and
  `__pycache__` (which is exactly pytest's default `norecursedirs` behaviour for this
  tree). This is the _whole_ fast-suite file set; the shards partition it.
- `parse_shard_spec(value: str) -> tuple[int, int]` — parses `"<index>/<count>"`,
  1-based. Raises `ValueError` with a remedy sentence on a malformed spec, a count < 1,
  or an index outside `1..count`. The phase text's "reject a shard index outside 1..N
  loudly" is this function plus its `pytest.UsageError` wrapper in the runner.
- `assign_shards(files, *, shard_count, table) -> list[list[str]]` — the partition.
- `load_shard_timings(path) -> ShardTimings` — reads the committed table.
- `shard_summary(...) -> str` — the one-line stderr summary the runner prints.

**One algorithm, no fallback branch.** Every file gets an estimated cost: its recorded
duration if the table has one, otherwise the table's `default_seconds`, otherwise `1.0`.
Files are sorted by `(-estimate, sha256(path).hexdigest())` and greedily placed in the
currently-lightest bin (longest-processing-time-first, provably within 4/3 of optimal).
The hash tiebreak is what makes the no-table case degrade to a stable hash split rather
than to a directory-clustered one, so there is no separate fallback path to test or to
rot. Ties between equally-loaded bins break on the lower index, so the result is a pure
function of `(files, shard_count, table)`.

Invariants the module owns, and `tests/test_test_shards.py` pins:

1. **Partition.** For any `shard_count >= 1`, the concatenation of all shards, sorted,
   equals `discover_test_files()`, and the shards are pairwise disjoint. Parametrise
   over `shard_count in (1, 2, 3, 6, 7, 64)` against the real repository file set — this
   is the invariant that says the gate skips no test.
2. **Determinism.** Two calls with the same inputs return the same lists; shuffling the
   input order does not change the result.
3. **Balance.** Against the committed table, the heaviest shard's estimated cost is
   within 10% of `total / shard_count` at `shard_count = 6`. (Measured 0.4%; the 10%
   headroom keeps the test from becoming a tripwire on ordinary table drift.)
4. **Empty shards are an error, not a pass.** `assign_shards` refuses `shard_count` >
   `len(files)`.
5. **Spec parsing.** `"0/6"`, `"7/6"`, `"6"`, `"a/6"`, `"6/0"`, `"-1/6"` each raise, and
   the message names `SASE_TEST_SHARD` and the accepted form.
6. **Discovery matches pytest's collection rules.** Assert `pyproject.toml` has
   `testpaths == ["tests"]`, sets no `python_files` override, and that the tree contains
   no `*_test.py` file. If any of those changes, `discover_test_files` is no longer the
   collection set and this test says so with that remedy.

Keep the module under 700 lines (`just _lint-toobig` warns there).

### 4.2 `tests/shard_timings.json` and `tools/refresh_shard_timings`

Payload:

```json
{
  "schema": 1,
  "recorded_at": "2026-08-26T22:42:42+00:00",
  "host": "athena",
  "measured_file_count": 3399,
  "default_seconds": 0.13,
  "durations": { "tests/ace/tui/widgets/test_vim_normal_key_containment.py": 70.4 }
}
```

`tools/refresh_shard_timings` merges the local recordings with
`tests._test_selection_timings.load_timing_table()`, keeps the `--limit` (default
**800**) slowest files rounded to 0.1 s, sets `default_seconds` to the mean of the
omitted files, records `measured_file_count` as the _full_ merged table's file count,
writes the JSON with sorted keys and a trailing newline, and prints what changed.
`--check` exits non-zero without writing when the file would change; `--print-plan N`
prints the resulting per-shard file counts and estimated seconds so a tuner can see the
balance without running CI.

Truncating to 800 entries is what keeps the file ~60 KB instead of ~230 KB while costing
0.4% of balance (Section 3.2). Refuse to write a table whose top entry is absent from
disk, so an obviously stale local store cannot be committed.

Wire it up as a Justfile recipe next to `refresh-contract-manifest`:

```just
# Regenerate tests/shard_timings.json from this host's per-test-file timing
# store. The gate's shard split reads it; see docs/development.md.
refresh-shard-timings *args: _setup
    {{ venv_bin }}/python tools/refresh_shard_timings {{ args }}
```

`tools/pyscripts-260801` requires every `tools/` file to be referenced from within the
repo, which the recipe satisfies. `tools/typecheck_extensionless_tools` type-checks it.

**Staleness is a legible failure, never a silent one.** The table only affects _balance_
— a file the table has never seen still runs, at `default_seconds` — so staleness must
not be able to drop a test. `tests/test_test_shards.py` still bounds the drift:

- `abs(measured_file_count - len(discover_test_files())) / len(discover_test_files()) <= 0.20`
- at least 90% of the table's paths still exist on disk

Both failures say `run just refresh-shard-timings`. With ~3,500 files, 20% is ~700 files
of churn, so this fires on a real restructuring and not on ordinary work.

### 4.3 `tools/run_pytest`

Read `SASE_TEST_SHARD` (`"<index>/<count>"`) in `main()`, next to where `SCOPED_MODE`
resolves its selection, and follow the same shape as scoped mode: compute the file list
in the parent, print a summary to stderr, then append the files to `pytest_args` so the
mode's marker expression, worker grant, and `execv` hand-off are all unchanged.

```
shard 3/6: 567 files, 758.3 estimated serial seconds (table covers 79.4% of the suite)
```

Rules:

- **`fast` mode only.** Any other mode with `SASE_TEST_SHARD` set is a
  `pytest.UsageError`. The lane must not be silently sharded under `cov`, `cost`, or
  `cov-contexts`, whose consumers all assume whole-suite coverage.
- **No explicit selectors.** A shard plus caller-supplied test paths is ambiguous;
  `pytest.UsageError`.
- **Unset or empty means today's behaviour exactly.** Every existing caller is
  untouched.
- **A shard is not a full lane.** Suppress `_full_lane_recording_args()` when a shard is
  active. `FULL_LANE_MODES` means "runs the whole fast suite, so its failures are
  evidence about what a scoped run should not have skipped", and a shard is a quarter of
  that evidence; recording it as `KIND_FULL_RUN` would poison the selection-health
  store's false-negative metric. Add the reason as a comment where it is suppressed —
  the temptation to "just leave it on" is exactly what the comment is for.
- **Keep the timings recorder armed.** `tests._test_selection_timings` merges partial
  recordings newest-wins by design, so a sharded run contributes real per-file
  durations.
- Errors surface through the existing `pytest.UsageError` path in `main()`, which
  already prints `pytest runner configuration error: ...` and returns the usage exit
  code.

`tests/_run_pytest_fixtures.py` must clear `SASE_TEST_SHARD` in its autouse isolation
fixture. This is not cosmetic: the gate runs the suite _with that variable set_, so
without the isolation every `tools/run_pytest` test would see a live shard request and
`tests/test_run_pytest_*.py` would fail only inside the gate — the worst possible place
to discover it.

`tests/test_run_pytest_shards.py` (new, using the existing `load_run_pytest` harness)
pins: a valid spec appends exactly that shard's files; an out-of-range or malformed spec
raises with a message naming `SASE_TEST_SHARD`; a non-`fast` mode raises; a shard plus
an explicit selector raises; an unset variable leaves `pytest_args` byte-identical; and
the full-lane health recorder is suppressed while the timings recorder stays armed.

## 5. The workflow

`.github/workflows/master-gate.yml`, `name: Master Gate`. Comment it at the density of
`ci.yml` — every non-obvious choice there carries the incident that motivated it, and
this file is going to be read by whoever is debugging the release gate at 2am.

```yaml
name: Master Gate

on:
  push:
    branches: [master]
  workflow_dispatch:

# Per-SHA, never cancelled. This is the whole point: a later push must not be
# able to cancel an earlier commit's gate, because a cancelled run is exactly
# the non-answer that stalled the release for 19 days (44 of the last 60
# master ci.yml runs were cancelled, 0 succeeded).
concurrency:
  group: master-gate-${{ github.sha }}
  cancel-in-progress: false

permissions:
  contents: read

env:
  # One place to tune the split. The matrix below must list 1..SHARD_COUNT;
  # tests/test_github_actions_ci.py asserts it.
  SHARD_COUNT: "6"

jobs:
  core-wheel: ...
  lint: ...
  test: ...
```

Every job carries `timeout-minutes` (`core-wheel` 20, `lint` 15, `test` 20) so a wedged
job cannot hold an org runner slot for GitHub's six-hour default.

### 5.1 `core-wheel`

Produces the same `sase-core-wheel` artifact `ci.yml`'s `build-core` does — the abi3
wheel, `sase-xprompt-lsp`, and `sase-core-sha.txt` — but pays the ~9-minute maturin
build only on a cache miss.

1. **Resolve the revision.** `git ls-remote https://github.com/sase-org/sase-core HEAD`
   (~2 s, no checkout, public repo so no token). Assert the result is 40 hex characters
   and fail loudly otherwise; a truncated read would silently key the cache on garbage.
   Emit it as a step output.
2. **`actions/cache/restore`** (`id: core-cache`) with `path: dist` and
   `key: sase-core-wheel-${{ runner.os }}-v1-<sha>`. The `v1` salt is hand-bumped when
   the toolchain identity changes; the wheel is abi3, so it is Python-version
   independent. Cache reads on a master run see master's scope, which is where this
   workflow runs.
3. **Build steps, each `if: steps.core-cache.outputs.cache-hit != 'true'`**: check out
   `sase-org/sase-core` **at the resolved SHA** (not `HEAD` — pinning closes the race
   where sase-core moves between the `ls-remote` and the checkout and the cache key then
   names a different tree than the one built), `astral-sh/setup-uv@v4`,
   `dtolnay/rust-toolchain@stable`, `Swatinem/rust-cache@v2` with
   `shared-key: sase-core-<sha>`,
   `uvx maturin build --release --out "$GITHUB_WORKSPACE/dist"`,
   `cargo build --release -p sase_xprompt_lsp`, install the LSP into `dist/`, and write
   `dist/sase-core-sha.txt`. Copy `ci.yml`'s `env` block
   (`CARGO_HTTP_MULTIPLEXING: "false"`, `CARGO_NET_RETRY: "10"`).
4. **`actions/cache/save`**,
   `if: success() && steps.core-cache.outputs.cache-hit != 'true'`. Explicit save rather
   than `actions/cache`'s automatic post-step, so a failed build can never publish a
   half-built `dist/` under a key every later run trusts.
5. **`actions/upload-artifact@v4`** as `sase-core-wheel`, `path: dist/`,
   `if-no-files-found: error` — on a hit and on a miss alike, so
   `./.github/actions/setup-sase` downloads it unchanged and needs no edit.

sase-core moves a few times a day against ~74 sase commits/day, so the steady state is a
hit: ~40 s instead of ~9 min.

### 5.2 `lint`

`needs: core-wheel`. **Byte-identical steps to `ci.yml`'s `lint` job** — same checkout,
same `./.github/actions/setup-sase` at 3.12, `tools/ci_bootstrap_sidecars` with
`SASE_RELEASE_TOKEN`, `sase init memory --no-commit`, `sase skill init --force`, the Go
and Prettier caches, then `just fmt-py-check`, `just fmt-md-check`, `just lint`,
`just validate`, `just validate-committed-plans`, `just build-check`.

The duplication is deliberate and _policed_: `tests/test_github_actions_ci.py` asserts
the two jobs' `steps` lists are equal, so the pair cannot drift. Extracting the sequence
into a composite action was considered and rejected for this phase — it would rewrite
`ci.yml`'s lint job and the five existing tests that assert its structure, which is real
risk added to a phase whose job is to remove risk. Phase `heavy` already edits `ci.yml`
and can do the extraction there if the duplication proves annoying.

### 5.3 `test`

```yaml
test:
  needs: core-wheel
  runs-on: ubuntu-latest
  timeout-minutes: 20
  strategy:
    # One shard failing must not erase the other five: the gate's value is
    # knowing everything that is wrong with this commit, not the first thing.
    fail-fast: false
    matrix:
      shard: [1, 2, 3, 4, 5, 6]
  steps:
    - uses: actions/checkout@v4
    - uses: ./.github/actions/setup-sase
      with:
        python-version: "3.12"
        # `just test` re-enters `_setup-visual` anyway; installing the visual
        # extras up front avoids a second uv resolve per shard.
        install-recipe: install-visual
    - name: Run the fast suite shard
      env:
        SASE_TEST_SHARD: ${{ matrix.shard }}/${{ env.SHARD_COUNT }}
      run: just test
```

Python 3.12 only. `just test` is the same fast suite every other lane runs — no
coverage, no cost plugin, no visual snapshots, and **no selection**. The gate skips no
test; what it omits is the _other_ lanes (the 3.13/3.14 matrix legs, `visual-test`,
`coverage-contexts`, `perf-floors`, the contention harness), which stay in `ci.yml` and
become the epic's scheduled heavy lane.

Step-level `env` is where `${{ env.SHARD_COUNT }}` resolves; a job-level `env`
referencing the workflow-level one is the shape that has historically been unreliable.

## 6. Tests

Extend `tests/test_github_actions_ci.py` with a `_load_master_gate_workflow()` helper
and:

- `on` is exactly `push: {branches: [master]}` plus `workflow_dispatch` — no `schedule`,
  no `pull_request`.
- `concurrency.group == "master-gate-${{ github.sha }}"` and `cancel-in-progress` is
  `False`. Assert the literal `${{ github.sha }}`: a group that loses the SHA
  interpolation reads fine and silently restores the cancellation behaviour this whole
  phase exists to remove.
- `permissions == {"contents": "read"}`.
- Every job declares `timeout-minutes` and none exceeds 20.
- `matrix.shard == list(range(1, int(env["SHARD_COUNT"]) + 1))`, each value once, and
  the test step's `SASE_TEST_SHARD` is `"${{ matrix.shard }}/${{ env.SHARD_COUNT }}"` —
  the invariant that ties the split's two halves together.
- The test job runs `just test` and none of `test-cov`, `test-cost`, `test-visual`,
  `test-contexts`, `test-slow`, `test-ace-page-group-isolated`.
- `lint`'s steps equal `ci.yml`'s `lint` steps (Section 5.2).
- `core-wheel` uploads `sase-core-wheel` from `dist/` with `if-no-files-found: error`;
  its cache key contains the resolved core SHA output; every maturin/cargo/checkout step
  is gated on the cache miss; the save step is gated on `success()` and the miss.
- Both `lint` and `test` declare `needs: core-wheel` and use
  `./.github/actions/setup-sase`.

**Extend `test_ci_never_runs_the_diff_scoped_test_lane` to cover `master-gate.yml`.**
Keep its docstring's reasoning and add the gate to the files it reads, so `test-scoped`,
`run_pytest scoped`, and `just check` remain absent from both. This is the assertion
that stops a future agent optimising the gate back into a selection heuristic; the
epic's research measured 10 of 10 master commits escalating the selector to the full
suite anyway, so a scoped gate would be slower _and_ narrower.

Do **not** assert anything about `ci.yml`'s trigger set here — that assertion belongs to
the epic's `heavy` phase, and adding it now would fail on arrival.

## 7. Docs

- `README.md`: add the fast-mode badge after the existing `CI` badge —
  `[![Master Gate](https://github.com/sase-org/sase/actions/workflows/master-gate.yml/badge.svg?branch=master)](https://github.com/sase-org/sase/actions/workflows/master-gate.yml)`.
  The `CI` badge stays and keeps meaning per-PR CI. The second (`Full CI`) badge and the
  sentence distinguishing the two lanes land in the epic's `heavy` phase, when
  `full.yml` exists.
- `docs/development.md`: a `#### Sharded gate runs` subsection right after
  `#### Per-test-file timings`, covering what `SASE_TEST_SHARD` does, that the shards
  are a partition and not a selection, where `tests/shard_timings.json` comes from and
  when to run `just refresh-shard-timings`, and that staleness costs balance, never
  coverage.
- `docs/configuration.md`: one row in the environment-variable table for
  `SASE_TEST_SHARD`.
- Do **not** touch `CHANGELOG.md`; release-please generates it from commit subjects and
  `tools/validate_changelog` rejects hand edits.

## 8. Verification

1. `just refresh-shard-timings --print-plan 6` — confirm six shards, similar estimated
   seconds, and a sane covered fraction.
2. `SASE_TEST_SHARD=1/60 just test` — a ~57-file shard proves the whole runner path end
   to end (spec parsing, file list, marker expression, worker grant, `execv`) in a
   couple of minutes rather than ten.
3. `python -c` over
   `assign_shards(discover_test_files(root), shard_count=6, table=...)`: total file
   count across shards equals `len(discover_test_files())`. The unit test pins this, but
   check it by hand once before trusting the workflow.
4. `just check`.
5. **`just check-full` through `/sase_monitor`** with the `TESTING` / `TESTED` status
   pair. This is not optional here: `tools/run_pytest` is selection tooling, so the
   scoped lane escalates on this diff anyway, and `just check-full` routinely outruns a
   single agent turn.
6. `yq`/`python -c 'import yaml; yaml.safe_load(...)'` on `master-gate.yml` before
   committing — a YAML error in a push-triggered workflow is invisible until it is on
   master.

The workflow itself cannot be verified before it lands, because it is push-triggered on
master. `workflow_dispatch` is in the trigger set precisely so the first post-land
action is a manual run rather than a wait.

## 9. Acceptance (measured after ~24 h, by the epic's `verify` phase)

- `gh run list --workflow=master-gate.yml --branch=master --limit 50` shows **zero**
  `cancelled` conclusions.
- p50 gate wall ≤ 8 min.
- ≥ 90% of master commits in a 24-hour window have a completed gate run.
- PR CI queue wait stays ≤ 1 min median — the canary that the gate has not reintroduced
  org-wide runner contention.

The gate being _green_ is the epic's `green` phase, not this one. This phase ships the
signal; that phase acts on it. A red gate on day one is the expected outcome and is the
point: it names one commit.

## 10. Risks

| Risk                                                                | Safeguard                                                                                                                                                                   |
| ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A shard boundary silently drops a test                              | The partition invariant is a parametrised test over the real file set (§4.1.1); the split is a pure function of the checked-out commit, so all shards agree by construction |
| `discover_test_files` diverges from pytest's collection set         | §4.1.6 fails with that exact remedy if `testpaths`, `python_files`, or the `*_test.py` convention changes                                                                   |
| A sharded run poisons the selection-health store                    | Full-lane health recording is suppressed under a shard, with the reason in a comment (§4.3)                                                                                 |
| `SASE_TEST_SHARD` leaks into the runner's own tests inside the gate | Cleared by the autouse isolation fixture in `tests/_run_pytest_fixtures.py` (§4.3)                                                                                          |
| The committed table goes stale                                      | Staleness costs balance, never coverage; the 20%-drift and 90%-existence bounds fail loudly with the refresh command (§4.2)                                                 |
| Durations measured on athena do not transfer to a 4-vCPU runner     | LPT needs only _relative_ cost; absolute wall time is measured from real gate runs, and N is a two-line change                                                              |
| A cache miss makes one gate run pay ~9 min                          | Bounded by `timeout-minutes: 20`; misses cluster and stop; the epic's `corepin` phase removes the dependency on sase-core's HEAD entirely                                   |
| A failed build poisons the wheel cache                              | Explicit `actions/cache/save` gated on `success()` and the miss (§5.1.4)                                                                                                    |
| The gate duplicates `ci.yml`'s lint job and they drift              | A test asserts the two step lists are equal (§5.2)                                                                                                                          |
| The gate adds org-wide runner contention                            | ~49 job-min/commit against 28,800/day is ~12%; PR CI queue wait is the acceptance canary                                                                                    |

## 11. Explicitly out of scope

Recorded so a later agent does not pull them in: any change to `ci.yml`; `full.yml` and
the scheduled heavy lane (epic phase `heavy`); `ci_watch`'s `gating_workflows`,
heavy-lane freshness, or merge strategy (phase `chop`); pinning the sase-core revision
in-repo (phase `corepin`); throttling release-please (phase `throttle`); fixing whatever
the gate turns red (phase `green`); and the `Full CI` badge with its explanatory
sentence (phase `heavy`).
