---
tier: epic
title: Cut athena's SSD writes from SASE's own churn
goal: 'SASE stops rewriting large derived state on athena''s root NVMe. The runner-slot
  capacity cache shrinks from ~104 MB to under ~1 MB. Codex launches stop rebuilding
  a 347 MB thread index (and start ~3 minutes sooner). The corrupt bead read-model
  cache heals itself instead of rebuilding and rolling back on every read. sase-core
  test builds link ~2.9 GB less per cycle. No admission decision, Codex transcript,
  or bead read result changes.

  '
decisions:
  prune_dead_pids:
    ask: Also prune capacity-scan records whose recorded runner pid is provably dead,
      not only stopped ones?
    default: true
    why: Shrinks athena's cache from ~9 MB to under 1 MB; sase's probe already treats
      them as never live
    answer: true
phases:
- id: core-capacity-prune
  title: Prune never-live dirs from the capacity_only scan
  depends_on: []
  size: medium
  description: 'core-capacity-prune: in sase-core''s agent_scan scanner, extend the
    capacity_only skip to drop stopped (and, per the decision, dead-pid) artifact
    dirs. A neutrality post-pass re-admits lineage, owner-key, and clan-tribe neighbours,
    and differential tests prove admission decisions are unchanged.'
- id: core-read-model-heal
  title: Self-heal the corrupt bead read-model cache
  depends_on: []
  size: medium
  description: 'core-read-model-heal: in sase-core''s bead read model, classify SQLITE_CORRUPT
    write faults and repair them by REINDEX or an in-place table recreate. Stop swallowing
    invalidation failures, back off after failed rebuilds, never downgrade a newer
    cache, and add a regression test that reproduces athena''s corrupt meta index.'
- id: core-test-binaries
  title: Consolidate sase-core integration-test binaries
  depends_on:
  - core-capacity-prune
  - core-read-model-heal
  size: medium
  description: 'core-test-binaries: merge sase_core''s 23 and sase_macro_lsp''s 10
    integration-test files into one binary per crate. Isolate tests that mutate process-global
    state, update every --test reference, and record the before/after link bytes.'
- id: sase-capacity-adopt
  title: Adopt the pruned capacity scan in sase
  depends_on:
  - core-capacity-prune
  - core-read-model-heal
  size: small
  description: 'sase-capacity-adopt: ratchet sase-core-revision.txt past both core
    phases, add a Python differential admission test through the real binding, sweep
    stale scan-cache temp files, and update the capacity_only docstring and runner-slot
    docs.'
- id: codex-private-sessions
  title: Stop per-launch Codex thread backfill
  depends_on: []
  size: medium
  description: 'codex-private-sessions: give each shadow CODEX_HOME a private sessions
    tree whose today/tomorrow day dirs link to the real rollout store. Rescue stray
    rollouts at teardown, sweep orphaned shadow homes safely, and update the Codex
    provider docs.'
- id: verify-athena
  title: Measure the write reduction on athena
  depends_on:
  - core-test-binaries
  - sase-capacity-adopt
  - codex-private-sessions
  size: small
  description: 'verify-athena: once the host runs the landed code (propose the reinstall
    through a gate if not), re-measure each source and athena''s daily root-NVMe writes
    against the baseline, record the results, and file follow-ups for any missed target.'
proposed_by: bbugyi200.apollo.ae
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-26 16:07:21
status: wip
bead_id: sase-1oj
---

- **PROMPT:** [prompts/202610/athena_disk_write_diet.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/athena_disk_write_diet.md)
- **BEAD:** [sase-1oj](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1oj/README.md)

# Plan: Cut athena's SSD writes from SASE's own churn

## Evidence (athena, measured 2026-10-24 to 2026-10-26)

Context: the research report `@research:202610/athena_load_swap_thrash_20261024.md`
(read it with `sase artifact read`) plus fresh measurements taken for this plan.

- **Overall.** The root NVMe (`nvme1n1`, a Samsung 970 PRO 1 TB rated for 1,200 TBW)
  wrote 18.6 TB in 11.2 days of uptime. Prometheus daily totals are 1.4–2.5 TB on active
  days and 167 GB on a quiet day. Swap-out is only 10–15% of that. `nvme0n1`
  (`/mnt/hades`) is idle.
- **Method.** Per-path writes come from `strace -f -y`. Per-category totals come from a
  reaped-child-aware `/proc/*/io` sampler: `/proc/<pid>/io` absorbs the lifetime IO of
  reaped children, so naive parent deltas are wrong. Daily totals come from Prometheus.

| #   | Source                                                     | Mechanism                                                                                                                                                                                                                                                                                       | Measured cost                                                                                                                                                                                                                                                         | Phase                                    |
| --- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| 1   | `~/.sase/runner_slots.scan.json`                           | One 104 MB JSON file holds 15,428 `AgentArtifactRecordWire` records: 13,429 stopped, about 1,925 more with dead pids, and only about 34 with live pids. Waiters rewrite it atomically whenever `runner_slots.token` changes or the 2 s `max_age` lapses. Every parked runner also re-parses it. | 9 rewrites in 5 min with 37 runners: 3.1 MB/s, ≈ 270 GB/day                                                                                                                                                                                                           | core-capacity-prune, sase-capacity-adopt |
| 2   | Codex shadow `CODEX_HOME`                                  | `sessions` is symlinked to the shared `~/.codex/sessions` (22 GB, 19.4k rollouts). Each fresh `state_5.sqlite` therefore indexes every rollout before the session starts.                                                                                                                       | Per launch: a 347 MB DB, ~3.1 GB written by the sqlx worker threads, and ~181 s before `session_meta`. About 59 launches/day: ≈ 180 GB/day, plus ~22 GB of rollout reads per launch.                                                                                  | codex-private-sessions                   |
| 3   | Bead read-model cache, primary beads clone (bead sase-1nw) | The `meta` primary-key index is corrupt. Every write that touches `streams_known` fails with SQLITE_CORRUPT. Each reader replays, rebuilds into the WAL, rolls back, and unlinks `-wal`/`-shm`.                                                                                                 | Cache frozen at `crate_version 0.40.0` since Oct 16. The TUI alone issues ~5 MB/s of WAL `pwrite64` and 10 WAL/SHM unlinks per 20 s. Most bytes are cancelled before writeback, but every read pays a full replay, a ~30 MB discarded transaction, and journal churn. | core-read-model-heal                     |
| 4   | sase-core test links                                       | 23 `sase_core` and 10 `sase_macro_lsp` integration-test files each link into their own 120–250 MB binary. All of them relink whenever `sase_core` changes.                                                                                                                                      | ~3.3 GB of integration-test executables per test build. Per-launch `CARGO_TARGET_DIR`s reached 13–14 GB. Rust builds were ~50% of attributed writes in a 5-minute busy window.                                                                                        | core-test-binaries                       |

## Safety invariants (every phase)

- **Only derived state changes.** No phase deletes or rewrites user data, bead events,
  artifact dirs, or Codex rollouts. Rollouts are moved, never deleted. A name collision
  leaves the file where it is.
- **Admission is decision-neutral.** Pruning the capacity scan must not change any
  claim, waiter, deference window, queue weight, lineage owner key, or hold identity.
  Deadlock notices may lose only nodes that were never live; today those are false
  positives.
- **Read-model repairs are atomic.** A repair either commits a consistent snapshot in
  one `BEGIN IMMEDIATE` transaction, or leaves the cache unusable so readers fall back
  to replay. It never serves stale rows and never unlinks a cache file that other
  processes may hold open.
- **Fail toward keeping things.** When liveness, ownership, or a path's origin is
  uncertain, keep the record, file, or directory.
- **Host actions go through gates.** Global reinstalls and any edit to athena's live
  caches go through `/sase_gate` (Global Install Is Human-Only).

## Phase core-capacity-prune

Repo: sase-core. Open it with `sase repo open sase-core` and work in the printed path.
Read sase-core's `AGENTS.md` first. Shared backend logic belongs here, so do not add a
Python copy of the predicate.

**Change.** In `crates/sase_core/src/agent_scan/scanner.rs`, extend the `capacity_only`
skip in both `scan_agent_artifacts` and `scan_agent_artifact_dirs`, next to the existing
`done.json` skip. Drop the dirs that sase's liveness probe can never count as live. That
probe is `is_process_alive` in sase's `src/sase/agent/names/_common.py`: a missing or
non-int pid, a pid ≤ 1, or a truthy `stopped_at` all mean "not live". It is fed by
`_record_liveness_probe` in `src/sase/axe/run_agent_wait_slots.py`, which prefers
`agent_meta.pid` and falls back to `running.json`'s pid.

Prune a non-done dir when it has no `pending_question.json` and either of these holds:

- **Stopped:** `agent_meta.stopped_at` is non-empty, and neither `running.json` nor
  `waiting.json` exists.

> [!decision] prune_dead_pids
>
> - **Dead:** the probe's pid (`agent_meta.pid`, else the `running.json` pid) is an int
>   greater than 1, and `kill(pid, 0)` reports ESRCH. Reuse the helper in
>   `crates/sase_core/src/host_liveness.rs`; EPERM counts as alive. Stale `waiting.json`
>   markers from dead waiters (882 of them on athena) are pruned; the probe already
>   ignores them.

> [!decision] prune_dead_pids = no Prune only stopped dirs. Records with dead pids stay
> in the scan (athena keeps about 2,000 records, ≈ 9 MB).

Always keep these: records with no pid or an unparsable pid; records whose pid is alive,
even if the pid may have been reused; and anything the post-pass re-admits.

**Neutrality post-pass.** After partitioning, re-admit every pruned record that:

1. is a transitive `parent_timestamp` ancestor of a kept record (the lineage index in
   `runner_capacity/records.rs`, the claim reuse in `claims.rs`, and the sibling scan by
   owner key);
2. shares `runner_claim_owner_key` with a kept record; or
3. belongs to the same clan generation as a kept record and declares an explicit
   `clan_tribe` (the hold identity overlay in sase's
   `src/sase/core/agent_hold_identity.py`).

Before coding, re-read every consumer of the scan records and extend the post-pass if
any other cross-record lookup exists:

- `runner_capacity_snapshot` and `capacity_record_from_scan` in
  `src/sase/core/runner_slots/`
- `enrich_candidate_from_records`, `hold_deadlock_armer_record` and
  `_hold_deadlock_armers` in `src/sase/axe/run_agent_wait_slot_candidate.py` and
  `run_agent_wait_slots.py`
- `proc_capacity_admission.py` and `launch_admission_engine.py`
- the Rust `runner_capacity` and `agent_hold_deadlock` modules

**Constraints.**

- Keep the `capacity_only` wire field and the schema unchanged; add no new option.
- Update its doc comment in `agent_scan/wire.rs`.
- Optionally add a pruned-dir counter to the scan stats.
- `capacity_only` has exactly one caller: `_RUNNER_SLOT_SCAN_OPTIONS`.

**Tests (Rust, beside the code).**

- Existing tests stay green, including
  `capacity_only_keeps_active_dir_without_running_or_waiting_marker` and the
  `marker_files_parsed` assertion in
  `capacity_only_skips_done_dirs_before_parsing_markers`.
- Each prune case and each keep case gets a test:
  - stopped and marker-less: pruned
  - stopped with `waiting.json`: kept
  - dead pid (use a pid above any `pid_max`, e.g. 2^30): pruned under the decision
  - live pid (`std::process::id()`): kept
  - no pid: kept
- Each post-pass re-admission rule gets a test.
- Differential test: on a mixed fixture, claims, waiters, lineage owner keys, and
  weights computed by the Rust capacity functions over the pruned records equal those
  computed over the unpruned records.

Verify with `sase tool run check` in sase-core.

## Phase core-read-model-heal

Repo: sase-core, `crates/sase_core/src/bead/read_model/`. This phase fixes bead
sase-1nw. Its filed hypothesis (a lost generation CAS) is wrong. Record the verified
mechanism below as a note on sase-1nw.

**Verified mechanism.** This was checked on a copy of athena's cache; line numbers are
from v0.42.6.

1. `PRAGMA integrity_check` reports
   `wrong # of entries in index sqlite_autoindex_meta_1`: the index holds a
   `streams_known` entry whose rowid does not exist. `quick_check` still says `ok`.
2. Every statement that touches `streams_known` fails with SQLITE_CORRUPT:
   - the DELETE in `invalidate_cache_in_place` (`store.rs` ~2235)
   - the upsert in `write_snapshot_in_txn` (~1629)
   - the UPDATE in `stamp_freshness_if_current` (~1308)
3. `drop_cache_file` discards that failure (`let _ =`, ~2368), so the meta stays at
   0.40.0.
4. `rebuild_cold_discard` then calls `rebuild_from_replay`, which writes about 7.6k
   issues and 10.8k edges into the WAL, fails at the `streams_known` upsert, and rolls
   back.
5. The caller replays a second time.

In-place invalidation replaced unlinking in 5c5bcdc4 (v0.37.1).

**Changes.**

1. Classify `DatabaseCorrupt` and `NotADatabase` errors as a new `WriteFault::Corrupt`,
   instead of `Other`. Do this on every commit path in `store.rs`, `tail.rs` and
   `publish.rs`.
2. On `Corrupt`:
   - roll back, run `BEGIN IMMEDIATE; REINDEX; COMMIT`, and retry once;
   - if the retry still fails, or `integrity_check` is not `ok`, drop and recreate every
     cache table in one IMMEDIATE transaction, carrying `generation`,
     `content_generation` and `invalidated_generation` forward +1 from their
     table-scanned values;
   - never unlink the file.
3. Stop discarding invalidation failures in `drop_cache_file`. Surface them in the
   returned fault and in diagnostics.
4. **Per-process backoff.** After any failed rebuild, serve through replay without
   another rebuild attempt for a bounded window (about 10 minutes). A persistently
   unwritable cache (corrupt, disk full, read-only) then costs one rebuild per window,
   not one per read.
5. **Never downgrade.** When the cache's `crate_version` parses as a semver newer than
   the reader's, return Unusable and replay without writing. This prevents ping-pong
   when a long-running process on an older core shares a clone with a newer core. Equal,
   older, or unparsable versions keep today's rebuild behaviour.
6. Out of scope: no schema v4 and no `temp_store` change.

**Tests (in the read model's `tests.rs`).**

- **(a) Reproduce athena's corruption:**
  1. Build a cache, then checkpoint it (`wal_checkpoint(TRUNCATE)`, or use
     `journal_mode=DELETE`).
  2. Save the root page of `sqlite_autoindex_meta_1`, found from `sqlite_schema` and
     `page_size`.
  3. Delete `streams_known` and set an old `crate_version`.
  4. Write the saved page back at `(rootpage-1)*page_size`.
  5. Assert that the first read heals the cache (`integrity_check` is `ok`, the version
     is current, exactly one rebuild ran) and that the second read serves without a new
     generation.
  6. Add a variant with a matching version and a stale token.
- **(b)** An older `crate_version` produces one committed rebuild, then serves.
- **(c)** A newer `crate_version` leaves the file's bytes and mtime unchanged and still
  returns correct replay results.
- **(d)** An injected persistent write fault produces exactly one rebuild attempt per
  backoff window.

Verify with `sase tool run check` in sase-core.

## Phase core-test-binaries

Repo: sase-core. This phase runs after both core phases, so it moves whatever tests they
added.

**Change.**

- Merge `crates/sase_core/tests/*.rs` (23 files) into one integration-test binary, e.g.
  `tests/integration/main.rs` with one `mod` per former file.
- Do the same for `crates/sase_macro_lsp/tests/*.rs` (10 files).
- Fixture paths built from `CARGO_MANIFEST_DIR` keep working.
- Keep `tests/fixtures/` where it is.

**Isolation audit.** These files mutate process-global state (`set_var`, `remove_var`,
`set_current_dir`, or global statics):

- `artifact_ref_commit_budget.rs`
- `hermetic_test_env.rs`
- `sase_macro_lsp`'s `jsonrpc_stdio.rs`, `jsonrpc_stdio_model_shortcut.rs` and
  `jsonrpc_stdio_model_alias_shortcut.rs`

For each, either confine the mutation (to a child process or an explicitly passed env),
or keep that file as its own binary. The `hermetic_test_env` canary must still guard
every test process. Keep it standalone if merging would weaken it.

**References.** Update every reference to an individual test target (`--test <name>`).
Grep both repos: `scripts/check.sh`, the justfile, `.github/workflows`, docs, and sase's
`tools/`.

**Evidence to record in the phase notes.**

- Test count before and after, as reported by `just test`.
- Total bytes of integration-test executables in `target/debug/deps` (expected: ~3.3 GB
  → ~0.5 GB).
- `just test` wall-clock.

Verify with `sase tool run check` in sase-core. Never weaken an assertion to get a green
run.

## Phase sase-capacity-adopt

Repo: sase.

1. Move `sase-core-revision.txt` past both core phases' commits with
   `just ratchet-core-revision` (see `docs/rust_backend.md`).
2. Update the `capacity_only` docstring in `src/sase/core/agent_scan_wire_records.py`
   (lines ~99–105).
3. Add a Python differential test through the real binding.
   - **Fixture tree** with these dirs:
     - done
     - stopped
     - stopped with `waiting.json`
     - dead-pid waiter (pid 2\*\*30)
     - live (`os.getpid()`)
     - no pid
     - lineage child of a stopped parent
     - clan member declaring `clan_tribe`
   - **Assert** that `runner_capacity_snapshot` and the candidate decision for several
     candidates are identical whether built from the `capacity_only` scan or from a full
     scan filtered to non-done dirs.
   - **Assert** that the pruned scan omits the stopped and dead dirs.
4. In `src/sase/core/runner_slots/_scan_cache.py` `_write_scan_cache`, best-effort
   unlink sibling `.runner_slots.scan.json.*.tmp` files older than 1 hour before
   writing. These are leftovers from killed writers; athena has ~130 MB of them.
5. Add a short paragraph on the shared capacity cache and its pruning to
   `docs/reference/runner-slots.md`. Read `sase/memory/docs_writing.md` first.

Verify per `sase/memory/lint_and_test.md` (`sase tool run check`).

## Phase codex-private-sessions

Repo: sase, `src/sase/llm_provider/_codex_home.py`.

**Background.**

- sase-11s and sase-1ko.8 stopped sharing Codex SQLite state but kept `sessions`
  symlinked. Every fresh `state_5.sqlite` therefore backfills all shared rollouts.
- SASE never resumes Codex threads. It reads rollouts only after the fact, through
  `find_codex_sessions` in `src/sase/instructions/run_index.py`. That function scans
  `real_codex_home()/sessions/YYYY/MM/DD` by local date.
- `CodexProvider._run_subprocess` is the only place that creates a shadow home.

**Changes.**

1. **Private sessions tree.**
   - Remove `sessions` from `CODEX_SHADOW_HOME_SYMLINK_ENTRIES`.
   - Create a real `sessions/` directory in the shadow home, with real `YYYY/MM`
     subdirectories.
   - Inside them, make `DD` symlinks to the real `<real home>/sessions/YYYY/MM/DD` for
     today and tomorrow in local time (create the real targets first). Handle month and
     year boundaries.
   - Codex then writes rollouts straight into the real store, crash-safe and visible
     while the run is live. Its backfill sees at most two days of rollouts.
2. **Teardown rescue.** In `codex_subprocess_env`, before `rmtree`:
   - Walk the private tree without following symlinks.
   - Move each regular file to the same relative path under the real sessions root
     (`mkdir -p` the parents, never clobber).
   - On a destination collision, log it and keep the shadow home instead of deleting it.
   - Across devices, fall back to copy, fsync, then unlink.
3. **Orphan sweep.** This is best-effort, bounded, and swallows its own errors. When
   creating a shadow home, sweep `~/.cache/sase/codex_home/` dirs that meet all three
   conditions:
   - the `<pid>-` prefix process is gone;
   - the mtime is older than 1 hour;
   - `is_sase_managed_codex_home` holds.

   For each one, run the rescue, then `rmtree`.
   - Never follow symlinks. Legacy homes symlink `sessions/`, and pre-Oct-4 homes
     symlink `state_5.sqlite*` into `~/.codex`; only the links themselves may be
     removed.
   - Live-pid dirs are left alone.
   - Concurrent sweepers must tolerate files that vanish under them.
   - Athena holds 87 such dirs (1.1 GB).

4. **Escape hatches.**
   - Keep `SASE_CODEX_DISABLE_SHADOW_HOME=1` working.
   - Read `sase/memory/sase_flags.md`. Gate the new layout behind a flag only if that
     note requires the old branch to stay reachable.
5. **Docs.** Update the shadow-home paragraph in `docs/reference/providers/codex.md`.
   Read `sase/memory/docs_writing.md` first.

**Tests.** Use temp HOME and real-home roots.

- **Layout:** there is no `sessions` symlink, and the day links resolve to real dirs,
  including across month and year rollover.
- **Rescue:** a straggler rollout is moved, and a collision keeps the shadow home.
- **Orphan sweep:**
  - a dead, old home is swept and its rollout rescued;
  - a legacy symlinked home is removed with its targets intact;
  - live-pid, young, and non-managed dirs are untouched.
- **Discovery:** `find_codex_sessions` finds a rollout written through a day link.

**Live probe** (only if `codex` is installed on the host).

1. Enter `codex_subprocess_env()` from the workspace code.
2. Run one `codex exec --json` with a trivial prompt.
3. Confirm that:
   - `state_5.sqlite` stays under ~5 MB;
   - `session_meta` appears within ~10 s of process start;
   - the rollout lands under the real day dir;
   - the shadow home is removed.
4. Record the numbers.

Verify per `sase/memory/lint_and_test.md`.

## Phase verify-athena

Run on athena, directly or over `ssh athena`.

**If the host is behind.** If its installed sase or sase-core lacks the epic's commits
(dev installs fast-forward the linked sase-core to origin), propose the reinstall
through `/sase_gate`. Agents never run global installs. Measure whatever does not need
the reinstall.

**Record on the epic bead** a comparison against the Evidence table:

- **Scan cache:** size, rewrites per 5 min, and parked-runner RSS and CPU.
- **Codex:** a fresh shadow home's `state_5.sqlite` size and its time to `session_meta`.
- **Read model:**
  - its meta (current `crate_version`, rising `outcome_serve`);
  - no `-wal` create/unlink churn in 20 s of `strace -f -y` on the TUI.
- **Daily root-disk writes:** daily `nvme1n1` writes from
  `increase(node_disk_written_bytes_total{device="nvme1n1"}[24h])` against the 1.4–2.5
  TB/day baseline.
- **Per-category attribution:**
  - sample `/proc/*/io` (write minus cancelled) at 1 Hz for 5 minutes;
  - when a pid disappears, subtract its last-seen total from its parent and credit its
    in-window part to its own category.

File follow-ups through `/sase_new_task` for any missed target.

## Not in this epic: host-level follow-ups

- **Immediate stopgap** until the read-model fix is deployed: one `REINDEX` inside
  `BEGIN IMMEDIATE`, run against athena's
  `~/projects/github/sase-org/sase/sase/repos/beads/.git/sase/bead-read-model/bead-read-model-*.sqlite`.
  It is safe while readers are live (verified on a copy), but it needs a gate.
- **Swap:** the 64 GB swap partition shares the root NVMe. Use zram, or a swapfile on
  `/mnt/hades`.
- **Build scratch:** pointing `SASE_TMPDIR` (cargo-targets, agent-tmp) at `/mnt/hades`
  would move most of the remaining Rust-build writes off the root disk.
- **SSD wear:** check it with `sudo smartctl -a /dev/nvme1n1`.
- **Considered and rejected for now:** warm per-workspace `CARGO_TARGET_DIR`s. Most
  target bytes are the workspace's own crates and test links, not dependencies, and disk
  use would grow by about 13 GB per workspace.
- **Minor:** SQLite temp-file churn (`/var/tmp/etilqs_*`) from axe routines is mostly
  cancelled before writeback (~0.2 MB/s net).
