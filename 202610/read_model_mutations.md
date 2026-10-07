---
tier: tale
title: Mutations load affected bead rows and write through the read model
goal:
  Complete sase-1h8.13 by removing full-store replay and hydration from eligible cached
  mutations while preserving canonical events, mutation behavior, and replay parity.
size: medium
proposed_by: bbugyi200.athena.sase-1h8.13
bead: sase-1h8.13
create_time: 2026-10-07 18:12:53
status: wip
---

- **PARENT:**
  [202610/bead_store_history_independent_performance.md](https://github.com/sase-org/sase--plans/blob/main/202610/bead_store_history_independent_performance.md)
- **BEAD:**
  [sase-1h8.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h8/sase-1h8.13.md)

# Scope and ownership

Implement the `read-model-mutations` phase assigned as **sase-1h8.13**. Its dependencies
`sase-1h8.11` (projection off the mutation path) and `sase-1h8.12` (indexed queries) are
closed. This is one cohesive Rust store refactor, sized medium for direct implementation
from this plan. Retain the existing Python mutation bindings and wire contracts; Python
remains glue.

Read the assigned bead with
`sase bead read sase-1h8.13 -r "Need the implementation scope and design"` and these
artifacts through `sase artifact read`:

- `plan:202610/bead_store_history_independent_performance.md`, especially
  `read-model-mutations` and the global constraints.
- `research:202610/bead_history_cost_and_lossless_archival/bead_history_cost_and_lossless_archival.md`.

Use `/sase_repo` and `sase repo open sase-core -r "Implement indexed bead mutations"`
from the primary checkout. Work only in the printed linked checkout and read its
`AGENTS.md`. Repository paths below are relative to their respective checkouts. Do not
edit versions, changelogs, memory files, or provider instruction files. Introduce no
daemon, rollout/backend switch, new CLI command, event-format migration, physical
archive, or loss of history.

The worker completes and closes only **sase-1h8.13**. Do not create any bead, modify its
assigned status, close `sase-1h8`, close an ancestor plan, or take over `sase-1h8.14`.
Record discovered work with
`sase bead note sase-1h8.13 'PROPOSED FOLLOW-UP: <summary — evidence and detail>'`.

# Current implementation and the resulting behavior

In sase-core, `crates/sase_core/src/bead/mutation/store.rs` defines `MutableStore` with
every `IssueWire` and every event stream in memory. `load` runs the removed-flag prune
across all stream bytes, reads all streams, and reduces everything. `save` validates all
issues and external refs. `jsonl.rs::write_event_store_changed` validates even
unselected streams and computes the manifest from the entire input stream slice.
Event-backed mutations already leave `issues.jsonl` untouched.

The read model under `bead/read_model/` already has issue, parent, dependency,
external-ref, suffix, provenance, and stream-signature indexes. `ensure_cache_ready` can
establish freshness without hydrating every row, but its warm read path skips the full
sweep for up to 60 seconds. Mutations must always sweep under the existing `beads.db`
flock. `tail.rs` already applies affected rows and updates edges and provenance,
although some paths load a compact global index and completion loads a full snapshot.
Reuse its reduction and transactional logic without importing those global loads into
the warm mutation path.

After this change, a warm note/update loads its target row, any actual related rows, and
its affected stream only. It validates the changed state, publishes canonical event
bytes, and updates the disposable cache before releasing the same flock. The next point
read serves with a token-only freshness check. Uncached or unusable-cache stores retain
replay behavior; genuine canonical-store corruption remains an error.

# Implementation

## 1. Establish evidence and an indexed mutation view

Before source edits, record the linked core revision and measure binding-level
`note_append` and `update` at scales 1 and 8 using the existing primary-repo harness:

```bash
.venv/bin/python tests/perf/bench_bead_scale.py --scale 1 --scale 8 --runs 20 \
  --only note_append,update --output /tmp/bead-mutations-before.json
```

Use only generated scratch corpora or copies; never benchmark mutations against live
stores or sidecars. Ensure the extension being measured matches the recorded core
revision. Retain the JSON and record corpus shape, p50/p95/max, sample count, and
revision in a note on the assigned bead. Route commands too long for a provider turn
through `/sase_monitor`, with a concrete continuation instruction.

Introduce a private store-view module beside `mutation/store.rs` and a private
read-model mutation adapter. Use a common view with cached and replay backings and an
in-memory overlay for loaded, changed, created, and removed rows. The fallback must use
the same mutation algorithms rather than a second implementation of each operation.
Expose lookups for:

- exact ID and shorthand resolution, preserving not-found/ambiguity errors;
- ordered direct children, recursive descendants, and ancestor chains;
- dependency targets and reverse dependents;
- normalized external-ref owners;
- top-level and child ID allocation using indexed allocation metadata;
- physical event stream ownership and projection-receipt checks.

Keep cached lookup results consistent with the overlay, including pending removals and
batch updates. Stable row handles or an ID-keyed map can replace Vec indexes; do not
make existing scans appear fast by first hydrating all cached rows. Ordering must retain
existing creation-time and replay-position tie behavior.

Force a full signature sweep on cached load inside the flock. Extend the existing
freshness adapter to reuse this sweep and establish a consistent cache baseline
(version, generation, token, signatures, config, frontier). A cold or externally changed
cache may tail/rebuild; an unchanged cache must not load a snapshot. Load stream bytes
lazily for actual event appends and receipt inspections, preserving the physical routing
rule used by `stream_id_for_issue`; the read model's lineage root is not automatically
the physical stream for every nested/relocated case.

Populate allocation metadata during rebuild and maintain it on incremental refresh and
write-through. Preserve the existing config-counter reconciliation and child-ID rules,
including removed IDs and malformed suffixes; do not silently invent a new allocation
policy. Version the disposable cache if its schema changes.

Remove the all-history retired-flag scan from warm cached loads. Remember pending
removed-flag tombstones from rebuild/classification and prune only relevant candidates
under the mutation lock, verifying their signatures. New, changed, unparseable, or
version-mismatched inputs still go through existing classification and repair. Keep the
full-prune fallback and the same live-flag error; unchanged ordinary streams must not be
read and JSON-parsed merely to prove they are not flags.

## 2. Port the complete mutation surface

Migrate `create.rs`, `notes_update.rs`, `close_remove.rs`, `claims.rs`,
`dependencies.rs`, `links.rs`, `plus_one_snooze.rs`, and ready-marking in `store.rs` to
the common view. Audit all `MutableStore` consumers beyond this directory too.

Replace full-store scans for parent existence, child allocation, shorthand, external-ref
checks, closing guards, removal cascades, epic reservations, and blockers with indexed
lookups. Load all genuinely affected rows: closing may need descendants, reopening needs
ancestors, removal needs reverse dependents and link holders, and paired links may alter
another bead. Preserve existing automatic parent transitions, force-close behavior,
reservation ownership, snooze/+1 semantics, note ordinals/attachments, refs, and
idempotent projection receipts.

For batch updates, validate every candidate and external-ref ownership against the final
overlay before writing anything. Retain atomic rejection, request/result ordering,
deduplication, unchanged IDs, old issues, and no-op outcomes. Checking each candidate
against only the unchanged database would mishandle exchanges or duplicates in the same
batch. Keep explicit JSONL export as a deliberate full-store operation; ordinary
mutations must not regenerate the projection.

## 3. Publish events and write through safely

Track dirty issues, deletions, streams, and appended events explicitly. Validate changed
rows and streams and all cross-row constraints before the first canonical write;
untouched cache rows were validated when admitted. Preserve existing append-prefix
verification, legacy event bytes, line-boundary handling, event IDs, and
temp-file-plus-rename publication. Modify/factor the partial writer so it validates only
selected streams and receives an accurate total physical stream count for the manifest.
Passing only lazy-loaded streams to today's writer would produce an invalid manifest.

After canonical publication, while still holding `beads.db`, use one SQLite transaction
to persist the dirty rows/deletes, dependency and parent edges, suffix/allocation
entries, indexed scalar columns, link provenance, affected stream signatures/content
hashes, config/manifest metadata, frontier, and freshness token/sweep time. Share
row/provenance writers with the tail path rather than duplicating reducer semantics.
Keep ordering and lineage metadata exact after create/remove; compact ordering work when
genuinely required must not hydrate all fat issue rows. Ordinary note/update must touch
no unrelated row or stream bytes.

Gate direct write-through on the existing snapshot-plus-tail equivalence conditions:
append-preserving prefixes, unchanged unrelated inputs, and valid merge ordering. Apply
newly published events in reducer order to the baseline; copying the requested issue
state alone is insufficient for provenance, post-pass effects, and backdated events.
Check the baseline generation and content/freshness identity inside the transaction and
guard the committed signatures against concurrent refresh or out-of-band changes. Never
attach newer signatures to older rows or serve a winner whose cache does not describe
the published store.

Backdated/clock-skewed events, rewritten/relocated streams, or other unproved ordering
must invalidate direct write-through and use the existing repair path. Include
same-second timestamps in tests: `now_utc` currently emits whole seconds, and merge
ordering also includes operation priority and event ID. Do not force the frontier
forward, rewrite historical timestamps, or weaken parity to manufacture a fast path.
Measure fallback frequency in the benchmark and explain any resulting scaling miss in a
proposed follow-up for the land agent.

A cache transaction/open/CAS failure after canonical publication must not report that
the event append was rolled back or repeat the mutation. Roll back partial cache updates
and leave/drop the cache so the next reader tails/rebuilds. Fail-open means a
replay-backed view of the complete authoritative store, never a mixture of partially
hydrated rows and a failed cache query. Preserve canonical I/O failures and their
existing error behavior.

Place new logic in small dedicated modules. Existing `store.rs`, `tail.rs`, and binding
facades are already large; keep new files below 1,500 lines and split reused helpers as
needed. Avoid adding unnecessary public Python bindings.

# Verification and acceptance

Add a test-only dual-mode fixture approach for the existing mutation scenarios:
otherwise identical git-backed cacheable stores and non-git replay stores. Refactor
shared scenarios where needed so cached coverage exercises the same assertions, not a
weaker alternative suite. Do not introduce a production backend-selection environment
variable. Preserve existing lock/timeout/contention tests.

Run the mutation, storage, event, and read-model parity suites in both modes. Add a
deterministic sequence harness that compares cache rows, indexed query answers, and
provenance with an independent full replay after each operation, including:

- create and ID allocation; external-ref collisions and batch candidate checks;
- notes append/edit/retract and attachments; refs/dependencies add/remove;
- links, inverse holders, and repeated projection-operation receipts;
- close/batch close rejection, force cascades, reopen and ancestor transitions;
- remove with descendant/dependent/link cleanup; claims and epic reservations;
- +1, stale evidence, snooze/wake, and writes to already-closed beads;
- shorthand ambiguity, nested parents, legacy/relocated streams, and legacy JSONL;
- monotonic, same-second, and backdated events; stale/corrupt/unwritable cache,
  generation races, and a cache write failure after successful event publication.

Extend `store_io_stats` with meaningful replay, hydration, stream-read, and validation
counters where the existing load/save/read counters cannot prove the requirement. After
warming and resetting counters, cached eligible note/update must load only affected rows
and streams, perform zero full replay, and leave the next indexed read on the token-only
path. Assert validation rejection leaves canonical bytes unchanged; historical prefixes
remain byte-identical; new-stream writes retain the correct full manifest count; partial
cache transactions cannot become fresh.

Use `just test -p sase_core <filter>` and the linked repo's supported binding/test
recipes, never bare cargo. Run formatting and `sase tool run check` in **every changed
repository**. The primary repo also requires `just check` through the wrapper if any
tracked file changes. Read `/sase_memory_read`'s `lint_and_test.md` before finishing
primary-repo changes. Do not run `just check-full`; it is not part of this phase's
requested gate. Use `/sase_monitor` for long verification, with instructions to consume
the result and finish this assigned phase.

Build/install the changed extension into the primary workspace venv using its supported
recipe and rerun the same 1x/8x benchmark under the same conditions. Exclude cache
construction explicitly from warm samples and report both raw and warm measurements,
their revisions, fallback counts, and any unavoidable sweep cost. Record before/after
p50/p95/max and sample counts in the bead notes and keep the JSON as evidence. Improve
the measured warm mutation path; do not loosen the epic's history-independence
criterion. Any criterion still missed needs a measured breakdown and
`PROPOSED FOLLOW-UP:` rather than a false pass claim. The wider acceptance/CI gate
remains sase-1h8.14's work.

If a primary Python caller requires a new binding, update its thin facade and the CI
core pin after the host-owned core commit is available, using the supported ratchet
workflow. Do not manually commit or invent an uncommitted SHA. With the existing binding
contracts preserved, this phase should primarily change core internals; record the
resulting core commit for the epic land agent's integration.

# Completion

Review all diffs and append verification/benchmark evidence to **sase-1h8.13**. Treat
UNKNOWN check failures as unresolved until investigated. A failure proven identical on
the clean base does not keep this phase open: record the exact reproduction and existing
tracking bead, if any, as a `PROPOSED FOLLOW-UP:` and close with that limitation stated.
Do not weaken tests or create follow-up beads.

Immediately before closing run `sase bead epic-symbols sase-1h8.13`. At planning time it
reported no entries. Resolve any that now exist or re-key their Justfile entries to the
still-open parent/later consuming phase, and recheck so no symbol exception expires with
this bead. Close only the assigned phase with
`sase bead close sase-1h8.13 --note "<specific verified behavior, gates, and benchmark evidence>"`.
Submit the root's `/sase_final` declaration as the last action of normal completion,
including every changed repository; leave ancestor closure to the epic land agent.
