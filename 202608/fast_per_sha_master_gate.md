---
tier: tale
title: Fast per-SHA master gate
goal:
  Every master SHA gets a bounded, never-cancelled gate that runs the complete fast
  suite in balanced shards.
size: medium
proposed_by: bbugyi200.athena.sase-um.1
bead: sase-um.1
create_time: 2026-08-27 07:13:35
status: wip
---

- **PARENT:** [202608/release_gate_liveness.md](release_gate_liveness.md)
- **BEAD:**
  [sase-um.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.1.md)

# Fast per-SHA master gate

## Goal

Complete phase bead `sase-um.1` by adding a fast GitHub Actions gate for every push to
`master`. The workflow must be attributable to one SHA, must never cancel an older SHA,
must run the entire non-visual fast test suite across deterministic balanced shards, and
must reuse a SHA-keyed Rust-core wheel cache. Add the runner tooling, tests,
documentation, and README badge needed to keep those properties durable.

This phase is additive. Do not alter `ci.yml` triggers, jobs, or concurrency; add the
scheduled heavy lane; change `ci_watch`; pin a core revision in-repo; throttle
release-please; or add the later `Full CI` badge. Those changes belong to later phases
of epic `sase-um`.

## Implementation

1. Add deterministic whole-suite shard support.
   - Create `tests/_test_shards.py` with sorted pytest-file discovery, strict parsing of
     1-based `SASE_TEST_SHARD=<index>/<count>` specs, committed timing-table loading,
     longest-processing-time-first assignment, and a concise shard summary.
   - Estimate every discovered file: use its recorded duration when available, then the
     table default, then `1.0`. Sort by descending estimate with a SHA-256 path tiebreak
     and place each file in the currently lightest bin, with lower-index bin ties
     winning. Refuse more shards than files.
   - Guarantee that all `tests/**/test_*.py` files are assigned exactly once. Skip
     hidden path components and `__pycache__`, use repo-relative POSIX paths, and pin
     assumptions about pytest's `testpaths`/`python_files` conventions in tests.
   - Create `tests/shard_timings.json`, generated from the existing merged local timing
     store, with schema, timestamp, host, full measured-file count, default duration,
     and the 800 slowest existing test files. Unknown or newly added files must still
     run; table staleness may affect balance but never coverage.

2. Add timing-table refresh tooling.
   - Create typed executable `tools/refresh_shard_timings` using
     `tests._test_selection_timings.load_timing_table()`. Support `--limit` (default
     800), `--check`, and `--print-plan N`; round retained durations to 0.1 seconds,
     derive the default from omitted files, emit sorted stable JSON with a trailing
     newline, and refuse a source table whose slowest retained path no longer exists.
   - Add `just refresh-shard-timings` beside `refresh-contract-manifest`, preserving
     caller arguments and the normal `_setup` dependency.

3. Integrate sharding into `tools/run_pytest` without changing existing callers.
   - Read `SASE_TEST_SHARD` after normal mode and selector resolution. Empty or unset
     must preserve existing arguments and behavior exactly.
   - Permit sharding only in `fast` mode and reject explicit caller test selectors.
     Convert parsing and configuration failures into the existing clear
     `pytest.UsageError` path.
   - In the parent process, discover and assign the whole fast-suite file set, append
     exactly the selected shard's files to pytest arguments, and print the shard summary
     to stderr. Preserve marker handling, worker grants, `execv`, and the timings
     recorder.
   - Suppress full-lane selection-health recording for an active shard because a shard
     is partial evidence; add an explanatory code comment. Keep timing recording active
     so shards refresh the local file-duration store.
   - Add `SASE_TEST_SHARD` to the autouse environment isolation in
     `tests/_run_pytest_fixtures.py` so the workflow's ambient variable cannot leak into
     runner contract tests.

4. Add `.github/workflows/master-gate.yml`.
   - Trigger exactly on pushes to `master` and `workflow_dispatch`, with read-only
     contents permission. Use `master-gate-${{ github.sha }}` concurrency and
     `cancel-in-progress: false`. Set `SHARD_COUNT` to `6`, enumerate matrix shards
     `1..6`, set `fail-fast: false`, and give every job a timeout no greater than 20
     minutes.
   - Add a `core-wheel` job that validates and exposes the 40-hex-character remote
     `sase-core` HEAD SHA, restores `dist/` from a cache keyed by OS, cache-version
     salt, and that SHA, and on a miss checks out exactly that resolved revision and
     builds the same abi3 wheel, xprompt LSP, and provenance file as `ci.yml`.
     Explicitly save only after a successful miss, then always upload the
     `sase-core-wheel` artifact from `dist/` with missing files treated as an error.
   - Add a `lint` job that needs `core-wheel` and duplicates `ci.yml`'s lint steps
     byte-for-byte, including setup, sidecars, SASE initialization, format checks,
     validation, and build verification. A contract test will police equality.
   - Add a `test` matrix job that needs `core-wheel`, uses Python 3.12 with the visual
     install recipe, and runs `just test` with
     `SASE_TEST_SHARD=${{ matrix.shard }}/${{ env.SHARD_COUNT }}`. Do not run scoped,
     coverage, cost, slow, contexts, or visual lanes here.

5. Add contracts and documentation.
   - Add `tests/test_test_shards.py` for partition/disjointness across multiple shard
     counts, determinism under shuffled input, committed-table balance within 10% at six
     shards, empty-shard rejection, strict spec errors, discovery assumptions, table
     file-existence coverage of at least 90%, and measured-count drift no worse than
     20%. Failures must point maintainers to `just refresh-shard-timings`.
   - Add `tests/test_run_pytest_shards.py` using the existing runner harness to pin
     valid selection, malformed/out-of-range rejection, fast-mode-only behavior,
     explicit-selector rejection, unchanged unset behavior, suppressed health recording,
     and retained timing recording.
   - Extend `tests/test_github_actions_ci.py` with a master-gate loader and invariants
     for triggers, per-SHA non-cancelling concurrency, permissions, timeouts, shard
     matrix/spec consistency, fast-only test command, core cache/build/artifact rules,
     dependencies, setup action use, and lint-step identity. Extend the existing ban on
     the diff-scoped lane to cover both `ci.yml` and `master-gate.yml`, without adding
     assertions about `ci.yml`'s trigger set.
   - Add the `Master Gate` badge after the existing `CI` badge in `README.md`.
   - Document sharded gate runs, timing refresh, partition semantics, and staleness in
     `docs/development.md`; document `SASE_TEST_SHARD` in the configuration environment
     variable table. Do not edit `CHANGELOG.md`.

## Verification and closeout

1. Run `just refresh-shard-timings --print-plan 6` and confirm six similarly weighted
   shards with a sensible covered fraction.
2. Exercise the real runner path with `SASE_TEST_SHARD=1/60 just test`.
3. Independently load the committed table, discover the files, assign six shards, and
   confirm that their combined file count and set exactly match discovery.
4. Parse `master-gate.yml` with the repository's YAML loader and inspect the resolved
   trigger/concurrency/job structure.
5. Run `just check`. Because the change touches pytest selection/runner tooling, run
   `just check-full` through the required SASE monitor workflow and record its result.
6. Run `sase bead epic-symbols sase-um.1`; resolve every symbol or re-key its Justfile
   entry to an open parent/later phase before closing.
7. Close only `sase-um.1` with a note naming the shard, workflow, YAML, `just check`,
   and monitored `just check-full` evidence. Do not close `sase-um` or any ancestor.
   Record any out-of-scope discovery only as a `PROPOSED FOLLOW-UP:` note on this phase.

The workflow's live cancellation rate, wall-time median, completed-run coverage, and PR
queue impact cannot be measured before it lands. Those 24-hour acceptance measurements
remain assigned to the epic's later verification phase.
