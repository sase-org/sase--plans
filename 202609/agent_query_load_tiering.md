---
tier: epic
title: Agent queries stop escalating to full-archive filesystem scans
goal: 'A committed Agents-tab filter never decides how much of the artifact archive
  a load reads. First paint always serves the bounded index window, full history arrives
  in the background from the SQLite index rather than a filesystem walk, and the common
  `machine:` filter is pushed down instead of falling off the indexed path — with
  a parity oracle proving no visible agent row is ever lost to any of it.

  '
phases:
- id: harness
  title: Load-path parity oracle and archive-scale benchmark
  depends_on: []
  size: medium
  description: 'harness: build the synthetic large-archive fixture, the parity oracle
    that proves every load path returns the same visible rows as an authoritative
    source scan, and the benchmark that times the three load paths.

    '
- id: defer
  title: Pushdown misses degrade to deferred history, not to a blocking full scan
  depends_on:
  - harness
  size: medium
  description: 'defer: stop letting a non-window-safe committed query escalate the
    load tier; serve the bounded index window first, arm the existing quiet-window
    full-history reconcile, and show honest partial-history state.

    '
- id: rust_index
  title: Artifact index gains full-history candidate filtering and machine provenance
  depends_on:
  - harness
  size: medium
  description: 'rust_index: in sase-core, apply the candidate filter on the non-windowed
    selection path, add an indexed machine-provenance column, and bump the artifact
    index schema version.

    '
- id: tui_full_history
  title: TUI full-history loads read the index instead of walking the filesystem
  depends_on:
  - rust_index
  size: small
  description: 'tui_full_history: route the TUI''s full-history load through the artifact
    index with a candidate filter, keeping the bounded source scan as an explicit
    fallback.

    '
- id: machine_pushdown
  title: machine filters become window-safe, and pushdown coverage becomes a contract
  depends_on:
  - rust_index
  size: small
  description: 'machine_pushdown: make `machine:` pushable with proven negation parity
    and add the coverage test that forces every future agents-live field to declare
    pushable or fallback.

    '
- id: refresh_reuse
  title: Refreshes stop re-paying for history the session already has
  depends_on:
  - defer
  - tui_full_history
  size: medium
  description: 'refresh_reuse: make the full-history upgrade a once-per-committed-query
    event so ordinary auto-refreshes use the delta path instead of repeating the expensive
    load.

    '
- id: land
  title: Remove the epic flags and land the measured result
  depends_on:
  - harness
  - defer
  - machine_pushdown
  - refresh_reuse
  size: small
  description: 'land: delete the three beta flags'' disabled branches, close their
    flag beads, publish before/after numbers in the perf runbook, and file the memory
    task bead.'
proposed_by: bbugyi200.kellys_mbp.05.f0
create_time: 2026-09-12 10:35:40
status: wip
bead_id: sase-zu
---

- **BEAD:** [sase-zu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zu/README.md)

# Plan: Agent queries stop escalating to full-archive filesystem scans

## Problem

On Athena, the ACE Agents tab takes ~42 s to become usable and then stalls repeatedly
during normal use. The measured cause is not the query engine and not the row count on
screen — it is a **tier escalation triggered by the committed filter**.

The user's saved filter is `not machine:apollo`. `machine` is not a field the pushdown
compiler can compile into an index candidate filter, so `load_tiered_agents`
(`src/sase/ace/tui/models/agent_loader.py:519`) computes:

```python
effective_full_history = full_history or (bool(raw_query) and not window_safe)
```

`effective_full_history=True` then means `requested_limit=None`, which means
`query_artifact_index_for_loader` bails out immediately
(`src/sase/ace/tui/models/_agent_loader_artifacts.py:116`,
`if full_history: return None`), which means `artifact_snapshot_for_tui_load` calls the
unbounded `scan_artifacts()` — a filesystem walk that opens roughly eight marker files
per artifact directory across the whole archive.

Athena's own `~/.sase/logs/tui_startup.jsonl` and `tui_agent_loads.jsonl` show the
cliff, with the saved-filter file's mtime (04:57:28) falling between the fast startup
and the slow ones:

| 2026-09-12, EDT | Load path    | Records examined | Agents ready |
| --------------- | ------------ | ---------------: | -----------: |
| 04:57           | Indexed      |              688 |        8.0 s |
| 05:42           | Full history |           13,190 |       23.3 s |
| 05:56           | Full history |           13,183 |       24.4 s |
| 06:20           | Full history |           13,190 |       42.7 s |

At 06:20 the loader's disk stage took **34.2 s** while _applying_ the loaded rows took
**0.26 s**. It read 13,190 records and materialized 1,271 agents in order to paint **23
visible rows**. Between 05:56 and 06:34 the same PID logged **41** such reloads, median
disk stage **25.5 s**, max **75.8 s** — because every broad refresh recompiles the same
committed query and re-takes the same escalation.

Host memory pressure made these worse but did not cause them; a separate effort is
already addressing the swapping. This epic addresses the load tiering.

## Why the current design produces a cliff

Four separate decisions compound. Each is individually reasonable; together they turn an
uncompilable filter term into an O(archive) filesystem walk on the startup path.

1. **Tier escalation is coupled to pushdown coverage.** "This query cannot be narrowed
   by the index" was treated as "therefore this load must read everything." Those are
   different claims. The TUI already applies the committed query itself, in
   `_loading_finalize.py:193` (`apply_agents_live_query_filter`), over whatever list the
   loader returned. Exactness of the _filter_ never depended on the load being complete;
   only _completeness of the result set_ did.

2. **Full history is defined as "source scan", never "index".** The artifact index
   already supports `include_full_history=True` (`AgentArtifactIndexQueryWire`, honored
   in sase-core `index.rs`), and many non-TUI callers use it (`gate_shell/store.py`,
   `main/var_cli.py`, `agents_sync/inventory_sources.py`, …). The TUI is the one caller
   that refuses the index precisely when the read is largest.

3. **The candidate filter is only consulted on the windowed path.** In sase-core,
   `candidate_matches_query_filter` is called only from `select_windowed_records`. The
   non-windowed selection (`select_records` with `include_full_history`) ignores
   `candidate_filter` entirely, so even an index-backed full-history read would decode
   every `record_json` blob in the archive.

4. **The pushdown vocabulary is narrow and undeclared.** Only `cl` and `model`
   (contains), `provider` (equals), plus `project` and `kind` are pushable
   (`agent_live_query_pushdown.py`). The `agents-live` profile exposes many more fields.
   Nothing anywhere states which fields are fast and which fall off the cliff, so adding
   a field to the profile silently adds a new performance trap.

## Shape of the fix

Four changes, in rough order of how much they buy:

- **Stop escalating.** A pushdown miss makes the result _incomplete_, which the codebase
  already knows how to express (`AgentLoadState`) and already knows how to repair
  (`_agents_history_reconcile_pending` → `input_quiet_tier2_reconcile`). Reuse that.
  This alone removes full-archive work from first paint for _every_ uncompilable query,
  not just `machine:`.
- **Make full history cheap.** Read it from the index, with the candidate filter
  applied, so the background upgrade is a bounded SQLite read instead of ~100k syscalls.
- **Make `machine:` pushable**, and make pushdown coverage an enforced contract so the
  next field cannot silently reintroduce the cliff.
- **Pay once.** A session that has already reconciled full history for a committed query
  must not redo it on the next auto-refresh.

### The safety invariant

The user asked for this to be **safe**. The failure mode that matters is not a crash —
it is an agent row silently missing from the list. Every phase below is therefore gated
on one invariant, enforced mechanically by the `harness` phase, not by reviewer
attention:

> For any committed query in the battery, the visible row set the Agents tab settles on
> is identical to the visible row set produced by an authoritative full source scan
> evaluated in Python.

"Settles on" is deliberate: a phase may make rows arrive _later_ (deferred history), and
that is the trade this epic buys. It may never make rows arrive _never_.

Two corollaries the implementing phases must respect:

- A candidate filter may over-select (return rows the exact query rejects) because the
  exact query runs afterwards. It may **never** under-select.
- Therefore any pushable atom appearing under `NotExpr` needs _exact_ index parity, not
  a superset — negating a superset yields a subset, which under-selects. `machine:` is
  exactly this case (`not machine:apollo`), so it gets its own parity proof.

### Flags

Three `beta` flags, each created by its introducing phase with `sase flag new`, each
removed by `land`. Three rather than one because the three routes land independently and
each can independently drop rows:

| Flag                        | Introduced by      | Enabled route                                            |
| --------------------------- | ------------------ | -------------------------------------------------------- |
| `agents_deferred_history`   | `defer`            | Pushdown miss serves bounded window + deferred reconcile |
| `agents_index_full_history` | `tui_full_history` | TUI full-history loads read the artifact index           |
| `agents_machine_pushdown`   | `machine_pushdown` | `machine:` compiles into an index candidate filter       |

Follow `sase/memory/sase_flags.md` (read it with `/sase_memory_read`): create only via
`sase flag new`, author the three required sentences, and test both states.

---

## Phase `harness` — Load-path parity oracle and archive-scale benchmark

This phase ships no behavior change. It ships the thing that makes every later phase
safe to land, and the numbers that decide whether the epic worked.

**Deliverables**

1. **A synthetic large-archive fixture.** A builder that materializes a temporary
   projects root with a configurable number of artifact directories (default ~13,000, to
   match Athena) spanning the marker shapes the TUI loader actually parses: active
   `running.json`, `waiting.json`, `done.json`, `workflow_state.json`,
   `agent_meta.json`, prompt-step markers, hidden rows, workflow families, and at least
   a few rows carrying archive provenance (`source_machine`). Build it once per session
   and cache it; it is too expensive to rebuild per test.

2. **A parity oracle.** Given a projects root and a query string, it returns the visible
   row set for each of: (a) authoritative source scan + Python-side query evaluation —
   the reference; (b) index-backed bounded window; (c) index-backed full history. It
   reports set differences by artifact dir, and distinguishes _missing_ rows (a
   correctness failure) from _extra_ rows (allowed for candidate over-selection, since
   the exact filter runs afterwards).

3. **A query battery** covering: empty query, every currently-pushable field (`cl`,
   `model`, `provider`, `project`, `kind`), `machine:apollo`, `not machine:apollo`,
   `machine:` (the "any remote" form — confirm its semantics against
   `src/sase/ace/query_profile/profiles/_agents_live.py` before asserting on it), a
   free-text `StringMatch`, and boolean compounds mixing pushable and non-pushable atoms
   under `and` / `or` / `not`. The negated compounds are the important ones.

4. **A benchmark**, marked `slow`, in the style of `tests/perf/` (see
   `tests/perf/README.md` and `docs/perf_runbook.md`), reporting p50/p95/max wall time
   for the three load paths against the fixture, and recording the **baseline numbers
   for the current code** in the phase's completion notes. Later phases compare against
   these.

**Acceptance**

- The oracle demonstrably _fails_ when pointed at an intentionally under-selecting
  candidate filter. A parity harness that cannot fail is not a harness — prove it can.
- Baseline numbers are recorded and reproduce the shape Athena logged: index-bounded
  load is roughly two orders of magnitude cheaper than the full source scan.

**Notes**

- Keep the fixture builder out of the default test lane's critical path; gate the
  archive-scale runs behind `-m slow`.
- Read `sase/memory/tui_perf.md` with `/sase_memory_read` before writing benches; it
  names the existing capture recipes and the conventions to reuse rather than reinvent.

---

## Phase `defer` — Pushdown misses degrade to deferred history, not to a blocking full scan

The startup fix. Independent of anything in sase-core.

**Change**

In `load_tiered_agents` (`src/sase/ace/tui/models/agent_loader.py`), separate the two
meanings currently fused in `effective_full_history`:

- `full_history=True` passed by the caller — an explicit request (manual `y` refresh,
  `input_quiet_tier2_reconcile`). Honor it as today.
- `bool(raw_query) and not window_safe` — a _pushdown miss_. Under
  `agents_deferred_history`, this must **no longer** escalate the load. Serve the
  bounded window exactly as an unfiltered load would (`requested_limit` honored, no
  candidate filter, since none could be compiled), and mark the returned
  `AgentLoadState` so the caller knows the result is query-incomplete.

Add a field to `AgentLoadState` (`src/sase/ace/tui/models/_agent_loader_artifacts.py`)
distinguishing "this load could not narrow the corpus for the committed query" from the
existing completeness signals, and include it in `needs_full_history_reconcile` so the
established arming path in `_loading_apply.py` (~lines 378–397) picks it up. Then the
existing `_maybe_trigger_input_quiet_tier2_reconcile` (`_loading_refresh_polling.py`)
performs the upgrade once input goes quiet — no new refresh code path.
`sase/memory/tui_perf.md` rule 5 is explicit that new refresh paths are not to be
invented; reuse this one.

Apply the same treatment to the legacy pushdown branch
(`src/sase/ace/agent_query/pushdown.py`), which is the `agents_unified_query`-off twin.
Both branches must behave identically here or the `agents_unified_query` sunset flag
becomes a second cliff.

**User-visible honesty**

A filtered list built from a bounded window can be missing older matches. Say so rather
than letting the user believe they are seeing everything: surface a brief, low-noise
indicator (footer/status, consistent with how `bounded_prefix` / `has_more` is already
communicated — find and match that existing treatment rather than adding new chrome)
that reads as "filtered on recent history; loading full history…" and clears when the
reconcile lands. Do not use a modal or a toast per refresh.

**Acceptance**

- Parity oracle passes for the whole battery, with the "settles on" reading: after the
  deferred reconcile completes, the visible row set matches the reference exactly.
- Benchmark: time-to-first-paint for `not machine:apollo` against the 13k fixture drops
  to the same order as the unfiltered bounded load.
- Both flag states tested. With `agents_deferred_history` off, the old escalation
  behavior is reproduced exactly.
- Textual event loop is untouched by the deferred work — the reconcile runs through the
  existing worker path, not on the pump. Re-read `sase/memory/tui_perf.md` rules 1, 2, 5
  and 9 before wiring anything.

---

## Phase `rust_index` — Artifact index gains full-history candidate filtering and machine provenance

Cross-repo. Open the Rust core with `/sase_repo` before touching it; it is **not** part
of this workspace checkout. Landing it dirty makes it a commit obligation on the
implementing agent's final declaration.

**Change A — candidate filter on the non-windowed path**

In `crates/sase_core/src/agent_scan/index.rs`, `candidate_matches_query_filter` is
currently reachable only from `select_windowed_records`. Apply it on the non-windowed
selection path as well, so a query carrying both `include_full_history=true` and a
`candidate_filter` selects only matching rows before decoding `record_json`.

Be careful about where the filter is applied relative to `record_matches_selection` and
the stale-row repair in `repair_stale_rows_for_query` — filtering must not cause a row
that _would_ have been repaired to be skipped and then served stale from a later,
unfiltered query. Prefer filtering at candidate-row selection (as the windowed path
does) over filtering decoded records.

**Change B — machine provenance column**

Add the scalar column(s) backing a `Machine` variant of
`AgentArtifactCandidateFieldWire`, populated on index write and returned by
`select_candidate_rows` into `IndexedCandidateRow::scalar_values`. What to store is
determined by the parity analysis the `machine_pushdown` phase owns; ship the column
shaped so that phase can express exact parity, and coordinate if the analysis says the
shape is wrong. Note that the current `agent_artifacts` table DDL has no machine column
at all, so this is additive.

**Change C — schema bump**

Bump `AGENT_ARTIFACT_INDEX_SCHEMA_VERSION` (currently `27`) and the mirrored constant in
`src/sase/core/agent_scan_wire_records.py`, and extend the Python
`AgentArtifactCandidateField` literal. Confirm the existing schema-staleness path
(`refresh_agent_artifact_index_if_schema_stale`, driven from
`src/sase/ace/tui/actions/_startup_loads.py`) rebuilds cleanly and that the TUI's
bounded-scan bypass keeps first paint interactive _during_ the rebuild — that machinery
already exists and must keep working, since a 13k-row rebuild is not instant.

**Acceptance**

- Rust tests cover: candidate filter applied on the full-history path; `All`/`Any`/`Not`
  nesting on that path; machine column populated and queryable; schema migration
  from 27.
- Existing `agent_scan_parity.rs` still passes.
- The Python-side wire round-trips the new field
  (`src/sase/core/agent_scan_wire_conversion.py`).
- Version-skew check: a Python tree at the new schema against an older binding, and vice
  versa, fails loudly rather than silently returning wrong rows. (A stale binding in a
  workspace venv already surfaces as `agent scan wire schema mismatch`; confirm the new
  version participates in that check.)

---

## Phase `tui_full_history` — TUI full-history loads read the index instead of walking the filesystem

**Change**

In `src/sase/ace/tui/models/_agent_loader_artifacts.py`, remove the unconditional
`if full_history: return None` at the top of `query_artifact_index_for_loader` under
`agents_index_full_history`. Instead, issue an `AgentArtifactIndexQueryWire` with
`include_full_history=True`, `freshness="cached"`, and — critically — the compiled
`candidate_filter`, which `load_tiered_agents` currently discards whenever
`effective_limit` is `None`
(`candidate_filter=candidate_filter if effective_limit else None`). Pass it through on
full-history reads too, now that `rust_index` honors it there.

Keep `scan_artifacts()` as the fallback for: index file absent, index operation lock
busy, index query raising, and schema mismatch — the same fallbacks
`query_artifact_index_for_loader` already encodes, with `AgentLoadState.repair_reason`
set so the existing repair notification still fires.

**The completeness question.** `complete_history=True` is a claim about the _archive_,
not about the index. Decide explicitly — and write the decision into the code as a
comment — whether an index-backed full-history read may set `complete_history=True`. It
may only do so if the index is authoritative for visible rows. If it is not (freshness
is `cached`, and rows can be added by other processes), either use `revalidate` for this
read or keep `complete_history=False` and let the existing reconcile machinery handle
the gap. Getting this wrong sets `_agents_seen_complete_history` on a partial corpus and
suppresses future reconciles — a silent row-loss bug, exactly what the safety invariant
forbids. The parity oracle is the arbiter.

**Acceptance**

- Parity oracle passes for the full battery on the index-backed full-history path,
  including rows written to the archive after the index was last built.
- Benchmark: index-backed full history against the 13k fixture is dramatically cheaper
  than the source scan (the source scan does ~8 file opens per artifact dir; the index
  does one row read). Record the actual ratio.
- Both flag states tested; every fallback branch covered.

---

## Phase `machine_pushdown` — `machine` filters become window-safe, and pushdown coverage becomes a contract

**The parity analysis comes first.** `machine` is multi-valued. `_machine_values`
(`src/sase/ace/tui/models/agent_live_query.py:223`) derives it from
`fleet_origin_alias or "here"`, plus values pulled out of `fleet_logical_locator` and
`fleet_exact_locator`, plus `source_machine`. Before writing any code, establish and
document which of those can appear on a row that the artifact index serves:

- `fleet_origin_alias` and the locators are set by the live fleet projection
  (`src/sase/ace/tui/models/_fleet_agents_rows.py`,
  `src/sase/ace/tui/actions/agents/_fleet_projection.py`), on rows injected into the
  list _after_ the loader returns. Confirm whether such rows can ever also be
  index-resident.
- `source_machine` is archive provenance on imported rows. Note that the agents-sync
  import leg has been deleted (`sase memory read decisions:agents-sync-publish-only`),
  so in practice these are legacy rows — confirm, do not assume.

If the index-resident value set reduces to `{"here"} ∪ {source_machine}`, exact parity
is achievable and `machine` can join `_PUSHABLE_EXACT_FIELDS` in
`agent_live_query_pushdown.py` (and the legacy twin in `agent_query/pushdown.py`). **If
the analysis shows exact parity is not achievable, do not ship the pushdown** — say so,
leave `machine` on the (now cheap, thanks to `defer` and `tui_full_history`) fallback
path, and deliver the coverage contract below instead. That is a legitimate outcome of
this phase, not a failure of it. Over-selection is safe; `not machine:apollo` turning a
superset into a subset is not.

Handle the bare `machine:` form explicitly — per
`src/sase/ace/tui/modals/help_modal/agents_bindings.py` it means "any remote", which is
not the same as `Equals` against an empty string, and note that sase-core's
`contains_case_insensitive` returns `true` for an empty needle. Either compile it
correctly or leave it unpushable; do not let it compile into something that matches
everything or nothing.

**The coverage contract.** Add a test that enumerates every field in the `agents-live`
query profile (`src/sase/ace/query_profile/profiles/_agents_live.py`) and asserts each
is either in the pushable set or in an explicit, named "known fallback" set. Adding a
field to the profile without classifying it fails the test. This is the durable
deliverable: it converts an invisible performance cliff into a decision someone has to
make on purpose. Include a short comment explaining _why_ the list exists, so a future
reader does not "fix" it by adding a wildcard.

**Acceptance**

- Parity oracle passes for `machine:apollo`, `not machine:apollo`, bare `machine:`, and
  the boolean compounds — with the oracle configured to treat missing rows as failures.
- Benchmark: `not machine:apollo` against the 13k fixture stays on the bounded window
  path end to end.
- Coverage test fails when a new profile field is added without classification (prove
  it).
- Both flag states tested.

---

## Phase `refresh_reuse` — Refreshes stop re-paying for history the session already has

Athena logged 41 expensive reloads in 38 minutes for one unchanged committed query. Even
at index speed, repeating the full-history read on every broad refresh is waste, and it
is the reason the TUI stayed unresponsive long after startup.

**Change**

Make the full-history upgrade a function of the _committed query_, not of each refresh:

- Track which committed query the session has already reconciled to full history.
  `_agents_seen_complete_history` (`_loading_apply.py`) is query-agnostic today; it
  needs to be keyed by the canonical query string (`canonical_query_for_profile` already
  exists for this in `agent_live_query_engine.py`) plus the profile digest.
- While that key is unchanged, ordinary auto-refreshes take the bounded/delta path. The
  artifact-delta machinery in `_loading_refresh_delta.py` already exists and already
  coalesces — route through it rather than adding a path.
- Re-escalate only when: the committed query changes, the user asks explicitly (`y`,
  `full_history_refresh`), or the delta path overflows and calls
  `_schedule_broad_fallback_for_agent_delta`.

**Also check the revalidate path.** In sase-core, `should_use_windowed_candidate_query`
requires `freshness == Cached`, so any refresh that passes `revalidate_index=True`
(`TIER1_INDEX_REVALIDATE_SOURCE` in `_loading_refresh_polling.py`) silently abandons the
window and takes the unbounded selection plus stale-row repair. Measure that path
against the 13k fixture. If it is expensive, bound it; if it is cheap, record the number
so the next person does not have to re-derive this.

**Acceptance**

- Benchmark: a simulated session doing N auto-refreshes with an unchanged committed
  query performs exactly one full-history read, not N.
- Changing the committed query still produces a correct, complete result (parity oracle,
  after settle).
- Coalescing guards are preserved and released on failure — `sase/memory/tui_perf.md`
  rule 2 is specific about releasing scheduled/running/pending guards when a spawn
  fails.

---

## Phase `land` — Remove the epic flags and land the measured result

1. Remove all three beta flags by deleting their **disabled** branches and making the
   enabled branches unconditional, remove the registry entries in
   `src/sase/feature_flags/registry.py`, and close each flag bead in the same change.
   `sase/memory/sase_flags.md` is explicit: an epic's scaffolding flags are removed by
   the epic, not left to age out on their `remove_by` thresholds.
2. Run the full benchmark and publish a before/after table in `docs/perf_runbook.md`
   alongside the existing capture recipes, including the reproduction command.
3. Verify on the real archive, not only the fixture: with the `not machine:apollo`
   filter committed, confirm `~/.sase/logs/tui_startup.jsonl` shows an indexed load and
   a first-paint time in line with the 04:57 baseline rather than the 06:20 one, and
   that `tui_agent_loads.jsonl` no longer accumulates repeated expensive full reloads.
4. File one `memory` task bead through `/sase_new_task` proposing a
   `sase/memory/tui_perf.md` rule capturing the durable lesson — _a query the index
   cannot narrow makes a result incomplete, never a load larger; pushdown coverage is a
   declared contract_ — as a companion to the existing rule 9. Do not edit memory
   directly from this phase; the user did not authorize a memory change when approving
   this plan.
5. `just check-full` through `/sase_monitor` (it outruns a single agent turn), per
   `sase/memory/lint_and_test.md`. Note that ephemeral `sase_<N>` workspaces may need
   `just install` first.

---

## Risks and how each is bounded

| Risk                                                                                                      | Bound                                                                                                                                                                             |
| --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A candidate filter under-selects and rows silently vanish                                                 | The `harness` parity oracle runs in every subsequent phase's acceptance, treats missing rows as failures, and is itself proven able to fail                                       |
| `machine:` negation parity is subtly wrong                                                                | Dedicated analysis-before-code step; explicit permission to _not ship_ the pushdown if exact parity is unachievable                                                               |
| Index-backed full history sets `complete_history=True` on a partial corpus, suppressing future reconciles | Called out as a named decision in `tui_full_history` with the oracle as arbiter                                                                                                   |
| Deferred history confuses the user ("where did my agents go?")                                            | Honest partial-history indicator in `defer`, reusing existing `bounded_prefix` chrome; rows arrive, they are not dropped                                                          |
| Schema bump forces a slow rebuild on first launch                                                         | The existing bounded-scan bypass in `_startup_loads.py` already keeps first paint interactive during rebuild; `rust_index` acceptance requires confirming it still does           |
| Deferred/background work lands on the Textual pump and trades a slow startup for a frozen UI              | Every phase reuses the established worker + quiet-window paths; `sase/memory/tui_perf.md` rules 1, 2, 5, 9 are required reading, and `~/.sase/logs/tui_stalls.jsonl` is the check |
| Cross-repo skew between sase and sase-core                                                                | Schema version bump plus the explicit version-skew test in `rust_index` acceptance                                                                                                |
| The `agents_unified_query`-off legacy path keeps the old cliff                                            | `defer` and `machine_pushdown` each require changing both the unified and legacy pushdown compilers                                                                               |

## Out of scope

- Host memory pressure and swapping on Athena. A separate effort owns that; the
  measurements above are host-pressure-inflated but the tier escalation reproduces
  independently of it.
- UI-thread stalls unrelated to agent loading (runtime-counter aggregation, notification
  ownership, style updates) seen in `tui_stalls.jsonl`. Worth filing separately; not
  this epic.
- Any change to agents-live query _semantics_. This epic changes only how much of the
  archive a load reads, never what a query means.
