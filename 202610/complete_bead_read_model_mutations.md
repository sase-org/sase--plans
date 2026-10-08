---
tier: tale
title: Complete indexed bead mutations and transactional read-model publication
goal: "Complete sase-1h8.13 so every ordinary git-backed bead mutation loads affected
  rows and streams, preserves replay semantics and historical bytes, and publishes its
  read-model changes before releasing the mutation flock.

  "
size: medium
bead: sase-1h8.13
proposed_by: bbugyi200.athena.sase-1h8.13
create_time: 2026-10-08 12:54:13
status: wip
---

- **PARENT:**
  [202610/bead_store_history_independent_performance.md](https://github.com/sase-org/sase--plans/blob/main/202610/bead_store_history_independent_performance.md)
- **BEAD:**
  [sase-1h8.13](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h8/sase-1h8.13.md)

# Complete the read-model-mutations phase

## Scope and authoritative context

Implement the remaining work for the already assigned, in-progress phase `sase-1h8.13`
of epic `sase-1h8`. This is one bounded implementation tale over an existing Rust read
model, tail reducer, mutation API, indexed lookup prototype, and parity harness.
Complete the whole mutation surface; the earlier note/update speedup alone does not
complete this phase.

Before implementation, reread the phase with
`sase bead read sase-1h8.13 -r "Need current phase evidence and scope"` and its parent
with `sase bead read sase-1h8 -r "Need frozen epic decisions"`. Read these artifacts
through `sase artifact read`, never by opening sidecar files directly:

- `plan:202610/bead_store_history_independent_performance.md`, especially global
  constraints, `read-model-store`, `read-model-tail`, and `read-model-mutations`.
- `research:202610/bead_history_cost_and_lossless_archival/bead_history_cost_and_lossless_archival.md`.

The epic's decisions and constraints are final. Change no memory notes, provider shims,
or memory templates. Add no feature flag, daemon, physical archive, CLI option, or
configuration knob. Keep shared behavior in the linked `sase-core` repository. Open it
with `sase repo open sase-core -r "Complete sase-1h8.13"` and read its `AGENTS.md`; use
only the printed checkout path. Never run bare `cargo`, edit versions/changelogs, or
create commits, branches, or PRs manually.

The phase is already reserved and in progress. Do not assign its status by hand, create
task beads, or close any parent or ancestor. Record discovered work using
`sase bead note sase-1h8.13 'PROPOSED FOLLOW-UP: <summary — evidence and detail>'`.
Earlier progress notes are useful evidence, but the current checkout is the authority
for what has landed.

## Verified starting point

At exploration, the linked core checkout was clean at `cd73d968`, with `7b3b9aa5`
containing the earlier indexed note/update implementation. Recheck HEAD and status
before editing; preserve any subsequently present work.

Files below are relative to the linked core checkout unless identified as SASE:

- `crates/sase_core/src/bead/mutation/store.rs`: `MutableStore::load` still loads all
  streams, reduces every event, and stores `issues: Vec<IssueWire>`. `save` validates
  every issue. Its normal event-store branch no longer writes `issues.jsonl`; preserve
  the legacy branch and explicit export behavior.
- `mutation/indexed.rs`: production fast paths for append-note and update. Other
  mutations still use `MutableStore`; external-ref changes and reopening closed beads
  fall back. It duplicates lookup logic from the prototype view. `publish_indexed_write`
  invokes forced freshness again after writing events.
- `mutation/view.rs`: test-gated cache/replay lookup prototype with staged rows and
  removals, suffix resolution, children, descendants, ancestors, dependency targets,
  reverse dependents, external refs, routing and projection receipts. Its constructor
  accepts an already loaded replay slice, and its cached top-level allocator performs
  `SELECT id FROM issues`. Those are not acceptable production lazy-load boundaries.
- `read_model/store.rs`: forced mutation freshness establishes the baseline without
  hydrating rows when already fresh. A stale refresh still materializes a complete
  snapshot. Cache versioning, bounded contention and rebuild/replay fallback already
  exist.
- `read_model/tail.rs`: reducer-backed partial row updates already exist, but
  `commit_tail` and the stat-only refresh rewrite the whole `streams` table and return
  `load_snapshot(connection)`. These hidden full-store costs are part of this phase's
  remaining work.
- `mutation/tests/read_model_mutations.rs`: existing dual-view and indexed note/update
  tests, plus test-only `store_io_stats` counters in `store.rs`.

Previous note #6 records 1x note/update p95 around 180/189 ms and 8x around 1184/1105 ms
on an earlier core plus WIP. Those are context, not a matched before/after comparison.
Notes #4/#5 record earlier clean-base failures; the parent has a more recent explanation
of the stale `issues.jsonl missing` expectation in `bead_read_parity.rs`. Reproduce
current failures before attributing them; do not assume historical failures still exist.

## 1. Capture a matched baseline and establish one mutation store API

Before source changes, install/build the current linked core into the SASE workspace's
local environment using the supported recipes. Confirm the extension path and the exact
core revision. Measure the current implementation with SASE's existing
`tests/perf/bench_bead_scale.py` harness:

```bash
.venv/bin/python tests/perf/bench_bead_scale.py --scale 1 --scale 8 --runs 20 \
  --only note_append,update --output /tmp/bead-mutations-before.json
```

Use generated disposable corpora and record seed, shape, core SHA and any dirty state.
Never benchmark by mutating the live bead store, its hidden clone, or a sidecar
checkout. Preserve the JSON as an audited artifact/evidence reference before a monitor
handoff if needed. Use `/sase_monitor` before long commands; wait for the handoff
command itself to exit.

Refactor `MutableStore` and `MutationView` into one production mutation abstraction with
cached and replay backings. Cached admission occurs inside the unchanged `beads.db`
flock, after the mandatory full freshness sweep. Hold a consistent SQLite read snapshot
and its generation/frontier/signature/config witness while retrieving baseline rows.
Instantiate the full replay backing only if cache admission fails or a proven repair
path requires it; never replay just to supply an otherwise unused constructor argument.

Expose point get/edit/stage/remove, shorthand resolution, children/descendants,
ancestors, reverse dependents, dependency targets, external-ref ownership and
physical-stream access through this API. Memoize hydrated rows and use an overlay so a
batch reads its staged final state. Preserve stable traversal and result ordering,
missing/ambiguous-ID error kinds and text, request deduplication, and before-images in
mutation outcomes. Do not silently turn SQLite/row-decode faults into semantic not-found
errors. Fall back safely before any durable append.

Lazily load only affected physical streams. Retain event ordinal/ID minting, receipt
history checks and the existing physical routing rule: a plan owns its stream; a
non-plan follows its existing parent when that is the current writer's rule. Do not
replace that routing with the read model's logical lineage-root column. Use
`write_event_store_changed_with_total` with the true physical stream total, including
additions and existing tombstone handling; preserve the manifest in a partial load. Keep
removed-flag pruning semantics without parsing every historical stream on each warm
mutation.

## 2. Add indexed allocation metadata with replay-equivalent semantics

Replace the global top-level ID scan and child-list allocation scans with disposable,
versioned allocation metadata in the read model, behind a new small module. Build it
during a full rebuild and update it transactionally during tail refresh and local
writes, including removals/collapse and config changes.

Top-level allocation remains the maximum of `config.next_counter` and the next valid
base36 top-level ID for that configured prefix. Child allocation must match
`store.rs::next_child_id`: it examines IDs with the textual `<parent>.` prefix and a
direct numeric suffix, regardless of a row's `parent_id`. The prototype's
`WHERE parent = ?1` is therefore insufficient. Cover mismatched parent fields, malformed
IDs, nested IDs, multiple prefixes, empty stores, staged creates/removes, and removal of
the maximum suffix. Preserve the replay oracle's current reuse and counter behavior; do
not introduce a different ID policy in this performance phase.

Bump the disposable cache schema version when adding tables/columns. Keep reducer and
public wire versions unchanged unless semantics truly require a coordinated change.
Rebuild older caches; no authoritative event migration is needed.

## 3. Port every ordinary mutation onto the common view

Replace whole-Vec scans and clone/revalidate cycles in these modules, preserving their
existing domain semantics through the shared API:

- `create.rs`: parent resolution, tier/creator inheritance, uniqueness checks,
  allocation, initial refs and initial notes.
- `notes_update.rs`: single and batch update, append/edit/retract notes and attachments,
  no-op outcomes, final-batch external-ref uniqueness, and reopening with ancestor
  close-history updates. Consolidate the specialized indexed note/update code so cache
  and replay use the same mutation algorithms.
- `close_remove.rs`: descendant guards, explicit force/resolution behavior, batch close
  with note, delegated-parent completion, ancestor reopening, removals and
  dependency/link cleanup of all actually affected survivors.
- `claims.rs`: wait claims, launch promotion, release, and all-or-nothing epic preclaim
  with ready/dependency checks and rollback data.
- `dependencies.rs`: add/remove dependencies, blocker status lookups and add/remove
  artifact references.
- `links.rs`: canonical target resolution, directed/undirected holder selection,
  add/remove and batch projection updates, receipt idempotency and provenance.
- `plus_one_snooze.rs`: evidence deduplication, observation-window/close semantics,
  promotions, wake behavior, snooze and cancellation.
- Ready marking/unmarking in `store.rs`.

Loading an affected subtree, ancestors, dependency targets or reverse neighbors is
legitimate; loading unrelated closed rows is not. Implement indexed validation of the
final overlay, including external-ref exchanges in a batch, instead of calling
whole-store uniqueness checks. Keep export/admin full scans explicit: exporting the
complete projection inherently needs all rows. Retain legacy/no-git replay behavior and
all existing wire/binding signatures. Do not leave routine mutation families on replay
and call the phase complete.

## 4. Publish reducer-consistent deltas under the mutation flock

Create a read-model mutation publication API, sharing the established reducer and tail
validation/post-pass helpers instead of inventing another reduction algorithm. Validate
changed candidates and streams before durable writes. Append events with the existing
byte-preserving prefix checks and persist config as required.

For an admitted append whose merge keys follow the baseline frontier, publish only its
changed rows/deletes, scalar index columns, edges, suffix/allocation entries, link
provenance and changed stream signatures. Commit these together with frontier,
manifest/config fingerprint, token, sweep time and telemetry in one SQLite transaction
before releasing `beads.db`. Capture signatures from the same handle/bytes that were
validated. Preserve the baseline full sweep; avoid the current second refresh plus
full-snapshot return on every local write.

Factor a refresh result that can report readiness/deltas without `load_snapshot`; retain
the snapshot-returning wrapper for callers that actually need all rows. Upsert only
changed stream-signature records on append/stat-only refresh instead of
`DELETE FROM streams` and reinserting history. Preserve proof of unchanged stream
membership and all current tail/rebuild preconditions.

Use a content revision/CAS witness that invalidates stale concurrent publishers: the
current tail path verifies an unchanged generation value, so generation alone cannot
distinguish two successive tail publications. Advance a content generation or compare
the complete baseline revision/frontier as appropriate. On a lost race, rollback without
pairing newer signatures with older rows; serve/repair from fresh authoritative events
rather than trusting the stale candidate. Keep bounded SQLite waits and the reader's
existing no-flock behavior.

Backdated events, rewrite/relocation, incompatible config, or other failed tail
preconditions take the correctness-preserving repair/replay path and record the reason.
Do not alter timestamps or reorder historical events to manufacture an eligible tail.
After an event append succeeds, cache failure does not turn the mutation into a
retryable cache error or append the event twice: the events stand, and a later read
tails/rebuilds/replays. Invalidate unusable cache state safely and prove that the next
read cannot serve stale rows. Preserve genuine event/config I/O errors and test the
existing partial-write recovery contract.

Keep new files at or below 1,500 lines and module facades limited to declarations and
exports. Put helpers in new files instead of enlarging the already oversized read-model
implementation files; keep any necessary extraction focused on the functions changed by
this phase.

## 5. Prove parity, affected-row work and failure recovery

Exercise the existing mutation tests in cached git-backed and uncached modes, using test
fixture support rather than duplicating two mutation implementations. Tests that commit
fixture repositories must set local git name/email under the hermetic recipe
environment. Preserve existing lock/contention tests unchanged. Extend the randomized
production-mutation parity harness to cover all families, comparing cached state, full
replay, outcomes, ordering, before-images and relevant error kinds after each operation.
Include closed-bead writes, ancestor reopen, descendant rejection, batch external-ref
exchange, removal cascades, ID allocation, claims/preclaim, links/receipts,
edited/retracted notes, attachments and +1/snooze.

Add targeted cases for no cache, legacy stores, corrupt/old cache, cache contention,
generation races, cache failure after append, no-op/invalid batch byte stability, new
streams, config changes, unsupported in-place changes within the 60-second window,
backdated/frontier failures and relocated/rewritten streams. In the cache-failure case
assert exactly one durable event and a fresh repaired read.

Extend `store_io_stats` or read-model test instrumentation to count the actual row
hydration, snapshot loads, replay, physical stream reads, validation and signature-table
writes, including work inside publication rather than only the front-door mutation. On a
warm history-shaped fixture, assert zero full replay, zero full-snapshot
deserialization, and work bounded to the affected rows/streams for representative
operations from each family. Test deltas do not rewrite untouched signature rows. A
no-op writes no event or config bytes and rejection leaves authoritative files
byte-identical. Compare persisted cache state to replay and verify that the read
immediately after a successful write uses token-only freshness.

Run targeted checks through linked core recipes, including:

```bash
just fmt
just fast
just test -p sase_core bead::mutation
just test -p sase_core bead::read_model
just test -p sase_core --test bead_read_parity
just test -p sase_core --test bead_event_parity
just test -p sase_core_py
```

Confirm test target names against the current checkout. Then run `sase tool run check`
in each repository changed, including core; targeted tests do not substitute for this
gate. Read SASE `lint_and_test.md` via the memory skill before finishing if changing
tracked SASE files. Use the monitor workflow for long checks after formatting. Run no
`check-full` unless separately explicitly requested. Fix failures introduced here
without weakening assertions. A failure reproduced identically on the clean base is
recorded as a `PROPOSED FOLLOW-UP:` with the command, base SHA and existing tracker if
any; it does not leave this phase open. Obtain clean-base evidence without losing
current changes or accessing a sibling workspace outside the repository-opening rules.

## 6. Measure, record acceptance and close only this phase

Build the final linked core into the local SASE environment and repeat the matched 1x/8x
note/update benchmark with the same seed, operation selection and 20 runs, writing
`/tmp/bead-mutations-after.json`. Record p50/p95/max, shape, revisions, host conditions
and tail/rebuild/fallback counts, with separate timing for mandatory sweep and
publication if scale remains visible. Demonstrate parity on generated 1x and 4x corpora
using a sampled mutation sequence. Any justified extension of the existing harness stays
focused on this evidence and uses the supported APIs.

Record both baseline and final evidence in the phase notes, and register durable JSON
evidence with `sase artifact create`. Report the 1x-to-8x p95 change honestly. The
epic's strict less-than-10-percent history-independence target and its `perf-gate` phase
remain intact; a missed criterion needs a measured breakdown and `PROPOSED FOLLOW-UP:`
rather than a relaxed threshold or hidden fallback. Distinguish an acceptance-gate
residual from an incomplete mutation cutover.

If a SASE consumer needs a new binding, update both sides and the core revision pin
using the documented host-owned finalization workflow. Prefer preserving the existing
bindings so the implementation remains core-only. Do not invent a future core SHA or
manually commit. Before finishing, read `/sase_final` for the primary and
opened-repository obligations; host finalizers own commits and pin updates.

Run `sase bead epic-symbols sase-1h8.13` again immediately before close (there were no
entries at planning time). Resolve every remaining entry or re-key its Justfile line to
a still-open parent/later phase, following `symvision.md` if necessary. Then close
exactly this phase:

```bash
sase bead close sase-1h8.13 --note "<implemented mutation families; cache/replay and failure-recovery evidence; check outcomes; matched 1x/8x measurement references>"
```

Do not close `sase-1h8`, the acceptance phase, or any ancestor plan bead. Return a
concise account of the completed behavior, verification and measured limitations, and
submit the root SASE final declaration as the last action before a normal turn-ending
response. A successful monitor handoff follows that skill's completion contract instead.
