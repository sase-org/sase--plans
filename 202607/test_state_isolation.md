---
tier: tale
title: Isolate pytest state writes and clean polluted telemetry
goal: Prevent unisolated pytest processes from writing telemetry, axe logs, or axe
  notifications into the real user state tree, repair the known leaking runner fixtures,
  and provide an explicit safe command that removes only test-labeled telemetry from
  the local metrics store.
create_time: 2026-07-20 17:07:53
status: wip
prompt: 202607/prompts/test_state_isolation.md
---

# Plan: Isolate pytest state writes and clean polluted telemetry

## Context and outcome

The live telemetry database contains a large, continuously refreshed body of samples labeled with `test-provider`,
`test-workflow`, and `fakey`, including the fixed runner-fixture instance `20260316_120000`. Axe crash-loop output has
also reached the production axe log and notification store while referring to pytest temporary homes. The existing
autouse `SASE_HOME` redirect and call-time path resolution prevent many leaks, but they are conventions rather than a
hard write-boundary invariant and do not protect against subprocesses or helpers that lose the redirected environment.

The implementation will make real-home writes impossible from a detected pytest context, without changing normal
production behavior or blocking tests whose SASE home is correctly isolated. It will also expose a deliberately invoked
telemetry maintenance command; the command will never run as startup, retention, doctor, or migration side effect.

## Pytest-to-production state guard

Add one reusable Python guard that evaluates state paths at the moment of the write. It will:

- recognize the established pytest context signals (`PYTEST_CURRENT_TEST` and the subprocess-compatible equivalent
  already used by the axe lifecycle guard);
- determine the actual account home independently of fixture-patched `HOME`/`Path.home()`, then compare the resolved
  target against that user's real `.sase` tree;
- permit every write outside pytest and every pytest write under an isolated SASE home;
- expose a strict refusal for synchronous test-visible writers and a fail-closed, warn-once refusal for best-effort
  daemon writers, with diagnostics naming the rejected target and explaining how to isolate `SASE_HOME`;
- avoid any override that would make production-home writes acceptable merely to get a test passing.

Keep the guard in shared Python state/path glue rather than Rust: it is process-environment policy, while the underlying
telemetry storage rules remain frontend-independent Rust behavior.

## Telemetry and axe write boundaries

Call the strict guard in `src/sase/telemetry/_registry.py` immediately before the Rust batch write, using the configured
store path and before pending samples are drained or retried. An unisolated pytest write must raise a focused isolation
error so the faulty fixture fails loudly; a correctly redirected store must continue through the real Rust binding.
Initialization and read-only telemetry queries remain unaffected.

Apply the non-crashing guard policy to the axe state writers that resolve their paths at call time, including bounded
aggregate-log appends and crash/error persistence. Gate the orchestrator crash-loop notification before it enters the
general notification store, so one rejected episode cannot create a production notification, pending action, error
record, or log entry. Repeated refused daemon writes should emit one warning per target/category rather than form a new
log storm.

Exercise the hard guard against the known agent-runner fixture family. Ensure the shared runner helper and any spawned
process environment preserve the per-test `SASE_HOME` that pytest established, including the fixed
`20260316_120000`/`test-workflow` case. Fix fixture isolation instead of mocking out telemetry or weakening the guard.

## Rust-owned test-data cleanup

Extend `sase-core`'s telemetry wire and store API with an explicit maintenance request/report for exact label matches.
The Rust implementation will own SQLite inspection and mutation, delete matching rows consistently from raw samples and
both rollup tiers, and avoid substring matching. The SASE caller will request only these known test identities:

- `llm_provider` equal to `test-provider` or `fakey`;
- `workflow` equal to `test-workflow`.

Perform deletion transactionally, leave unrelated metrics and labels untouched, refresh store metadata so freshness does
not describe deleted rows, and vacuum only after a successful committed deletion. Return per-tier and total row counts
plus useful before/after store-size information. Include a dry-run/report path so callers can preview the exact scope
without mutation. Expose the operation through `sase_core_rs` and keep Python as a thin configured-path adapter.

## Explicit CLI maintenance action

Add an alphabetically registered `sase telemetry cleanup-test-data` command with complete help and examples. It should
preview the known label criteria and counts, require an explicit `-y/--yes` before deletion (and refuse destructive
non-interactive use without it), support `-n/--dry-run`, and render a concise colored report of raw/rollup removals and
space reclaimed. Follow the CLI rule that every public long option has a short alias, and update telemetry dispatch and
usage text without changing the existing bare `sase telemetry` → `list` behavior.

Do not invoke the cleanup command against the user's live database as part of implementation or verification.

## Validation and regression coverage

Add Rust unit coverage that builds a temporary telemetry database containing production rows, exact test-labeled rows,
near-miss labels, and data folded into both rollup tiers. Verify preview is read-only, deletion removes only exact
matches from every tier, metadata is recomputed, vacuum/reporting succeeds, and a second cleanup is idempotent. Add a
PyO3 round-trip test for the new request/report.

Add Python tests covering unisolated pytest telemetry refusal before the binding is called, successful writes to an
isolated temporary store, warn-once axe suppression without touching a simulated real state tree, the repaired frozen
runner fixture, parser/help sorting and short aliases, confirmation behavior, dry-run behavior, and end-to-end cleanup
of a temporary database while preserving non-test samples.

After implementation, rebuild the local Rust extension through `just install`, run focused Rust and Python tests during
iteration, run `just rust-check` for the linked core, and finish with the required `just check` in the SASE checkout.
Inspect both repository diffs and statuses, then close only `sase-8g.11` and verify parent epic `sase-8g` remains open.

## Risks and safeguards

- Real-home detection must not trust environment values that pytest commonly patches; tests will inject account-home
  resolution so they prove the comparison without writing to the actual user tree.
- The daemon path must fail closed without throwing into the crash-loop supervisor; the telemetry path must fail loudly
  enough to identify leaking fixtures.
- Cleanup predicates must be exact and apply to rollups as well as raw rows. A preview, explicit confirmation, selective
  fixtures, and idempotence checks protect production telemetry.
- SQLite `VACUUM` must run outside the deletion transaction and only after success; busy/corrupt-store behavior should
  use the telemetry store's existing timeout and error conventions.
