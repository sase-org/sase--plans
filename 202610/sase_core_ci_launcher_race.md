---
tier: tale
title: Stabilize sase-core CI launcher argument recording
goal:
  Remove the incomplete-file race that intermittently fails native and Python-hosted
  sudo launcher tests in GitHub Actions.
size: small
proposed_by: bbugyi200.athena.0wy
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0wy](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.0wy.md)
- **COMMITS:**
  - [c57b662](https://github.com/sase-org/sase-core/commit/c57b662c7e95794520fdb025acd4cb76c90ebc90)
    — fix(tests): publish complete sudo launcher fixture records atomically

# Stabilize sase-core CI launcher argument recording

## Scope and execution context

Implement this as one focused, test-only repair in `sase-org/sase-core`. Open the
repository with
`sase repo open sase-core -r "Implement the approved CI launcher race repair"` and use
only the printed checkout. Read its `AGENTS.md` before editing. Paths below are relative
to that checkout. The diagnosis was made at master
`16095fcf715cf64a6f4d217961ce3f2083ab3b7f`; refresh the live run status and inspect any
intervening changes before implementing.

A `tale` sized `small` is appropriate because two failing tests share one identified
fixture defect, with a bounded repair and regression tests that one coding agent can
complete. No epic phases or parallel agents are needed. This planning turn made no
implementation changes and did not run a local Rust build.

## Evidence and diagnosis

The requested command was run as `actstat --repo sase-org/sase-core -n 4 --format json`.
It repeatedly returned September 24 results even though `actstat` with `-n 1` and
`gh run list --repo sase-org/sase-core` returned October 5 results. Verify dates and
SHAs using live GitHub run metadata; do not assume the older report describes HEAD. The
cause of that reporting discrepancy has not been established and is outside this
repository repair.

The live failures that still require a fix are:

| Run                                                                              | Commit / context                                                                                                              | Failure                                                                                                                                                                                                                                                                                  |
| -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [CI 37333225769](https://github.com/sase-org/sase-core/actions/runs/37333225769) | Release PR [#321](https://github.com/sase-org/sase-core/pull/321), head `1295ad4013b82f51e8ff9abc8a3851892000c27d`, October 5 | Ubuntu `cargo test --workspace`: `sudo_runner::tests::cli::python_hosted_launcher_preserves_isolated_module_prefix`, `tests/cli.rs:237`, `worker did not record Python-hosted argv: Ok(())`. Gateway result: 223 passed, 1 failed. macOS, wheel smoke, and release script checks passed. |
| [CI 37329509253](https://github.com/sase-org/sase-core/actions/runs/37329509253) | Master `d65f7246681d44bc81e2d48a180e869dbe538b44`, October 5                                                                  | Ubuntu: sibling `native_launcher_places_internal_modes_directly_after_executable`, `tests/cli.rs:201`, `worker did not record argv: Ok(())`.                                                                                                                                             |
| [CI 36015335975](https://github.com/sase-org/sase-core/actions/runs/36015335975) | `9956773f1fee51305700b0a1e5004873dbc36e5c`, September 24                                                                      | Same Python-hosted launcher assertion, showing the defect predates today's macro changes.                                                                                                                                                                                                |

The latest master
[CI 37333078904](https://github.com/sase-org/sase-core/actions/runs/37333078904) passed
at `16095fcf`. A local Git comparison between this commit and release PR head `1295ad4`
shows only Cargo version/lockfile and changelog differences, with no changes to the sudo
runner tests. This and the alternating failing launcher tests support an intermittent
fixture race rather than a release-version defect.

The concrete race is in `crates/sase_gateway/src/sudo_runner/tests/support.rs`:

1. `waiting_worker_script()` (around lines 481–515) opens `$0.argv` with shell
   redirection before writing `BEGIN`, individual arguments, and `END` in separate
   `printf` calls. The path can exist while the record is empty or incomplete.
2. `run_waiting_worker_exec()` (around lines 556–601) waits only for
   `argv_path.exists()`, then reads once. Its existing ten-second deadline does not help
   after the path appears.
3. `recorded_argv()` (around lines 530–547) maps read errors to empty content and
   returns accumulated arguments even if no terminating `END` was observed. Thus a
   successful spawn can produce the exact empty/short vector seen in CI.
4. The helper may then destroy its temporary directory before the child has finished its
   record or published `worker.pid`, making cleanup unreliable as well.

Related existing task `sase-15e` documents the same stub's non-atomic `worker.pid`
publication: forced termination can leave an empty PID file and panic the two
`post_spawn_*_reaps_barred_worker` tests. Address that publication in the same small
fixture change; preserve their process-liveness assertions. Do not create a duplicate
work item.

Two other observed failure classes are already repaired:

- Historical
  [Release-plz 36015335925](https://github.com/sase-org/sase-core/actions/runs/36015335925)
  failed because the `sase_gateway = "^0.34.0"` path requirement rejected the next
  workspace version, `0.35.0`. Commit `89ad2e93e61a881acd873da0b3f6bd3e4479f74e` removed
  that pin on September 26. The latest
  [Release-plz run 37333079044](https://github.com/sase-org/sase-core/actions/runs/37333079044)
  passed. Preserve the path-only dependency.
- Earlier October 5 runs, including `37329418511`, failed stale macro/xprompt wire
  expectations. Commit `d65f7246` updated those expectations and related fixtures; later
  master and release PR runs passed the core tests. Do not undo that migration.

## Implementation

1. **Publish complete fixture records.** In `tests/support.rs`, have
   `waiting_worker_script()` write its full framed argv record to a private temporary
   file beside the final file, then rename it into place only after completion. Publish
   `worker.pid` using the same write-then-rename pattern. Publish the complete PID
   before the final argv path so observing argv also makes cleanup identity available.
   Preserve the original arguments before the shell argument-parsing loop shifts them.
   Keep temporary and destination files in the same directory and use tool paths that
   work on both Linux and macOS. Preserve the existing started-handshake barrier, worker
   markers, isolated Python prefix, and failure-injection behavior.

2. **Require a complete record when reading.** Change `recorded_argv` to distinguish a
   complete `BEGIN`/`END` record from a missing, empty, or incomplete file; never return
   a partial vector as success. Preserve argument order and empty arguments. An
   `io::Result<Option<Vec<String>>>` or equivalent explicit result is sufficient. In
   `run_waiting_worker_exec`, retry pending/incomplete observations with the existing
   bounded deadline instead of using file existence as readiness. Surface real I/O
   errors and timeout context, including the launch result and observed record, rather
   than disguising them as empty argv. Complete child cleanup before returning or
   asserting, including on timeout/error. Keep this logic in test support; no production
   launch, privilege, process-identity, or wire behavior should change.

3. **Add regression coverage for the failing interleaving.** Put focused tests beside
   the helper or in a small sibling test module, registered in `tests/mod.rs` if needed.
   Cover an absent file, an existing empty file, `BEGIN` without `END`, a partially
   written argument list, and a complete record. Use a controlled staged producer with
   synchronization to hold a record incomplete until the reader has observed the pending
   state, then publish/finish it and require the full vector. Exercise the same
   readiness helper used by the launcher tests. Include a bounded incomplete-record
   failure case with a configurable short test deadline; do not depend on winning a
   scheduler race or fixed sleeps for correctness. Demonstrate that the regression fails
   with the old behavior and passes with the fix.

4. **Keep integration assertions strong.** Retain the native and Python-hosted launcher
   tests in `tests/cli.rs`, checking `--internal-root-worker` placement and the complete
   `-I -m sase_core_rs.sudo_runner` prefix. Assert the internal executor result succeeds
   as well as validating captured argv. Recheck both
   `post_spawn_identity_failure_reaps_barred_worker` and
   `post_spawn_publish_failure_reaps_barred_worker` in `tests/worker.rs` against the
   atomically published PID. Missing PID still means the worker was stopped before
   publishing; a published PID must remain valid and its process must be dead in these
   failure tests. Do not weaken assertions, add ignores/retries in CI, increase the
   readiness timeout, or serialize the entire suite to hide the race.

## Verification and completion

All Rust verification must use the repository's `just`/`scripts/check.sh` entry points;
never run bare Cargo. These set up Python >= 3.12, libpython, and the hermetic test
environment. No new Python binding or wire contract is introduced, so no sase core
revision pin update is needed. Do not edit release versions, changelogs, workflow
cadence, or unrelated macro code.

- Run the new regression tests, then
  `sase tool run -- just test -p sase_gateway sudo_runner::tests::cli::` and
  `sase tool run -- just test -p sase_gateway sudo_runner::tests::worker::post_spawn_`.
- Because the original defect is intermittent, perform a bounded stress pass of the two
  launcher tests (for example 20 iterations with the normal parallel harness), stopping
  on the first failure and preserving its output. Also run
  `sase tool run -- just test -p sase_gateway sudo_runner` once to cover the other
  consumers of the changed fixture. Use the deterministic regression as the primary
  proof; stress passes alone do not prove the race fixed.
- Apply `just fmt` as needed. Run **`sase tool run check` from the opened sase-core
  root** and require success: it covers formatting, dependency-feature consistency,
  clippy, all workspace tests including PyO3, and release script tests. Targeted tests
  do not replace this gate. Allow at least ten minutes for the full check. If the tool
  escalates or refuses inline execution, follow `/sase_monitor` and join the exact
  existing run; do not restart or abandon it. Fix failures attributable to this change
  and rerun affected checks and the full gate after the final edits.
- Inspect the final diff for test-only scope and confirm temporary fixture files and
  child processes are cleaned up. Record the full-check result and specific regression
  evidence. If `sase-15e` is fully resolved, update that existing task through the
  audited bead workflow with verification evidence; do not claim an isolated passing run
  alone proves all its cases fixed.
- Use `/sase_final` for host-owned completion of the modified linked repository. Let the
  host manage commits and publication. After publication, check the CI run for the
  resulting commit and the refreshed release PR with live GitHub metadata and `actstat`.
  Confirm both Ubuntu and macOS jobs, release scripts, and wheel smoke pass. If
  publication or a new run has not happened yet, clearly report that remote confirmation
  remains pending instead of treating an older green run as verification of this fix. Do
  not manually merge or release the package to clear CI.

Acceptance: complete-record publication and consumption have deterministic regression
coverage; both launcher variants retain their exact argv guarantees; forced worker
failure tests retain their cleanup checks; the full sase-core check passes on the final
tree; and the final report distinguishes local verification from fresh GitHub Actions
results.
