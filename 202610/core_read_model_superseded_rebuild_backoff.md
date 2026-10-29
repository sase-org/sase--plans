---
tier: tale
title: Stop superseded bead read-model rebuilds from arming the rebuild backoff
goal:
  sase-core master CI is green again because a read-model rebuild that a concurrent
  mutation superseded no longer disables cache refresh for 10 minutes.
size: small
proposed_by: bbugyi200.athena.15q
proposed_with: claude/opus
create_time: 2026-10-29 01:22:13
status: wip
---

# Plan: Fix sase-core CI by stopping superseded read-model rebuilds from arming the rebuild backoff

## Goal

Get sase-core master CI green again. Fix the product bug behind the intermittent
`bead_read_model_parity::concurrent_readers_see_consistent_snapshots` failure. Do not
weaken any assertion.

All work happens in the linked **sase-core** repo. Open it with
`sase repo open sase-core -r "<reason>"`, use only the path it prints, and read its
`AGENTS.md` first. Its rules apply: never run bare `cargo`, use `just` recipes, run the
gate as `sase tool run check`, use Conventional Commit subjects, and never edit versions
or changelogs.

## Diagnosis (already done; do not redo it)

### What `actstat` shows

`sase-org/sase-core` master `cd62c83d`: CI #2766 failed in
`cargo fmt + clippy + test (ubuntu-latest)`, step `cargo test --workspace`. Only one
test failed:

```
bead_read_model_parity::concurrent_readers_see_consistent_snapshots
panicked at crates/sase_core/tests/integration/bead_read_model_parity.rs:934:5:
cache is stale: the next read rebuilds
```

Line 934 is the test's final `assert!(verify.matched, ...)`. The verify comes after the
reader threads join, `resync`, and `assert_parity`.

### Failure history (last ~80 CI runs)

Two failure signatures appear in the history:

1. **`concurrent_readers_see_consistent_snapshots`, "cache is stale" at :934.** This is
   the target of this plan. It failed 10 times on both ubuntu and macOS:
   - runs 40413292219, 40415098642, 40415835481, 40424599394, 40434321453, 40684718736,
     40684857269, 40694855276, 40797498289, 40807288117
   - the first failure came about an hour after `651d1933` "fix(bead-read-model): repair
     corrupt caches in place" (2026-10-26 21:25 UTC)
   - the latest two master failures (`0fd85e4a`, `cd62c83d`) are this signature only
   - bead sase-1pi.1 note #2 recorded it as an unowned PROPOSED FOLLOW-UP
2. **macOS
   `tool_run::store::connection::tests::wal_mode_probe_consumes_conversion_budget`.**
   This one is **out of scope**. It was fixed by `0fd85e4a` "fix(tool-run): enforce
   SQLite wall-clock busy deadlines", and macOS has been green on every run since. Bead
   sase-1pi.1 owns it. Do not touch `tool_run`.

### Root cause

`crates/sase_core/src/bead/read_model/store.rs` keeps per-process state
(`CACHE_PROCESS_STATE.failed_rebuilds`). In `rebuild_from_replay`, any
`Err(Fault::CacheDiagnostic(_))` returned by `rebuild_from_replay_inner` calls
`mark_failed_rebuild(cache_path)`. That arms `FAILED_REBUILD_BACKOFF`, which lasts 10
minutes. While it is active, every `rebuild`, `ensure_fresh*`, and `rebuild_from_replay`
on that cache path returns `rebuild_backoff_fault` and only replays. The cache is never
refreshed again during that time.

`651d1933` changed the post-commit guard in `rebuild_from_replay_inner`. The branch
where the sweep after a successful COMMIT no longer matches the committed stream
signatures used to be `drop_cache_file` + `Err(Fault::Cache)`. It is now
`invalidate_cache_and_report(cache_path, "post-rebuild stream sweep changed")`, which
returns `Fault::CacheDiagnostic`. That change pushed this branch into the backoff-arming
arm.

This branch is not a failed rebuild. It is the race the module docs call expected: "any
concurrent change invalidates the file in place and serves a replay instead." The
rebuild committed fine, and a concurrent mutation then moved the store.

The test hits this race on purpose:

1. Four reader threads call `read_store_issues` while the main thread runs
   `append_issue_note` ten times.
2. A tail commit loses its own post-commit guard (`tail.rs` `guard_committed_streams`,
   "post-commit stream sweep changed"). That guard does not arm the backoff, but its
   in-place invalidation deletes `streams_known`.
3. The next read's tail therefore falls back ("stream signatures not recorded") to
   `rebuild_from_replay`.
4. Another append lands between that rebuild's COMMIT and its post-commit sweep.
5. The backoff is armed, and every later read in the process only replays.
6. After the join, `assert_parity` returns correct data (served by replay), but the
   cache is left stale. `read_model_verify_cache` then reports `Probe::Stale`.

The bug also matters in production. Any long-lived process that holds the core can hit
it: the gateway, the macro LSP, or a sase Python process using `sase_core_rs`. One lost
race while agents are mutating beads turns off the read-model cache for 10 minutes, so
every bead read pays a full replay.

### Reproduction evidence

The repro used a throwaway out-of-repo harness that path-depends on `sase_core` and runs
the test's exact scenario. Nothing in the repo was modified.

- **Unmodified core, 48 parallel processes for load:**
  - 6 of 2,880 iterations failed, then 3 of 1,920 in a second run.
  - Every failure had verify `"cache is stale: the next read rebuilds"`, the exact CI
    message.
  - `read_model_status` reported
    `"read-model rebuild is backed off after a previous cache failure for 600 seconds; replay serves this read"`.
- **Instrumented `git archive` copy (eprintln at both `mark_failed_rebuild` sites):**
  - Every arming was `Fault::CacheDiagnostic(post-rebuild stream sweep changed)`, with
    rebuild reason `tail fallback: stream signatures not recorded`.
  - No `Fault::Cache` arming was seen.

## Implementation

### 1. Classify a superseded rebuild explicitly (store.rs)

In `rebuild_from_replay_inner`, the rebuild committed but the store moved before the
post-commit sweep. That is the `load_stream_signatures(&connection) != current.streams`
branch.

- Make this branch report "superseded" to `rebuild_from_replay`, separately from a
  genuine failure.
- Keep calling
  `invalidate_cache_and_report(cache_path, "post-rebuild stream sweep changed")` so the
  file is still invalidated in place and doctor still sees the diagnostic.

Suggested shape: a private enum in `store.rs` that `rebuild_from_replay_inner` returns.
Any equivalent explicit classification is acceptable.

```rust
/// Why a rebuild attempt did not serve its own rows.
enum RebuildMiss {
    /// The rebuild could not complete; arms the failed-rebuild backoff.
    Failed(Fault),
    /// The rebuild committed, but a concurrent mutation moved the store
    /// before the post-commit sweep. The file was invalidated in place and
    /// the next read refreshes normally, so this never arms the backoff.
    Superseded(Fault),
}

impl From<Fault> for RebuildMiss { /* -> Failed */ }
```

Then in `rebuild_from_replay`:

- `Superseded(fault)` returns `Err(fault)` without `mark_failed_rebuild`.
- `Failed(fault)` goes through the existing match unchanged. That match is the
  `another_writer_advanced_cache` exemption for `Fault::Cache`, plus arming for
  `Fault::Cache` and `Fault::CacheDiagnostic`.

Constraints:

- Keep `rebuild_from_replay`'s signature (`Result<CachedStoreSnapshot, Fault>`). Its
  callers (`refresh.rs`, `publish.rs`, `rebuild_cold`, `rebuild_cold_discard`) and
  `tests_version.rs` depend on it.
- Do **not** string-match the diagnostic text.
- Do not add or remove any `sweep_store_signatures` call. Tests count full sweeps
  through `store_io_stats` (for example `mutation/tests/publish_direct.rs`).
- Leave every other arming site as it is. Specifically:
  - "post-rebuild stream sweep unreadable" stays a failure. A store that persistently
    cannot be stat'ed would otherwise pay replay plus a cache write on every read.
  - Write faults (`Other`, `Corrupt`, `Recreated`) and open or busy faults keep arming.
    `failed_rebuild_backs_off_for_the_process_window` must keep passing.
- Don't change the tail path. Its guard already returns its fault without arming.

### 2. Test-only interleaving hook (store.rs `rebuild_write_stats`)

Add a one-shot, thread-local hook to the existing `#[cfg(test)] rebuild_write_stats`
module:

- It is a `RefCell<Option<Box<dyn FnOnce()>>>`, for example set with
  `set_after_rebuild_commit_hook(impl FnOnce() + 'static)` and taken with
  `take_after_rebuild_commit_hook()`.
- `reset()` also clears it.
- Run it in `rebuild_from_replay_inner` behind `#[cfg(test)]`, right after the
  successful commit's `drop(connection);` and before the post-commit
  `sweep_store_signatures`.
- Being thread-local keeps parallel tests from interfering with each other.

### 3. Deterministic regression test (read_model/tests.rs)

Add a test named something like
`rebuild_superseded_by_a_concurrent_mutation_does_not_back_off`.

Setup:

1. `test_git_store_path` + `seed_event_store` + `rebuild_read_model_at` to warm the
   cache. The git layout makes `read_store_issues` discover the cache.
2. Record `expected = read_event_store_issues(&beads_dir)`.
3. Set the hook. It should rewrite one existing `events/streams/*.jsonl` file with
   **identical bytes** through tmp + `fs::rename`, the same way the parity test's
   `rewrite_stream_file` does. The new inode changes the stream signature but not the
   content. Pick the file from `read_dir` rather than hard-coding a stream name.
   - Do not call a real mutation such as `append_issue_note` from the hook. Its
     write-through can re-sync the cache's signatures and hide the race.

Assertions:

- `rebuild_read_model_at(&beads_dir, &cache_path)` returns `Ok(None)`: the superseded
  rebuild falls back.
- `read_store_issues(&beads_dir)` equals `expected`.
- After that read, `read_model_status_at(...).fresh` is true (assert with the status
  reason as the message).
- `read_model_verify_cache_at(...)` is `compared && matched`.

Write and run the test **before** the step 1 fix and confirm it fails. Before the fix,
the status reason is the "backed off ... 600 seconds" message. After the fix it must
pass.

### 4. Keep the docs truthful (store.rs)

- Add one bullet to the module-level "Concurrency rules":
  - the failed-rebuild backoff arms only when a rebuild cannot complete
  - a committed rebuild that a concurrent mutation superseded is invalidated in place
    and leaves the next read free to refresh
- Update the doc comment on `FAILED_REBUILD_BACKOFF` to match.
- No other docs, memory notes, wire types, Python bindings, or sase pin changes are
  involved. This is a pure core fix with no binding surface.

## Verification

1. `just fast`
2. `just test -p sase_core read_model::`. The new test should be red before step 1 and
   green after.
   - `failed_rebuild_backs_off_for_the_process_window` and every `tests_version.rs`
     rebuild test must stay green.
3. `just test -p sase_core --test integration bead_read_model_parity`
4. `just fmt`, then the full gate with `sase tool run check`.
   - It takes about 5–10 minutes. Give it an explicit tool timeout of at least 15
     minutes, or hand it to `/sase_monitor`.
   - If an unrelated test fails, look up its `sase-core flake:` bead before treating it
     as yours.

## Commit

Use a Conventional Commit subject in sase-core, for example
`fix(bead-read-model): don't back off rebuilds a concurrent mutation superseded`. The
body should name the `concurrent_readers_see_consistent_snapshots` CI flake and the
`651d1933` regression. Not a breaking change.
