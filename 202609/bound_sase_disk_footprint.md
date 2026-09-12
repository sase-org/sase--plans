---
tier: epic
title: Bound SASE's disk footprint on a long-running host
goal: 'Every class of disk SASE creates — Rust build output, managed scratch, proc
  runtime state, agent artifact directories, and managed workspace clones — has an
  owner that reclaims it on a bounded horizon, the host notices disk pressure before
  it runs out, and the ~340 GiB already leaked on athena is reclaimed.

  '
phases:
- id: triage
  title: Reclaim the measured backlog under one gate
  depends_on: []
  size: small
  description: 'triage: reclaim the already-leaked bytes through one approval gate,
    then re-measure and record the result.'
- id: tmpreap
  title: Close the managed-temp reaper's coverage gaps
  depends_on: []
  size: small
  description: 'tmpreap: prune children of unregistered managed-temp buckets, register
    the missing buckets, and add a test that the horizon table covers every bucket
    the code creates.'
- id: procreap
  title: Reap proc runtime directories with proc-row retention
  depends_on: []
  size: small
  description: 'procreap: delete a pruned proc''s runtime directory alongside its
    logs and sweep the runtime directories whose proc rows are already gone.'
- id: cargo
  title: Stop the Rust dev-build target leak at its source
  depends_on: []
  size: medium
  description: 'cargo: make the dev-update profile non-incremental, keep every dev-install
    entry point on a managed or repo-owned target root, and document the rule that
    agents never invent a CARGO_TARGET_DIR.'
- id: artifacts
  title: Bound per-project agent artifact directories
  depends_on: []
  size: medium
  description: 'artifacts: give ace-run month shards a retention horizon that protects
    referenced and recent runs, and drop the empty out-of-range shards that starve
    shard watches.'
- id: wsobjects
  title: Share Git objects across managed workspace checkouts
  depends_on: []
  size: large
  description: 'wsobjects: stop every managed checkout from carrying its own full
    copy of the primary''s pack, and retrofit the existing checkouts.'
- id: pressure
  title: Make the footprint visible and self-correcting
  depends_on:
  - triage
  - tmpreap
  - procreap
  - cargo
  - artifacts
  - wsobjects
  size: medium
  description: 'pressure: add the sase disk command group over every owner the earlier
    phases created, make the disk doctor check proportional to volume size, and act
    on pressure from the housekeeping lane.'
proposed_by: bbugyi200.athena.0ka
create_time: 2026-09-12 13:26:38
status: wip
bead_id: sase-zw
---

- **BEAD:** [sase-zw](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zw/README.md)

# Plan: Bound SASE's disk footprint on a long-running host

## Problem

athena's root filesystem is at 98% — 848 GiB used of 875 GiB, 18 GiB free — and it is
still falling while this plan is written. Roughly 340 GiB of that is SASE-attributable,
and essentially none of it has an owner that would ever reclaim it.

The failure is not one bug. It is five independent classes of SASE-created bytes that
nothing is responsible for deleting, plus a disk check whose thresholds are too small to
fire on a volume this size.

### Measured inventory (2026-09-12)

Sizes are `du -xs` on athena. Re-measure before deleting; several are still growing.

**Class 1 — Cargo target directories outside any owner (~155 GiB).** Agents working
Rust-touching epics set `CARGO_TARGET_DIR` to a path they invented, then abandoned it.
The names carry the epic slug they were created for. Nothing in the sase repo,
`sase-core`, the installed `sase` distribution, `~/.config`, or any shell rc references
any of them.

| Path                                                                 | Size     |
| -------------------------------------------------------------------- | -------- |
| `~/.cache/sase-core-target-yj4-check`                                | 22 GiB   |
| `~/.cache/sase-core-cargo-target-sase13`                             | 19 GiB   |
| `~/.cache/sase21-cargo-target`                                       | 19 GiB   |
| `~/.cache/cargo-target-sase-yj`                                      | 18 GiB   |
| `~/.cache/sase-core-target-sase-yf-3-1`                              | 16 GiB   |
| `~/.cache/sase-core-zl2-target`                                      | 15 GiB   |
| `~/.cache/sase-core-target-yj4`                                      | 6.0 GiB  |
| `~/.cache/sase-core-target-yj4-lsp`                                  | 3.6 GiB  |
| `~/.cache/sase-research-core-target-yj4`                             | 739 MiB  |
| `~/.cache/sase-core-target-yj4-install`                              | 739 MiB  |
| `~/Sync/home/tmp/sase-targets`                                       | 20 GiB   |
| a `*cargo-target*` tree nested inside one managed checkout's `.git/` | 9.6 GiB  |
| `/var/tmp/sase-core-target-sase20` (+ two `-release` siblings)       | ~7.3 GiB |

The `~/Sync` entry is inside a Syncthing `sendreceive` folder with an empty `.stignore`,
so 20 GiB of Cargo output is replicated to every machine on the tailnet.

**Class 2 — Incremental compilation cache in the target root the Justfile owns (67
GiB).** `just install` → `rust-dev-install` builds `sase_core_rs` and `sase-xprompt-lsp`
into `<sase-core>/target/uv-tool-py` and `<sase-core>/target/uv-tool-lsp` with the
`dev-update` profile, and `sase-core`'s `Cargo.toml` declares
`[profile.dev-update] incremental = true`. Cargo garbage-collects sessions _within_ a
crate-hash directory but never _across_ them, so every changed build input orphans a
fresh ~1.3 GiB directory permanently:

| Path                                                    | Size   | Session dirs |
| ------------------------------------------------------- | ------ | ------------ |
| `<sase-core>/target/uv-tool-py/dev-update/incremental`  | 33 GiB | 92           |
| `<sase-core>/target/uv-tool-lsp/dev-update/incremental` | 34 GiB | 123          |

Measured accumulation is ~4 GiB/day, and the oldest directories date to 2026-08-26 — the
whole 67 GiB is 17 days of pure leak. `<sase-core>/target` totals 76 GiB, so
`incremental/` is 88% of it.

**Class 3 — Proc runtime directories (3.7 GiB, and the clearest invariant break).**
`~/.sase/procs/procs.jsonl` holds 101 rows after retention. `~/.sase/procs/runtime/`
holds **3,933** directories. `procs/store.py` prunes rows and calls `delete_proc_logs()`
for the pruned ids, but nothing deletes `proc_runtime_dir(proc_id)`, so 3,832 runtime
directories outlive the rows that named them with no way to reach them.

**Class 4 — Agent artifact directories (7.4 GiB in one project).** Under
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/`: `202607` is 2.8 GiB, `202608`
is 3.6 GiB, `202609` is 971 MiB. `sase artifact prune` governs _indexed file rows_, not
these per-run directories, so no month shard is ever reclaimed. Separately, 381 of the
386 month shards are out-of-range dates (`200003` through `205703`) holding 1–3 empty
entries each; they cost no bytes but are the shard-starvation input that
`doctor/checks_resources_ace_run_watches.py` already reports on.

**Class 5 — Duplicated Git objects across managed workspace checkouts (~50 GiB).** The
project's managed workspace root holds 112 GiB across 30 claimed checkouts.
`_utils_checkout.py` materializes each one with `git clone <primary-dir> <target>`,
which hardlinks objects at clone time — but the clone then fetches, auto-gc repacks, and
the hardlink is gone. Verified: the primary's 1.8 GiB pack and a managed checkout's 1.8
GiB pack are distinct inodes, each with `links=1`. Every checkout therefore carries its
own full copy of the repository history; 30 copies of ~1.8 GiB is ~54 GiB, of which one
is legitimate.

**Plus two pre-existing conditions the classes above expose:**

- `~/cutover-backups/backups/` holds three full `~/.sase` snapshots from 2026-09-06 (53
  GiB) and `~/Sync/home/tmp/dot_sase_20260706` is a fourth from 2026-07-06 (15 GiB),
  while the live `~/.sase` is healthy. `migration_kit` writes these deliberately outside
  every SASE runtime root and has no retention for them.
- `doctor/checks_resources_disk.py` warns below 3 GiB free and errors below 1 GiB free.
  On an 875 GiB volume those thresholds mean the check reports OK at 98% full — it has
  been OK for the entire slide from 700 GiB to 848 GiB used.

### What already works, and must not be rebuilt

`sase-zn.6` landed `core/managed_tmp_reaper.py` earlier today (commit `2614668f4`). It
is the right model and the later phases extend it rather than replace it:

- `agent/launch_spawn.py::_managed_agent_scratch_env` already exports per-launch
  `TMPDIR`/`TMP`/`TEMP` and `CARGO_TARGET_DIR` under the managed root, so an agent that
  simply uses its environment can no longer create a Class 1 stray.
- `MANAGED_TMPDIR_HORIZONS` gives each bucket its own age horizon, the reaper never
  follows symlinks, it caps removals per pass, it de-indexes reaped agent-artifact
  directories, and it has a size-pressure pass for `agent-tmp` and `cargo-targets`.
- The `housekeeping` lumberjack lane (`default_config.yml`, hourly) already runs it as
  the `managed_tmp_reap` chop. Every new sweeper in this epic belongs on that same lane.
- `dev_update/prebuild_cache.py` already bounds `~/.sase/cache/rust-prebuild/sets` at
  `COMPLETED_SET_RETENTION = 2`. That cache is 5.7 GiB in 2 entries and is **working as
  designed** — leave it alone.

### Root causes, in one line each

1. A target directory an agent names itself has no owner. (Class 1)
2. `incremental = true` on a profile whose target root nothing ever prunes. (Class 2)
3. Row retention and on-disk retention were implemented separately. (Class 3)
4. Artifact _index_ retention was mistaken for artifact _directory_ retention. (Class 4)
5. Local clone object sharing is created at clone time and destroyed by the first
   repack. (Class 5)
6. Free-space thresholds are absolute, so they do not scale with the volume. (Both
   pre-existing conditions)

## Non-goals

- Prometheus retention (`/var/lib/prometheus`, 52 GiB) and `~/.codex/sessions` (16 GiB)
  are not SASE-created. Report them; do not touch them.
- `~/.sase/cache/rust-prebuild` is already bounded. Do not add a second policy over it.
- No phase may weaken `state_write_guard`, the pytest sandbox roots, or the reaper's
  never-follow-symlinks rule to make a sweep reach further.
- Retention horizons are configuration, not hardcoded policy the user cannot move.

---

## `triage`: Reclaim the measured backlog under one gate

Do this phase first and do not wait for the others: the volume has under 18 GiB free and
the machine has already rebooted once during this investigation.

This phase reclaims bytes that were created _before_ the durable fixes exist. It ships
no production code except one cross-repo ignore file. Its deliverable is the reclamation
itself plus a recorded before/after measurement.

**Steps**

1. Re-measure every path in the inventory above with `du -xs`, plus `df -h /`, and
   record the result. `/var/tmp` needs `sudo -n`, which is available non-interactively;
   it held 41 `sase*` entries at last count.
2. Confirm each candidate is still unreferenced before proposing it. For each Class 1
   path: `grep -rn` the sase repo, `~/.config`, and shell rc files for its basename, and
   check that no live process holds it (`sudo -n lsof +D <path>` or `fuser -m <path>`).
   A path that anything still references drops out of the set.
3. Build the deletion set in these groups, each with its own measured size:
   - **A — Class 1 strays.** Every path in the Class 1 table. For the tree nested inside
     a managed checkout's `.git/`, delete only that subdirectory, never its parent
     `.git`; resolve it from the managed workspace root that `sase workspace list`
     prints, and confirm with `sase workspace list` that you are not disturbing a
     checkout that is mid-run.
   - **B — Class 2 incremental caches.** The two
     `<sase-core>/target/uv-tool-{py,lsp}/dev-update/incremental` directories. Deleting
     `incremental/` alone is safe: `deps/` and `.fingerprint/` stay valid, so the next
     `just install` recompiles only the workspace crates, not the dependency graph. Do
     **not** delete `deps/` and do not run a bare `cargo clean`.
   - **C — stale full-state backups.** `~/cutover-backups/backups/` (three 2026-09-06
     snapshots) and `~/Sync/home/tmp/dot_sase_20260706`. These are backups; they need
     their own explicit line in the gate and their own yes/no, separate from A and B.
     Verify the live `~/.sase` is healthy first (`sase doctor`), and read
     `provenance.json` in each snapshot so the gate can say what is being given up.
   - **D — leaked SASE state.** `~/.sase/procs/runtime/` directories with no row in
     `procs.jsonl`; `ace-run/202607` and `ace-run/202608`; the 381 empty out-of-range
     `ace-run` month shards; `$SASE_TMPDIR/build-targets/` (783 MiB, an unregistered
     bucket the current reaper cannot reach into); and `/var/tmp/sase-<8hex>` scratch
     older than two days. Leave anything created today alone — an agent may hold it.
4. Raise **one** gate with `/sase_gate` listing groups A–D with measured sizes, the
   projected `df` after each group, and the one-line justification for each. Artifact
   and backup removal require explicit user authorization; nothing in group C or D is
   deleted without it. Do not delete anything before the gate is answered.
5. On approval, delete the approved groups, then re-run `df -h /` and record the
   before/after delta.
6. Write `~/Sync/.stignore` so Syncthing stops replicating SASE build output and state
   snapshots across the tailnet. At minimum: `home/tmp/`, `**/target/`,
   `**/*cargo-target*`, `**/dot_sase_*`. First run `chezmoi managed | grep -i stignore`
   — if the file is chezmoi-managed, make the change in the chezmoi repo through
   `/sase_repo` and deploy it; if it is not, create it directly and say so in the phase
   note. Deleting the `~/Sync` entries in group A propagates the deletion to every peer,
   so write `.stignore` first.
7. `sase artifact create` the before/after measurement report and record the reclaimed
   total on the phase bead.

**Acceptance**

- `df -h /` shows at least 250 GiB free, or the phase note explains exactly which groups
  the gate declined and what that cost.
- No path outside the approved gate list was removed.
- `sase doctor` is no worse than before the phase.
- The Class 1 `~/Sync` paths are gone _and_ `.stignore` prevents their return.

---

## `tmpreap`: Close the managed-temp reaper's coverage gaps

`reap_managed_tmpdir()` keys its horizons off an exact bucket name. When
`horizons.get(entry.name)` returns `None` for a _directory_, the whole bucket becomes a
single candidate judged by its own `st_mtime` — and a bucket that keeps receiving
children keeps a fresh mtime forever. So an unregistered bucket is never reaped at all,
while a briefly-quiet one could be deleted whole. `$SASE_TMPDIR/build-targets/` is the
live proof: 783 MiB of Cargo output under a bucket name that appears nowhere in the sase
repo, `sase-core`, or the installed distribution, which the reaper has never touched.

**Steps**

1. In `core/managed_tmp_reaper.py`, split unknown top-level entries by type: an unknown
   **directory** has its _children_ pruned at `DEFAULT_HORIZON_SECONDS` and the
   directory itself survives, exactly like a registered bucket; an unknown **file**
   keeps today's behavior. This makes an unregistered bucket safe by default instead of
   invisible, and removes the whole-bucket-deletion edge. Keep the removal budget, the
   symlink rule, and the de-index step unchanged.
2. Register the buckets the code creates but the table omits: `chezmoi-deploy-locks`
   (`main/_init_chezmoi_deploy.py`, command-scratch horizon) and `muse-prompts`
   (`llm_provider/muse.py`, `_PROMPT_FILE_TMPDIR_PART`, handoff horizon — a provider
   re-reads it mid-run). Keep `MANAGED_TMPDIR_HORIZONS` alphabetically grouped as it is
   today, with the comment bands intact.
3. Add `build-targets` to `PRESSURE_REAP_BUCKETS` alongside `agent-tmp` and
   `cargo-targets` if step 1 leaves it unregistered, so multi-gigabyte build scratch is
   still pressure-reapable under whatever name it arrives with. Prefer making the
   _generic_ unknown-directory path pressure-eligible over enumerating names.
4. Add the regression that would have caught this: a test that walks the repo for
   `get_sase_managed_tmpdir("<part>", ...)` call sites, extracts every literal first
   argument, and asserts each one is either in `MANAGED_TMPDIR_HORIZONS` or explicitly
   listed as intentionally defaulted. Point its failure message at the horizon table so
   the next bucket author knows what to add.
5. Update the `managed_tmp_reap` chop description in `src/sase/default_config.yml` and
   the `docs/axe.md` chop table if the behavior statement changes.

**Acceptance**

- A directory named by no horizon entry has its aged children pruned and survives
  itself.
- The coverage test fails when a new `get_sase_managed_tmpdir` part is added without a
  horizon.
- `just check` passes.

---

## `procreap`: Reap proc runtime directories with proc-row retention

101 rows, 3,933 runtime directories, 3.7 GiB. Row retention and on-disk retention need
to be the same operation.

**Steps**

1. Add a `delete_proc_runtime_dirs(proc_ids)` helper next to `delete_proc_logs()` and
   call it from every site that already calls `delete_proc_logs()` in `procs/store.py`
   (the append-retention paths and `prune_procs`). Removal must be `OSError`-tolerant:
   losing a race with a running proc is a no-op, not a failure.
2. Guard it the way the reaper guards itself. Only remove a directory that is a direct
   child of `procs_dir() / "runtime"`, is a real directory rather than a symlink, and
   whose name is a well-formed proc id (`procs/ids.py`). Never remove the `runtime/`
   root.
3. Add a bounded orphan sweep for the directories whose rows are already gone: list
   `runtime/` children, subtract the ids present in the store, and remove the remainder
   that is older than a horizon and not referenced by a live proc. Bound it with a
   per-pass removal budget the way `DEFAULT_MAX_REMOVALS` bounds the temp reaper — 3,832
   directories must converge over a few passes without one pass stalling the lane.
4. Run the sweep from the hourly `housekeeping` lane: either extend the
   `managed_tmp_reap` chop or add a sibling chop with its own `default_config.yml` entry
   and `docs/axe.md` row. Do not put it on an interactive path.
5. Check whether `proc_log_max_bytes()`'s siblings under `~/.sase/procs/logs` have the
   same orphan shape and say so in the phase note; fix it here only if it is the same
   one-line call-site omission.

**Acceptance**

- Pruning a proc row removes its runtime directory in the same operation.
- The orphan sweep converges: a run over a synthetic 4,000-directory root removes only
  rowless, aged, well-formed entries, respects its budget, and leaves live procs intact.
- A directory belonging to a running proc is never removed.
- `just check` passes.

---

## `cargo`: Stop the Rust dev-build target leak at its source

Two changes stop 4 GiB/day, and one documented rule plus the existing per-launch
`CARGO_TARGET_DIR` export keeps Class 1 from coming back.

**Steps**

1. In `sase-core` (open it with `/sase_repo`; `gh:sase-org/sase-core` resolves as an
   external repo), set `incremental = false` in `[profile.dev-update]` in the workspace
   `Cargo.toml`. The profile inherits `release`, so incremental compilation is buying
   little on an optimized build while costing an unbounded 1.3 GiB per changed build
   input. This is the root fix and it covers every entry point, including a bare
   `cargo build --profile dev-update` an agent runs by hand.
2. In this repo's `Justfile`, add `CARGO_INCREMENTAL=0` to the env block of every
   `rust-dev-install*` recipe, beside the existing `CARGO_NET_RETRY` and
   `CARGO_HTTP_MULTIPLEXING` entries, so the guarantee holds even against a `sase-core`
   checkout that predates step 1. `CARGO_INCREMENTAL` takes precedence over the profile
   setting; verify that empirically rather than trusting it — after a full
   `just rust-dev-install`, assert no `incremental/` directory appears under
   `<sase-core>/target/uv-tool-py` or `uv-tool-lsp`.
3. Keep both target roots repo-owned and reportable. Do not move them into
   `$SASE_TMPDIR`: they are deliberately shared across all workspaces, and a 3-day
   managed horizon would turn every `just install` into a cold build. Instead make sure
   `pressure` can see and prune them (`incremental/` is always safe to delete; `deps/`
   never is).
4. Document the rule in `docs/rust_backend.md`: SASE exports a managed
   `CARGO_TARGET_DIR` into every launched agent, and an agent must use it rather than
   invent a path. Name the two repo-owned exceptions (`uv-tool-py`, `uv-tool-lsp`, set
   by the Justfile) and state the consequence — an invented target root has no owner and
   is what produced 155 GiB of orphans.
5. Add the machine backstop for step 4: a test asserting `_managed_agent_scratch_env()`
   exports `CARGO_TARGET_DIR` under the managed root for every launch, so the guarantee
   the docs rely on cannot be silently dropped.

**Acceptance**

- A full `just rust-dev-install` from clean creates no `incremental/` directory under
  either `uv-tool-*` target root.
- `just install` still succeeds and `sase_core_rs` plus `sase-xprompt-lsp` are installed
  and importable; record the wall-clock delta for a no-op and a one-crate-changed
  rebuild, since removing incremental trades some rebuild time for bounded disk.
- `docs/rust_backend.md` states the target-dir rule.
- `just check` passes, and the `sase-core` change is committed as its own repository
  obligation.

---

## `artifacts`: Bound per-project agent artifact directories

`sase artifact prune` protects explicit snapshots and prunes indexed _rows_. Nothing
prunes the `ace-run/<YYYYMM>/<agent>/` directories themselves, which is 7.4 GiB in one
project and grows with every agent run.

This is the phase most able to break something by deleting too much. The ACE Agents tab,
`sase chats`, `sase agent prompts`, wait-check resolution
(`scripts/sase_chop_wait_checks.py`), the incremental index
(`scripts/_chop_incremental_index.py`), and the agent artifact index all read these
directories. Work outward from what must be protected.

**Steps**

1. Enumerate the readers of `artifacts/ace-run/**` and write down, in the phase note,
   what each needs and for how long. Derive the protection rule from that list, not from
   an age guess.
2. Implement month-shard retention with these protections at minimum: the newest N
   months are kept whole (default N ≥ 2, configurable under the existing
   `artifacts.retention` config section); any run directory referenced by a live bead,
   Patch, gate, artifact link, or artifact-index row is kept regardless of age; any run
   belonging to a non-closed bead is kept. Reclaim only whole run directories, and
   de-index them the way `reap_managed_tmpdir` already de-indexes what it removes.
3. Make it dry-run-first, matching `sase artifact prune`: preview unless `--apply`, and
   never purge trash. Agents must not apply it without explicit user authorization — the
   `pressure` phase wires the unattended path, and it must route through a notification
   or gate rather than deleting silently.
4. Separately, drop the 381 empty out-of-range month shards and stop new ones from being
   created. Find out where an out-of-range shard date comes from before deleting them —
   a real clock-source or test-isolation bug is worth more than the empty directories.
   `doctor/checks_resources_ace_run_watches.py` already reports future-dated shards;
   reuse its range logic rather than writing a second definition of "out of range".
5. Wire the reclamation pass onto the hourly `housekeeping` lane with its
   `default_config.yml` and `docs/axe.md` entries.

**Acceptance**

- A referenced, recent, or open-bead run directory is never selected, proven by test.
- Preview is the default; `--apply` is required to move anything.
- Empty out-of-range shards are gone and the cause of new ones is either fixed or
  recorded as a `PROPOSED FOLLOW-UP:` note with evidence.
- `sase chats`, `sase agent prompts`, and the ACE Agents tab still resolve runs that
  retention kept.
- `just check` passes.

---

## `wsobjects`: Share Git objects across managed workspace checkouts

30 claimed checkouts, 112 GiB, and each one carries its own 1.8 GiB copy of the same
history. `git clone <primary-dir> <target>` in
`workspace_provider/_utils_checkout.py::ensure_git_clone_at` hardlinks objects at clone
time, but the clone's first auto-gc repack writes a fresh pack and the sharing is gone —
verified by distinct inodes with `links=1` on both the primary's and a managed
checkout's 1.8 GiB pack.

This is the largest single lever (~50 GiB) and the riskiest phase: it changes how every
managed checkout is materialized, and a bad alternates configuration can make a
workspace unable to resolve its own objects. Plan before implementing, and treat "a
workspace never loses access to an object it needs" as the hard constraint.

**Steps**

1. Decide and record the mechanism, with the alternative you rejected and why. The
   expected answer is a persistent `objects/info/alternates` entry pointing at the
   primary's object store, so sharing survives repacking, rather than clone-time
   hardlinks that do not. SASE already uses both nearby shapes —
   `sdd/_store_clone_ops.py` uses `--reference-if-able ... --dissociate` and
   `dev_update/prebuild_producer.py` uses `--shared` — so match an existing idiom.
   `--dissociate` copies the objects and saves nothing; do not use it on the create
   path.
2. New checkouts: materialize with object sharing in `ensure_git_clone_at`, keeping the
   existing origin-rewrite, fetch, and clone-failure healing behavior intact.
3. Protect the dependency the sharing creates. The primary must not prune an object a
   checkout still needs: set `gc.auto` and any `maintenance` configuration on the shared
   checkouts so a routine repack cannot break borrowing, and make the primary's pruning
   behavior explicit rather than incidental. State what happens if the primary is moved
   or deleted, and make `sase workspace repair` detect a broken alternates entry and
   recover it (re-point, or dissociate into a standalone checkout).
4. Retrofit the existing checkouts. Add a `sase workspace compact` subcommand (sorted
   into the existing `cleanup|list|migrate|path|repair` group, every long option with a
   short alias, dry-run-first per `sase workspace cleanup -n`) that for each unclaimed
   or safely-quiescent checkout writes the alternates entry, runs a local-only repack so
   objects reachable from the alternate are dropped from the checkout's own pack, and
   verifies connectivity with `git fsck --connectivity-only` before reporting success.
   Never touch a checkout with a dirty working tree or an in-flight agent; skip it and
   say so.
5. Add a config field for the sharing decision under the existing workspace config
   section — a user on an exotic filesystem or a non-local primary must be able to turn
   it off — and mirror it into `src/sase/default_config.yml` with its documentation.
   This is a permanent user choice, so it is a config field and not a feature flag.
6. Verify the reclamation on a real checkout before claiming it: record pack size and
   `git fsck` result before and after, and confirm `git log`, `git status`, a branch
   checkout, a fetch, and a `just install` all still work in the compacted checkout.

**Acceptance**

- A newly created managed checkout does not duplicate the primary's pack, and stays
  correct after a fetch and a repack.
- `sase workspace compact -n` previews; applying it reclaims measured bytes and leaves
  every compacted checkout passing `git fsck --connectivity-only`.
- A checkout whose primary object store has gone missing is detected and repaired by
  `sase workspace repair` rather than failing obscurely.
- A claimed, dirty, or in-flight checkout is never modified.
- `just check-full` passes through `/sase_monitor` — this phase touches the workspace
  provider, which is in the broadening set.

---

## `pressure`: Make the footprint visible and self-correcting

Lands last, because it is the single entry point over every owner the earlier phases
created. Two things were missing: nobody could ask "what is SASE using and who owns it",
and the disk check could not fire on a large volume.

**Steps**

1. Add the `sase disk` command group with an exact `list` child, so bare `sase disk`
   delegates to `sase disk list` through `_default_list_subcommands()` in
   `main/parser.py` — do not re-implement the delegation. Subcommands and options sorted
   alphabetically, every public long option with a short alias, no required options,
   `-j/--json` for machine-readable output, and colored output where it helps a size
   table read.
2. `sase disk list` attributes SASE's footprint by owner: the managed temp root broken
   down by bucket and the horizon each one is subject to; `~/.sase` by subtree
   (`procs/runtime`, per-project `artifacts/ace-run`, `cache/rust-prebuild`); managed
   workspace roots per project; and the repo-owned `<sase-core>/target/uv-tool-*` roots
   with their `incremental/` split out. For each row: size, the owner that reclaims it,
   and its horizon — or **`unowned`**, which is the value that matters.
3. Include a stray scan for Class 1 regressions: find Cargo-shaped directories
   (`.rustc_info.json` or `CACHEDIR.TAG` at the root) under `$HOME` that sit outside the
   managed root and outside the repo-owned target roots, and report them as `unowned`.
   Bound the walk by depth and skip `.git` object stores and other repo checkouts, so it
   cannot become the slow command nobody runs.
4. `sase disk reap` delegates to the owners the earlier phases built — the managed temp
   reaper, the proc runtime sweep, artifact-directory retention,
   `sase workspace compact` — and never reimplements a policy. Preview unless `--apply`,
   matching `sase artifact prune`. It does not delete `unowned` strays on its own
   authority: it reports them for a human decision, because "it looks like a target
   directory" is not ownership.
5. Fix `doctor/checks_resources_disk.py` to scale with the volume: keep the absolute
   floors as one input and add a proportional threshold, so 18 GiB free on an 875 GiB
   volume is not `OK`. Add the largest `unowned` owners from step 2 to `next_steps` so
   the check tells the user where the space went, not just that it is gone.
6. Add a housekeeping chop that acts on pressure: when free space crosses the warn
   threshold, notify with the top owners from `sase disk list`, and run the owned
   sweepers early instead of waiting for their full horizons — the temp reaper already
   has exactly this shape in `_reap_pressure_candidates`, so follow it. Unowned strays
   and artifact or backup deletion are notified, never auto-deleted.
7. Give the thresholds and horizons `default_config.yml` entries with documentation, and
   add the chop's `docs/axe.md` row.

**Acceptance**

- `sase disk` and `sase disk list` produce the same output, with the delegation notice
  on the bare form; `-h` output for the group and both children is complete and sorted.
- Every row carries an owner and a horizon, or is explicitly `unowned`.
- On athena, after the earlier phases, `sase disk list` reports no `unowned` row above 1
  GiB.
- The disk check reports WARN or ERROR at 98% full on an 875 GiB volume, and OK on a
  volume with proportionally ample space; both are unit-tested with an injected
  `disk_usage_fn`.
- `sase disk reap` previews by default and reclaims only through existing owners.
- `just check-full` passes through `/sase_monitor`.

---

## Landing notes

- Phases `triage` through `wsobjects` have no dependencies on each other and can run in
  parallel. `pressure` depends on all of them because it is the aggregating surface.
- `triage` is time-critical. If the volume fills before the gate is answered, the safest
  single reclamation is group B — the two `incremental/` directories, 67 GiB, pure
  cache, regenerated by the next `just install`.
- Every phase that adds or changes a config value must update
  `src/sase/default_config.yml`, and every new chop needs both its `default_config.yml`
  entry and its `docs/axe.md` row.
- `just check` is the per-phase gate. `wsobjects` and `pressure` must use
  `just check-full` through `/sase_monitor`; the land agent runs `just check-full` on
  the combined tree.
- Nothing in this epic edits a SASE memory note. A core-memory rule telling agents never
  to set their own `CARGO_TARGET_DIR` would reinforce the `cargo` phase's documented
  rule, but the user did not ask for a memory change, so it is deliberately left to them
  to request.
