---
tier: epic
status: done
title: Reclaim disk space on / and /mnt/hercules
goal: "Both pressured filesystems return to a healthy steady state with durable
  headroom: `/` drops below 75% used and `/mnt/hercules` below 80% used, the backup
  system stops storing redundant full copies of regenerable build artifacts, and
  monitoring alerts before either filesystem fills again.

  "
phases:
  - id: triage
    title: Emergency triage on /mnt/hercules
    depends_on: []
    size: small
    description:
      "triage: reclaim immediate headroom on the 100%-full RAID array by clearing
      interrupted-rotation leftovers and the oldest redundant backup rotations, without
      touching Plex media."
  - id: rootreclaim
    title: Ephemeral cache reclamation on /
    depends_on: []
    size: medium
    description:
      "rootreclaim: purge regenerable caches and stale temp data on the root filesystem
      - Docker, APT archives, journald, /var/tmp, the uv cache, disabled snap revisions,
      and root's own caches."
  - id: plexthumbs
    title: Plex video preview thumbnails
    depends_on: []
    size: small
    description:
      "plexthumbs: remove the 73.8G of .bif preview thumbnails under
      /var/lib/plexmediaserver and turn off the generation setting that recreates them."
  - id: buildart
    title: Rust and agent-workspace build artifacts
    depends_on: []
    size: medium
    description:
      "buildart: prune the 125G sase-core target directory and the per-workspace Rust
      target copies under ~/.local/state/sase/workspaces, then redirect future builds to
      a single shared, size-capped location."
  - id: bkexclude
    title: Backup exclusion list for regenerable data
    depends_on:
      - triage
    size: medium
    description:
      "bkexclude: extend the rsync exclude list in backup.sh so build artifacts, agent
      workspaces, and toolchain caches stop being copied into every backup rotation."
  - id: bklinkdest
    title: Hardlinked backup rotations via --link-dest
    depends_on:
      - bkexclude
    size: medium
    description:
      "bklinkdest: make each rotation an incremental hardlink snapshot instead of a full
      independent copy, which is the single largest structural win on the array."
  - id: guardrails
    title: Monitoring, alerting, and regression guards
    depends_on:
      - rootreclaim
      - buildart
      - bkexclude
      - bklinkdest
    size: small
    description:
      "guardrails: add Prometheus disk alerts, recurring cleanup jobs, and a documented
      steady-state baseline so neither filesystem silently refills."
proposed_by: bbugyi200.athena.0dy
create_time: 2026-09-09 20:00:01
---

- **PROMPT:**
  [prompts/202608/disk_space_reclamation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/disk_space_reclamation.md)

# Plan: Reclaim disk space on / and /mnt/hercules

## Audit summary

Measured on 2026-08-26 with `df`, `du -x`, `docker system df`, and `stat`.

### Filesystem status

| Filesystem      | Device                          | Size | Used | Avail | Use%     |
| --------------- | ------------------------------- | ---- | ---- | ----- | -------- |
| `/`             | `/dev/nvme1n1p2` (ext4)         | 875G | 811G | 55G   | **94%**  |
| `/mnt/hercules` | `/dev/md0` (ext4, 3-disk RAID)  | 11T  | 11T  | **0** | **100%** |
| `/mnt/poseidon` | `/dev/sdd1` (ext4)              | 229G | 28K  | 217G  | 1%       |
| `/mnt/hades`    | `/dev/nvme0n1p1` (ntfs, **ro**) | 954G | 683G | 271G  | 72%      |

`/mnt/hercules` reached 0 bytes available during this audit. A
`/bin/rm -rf /mnt/hercules/backup/home/.hourly-4` launched by the 07:56 hourly backup
was still running in `D` state at the end of the audit; it will return space as it
completes. That oscillation — write a full new copy, then delete the oldest — is the
array's normal cycle today, and it means the array routinely runs with no margin.

`/mnt/poseidon` is a mounted, formatted, effectively empty 229G SSD. It is a viable
relocation target. `/mnt/hades` is a read-only NTFS Windows partition and is **not**
usable for Linux storage without remounting it read-write, which this plan does not
propose.

### Where `/` is going (818G)

- **`/home` 561G**
  - `/home/bryan/projects` 184G
    - **`projects/github/sase-org/sase-core/target` 125G** — the single largest object
      on the filesystem. `uv-tool-lsp` 41G (35G of it `incremental/`), `uv-tool-py` 38G
      (34G `incremental/`), `debug` 28G (16G `deps`, 11G `incremental`), `release` 20G.
      The tracked source next to it is 26M.
    - Other Rust targets: `bobs-org/bob-cli` 4.4G, `zettel-org/zorg` 1.7G,
      `bbugyi200/actstat` 1.3G.
  - `/home/bryan/.local` 126G
    - **`.local/state/sase/workspaces` 108G** — `sase-org` 81G across ~20 numbered
      workspaces, `bobs-org` 25G. Dominated by per-workspace
      `sase/repos/linked/sase-core/target` copies: 9.7G (`sase_14`), 5.3G (`sase_17`),
      then ~780M–1.1G each across roughly 18 more.
  - `/home/bryan/Sync` 81G — `Sync/var/projects/work` 43G, `Sync/home/tmp` 33G
    (including a 15G `dot_sase_20260706` snapshot).
  - `/home/bryan/.cache` 42G — **uv 38G** (`archive-v0` 36G), ms-playwright 1.9G, pip
    1.2G.
  - `.stack` 17G, `.sase` 15G, `.npm` 14G, `.codex` 13G (12G `sessions`), `.config` 12G,
    `var` 11G, `go` 11G, `.rustup` 7.6G, `.pyenv` 6.1G, `.grok` 4.4G, `.claude` 2.4G.
- **`/var` 211G**
  - **`/var/lib/plexmediaserver` 84G** — 76G is `Media/localhost`, of which **73.8G is
    8,211 `.bif` video preview thumbnail files**.
  - **`/var/lib/docker` 54G** — `docker system df` reports 22.17G reclaimable images,
    **12.58G in 2,062 local volumes with zero attached to running containers**, and
    16.13G of fully reclaimable build cache. ~50G reclaimable.
  - `/var/lib/prometheus` 33G — `ARGS=""` in `/etc/default/prometheus`, so retention is
    the package default rather than an explicit policy.
  - **`/var/tmp` 18G** — 4,056 entries older than 7 days, some dating to 2020 (`sort*`
    scratch files, ~25 `sase-*` scratch directories, a 3.4G
    `systemd-private-*-locate.service-*` leftover).
  - `/var/cache/apt` 15G — 4,963 `.deb` files.
  - `/var/log` 4.4G — 3.5G of journald.
- `/usr` 29G, `/root` 11G (6.8G `.cache`, 2.7G `.npm`), `/opt` 4.4G, `/nix` 3.5G.

### Where `/mnt/hercules` is going (11T)

- `/mnt/hercules/plex` **5.8T** — TV 3.8T, Movies 2.0T. Real irreplaceable media; this
  plan does not delete any of it.
- `/mnt/hercules/backup` **~5T** — see root cause below.
- `archive` 3.0G, `motion` 2.7M, `misc` 80K — negligible.

> Accuracy note: the `~5T` backup figure is derived by subtraction (11T total − 5.8T
> plex − 3G archive). The recursive `du` over `/mnt/hercules/backup` was deliberately
> killed mid-run because it was competing for I/O with the live `rm -rf`. The first task
> in `triage` is to obtain real per-rotation numbers.

### Root cause on /mnt/hercules

`/etc/backup.sh` symlinks to `/home/bryan/Sync/bin/cron.jobs/backup.sh`, driven by
`/etc/cron.{hourly,daily,weekly,monthly}/*_backup`. Two independent defects compound:

**1. Rotations are full copies, not incremental snapshots.** `_run_rsync()` has **no
`--link-dest`** (verified: `grep -c 'link-dest'` returns 0). Verified empirically — the
same 17-byte `.bashrc` across seven rotations:

```
home/hourly/bryan/.bashrc     links=1  inode=7552237
home/hourly-2/bryan/.bashrc   links=1  inode=357177224
home/daily/bryan/.bashrc      links=1  inode=315895939
home/daily-2/bryan/.bashrc    links=1  inode=67910252
home/weekly/bryan/.bashrc     links=1  inode=4348075
home/monthly/bryan/.bashrc    links=1  inode=182336028
home/yearly/bryan/.bashrc     links=1  inode=62066449
```

Distinct inodes, link count 1. Every one of the ~20 `/home` rotations (hourly×4,
daily×4, weekly×4, monthly×4, yearly×4) is a **complete independent copy**, as are 4
rotations each of `/bin`, `/boot`, `/etc`, `/lib32`, `/lib64`, `/opt`, `/sbin`, `/usr`,
and `/var`.

**2. The exclude list misses the largest regenerable data.** The current list covers
`.cache`, `.venv`, `.git`, `.tox`, `__pycache__`, `venv`, `lost+found`, and
`bryan/{.ansible,.cargo,.gems,.npm,.pyenv,.rustup,.virtualenvs,go}` plus
`projects/work/`. It does **not** exclude:

- `**/target/` — Rust build output
- `**/node_modules/`
- `bryan/.local/state/sase/workspaces/` — the 108G of agent workspaces
- `bryan/.stack/`, `bryan/.codex/sessions/`, `bryan/.grok/`, `bryan/.gradle/`,
  `bryan/Android/`

Verified:
`/mnt/hercules/backup/home/hourly/bryan/projects/github/sase-org/sase-core/target` is
**120G**, and a per-rotation probe found that directory present in **11 of the 20**
`/home` rotations (all four `hourly`, all four `daily`, and three of four `weekly`;
absent from `monthly` and `yearly`). The `target/` directories even contain a
`CACHEDIR.TAG` marker, but rsync has no native support for honoring it.

That single directory therefore accounts for on the order of **1–1.5T** of the array by
itself, and it is 100% regenerable by `cargo build`.

### Reclamation estimates

| Target                            | Phase         | Estimate  | Confidence                           |
| --------------------------------- | ------------- | --------- | ------------------------------------ |
| Backup rotations → hardlinks      | `bklinkdest`  | 3–4T      | Medium — needs `triage` measurements |
| Backup excludes (rust/workspaces) | `bkexclude`   | 1–1.5T    | High                                 |
| `sase-core/target` prune          | `buildart`    | ~120G     | High                                 |
| Workspace `target/` copies        | `buildart`    | ~90G      | High                                 |
| Plex `.bif` thumbnails            | `plexthumbs`  | 73.8G     | High (measured exactly)              |
| Docker prune                      | `rootreclaim` | ~50G      | High (`docker system df`)            |
| uv cache prune                    | `rootreclaim` | up to 38G | High                                 |
| `/var/tmp` sweep                  | `rootreclaim` | ~18G      | High                                 |
| APT archives                      | `rootreclaim` | 14G       | High                                 |
| journald vacuum                   | `rootreclaim` | ~3G       | High                                 |

Root filesystem total: **~300–400G**, taking `/` from 94% to roughly 55–60%. Hercules
total: **~4–5T**, taking the array from 100% to roughly 55–65%.

## Cross-cutting safety rules

These apply to every phase and are not optional.

1. **Never delete anything under `/mnt/hercules/plex`.** It is irreplaceable media and
   is out of scope for all phases.
2. **Propose every destructive command through the `/sase_gate` skill** for explicit
   user confirmation before running it. Batch related commands into one gate rather than
   prompting per file.
3. **Measure before and after.** Record `df -h / /mnt/hercules` at the start and end of
   each phase and report actual reclaimed bytes, not estimates.
4. **Prefer built-in prune subcommands** (`docker system prune`, `journalctl --vacuum`,
   `uv cache prune`, `cargo clean`) over hand-rolled `rm -rf`. They understand what is
   safe to remove.
5. **Never run a recursive delete while a backup cron job is active.** Check with
   `pgrep -af 'rsync|backup.sh'` first; the hourly job runs on the hour.
6. **Do not disable or bypass the backup system** while modifying it. At every point
   there must be at least one complete, recent `/home` rotation on disk.
7. **Avoid heavy `du` scans on `/mnt/hercules` while a backup or `rm -rf` is running.**
   They cause I/O starvation. This audit had to abort a scan for exactly this reason.

## Emergency triage on /mnt/hercules

The array is at 0 bytes available, so this phase runs first and is deliberately
conservative — it only removes things that are unambiguously garbage.

1. Confirm the in-flight `rm -rf /mnt/hercules/backup/home/.hourly-4` from the 07:56
   rotation has finished (`ps aux | grep 'rm -rf /mnt/hercules'`). Let it complete
   rather than killing it; it is already returning space. Re-check `df`.
2. Sweep for other interrupted-rotation leftovers — the script's temp destinations are
   dot-prefixed siblings (`.hourly`, `.daily`, `.hourly-4`, `*.tmp`). One (`.hourly-4`)
   was found during the audit. Any dot-prefixed directory under
   `/mnt/hercules/backup/*/` with no live `rm`/`rsync` touching it is safe to delete:

   ```bash
   sudo find /mnt/hercules/backup -maxdepth 2 -type d -name '.*' -not -name lost+found
   ```

3. Now that I/O is quiet, get the real per-rotation numbers that the audit could not
   finish. Run this with `ionice -c3` and let it take as long as it takes:

   ```bash
   sudo ionice -c3 du -x -h -d1 /mnt/hercules/backup/home /mnt/hercules/backup/var \
     /mnt/hercules/backup/usr /mnt/hercules/backup/etc 2>/dev/null | sort -hr
   ```

   Record the output in the phase report — `bklinkdest` needs it to verify its savings.

4. Reclaim the deepest redundant rotations. The `yearly-2`/`yearly-3`/`yearly-4`
   rotations (Jan 2025, Jan 2024, Jan 2023) are full copies of a home directory from one
   to three years ago. Confirm with the user which retention depth they actually want,
   then drop the surplus. Deleting `yearly-3` and `yearly-4` alone should free several
   hundred gigabytes.
5. Do **not** attempt the exclusion or hardlink work here. Those are `bkexclude` and
   `bklinkdest`, and both need headroom to run safely.

Exit criterion: `/mnt/hercules` has at least 500G available and real per-rotation sizes
are recorded.

## Ephemeral cache reclamation on /

Everything in this phase is regenerable. It is the fastest, lowest-risk path to headroom
on `/`.

1. **Docker (~50G).** Take the pieces in increasing order of risk:

   ```bash
   docker builder prune -af          # 16.13G, zero risk
   docker image prune -af            # 22.17G of unreferenced images
   ```

   The 2,062 local volumes holding 12.58G report zero attached containers, but **verify
   before pruning**: `actual_server` and `actual_caddy` are running, and losing an
   Actual Budget data volume would be real data loss. Run
   `docker inspect actual_server actual_caddy | grep -A5 Mounts` first to confirm they
   use bind mounts rather than named volumes. Only then run `docker volume prune -af`.
   The `bbugyi/neovim` image set alone has ~18 tags spanning 17 months and is the bulk
   of the reclaimable image space.

2. **APT archives (14G).** `sudo apt clean` — 4,963 cached `.deb` files, all
   re-downloadable.
3. **journald (~3G).** Currently unbounded; `/etc/systemd/journald.conf` has an empty
   `[Journal]` section. Vacuum and then cap it:

   ```bash
   sudo journalctl --vacuum-size=500M
   ```

   Then set `SystemMaxUse=500M` in `/etc/systemd/journald.conf` so it stays capped.

4. **`/var/tmp` (~18G).** 4,056 entries older than 7 days, some from 2020. Purge by age,
   not wholesale, and skip anything a running process holds open. The 3.4G
   `systemd-private-*-locate.service-*` leftover is safe to remove. Then add a
   `systemd-tmpfiles` rule so `/var/tmp` self-cleans on a 30-day age.
5. **uv cache (up to 38G).** Prefer `uv cache prune` (removes unused entries) over
   `uv cache clean` (nukes everything and forces re-download of every wheel). The 36G
   `archive-v0` directory is unpacked wheel storage.
6. **Disabled snap revisions.** Nine disabled revisions are held (`core`, `core18`,
   `core20`, `core26`, `flutter`, `gnome-3-28-1804`, `heroku`, `obsidian`, `snapd`).
   Remove them and set `snap set system refresh.retain=2`.
7. **`/root` (11G).** 6.8G `.cache` and 2.7G `.npm`, both regenerable.
8. **Prometheus (33G).** `ARGS=""` means no explicit retention policy. Decide a
   retention window with the user (30d is a reasonable default for a home server) and
   set `--storage.tsdb.retention.time=30d` in `/etc/default/prometheus`. Do not delete
   TSDB blocks by hand.

## Plex video preview thumbnails

An exactly-measured 73.8G across 8,211 `.bif` files under
`/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Media/localhost`.
These are scrubber preview images — a pure convenience feature that Plex regenerates on
demand.

Order matters here: **change the setting first, then delete**, or Plex will simply
rebuild them.

1. In Plex → Settings → Library, turn off **"Generate video preview thumbnails"** (set
   it to `never` for every library). This can also be done per-library in the library's
   advanced settings.
2. Stop the Plex service so it does not rewrite entries mid-delete:
   `sudo systemctl stop plexmediaserver`.
3. Delete only `.bif` files, leaving the surrounding bundle structure and the 6.6G
   `Metadata` tree (posters, artwork, agent data) intact — that tree is expensive to
   rebuild and is not the problem:

   ```bash
   sudo find "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Media" \
     -name '*.bif' -delete
   ```

4. Restart Plex and confirm playback plus library browsing still work.

Note that this directory is on `/`, not on the array — the media itself lives at
`/mnt/hercules/plex` and is untouched.

## Rust and agent-workspace build artifacts

This is the largest reclaimable category on `/` — roughly 210G across two related
sources, all of it regenerable by rebuilding.

1. **`sase-core/target` (125G).** Do not `rm -rf target` blindly; use `cargo clean` from
   the crate root so Cargo removes exactly what it owns. If a full clean is undesirable
   because it forces a long rebuild, the `incremental/` subdirectories are the cheapest
   large win — 69G across `uv-tool-lsp/dev-update/incremental` (35G) and
   `uv-tool-py/dev-update/incremental` (34G). Incremental caches are per-machine scratch
   and are never needed for a correct build.
2. **Per-workspace target copies (~90G).** Every numbered agent workspace under
   `~/.local/state/sase/workspaces/sase-org/sase/sase_NN/sase/repos/linked/sase-core/`
   carries its own `target/`: 9.7G in `sase_14`, 5.3G in `sase_17`, and ~780M–1.1G each
   across roughly 18 more. Check whether `sase workspace` offers a GC or prune
   subcommand and prefer it over manual deletion; workspaces may hold uncommitted agent
   work. Only reclaim workspaces with no live agent and no uncommitted changes.
3. **Prevent recurrence.** The structural fix is to stop having N copies of the same
   build output:
   - Set a shared `CARGO_TARGET_DIR` (for example `/mnt/poseidon/cargo-target`) so all
     checkouts share one target directory on the empty 229G SSD instead of each carrying
     its own on `/`.
   - Consider `sccache` for cross-checkout artifact reuse.
   - Add a periodic `cargo-sweep`-style job that removes target files not touched in 30
     days.
   - Confirm the `CACHEDIR.TAG` files already present in these directories, and make
     sure the new backup excludes from `bkexclude` cover them regardless.
4. **Smaller targets.** `bob-cli` 4.4G, `zorg` 1.7G, `actstat` 1.3G, plus 3.7G of
   `~/.sase/cache/rust-prebuild` — clean if the user does not need warm builds there.

## Backup exclusion list for regenerable data

The highest value-to-risk edit in the whole plan. It is a contained change to one
function and it stops the array from re-accumulating what the other phases just cleaned.

Edit `_run_rsync()` in `/home/bryan/Sync/bin/cron.jobs/backup.sh` to add:

```
--exclude="**/target/"
--exclude="**/node_modules/"
--exclude="bryan/.local/state/sase/workspaces/"
--exclude="bryan/.stack/"
--exclude="bryan/.codex/sessions/"
--exclude="bryan/.grok/"
--exclude="bryan/.gradle/"
--exclude="bryan/Android/"
```

Points to get right:

- **`**/target/`is broad.** Confirm with the user that no tracked, non-regenerable directory named`target`
  exists anywhere under the backed-up trees. If that is a concern, scope it to the known
  Rust crate roots instead of using a global glob.
- **`--delete-excluded` is already set**, which is what makes this work: newly excluded
  paths are actively removed from each rotation as it is rewritten, rather than
  lingering. Space is therefore reclaimed progressively — the full benefit lands only
  after every rotation tier has cycled (up to a year for `yearly`). Accelerate it by
  deleting the already-known-stale `target/` copies out of existing rotations directly.
- **Discuss `.codex/sessions`, `.grok`, and the workspaces exclusion with the user
  first.** These contain agent conversation history, which may be worth keeping even
  though it is large. This is a retention judgment, not a purely technical one.
- **`Sync/` deserves a conversation too.** At 81G it is already replicated by Syncthing,
  so backing it up to the same machine may be redundant — but confirm the user's intent
  rather than assuming.
- Validate the changed script with `bash -n` and do a `--dry-run` rsync with the new
  filter set before letting cron use it.

Expected: 1–1.5T from the `target/` exclusion alone, plus the workspace directories.

## Hardlinked backup rotations via --link-dest

The structural fix, and the largest single win available anywhere on this system. Today
each rotation is an independent full copy; it should be a hardlink snapshot that only
stores changed files.

Design:

1. Pass `--link-dest` to rsync pointing at the current newest rotation. Because
   `_backup()` rsyncs into `_to` (`.hourly`) and only afterward performs the `mv` chain,
   the correct reference at rsync time is the still-in-place `${to}` (`hourly`):

   ```
   --link-dest="${to}"
   ```

   Use an absolute path, since a relative `--link-dest` is interpreted relative to the
   destination directory. Unchanged files become hardlinks to the previous rotation and
   consume no additional space; changed files are written fresh.

2. rsync accepts up to 20 `--link-dest` directories. Cross-tier deduplication (letting
   `daily` link against `hourly`, `weekly` against `daily`, and so on) multiplies the
   savings, since those tiers hold near-identical content. Add them in most-recent-first
   order.
3. **This is safe with rsync specifically** because rsync replaces files rather than
   modifying them in place, so writing to one rotation cannot corrupt a hardlinked peer.
   Any _other_ tool that edits files inside `/mnt/hercules/backup` in place would now
   silently corrupt every rotation sharing that inode. Document this constraint in the
   script.
4. A welcome side effect: `rm -rf` of an aged-out rotation becomes dramatically cheaper,
   because it decrements link counts instead of freeing millions of unique blocks. That
   directly addresses the long-running `rm` that had the array pinned at 0 bytes during
   this audit — and should keep the atomic block inside `MAX_ATOMIC_TIME`.
5. **Preserve the existing atomicity guarantees.** The script is careful about the
   `mv`-chain atomic block, the `ETBB` too-soon guard, and the `MAX_ATOMIC_TIME` check.
   Do not regress any of them. Existing non-hardlinked rotations are not retroactively
   deduplicated; savings accrue as each tier cycles.
6. **Test before trusting.** Exercise the new logic against a scratch source tree and a
   scratch destination, then verify with `stat -c %h` that unchanged files across two
   consecutive rotations report a link count greater than 1 — the exact check that
   proved the current defect.

Alternative worth raising with the user: a purpose-built deduplicating backup tool
(`borg` or `restic`) would provide content-level dedup, compression, and encryption, and
would subsume both this phase and `bkexclude`. That is a larger migration than this plan
scopes, but it is the better long-term answer if the user is open to replacing the
homegrown script. `rsnapshot` is a middle option that implements exactly this
hardlink-rotation scheme as a maintained package.

## Monitoring, alerting, and regression guards

Prometheus and node_exporter are already running on this box, so alerting is mostly
configuration rather than new infrastructure.

1. Add Prometheus alert rules on `node_filesystem_avail_bytes` for both `/` and
   `/mnt/hercules` — warn at 80% used, critical at 90%. Neither filesystem should ever
   again reach 100% without warning.
2. Add a `predict_linear` rule on the array to warn when it is trending toward full
   within seven days, which catches the slow refill that produced this situation.
3. Wire the alerts to whatever notification path the user already uses.
4. Schedule the recurring cleanups established in earlier phases: `docker system prune`,
   `apt clean`, `uv cache prune`, and the `cargo`-target sweep. Reuse the existing
   `~/Sync/bin/cron.jobs/` structure so they live alongside the backup jobs.
5. Record a steady-state baseline — post-cleanup `df` for both filesystems, plus the
   per-rotation backup sizes from `triage` — so future growth is measurable against a
   known-good starting point.
6. Verify the backup system still works end to end after all changes: confirm a fresh
   rotation completes, `backup.txt` timestamps advance, hardlink counts are greater than
   1, and a test file restores correctly from a rotation.

## Out of scope

- Deleting, transcoding, or re-encoding anything under `/mnt/hercules/plex` (5.8T).
  Space can be reclaimed there, but it is a content decision for the user, not a
  technical cleanup.
- Remounting `/mnt/hades` read-write. It is a Windows system partition holding
  `pagefile.sys` and `hiberfil.sys`.
- Expanding the RAID array or adding disks.
- Migrating `/home` to `/mnt/poseidon` wholesale. `buildart` uses poseidon for a shared
  Cargo target directory, but a broader migration is a separate decision.
