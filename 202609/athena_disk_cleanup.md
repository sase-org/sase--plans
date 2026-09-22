---
tier: tale
title: Free athena root-disk space via SASE reaper and rebuildable caches
goal:
  Reclaim roughly 180-190 GB on athena's nearly full root filesystem by running SASE's
  own disk reaper and clearing only rebuildable build and package caches, without
  breaking any running agent, the sase uv-tool, the sase_core_rs binding, or
  sase-xprompt-lsp.
size: small
proposed_by: bbugyi200.athena.0pk.f0
create_time: 2026-09-22 18:21:53
status: wip
---

# Plan: Free disk space on athena with SASE's own reaper and by clearing rebuildable caches

## Goal

The root filesystem on athena (`/dev/nvme1n1p2`, mounted at `/`) is 97% full (834G used,
~33G free). Free roughly 180–190 GB by deleting only data that is either (a) owned by a
SASE cleanup pass that knows what is still in use, or (b) a download/build cache that
its tool recreates on demand. Nothing may break: every running agent, the installed
`sase` uv-tool, the `sase_core_rs` binding, and `sase-xprompt-lsp` must keep working.

This is an operational task on the host. It changes **no git-tracked files** in any
repository, so there is nothing to commit, and there are no lint or test steps.

## Scope

In scope (the previous survey's unconditional "Tier 1" and "Tier 2" recommendations):

| #   | Target                                                                         | Size now                                    | Method                          |
| --- | ------------------------------------------------------------------------------ | ------------------------------------------- | ------------------------------- |
| 1   | SASE-owned disk (`~/.cache/sase/tmp`, workspace compaction, old run artifacts) | ~114 GiB reclaimable (dry run)              | `sase disk reap --apply`        |
| 2   | `~/projects/github/sase-org/sase-core/target`                                  | 35G (`uv-tool-lsp` 19G + `uv-tool-py` 17G)  | `rm -rf` of the build cache dir |
| 3   | `~/.npm/_cacache`                                                              | 16–17G                                      | `npm cache clean --force`       |
| 4   | `~/.cache/uv`                                                                  | 15G total, ~10.2G not hardlinked into venvs | `uv cache clean` (no `--force`) |
| 5   | `~/go/pkg/mod`                                                                 | 9–11G                                       | `go clean -modcache`            |
| 6   | `~/.cache/pip`                                                                 | ~2G                                         | `rm -rf`                        |
| 7   | `~/.gradle/caches`                                                             | ~1.7G                                       | `rm -rf`                        |

Out of scope. Do **not** touch these. Each needs a decision from the user, or is unsafe:

- `~/cutover-backups/backups/*` (55G), `/var/lib/prometheus` retention,
  `~/.codex/sessions`, `~/.stack/programs`, `~/.pyenv/versions`,
  `~/.local/state/sase/migration-backups/20260522-xdg-project-keys`, disabled snap
  revisions, `docker image prune -a`, and the desktop Trash. These are the "Tier 3,
  user's call" items. The user has not made those calls, so leave them alone. If the
  user's plan feedback names any of them, handle only the ones named.
- `~/Sync` (Syncthing: deletes propagate to other devices).
- `~/.local/state/sase/workspaces` (live agent workspaces; SASE's own compaction in step
  1 is the only permitted change there).
- Git checkouts under `~/projects/github/*` (agent workspaces share their git objects).
- `/var/tmp/sase-*` (in use by running agents; the old ones are ~0 bytes).
- `~/.local/share/uv` (uv-managed Pythons and tool venvs, including the `sase` tool).
- Anything that needs `sudo`.

## Facts already verified during planning (re-check the cheap ones before acting)

- **Shell alias traps on this host:** `du` is aliased to `sudo ncdu`, and it hangs or
  fails without a TTY. `cp`/`mv` are aliased to interactive `-i`. `ls` is `exa`. `pip`
  is a shell function. Always use `command du -sh …`, `command rm -rf …`,
  `command ls …`, and `python3 -m pip …`, or run the command via `timeout …`, which
  bypasses aliases.
- **`sase disk reap` owns step 1.** Its dry run reports `managed_tmp_reaper` ≈106 GiB
  (417 entries under `~/.cache/sase/tmp`: cargo-targets, agent-tmp, gh-diffs; 19 of them
  as pressure prunes because the disk is below its free-space floor),
  `workspace_compact` ≈7.8 GiB across three projects, and `artifact_run_retention` ≈45
  MiB. The reaper preserves build trees with fresh descendants, generic agent scratch,
  handoff data, and unknown buckets. It enforces minimum ages and removes at most 2,000
  entries per pass. It is safe to run while agents are running.
- **The sase-core `target/` is a pure build cache.** It contains only `uv-tool-lsp/` and
  `uv-tool-py/`, which are the `CARGO_TARGET_DIR`s of the sase Justfile's
  `rust-dev-install` recipe (`dev-update` profile). The installed binding the `sase`
  uv-tool imports is
  `~/projects/github/sase-org/sase-core/crates/sase_core_py/python/sase_core_rs/sase_core_rs.abi3.so`.
  maturin copies it there: link count 1, outside `target/`. `sase-xprompt-lsp` is copied
  into the venv's `bin/`, and `~/.cargo/bin/sase-xprompt-lsp` is also a separate file.
  Deleting `target/` costs only a full Rust rebuild on the next `just install` /
  `sase dev-update`.
- **The uv cache is safe to clear.** No `UV_LINK_MODE` env var and no uv.toml
  `link-mode`, so uv uses its Linux default (hardlink). A search found no
  `site-packages` symlink pointing into `~/.cache/uv`. Existing venvs keep their
  hardlinked files, which is why only ~10.2G of the 15G is actually freed.
- During planning, no running process had its executable or command line under
  `~/.cache/uv`, `~/go/pkg/mod`, `~/.npm/_*`, `~/.gradle`, or `sase-core/target`, and no
  Gradle daemon was running.

## Steps

Run every command in the foreground with a generous explicit timeout (for example
`timeout: 1800000` on the Bash tool for the reaper). If a command is killed by its
timeout, rerun it with a larger timeout rather than backgrounding it.

### 0. Baseline

1. Record `df -h /` and `command du -sh` of each in-scope target (skip any that no
   longer exist). Keep these numbers for the final report.

### 1. SASE-owned cleanup

1. Run `sase disk reap` (dry run) and confirm it still lists the owners described above
   with roughly the same reclaimable sizes. If it now reports something unexpected, such
   as an owner that wasn't there or a class outside `~/.cache/sase` and the
   workspace/artifact owners, stop and report instead of applying.
2. Run `sase disk reap --apply`. Its output is the record of what was deleted.
3. Run `sase disk reap` (dry run) again. If `managed_tmp_reaper` still reports more than
   ~5 GiB reclaimable, e.g. because the 2,000-entry budget or pressure rules left a
   backlog, run `sase disk reap --apply` **one** more time. Do not loop beyond that.
4. Do not hand-delete anything under `~/.cache/sase/tmp` or the workspaces. Whatever the
   reaper keeps, it keeps on purpose, for example live build dirs.

### 2. sase-core Rust build cache

1. For the audit trail, run
   `sase repo open sase-core -r "Delete the primary sase-core checkout's gitignored target/ build cache to free disk"`.
   The directory being deleted is the **primary** checkout's build cache at
   `~/projects/github/sase-org/sase-core/target`, not the linked-workspace path that
   command prints. Do not edit any tracked file in either location.
2. Confirm that no Rust build is using it right now. `pgrep -af 'cargo|rustc|maturin'`
   should show no process whose command line or `CARGO_TARGET_DIR` (check
   `/proc/<pid>/environ`) refers to `sase-core/target`. If one is running, wait for it
   to finish and check again, or skip this step and report it.
3. Confirm that `target/` still contains only `uv-tool-lsp` and `uv-tool-py`
   (`command ls -A`). If anything else is there, delete only those two subdirectories
   and report the rest.
4. `command rm -rf ~/projects/github/sase-org/sase-core/target`.
5. Verify the binding and LSP still work:
   - `~/.local/share/uv/tools/sase/bin/python -c 'import sase_core_rs; print(sase_core_rs.__file__)'`
   - `sase --help >/dev/null && echo ok`
   - `command -v sase-xprompt-lsp`, and confirm the file it points to still exists.

   Do **not** rebuild (`just install`, `sase dev-update`) as part of this task. The
   rebuild would immediately refill the space, and the user can trigger it on their own
   schedule.

### 3. Package-manager caches

For each item: check that the tool is not mid-operation (`pgrep -af <tool>`), run the
tool's own cleaner where one exists, then re-measure.

1. **npm:** `npm cache clean --force`. Leave `~/.npm/_npx` and the rest of `~/.npm`
   alone. Only `_cacache` is in scope.
2. **uv:** `timeout 900 uv cache clean`. Do **not** pass `--force`: its in-use check is
   what protects concurrent agent `uv run` / `uvx` processes. If it times out waiting
   for in-use checks, run `timeout 900 uv cache prune` instead, and report that the full
   clean was blocked by concurrent uv processes. Never touch `~/.local/share/uv`.
3. **Go:** `go clean -modcache`. It correctly handles the read-only module files. Do not
   `rm -rf` `~/go/pkg/mod`.
4. **pip:** make sure no `pip` process is running, then `command rm -rf ~/.cache/pip`.
   (`python3 -m pip cache purge` is an acceptable alternative, but it leaves some
   non-wheel files behind.)
5. **Gradle:** confirm `pgrep -af GradleDaemon` shows nothing, then
   `command rm -rf ~/.gradle/caches`. Leave `~/.gradle/wrapper` and any other
   `~/.gradle` content.

### 4. Final verification and report

1. `df -h /` again. The expected result is on the order of 180–190 GB freed, which takes
   usage from ~97% to roughly 75%. The exact number depends on what the reaper keeps for
   live agents.
2. Re-run the sanity checks from step 2.5. Also confirm that `uv --version`,
   `npm --version`, and `go version` still run.
3. Report a table with each target's before size, after size, and the method used.
   Include anything skipped and why, e.g. a live build, the uv in-use timeout, or an
   unexpected reaper owner. List the out-of-scope Tier 3 items again as still pending
   the user's decision.
4. Point out that the SASE cargo scratch refills quickly: ~151 GiB accumulated in under
   3 days. This suggests the `managed_tmp_reap` job cadence or the
   `managed_tmp.horizons.build_scratch_seconds` (3-day) horizon is too loose for this
   host. Mention it as a possible follow-up; do not change SASE config in this task. If
   the user wants it tracked, a task bead can be filed via `/sase_new_task`.

## Acceptance criteria

- Every in-scope target has either been cleaned with the method above, or is reported as
  skipped with a concrete reason.
- No out-of-scope path was modified, and nothing was run with `sudo`.
- `sase`, the `sase_core_rs` import from the sase uv-tool venv, `sase-xprompt-lsp`, uv,
  npm, and go all still work after cleanup.
- No git-tracked file in any repository was changed, and nothing was committed.
