---
tier: epic
title: Finish read-model mutations so sase-1h8.13 can close
goal: 'Every ordinary bead mutation runs one shared algorithm on an indexed mutation
  view. On a warm cache it loads only the rows and streams it affects. It writes its
  delta through to the read model inside the same beads.db critical section, with
  no second sweep and no full-snapshot load. Randomized parity shows cache equals
  full replay after every operation. Matched 1x/8x note/update evidence is recorded.
  sase-1h8.13 then closes and the waiting acceptance gate sase-1h8.14 can start. Phase
  planners also stop re-planning a phase that was left unfinished as another single-agent
  tale.

  '
parent_bead: sase-1h8.13
phases:
- id: publish-direct
  title: Direct write-through publication without a second sweep or full snapshot
  depends_on: []
  size: medium
  description: 'publish-direct: capture the epic-start note/update baseline; fix the
    stale bead_read_parity expectation so check reaches every crate; add a snapshot-free
    read-model publication API with writer-captured signatures and a content-generation
    CAS against the admission witness; route create/note/update publication through
    it; fix the forced-freshness token shortcut; prove one sweep, zero snapshot loads
    and token-only next reads.'
- id: dual-mode-tests
  title: Run every mutation suite in cached and replay modes
  depends_on:
  - publish-direct
  size: medium
  description: 'dual-mode-tests: add fixture support so the existing mutation suites
    run against a git-backed cached store and a plain replay store without macro_rules,
    check cache-equals-replay after cached-mode tests, split over-cap test files,
    and fix any parity defects the cached runs expose in the already-cached create/note/update
    paths.'
- id: view-core
  title: One mutation view with shared algorithms, and the full notes family on it
  depends_on:
  - dual-mode-tests
  size: medium
  description: 'view-core: fold indexed.rs into MutationView with cached and replay
    backings, staged events and a single commit; share event minting, lazy stream
    loading and manifest totals; rewrite create and the whole notes family (update
    batches with external-ref changes, reopen, append/edit/retract notes, attachments)
    as single algorithms; fix allocation range queries and oracle parity; close the
    io-stats counting gaps.'
- id: port-lifecycle
  title: Port open, close and remove onto the mutation view
  depends_on:
  - view-core
  size: medium
  description: 'port-lifecycle: move close_remove.rs (open, close, close with note,
    descendant guards, delegated-parent completion, ancestor reopening, removal cascades
    and survivor dependency cleanup) onto the view with affected-row-only work, dual-mode
    tests and cached-path counters.'
- id: port-claims-deps
  title: Port claims, ready marking and dependencies onto the mutation view
  depends_on:
  - view-core
  size: medium
  description: 'port-claims-deps: move claims.rs (wait and launch claims, release,
    all-or-nothing epic preclaim), ready marking in store.rs, and dependencies.rs
    (dependency and reference add/remove, blocker status) onto the view with affected-row-only
    work, dual-mode tests and cached-path counters.'
- id: port-links-evidence
  title: Port links, +1 and snooze onto the mutation view
  depends_on:
  - view-core
  size: medium
  description: 'port-links-evidence: move links.rs (canonical targets, undirected
    holders, projections, receipts, provenance) and plus_one_snooze.rs (+1 evidence,
    promotions, snooze, cancel) onto the view with affected-row-only work, dual-mode
    tests and cached-path counters.'
- id: proof
  title: Parity, affected-row and failure-recovery proof plus matched 1x/8x evidence
  depends_on:
  - port-lifecycle
  - port-claims-deps
  - port-links-evidence
  size: medium
  description: 'proof: clean up after the parallel ports, prove no ordinary mutation
    still replays, extend the randomized production-mutation parity harness to every
    family, cover the remaining failure and edge cases, assert bounded work on a history-shaped
    fixture, run sampled corpus parity, rerun the matched 1x/8x benchmark, and record
    phase-acceptance evidence for sase-1h8.13.'
- id: planner-guard
  title: Steer re-planned unfinished phases toward a child epic
  depends_on: []
  size: small
  description: 'planner-guard: extend the built-in work_phase_bead prompt so a planner
    whose phase already holds an earlier agent''s unfinished increment authors a child
    epic instead of another single-agent tale; pin the prose with a test and update
    the bead-work docs.'
proposed_by: bbugyi200.athena.0yi
create_time: 2026-10-08 14:46:00
status: done
bead_id: sase-1h8.13.1
---

- **PROMPT:** [prompts/202610/finish_read_model_mutations_child_epic.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_read_model_mutations_child_epic.md)
- **BEAD:** [sase-1h8.13.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1h8/sase-1h8.13.1.md)

# Plan: Finish the read-model-mutations phase (sase-1h8.13) as a child epic

## Why this epic exists

Phase `sase-1h8.13` ("Mutations load and write through the read model", size `large`) of
epic `sase-1h8` has been planned and implemented three times. Each round was a single
`medium` tale. Each coder landed about one increment, recorded "Bead NOT closed" with
the remaining work, and declared `keep`:

- `4d5cf650`: groundwork. Added a test-gated view, forced-sweep mutation freshness, a
  partial writer that keeps the full manifest count, and `store_io_stats` counters.
- `7b3b9aa5`: added the indexed note-append and update fast path, plus tail-refresh
  write-through.
- `1ff43605`: added allocation metadata (schema 3), ungated `MutationView`, made tail
  signature writes upsert-only, added a `content_generation` CAS and a shared
  `mutation/publish.rs`, and ported `create_issue`.

Nothing mechanical blocked the close. The phase has no `--epic-symbol` leftovers, and no
close was ever attempted or refused. The cause was under-tiering: the remaining scope
never fit one coding turn. The built-in phase prompt also says "nothing relaunches a
phase left open". So the phase stays `IN_PROGRESS`, and `sase-1h8.14` and
`sase-1h8.land` sit `WAITING` behind it.

This plan splits the remainder into phases that each fit one agent. It is a child epic
of `sase-1h8.13` (`parent_bead`). Landing it closes `sase-1h8.13` through the land agent
and the child-epic close cascade, which releases the acceptance gate. Phase
`planner-guard` addresses the root cause so a relaunched, partly done phase is not
squeezed into another single tale.

## Authoritative context

Read these before starting any phase:

- `sase bead read sase-1h8.13 -r "<why>"` for the phase scope and progress notes #1–#7.
  Notes #2 and #3 are the remaining work this epic absorbs.
- `sase bead read sase-1h8 -r "<why>"` for the epic and its DISCOVERED ISSUE notes. Note
  #3 is the `bead_read_parity` expectation fixed in `publish-direct`.
- Through `sase artifact read`:
  - `plan:202610/bead_store_history_independent_performance.md`: the global constraints
    and the `read-model-store`, `read-model-tail` and `read-model-mutations` sections.
  - `research:202610/bead_history_cost_and_lossless_archival/bead_history_cost_and_lossless_archival.md`.
  - `plan:202610/complete_bead_read_model_mutations.md`: the last tale. Its sections 1–6
    are the detailed requirements this epic distributes.

The parent epic's decisions and global constraints are final and apply to every phase:

- Shared logic lives in the linked `sase-core` repository's `sase_core` crate. Open it
  with `sase repo open sase-core -r "<why>"`, use only the printed path, and read its
  `AGENTS.md`.
- No feature flag, daemon, CLI option or config knob.
- Never run bare `cargo`.
- Never edit versions or changelogs.
- `beads.db` stays the mutation flock.
- No Python fallback or backend switch.
- Never weaken an assertion to get a green run.

## Verified starting point (sase-core `1ff43605` = `origin/master`, clean, at planning time)

Paths are relative to the linked sase-core checkout. Line numbers are approximate.
Re-verify `HEAD` and status first.

**Mutation entry points by path:**

- Through `MutationView` (cached, with replay fallback): `create_issue` (`create.rs`).
- Through `mutation/indexed.rs`: `update_issues`/`update_issue` and `append_issue_note`.
  They fall back to replay for external-ref changes, for reopening a closed bead, and
  for missing streams or manifests.
- Still on full replay (`MutableStore::load`):
  - `edit_issue_note`, `remove_issue_note`
  - every `close_remove.rs`, `claims.rs`, `dependencies.rs`, `links.rs` and
    `plus_one_snooze.rs` entry point
  - `set_ready_to_work` (mark/unmark ready) in `store.rs`

**Duplication:**

- `indexed.rs` repeats the view's lookups. Its `fetch_descendants` is pre-order, while
  replay and the view are post-order.
- It keeps a private publish copy, `publish_indexed_write` (~306–361), that takes no
  witness.
- Event minting exists four times: `create.rs` `push_event`, `indexed.rs`
  `LazyStream::append_event`, the inline minting in `indexed.rs` (~568–592), and
  `MutableStore::append_issue_event`.
- The manifest-total logic exists twice: inline in `create.rs` (~445–456) and as
  `indexed::manifest_stream_total`.
- `admit_indexed_mutation` turns store errors into `None`, while
  `MutationView::load_cached` propagates them.

**Publication cost:**

- After an append, `publish.rs` and the `indexed.rs` copy call
  `ensure_cache_ready_for_mutation_at` again.
- That sweep finds the cache stale, goes through `rebuild()`, tails via `commit_tail`,
  runs a post-commit guard sweep, and ends in `load_snapshot`. That deserializes every
  row, and the result is discarded.
- Net effect per warm write: up to four full stat sweeps and one full-row
  deserialization, none of it counted by `store_io_stats`.
- The CAS witness (`CacheWitness`: generation, frontier, token) lacks
  `content_generation`.

**Freshness bug:** the `Stale` branch of `ensure_fresh_forced` (`read_model/store.rs`
~877) calls `rebuild(.., force=false)`. `rebuild()` has the token-only fast path
(~566–586), so a mutation admission can serve a stale cache inside the 60 s window. This
breaks the forced-sweep guarantee.

**Snapshot coupling:** `commit_tail` (`tail.rs` ~1070) and `commit_sweep_refresh` (~746)
always return `load_snapshot`. Only `cached_store_snapshot[_at]` (`read_store_issues`,
detail fallback), `rebuild_read_model_at` and `open_and_serve` need all rows.

**Hot-path waste:**

- `backfill_meta_keys` runs 16 autocommit inserts on every freshness call.
- The allocation recompute uses `LIKE` (`alloc.rs` ~272/311, `view.rs` ~688/775). `LIKE`
  cannot use the BINARY primary-key index, and `%`/`_` are not escaped.
- `alloc.rs` rejects any ID containing `.`, while the replay oracle (`mutation/store.rs`
  ~582–597) rejects `.` only in the suffix.

**File sizes:**

- Over the 1,500-line cap: `read_model/store.rs` (2,099), `read_model/tail.rs` (1,519),
  `mutation/tests/notes_update.rs` (1,501).
- Near the cap: `mutation/tests/links.rs` (1,450), `queries.rs` (1,403).
- Put new code in new files, and extract only the functions a phase changes.

**Tests:**

- The general mutation suites
  (`mutation/tests/{claims,close,create,delegation_remove,dependencies,links,`
  `notes_update,snooze_plus_one,store}.rs`) never create a `.git`, so they only ever
  exercise replay.
- `mutation/tests/read_model_mutations.rs` holds the 16 dual-mode, indexed and create
  tests.
- `crates/sase_core/tests/bead_read_model_parity.rs` holds the randomized, adversarial
  and bench-corpus parity tests.

**Known check failure:** `crates/sase_core/tests/bead_read_parity.rs` (~486) still
asserts that `bead_doctor` warns "issues.jsonl missing" for an event store. `7df86f4a`
(sase-1h8.11) deliberately limited that warning to legacy stores (`read.rs` ~537–541).
This fails `just check` fast, before `sase_core_py` and the later crates run.
`editor::directive::tests::contract_covers_the_audited_directive_matrix` (sase-1h8.13
note #5) appears fixed by `ec92ecce`.

**Prior measurements:**

- Note #1, core `d2a56b4`, 1x: note p95 1185 ms, update p95 1243 ms.
- Note #6, `ec92ecce` + WIP:
  - 1x: note p95 180 ms, update p95 189 ms.
  - 8x: note p95 1184 ms, update p95 1105 ms.

These are context only, not a matched before/after.

## Rules for every phase

- Close only your own phase bead. Never close `sase-1h8.13`, `sase-1h8`, this child epic
  or any ancestor; this epic's land agent owns those. Create no beads. Record discovered
  work as `sase bead note <your-phase-bead> 'PROPOSED FOLLOW-UP: <summary — evidence>'`.
- Do not edit `sase/memory/**`, provider shims or memory templates.
- Keep shared behavior in sase-core. Keep existing bindings and wire signatures. No sase
  pin move is needed unless sase code calls a new binding; host finalizers own commits
  and pin updates.
- New files stay at or under 1,500 lines, and `mod.rs` files stay facades. Do not grow
  over-cap files. Move only the functions you change into new files.
- Semantics are fixed by the replay oracle:
  - outcomes, before-images, ordering, deduplication, error kinds and error text
  - event minting and ordinals, and receipt checks
  - physical stream routing: a plan owns its stream, and a non-plan follows its parent's
    stream per the current writer rule; never the read model's logical lineage column
- Loading an affected subtree, ancestors, dependency targets or reverse neighbors is
  legitimate. Loading unrelated closed rows is not.
- Legacy, no-git and unreadable-cache stores keep the full-replay behavior byte for
  byte.
- After a durable append, a cache problem never fails the mutation, never retries it and
  never appends twice. The events stand, and the next read tails, rebuilds or replays.
- Never benchmark against the live bead store, its hidden clone or a sidecar. Use
  generated corpora under `/tmp`.
- Verify with linked-core recipes:
  - `just fmt`, `just fast`
  - targeted runs: `just test -p sase_core bead::mutation`, `bead::read_model`,
    `--test bead_read_parity`, `--test bead_event_parity`,
    `--test bead_read_model_parity`, and `just test -p sase_core_py`
  - then `sase tool run check` in every repository you changed
- Hand long commands to `/sase_monitor`. Run no `check-full`.
- A failure that reproduces identically on the clean base becomes a
  `PROPOSED FOLLOW-UP:` and does not block your close. Obtain clean-base evidence
  without losing your changes.
- Before closing, run `sase bead epic-symbols <your-phase-bead>` and resolve any
  entries.

## Phase `publish-direct`: direct write-through publication

1. **Baseline first, before any source change.**
   - Build the current linked core into the sase workspace's `.venv`. Previous runs used
     `SASE_ALLOW_STALE_CORE=1 just rust-install "$PWD/.venv"` because the linked core is
     ahead of the pin.
   - Through `/sase_monitor`, run:
     `.venv/bin/python tests/perf/bench_bead_scale.py --scale 1 --scale 8 --runs 20 --only note_append,update --output /tmp/bead-mutations-epic-before.json`
     using the default seed.
   - Register the JSON with `sase artifact create` (read the `sase_artifacts.md`
     reference memory first).
   - Note p50/p95/max per op and scale, the corpus shape, the core SHA and the host load
     on your phase bead.
2. **Fix the stale parity expectation.**
   - In `crates/sase_core/tests/bead_read_parity.rs` (~486), assert that an event store
     does _not_ get the "issues.jsonl missing" warning. This is the behavior `7df86f4a`
     intended.
   - Keep or add a legacy-store assertion that the warning still appears.
   - Confirm the directive-matrix test passes.
   - Note on your bead that sase-1h8 DISCOVERED ISSUE #3 is fixed, so the `sase-1h8`
     land agent can skip it.
3. **Writer-captured signatures.**
   - `jsonl::write_event_store_changed_with_total`, or a sibling it delegates to,
     returns each written stream's signature: size, `mtime_ns`, inode, byte length and
     the read model's content hash.
   - Capture it from the same handle and bytes it wrote. That handle's inode survives
     the rename.
   - The next sweep must therefore see those streams as unchanged, with no re-read and
     no re-hash.
4. **Publication API** (new file, for example `read_model/publish.rs`).
   - **Input:** the cache path, the admission witness, the appended and
     already-validated events per stream, the writer signatures, and the post-write
     config.
   - **One transaction:**
     - CAS the stored generation, `content_generation`, frontier and token against the
       admission witness. Add `content_generation` to `CacheWitness`.
     - Require every appended merge key to sort after the stored frontier.
     - Reject a non-counter config change.
     - Apply the events through the existing reducer resume and incremental post-pass.
       Factor `try_tail_apply`'s plan/load/apply/resume core and `commit_tail`'s row,
       edge, suffix, allocation, provenance and signature writers into shared helpers,
       so publication and read-side tail produce identical rows.
     - Upsert only the changed stream signatures.
     - Write the frontier, config fingerprint, token (recomputed after the write),
       outcome counters and a `content_generation` bump.
     - Keep `last_sweep_ns` at the admission sweep time, so out-of-band writers stay
       bounded by the 60 s rule.
   - **Output:** `Published`, `Skipped` (no cache, or a lost CAS) or
     `Invalidated(reason)`. No `load_snapshot`, and no full sweep.
   - **On a lost CAS:** write nothing. The append already changed the token, and the
     read path's guarded tail repairs.
   - **On a backdated, config or fault failure:** invalidate safely, so the next read
     cannot serve stale rows.
   - Return reducer-truth corrections from the resumed upserts, without reopening the
     database.
5. **Callers.**
   - Route `mutation/publish.rs` (used by create) and `indexed.rs`'s private copy
     through the new API, and delete the sweep-based copies.
   - Keep the fail-open contract.
6. **Snapshot-free refresh.**
   - Make the tail and sweep-refresh decisions return readiness and deltas.
   - Keep a snapshot-returning wrapper only for `cached_store_snapshot[_at]`,
     `rebuild_read_model_at` and `open_and_serve`. Indexed queries and admission use
     readiness only.
7. **Freshness bug.**
   - Route `ensure_fresh_forced`'s `Stale` branch to a forced refresh that reuses the
     sweep it just did. Admission must never take the token-only shortcut.
   - Run `backfill_meta_keys` only after schema creation or a version mismatch.
8. **Telemetry.**
   - Count publications through the existing outcome counters, using
     `last_refresh_reason` for publish and invalidation reasons.
   - Do not change the status wire version. If a change proves unavoidable, follow the
     `AGENTS.md` wire-change recipe in both repositories within this phase.
9. **Instrumentation.**
   - Extend the test-only `store_io_stats` with full sweeps, snapshot loads, signature
     rows written and published rows.
   - Count them inside the read model, not only at the mutation front door.
10. **Tests** (new test file).
    - For warm create, note append and update:
      - exactly one full sweep (admission) and zero snapshot loads or full replays
      - signature rows written equals the number of changed streams
      - the next `read_store_issues` and detail read are token-only serves
      - cache equals replay
    - A backdated append invalidates, and the next read equals replay.
    - A simulated concurrent publication between admission and publish produces no stale
      pairing.
    - A cache fault after the append leaves exactly one durable event, the mutation
      succeeds, and the next read is fresh and equal to replay.
    - An in-place change inside the 60 s window is caught at admission.
    - Existing read-model and parity suites pass unchanged.

## Phase `dual-mode-tests`: every mutation suite in both modes

1. Add fixture support in `mutation/tests/support.rs`, without `macro_rules!`, that runs
   each existing suite against:
   - a git-initialized store, so the read model is used. Fixtures that commit set local
     `user.name`/`user.email`.
   - a plain store, which uses replay.

   The suites are
   `mutation/tests/{claims,close,create,delegation_remove,dependencies,links,notes_update,`
   `snooze_plus_one,store}.rs`.

   Candidate mechanisms:
   - including the suite modules twice under mode-specific parent modules with `#[path]`
   - a mode-parameterized fixture plus a per-test helper

   Choose the least invasive one. Lock and contention tests stay unchanged.

2. In cached mode, assert after each mutating test that the read model equals a full
   replay. A fixture check or helper is fine, and so is a non-panicking-on-unwind
   `Drop`.
3. Families that are still replay-only pass through their replay fallback until ported;
   that is expected. For create, note append and update, also assert that the cached
   path ran: zero full replays.
4. Split `mutation/tests/notes_update.rs` (1,501 lines) if you touch it. Keep every test
   file at or under 1,500 lines.
5. If the cached runs expose a parity defect in the cached create, note or update paths,
   fix it with a regression test. Never weaken an assertion.

## Phase `view-core`: one view, one algorithm, the whole notes family

1. **One mutation abstraction.** `MutationView` (rename if clearer) owns:
   - **Backings:** a cached backing, admitted inside the flock after the forced sweep
     and holding the admission witness, and a replay backing built from
     `MutableStore::load`. Build the replay backing only when admission fails or a
     repair path needs it.
   - **Lookups** (memoized, overlay-aware):
     - get, resolve (shorthand, missing and ambiguous)
     - children (replay order: `created_at`, then ID), descendants (replay post-order),
       ancestors
     - dependency targets, reverse dependents, external-ref owner
     - allocators, stream routing, receipt checks
   - **Staging:** rows, removals and events, with lazy loading of only the affected
     physical streams.
   - **A single `commit`:**
     - Validate the changed candidates and the final-overlay external-ref uniqueness
       through indexes, including batch exchanges.
     - Write the changed streams with the true physical stream total.
     - Persist the config.
     - Publish through `publish-direct`'s API and apply reducer-truth corrections.

   On the replay backing, `commit` keeps today's `MutableStore::save` semantics exactly.
   Store and decode faults stay errors and never become not-found; only cache faults
   before the append fall back to replay.

2. **One copy of each helper.** Fold event minting, lazy stream loading and the manifest
   total into one helper each. Delete `indexed.rs`'s duplicate lookups, its admission
   and its publish code. Give the view a cache-path accessor.
3. **Single algorithms.**
   - Rewrite `create_issue` (top-level, child, plan; refs; initial notes) as one
     algorithm over the view.
   - Do the same for the whole notes family:
     - `update_issues` and `update_issue`: batches, external-ref changes, no-op outcomes
       that write no bytes, and reopening with ancestor close-history updates
     - `append_issue_note`, `edit_issue_note`, `remove_issue_note`, and attachments
   - Use the descendants lookup instead of
     `reject_unclosed_descendants_in_batch(&store.issues)`.
   - Remove the external-ref and reopen fallbacks.
4. **Allocation parity.**
   - Replace the `LIKE` recompute with index range scans:
     `id >= '<prefix>-' AND id < '<prefix>.'` for top-level IDs, and `'<parent>.'` up to
     `'<parent>/'` for children.
   - Match the oracle's treatment of `.` in prefixes.
   - Keep the replay oracle's counter and reuse behavior.
   - Bump the cache schema only if storage changes.
   - Test mismatched parent fields, malformed and nested IDs, multiple prefixes, empty
     stores, staged creates/removes, and removal of the maximum suffix.
5. **Counters.**
   - Count hydrated rows for children and external-ref lookups.
   - Stop counting a stream read for a brand-new stream.
   - Distinguish fallback loads from admission.
6. **Tests.**
   - The dual-mode suites and `read_model_mutations.rs` stay green.
   - Add warm-path assertions for every notes-family entry point: zero full replays and
     snapshot loads, with hydrated rows and stream reads bounded by the affected set.
   - Add ordering-parity tests for `created_at` ties.

## Phases `port-lifecycle`, `port-claims-deps`, `port-links-evidence`: the remaining families

These three run in parallel after `view-core`. Each rewrites its family as single
algorithms over the view, so both backings run the same code and no ordinary mutation in
the family calls `MutableStore::load` directly. Each adds warm-path assertions for every
entry point: zero full replays and snapshot loads, one admission sweep, and bounded
hydrated rows and stream reads. Its existing suites must stay green in both modes.

To stay merge-safe:

- Keep family-specific helpers in your family's files. Inherent `impl MutationView`
  blocks or free functions are fine.
- Do not edit `view.rs`, the publication modules or `tests/support.rs` unless a
  genuinely shared lookup is missing. If so, make one minimal addition.
- Put new tests in a new family-named test file when the existing one is near the cap.
- Delete helpers that your port leaves unused.

**`port-lifecycle`** (`close_remove.rs`):

- Port `open_issue`, `close_issues` and `close_issues_with_note`, including:
  - descendant guards, and the force/resolution behavior
  - batch close with a note
  - delegated-parent completion, using the children lookup instead of an all-issue scan
    (~389–404)
  - ancestor reopening, using the ancestors lookup instead of `position` scans
- Port `remove_issues` and `remove_issue`, including:
  - plan cascades via descendants, with replay-identical `cascade_removed_issue_ids`
    order
  - survivor dependency cleanup via reverse dependents (`edges_dst`) instead of
    iterating every issue
- Tests prove that published deletes update edges, suffixes and allocation, including
  removing the maximum suffix.

**`port-claims-deps`** (`claims.rs`, ready marking in `store.rs`, `dependencies.rs`):

- Port `claim_for_agent_launch`, `claim_for_agent_wait` and `release_agent_claim`.
- Port `preclaim_epic_work_plan`: all-or-nothing, with ready and dependency checks and
  rollback data. Claims keep raw-ID lookups with no shorthand resolution.
- Port `set_ready_to_work` (mark/unmark).
- Port `add_dependency`, `remove_dependencies`, `add_bead_references` and
  `remove_bead_references`. Compute blocker status (`active_blocker_ids`) from
  dependency-target point lookups instead of a whole-store status map.

**`port-links-evidence`** (`links.rs`, `plus_one_snooze.rs`):

- Port `add_bead_link`, `set_bead_link_projection(s)` /
  `apply_prepared_link_projections` and `remove_bead_link`. Cover:
  - canonical `bead:` target resolution via the view, and undirected holder selection
  - removable-link collection, receipt idempotency through the receipt lookup, and
    provenance
- Port `add_task_plus_one`, `snooze_task` and `cancel_task_snooze`. Cover evidence
  deduplication, the observation window and close semantics, promotions, and wake
  behavior.
- `tests/links.rs` is at 1,450 lines, so add new tests in a new file.

## Phase `proof`: parity, bounded work, recovery, measurement

1. **Cleanup.**
   - Run `sase tool run check` on merged master first. Fix any dead code, clippy
     findings or merge drift left by the three parallel ports.
   - Confirm by search that `MutableStore::load` is reached only by the replay backing
     and the explicit full-scan admin/export paths.
2. **Randomized parity.**
   - Extend the randomized production-mutation sequence in
     `crates/sase_core/tests/bead_read_model_parity.rs`, or a new sibling file, since
     that one is 1,130 lines.
   - Cover every family:
     - creates (top-level, child, plan)
     - update batches with external-ref exchange; note append, edit and retract;
       attachments
     - close, open and close-with-note, including descendant rejection, ancestor reopen
       and delegated completion
     - removal cascades; claims and preclaim; dependencies and references
     - links and receipts; +1, snooze and cancel
     - writes to closed beads, and ID allocation including removal of the maximum suffix
   - After every operation, compare the cached state with a full replay. Also compare
     outcomes, before-images, ordering and error kinds.
3. **Edge cases.** Ensure each case below is covered somewhere, and add what is missing:
   - no git dir; a legacy store; a corrupt or old-version cache
   - cache contention with bounded waits; a generation or content-generation race
   - a cache failure after the append, asserting exactly one durable event and a fresh
     repaired read
   - a no-op, and an invalid batch that leaves bytes untouched
   - new streams; counter-only versus other config changes
   - an unsupported in-place change inside the 60 s window
   - backdated or frontier failure; relocated and rewritten streams
4. **Bounded work on history.**
   - On a history-shaped fixture with many closed lineages, run a representative
     operation from each family.
   - Assert zero full replays and snapshot loads, one admission sweep, and hydrated rows
     and stream reads bounded by the affected set.
   - Assert that no untouched signature row is rewritten and that the next read is
     token-only.
5. **Sampled corpus parity.**
   - Extend the bench-corpus sampled parity to a sampled mutation sequence on generated
     `/tmp` corpora.
   - Run 1x in the normal suite. Run 4x locally, slow-gated like the existing
     bench-corpus tests.
6. **Matched measurement.**
   - Rebuild the final core into the sase `.venv` and rerun `publish-direct`'s exact
     benchmark command, writing `/tmp/bead-mutations-epic-after.json`. Register it as an
     artifact.
   - Record a before/after p50/p95/max table at 1x and 8x, the corpus shapes, core SHAs,
     host conditions and publish/tail/rebuild counts. State the honest 1x→8x p95 ratio.
   - If 8x is still visibly slower, record a breakdown separating the admission sweep
     from publication and the binding overhead.
   - A miss against the parent epic's "< 10% from 1x to 8x" target becomes a measured
     `PROPOSED FOLLOW-UP:` for `perf-gate` (`sase-1h8.14`). Never relax the threshold.
7. **Acceptance note.** Write a phase-acceptance note on your phase bead covering:
   - every family ported, and write-through with fallback
   - the test and check outcomes
   - the evidence artifact references

   The land agent copies this note into `sase-1h8.13`'s close note.

## Phase `planner-guard`: stop re-planning unfinished phases as one tale

This is a sase repository change. Read the `lint_and_test.md` reference memory via
`/sase_memory_read` before finishing.

1. In `src/sase/default_config.yml`, extend `bd/work_phase_bead`'s content with one
   concise rule:
   - Before planning, check the phase's notes.
   - If an earlier agent already worked this phase and left it open with recorded
     remaining work, that remainder did not fit one agent. If you plan it, author a
     child epic whose phases each fit one coding agent; its land agent closes this
     phase. Do not author another single-agent tale unless one coder can clearly finish
     everything left.

   Keep every existing sentence that `tests/test_bead_macro_tags.py` pins.

2. Add an assertion for the new prose in `tests/test_bead_macro_tags.py`.
3. Add one sentence to the bead-work section of `docs/beads.md`, near the paragraph on
   phase agents that auto-approve epic-tier plans.
4. Run sase `just check` through `sase tool run check`. No sase-core change is needed.

## Landing (this child epic's land agent)

1. Verify that every phase closed with its evidence.
2. Re-run `sase bead epic-symbols` for this epic, and confirm `sase tool run check` is
   green in sase-core and sase, or that any failure is a documented clean-base
   reproduction.
3. Triage the phases' `PROPOSED FOLLOW-UP:` notes.
4. Close this epic.
5. Then handle `parent_bead` `sase-1h8.13`:
   - Confirm its acceptance is met:
     - every ordinary mutation loads only affected rows on the cached path
     - write-through publishes in the same critical section
     - no-cache stores keep full replay
     - every mutation suite passes in both modes
     - parity asserts cache equals replay after each mutation
     - lock and contention tests are unchanged
     - `store_io_stats` proves no full replay
     - binding-level `append_note`/`update` costs at 1x and 8x are recorded
   - If the child-epic close cascade has not already closed it, close it with
     `sase bead close sase-1h8.13 --note "<summary plus evidence references>"`.
   - Leave `sase-1h8` to its already-waiting land agent. Note on `sase-1h8` that
     DISCOVERED ISSUE #3 was fixed by `publish-direct`.
