---
tier: epic
title: 'Bead store performance: remove replay waste, add a Rust read model, take issues.jsonl
  off the commit path'
goal: 'Hot-path bead reads and writes stop scaling with closed history. Every recommendation
  in the bead-history research report is implemented: the measured waste is removed,
  a Rust-owned, disposable, fingerprint-validated read model serves current state,
  issues.jsonl leaves the per-mutation commit path, and the physical sealed-archive
  design sits behind measured triggers. No bead event is ever edited, compressed in
  place, or deleted.

  '
phases:
- id: bench
  title: Scaled-corpus bead benchmark harness
  depends_on: []
  size: medium
  description: 'bench: add a deterministic realistic-shape synthetic bead corpus at
    any scale, a local prefix-copy tool for real stores, a scale-aware bead benchmark,
    and a record-only 4x CI run.'
- id: outbox
  title: Constant-cost artifact-link outbox append
  depends_on: []
  size: small
  description: 'outbox: stop re-reading and re-canonicalizing every outbox entry on
    each append (~2.9 s of every audited read) while keeping operation-id collision
    semantics.'
- id: maintenance
  title: Hidden-clone gc and bead push-log retention
  depends_on: []
  size: small
  description: 'maintenance: make the sidecar gc pass reach the host-owned hidden
    beads clone, fix the stale projection-size comment, and add bounded retention
    for ~/.sase/bead_push_logs.'
- id: parse-once
  title: One parse, one validation, no lockless-read deletes
  depends_on: []
  size: medium
  description: 'parse-once: in sase-core, move the removed-flag stream prune off the
    read path, drop the deep clone of every stream, and validate each event and issue
    once (~480 ms to ~290 ms per replay).'
- id: fingerprint
  title: Store fingerprint binding and consumer migration
  depends_on: []
  size: medium
  description: 'fingerprint: add an exact stat-only bead_store_fingerprint core binding
    and move the five issues.jsonl mtime-keyed consumers onto it.'
- id: tui-board
  title: TUI Beads and Plans pane refresh
  depends_on:
  - fingerprint
  size: medium
  description: 'tui-board: group phases in one pass, stop forced reloads on auto-refresh
    ticks, and load list/ready/blocked from one core read via a board-snapshot binding.'
- id: one-replay
  title: One store read per CLI command
  depends_on:
  - parse-once
  size: medium
  description: 'one-replay: route targets without a full read, resolve inside the
    locked mutation load, and collapse the Python lanes'' redundant resolve/show calls
    so each command reads the store once.'
- id: read-model-store
  title: Read-model substrate, freshness protocol, and parity harness
  depends_on:
  - parse-once
  - bench
  size: medium
  description: 'read-model-store: add the versioned SQLite read model under the clone''s
    git dir with an O(1) freshness token, full-rebuild fallback, transparent read
    integration, doctor --verify-cache, and a cache-vs-replay parity harness.'
- id: read-model-tail
  title: Snapshot-plus-tail incremental refresh
  depends_on:
  - read-model-store
  size: medium
  description: 'read-model-tail: apply appended events after the merge frontier instead
    of rebuilding, falling back to a rebuild on any precondition failure, with randomized
    parity and outcome telemetry.'
- id: seal-watch
  title: Sealed-segment triggers and design
  depends_on:
  - read-model-tail
  size: small
  description: 'seal-watch: report the measurable sealed-archive triggers in bead
    doctor and document the gated sealed-segment design without building it.'
- id: projection-off
  title: issues.jsonl off the per-mutation path
  depends_on:
  - fingerprint
  - read-model-store
  - one-replay
  size: medium
  description: 'projection-off: stop rewriting and committing issues.jsonl on every
    mutation, untrack it with a mixed-version-safe migration, migrate its content
    consumers, and add on-demand export.'
- id: read-model-queries
  title: Indexed queries over the read model
  depends_on:
  - read-model-tail
  - one-replay
  size: medium
  description: 'read-model-queries: serve detail, ready, blocked, list, stats, resolve,
    search, and multi-get from indexed read-model tables so hot queries touch only
    active or requested rows.'
- id: read-model-mutations
  title: Mutations load and write through the read model
  depends_on:
  - read-model-queries
  - projection-off
  size: large
  description: 'read-model-mutations: make MutableStore load only affected rows from
    the read model and write rows plus frontier through in the same locked critical
    section.'
- id: perf-gate
  title: History-independence acceptance gate
  depends_on:
  - read-model-mutations
  - tui-board
  - outbox
  - maintenance
  - seal-watch
  size: medium
  description: 'perf-gate: enforce the A1 history-independence criteria on scaled
    corpora in CI and locally, and record the final before/after results.'
proposed_by: bbugyi200.athena.research.3u.linker.w0
create_time: 2026-10-06 18:59:25
status: done
bead_id: sase-1h8
---

- **PROMPT:** [prompts/202610/bead_store_history_independent_performance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/bead_store_history_independent_performance.md)
- **BEAD:** [sase-1h8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h8/README.md)

# Plan: Make bead-store latency independent of closed history

Source research (read it with `sase artifact read` before starting any phase):
`research:202610/bead_history_cost_and_lossless_archival/bead_history_cost_and_lossless_archival.md`.
Its predecessor
`research:202609/bead_speedup_daemon_vs_direct_path/bead_speedup_daemon_vs_direct_path.md`
specified Phases 0–1 in more detail and is useful background.

## Why

Every bead read and write replays the whole event store (2,108 streams, ~48,400 events,
35 MB today; 91.6% of beads closed). One replay costs ~0.48–0.52 s in Rust. Reads do 1–2
replays, writes 3–5 (up to 12 on the Python `update -n` lane), and each write rewrites
and commits the 20.8 MB `issues.jsonl` projection. The store grows ~88 beads/day, so the
cost doubles by about January 2027 and quadruples by mid-2027. The TUI Beads pane does 3
replays plus an O(epics × issues) loop every 10 s (2.8 s today, ~48 s at 4×). An audited
`sase bead read` takes 5–8 s, of which 2.9 s is an artifact-link outbox scan that has
nothing to do with beads.

Where one replay's ~480 ms goes: stream I/O 15 ms (3%); the removed-flag prune re-parse
128 ms (26%); typed parse 88 ms (18%); duplicate validation ~28 ms per pass; deep clone
32 ms (6%); merge/apply/post-pass ~130 ms (26%). About 40% is removable without a cache.
The rest is inherent to replay, so a cache is what makes cost independent of history.

## Requirement adjustments adopted

The research's adjustments A1–A7 are requirements for this epic:

- **A1, the metric.** Hot-path latency must be independent of closed history. On 1× → 8×
  scaled corpora, p95 of `ready`, `list`, `read` (detail) and a local mutation moves by
  less than 10%. Other targets: warm core point read ≤ 20 ms, active list ≤ 50 ms, TUI
  no-change refresh < 100 ms.
- **A2.** Compression is not a performance lever. No event file is compressed.
- **A3, lossless.** Events are byte-immutable and never deleted. Derived files
  (projection, pages, caches) may be dropped and regenerated. Git history is never
  rewritten. These must keep working: old IDs, shorthand, history, search,
  `/sase_new_task` duplicate detection (which must still see closed beads), dependency
  resolution, edited and retracted note revisions, `+1` evidence and close history.
- **A4.** "Archived" means logical: the read model never re-parses unchanged closed
  history. Physical sealing is not built. It waits behind measured triggers
  (`seal-watch`).
- **A5.** Scope includes the derived and adjacent costs on the bead path:
  `issues.jsonl`, the outbox scan, hidden-clone gc, `bead_push_logs` retention, and the
  TUI loop.
- **A6.** `beads.db` stays the mutation flock. The cache gets a different file. No bead
  daemon or service proc.
- **A7.** Benchmarks run on scaled corpora (≥ 4× in CI), never only on toy stores.

This plan adds four adjustments of its own:

- **P1. O(1) freshness check.** The research's design runs a full stat sweep on every
  read: 4 ms at 1× but ~48 ms at 8×. That alone breaks A1's "< 10% from 1× to 8×" for a
  20 ms point read. So the read model first checks a constant-cost token: the streams
  directory's `(mtime_ns, inode)` plus the `events/manifest.json` and `config.json`
  signatures. Every supported writer changes the directory entry: the Rust writer does
  temp-file-plus-rename and git unlinks then creates. A full sweep still runs whenever
  the token changes, on every write, on `--verify-cache`, and at least every 60 s (a
  core constant). So even an unsupported in-place edit is caught within a bounded
  window.
- **P2. Cache outside the working tree.** The research proposed a gitignored
  `beads.cache.sqlite` inside the store. Instead, the cache lives under the clone's git
  dir, so no commit path, `copytree` (the reconciliation finalizer copies the store),
  adoption copy, or stale long-running TUI with old ignore rules can ever stage or copy
  it. A store with no git dir is served by plain replay.
- **P3. A content consumer the research missed.**
  `src/sase/core/artifact_file_protection.py` scans `issues.jsonl` text to protect
  bead-referenced artifacts from pruning. Untracking the projection without migrating it
  first could let artifact prune delete referenced artifacts, which is permanent loss.
  `projection-off` must migrate it first.
- **P4. No feature flag.** Per the September research's RA5, the read model must return
  exactly what a replay returns. That is enforced by parity tests,
  `sase bead doctor --verify-cache`, version-keyed rebuilds and fail-open-to-replay, not
  by a rollout switch (and per `decisions:rust-core-required`, no backend switch).
  Untracking the projection is a self-healing store-layout migration, not a user choice.

## Global constraints for every phase

- **Rust-core boundary.** Reduction, cache, query, fingerprint, routing and
  ready/blocked semantics live in sase-core's `sase_core` crate. Open it with
  `sase repo open sase-core` and read its `AGENTS.md`. Python stays thin glue. Any sase
  code that calls a new binding needs sase's `sase-core-revision.txt` moved past the
  sase-core commit (`just ratchet-core-revision`; see `docs/rust_backend.md`). Never
  edit sase-core versions or changelogs.
- **Bindings.** `crates/sase_core_py/src/beads/mod.rs` (1,526 lines) and `bead/read.rs`
  (1,440 lines) are at or near the 1,500-line cap. Put new bindings and new core code in
  new files in the same domain, behind `mod.rs` facades.
- **Benchmarking.** Never mutate the live bead store or its clones for benchmarking. Use
  copies under `/tmp` or the `bench` corpus.
- **Verification.** Run `sase tool run check` in each repo you change (sase-core and
  sase). Never weaken an assertion to get green.
- **New CLI options** follow the CLI rules (sase memory `cli_rules.md`): short alias,
  alphabetical order, excellent `-h`. Any new config value goes into
  `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`.
- **Memory.** Do not edit `sase/memory/**`, provider shims, or the memory templates
  under `src/sase/main/init_memory/templates/`. If a memory note becomes inaccurate,
  append a `PROPOSED FOLLOW-UP:` note to your phase bead.
- **Beads.** Phase workers never create beads. Record discovered work as
  `PROPOSED FOLLOW-UP:` notes on your own phase bead.
- **Measurement.** Every phase that claims a speedup records before/after numbers in its
  bead notes, using the `bench` harness once it has landed.

---

## Phase `bench`: scaled-corpus harness

The existing `just bead-perf-smoke` (`Justfile`, `tests/perf/bench_bead.py`) runs 50
issues with no remote. That is how a 20 s regression went unnoticed in September.

1. **Synthetic corpus generator.** Add it under `tests/perf/` (for example
   `_bead_corpus.py`).
   - It must be deterministic and seedable, and write a valid event store
     (`events/manifest.json`, `events/streams/*.jsonl`, `config.json`) that the
     production reducer accepts: `bead_read_store` succeeds and `bead_doctor` reports no
     errors.
   - At scale 1× it matches today's shape: ~7,000 beads (phase ~71%, task ~16%, plan
     ~13%), ~2,100 streams, ~48,000 events, 91.6% closed. It must include:
     - phases sharing their epic's stream, including ~60 live epics that carry closed
       phases;
     - cross-lineage dependencies, including dependencies on closed beads;
     - `link_added`, `issue_updated` and `note_appended` events landing after close (the
       research counted 2,899 such events ≥ 7 days after close);
     - note edits and retractions, `+1` events, external refs, and removed issues.
   - Scale `k` multiplies history natively (beads, streams and events).
   - Mint event IDs exactly as core does (`events/merge.rs` mint, ~`751-775`). If Python
     cannot reproduce the mint cheaply, call an existing binding. Add a test-support
     binding only as a last resort.
   - The generator runs `git init` and makes one commit in the corpus directory, so
     stores sit in a git work tree as they do in production (the read model needs that;
     see P2).
2. **`tools/bead_scale_corpus`.** A local-only helper that copies a real store into a
   temp directory, with optional length-preserving prefix-renamed k× copies (the
   research's method).
   - It is read-only on the source.
   - It refuses a destination inside any sidecar clone.
3. **Scale-aware benchmark.** Extend `tests/perf/bench_bead.py`, or add a sibling
   module, so it takes `--scale` (repeatable) or `--store <dir>` and measures
   p50/p95/max for:
   - binding reads: `bead_show_issue_detail`, `bead_ready`, `bead_blocked`, default
     `bead_list`, a closed listing with limit 20, `bead_stats`, `bead_search`;
   - CLI: `sase bead show <id>` and `sase bead ready`;
   - mutations: `bead_append_note` and `bead_update` at the binding level, and the full
     CLI `note` tail against a local bare remote;
   - the TUI Beads board loader (`load_beads_snapshot` in
     `src/sase/ace/tui/widgets/artifacts/beads_data.py`), both cold and with no change.

   It emits JSON with per-op stats, the corpus shape, and the core revision.

4. **Recipes.** Add `just bead-perf-scale` (local, slow-marked, scales 1/2/4/8). Make
   CI's `bead-perf-smoke` job (`.github/workflows/ci.yml` ~348) also run the generated
   4× corpus in **record-only** mode with no thresholds; `perf-gate` turns enforcement
   on. Keep the CI addition within a few minutes; trim the 4× op set if needed.

**Acceptance**

- At 1× the generated corpus reproduces the research's replay cost
  (`bead_show_issue_detail` ~0.5–0.6 s) within host noise, and cost grows roughly
  linearly with scale.
- A baseline JSON at 1×/2×/4× (8× if feasible) is recorded in the bead notes.
- Generator tests cover validity, determinism and shape.

## Phase `outbox`: constant-cost outbox append

`append_artifact_link_outbox_entry` and `append_artifact_link_outbox_event`
(`src/sase/sdd/_artifact_link_outbox_io.py`, ~100-177) call
`_reject_outbox_operation_collision` (~477) under the exclusive lock. That function
re-reads the whole outbox (11,400+ lines, 7.8 MB) and runs Rust classify, `json.loads`
and canonicalize on every line. This is the 2.9 s inside every audited `sase bead read`,
`sase artifact read` and `sase memory read`.

1. When the operation ID was minted by `uuid4()` in this call (no caller-supplied
   `entry_id`), skip the scan. A collision is impossible.
2. When the ID is caller-supplied, read raw lines and select only the lines containing
   the exact quoted ID token. Only those candidates are classified, parsed and compared.
3. Semantics are unchanged:
   - the same ID with different canonical payload bytes still raises;
   - the same ID with an identical payload is still accepted.
4. Do not change drain eligibility, release-evidence rules or retention; that is a
   product decision.
   - Record in the bead notes what the current outbox holds (by kind and agent state).
   - If entries that are already eligible under today's policy are stuck because a drain
     path never runs, fix that path.
   - Otherwise record a `PROPOSED FOLLOW-UP:`.

**Tests**

- A ≥ 20k-line synthetic outbox in which an append performs no per-line Rust classify or
  canonicalize call (spy on the bindings).
- Collision-with-different-payload still raises; identical-payload re-append still
  works.

**Acceptance:** the bead notes record a before/after cProfile of an audited
`sase bead read`, showing ~2.9 s removed.

## Phase `maintenance`: hidden-clone gc and push-log retention

1. **Hidden-clone gc (`sase-17r`).** `maybe_gc_sidecar_clone`
   (`src/sase/sdd/_store_maintenance.py`) is only called from
   `src/sase/scripts/sase_chop_sidecar_auto_sync.py` with primary sidecar roots
   (`writable_sidecar_root`). It never sees the host-owned hidden clone
   (`hidden_sidecar_clone_dir`, `src/sase/_linked_repo_paths.py`). That clone holds
   ~2,400 loose objects (~1.9 GiB) plus 37 packs, above every threshold.
   - Extend the maintenance pass to visit the hidden sidecar clones of enabled projects.
   - Use the same try-lock semantics, and serialize with the machine artifact-link
     store's writer so gc never races a machine commit.
   - Gc stays in the scheduler chop, never inline in a user command.
2. **Stale comment.** Fix the "~5 MB" `issues.jsonl` comment (now ~21 MB).
3. **Push-log retention.** `~/.sase/bead_push_logs` holds ~126k `sync-*.log` files with
   no retention. `recent_bead_sync_log_paths` and `latest_bead_sync_log`
   (`src/sase/bead/_sync_logs.py`) glob and stat all of them.
   - Add age-plus-count retention, applied at most once per interval from the
     maintenance pass.
   - Add new config keys with sensible defaults in `default_config.yml` and the schema.
   - Diagnostics that read recent logs keep working.

**Tests:** hidden-clone visitation, lock serialization, and retention boundaries (age,
count, interval gating).

**Acceptance:** run the maintenance entry point once on athena and record
`git count-objects -vH` for the hidden clone before and after in the bead notes.

## Phase `parse-once`: one parse, one validation (sase-core)

All files are in `crates/sase_core/src/bead/`.

1. **Prune off the read path.** Remove `prune_removed_flag_event_streams` (`jsonl.rs`
   ~152-208) from `read_event_store` (~392). The read path must never delete files or
   rewrite the manifest.
   - The read path parses typed.
   - If a stream fails to parse and `classify_flag_stream` (~217) identifies a removed
     flag stream, skip it in memory and account for it in the `stream_count` check.
   - A live flag stream stays an error with today's message.
   - The physical prune runs only under the mutation lock (`MutableStore::load`,
     `mutation/store.rs` ~273) and through the existing
     `bead_prune_removed_flag_event_streams` binding and doctor repair.
2. **List the directory once** per read.
3. **No deep clone.** `validated_event_streams` (`events/reduction.rs` ~262-278) clones
   every stream (`streams.to_vec()`). Validate in place, or take ownership. Keep the
   pure `bead_reduce_event_streams` binding's behavior.
4. **Validate once.**
   - Each event is validated exactly once on the read path, at parse time. Today it is
     validated three times: `jsonl.rs` ~600, `reduction.rs` ~269 and ~284.
   - Each issue is validated once per read, for the issues the replay touched, in the
     post-pass. Today `IssueWire::validate` runs after nearly every applied event, which
     is O(notes²) for long note logs.
   - Keep validation wherever input did not come from the parse-validated path
     (Python-supplied streams).
   - A store whose final state is invalid must still fail with the same error kind.
   - If any intermediate-state check is relaxed, state it in the bead notes and cover it
     with a test.
5. **Single external-ref check.** Run `validate_unique_external_refs` once on load (it
   also runs at `mutation/store.rs` ~301), and make it O(n) instead of a linear `find`
   over a `BTreeSet`.

**Tests**

- All existing bead parity suites (`tests/bead_event_parity.rs`, `bead_read_parity.rs`,
  `bead_storage_parity.rs`) and fixtures pass unchanged.
- New tests: a removed flag stream is skipped without deletion on read and pruned on the
  next locked load; a live flag stream errors; a read leaves the directory
  byte-identical.

**Acceptance:** the bead notes record stage timing on a `/tmp` copy of the live store
(or the `bench` corpus), showing `read_store_issues` ~480 ms → ~290 ms.

## Phase `fingerprint`: store fingerprint (sase-core + sase)

At least five consumers use `issues.jsonl`'s `(mtime_ns, size)` as the store's change
token. They go stale once `projection-off` stops rewriting it.

1. **Core.**
   - Add `bead_store_fingerprint(beads_dir)` in a new
     `crates/sase_core/src/bead/fingerprint.rs`.
   - It is stat-only and exact: it covers `config.json`, `events/manifest.json`, and
     every stream file's `(size, mtime_ns, inode)`. It covers legacy `issues.jsonl` only
     for stores without `events/`.
   - It returns a stable hex token plus the counts it used. Regenerating `issues.jsonl`
     alone must not change the token of an event store.
   - Bind it in a new file under `crates/sase_core_py/src/beads/` and expose it through
     `src/sase/core/bead_read_facade.py`.
2. **Migrate the consumers.**
   - `_read_cached_bead_store` in
     `src/sase/ace/tui/widgets/_artifact_ref_entity_catalogs.py` (~103)
   - `_index_token` / `_load_raw_catalog` in
     `src/sase/ace/tui/models/wait_bead_catalog.py` (~100-139)
   - `store_mtime_key` in `src/sase/ace/tui/widgets/artifacts/beads_data_sources.py`
     (~36-44, ~129-137)
   - the bead part of the Plans key in `plans_data_sources.py` (~327-355)
   - `src/sase/ace/tui/actions/event_refresh/_surface_tokens.py` (~36-39, ~290-313)

   Keep each consumer's existing cadence and off-render-path placement (TUI perf rule
   8). Audit the other `issues.jsonl` mentions and list in the bead notes which ones
   read content (those are `projection-off`'s job).

**Tests**

- A mutation that leaves `issues.jsonl` untouched still changes the token.
- Each migrated consumer reloads on a stream change and stays cached when nothing
  changed.

## Phase `tui-board`: Beads and Plans panes (supersedes `sase-1h5`)

Read sase memory `tui_perf.md` first.

1. **One-pass grouping.** Replace the O(epics × issues) grouping in
   `load_beads_snapshot` (`src/sase/ace/tui/widgets/artifacts/beads_data.py` ~141-154)
   with a single pass that groups by `parent_id`, keeping today's ordering.
2. **No forced auto-refresh reloads.** `BeadsPane.on_refresh` (`beads_pane.py` ~149-150)
   and the Plans pane (`plans_pane.py` ~191-192) call `_request_load(force=True)` on
   every 10 s auto-refresh tick, bypassing the key. Auto-refresh ticks must not force:
   the fingerprint-based key decides. An explicit user-initiated refresh still forces.
3. **One read for the board.**
   - `load_project_beads` (`beads_data_sources.py` ~24-33) runs `list_issues`, `ready`
     and `blocked`: three replays.
   - Add a core `bead_board_snapshot(beads_dir)` that returns the issues plus ready and
     blocked IDs from one read, in a new core file (ready/blocked semantics stay in
     core), and use it.
   - Do the same for the Plans pane's bead load if it needs more than one read.

**Tests**

- Grouping parity with the old algorithm.
- An unchanged store performs no reload on a tick; a store change reloads.
- The board snapshot matches separate `list`/`ready`/`blocked` results.

**Acceptance:** with no change, a refresh does no store read. The bead notes record the
cold load time on the `bench` corpus.

## Phase `one-replay`: one store read per command

Today a targeted command pays a routing replay plus the work. Python-lane `update -n`
costs 9–12 replays, `close` 5–7, `create --parent` 5, and Rust `bead_cli_execute`
mutations 2 (`dep rm` 3).

1. **Routing without a full read.**
   - `_descriptor_for_location` (`src/sase/bead/operation_context.py` ~269-293) calls
     `project.list_issues()` (a full replay plus Python hydration of every issue) only
     to get the ID set.
   - Shorthand never routes to foreign stores, so it is local by definition. Full IDs
     route by store prefix plus stream-path probing in core: a root ID owns
     `events/streams/<id>.jsonl`, and a phase lives in its parent plan's stream; walk
     the lineage prefixes.
   - The cross-project fallback (`src/sase/bead/cross_project.py` ~146-165) follows the
     same rule.
   - The mutation's own locked resolution stays the authority for existence and
     ambiguity, so error messages are unchanged.
   - Cover relocated and tombstoned stems explicitly.
2. **Rust CLI.** Mutation handlers in `crates/sase_core/src/bead/cli/`
   (`mutate_commands.rs` open ~35, update ~83, close ~166, dep add ~235, dep rm
   ~262/282, ref add ~339, ref rm ~386, rm ~557; `create_command.rs` ~69) pre-read the
   store to resolve IDs and capture "old" snapshots.
   - Move resolution and the pre-mutation snapshot inside the mutation's single locked
     load: add or extend core entry points that accept raw IDs and return resolution
     outcomes plus before-images.
   - This also removes the resolve-outside-the-lock race.
3. **Python lanes.** In `cli_crud_update.py`, `cli_crud_lifecycle.py`,
   `cli_crud_evidence.py`, `cli_crud_create.py`, `_project_queries.py` and
   `_project_mutations_*.py`:
   - remove repeated `resolve_id` → `show` pairs (`BeadProject.show` is two replays);
   - reuse the issue returned by the mutation outcome instead of re-reading;
   - pass resolved IDs forward.

   Target: at most one read for authoring context plus the mutation's own load. Where
   the outcome already carries what is printed, the mutation load is the only read.

4. **Skip Rust deferrals.** `src/sase/main/bead_fast_path.py` must not resolve context
   and call Rust for verbs the Rust dispatcher always defers (`note`, `+1`, `snooze`,
   `doctor`, …; `crates/sase_core/src/bead/cli/dispatch.rs`).

**Tests**

- Count store reads per command, using Python binding spies and core's test-only
  `store_io_stats`.
- Assert the targets for `update` (with and without `-n`), `close` (with and without
  `--note`), `note`, `create --parent`, `open`, `dep add`/`rm`, `ref add`/`rm` and `rm`.
- All CLI output and error-text tests pass unchanged.

## Phase `read-model-store`: substrate, freshness, parity (sase-core)

Use a new module, `crates/sase_core/src/bead/read_model/`, with a facade `mod.rs` and
files ≤ 1,500 lines. Precedents are the agent-artifact SQLite index
(`agent_scan/index/`), `touch_index.rs` signatures, `fs_sig.rs`, and the goal ledger's
`goals-hot.json`.

1. **Location (P2).**
   - Use `<git-dir>/sase/bead-read-model/<key>.sqlite` (WAL), where `<git-dir>` is found
     by checking only `<beads_dir>/.git` and `<beads_dir>/../.git` (directory, or a
     `gitdir:` file), and `<key>` hashes the store's path relative to the work-tree
     root.
   - With no git dir there is no cache; plain replay serves the read.
   - Never use the name `beads.db`.
   - Provide an internal explicit-path API for tests.
2. **Schema.**
   - **meta:** schema version, a reducer version constant (`READ_MODEL_REDUCER_VERSION`,
     documented next to `reduce_event_streams`; every change to merge, apply or
     post-pass semantics must bump it), the `sase_core` crate version, the freshness
     token, the last full sweep time, and the merge frontier.
   - **streams:** stream ID, size, `mtime_ns`, inode, byte length and a content hash of
     the reduced bytes.
   - **issues:** ID primary key, encoded row, and indexed status, type, tier, parent,
     stream, `created_at` and external ref.
   - **edges:** dependencies in both directions.
   - **link provenance**, keyed by target.
   - **ID-suffix catalog** for shorthand.
   - **Persisted reducer state:** everything `apply_event` needs to resume exactly, not
     just the final issues.

   Row encoding is an internal choice. Any mismatch in schema, reducer or crate version,
   or any unreadable file, means drop and rebuild.

3. **Freshness protocol (P1).**
   - The O(1) token is the streams directory's `(mtime_ns, inode)` plus the manifest and
     `config.json` signatures.
   - If the token matches and the last full sweep is under 60 s old, serve.
   - Otherwise run a full stat sweep against the stored signatures. If nothing differs,
     serve and update the token and sweep time. If anything differs, do a full rebuild
     in this phase (`read-model-tail` adds incremental apply).
   - Capture each stream's signature with `fstat` on the same open file whose bytes were
     reduced, so signatures and content always describe one inode.
4. **Concurrency.**
   - Readers never take the bead flock.
   - Cache writes are SQLite transactions with a generation compare-and-swap: a writer
     commits only if the generation it started from is current, and never pairs newer
     signatures with older content.
   - A reader that cannot obtain the cache within a short bounded wait computes the
     replay itself and serves it without writing. It never waits on another process's
     rebuild for long, and never serves stale data.
   - Any cache error falls back to replay (debug-logged). A read never fails because of
     the cache.
5. **Integration.** `read_store_issues` and `show_issue_detail_with_options` (with link
   provenance) serve from the cache, so every existing binding benefits with no Python
   change. The pure replay path stays as the rebuild path and as the oracle for
   `--verify-cache`.
6. **Doctor.**
   - Add `sase bead doctor --verify-cache` (with a short alias): it compares a forced
     full replay against the cache and reports drift.
   - Add a cache status line to `sase bead doctor`: location, size, generation, last
     sweep.
   - Add core bindings for both, in a new binding file.
7. **Docs.** In `docs/beads.md` (~1239, ~1285-1288) and `docs/rust_backend.md`
   (~527-535):
   - document the read model;
   - fix the stale text that calls `beads.db` a SQLite cache (it is the mutation flock);
   - fix the stale "writes refresh `beads.db`" text.

**Parity harness (the core proof)**

- Run the full `bead_read_parity.rs` and `bead_event_parity.rs` query surface in both
  replay mode and cache mode, and assert identical outputs: issues, detail views and
  link neighborhoods.
- Add a randomized test: mutation sequences through real mutation functions, interleaved
  with cached reads, asserting that cache equals replay after every step.
- Run it on the `bench` corpus at ≥ 1× in CI (sampled) and 4× locally.

**Acceptance:** the bead notes record unchanged-store reads on the `bench` corpus at 1×
and 8× (token-only path), the rebuild cost, and the cache's disk footprint per scale.

## Phase `read-model-tail`: snapshot plus tail (sase-core)

`merge_stream_events` (`events/merge.rs` ~786) is a k-way merge on
`(timestamp, op priority, event_id)` that preserves order within a stream. Snapshot plus
tail therefore equals a full replay when all four of these hold:

1. every changed stream is a pure append (stored length plus content hash proves the old
   bytes are a prefix);
2. no stream disappeared or was renamed, and no config change affects reduction;
3. every new event's merge key sorts after the stored frontier;
4. the post-pass re-runs over the result.

Implementation:

- **Tail apply.** On a sweep with changes:
  - verify the preconditions;
  - parse only the appended bytes;
  - merge the tail events;
  - apply them to the persisted reducer state;
  - run the post-pass incrementally, through indexes, over touched rows: validation of
    touched issues, external-ref uniqueness, and removal cascades through the edge
    tables;
  - commit rows, signatures and frontier in one transaction.

  Event-ID de-duplication and relocation behave exactly as the reducer does today.

- **Fallback.** Any precondition failure triggers a full rebuild. Examples: a
  clock-skewed event from another machine, a relocation, a conflict resolution, a
  backdated append, or a rewritten stream.
- **Telemetry.** Count serve, tail and rebuild outcomes with reasons, and surface them
  in `sase bead doctor`'s cache status. The research never measured how often the
  frontier fails in practice across athena, apollo and mac.
- **Parity.** Extend the randomized harness with:
  - incoming events from a second simulated machine, with skewed clocks both ahead and
    behind;
  - relocations (`bead_merge_event_streams_with_relocation`);
  - conflict-resolver rewrites, removals and tombstones;
  - writes to closed beads;
  - concurrent reader/writer interleavings.

  Assert cache equals replay after every step.

- **If parity proves infeasible,** stop. Record a `PROPOSED FOLLOW-UP:` with the
  counterexample, recommending promotion of the sealed-segment design (research Phase
  3). Do not ship a cache that can diverge.

**Acceptance:** the bead notes record a local note append followed by a read on the
`bench` corpus at 1× and 8×, with the tail path taken and cost flat with history.

## Phase `seal-watch`: sealed-segment triggers (research Phase 3)

The research recommends building the physical archive only when a trigger fires.
Implement the watch, not the archive.

1. Add a `sase bead doctor` check (core computes, Python renders) that reports the
   measurable triggers:
   - hot stream files > 10,000;
   - full stat sweep > 50 ms (from read-model telemetry);
   - store working tree > ~250 MB.

   Each prints OK with the current value, or WARN with a pointer to the design section
   below. Make the thresholds core constants.

2. Document the gated design in `docs/beads.md`, summarizing research Phase 3:
   - seal fully closed root lineages quiet for ≥ 30 days that have no live dependents,
     claims or pending outbox entries;
   - move them verbatim with `git mv` into `events/sealed/<YYYY-MM>/`;
   - keep a SHA-256 manifest and a sealed index;
   - thaw by overlay stream, with the reducer de-duplicating by `event_id`;
   - the integrity guard and conflict resolver compare hot + sealed;
   - every seal commit carries a parity proof;
   - zstd bundles only as off-repo backups.

   Also list the non-measurable triggers: clone complaints that partial clones cannot
   fix, and parity proving infeasible.

**Tests:** threshold boundaries for each trigger.

## Phase `projection-off`: `issues.jsonl` off the per-mutation path

`MutableStore::save` (`crates/sase_core/src/bead/mutation/store.rs` ~310-329)
serializes, fsyncs and renames the whole 20.8 MB `issues.jsonl` on every mutation. That
file is 74% of the beads repo's packed history, the top conflict file, and the source of
the loose-blob bloat behind `sase-17r`.

1. **Consumer audit first.** Using the `fingerprint` phase's list, migrate every content
   reader before the writer stops:
   - `src/sase/core/artifact_file_protection.py` (~21-27, **P3**). Bead-referenced
     artifact IDs must come from a core query over current state, for example a
     `bead_referenced_artifact_ids` binding served by the read model. Protection
     coverage must never silently shrink. Add a regression test: an artifact referenced
     only by a bead ref stays protected after the projection is gone.
   - The conflict resolver (`src/sase/bead/conflict_resolver_store_writer.py` ~58-64,
     `conflict_resolver_paths.py`).
   - The reprojection auto-commit
     (`src/sase/llm_provider/commit_finalizer_git_autocommit.py` ~385-424,
     `src/sase/finalizers/reconciliation_auto_commit.py`). For event stores this becomes
     moot; keep or remove the legacy-layout path deliberately.
   - Core doctor's projection comparison (`bead/read.rs` ~355), which must not flag an
     absent projection.
   - `src/sase/bead/cli_admin.py` `--fix-projection`.
   - `src/sase/core/health.py`, `tools/check_bead_note_migration`,
     `tools/check_bead_store_soak`.
   - The sidecar README template (`src/sase/sdd/templates/sidecar-beads-README.md`).
   - `src/sase/config/sase.schema.json` text.

   If a consumer truly needs a tracked file, shard it instead (`issues/active.jsonl`
   plus `issues/closed-<YYYY-MM>.jsonl`) and record why.

2. **Stop writing.** Event stores no longer write `issues.jsonl` on save. Legacy stores
   without `events/` keep today's behavior.
3. **On-demand export.** Add `sase bead export` (`-o/--output`, default the store's
   `issues.jsonl`; listed alphabetically) through the existing `bead_export_jsonl`
   binding. `--fix-projection` regenerates the local, untracked file.
4. **Mixed-version-safe untracking.** Add `issues.jsonl` to
   `bead_store_gitignore_patterns` (`src/sase/sdd/_bead_ignore.py`, both layouts). Then,
   as an idempotent migration under the store lock, `git rm --cached` the tracked file
   and delete the stale local copy inside the next bead commit.
   - Old clients are safe because the commit path lists changes with
     `--exclude-standard` (`changed_sdd_files`, `src/sase/sdd/_commit_store.py` ~598):
     an old client that regenerates the ignored file never stages it.
   - If an old client's conflict resolution re-tracks the file, the next new-client
     mutation untracks it again.
   - Test both mixed-version scenarios at the git level.
   - Pages are out of scope.
5. **Docs.** Update `docs/beads.md` storage and publication sections and
   `_store_maintenance.py`'s comment.

**Tests**

- A mutation leaves no `issues.jsonl` diff.
- Export output matches today's projection byte for byte.
- The migration is idempotent and self-heals.
- Artifact protection keeps covering bead-referenced artifacts.
- The conflict resolver works with no projection.

## Phase `read-model-queries`: indexed queries (sase-core + sase)

Make hot queries touch only the rows they need. Every result must equal the replay-based
result (parity harness in both modes).

- **Detail (`sase bead read`/`show`).** `show_issue_detail_with_options` (`bead/read.rs`
  ~112, ~712-801) today clones and sorts every issue and scans for children, reverse
  deps, ancestors, shorthand and link neighborhood. Replace those scans with:
  - a point lookup;
  - the children index;
  - the reverse-dependency edge table;
  - the parent chain;
  - the suffix catalog;
  - provenance-by-target rows, which also fixes the O(links × issues) `row_touches_bead`
    resolution.
- **ready, blocked and default list.** Serve active rows only, plus status lookups for
  dependency targets. Push the list filters (status, type, tier, task type, since/until,
  limit) down into core so closed listings read only matching rows (the default closed
  listing is the newest 20).
- **Other queries.**
  - `stats`: index aggregates.
  - `resolve_id`: the suffix catalog.
  - `search`: scans cached rows with today's substring and regex semantics (no replay).
  - `history`: keeps reading streams, but only the target lineage's stream when
    possible.
  - `/sase_new_task` duplicate detection must still see closed beads.
- **Multi-get (`sase-x3`).** Add `bead_statuses_for_ids` and use it in
  `bead_statuses_for_project`. Back the `one-replay` routing and ID catalog with the
  cache.
- **Python callers.** Move Python callers that hydrate `list_issues()` only for IDs or
  statuses onto the new bindings: wait-bead catalog, artifact-ref catalogs, routing,
  plan archive health.
- **Layout.** `read.rs` is near the cap, so put the query implementations in the
  `read_model` module or new files.

**Acceptance:** the bead notes record warm point read, `ready` and default list on the
`bench` corpus at 1× and 8×.

## Phase `read-model-mutations`: load affected rows, write through (sase-core)

`MutableStore::load` (`mutation/store.rs` ~273-308) replays everything and holds
`issues: Vec<IssueWire>`. The mutation modules scan that Vec, for example:

- external-ref checks in `create.rs` ~138;
- shorthand resolution in `links.rs` and `plus_one_snooze.rs`;
- descendant checks on close;
- removal cascades;
- `next_child_id`;
- claims and epic preclaim.

The change:

1. **Store view.** Introduce an indexed view backed by the read model with these
   lookups:
   - by ID;
   - children by parent;
   - dependents by target;
   - external-ref owner;
   - suffix resolution;
   - ID allocation high-water marks.

   Port every mutation to it so a write loads only affected rows. This happens under the
   existing `beads.db` flock after a full freshness sweep, which is cheap relative to a
   write.

2. **Write-through.**
   - `save` appends event bytes exactly as today (append-preserving, with the prefix
     check), and validates only changed streams and issues.
   - It writes changed rows, edges, provenance, stream signatures and the frontier into
     the cache in the same critical section, so the next read takes the token-only path.
   - If the cache write fails, the event append still stands, and the next read repairs
     the cache through tail or rebuild.
3. **Fallback.** With no cache available (no git dir, or unreadable), keep the
   full-replay load.

**Tests**

- Every existing mutation test passes in both cached and uncached modes.
- The parity harness asserts cache equals replay after each mutation.
- Lock and contention tests are unchanged.
- `store_io_stats` proves no full replay on the cached path.

**Acceptance:** the bead notes record the binding-level `append_note` and `update` cost
at 1× and 8×.

## Phase `perf-gate`: acceptance

1. Turn the `bench` CI run from record-only into a gate on the generated corpora.
   - Criteria (A1):
     - p95 of `ready`, default `list`, detail read and a local mutation at 4× within a
       tolerance of 1×;
     - warm point read ≤ 20 ms;
     - active list ≤ 50 ms;
     - TUI board no-change refresh < 100 ms.
   - Calibrate the CI tolerance to runner noise. Keep the strict "< 10% from 1× to 8×"
     check in `just bead-perf-scale` locally.
2. Run the full local suite at 1×/2×/4×/8×, plus the remote-backed CLI mutation and
   audited `sase bead read`. Record a before/after table against the `bench` baseline,
   both in `docs/beads.md` (or `docs/perf_runbook.md`) and in the bead notes.
3. Do not loosen thresholds silently. A missed criterion gets a measured breakdown and a
   `PROPOSED FOLLOW-UP:`.

## Land

- Verify on the live sase store, read-only:
  - `sase bead doctor --verify-cache` reports no drift;
  - doctor shows the cache status and the seal-watch values;
  - `sase bead show`/`ready`/`list` outputs match replay.
- Close the superseded task beads with notes naming the verifying phases:
  - `sase-1h5` (by `tui-board`);
  - `sase-17r` (by `maintenance`, with the root cause removed by `projection-off`);
  - `sase-x3` (by `read-model-queries`).
- Triage every phase's `PROPOSED FOLLOW-UP:` notes. That includes any memory notes made
  stale; route those through `/sase_memory_write`, which needs user approval.

## Out of scope

These are explicitly not recommended by the research, or are optional:

- physical sealing itself (gated by `seal-watch`);
- compressing event files;
- `sase bead rm`/history rewrite/summary compaction;
- a bead daemon or cache-warmer proc;
- dropping closed pages from HEAD (checkout size only);
- folding non-root stream stems;
- a separate archive repo.

Fixed costs outside the store also stay out: CLI import time and the creator-URL lookup
inside `sase bead read` (~0.9 s) are not history-dependent. A phase that finds them
dominant records a `PROPOSED FOLLOW-UP:`.
