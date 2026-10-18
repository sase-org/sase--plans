---
tier: tale
title: Fix the sase-core CI flakes in the bead read model and tool_run tests
goal: sase-core master CI goes green and stays green; the two recurring flakes are
  fixed at the root.
size: medium
decisions:
  cas_loss:
    ask: Should a stamp or sweep refresh that loses its CAS make that one read replay?
    choices:
      replay: This read replays; never serves rows the probe did not validate
      serve_winner: Serve the winner via open_and_serve like commit_tail; may serve
        just-invalidated rows
    default: replay
    why: The race is rare, a replay is always correct, and the winner may be an invalidation
    answer: replay
proposed_by: bbugyi200.athena.11f
proposed_with: claude/opus
decided_by: auto
status: done
---

# Plan: Fix the sase-core CI flakes: unguarded read-model freshness stamps, and a hex-collision test

## Context

`actstat` shows sase-core `master` red at `07bfb83` (`chore: release v0.42.0`). CI run
#2612 failed in job `cargo fmt + clippy + test (macos-latest)`, step
`cargo test --workspace`:

```
test concurrent_readers_see_consistent_snapshots ... FAILED
panicked at crates/sase_core/tests/bead_read_model_parity.rs:934:5:
cache is stale: the next read rebuilds
```

Earlier red commits (`72bf920`, `99b10bd`) failed on
`project_stats::snapshot::tests::odd_filenames_and_non_utf8_paths_are_preserved`.
`80dc704` has already fixed that (its CI is green), and Release-plz then shipped
v0.42.0. That failure needs no further work.

A scan of every failed CI run since Oct 7 shows two flakes still active on master:

1. **`concurrent_readers_see_consistent_snapshots`**
   (`crates/sase_core/tests/bead_read_model_parity.rs`).
   - Before `5c5bcdc4` (Oct 9, "stop unlinking the live read-model cache under
     concurrent access"), it failed 21 times, mostly with `no such table: meta`. That
     was the unlink bug `5c5bcdc4` fixed.
   - Since `5c5bcdc4`, it has failed only with `cache is stale: the next read rebuilds`:
     Oct 11 (ubuntu), Oct 13 (ubuntu), and Oct 18 (macos). The bug is not OS-specific.
   - There is no `sase-core flake:` bead for it.
2. **`tool_run::store::tests::private_argv_is_not_serialized_on_queries`**
   (`crates/sase_core/src/tool_run/store/tests.rs`). It failed 4 times on both OSes with
   `assertion failed: !encoded.contains("abc")`.

Neither flake reproduced in 192 parallel local runs, so the fix relies on deterministic
tests of the exact interleaving, not on reruns. Work in the linked sase-core repo
(`sase repo open sase-core`) and read its `AGENTS.md` first.

## Root cause 1: freshness stamps re-arm an invalidated cache

The read model (`crates/sase_core/src/bead/read_model/`) serves warm reads through a
token-only fast path. `rebuild` and `ensure_fresh` in `store.rs` serve directly when
`freshness_token == meta.token` and `last_sweep_ns` is under 60 s old. They check
nothing else: not `streams_known`, the stream signatures, or the generation.

The `store.rs` module doc promises that every cache write runs in one `IMMEDIATE`
transaction guarded by a generation compare-and-swap. `5c5bcdc4` relies on that promise:
`drop_cache_file` now invalidates in place (`invalidate_cache_in_place` sets
`token = ""` and `last_sweep_ns = 0`, deletes `streams_known`, and bumps `generation`
and `content_generation`) "so any in-flight writer's compare-and-swap loses". Two
writers have no such guard:

- **Probe::Fresh stamps.** `update_token_and_sweep(&connection, &token, now)` runs in
  three places: `rebuild` (~store.rs:610), `ensure_fresh` (~:827), and
  `ensure_fresh_forced` (~:901). Each is two autocommit `UPDATE`s with no transaction,
  no generation check, and no `streams_known` check. The `meta` and signatures the probe
  validated were read earlier, in separate statements.
- **`commit_sweep_refresh`** (`tail.rs`, the zero-tail-event branch of
  `try_tail_apply`). It runs `BEGIN IMMEDIATE`, but it has no generation or
  `content_generation` CAS and bumps neither. It writes back a `stored` signature map it
  loaded before the transaction, plus a token it computed before its sweep.

By contrast, `commit_tail`, `commit_published`, and `write_snapshot_in_txn` all compare
the generation counters inside their transaction.

Here is the interleaving that produces exactly the observed failure: rows correct,
verify reports stale. It needs no special timestamp granularity:

1. Writer W (main thread, note append k) publishes, so the cache holds token T_k,
   signatures S_k, and rows_k.
2. Reader R1 loaded `stored` = S*(k-1) before W's publish and is in
   `commit_sweep_refresh`. With no CAS, it commits signatures S*(k-1) over W's S_k. Its
   post-commit guard (`guard_committed_streams`) sees live S_k ≠ committed S_(k-1) and
   calls `drop_cache_file`, which invalidates in place. Token is now `""`,
   `streams_known` is gone, and rows_k are intact.
3. Reader R2 probed Fresh between steps 1 and 2: it saw meta T_k and S_k, computed T_k,
   and swept S_k. It now runs `update_token_and_sweep(T_k, now)`, which re-arms the
   token and `last_sweep_ns` on the invalidated cache.
4. If k is the last write, every later read takes the fast path: the token matches and
   the sweep time is fresh. They serve rows_k, so `assert_parity` passes.
   `read_model_verify_cache` runs the full probe, finds `streams_known` missing and
   stale signatures, and reports `cache is stale: the next read rebuilds`. In production
   the same race can keep serving an invalidated cache, possibly with stale rows, until
   the 60 s re-sweep.

Before `5c5bcdc4`, the stale stamp landed in an unlinked inode, which is why this
message appears only after that commit. Any in-place invalidation can be re-armed by
step 3: tail and rebuild guards, publish `Invalidated`, and query faults.

## Root cause 2: random hex IDs contain the secret

`private_argv_is_not_serialized_on_queries` uses `"abc"` as the private token and
asserts `!encoded.contains("abc")` over the serialized run. The run also carries random
IDs: `nonempty_or_generated` → `new_hex_id()`, 32 hex chars from `OsRng`, for the run ID
and event IDs. `"abc"` is valid hex, so each ID contains it with probability ≈ 30/4096 ≈
0.7%. The redaction itself works.

## Changes

### A. Guard every freshness stamp (`read_model/store.rs`)

- Replace the three unguarded Probe::Fresh `update_token_and_sweep` calls with one
  helper. It stamps `token` and `last_sweep_ns` inside one `BEGIN IMMEDIATE`
  transaction, and only if all of these still hold:
  - `generation` and `content_generation` equal the values in the `meta` the probe used
  - `streams_known` is present
  - `token` still equals `meta.token`

  It returns whether the stamp held. Use the same `UPDATE … WHERE key = … AND value = ?`
  rows-affected CAS style as `commit_tail`.

- If the stamp is lost, the caller must not serve from that decision. Per the `cas_loss`
  decision (default `replay`), return `Err(Fault::Cache)` so the read replays and the
  next freshness pass rebuilds. Do this in `rebuild`, `ensure_fresh`, and
  `ensure_fresh_forced` (mutation admission then proceeds as an uncached mutation, which
  already works).
- > [!decision] cas_loss = serve_winner Instead, after a lost stamp, roll back and call
  > `open_and_serve(cache_path)` exactly like `commit_tail`'s `CasLost` branch, falling
  > back to `Err(Fault::Cache)` when that fails.
- Remove `update_token_and_sweep` if nothing else needs it unguarded, or keep it only as
  the in-transaction writer that `commit_tail` and `commit_sweep_refresh` call after
  their own CAS. Keep `note_serve_outcome` as is: it only bumps a counter.

### B. Give `commit_sweep_refresh` the admission CAS (`read_model/tail.rs`)

- Pass the `meta` that `try_tail_apply` already holds into `commit_sweep_refresh`.
  Inside its `IMMEDIATE` transaction, compare `generation` and `content_generation`
  against `meta`, and bump `content_generation`, mirroring `commit_tail`'s tail of the
  transaction. The bump is needed because a sweep refresh rewrites signatures and the
  token. Without it, a concurrent stamp, sweep refresh, or `commit_published` whose
  witness predates it would pass its own CAS.
- On a lost CAS, roll back and follow the `cas_loss` decision: by default return
  `Err(Fault::Cache)`; with `serve_winner`, use the `open_and_serve` branch
  `commit_tail` uses. Never drop the file for a lost race. Keep the post-commit
  `guard_committed_streams` on the success path.

### C. Document the invariant

Update the "Concurrency rules" bullet in the `store.rs` module doc so it names freshness
stamps. Only a compare-and-swap writer may set a token and a fresh `last_sweep_ns`,
because those two fields alone arm the token-only fast path. A stamp that lost its race
must never resurrect a cache that was invalidated in place.

### D. Deterministic regression tests (`read_model/tests.rs`, or a new sibling test module under 1,500 lines)

- **Stamp after in-place invalidation.**
  1. Seed a store and rebuild.
  2. Capture the reader view a Fresh probe would hold: `read_meta`, `freshness_token`,
     and `sweep_store_signatures`.
  3. Call `drop_cache_file`.
  4. Call the new guarded stamp with the captured view, and assert it reports a lost
     race and wrote nothing: token still `""`, `last_sweep_ns` still 0, `streams_known`
     still absent.
  5. Assert the next `ensure_cache_ready_at` rebuilds and `read_model_verify_cache_at`
     reports `matched`.

  Confirm, for example by temporarily calling the old `update_token_and_sweep`, that the
  old code fails this test with exactly `cache is stale: the next read rebuilds`.

- **Sweep refresh losing to a concurrent commit.**
  1. Seed and rebuild.
  2. Capture `meta` and the stored signature map as a reader would.
  3. Land a real commit that moves the counters, for example `append_issue_note` through
     the publish path.
  4. Drive `commit_sweep_refresh` with the stale `meta` and stale `stored`. Widen its
     visibility to `pub(super)` or add a `#[cfg(test)]` seam if needed.
  5. Assert it takes the lost-race path, leaves the winner's signatures and token
     untouched, never invalidates the file, and that `read_model_verify_cache_at` still
     reports `matched`.
- Do not weaken or loosen `concurrent_readers_see_consistent_snapshots` or any other
  existing assertion. If an existing test encoded the unguarded behavior, investigate
  before changing it.

### E. Make the tool_run privacy test collision-proof (`tool_run/store/tests.rs`)

In `private_argv_is_not_serialized_on_queries`, replace the `"abc"` secret in both
`argv` and `private_argv` with a sentinel that cannot occur in hex IDs, digests, or temp
paths, for example `"hunter2-private-token"`. Assert that the encoded run does not
contain it. Keep the `has_private_argv` and `"***"` assertions. No other test in the
crate uses the same negative-substring pattern on a hex-collidable literal.

## Verification

- `just test -p sase_core read_model` and
  `just test -p sase_core --test bead_read_model_parity` while iterating, plus the
  tool_run test by name.
- Smoke-stress the parity test: run the built `bead_read_model_parity-*` test binary
  with `concurrent_readers_see_consistent_snapshots --exact` about 300 times, 48 in
  parallel. This is only a smoke check; it did not reproduce the race before the fix.
  The deterministic tests in D are the proof.
- `sase tool run check` in sase-core (guarded; about 5 minutes, so give it a ≥10-minute
  tool timeout). It must pass before finishing.
- Use Conventional Commit subjects, for example
  `fix(bead-read-model): guard freshness stamps and sweep refreshes with the generation CAS`
  and `test(tool-run): use a non-hex private argv sentinel`. Do not touch versions,
  changelogs, or `release-plz.toml`. No Python binding or wire change is involved, so
  sase's `sase-core-revision.txt` pin does not need to move.

## Out of scope

- `write_snapshot_in_txn` compares only `generation`, so a rebuild that read the store
  before a publish can still commit over the publish. Its post-commit sweep already
  invalidates that state. Once stamps are guarded, nothing can re-arm the invalidated
  cache, so the next read rebuilds.
- On filesystems with coarse timestamps, a temp-file create and the rename can share one
  directory-mtime tick. The token could then reach its post-write value before the
  rename is visible. `write_file_atomic`'s `sync_all` between create and rename makes
  that window unlikely, no CI failure matches it, and fixing it would require a
  racy-token rule in `freshness.rs`.
