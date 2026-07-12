---
create_time: 2026-07-12 14:45:29
status: wip
prompt: 202607/prompts/fast_bead_mutations.md
tier: tale
---
# Make `sase bead` Mutating Commands Fast

## Problem

The mutating `sase bead` commands (`create`, `update`, `open`, `close`, `rm`, `dep add`) are far slower than the read
commands that already ride the Rust fast path, and their latency is unbounded when the network is slow.

Measurements taken against a warm companion-repo bead store on a fast network:

| Command / cost                                        | Wall time      |
| ----------------------------------------------------- | -------------- |
| `sase bead ready` (Rust fast path)                    | ~0.21s         |
| Slow-path fixed overhead (`show`, `update` w/ bad id) | ~0.80–0.90s    |
| Full mutation (estimated: fixed + write side effects) | ~1.3–2.5s      |
| Worst case observed (network wait on one command)     | 21.9s (3% CPU) |

The slow-path fixed overhead breaks down as roughly ~0.35s of Python interpreter + full `sase.main.parser` argparse
construction + handler imports, and ~0.5s of a **synchronous `git pull --rebase` against the GitHub companion remote**.
The pull runs with a 120s network timeout (`DEFAULT_NETWORK_GIT_TIMEOUT_SECONDS`), so any GitHub latency spike or flaky
link serializes directly into every bead mutation. Agent workflows that touch several beads in a row (claim, close,
dep-link) pay this repeatedly.

## Root Causes (ranked by impact)

1. **Synchronous network pull on every slow-path bead command.** `get_project()` in `src/sase/bead/cli_common.py` calls
   `find_beads_location(materialize=True)` → `materialize_sdd_store` → `ensure_companion_sdd_clone`
   (`src/sase/sdd/_store_link.py`), which runs `git pull --rebase` against the remote even when the clone already exists
   and matches. This is the dominant and least-bounded cost.
2. **Mutations are excluded from the Rust fast path on companion stores.** `src/sase/main/bead_fast_path.py` bails for
   `_FAST_WRITE_COMMANDS` when `beads_dirname == "beads"` (the migrated companion layout — i.e. the standard setup), so
   every mutation pays full Python import + argparse + `BeadProject` startup. Reads like `ready`/`search` already prove
   the Rust path (`sase_core_rs.bead_cli_execute`) does the same work in ~0.2s.
3. **Eager SQLite compat-mirror rebuild per mutation.** `BeadProject._refresh_db_from_jsonl`
   (`src/sase/bead/project.py`) deletes `beads.db` (+wal/shm) and rebuilds it from `issues.jsonl` after every mutation,
   even though the process exits immediately and `rebuild_from_jsonl` already has an mtime-based lazy rebuild for the
   next reader.
4. **Auto-commit subprocess chain.** `auto_commit_bead_store` → `commit_sdd_store_files` runs ~5–6 git subprocesses
   (ls-files, add, diff --cached, commit, rev-parse) plus merged-config loads and commit-marker recording on every
   mutation. Individually cheap; together ~0.3–0.6s.

## Goals

- Warm-store mutating commands complete in **≤ ~0.4s wall time** with **no network subprocess on the critical path**
  (target: parity with today's fast-path reads plus one local git commit).
- No semantic regressions: identical commit messages / `SASE_*` runtime tags / commit markers, identical telemetry
  counters, identical stored bead state and design-path normalization, unchanged first-use materialization (missing
  clones are still created), unchanged conflict-resolution guarantees.
- Cross-clone convergence gets **more** robust, not less (the current detached push chain can strand a clone mid-rebase;
  see Phase 3).

## Non-Goals

- `sase bead work` orchestration/launch performance (separate flow with its own commit/push helpers; it inherits the
  Phase 3 sync worker but is otherwise untouched).
- Changing `list`/`show` output or fast-pathing their rendering (they still benefit from Phase 1's removal of the
  blocking pull).
- Any resident daemon. A daemon-based read-routing approach was previously rolled out and reverted
  (`5a65fa4fc feat: revert sase-3e daemon rollout`); this plan sticks to one-shot detached workers, which the async push
  path already precedents.

## Design

### Phase 1 — Take the network off the critical path (biggest win, smallest risk)

- **Resolve-first store opening.** Change `get_project()` (and thereby `get_read_view()`) to resolve the store via the
  existing lightweight `resolve_beads_location(require_existing=True)` path — the exact resolution the Rust fast path
  already trusts for reads — and only fall back to `materialize_sdd_store` when the store or its clone is genuinely
  missing/mismatched. Result: no `git pull --rebase`, no store re-init planning on the warm path; first use in a fresh
  workspace still clones exactly as today.
- **TTL-gated background freshness.** After a bead command completes (fast or slow path), if the clone's last
  integration with the remote is older than a configured TTL, spawn a detached one-shot sync worker (Phase 3) instead of
  ever blocking. Track "last integration" with a small state marker in the clone (or FETCH*HEAD mtime). Default TTL on
  the order of 60–300s keeps cross-workspace bead status (e.g. epic phase closes → dependent `ready` in another
  workspace) propagating at least as fast as today's effective behavior — note fast-path reads _never* pull today, so
  background refresh strictly improves read freshness.
- **Escape hatch.** New config keys (documented in `src/sase/default_config.yml`, per the config gotcha): something like
  `sdd.bead_refresh` with `mode: background|blocking|off` and `ttl_seconds`, where `blocking` restores today's
  pull-before-op behavior for anyone who wants strict freshness over speed. Follows the existing `sdd.push_after_commit`
  precedent.

### Phase 2 — Route companion-store mutations through the Rust fast path

- Remove the non-VC write exclusion in `_resolve_fast_path_context` so `create`/`open`/`update`/`close`/`dep` fast-path
  on companion stores too. The Rust binding already performs the store mutation (events + `issues.jsonl` + manifest); it
  was excluded only because the Python wrapper never performed the companion-store commit/push side effects.
- Extend `_apply_mutation_side_effects` in `src/sase/main/bead_fast_path.py` to fully own those side effects using only
  lightweight imports:
  - auto-commit the store repo with the same message format (`chore(beads): <op> <ids>`, derived from the Rust
    `mutation_summary`), preserving runtime tags (`apply_auto_commit_tags_with_runtime`) and commit markers
    (`record_sdd_commit_result_marker`) — both helpers are already import-light;
  - spawn the existing detached push (`push_bead_work_launch_async` / the Phase 3 worker);
  - trigger the TTL background refresh;
  - keep the existing SQLite rebuild + telemetry counters.
- **Command parity audit.** Confirm `sase_core_rs.bead_cli_execute` handles the full mutating surface and flag set
  (including `rm`, `dep add`, `--tier`, `--model`, `--changespec`/`--bug-id`, multi-id `close`); close any gaps in the
  sase-core repo (`../sase-core`, sibling of the primary checkout) per the Rust core backend boundary — bead mutation
  semantics belong in `sase_core`; the thin git orchestration stays in the existing shared Python SDD commit helpers
  rather than being duplicated in Rust. Also audit that `rm` today cannot be handled by the Rust path on a companion
  store _without_ the commit side effects (it is not in `_FAST_WRITE_COMMANDS`, so it must either be proven unhandled in
  Rust or added to the gated set).
- **Path semantics parity.** `create --type plan(<path>)` must store the same normalized design path on both paths
  (`storage_plan_path` / `relativize_design_paths`); add explicit equivalence tests.
- The argparse slow path remains the fallback (help output, exotic flags, legacy stores, stale/missing Rust wheel) and
  is itself no longer network-blocking after Phase 1.

### Phase 3 — Robust convergence: a managed sync worker (the "without breaking anything" insurance)

Deferring pulls makes the push/integrate path more load-bearing, so harden it:

- Replace the raw detached `sh -c "git push || (git pull --rebase && git push)"` in `push_bead_work_launch_async`
  (`src/sase/bead/sync.py`) with a managed one-shot worker (an internal sase entry point, not a resident process) that:
  fetches → rebases → auto-resolves bead-store conflicts via the existing event-stream merge machinery → pushes; on
  unresolvable failure it **aborts the rebase and leaves the clone clean**, logs a structured outcome to the existing
  push-log location, and never prompts. A lock file dedupes concurrent workers per clone.
- **Fix the companion-layout gap in the conflict resolver.** `resolve_bead_conflicts`
  (`src/sase/bead/conflict_resolver.py`) hardcodes the in-tree `sdd/beads` prefix, so conflicts in a companion store's
  `beads/` directory currently cannot be auto-resolved at all — a latent bug that this plan's reliance on
  merge-on-integrate makes worth fixing regardless. Parameterize it by store layout and reuse it from the sync worker.
- **Doctor visibility.** Extend `sase bead doctor` to detect a diverged / mid-rebase / unpushed store clone and point at
  the latest sync-worker log, so silent convergence failures are surfaced instead of masked.
- Any new/changed CLI surface follows `memory/cli_rules.md` (alphabetized help, short aliases for public long options,
  excellent `-h` output).

### Phase 4 — Trim residual local overhead and lock in a perf floor

- Make the SQLite compat-mirror refresh lazy: stop deleting + rebuilding `beads.db` inside every mutation; let the
  existing mtime-gated `rebuild_from_jsonl` refresh it on next read. Audit in-process consumers (e.g.
  `_max_local_child_counter`) to confirm none read the mirror after a mutation within the same command.
- Trim trivially redundant git subprocesses in the auto-commit chain (e.g. derive the commit SHA without a separate
  `rev-parse`).
- Extend `tests/perf/bench_bead.py` with companion-store mutation benchmarks (shell-level, warm store) and add a
  regression guard that fails if a warm-store mutation's critical path executes any network git operation (interceptable
  via `run_sdd_git`'s `op` labels / subprocess monkeypatching).

## Acceptance Criteria

1. Warm companion store, fast-path mutation (`update --status`, `close`, `dep add`, `create`): ≤ ~0.4s wall, zero
   network subprocesses before the command prints its result and exits.
2. Cold store (no clone yet): behavior unchanged — clone/materialize happens, command succeeds.
3. Two-clone integration test (two local clones + a bare "remote"): concurrent mutations on both clones converge —
   events merged, `issues.jsonl`/manifest regenerated, no lost updates, no clone left mid-rebase, dependent-bead `ready`
   visibility propagates after sync.
4. Commit parity: identical commit message format, `SASE_*` tags, commit markers, and telemetry counters between old
   slow path and new fast path (golden comparisons in tests).
5. `blocking` config mode reproduces today's pull-before-op semantics.
6. All existing bead test suites pass (`tests/main/test_bead_fast_path.py`, `tests/test_bead/*`,
   `tests/test_core_facade/*`), with the non-VC write-exclusion tests updated to the new contract.

## Risks and Mitigations

- **Mutating against a stale clone** produces divergent event appends. The store is event-sourced (append-only per-bead
  streams + regenerated projections) and the merge machinery exists precisely for this; the managed worker
  auto-resolves, the TTL bounds staleness, and the `blocking` mode remains available. Today's pre-pull only narrows
  (never closes) the same race, and pushes are already async — so divergence handling is already a required property of
  the design.
- **Detached worker failure modes.** Crash-safe by construction (abort rebase, clean tree), lock-deduped, structured
  logs, doctor surfacing. Strictly better than the current unmonitored `sh -c` chain.
- **Fast-path import creep.** The wrapper's new side-effect imports must stay light; add a smoke check that the
  fast-path process never imports `sase.main.parser` (or another heavy marker module).
- **Rust CLI drift.** Parity audit + slow-path fallback (`handled: false`) keeps unknown flags correct while gaps are
  closed in sase-core.

## Implementation Notes

- Repos touched: this repo (Python CLI/orchestration) and the sibling Rust core repo (`../sase-core`,
  `crates/sase_core`) if the parity audit finds gaps in `bead_cli_execute` or the conflict/merge facade needs a layout
  parameter. Wire/API, bindings, and Rust tests land there first per the core-boundary rule.
- New config keys must be added to `src/sase/default_config.yml`.
- If user-visible bead command semantics change (freshness/pull behavior), the generated `sase_beads` skill docs may
  need regeneration from `src/sase/xprompts/skills/` — follow `memory/generated_skills.md`; memory files themselves are
  not to be edited without explicit user approval.
- Suggested landing order: Phase 1 → 2 → 3 → 4. Phase 1 alone removes the unbounded network hazard and cuts ~0.5s+
  typical; Phase 2 delivers the ~4–5× headline win; Phase 3 is the safety net that should land before or with Phase 2's
  default-on fast-path writes; Phase 4 is cleanup + regression floor.
