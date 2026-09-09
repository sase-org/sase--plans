---
tier: tale
title: Remove O(waiters × artifacts) realpath churn from wait-dependency resolution
goal:
  The wait_checks chop completes in seconds instead of minutes at production scale, with
  byte-identical observable behavior, by memoizing artifact-dir key resolution and
  replacing full-index scans with lazily built, mutation-invalidated lookups.
size: medium
proposed_by: bbugyi200.athena.0a9
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0a9](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0a9.md)
  - [bbugyi200.athena.sase-xe.16.11.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xe.16.11.3.md)
  - [bbugyi200.athena.sase-xe.16.11.4](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.11.4/README.md)
  - [bbugyi200.athena.sase-xe.16.11.5](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xe.16.11.5/README.md)
  - [bbugyi200.athena.sase-ys.2](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-ys.2/README.md)
- **COMMITS:**
  - [3b338c2](https://github.com/sase-org/sase/commit/3b338c208a4252aeea9fdd023d0f6763eb4514e0)
    — perf(wait-deps): cache artifact directory lookups
  - [1852f09](https://github.com/sase-org/sase/commit/1852f091ac3a4ebe8ac0cc25c6298d87d7edd3ee)
    — feat(xprompt): pin core and share ACE/LSP star-alias completion
  - [d015f48](https://github.com/sase-org/sase/commit/d015f48cb014c70483ee31d3b729c099bdd3e9d5)
    — fix(dispatch): clarify replayed-bootstrap 409 handling and prove it with real
    gateway + Fleet fault tests
  - [20c7b98](https://github.com/sase-org/sase/commit/20c7b9804577b7db315f42131b5702379f4d1c49)
    — feat(dispatch): integrate core setup policy and durable activation
  - [b7c6bc0](https://github.com/sase-org/sase/commit/b7c6bc0067032b53f30e841b54a6f179d4ff52e1)
    — fix(ace): decode live Fleet worker envelopes for Apollo catalog rows

# Make the `wait_checks` chop much faster without changing its behavior

## Problem

The `wait_checks` chop (`src/sase/scripts/sase_chop_wait_checks.py`) currently takes
**minutes of CPU per tick** at real-world scale, pegging a core continuously (AXE
re-triggers it on fs events under `*/artifacts/ace-run/*` with a 120s quiet backstop).
On a loaded host this makes waiting agents resume late and the chop itself is a
significant load generator. Measured on a production-scale state dir:

- ~12,200 ace-run artifact dirs across 32 projects; ~700 `waiting.json` markers of which
  **~615 are pending** (no `ready.json`) and get fully re-evaluated every tick.
- A 20s py-spy sample of a live run attributes **99.7% of wall time** to
  `same_artifact_dir` → `_artifact_dir_key` → `Path.resolve()` (realpath syscalls),
  reached via `_excluded_member_name` / `_excluded_present_names` / `_family_entity`,
  split ~52% under `dependency_resolution_status` and ~48% under the chop's
  `_terminal_blockers` pass.

## Root cause

In `src/sase/core/wait_dependency_resolution/`:

1. `_artifact_state._artifact_dir_key()` recomputes
   `Path(value).expanduser().resolve(strict=False)` on **every** call. Nothing caches
   it, and every `same_artifact_dir()` comparison resolves **both** sides.
2. `_index_entities.WaitDependencyEntityQueries._excluded_member_name()` linearly scans
   **all** of `artifacts_by_dir` (~12k candidates) calling `same_artifact_dir()` per
   candidate — ~24k realpath calls per invocation. It is invoked (via
   `_excluded_present_names`) on **every** `_family_entity()` call.
3. `is_resolved()` (per waited-on name, per waiter) and
   `terminal_blocking_artifacts_for_name()` (per blocked name, per unresolved waiter)
   each call `_family_entity()`. With ~615 pending waiters this multiplies out to tens
   of millions of realpath syscalls per tick — the observed minutes of CPU.
4. `_index_queries.WaitDependencyIndexQueries.tribe_candidate()` additionally rebuilds
   an O(all-artifacts) `TribeMemberRow` list and a `direct_tribes_by_artifact` map on
   every `@tribe` wait check, and finds `exclude_identity` with another full
   `same_artifact_dir()` scan, even though none of that depends on the call arguments.

The same package is used in-process by the runner fallback
(`src/sase/axe/run_agent_wait_deps.py`) and by TUI/agents wait displays, so fixing the
shared package speeds all of them.

## Goal

Reduce a `wait_checks` tick from minutes to seconds at the measured scale by removing
the O(waiters × artifacts) realpath churn, **without changing any observable behavior**:
identical `ready.json` decisions and contents, identical log lines, identical summary
counters and structured result, identical resolution semantics in every caller of the
package.

## Non-goals (explicitly out of scope)

- No change to marker discovery, index-vs-filesystem source selection, or the
  `SASE_CHOP_SCAN_FULL_WALK` parity path in `sase_chop_wait_checks.py`.
- No change to `add_many()`'s live `done.json` / `waiting.json` reads — those are
  deliberate freshness semantics (live marker state wins over the cached index).
- No pruning, skipping, or caching of pending waiting markers across ticks — every
  pending marker is still fully re-evaluated each run.
- No import-graph slimming of the `sase.gate_shell` → `sase.agent` → `sase.xprompt`
  chain (a separate follow-up task bead covers it).
- No Rust port. This is a performance-only refactor internal to existing Python modules:
  no new shared domain behavior, no wire/API change, so the Rust core backend boundary
  is not crossed. (A future port of wait resolution to `sase-core` would be its own
  decision; nothing here moves the seam.)

## Implementation

All changes are in `src/sase/core/wait_dependency_resolution/` unless noted.

### 1. Memoize `_artifact_dir_key`

In `_artifact_state.py`, cache `_artifact_dir_key()` results with a module-level memo
(e.g. `functools.lru_cache` with a generous bound such as 65536; ~12k unique dirs exist
today). Keep the existing exception fallback (return the raw value on
`OSError`/`RuntimeError`/`ValueError`) inside the cached function so failures are
memoized consistently too.

Correctness note to preserve: the function is a pure string → string mapping for a given
filesystem state. Caching only changes behavior if a symlink in a previously resolved
prefix is retargeted **within one process lifetime** — pathological, and the cache
actually makes comparisons more self-consistent since every caller shares it. Chop runs
are fresh subprocesses, so cross-run staleness is impossible there; the bound keeps
long-lived TUI/runner processes from growing without limit.

### 2. Replace the `_excluded_member_name` full scan with a lazy reverse map

Give the index a lazily built reverse map from resolved dir key → candidate:

- Build it on first use from `artifacts_by_dir` **in iteration order, first key wins**
  (use `setdefault`), so the "first matching candidate" semantics of the current scan
  are preserved exactly even if two distinct dir strings resolve to the same key.
- Store it as a cache attribute that does not disturb the `WaitDependencyIndex`
  dataclass contract (e.g. `field(init=False, repr=False, compare=False, default=None)`
  or an ordinary attribute initialized in `__post_init__`).
- Invalidate it (set back to unbuilt) in `_index.WaitDependencyIndex._add_prepared()`,
  the single funnel for all mutations (`add`, `add_many`, `add_scan_record`).
- Rewrite `_index_entities._excluded_member_name()` as: resolve the exclude dir once via
  `_artifact_dir_key`, then one dict lookup. Return `candidate.name or None` exactly as
  today; `None` when no candidate matches or `exclude_artifact_dir is None`.
- Reuse the same map for `tribe_candidate()`'s `exclude_identity` lookup in
  `_index_queries.py` (also currently a first-match full scan with the same semantics).

### 3. Cache `tribe_candidate()`'s argument-independent snapshot

`direct_tribes_by_artifact` and the `rows: list[TribeMemberRow]` built at the top of
`tribe_candidate()` depend only on index contents, not on the `tribe` / `newer_than` /
`exclude_artifact_dir` arguments. Hoist them into a lazily built, mutation-invalidated
cache alongside the reverse map (same invalidation point in `_add_prepared()`).
`resolve_tribe_wait_binding()` must keep receiving an equivalent rows list (order
included — preserve the current construction order).

### 4. Tests

- **Parity**: the existing suites are the behavior oracle and must pass unchanged:
  `tests/test_axe_chop_wait_checks*.py`, `tests/test_clan_wait_dependency.py`,
  `tests/test_tribe_wait_dependency.py`, `tests/test_gate_wait_dependency.py`,
  `tests/test_monitor_wait_dependency.py`, `tests/test_run_agent_wait_deps.py`, plus
  anything `just check`'s scoped selection adds.
- **New unit tests** (a focused new test module is fine):
  - Cache invalidation: after `add`/`add_many`/`add_scan_record` mutates the index,
    `_excluded_member_name` and `tribe_candidate` see the new artifacts.
  - First-match semantics: two candidates whose dir strings normalize to the same
    resolved key (e.g. one with a redundant `.` segment or via a symlinked tmp dir) make
    the reverse map return the same candidate the pre-change linear scan would have
    returned.
  - **Complexity regression guard**: with a synthetic index of N artifacts and M waiters
    (e.g. N=300, M=40), count invocations of the uncached resolve computation (wrap or
    monkeypatch the underlying function; do **not** assert on wall time — that is flaky
    on exactly the loaded hosts this fixes) and assert the count is O(N + M + distinct
    dependency dirs), not O(M × N).

### 5. Verify and measure

- Run `just check` (scoped lane) inline; if it runs long, hand it to `/sase_monitor` per
  the two-speed rule.
- Manual measurement for the tale summary: run the chop once against a synthetic sandbox
  at realistic scale (thousands of artifacts, hundreds of pending waiters, `SASE_HOME`
  pointed at a tmp dir so nothing touches real state) and record before/after wall time
  (baseline via `git stash` or the parent commit). A py-spy sample of the after-run
  should show `realpath` gone from the hot profile.

## Acceptance criteria

1. All existing wait-resolution and wait_checks chop tests pass without modification.
2. The complexity guard test proves resolve computations no longer scale with (waiters ×
   artifacts).
3. At synthetic production scale the chop completes in seconds, not minutes (expected
   ≥50× improvement; the dominant remaining costs are interpreter/import startup, the
   discovery walk, and the index query — all unchanged by design).
4. No public API, wire format, config, log format, summary counter, or resolution
   semantics change; `SASE_CHOP_SCAN_FULL_WALK=1` parity path still works.
