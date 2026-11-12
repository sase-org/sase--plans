---
tier: epic
title: Bound Rust capacity scans while preserving admission records
goal:
  Complete sase-1sl.1 with a metadata-light capacity scanner, a persistent never-live
  cache, and measured parity and scan-cost evidence.
parent_bead: sase-1sl.1
phases:
  - id: capacity-projection
    title: Classify capacity candidates before constructing full records
    size: medium
    depends_on: []
    description:
      "capacity-projection: in linked sase-core, implement the marker precheck and
      compact metadata classification for both capacity scan entry points; preserve
      retention closure and pre-retention clan context, remove repeated archive-wide
      clan-member cloning, and add an independent legacy-path parity oracle."
  - id: capacity-verdict-cache
    title: Persist validated never-live classifications across processes
    size: medium
    depends_on:
      - capacity-projection
    description:
      "capacity-verdict-cache: persist the compact classification and closure/context
      keys for safely cacheable never-live candidates, invalidate changed metadata and
      directories, keep live-marker and PID uncertainty conservative, and prove
      corrupt-cache, concurrent-writer, and fresh-process behavior."
  - id: capacity-cost-proof
    title: Prove archive-scale scan budgets and complete the Rust change
    size: medium
    depends_on:
      - capacity-projection
      - capacity-verdict-cache
    description:
      "capacity-cost-proof: add and run a reproducible 20k-directory release-mode
      benchmark against the preserved legacy path, enforce cold and warm budgets with
      identical records and clan context, add deterministic work-count guards, and
      record the completed core revision and check evidence for adoption."
proposed_by: bbugyi200.athena.sase-1sl.1
proposed_with: codex/gpt-6-astra
create_time: 2026-11-12 17:02:07
status: wip
---

- **PROMPT:**
  [prompts/202611/capacity_scan_core.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/capacity_scan_core.md)
- **PARENT:**
  [202611/capacity_scan_tui_runaway.md](https://github.com/sase-org/sase--plans/blob/main/202611/capacity_scan_tui_runaway.md)

# Rust capacity scan implementation for sase-1sl.1

## Authority, scope, and sizing

This is a child implementation epic for phase `core-capacity-scan` of
`plan:202611/capacity_scan_tui_runaway.md`. The assigned phase was in progress with no
earlier-work notes when inspected. The linked core checkout was clean at `6111b9b5`; the
scanner still fully loads every non-done candidate before pruning. The existing
`d14a32c8` pruning change is the baseline, not completion of this work.

The parent's final decision is **verdict_cache = persist**. Implement that branch; do
not re-ask it or introduce a memory-only alternative. No memory edits are authorized or
needed. The parent's `live_tui_restart = separate_instance` remains binding, but no TUI
session work belongs here.

Three sequential medium phases separate the parity-sensitive scan algorithm, cache
correctness, and performance proof. Each phase has one implementation responsibility and
must finish its own tests. The final phase is a bounded benchmark and tuning task, not a
place to defer missing scanner or cache functionality.

All implementation belongs in the linked `sase-core` repository. Open it with
`sase repo open sase-core -r "Implement the capacity scan child phase"` and read its
`AGENTS.md` before editing. Paths below are relative to that printed checkout. Leave
Python callers, `sase-core-revision.txt`, real-archive measurement, GC policy, and live
TUI operations to the parent epic's other phases. Keep admission independent of the
artifact index and SQLite. Do not add a feature flag or change the public wire.

## Current behavior and invariants

Relevant code is in `crates/sase_core/src/agent_scan/scanner.rs`, `context.rs`,
`layout.rs`, and `wire.rs`; clan declaration selection is in
`crates/sase_core/src/agent_clan_tribe.rs`. Current coverage includes the scanner's
`capacity_only_*` tests and `crates/sase_core/tests/integration/agent_scan_parity.rs`,
registered by `tests/integration/main.rs`.

`scan_agent_artifacts` and `scan_agent_artifact_dirs` currently skip existing
`done.json` entries, construct full records, sort, compute clan context, and then call
`retain_capacity_records_for_admission`. Preserve these observable rules:

- For a fixed filesystem and process-liveness observation, serialized retained records
  and `clan_context` must be byte-identical to the old capacity scan. Preserve record
  shape, sorting, filters, canonical/day-sharded versus legacy layout precedence,
  exact-scan deduplication, and all retained marker fields.
- Existing done-marker presence semantics stay intact. Other marker uncertainty uses
  `symlink_metadata`: a dangling symlink or a non-NotFound error requires the existing
  parse/classification path. Marker presence alone is not a positive liveness verdict;
  the existing code can still prune a dead-PID waiter.
- A pending question can keep a stopped record. A markerless record without a valid PID,
  with a live PID, or with uncertain process existence cannot be dropped merely because
  it has no running/waiting marker. Preserve PID precedence, numeric coercion, range
  checking, and `process_is_provably_missing` semantics.
- Retention is the transitive closure of project/timestamp ancestors,
  project/normalized-owner-key peers, and explicit tribe declarations for hold clan
  generation keys. Keep the existing normalization and generation fallback rules. Exact
  scans must never pull closure targets from outside their candidate set.
- Clan context is computed from **all non-done candidates before retention**, including
  members subsequently pruned. Its clan/generation key rules differ from hold closure
  keys. Preserve summaries, explicit tribes, legacy parallel-family aliases, empty-value
  handling, and timestamp/identity winner selection. Apply durable `clan_records_dir`
  overrides afterward as today.
- Diagnostic counters should report actual work; avoided parses should not be counted as
  performed. Do not assert identical performance/error counts for files the optimized
  scan deliberately no longer opens. Non-capacity behavior is unchanged.

The scan will still enumerate archive entries and validate cheap metadata stamps. The
intended bound is archive-sized cheap discovery plus live/closure-sized full parsing,
not a claim that all filesystem operations become O(live records).

## Phase capacity-projection

1. Extract a compact internal capacity projection and selection pipeline into new small
   modules. Leave full record construction with the existing parser. After the existing
   done skip, probe running, waiting, and pending-question entries. Candidates with a
   possible marker use today's full parser and liveness logic. For conclusively
   markerless candidates, parse only `agent_meta.json` into the compact projection,
   select the initial live set, compute closure, and hydrate only retained candidates
   into full wire records.
2. The projection contains liveness inputs, parent timestamp, normalized owner key,
   hold-clan inputs, and the complete clan-context member projection. Share existing
   coercion/alias helpers where practical; typed deserialization must not silently
   reject values accepted by the old parser. Ignore unrelated large metadata values
   without constructing their object trees where possible. Missing/malformed metadata
   remains conservative. Keep full records already parsed for marker candidates so they
   are not parsed twice.
3. Make both multi-directory entry points use this pipeline, while preserving their
   discovery/counting differences. The singular `scan_agent_artifact_dir` performs no
   collection-wide retention today; leave that API's behavior unchanged and cover it
   with a regression test.
4. Preserve the full pre-retention clan member set without allocating full wire records
   for it. `context::resolve_clan_context` currently clones all members per key and
   resolves against the entire list. Group members by the exact existing clan/generation
   key before invoking the existing declaration resolvers, preserving sorted output and
   tie-breaks. Add multi-generation and legacy-alias parity cases.
5. Add a test-only oracle that executes the original full-parse capacity algorithm,
   including original retention and clan semantics. Keep it independent of the new
   projection/cache selection so the same omission cannot affect both sides. Compare
   serialized records and context, then runner-capacity decisions. Keep the original
   parser as the oracle's source of full wire fields.

Parity fixtures cover no-marker live/no/malformed/dead PID, stopped and unstopped
metadata, pending-question-only records, running/waiting markers with dead PID, dangling
symlinks, invalid marker JSON, permission uncertainty where portable, nested ancestors
and cycles, whitespace owner keys, cross-project isolation, shared owner keys, clan
generation fallbacks, summary-only sources, an entirely pruned clan, legacy
parallel-family metadata, duplicate timestamps across workflows, both layouts, option
combinations, exact subsets, and durable clan-record overrides. Inject process
observations where needed rather than relying on a PID remaining absent by chance.

Add new test modules instead of growing the already oversized scanner/parity files. Move
the touched capacity test section if needed; keep new files at or below 1,500 lines and
avoid unrelated file-splitting work. Finish with the targeted capacity, context, and
runner-capacity tests and the core repository check gate.

## Phase capacity-verdict-cache

1. Add an internal, versioned derived cache under the scanned projects root, using a
   reserved non-directory filename such as `.capacity_scan_cache_v1.json` so discovery
   never treats it as a project. Namespace entries by root identity and relative
   artifact path. Never put cache files inside artifact directories, where cache writes
   would invalidate their own signatures. A missing projects root remains an empty scan
   without creating it.
2. Persist the accepted directory-mtime-keyed never-live verdict plus compact closure
   and clan-context inputs. Strengthen validation with the metadata file's identity,
   size, and high-resolution modification/change stamp as available: directory mtime
   alone misses in-place writes. Reuse `fs_sig` helpers where appropriate, but an
   unavailable timestamp must not become a reusable zero stamp. Conservatively avoid
   caching symlinked or uncertain metadata unless its target changes are validated.
3. Every scan checks for possible live markers before using a negative verdict. Terminal
   stopped classifications are sticky only while all relevant signatures remain valid. A
   missing PID by itself is not permanently terminal: PID reuse and host reboot can
   change the old algorithm's result without a directory mutation. Either re-probe
   PID-only classifications every scan or leave them uncached; never turn a cached
   dead-PID observation into permanent exclusion. Closure can always re-admit an
   unchanged cached negative candidate, which must then be fully parsed.
4. Build entries from a stable observation: compare relevant signatures around the
   metadata read and do not publish a reusable entry if they changed. Failed stat,
   invalid metadata, unknown cache version, truncated or invalid cache, root mismatch,
   or unwritable cache location falls back to the uncached conservative path. Cache
   failure must never turn into an admission failure or skipped candidate.
5. Use atomic replacement with unique temporary files and no admission lock; the
   existing `fs_sig::write_json_atomic` is a candidate. Concurrent scanners may lose
   cache hits but cannot publish a valid-looking partial file or return stale
   authoritative data. Avoid rewriting an unchanged cache on every scan. Exact or
   filtered scans must not treat their subset as the complete archive when pruning
   entries; a full successful enumeration may discard entries for removed paths. Keep
   optional in-memory state scoped and bounded if introduced; disk reuse must also work
   in a new process.

Test a persisted hit in a fresh process; cache missing/corrupt/wrong-version/read-only
cases; metadata replacement and in-place edits without directory-mtime changes;
addition/removal/replacement of each marker; stopped-field and closure/context-key
changes; delete/recreate of a directory; changed PID observations without file mutation;
cold, warm, and corrupt-cache parity; closure re-admission from a negative entry; and
overlapping full/exact/filtered concurrent writers. Do not change global environment
variables in parallel tests. All fixtures use disposable roots.

Finish with targeted cache/parity tests and the repository check gate. Record the cache
format/location and invariants in code comments and the phase completion note; no new
public CLI, binding, or memory note is required.

## Phase capacity-cost-proof

Add a reproducible ignored release-mode Rust test or benchmark fixture in a new file
that can compare the test-only legacy algorithm to the optimized scanner. Prefer the
existing test runner over a new dependency or build recipe. For example, an ignored test
named `capacity_scan_archive_cost` can run with:

```sh
just test -p sase_core --release capacity_scan_archive_cost -- --ignored --nocapture
```

Build approximately 20,000 no-done historical directories plus a done-marker subset and
about 175 live/closure-relevant rows. Use valid distinct timestamps, both supported
layouts, realistic metadata/workflow-state sizes, stopped markerless history, dead PIDs,
and many clan generations. Include a second larger archive with the same live set to
detect a return to archive-wide full parsing or quadratic context resolution. Keep
fixture construction and compilation outside the measured interval.

Measure the old full-parse path, a cache-absent optimized scan, a persisted-cache hit in
a fresh process, and repeated warm scans. Measure the entire scan, including cache
read/write and context computation; do not report only the classification loop. Assert
byte-identical records/context for each mode before accepting timings. Report archive
counts, retained counts, metadata/full-record work counts, cache bytes,
baseline/cold/warm median durations, and repeat count.

Acceptance on an idle-enough host in release mode is cold at least 10x faster than the
same-fixture old path, warm below 1 second with a target in the tens of milliseconds,
and unchanged results. Keep explicit budget assertions in the opt-in benchmark, while
normal CI uses smaller deterministic work-count tests to avoid hardware-sensitive timing
flakes. Warm unchanged terminal history must not reread each metadata JSON or hydrate
pruned records. Fresh-process measurements prevent a process-local memo from masking an
ineffective persistent cache.

If these thresholds fail, profile and tune only this scanner/cache/context path; do not
lower the threshold or move admission onto the artifact index. Demonstrate the result
with a successful benchmark and the full core check before claiming completion. Record
reproducible commands/results and the final host-committed core revision for the
parent's `sase-adopt-pin` phase, which owns testing the real athena archive.

## Verification, completion, and ownership

Follow the opened core repository's current `AGENTS.md`: no bare cargo; use `just fmt`,
`just test -p sase_core <filter>`, applicable binding tests, and `sase tool run check`
from that repository. A targeted run does not replace its guarded check. Hand long
verification to `/sase_monitor` as required by the SASE single-turn contract. Do not
invoke global install or update package versions.

Each worker closes only its own assigned child phase after verification and
`sase bead epic-symbols <assigned-phase-id>`; resolve or re-key any leftover symbols to
a still-open owner first. Record discoveries as `PROPOSED FOLLOW-UP:` notes on the
assigned phase, never create task beads. A check failure reproduced identically on the
clean base is documented with that evidence and any existing tracking bead, then treated
according to the parent worker instructions rather than leaving a completed phase
abandoned.

The child epic's land agent integrates all three phases, reviews the final parity,
cache, and benchmark evidence, and completes the reserved outer phase `sase-1sl.1`
through the host's parent-phase completion workflow. Before closing that outer phase,
run `sase bead epic-symbols sase-1sl.1` and clear/re-key remaining owned symbols. Its
closing note must identify verified behavior, timings, checks, and the core revision
usable by adoption. Child phase workers must not close `sase-1sl.1`, this child epic, or
the outer epic `sase-1sl`; ancestor completion belongs to the corresponding land agent.
This proposal itself leaves `sase-1sl.1` in progress.
