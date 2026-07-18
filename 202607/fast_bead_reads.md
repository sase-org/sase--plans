---
tier: epic
title: Fast bead reads
goal: 'Reading beads is consistently sub-second: bead CLI reads never pay a multi-second
  SQLite mirror rebuild, and sidecar store syncs stop issuing redundant network fetches
  on hot read paths.

  '
phases:
- id: mirror-batch
  title: Single-transaction, lazy SQLite bead mirror
  depends_on: []
  description: '''Single-transaction, lazy SQLite bead mirror'' section: batch the
    JSONL-to-SQLite mirror import into one transaction and open/rebuild the mirror
    lazily so read commands never pay for it.'
- id: fetch-ttl
  title: TTL-gate sidecar store network integration
  depends_on: []
  description: '''TTL-gate sidecar store network integration'' section: reuse the
    bead-refresh integration marker to skip the synchronous fetch+rebase in ensure_sidecar_sdd_clone
    when the clone integrated recently.'
- id: smoke
  title: Bead read latency and fetch-gating exercises
  depends_on:
  - mirror-batch
  - fetch-ttl
  description: '''Bead read latency and fetch-gating exercises'' section: end-to-end
    exercises proving cold and stale-mirror bead reads are fast and repeated clone
    syncs within the TTL do not refetch.'
create_time: 2026-07-18 07:25:38
status: wip
---

# Plan: Fast bead reads

## Context and evidence

`sase bead show <epic>` took 1m 28s inside a freshly launched agent workspace (agent "d2", 2026-07-18 07:00), and a
follow-up loop showing each child bead took another 26s. Reproduced locally: a first `sase bead show sase-6p` in a fresh
workspace took 25s of wall time with only ~0.9s of CPU (3% utilization — almost pure I/O wait).

Stage-level timing of the read path pinned the cost:

- `resolve_beads_location(materialize=True)` (including a real fetch+rebase): 0.74s
- `BeadProject()` construction: **24.5s**
- every actual read (`show`, `get_epic_children`, `list_issues`): ≤ 0.05s

`BeadProject.__init__` calls `rebuild_from_jsonl()` (`src/sase/bead/sync.py`) whenever `issues.jsonl` is newer than
`beads.db`. The import (`import_from_jsonl` in `src/sase/bead/jsonl.py`) upserts every issue through
`db_mod.create_issue` / `update_issue` / `add_dependency` (`src/sase/bead/db.py`), each of which ends with
`conn.commit()` — one fsync-backed WAL commit per row. The current store has 1,627 issues (1.3MB JSONL). Measured on the
workspace filesystem:

- current per-row-commit import: **33.1s**
- identical import wrapped in a single transaction: **0.18s** (~180x)

This rebuild fires constantly in practice:

- `beads.db` is gitignored, so **every fresh workspace clone pays a full rebuild on its first bead command** — exactly
  the d2 case (plans sidecar cloned at launch 06:59:43, first `bead show` at 07:00:03 stalled 88s).
- Every background sync integration (fetch+rebase) that touches `issues.jsonl` makes the mirror stale again, so with ~30
  agents syncing on a 120s TTL, agents re-pay the rebuild all day.
- Concurrent rebuilds contend on fsync, stretching 13–33s to 88s under load.

The kicker: production reads never touch that SQLite database. Every read op in `BeadProject` (`show`, `list_issues`,
`search`, `ready`, `blocked`, `stats`, `get_epic_children`) goes through the Rust facade
(`src/sase/core/bead_read_facade.py` → `sase_core_rs`), which reads the JSONL store directly. `self._conn` is only used
by `_max_local_child_counter` (a legacy child-ID helper) and the `_export` fallback when the Rust binding is
unavailable. The docstrings already call it a "compatibility mirror".

A second, independent problem showed up in the SDD git telemetry (`~/.sase/logs/tui_git_ops.jsonl`): the op
`sdd.clone.fetch` ran `git fetch --prune origin` against the primary checkout's plans clone **238 times in ~10 minutes**
(bursts of ~23 back-to-back fetches every ~70s), plus per-workspace fetches. `ensure_sidecar_sdd_clone`
(`src/sase/sdd/_store_link.py`) always calls `_pull_sdd_clone` → `integrate_sdd_repository`, i.e. a synchronous network
fetch + rebase with **no freshness gating**, every single time any caller (materialize path, `sase repo open`-style
flows, launch sync) touches a sidecar clone. Each call costs ~0.5–1.5s of network plus git subprocess work, hammers
GitHub, and adds the disk/CPU contention that turns 13s rebuilds into 88s ones. This violates the established TUI perf
rules ("periodic ticks revalidate; recomputes get a longer cadence", "keystroke paths never clone").

## Root cause summary

1. **Per-row commits in the mirror rebuild** — 1,627+ fsyncs per rebuild (13–33s; 88s under contention) to build a
   database reads never use.
2. **Eager mirror rebuild on every `BeadProject` open** — read commands pay the rebuild although only legacy helpers
   need the mirror.
3. **Ungated synchronous fetch+rebase in sidecar clone sync** — every `ensure_sidecar_sdd_clone` call is a network round
   trip, with no TTL reuse of the existing `sase-bead-sync.integration` marker.

## Design

### Single-transaction, lazy SQLite bead mirror

Two changes in `src/sase/bead/`, both Python-side (the mirror is explicitly a legacy Python compatibility artifact; see
"Rust core boundary" below):

**Batch the import.** Make `import_from_jsonl` perform its whole upsert loop in one SQLite transaction with a single
commit at the end. Preferred shape: give the row-level helpers in `db.py` (`create_issue`, `update_issue`,
`add_dependency`) an explicit way to skip their per-row `conn.commit()` (e.g. a `commit: bool = True` keyword), and have
`import_from_jsonl` pass `commit=False` and commit once after the loop. Preserve current semantics: upsert behavior,
corrupt-line skipping, dependency FK-violation tolerance (the per-dep `except` must not abort the transaction — SQLite
statement-level errors don't, but the implementation should verify with a test). All other callers of the row helpers
keep their existing commit-per-call behavior.

**Make the mirror lazy.** In `BeadProject.__init__`, keep the cheap eager work (beads-dir existence check,
`load_config`, id-generator setup) but stop calling `rebuild_from_jsonl` and `db_mod.init_db` eagerly. Expose the
connection through a lazy accessor (e.g. a `_conn` property backed by `_conn_cache`) that on first use runs
`rebuild_from_jsonl` + `init_db`. `__exit__` closes the connection only if it was opened. The two consumers —
`_max_local_child_counter` and the `_export` Rust-binding fallback — work unchanged through the lazy accessor. Net
effect: every bead read command (`show`, `list`, `search`, `ready`, `blocked`, `stats`) skips the mirror entirely, so
even a cold workspace's first read is sub-second, and the batched rebuild only runs when a legacy consumer actually
needs the mirror.

Keep `rebuild_from_jsonl`'s public contract (used by tests) intact: same mtime-based staleness check, returns whether a
rebuild ran.

### TTL-gate sidecar store network integration

Reuse the existing bead-refresh freshness machinery instead of inventing new policy. Today `src/sase/bead/sync.py` owns
the marker (`mark_bead_integration`, `_integration_is_fresh`, TTL from `sdd.bead_refresh.ttl_seconds`, default 120s),
but only the managed sync worker writes it.

1. **Move the marker helpers into the SDD layer** (e.g. a small `src/sase/sdd/_integration_marker.py`) so
   `_store_link.py` can use them without a `sdd → bead` import inversion. `sase.bead.sync` re-exports the existing names
   for compatibility.
2. **Mark on every successful integration.** `integrate_sdd_repository` (or its callers, whichever is cleaner given its
   outcome type) touches the marker whenever an integration succeeds with the upstream present. The sync worker's
   explicit call becomes redundant but harmless.
3. **Gate the fetch.** In `ensure_sidecar_sdd_clone` /`_pull_sdd_clone`, skip the `integrate_sdd_repository` call (treat
   as success) when the clone's integration marker is fresher than the bead-refresh TTL. Honor the configured mode: when
   `sdd.bead_refresh.mode` is `blocking`, never skip. Fresh clones have no marker, so launch-time materialization still
   fetches. Add an explicit escape hatch (e.g. `fresh: bool = False` parameter) so any caller that genuinely requires an
   immediate remote round-trip can force one; no current caller is known to need it (the sync worker calls
   `integrate_sdd_repository` directly and is unaffected).

This one gate fixes every hot caller uniformly: the materialize path taken by bead commands, `sase repo open`-style
sidecar syncs, and whatever drives the primary-clone fetch bursts. During implementation, use the `tui_git_ops.jsonl`
telemetry to identify the burst caller (bursts of ~23 fetches every ~70s from the primary checkout) and confirm the gate
covers it; if it turns out to bypass `_pull_sdd_clone`, apply the same marker check there. Skipping the fetch must not
skip cheap local repairs that callers rely on (`_set_sdd_origin` normalization and the git-info excludes in the
`finally` block still run).

Staleness impact: background mode already tolerates ≤TTL-stale reads (that is the documented `bead_refresh` policy); the
gate simply extends the same tolerance to redundant re-fetches, so no consistency guarantee changes. Writers are
unaffected: mutations write locally and converge through the managed sync worker's fetch/rebase/repair/push, which stays
ungated.

### Bead read latency and fetch-gating exercises

End-to-end proof that the fix landed and stays landed:

- **Cold-store read exercise:** build a synthetic sidecar-style store with ~1,500 issues (JSONL only, no `beads.db`),
  open a `BeadProject`, and run `show`/`get_epic_children`/`list_issues`. Assert no `beads.db` is created by reads
  (laziness) and that a forced legacy-consumer access performs the rebuild correctly.
- **Commit-batching guard:** count commits during `import_from_jsonl` via an instrumented connection (assert exactly
  one), rather than asserting wall time (CI-variance-proof). Verify the imported mirror matches the golden fixtures
  already in `tests/test_bead/`.
- **Fetch-gating exercise:** with a stubbed git runner / local-remote pair, call `ensure_sidecar_sdd_clone` repeatedly
  within the TTL and assert a single fetch; advance past the TTL (or use the `fresh` hatch / blocking mode) and assert a
  refetch. Cover the fresh-clone (no marker) case.
- **Regression sweep:** run the existing bead, sync, and store test suites; `just check` per repo policy.

## Rust core boundary

No Rust changes. Bead reads already go through `sase_core_rs` (`bead_read_facade`); the SQLite database being fixed is
the Python-only legacy compatibility mirror, and sidecar clone synchronization is Python workspace-infrastructure glue.
Nothing here adds domain behavior another frontend would need to match.

## Risks and mitigations

- **Transaction rollback on interrupted import:** a crash mid-import now loses the whole rebuild instead of a prefix.
  Acceptable — the mirror is derived data and rebuilds on next access; WAL keeps prior contents intact for concurrent
  readers.
- **Legacy consumer misses eager mirror:** audited consumers are `_max_local_child_counter` and `_export` fallback; both
  go through the lazy accessor. Tests cover both.
- **A caller that needed the always-fetch behavior:** the `fresh` escape hatch plus blocking-mode bypass covers it; the
  sync worker path is untouched.
- **Marker semantics drift:** marker move is a pure relocation with re-exports; the sync worker keeps working with
  either write site.

## Non-goals

- CLI surface changes (e.g. multi-ID `sase bead show`) — startup cost per invocation (~0.4s) is acceptable once the
  rebuild is gone.
- Removing the SQLite mirror entirely — worthwhile someday, but riskier and unnecessary for the latency goal.
- Python interpreter startup time.
