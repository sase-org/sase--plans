---
tier: epic
title: Reclaim apollo root disk and stop SASE build scratch from refilling it
goal: 'apollo''s root filesystem regains roughly 100G of free space without disturbing
  running agents, and per-launch cargo targets plus per-checkout pytest scratch are
  reclaimed automatically so the disk does not refill.

  '
phases:
- id: apollo-emergency-reclaim
  title: Emergency reclaim on apollo
  depends_on: []
  size: small
  description: 'apollo-emergency-reclaim: over ssh with no repo changes, delete finished
    agents'' cargo targets, stale zorg target dirs, trashed pre-cutover SASE trees,
    stale pytest scratch, and legacy cargo strays, with liveness guards and before/after
    df accounting.'
- id: agent-scratch-exit-cleanup
  title: Runner-exit scratch cleanup and low-free-space pressure reaping
  depends_on: []
  size: medium
  description: 'agent-scratch-exit-cleanup: remove the runner''s launch-assigned cargo-targets
    and agent-tmp directories at exit, and use a 1h pressure min age whenever the
    free-space floor is breached regardless of which pressure trigger fired, with
    tests.'
- id: pytest-scratch-sibling-reap
  title: Reap sibling pytest scratch roots
  depends_on: []
  size: small
  description: 'pytest-scratch-sibling-reap: make tools/run_pytest also reap stale
    runs in other /var/tmp/sase-<sha8> roots owned by the user and remove empty stale
    roots, with tests.'
proposed_by: bbugyi200.kellys_mbp.0l
create_time: 2026-09-14 06:56:18
status: wip
bead_id: sase-10r
---

- **PROMPT:** [prompts/202609/apollo_disk_reclaim_1.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/apollo_disk_reclaim_1.md)
- **BEAD:** [sase-10r](https://github.com/sase-org/sase--beads/blob/main/pages/sase-10r/README.md)

# Reclaim apollo's root disk and stop SASE build scratch from refilling it

## Problem

apollo's `/` (`/dev/vda1`, 193G) is 100% full. Re-measured 2026-09-14 03:02 UTC: only
**444M** available (down from 783M at first measurement ~01:45 UTC, and from ~2.1G an
hour before that). Agents on apollo fail cargo/rustc builds with "No space left on
device". The `bryan` user has no passwordless sudo, so every step must work as `bryan`.
Inodes are fine (18%).

`sase disk reap` (dry run) reports 0 B reclaimable. The housekeeping `managed_tmp_reap`
chop keeps running with `pressure_trigger=size` and `removed=0` while the disk fills
(see root cause 1 for why).

### Where the space is (measured with `du -x`, re-verified 2026-09-14 03:0x UTC)

| Path on apollo                                       | Size | What it is                                                                                                         |
| ---------------------------------------------------- | ---- | ------------------------------------------------------------------------------------------------------------------ |
| `~/.local/share/Trash/files/sase-org`                | 30G  | Old numbered `sase_N` / `sase-core_N` clones, trashed 2026-07-03 from `~/projects/github/sase-org`                 |
| `~/.local/share/Trash/files/sase_1`                  | 19G  | Old `~/.local/state/sase` workspace store, trashed 2026-08-31 during the cutover                                   |
| `~/.local/share/Trash/files/_cacache`                | 263M | Trashed npm cache                                                                                                  |
| `~/projects/github/zettel-org/zorg_10{0,1,2}/target` | 28G  | Cargo `target/debug` in checkouts untouched since 2026-05-04; clean git status, not registered in any SASE project |
| `~/projects/github/zettel-org/zorg/target`           | 1.0G | Cargo target in the primary zorg checkout (rebuildable)                                                            |
| `~/.cache/sase/tmp/cargo-targets/*`                  | 27G  | Per-agent-launch `CARGO_TARGET_DIR`s (3–5 GiB each); most belong to agents that already finished                   |
| `~/.local/state/sase/workspaces`                     | 25G  | Live SASE workspace clones (owned by workspace cleanup; out of scope)                                              |
| `~/projects/github/sase-org/sase-core/target`        | 11G  | `just rust-dev-install` targets (out of scope except `dev-update/incremental`)                                     |
| `/var/tmp/sase-<sha8>/pytest-of-bryan`               | ~7G  | `tools/run_pytest` scratch, one root per workspace checkout (14 roots, largest 1.2G)                               |
| `~/.sase/cache/rust-prebuild/sets`                   | 5.5G | Two live prebuild sets (out of scope)                                                                              |
| `~/cutover-backups`, `~/tmp/old_sase`                | 6.8G | User backups (out of scope)                                                                                        |
| `~/tmp/sase/cargo-targets`                           | 1.8G | Leftover cargo targets from the old `SASE_TMPDIR` location                                                         |
| `~/.cache/uv`, `~/.npm/_cacache`                     | 6.1G | Regenerable package caches                                                                                         |

### Root causes inside SASE

1. **Per-launch cargo targets are never removed when the agent ends, and pressure
   reaping cannot help during exactly this kind of emergency.**
   - `_managed_agent_scratch_env` in `src/sase/agent/launch_spawn.py` creates
     `cargo-targets/<scratch_key>` and `agent-tmp/<scratch_key>` under the managed tmp
     root for every launch. The key includes the launch timestamp. It also sets
     `CARGO_INCREMENTAL=0`.
   - Nothing deletes these directories when the runner exits.
   - `src/sase/core/managed_tmp_reaper.py` only ages them out after
     `BUILD_SCRATCH_HORIZON_SECONDS` (3 days).
   - The pressure pass applies `DEFAULT_PRESSURE_MIN_AGE_SECONDS` (12h) uniformly. When
     many agents build on the same day, every leftover directory is younger than 12h, so
     pressure reaping removes nothing while the disk fills.
   - **Trigger-precedence trap:** `_pressure_trigger` returns `"size"` whenever the
     managed root exceeds `DEFAULT_PRESSURE_MAX_BYTES` (16 GiB), _before_ it considers
     the free-space floor. On apollo the cargo-targets bucket alone is 27G, so during
     the emergency the trigger is always `size` — never `free_space` — even at 444M
     available. The chop logs confirm it: `pressure_trigger=size`, `removed=0`. Any fix
     that special-cases only the `free_space` trigger will therefore never activate in
     this emergency. The min-age relaxation must key off the free-space floor breach
     itself (available bytes below `DEFAULT_PRESSURE_MIN_AVAILABLE_BYTES`), whatever the
     trigger.
2. **Pytest scratch is only reaped by the same checkout that created it.**
   - `tools/run_pytest` sets `DEFAULT_PYTEST_TMPDIR` to
     `/var/tmp/sase-<sha256(REPO_ROOT)[:8]>`. `_prepare_pytest_tmpdir` only reaps that
     single root.
   - A workspace that stops running pytest keeps up to ~1.2 GiB there forever. On
     apollo, 10 of 14 roots were untouched since 2026-09-03..06, and one had no matching
     checkout at all.

The Trash and zorg `target/` entries are one-time leftovers, not ongoing SASE behavior.
They are still the fastest safe wins.

## Phases

### Phase 1: Emergency reclaim on apollo (`apollo-emergency-reclaim`)

- size: small
- depends on: none

This is operations work only, run over `ssh apollo`. It makes **no repository changes
and no commits**. Run each command in a shell that has sourced `~/.profile` (for
`SASE_TMPDIR=$HOME/.cache/sase/tmp`) and has `~/.local/bin` on `PATH`.

**Rules for every step:**

- Record `df -h /` before and after, and note the bytes freed per step on this phase's
  bead.
- Delete only the paths listed below. Anything else is out of scope.
- A directory counts as **live** if any of these is true, checked at execution time:
  - a process's `/proc/<pid>/environ` sets `CARGO_TARGET_DIR` or `TMPDIR` to it or below
    it — scan **every** process owned by `bryan`, not just cargo/rustc: the long-lived
    `run_agent_runner.py` processes (9 were alive at planning time) carry these
    variables in their environment even while no build is running, and their scratch
    must survive;
  - a process's `/proc/<pid>/cwd` or an open fd (`/proc/<pid>/fd`) is under it;
  - `find <dir> -xdev -mmin -30 -print -quit` finds anything.

  Skip live directories. Never delete anything a running agent is using.

**Steps, in order (fast relief first):**

1. **Finished agents' cargo targets.** For each child of
   `~/.cache/sase/tmp/cargo-targets/`, `rm -rf` it unless it is live. Apply the same
   rule to children of `~/.cache/sase/tmp/agent-tmp/`. Keep both bucket directories
   themselves. Expect roughly 15–22G back (27G total minus live launches).
2. **Trashed pre-cutover SASE trees.** Permanently delete exactly these Trash items,
   each together with its `~/.local/share/Trash/info/<name>.trashinfo`:
   - `~/.local/share/Trash/files/sase-org`
   - `~/.local/share/Trash/files/sase_1`
   - `~/.local/share/Trash/files/_cacache`

   Before deleting, confirm each `.trashinfo` still matches:
   - `sase-org`: `Path=/home/bryan/projects/github/sase-org`,
     `DeletionDate=2026-07-03T09:50:42`
   - `sase_1`: `Path=/home/bryan/.local/state/sase`, `DeletionDate=2026-08-31T17:37:00`
   - `_cacache`: `Path=/home/bryan/.npm/_cacache`

   If one does not match, skip it. Do **not** touch `.password-store*`,
   `.old_password_store`, or any other Trash entry. Expect ~49G back.

3. **Old zettel-org build output.** `rm -rf` only the `target/` directory inside each of
   `~/projects/github/zettel-org/zorg_100`, `zorg_101`, `zorg_102`, and — if they exist
   and have one — `zorg_103`, `zorg_0`, `zorg`. Never delete the checkouts themselves.
   First confirm again that no process cwd is under them. At planning time only
   `zorg_100` (13G), `zorg_101` (11G), `zorg_102` (4.2G), and `zorg` (1.0G) had target
   dirs. `zorg` is the primary checkout; its target is still safe to delete
   (rebuild-only cost), but note on the bead that it was included. Expect ~29G back.
4. **Stale pytest scratch.**
   - Under each `/var/tmp/sase-<8 hex>/pytest-of-bryan/`, remove `pytest-<N>` and
     `garbage-*` run directories whose own mtime and `.lock` mtime (if present) are both
     older than 2 hours, unless live.
   - A `/var/tmp/sase-<8 hex>` root may be removed entirely if two things hold. First,
     its hash matches none of the current checkout paths:
     `sha256(os.fsencode(path))[:8]` for each directory under
     `~/.local/state/sase/workspaces/*/*/*_*` (trailing slash stripped), for
     `~/projects/github/sase-org/sase`, and for `/tmp/release-core-floor`. Second,
     nothing in it was modified in the last 12 hours. `sase-5d01d5fa` was the orphan at
     planning time.
   - Expect ~4–6G back.
5. **Legacy cargo strays.** `rm -rf ~/tmp/sase/cargo-targets` unless live. Expect ~1.8G
   back.
6. **Only if `df` still shows less than 60G available:** run `uv cache prune` and then
   `npm cache clean --force`.

**Explicitly out of scope:**

- `~/.local/state/sase/workspaces/**`
- `~/.sase/**`, including rust-prebuild sets and the artifact stores
- `~/projects/github/sase-org/sase-core/target/uv-tool-{py,lsp}`. Only a
  `dev-update/incremental` subdirectory inside them may be deleted, if present.
- `~/cutover-backups`, `~/tmp/old_sase`, `~/org`, `~/bob`
- system journals (need sudo)

Report any further large candidates you find as `PROPOSED FOLLOW-UP:` notes on this
phase's bead; do not delete them.

**Done when:**

- `df -h /` on apollo shows at least 80G available (roughly 100G is expected).
- The `run_agent_runner.py` processes seen before the cleanup are still alive, apart
  from ones that finished normally.
- `sase disk list` runs without errors.

### Phase 2: Remove per-launch agent scratch at runner exit and make low-free-space pressure effective (`agent-scratch-exit-cleanup`)

- size: medium
- depends on: none

Fixes root cause 1 in this repository.

1. **Runner exit cleanup.**
   - Where: `main()` in `src/sase/axe/run_agent_runner.py`, after
     `finalize_runner_shutdown` in the `finally` block. A small helper module is fine;
     keep the runner file short.
   - What: delete the runner's own launch-assigned scratch directories: the
     `CARGO_TARGET_DIR` and `TMPDIR` values in the runner's environment.
   - Only delete a path that resolves to a **direct child** of
     `managed_tmpdir_root() / "cargo-targets"` or `managed_tmpdir_root() / "agent-tmp"`
     respectively.
   - Never delete a bucket itself, a symlink, or a user-supplied override that points
     anywhere else.
   - Swallow `OSError`. Cleanup must never change the runner's exit status or hide the
     original error.
   - It must also run on the user-kill `SystemExit` path, which the `finally` block
     already covers.
   - The runner's in-process retry loop (see
     `tests/test_axe_run_agent_runner_retry_loop.py`) shares the launch env, so all
     retries reuse the same scratch directories; placing the cleanup in `main()`'s
     `finally` block after `finalize_runner_shutdown` runs it after every retry has
     finished. Verify that nothing after `finalize_runner_shutdown` still needs
     `TMPDIR`; if that assumption fails, move the cleanup to the last point where it
     holds.
   - Hard-killed runners (SIGKILL) keep relying on the reaper as a backstop.
2. **Low-free-space pressure age.**
   - In `src/sase/core/managed_tmp_reaper.py`, apply a shorter minimum age — a new
     `DEFAULT_LOW_FREE_SPACE_PRESSURE_MIN_AGE_SECONDS` of 1 hour — **whenever the
     free-space floor is breached** (available bytes below
     `DEFAULT_PRESSURE_MIN_AVAILABLE_BYTES`), regardless of which trigger
     `_pressure_trigger` returned. Do **not** condition it on `trigger == "free_space"`:
     `_pressure_trigger` checks the `size` trigger first, so whenever the managed root
     exceeds 16 GiB (true throughout this emergency) the trigger is `size` even with
     under 1 GiB free, and a `free_space`-only relaxation would never activate. The
     chop's own `pressure_trigger=size removed=0` records during the incident prove this
     path.
   - Keep 12h (`DEFAULT_PRESSURE_MIN_AGE_SECONDS`) when the floor is not breached.
   - Keep the `_has_fresh_descendant` guard. It is what protects directories that are
     actively building.
   - Update `sase_chop_managed_tmp_reap` counters or summary only if they need the new
     value.
3. **Descriptions.** Update any timing text in `src/sase/default_config.yml` (the
   managed tmp reaper and cargo scratch descriptions near the `managed_tmp_reap` chop)
   and in `src/sase/config/sase.schema.json` if either mentions these horizons.
4. **Tests.**
   - Extend `tests/test_managed_tmp_reaper.py`:
     - with available bytes below the floor and the `size` trigger active (root above
       `DEFAULT_PRESSURE_MAX_BYTES`), a 2h-old large cargo target with no fresh
       descendant is reaped — this is the exact emergency configuration;
     - with available bytes below the floor and the `free_space` trigger active, the
       same directory is reaped;
     - with ample free space under `size` pressure, the same directory is kept until
       12h;
     - a fresh descendant still protects a directory in all cases.
   - Add runner-lifecycle tests next to the existing `tests/test_run_agent_runner_*`
     files:
     - scratch directories inside the buckets are removed at exit;
     - paths outside the buckets, the buckets themselves, and symlinks survive;
     - a removal failure does not change the exit code.
5. Run the verification required by the `lint_and_test` reference memory.

### Phase 3: Reap sibling pytest scratch roots (`pytest-scratch-sibling-reap`)

- size: small
- depends on: none

Fixes root cause 2 in this repository.

1. **Reap sibling roots.** In `tools/run_pytest`, when `_configured_pytest_tmpdir()`
   resolves to the default (no `SASE_PYTEST_TMPDIR` override), `_prepare_pytest_tmpdir`
   must also reap the other `sase-<8 lowercase hex>` directories directly under
   `/var/tmp`.
   - Only reap directories owned by the current uid that are not symlinks.
   - Use the existing `_reap_stale_pytest_runs` rules: 12h horizon
     (`PYTEST_TMP_REAP_HORIZON_SECONDS`), `.lock` mtime respected, `OSError` swallowed.
   - After reaping, `rmdir` a sibling's empty `pytest-of-<user>` directory, and the
     sibling root itself if empty, only when older than the horizon.
   - Never touch the current root's live runs beyond today's behavior, and never touch
     names that do not match the pattern.
2. **Keep it cheap.** Sibling reaping is one directory listing plus stat calls. Make the
   `/var/tmp` parent injectable for tests rather than hardcoding it in the helper.
3. **Tests.** Extend `tests/test_run_pytest_tmpdir.py`:
   - stale sibling runs are removed;
   - fresh sibling runs and locked runs survive;
   - non-matching names and symlinks survive;
   - an override disables sibling reaping;
   - an empty stale sibling root is removed.
4. Run the verification required by the `lint_and_test` reference memory.

## Notes

- Phases 2 and 3 do not wait for phase 1. Phase 1 should start immediately because
  apollo is out of space right now (444M free and falling).
- `sase disk list` run from a non-login ssh shell misses `SASE_TMPDIR`, which is
  exported from `~/.profile`. It then labels the managed cargo targets "unowned strays".
  This explains the earlier agent's "~100GB outside SASE accounting" note. It is not a
  code bug, so no phase covers it.
