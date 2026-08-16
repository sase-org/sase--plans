---
tier: tale
size: medium
title: Per-stream bead event-store writes in sase-core
goal:
  The bead event store rewrites only the streams whose events actually changed, so a
  single bead mutation touches one stream file instead of all ~900, the append-only
  guard can no longer be tripped store-wide by one non-round-trip-stable stream, and
  `events/manifest.json`, the on-disk format, and the guard's semantics are unchanged.
proposed_by: bbugyi200.athena.sase-ns.3
bead: sase-ns.3
create_time: 2026-08-16 17:25:22
status: done
---

- **PROMPT:**
  [prompts/202608/per_stream_bead_event_writes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/per_stream_bead_event_writes.md)
- **PARENT:** [202608/top_task_bead_sweep.md](top_task_bead_sweep.md)
- **BEAD:**
  [sase-ns.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ns/sase-ns.3.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-ns.3](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-ns.3.md)
- **COMMITS:**
  - [291ea25](https://github.com/sase-org/sase-core/commit/291ea25baa1c49db70341e558160f58db8f25ecd)
    — perf(bead): write only changed event streams

# Plan

This plan implements epic phase `sase-ns.3` and closes task bead `sase-mr`. The code
change is cross-repo: it lands in the sibling `sase-core` repo.

## Repo Access

Open `sase-core` through the repo skill before reading or editing anything in it, and
use only the path it prints:

```bash
sase repo open sase-core -r "Implement per-stream bead event-store writes for task bead sase-mr"
```

Every path below is relative to that checkout unless it is called out as a `sase`-repo
path. Do not clone, path-guess, or web-fetch `sase-core`.

## What Is Wrong

`crates/sase_core/src/bead/jsonl.rs::write_event_store` re-serializes and rewrites
**every** stream file on every call.
`crates/sase_core/src/bead/mutation.rs::MutableStore::save` is its only production
caller, so appending one note to one bead rewrites all ~900 stream files in the live
store. Two observed consequences, both recorded on `sase-mr`:

1. **Blast radius.** One event that is not round-trip stable wedges the entire store
   rather than one bead. One `resolution: null` event in `sase-mk.jsonl` failed to
   round-trip, and because every mutation re-emitted that stream, the append-only
   integrity guard (`cannot publish non-append-only bead event stream …`) refused every
   commit, push, and rollback store-wide for hours. Per-stream writes would have
   confined it to `sase-mk`.
2. **Latency.** A failed `sase bead work` reported
   `slow_launch_stage phase_creation elapsed_ms=200901.7` and
   `dependency_creation elapsed_ms=170376.8`. Each stage's saves re-serialize every
   event in the store and pay an fsync per file plus a directory fsync.

The round-trip defect itself is already fixed
(`fix(bead): make issue_updated resolution round-trip stable`), so today's writes are
byte-identical and git sees no spurious diff. The cost and the blast radius are what
remains.

## Design

### 1. `crates/sase_core/src/bead/jsonl.rs` — selective writes

Split the current body into a shared helper and expose two entry points:

- `pub fn write_event_store(beads_dir, streams)` — unchanged signature and unchanged
  "write every stream" contract. It stays the migration/test entry point; the existing
  callers in `crates/sase_core/tests/bead_event_parity.rs`,
  `crates/sase_core/tests/bead_read_parity.rs`, and the `mutation.rs` test module keep
  compiling untouched.
- `pub fn write_event_store_changed(beads_dir, streams, changed_stream_ids: &BTreeSet<String>)`
  — validates and manifests over **all** `streams`, but serializes and writes only the
  streams whose `stream_id` is in `changed_stream_ids`. (Any equivalent selector shape
  is fine — an `Option<&BTreeSet<String>>` parameter on one function also works. Do not
  accept a selector of streams the caller has already filtered: the manifest and the
  validation sweep need the full set.)

Both go through one internal helper so the two paths cannot drift. Inside it:

- Keep validating **every** stream (`stream.validate()?`) before writing anything, in
  `stream_id` order. That store-wide fail-closed check is in-memory only, it is what
  keeps a bad stream from being persisted, and it must not become "validate only what we
  write".
- Iterate over a sorted `Vec<&BeadEventStreamWire>` instead of `streams.to_vec()`. The
  current deep clone of every event on every save is pure waste and disappears with the
  same edit.
- Write each selected stream through a new
  `fn write_file_atomic_if_changed(path, bytes) -> Result<bool, BeadError>` that reads
  the existing file first and returns without writing when the bytes already match. This
  eliminates no-op writes (and preserves mtimes) on both paths, and it is a second line
  of defense if a stream is ever marked changed when it is not. `write_file_atomic`
  keeps its current behavior and is what `write_file_atomic_if_changed` delegates to.
- Build the manifest from **all** `streams` exactly as today
  (`BeadEventStoreManifestWire::from_streams`), and write it through
  `write_file_atomic_if_changed` as well. `stream_count` must keep meaning "the number
  of streams the writer holds", so that `read_event_store`'s
  `stream_count != streams.len()` guard keeps its current teeth. Do not recompute
  `stream_count` by counting files on disk: that would silently paper over a lost or
  stale stream file, which is exactly what the guard exists to catch.

### 2. `crates/sase_core/src/bead/mutation.rs` — track which streams changed

`MutableStore` owns `streams: Vec<BeadEventStreamWire>`. Replace it with a small private
type that owns the vector **and** the set of stream ids that were handed out mutably, so
"changed" cannot be forgotten at a call site:

```rust
mod tracked_streams {          // private module inside mutation.rs
    pub(super) struct TrackedEventStreams { /* streams + BTreeSet<String> changed */ }

    impl TrackedEventStreams {
        pub(super) fn loaded(streams: Vec<BeadEventStreamWire>) -> Self;   // nothing changed yet
        pub(super) fn imported(streams: Vec<BeadEventStreamWire>) -> Self; // every stream is new
        pub(super) fn all(&self) -> &[BeadEventStreamWire];
        pub(super) fn changed(&self) -> &BTreeSet<String>;
        pub(super) fn stream_mut(&mut self, stream_id: &str) -> &mut BeadEventStreamWire;
    }
}
```

The private field in a nested module is the point: the only way to reach a
`&mut BeadEventStreamWire` is `stream_mut`, which records the id, so a future mutation
path cannot silently skip its write. Wire it up:

- `MutableStore::load` — the `event_store_present` branch uses `loaded(...)` (files
  already on disk match memory). The `import_issues_from_jsonl` migration branch uses
  `imported(...)` so the first save after a legacy-JSONL migration still writes every
  stream file.
- `MutableStore::stream_for_mut` — delegates to `stream_mut`, keeping its "create the
  stream when it is missing" behavior (a newly created stream is changed, so it is
  written).
- `MutableStore::save` — calls
  `write_event_store_changed(&self.beads_dir, self.streams.all(), self.streams.changed())`.
  Everything else in `save` (issue validation, unique external refs, `save_config`,
  `save_issues`) stays exactly as it is, in the same order.
- The handful of in-module test call sites that read `store.streams` become
  `store.streams.all()`.

Confirm with a grep that `stream_for_mut` is still the only mutable path into the vector
before relying on this (`self.streams` currently appears only in `save`,
`stream_for_mut`, and the `load` constructor).

### 3. What must not change

The on-disk event-stream format, the `manifest.json` schema, and the append-only guard's
semantics all stay exactly as they are — this fix is a write-selection change above
them. If implementation shows any of those three must change, that is a project-owner
decision: leave `sase bead note sase-mr "TASK NEEDS APPROVAL: …"` describing the options
and a recommendation, do every part that does not depend on it, and say so plainly.

Also do not hand-edit any version in `sase-core`: per that repo's `AGENTS.md`,
release-plz owns `[workspace.package].version`, crate versions, and path-dependency
pins.

## Tests

All Rust tests land in `sase-core`.

1. `crates/sase_core/src/bead/mutation.rs` test module — **one mutation touches one
   file.** Build a store with at least three streams, snapshot every
   `events/streams/*.jsonl` file's bytes _and_ mtime, append a note to a bead in one
   stream, then assert: the mutated stream's bytes changed, and every other stream
   file's bytes and mtime are identical. The existing helper near the batch-remove
   fixtures that reads `issues.jsonl` plus every stream file's bytes is a good starting
   point; extend it with `fs::metadata(...).modified()`. Assert bytes for "changed"
   (mtime resolution is not worth trusting for a positive) and both bytes and mtime for
   "untouched" (an untouched file's mtime cannot move).
2. `crates/sase_core/src/bead/mutation.rs` test module — **blast-radius regression.**
   After creating a multi-stream store, hand-rewrite one _unrelated_ stream file into a
   form that parses to the same events but does not re-serialize byte-for-byte (for
   example, emit one event's JSON object with its keys in a different order). Mutate a
   different bead, then assert the hand-written file's bytes are unchanged. This is the
   `sase-mk` wedge: before the fix that file is rewritten (and would trip the publish
   guard); after it, it is never touched. Name it so the next reader knows it encodes
   that incident.
3. `crates/sase_core/src/bead/jsonl.rs` unit tests — **selector and manifest.**
   `write_event_store_changed` writes only the selected streams; a selector naming a
   stream that does not exist on disk yet still creates it; after a selective write the
   manifest's `stream_count` still matches what `read_event_store` finds, so a store
   written this way reloads cleanly. Also cover `write_file_atomic_if_changed` skipping
   a byte-identical write (mtime unchanged) and performing a differing one.
4. Keep
   `write_event_store_leaves_an_unrelated_streams_bytes_unchanged_across_a_mutation` in
   `crates/sase_core/tests/bead_event_parity.rs` passing unmodified. It exercises the
   write-every-stream path and still has teeth: a stream that does not round-trip
   produces different bytes, so it is still written and the assertion still fires.
5. Confirm an existing test covers the legacy-JSONL migration path (no `events/`
   directory → first save writes every stream). If none does, add one; that path is the
   only reason `imported(...)` exists.

## Verification

1. In the `sase-core` checkout, run its full gate — never `cargo test -p sase_core`
   alone, which skips the `sase_core_py` binding tests:

   ```bash
   just check      # == ./scripts/check.sh all: fmt-check, clippy -D warnings, cargo test --workspace
   ```

2. **Scratch-store evidence, never the live store.** Copy the effective bead store
   (`$SASE_SDD_BEADS_DIR`, ~900 streams) into a scratch directory under `/tmp` with
   `cp -a`, and only ever point the experiment at that copy. Drive one mutation through
   the binding against the copy, for example:

   ```python
   import sase_core_rs as core
   core.bead_append_note("<scratch>/", "<some-bead-id>", "probe", ...)  # check the real signature
   ```

   Record, before and after the fix: wall-clock time for the single mutation, and the
   number of `events/streams/*.jsonl` files whose mtime moved (compare a `stat` snapshot
   taken before the call). Expected: ~900 files touched before, exactly one after (plus
   `issues.jsonl`, which is a whole-store projection and is out of scope), with the
   manifest unwritten because its bytes did not change. Put those two numbers in the
   bead note — they are the evidence the defect is gone. Delete the scratch copy
   afterwards.

3. In the `sase` repo workspace, rebuild the binding from the linked checkout and run
   the repo gate:

   ```bash
   just install    # builds sase_core_rs from the linked sase-core checkout
   just check
   ```

   `just check-full` is required if the scoped run escalates or reports an unusual
   selection; hand it to `/sase_monitor` with a `--next` action rather than running it
   inline.

4. No `sase`-repo source change is expected: `write_event_store` is not exposed through
   `sase_core_py`, so no Python caller or adapter has to move. If `just check` says
   otherwise, fix the caller here rather than reshaping the Rust API.

## Landing And Bookkeeping

- The `sase-core` edit is a cross-repo change that lives in a linked checkout, not in
  the `sase` tree the epic's land agent lands. Do **not** commit or push it on your own
  initiative. Report it explicitly in your final reply — repo, files, and that it is
  uncommitted — so the user or the finalizer can decide. If a commit is requested, make
  it from inside the `sase-core` checkout through `/sase_git_commit`, with a
  Conventional Commits subject (`fix(bead): …`) so release-plz computes the right
  version.
- `sase`'s `pyproject.toml` pins `sase-core-rs>=0.27.15,<0.28.0`. That window does not
  need editing: local dev installs build the binding from the linked checkout, and the
  published pin is bumped in its own commit after release-plz cuts a release. Record
  that follow-up rather than doing it.
- Task bead `sase-mr` is a standalone bead the runner does not transition. Claim it
  before implementing (`sase bead update sase-mr -s in_progress`), note what you
  changed, the commands you ran, and the before/after numbers from step 2 of
  Verification, then close it:

  ```bash
  sase bead close sase-mr --note "<what you verified>"
  ```

  If you could not finish, leave it `in_progress` with a note that says plainly what you
  tried, what you found, and what the next agent should pick up.

- Record discovered work as `PROPOSED FOLLOW-UP: <summary — detail>` notes on phase bead
  `sase-ns.3` — never as new beads. At least these two are already known:
  - `sase-li` (concurrent sync silently deleting an event in this same write path) is
    plausibly mitigated by per-stream writes: a save no longer rewrites another
    process's stream file from a stale in-memory copy. Say "plausibly mitigated,
    unverified" — do not close a bead you did not verify, and do not expand this phase
    to chase it.
  - Bumping the `sase-core-rs` pin in `sase`'s `pyproject.toml` once a release
    containing this fix exists.
- Then close **only** phase bead `sase-ns.3` (`sase bead close sase-ns.3 --note "…"`).
  Never close the parent epic `sase-ns` or any ancestor.
