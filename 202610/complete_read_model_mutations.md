---
tier: tale
title: Complete indexed bead mutations and transactional read-model write-through
goal:
  Finish sase-1h8.13 with affected-row mutation loading, safe cache publication, replay
  parity, and scaled performance evidence.
size: medium
proposed_by: bbugyi200.athena.sase-1h8.13
bead: sase-1h8.13
create_time: 2026-10-08 07:37:27
status: wip
---

- **PARENT:**
  [202610/bead_store_history_independent_performance.md](https://github.com/sase-org/sase--plans/blob/main/202610/bead_store_history_independent_performance.md)
- **BEAD:**
  [sase-1h8.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h8/sase-1h8.13.md)

# Complete sase-1h8.13: indexed mutation loading and transactional write-through

## Objective and scope

Finish the entire `read-model-mutations` phase of epic `sase-1h8`. Warm, git-backed bead
mutations must load only affected issue rows and physical event streams, append
historical-byte-preserving events, and update the SQLite read model before releasing the
existing `beads.db` flock. The next indexed read must use the token-only cache path.
Replay remains the uncached/corrupt-cache fallback and correctness oracle.

This is one bounded implementation, reusing the already-landed mutation view, partial
event writer, and incremental reducer/SQL machinery. It includes the whole mutation
surface, not only note/update. Do not treat the previous phase notes' remaining-scope
proposals as permission to defer this work again. `sase-1h8.14` owns the overall
performance acceptance gate and CI thresholds.

## Authoritative context and constraints

Before implementing, read:

- `sase bead read sase-1h8.13 -r "Need current phase scope and implementation evidence"`.
- `sase bead read sase-1h8 -r "Need final epic decisions and concurrent landing context"`.
- `sase artifact read plan:202610/bead_store_history_independent_performance.md "Need the approved read-model-mutations design"`.
- `sase artifact read research:202610/bead_history_cost_and_lossless_archival/bead_history_cost_and_lossless_archival.md "Need source research and snapshot-plus-tail correctness requirements"`.

Open `sase-core` with the `sase_repo` skill and
`sase repo open sase-core -r "Complete sase-1h8.13 mutation loading and write-through"`.
Use only the printed path and read its `AGENTS.md`. Paths below are relative to the
named repository, never to a numbered workspace. If the research ref reports missing,
open `sase--research` with an audit reason first and retry the artifact read; opening
hydrated the sidecar successfully during planning.

Honor all DECISIONS returned by the current bead reads and these approved epic
constraints:

- Backend logic belongs in `sase-core/crates/sase_core`; Python stays thin glue.
- Keep `beads.db` as the mutation flock. SQLite remains disposable under the git
  directory. Readers never take the mutation flock; no daemon or rollout flag.
- Event history stays lossless. Preserve append-prefix bytes, IDs, ordering, shorthand
  ambiguity, note revisions, evidence, link receipts, and close history. Retain the
  already-authorized retired-flag prune semantics under the mutation lock; do not
  introduce any other history deletion or rewrite.
- Do not edit memory files, provider shims, or init-memory templates. The epic
  authorizes no memory edits for this phase. Do not edit versions/changelogs.
- No manual commits, branches, or PRs; completion is host-owned. Never set the phase
  status manually or close an ancestor. Never create follow-up beads: use
  `sase bead note sase-1h8.13 'PROPOSED FOLLOW-UP: ...'`.
- Keep new files at or below 1,500 lines and module facades to module/export
  declarations. Existing read-model `store.rs` is already 2,004 lines and `tail.rs` is
  1,490; extract needed shared internals into focused sibling files instead of growing
  either with new algorithms.

## Current implementation and remaining defects

In sase-core:

- `bead/mutation/store.rs::MutableStore::load` still calls `read_event_store` plus
  `reduce_parsed_event_streams` and holds every issue and stream. `save` validates all
  issues and calls the full-slice writer.
- `bead/mutation/view.rs` already implements cached/replay indexed lookups with a staged
  row/removal overlay, but `mutation/mod.rs` test-gates it. Its replay slice is
  borrowed, `get` reopens SQLite for each lookup, and `next_top_level_counter` scans all
  cached IDs. It is groundwork, not a production-ready replacement for the store.
- `bead/jsonl.rs::write_event_store_changed_with_total` already writes lazy slices
  without corrupting manifest `stream_count`, validates selected streams, and uses the
  historical-prefix-preserving writer.
- `bead/read_model/store.rs::ensure_cache_ready_for_mutation[_at]` already forces a full
  signature sweep, even inside the 60-second token window.
- `bead/read_model/tail.rs` contains the incremental reducer, affected-row plan,
  row/edge/suffix/provenance SQL, append-prefix/frontier checks, and transactional
  publication. Calling it unchanged is insufficient: its commit path rewrites the
  complete stream-signature table and returns a fully hydrated snapshot. Extract/reuse
  its semantics without those costs.
- `mutation/tests/read_model_mutations.rs` has eight groundwork tests. `store_io_stats`
  already counts loads, saves, full replays, hydrated rows, stream reads, and validation
  runs. Existing mutation tests mostly use non-git stores and therefore do not prove
  production cache behavior.
- `MutableStore::load` calls `prune_removed_flag_event_streams`, whose implementation
  classifies every physical stream. Keeping that unconditional historical parse on the
  warm path would defeat this phase's goal.

Planning found no `--epic-symbol` entries for `sase-1h8.13`; recheck at completion.

## Implementation

### 1. Establish measurements and a genuinely lazy locked store

Record the current linked-core SHA and primary SHA, and identify the extension loaded by
the benchmark. Build/install the opened core into the primary workspace's development
venv using supported recipes when necessary; never run bare cargo or benchmark a stale
installed extension. Generate scratch corpora and measure the existing harness at scales
1 and 8 before changing the mutation implementation, with
`--runs 20 --only note_append,update`. Only generated stores or copies under `/tmp` may
be mutated for benchmarks.

Make `MutableStore` own its mutation view and fallback storage; remove the
self-borrowing slice obstacle. The cached branch must run the forced freshness sweep
under the flock, open a consistent bounded-wait SQLite view, capture its version/content
baseline, and hydrate no complete issue/stream collection. Load the replay backing only
after cache admission fails. Genuine event/config corruption must preserve the replay
error; SQLite availability/decoding faults must select a coherent replay fallback rather
than leak cache errors or mix cached and replay generations. Track rows already fetched
separately from dirty upserts and removals so repeated lookups reuse rows and reads are
not writes.

Capture enough cache identity to detect concurrent reader tail/rebuild commits and
cache-file replacement: version, generation, frontier/token/config state. The existing
generation increments only on rebuild, so generation equality by itself cannot prove
that no tail advanced the baseline. Keep lookups consistent through a read snapshot or a
verified equivalent baseline; release/revalidate it before publication as SQLite locking
requires.

Make `TrackedEventStreams` lazily load the physical stream on first event append or
receipt inspection. Cache both the parsed stream and its original bytes/
signature/length once, preserve ordinal-based event-ID minting, and remember the
appended records. Distinguish absent new streams from existing tombstoned streams.
Preserve the total physical stream count and increase it only once for each genuinely
new stream; a removed bead does not remove its stream.

Avoid classifying unchanged historical streams for retired-flag pruning. Reuse validated
cache/sweep information to select new/changed or previously skipped flag candidates,
with the complete existing prune as the cold/uncached repair path. Preserve
removal-on-next-locked-load and live-flag errors, and refresh the cache after a physical
prune. Do not skip a needed prune because a removed flag was excluded from cached stream
signatures.

### 2. Port every mutation and its relation queries

Ungate `mutation/view.rs` after its production callers are in place. Replace direct
`store.issues`, global clones, and slice-only lookup/validation helpers in all of:

- `create.rs`: parent/owner lookup, top-level and child allocation, initial
  notes/refs/dependencies, and external-ref rejection.
- `notes_update.rs`: scalar and bulk updates, append/edit/retract notes, close guards,
  and ancestor reopening.
- `close_remove.rs`: explicit close/no-op/conflicting re-close, force-close descendants,
  reopen ancestors, removal descendants, dependent cleanup, and any actual automatic
  parent behavior already implemented.
- `claims.rs`: claim/reserve/promote/release and atomic epic preclaim batches.
- `dependencies.rs`: dep add/rm, returned blocker calculation, and refs.
- `links.rs`: canonical IDs, target routing, link ownership/direction,
  add/remove/projection batches, receipts, origin, uses, and source outcomes.
- `plus_one_snooze.rs`: evidence identity/observation windows, closed-task reopening,
  promotion, snooze and cancel rules.
- Ready marking/unmarking and `close_one` helpers in `store.rs`.

Use indexed children/ancestors/dependents and dependency-target status fetches. On
removal, hydrate only surviving rows whose dependencies or bead links are affected;
canonical link targets and link provenance need the same cleanup as full replay.
Preserve descendant traversal order, response issue order, old snapshots, no-op
outcomes, errors, timestamps, and event payloads. Compare view helpers with the old
algorithms before deleting dead slice helpers.

Stage the whole batch's final candidates before checking external refs, using indexed
owners plus overlay deletions/reassignments. Do not reject a legal exchange or overlook
an unchanged outside owner. Validate only changed rows; the forced freshness admission
already validated untouched rows. All rejected batches must leave event, manifest,
config, and projection bytes unchanged.

Finish allocation with persisted indexed allocator metadata, initialized by cold rebuild
and maintained by write-through and reader-tail refresh. Preserve the actual old
base-36/top-level and decimal-child behavior, including config counter, sparse IDs,
removals, and malformed IDs; do not infer new semantics from the view's comments or
impose monotonic child allocation if replay reuses a removed maximum. The old child
allocator finds numeric direct-child IDs by prefix rather than exclusively by the
`parent` column. Keep that parity. Bump the internal cache schema if necessary so old
caches rebuild. No new public wire/binding is expected.

Keep `export_jsonl` and legacy-store migration correct: these explicitly need the
complete issue set and may materialize it through a dedicated full view. An ordinary
event-store mutation must never accidentally export a partial projection or rewrite
`issues.jsonl`.

### 3. Publish safe write-through before unlocking

Validate the final changed rows/stream events before durable writes. Save selected
streams with `write_event_store_changed_with_total` and keep existing config/legacy
projection semantics. Preserve the append-prefix check and original history bytes.

Add a focused `read_model` write-through module. Reuse/extract the existing tail reducer
and SQL helpers so row normalization, removal post-pass, edge cleanup, suffixes,
lineage, ordering, and link provenance have one definition. The ordinary note/update
path must perform neither a full replay nor a full snapshot hydration, complete ID scan,
signature-table rewrite, or blanket validation. Explicitly track the newly appended
event batch and dirty rows; validate its cache state against the reducer, not only the
imperative overlay.

Prove snapshot-plus-tail equivalence before publishing: unchanged stored prefixes,
accounted stream identities, compatible config/manifest, and merged event keys strictly
after the captured frontier. Use the production merge order, including priority and
intra-stream order. Backdated/skewed events, relocation, rewrites, or incompatible
config must take repair rather than blindly publishing the mutation's final overlay.
Local creates advance the config allocator; handle that known update coherently instead
of rejecting every create as an unexplained config change.

In one bounded SQLite transaction under the bead flock, compare the captured baseline
with current metadata and publish:

- Changed/deleted issue rows and every indexed scalar (including task type, evidence
  counts, flag marker, parent/stream, external ref, and position).
- Affected edges, suffix entries, allocator entries, reducer continuation state, and
  event-ordered link provenance/receipt state.
- Only changed/new stream signatures, byte lengths and hashes. Capture these by fstat on
  the same opened files whose persisted bytes were read; never pair a new stat result
  with old data. Preserve untouched signatures.
- Manifest/config fingerprints, the exact new merge frontier, freshness token,
  full-sweep time, and existing refresh telemetry.

Guard publication against reader refreshes, SQLite contention, cache replacement, and
out-of-band stream changes. A lost baseline comparison may accept a provably covering
winner or leave the cache stale for repair; never overwrite a newer winner. A cache
fault after durable event writes must not report the mutation as failed or cause its
retry/duplicate event. Roll back partial SQLite work and ensure a subsequent read
selects tail/rebuild/replay. Keep faults observable through existing diagnostics.
Event/config I/O errors remain real failures; disposable-cache errors alone are
fail-open.

### 4. Prove semantics, failure behavior, and performance

Run the existing mutation scenarios in both cacheable git-backed and no-git replay
modes. Refactor fixtures into reusable scenarios rather than weakening or duplicating
assertions. Include create, all update/note operations, claim batches,
close/reopen/remove, ready marks, dependencies/refs/links/receipts, evidence and snooze.
Assert byte preservation and output/error equivalence.

Extend the real-mutation randomized parity harness. After every successful operation and
rejection, compare cached issues, indexed queries, detail/link provenance and
`read_model_verify_cache` with an independent forced replay. Include writes after close,
bulk ref exchanges/collisions, overlapping removal cascades, sparse/nested child
allocation, new streams, duplicate receipts, old note encodings, external tail arrival,
clock skew, rewrites and relocation.

Add meaningful fault/interleaving tests for SQLite unavailable/busy/corrupt, cache
faults between event persistence and SQLite commit, CAS/content baseline loss,
concurrent reader refresh, and consistent lookup snapshots. The event append must stand
after a cache fault and the next read must repair exactly. Keep existing mutation
lock/timeout/contention tests and bindings tests green.

Strengthen `store_io_stats` at the actual read/reduce/hydrate/validation sites,
including cache repair, rather than only counting the removed old load call. Warm
note/update must assert zero full replays, bounded affected-row hydration, one
affected-stream load, and changed-only validation. The immediately following indexed
read must not trigger tail/rebuild. Assert untouched cache rows, signatures and event
files are unchanged; no-op calls produce no durable edits.

Run relevant inner-loop tests with `just test -p sase_core bead::mutation` and the bead
event/read/storage parity integration suites, plus targeted `sase_core_py` bead binding
tests. Never run bare cargo. Format with the core recipe. Run `sase tool run check` in
each repository changed; targeted tests are not a substitute. In sase, read
`lint_and_test.md` through the audited memory skill before finishing any tracked edit,
and use its formatting/check requirements. Do not run `check-full` unless explicitly
instructed.

Known baseline failures recorded on this phase are the event-store doctor's obsolete
`issues.jsonl missing` expectation and the editor directive contract. The parent
additionally records a Python `_ReadView.list_issue_page` test double issue. Check
current status before assuming any still exists. For a failure unrelated to this
implementation, reproduce the identical failure on the clean base without discarding
work, and append a `PROPOSED FOLLOW-UP:` with test, SHAs, output, and an existing
tracking bead if found. Such proven baseline failures do not keep this phase open; never
weaken assertions or reintroduce the deliberately removed event-store projection warning
to force green.

Use the already-existing scale benchmark after rebuilding the changed core:

```sh
.venv/bin/python tests/perf/bench_bead_scale.py --scale 1 --scale 8 --runs 20 --only note_append,update --output /tmp/read-model-mutations-after.json
```

Use the same seed/op count/core provenance for before/after, warm each corpus outside
the timing window, and record cold setup separately. Record p50/p95/max, shape,
replay/hydration counts, cache tail/rebuild fallback reasons/frequency, and
post-mutation verification. The older recorded scale-1 baseline was note p50 749 ms /
p95 1,185 ms and update p50 764 ms / p95 1,243 ms; it is context, not a substitute for
the comparable current baseline. Aim for the epic's less-than-10% 1x-to-8x p95 change.
If the mandated per-write signature sweep prevents it, provide the measured stage
breakdown and a phase note for the acceptance/land agent rather than hiding the miss or
changing thresholds.

For long commands use the SASE monitor skill. Start `sase tool run check` inline; on
exit 124 join the exact escalated run via `sase monitor start -J` with a concrete
continuation. Wait until the monitor-start command itself exits. Never finish a turn
with an ordinary shell/test/build still running.

## Completion and landing

Preserve existing public binding names/wires. If a new binding is necessary, put it in a
new focused binding file with registration/round-trip coverage and ensure sase's CI core
revision includes it. For a declaration changing both repositories the host commits core
first and updates the configured revision pin; do not manually commit to obtain a SHA.
`just ratchet-core-revision` only targets an already-landed remote core revision and
cannot pin uncommitted work. Update `docs/beads.md` only if actual mutation/fallback
behavior needs correcting; avoid unrelated Python/TUI work or changes to the
concurrently owned perf gate.

Append a phase evidence note naming the implementation, parity/fault/lock verification,
full check outcomes and any clean-base failures, and the 1x/8x before/after numbers with
durable evidence references where useful. Do not declare this phase complete while any
production mutation still depends on the full-issue Vec or safe write-through remains
unimplemented.

Immediately before closing, run `sase bead epic-symbols sase-1h8.13`. Resolve every
remaining symbol or re-key its Justfile line to the still-open parent epic or a
justified later phase; read Symvision memory if such changes are needed. Then close only
the assigned phase:

```sh
sase bead close sase-1h8.13 --note "Implemented indexed loading for the complete mutation surface and safe transactional write-through; verified cached/replay parity, failure recovery, lock behavior, affected-row/stream counters, and 1x/8x measurements; checks and any independently reproduced base failures are recorded in the phase notes."
```

Do not close `sase-1h8`, `sase-1h8.14`, or any ancestor plan created by this handoff.
Finally use the root-only `sase_final` skill as the last action of a normal ending,
declaring every repository changed for host-owned completion.
