---
tier: tale
title: Keep Poseidon clear with bounded compiler caching and managed Cargo output
goal:
  Athena's normal Cargo and SASE builds cannot recreate the unbounded Poseidon target
  tree, reusable compiler results have a 40 GiB budget, and disk pressure produces
  actionable local alerts.
size: medium
proposed_by: bbugyi200.athena.3u
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.3u](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.3u.md)
- **COMMITS:**
  - [eea8af0](https://github.com/sase-org/sase/commit/eea8af0421796fed2b7458002c7a00eaa5ccccd2)
    — feat(agent): isolate Cargo build-dir for launched and recipe builds

# Keep Poseidon clear

This is one bounded implementation for a single follow-up agent. Existing SASE launch
isolation and scratch reaping already supply the lifecycle machinery; the remaining work
is host configuration, small environment-plumbing changes, and operational verification.
Implement this approved tale without introducing a new workspace lease service, a second
recursive garbage collector, or a general storage subsystem.

## Evidence and corrections to the earlier advice

The requested source is agent **0d**, on **mac**:
`~/.sase/chats/202609/gh_sase_org__sase-ace_run-0d-260913_054948.md`. Its complete
transcript was read over SSH with `sase chat show`. The affected filesystem is on
**athena**, not mac. The Mac is the transcript source; do not apply Athena paths or
Linux services to it.

The subsequent Claude Code cleanup session was found and read on **athena**:
`~/.claude/projects/-home-bryan-projects-github-sase-org-sase/9ef6816e-8c47-489f-8a1e-0b83e6311ec7.jsonl`.
Its final response is timestamped 2026-09-13 12:08 UTC. It deleted the inspected cache
contents, retained the containing directory, trimmed Poseidon, and enabled weekly TRIM.
It deliberately left Cargo configuration unchanged.

Read-only observations on 2026-09-13:

- `/mnt/poseidon` is ext4, UUID `e0d96fde-be60-4f3b-bed7-3e9700060fdb`, currently
  `/dev/sdc1`. It has approximately 217 GiB available, with only 32 KiB used. Identify
  it by mountpoint and UUID when applying changes; device names can change.
- `~/.cargo/config.toml` still consists of `[build]` and
  `target-dir = "/mnt/poseidon/cargo-target"`. That directory is empty and owned by
  `bryan:bryan`, mode 0755. No sccache executable was found.
- This agent actually inherited `CARGO_TARGET_DIR` under
  `~/.cache/sase/tmp/cargo-targets/`, with `SASE_TMPDIR=~/.cache/sase/tmp`.
  `src/sase/agent/launch_spawn.py` creates those per-launch directories.
- `src/sase/core/managed_tmp_reaper.py` already recognizes `cargo-targets` and
  `build-targets`. Its normal build retention is three days; pressure pruning requires
  at least twelve hours of staleness and checks descendants. Defaults trigger above 16
  GiB or below 32 GiB available, aiming for 8 GiB and 48 GiB available. The
  `managed_tmp_reap` housekeeping job runs hourly. These are **soft reclamation targets,
  not a hard size ceiling or proof that a build is inactive**.
- Managed Cargo scratch already occupies about 80 GiB on Athena's root filesystem, which
  has about 174 GiB available. Moving another unbounded shared target there would only
  move the failure.
- Cargo is 1.98.1. Weekly `fstrim.timer` is enabled and waiting. SMART reports PASSED,
  41 reallocated/runtime bad blocks, and zero uncorrectable/CRC errors. An existing
  `prometheus-node-exporter-smartmon.timer` collects SMART data every fifteen minutes;
  its textfile is readable at `/var/lib/prometheus/node-exporter/smartmon.prom`. The
  user systemd manager has `Linger=yes`.

I agree with the original diagnosis of unbounded, disposable build output and the need
for capacity-aware prevention. I do not adopt these claims or actions unchanged:

1. The exact reason for the many artifact variants is unresolved. The cleanup agent
   found identical compiler/features/flags and only eighteen source-path fingerprint
   values across 560 records, and hypothesized version/dependency changes. That evidence
   does not prove that checkout paths never contribute to other identity hashes. The
   deleted corpus cannot now establish a single cause. Neither hypothesis is a
   prerequisite for this fix.
2. Per-run targets and cleanup already exist. Preserve them instead of adding a new
   deletion hook on workspace release, which can precede completion of monitors,
   handoffs, or other consumers.
3. sccache bounds its own stored compiler results. It does not bound Cargo targets and
   cannot cache linked binaries, cdylibs, or proc macros. Treat dependency reuse as a
   measured benefit, not a promise to eliminate the dominant test executables.
4. Replace the proposed emergency purge with stopping new cache writes and alerting.
   Never delete arbitrary files, active targets, or everything on the volume to meet a
   percentage. The prevention mechanism is removing the unbounded writer and limiting
   its replacement, rather than relying on emergency deletion.
5. Keep the completed cleanup and TRIM work. Do not repeat it, reduce ext4 reserved
   blocks, or infer an urgent SSD replacement solely from power-on hours. Watch changes
   in health counters; do not interpret a vendor-normalized attribute as a precise
   remaining lifetime.

Relevant primary documentation checked while planning:

- [Cargo build cache](https://doc.rust-lang.org/cargo/reference/build-cache.html):
  separate final artifacts from intermediates and use sccache for dependency reuse.
- [Cargo configuration](https://doc.rust-lang.org/cargo/reference/config.html):
  `build.build-dir` supports `{workspace-path-hash}`; `CARGO_BUILD_BUILD_DIR` overrides
  it. Built-in cache GC still does not clean build artifacts.
- [Cargo profiles](https://doc.rust-lang.org/cargo/reference/profiles.html):
  `line-tables-only` retains file/line backtraces while dropping richer debug data.
- Upstream `mozilla/sccache`, read through `sase repo open`, especially `docs/Local.md`,
  `docs/Rust.md`, and `docs/Configuration.md`: local cache sizing, one server per local
  cache, incremental incompatibility, and uncached compiler fallback.

## Intended layout and behavior

| Output                               | Location and owner                                            | Retention/protection                                                                     |
| ------------------------------------ | ------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Old shared target                    | `/mnt/poseidon/cargo-target`                                  | Retired, empty, root-owned, nonwritable to ordinary builds                               |
| Compiler cache                       | `/mnt/poseidon/sccache`                                       | One local sccache server; 40 GiB configured LRU budget                                   |
| SASE run outputs and intermediates   | Existing per-launch `cargo-targets` directory                 | Existing SASE reaper and isolation                                                       |
| Ordinary Cargo intermediates         | `~/.cache/sase/tmp/build-targets/cargo-{workspace-path-hash}` | Existing managed build bucket; separate directory for each Cargo workspace               |
| Ordinary Cargo final artifacts       | Cargo's normal workspace-local `target`                       | Remain available for running/installing/debugging; never globally redirected to Poseidon |
| Explicit dev-update/prebuild outputs | Existing recipe/prebuild destinations                         | Preserve their existing isolation and retention                                          |

The 40 GiB setting is a compiler-cache budget, not a filesystem quota; metadata and
in-flight files need headroom. No claim is made that it constrains arbitrary programs or
explicit user-selected paths. Known default raw-output writers are removed from Poseidon
and its old writable destination is retired. The remaining managed scratch is observed
separately so pressure on the root filesystem does not go unnoticed.

## Implementation sequence

### 1. Recheck state and prepare reversible changes

Work locally on Athena, or connect there with the saved SSH alias. Before reading or
editing another repository, use `/sase_repo` and only its returned checkout path. The
relevant repositories are `sase` and the chezmoi source, whose origin is
`gh:bbugyi200/dotfiles`. If the `chezmoi` inventory alias does not resolve in the
implementation run, `sase repo open gh:bbugyi200/dotfiles -r "..."` works. Read their
current instructions and preserve unrelated work. Do not copy any numbered workspace
path from the planning session.

Refresh Cargo version, the live config, mount UUID, directory contents/ownership, active
build usage, sccache availability, timer state, and free space on both filesystems.
Confirm the installed SASE housekeeping job uses the same managed root as its agents and
is actually running; a source-code default alone is not verification.

Record the exact pre-change Cargo config and directory metadata for rollback. Store
host-only backups under a dated, user-private configuration backup directory, not a
managed scratch bucket. Do not back up hundreds of GiB of disposable output.

Stage all configuration and helper code in chezmoi source under `home/`. Scope its new
Cargo config, sccache files, helper executables, and units to hostname `athena` using
the existing `.chezmoiignore`/template conventions. Render against Mac and Apollo data
to prove they do not acquire these settings. Preserve any newly discovered unrelated
Cargo settings rather than replacing the live file blindly.

### 2. Preserve target isolation when using Cargo's separate build directory

Before enabling the host's `build.build-dir`, update the small amount of SASE host
environment glue that explicitly chooses Cargo destinations:

- In `src/sase/agent/launch_spawn.py`, set `CARGO_BUILD_BUILD_DIR` alongside each
  managed `CARGO_TARGET_DIR`. Derive its default from the **final effective** target
  after explicit launch overrides, so an explicitly supplied target does not leave
  intermediates in another run's bucket. An independently supplied build-dir override
  must remain effective, including a deliberately supplied empty value according to
  Cargo's semantics. Scrub stale inherited per-agent build-dir values as is already done
  by assigning a fresh target for each launch.
- Audit `Justfile` and `src/sase/dev_update/prebuild_producer.py` sites that set
  `CARGO_TARGET_DIR`: coordinate the build-dir with their deliberately isolated Python,
  LSP, and prebuild targets. Do not merge those caches accidentally through the new
  global default. Preserve final artifact paths consumed by maturin and install steps.
- For automated agent builds, default `CARGO_INCREMENTAL=0` and
  `CARGO_PROFILE_DEV_DEBUG=line-tables-only` /
  `CARGO_PROFILE_TEST_DEBUG=line-tables-only`. Respect explicit user overrides. Existing
  dev-update release/profile choices remain intact. Do not reduce debug information
  globally for interactive builds.
- Update the existing launch/recipe/prebuild tests and any relevant default-config
  explanatory text. This is environment plumbing, not a second Python implementation of
  backend retention policy. No new SASE CLI or backend API is expected.

### 3. Configure managed Cargo intermediates and capped compiler reuse on Athena

Adopt `~/.cargo/config.toml` into chezmoi source, normally
`home/dot_cargo/config.toml.tmpl`, and remove the global `build.target-dir` entry.
Configure `build.build-dir` to the absolute Athena managed `build-targets` path shown
above, retaining the literal Cargo `{workspace-path-hash}` placeholder. Use the host
home path from chezmoi data, not the implementation checkout path. This captures the
large dependency/incremental trees from ordinary Cargo and maturin invocations without
creating another shared global target.

Prove Cargo 1.98.1 and the installed maturin support this split before applying it to
the live configuration. If a real installed-tool incompatibility appears, resolve it
before enabling the split; do not silently fall back to a new unmanaged shared target.

Install sccache from the system package manager or a verified upstream binary, choosing
a version that supports the selected options. Prefer this to compiling a large new
toolchain cache just to install the cache. Verify its version and actual disk-backend
configuration; do not assume upstream HEAD options exist in an older package.

Create `/mnt/poseidon/sccache` for `bryan`, mode 0700, only after confirming the mounted
UUID. Configure a single disk backend with `dir` pointing there and `size = 42949672960`
bytes. Use one dedicated local Unix socket for this cache so an unrelated default server
cannot silently supply a different configuration. Every managed client and
administrative stats command must use the same configuration/socket. Ensure inherited
sccache configuration cannot select another directory, a larger budget, another server,
or a remote backend. Normalize the wrapper's sccache-specific environment to the managed
settings while preserving compiler/build inputs; never log credentials. Do not enable
remote storage or experimental path remapping.

Set Cargo's `build.rustc-wrapper` to a chezmoi-managed Athena wrapper, using an absolute
path so non-login commands and maturin find it. It must preserve the compiler argument
vector and exit status. Before contacting/starting sccache, check the exact mount and
UUID and reject symlink/redirection to a different filesystem. If the volume is missing,
inaccessible, at least 80% full, or has less than 32 GiB available, execute the supplied
real compiler without caching. Do not create a fallback cache beneath an unmounted
`/mnt/poseidon` or elsewhere on `/`. Use the supported sccache server-I/O fallback
behavior for cache failures; never retry a real compiler error as success.

Use `[build] incremental = false` as Athena's normal default so eligible Rust crates can
use sccache. Document an explicit interactive debugging invocation with
`CARGO_INCREMENTAL=1`, full profile debug information, and `RUSTC_WRAPPER=''`. The
wrapper remains optional for correctness; cache hits are an optimization.

Document the separate-directory semantics explicitly: outside SASE's launch handling, an
ad hoc `CARGO_TARGET_DIR` override changes final output, while the configured build-dir
still controls intermediates. Show how to override both variables when the user wants a
wholly separate build tree, and verify `cargo clean` honors the selected pair. Do not
assume a target-dir override alone still relocates every artifact.

After switching defaults and confirming there are no remaining consumers, retire the old
**empty** `/mnt/poseidon/cargo-target` with owner `root:root` and mode 0555. Check that
it is a real directory under the expected mount, not a symlink. This prevents stale
ordinary-user target overrides from rebuilding the old global cache. If it has become
nonempty or is in use, do not delete or chmod it underneath the writer: identify and
migrate that use first, preserving newly created contents until safely handled.

### 4. Add pressure visibility without a competing deletion job

Add an Athena-only `poseidon-cache-watch` helper plus a systemd user oneshot/timer in
chezmoi. Run inexpensive filesystem checks every five minutes, including after boot;
check larger directory totals at most hourly with a bounded execution time. Use the
existing lingering user manager, absolute executable paths, and a nonoverlap lock.

Compute utilization using ordinary-user available blocks, consistent with `df`, and also
report absolute available bytes. Watch the entire Poseidon filesystem, not just the
compiler-cache directory. Emit state-transition notifications to Bryan's local SASE
inbox using the existing notification interface, with useful journal output when that
interface fails. Persist compact state atomically outside reapable scratch. Notification
state advances only after successful delivery; deduplicate repeated ticks and report
recovery once, with hysteresis below 70% utilization and above 48 GiB free.

- At 75% utilization, warn with mount identity, free bytes, and cache location/budget.
- At 80%, or below the 32 GiB floor, report that new compiler-cache requests bypass
  sccache. Already executing builds are allowed to finish; do not kill/restart their
  cache server as an emergency action.
- At 90%, issue a critical local alert naming unexpected raw output/other disk use as
  something to inspect. Do not claim sccache can clean arbitrary target directories.
- Warn immediately if the mount is missing/wrong, the old target is writable or gains
  contents, the configured cache budget/socket drifts, or the watcher cannot inspect
  required state. Do not mask failures as a healthy empty volume.
- Watch root-filesystem free space at the existing 32/48 GiB floor/recovery thresholds.
  If managed scratch stays above its configured soft target without successful reaper
  progress across two hourly checks, include an actionable warning about stale or
  ineligible entries and the last housekeeping outcome. Do not lower age thresholds or
  delete entries merely to make a 16 GiB number pass.
- Reuse the readable existing SMART collector output for a warning on reallocated or
  runtime bad-block counts increasing beyond the confirmed baseline, new uncorrectable
  errors, failed health, or persistently stale collector output. Preserve the existing
  SMART and TRIM services instead of installing competing maintenance jobs.

Keep monitor state/log volume bounded. Routine successful checks should be quiet.
Notifications are local to the user's existing SASE inbox; no email, Slack, or other
third-party messaging is part of this plan.

### 5. Verify, deploy, and leave a usable rollback

Add focused tests for the behaviors that can redirect writes or mis-handle pressure:
environment precedence, argument/exit-code preservation, missing/wrong mount, threshold
boundaries and recovery, notification retry/deduplication, and host template isolation.
Use fake filesystem samples and isolated fixtures for pressure tests; do not fill a real
disk, unmount Poseidon, kill a user build, or alter live SMART data for testing.

Exercise the finished implementation end to end with small disposable Rust projects:

1. Run ordinary Cargo, a SASE launch-environment path, and a maturin build. Observe
   actual output locations, including `deps`, final binaries, and the Python artifact.
   Check custom target/build-dir overrides and the existing isolated dev-update paths.
2. Build eligible library code twice with different empty output directories, observing
   real sccache statistics showing reuse. Verify linked/test outputs are still present
   where expected. Do not label an unchanged Cargo no-op as a compiler-cache hit.
3. With a separate temporary cache and socket, exercise LRU eviction under a small
   budget using real compilations. Verify output remains correct and production cache
   state is untouched. Document any cache overhead rather than promising an exact
   filesystem quota.
4. Exercise the wrapper's missing-mount/pressure fallback and a failing compiler through
   the public command interface. Both successful uncached compilation and propagation of
   compiler failure must be observed.
5. Exercise the existing reaper on an isolated managed-root fixture containing an old
   build tree, a tree with a fresh descendant, unrelated scratch, and a symlink. Verify
   the new ordinary-Cargo directory naming is eligible and protected entries survive. Do
   not run a destructive reaper against live state as a test.
6. Verify rendered systemd units, execute the watcher with normal live read-only
   samples, and exercise notification transitions through isolated notification state.
   Confirm its scheduled next run and inspect journal output. Validate delivery to the
   actual local inbox with one clearly marked setup notification after deployment.

Read applicable verification memory. For tracked SASE changes run `just check`; use the
required monitor workflow if checks become long-running. Run relevant chezmoi
tests/linters, template rendering and `systemd-analyze --user verify` for its units. Do
not suppress failures or claim success based only on configuration text.

Deploy the SASE environment changes before enabling the new global Cargo build-dir.
Verify the installed SASE version actually contains the new environment handling;
changing source in a checkout does not update a running installation by itself. Use the
repository's normal host-owned completion/deployment path; no manual commits or
unreviewed global reinstalls. For dotfiles, obey its instruction to apply after the host
lands the commit (`chezmoi update -a --force`), inspect the pending diff first, and
ensure the implementation checkout's changes are the version being applied. Coordinate
that host-owned continuation so the task includes the actual Athena apply, not just
source edits waiting indefinitely for a future login. Apply the scoped units, reload the
user manager, enable the watcher timer, and verify the live Cargo config, cache
directory/budget/socket, retired directory, and completed build behavior.

Add a concise ordinary runbook in the dotfiles repository, for example
`docs/poseidon_cargo_cache.md`. This is documentation, not a SASE memory edit. Include
effective paths, the soft-versus-bounded distinction, cache stats, watcher status,
uncached/full-debug overrides, how to investigate unexpected usage, and rollback.
Rollback disables the new watcher and compiler wrapper and restores the recorded
configuration deliberately; it must not automatically reinstate the unbounded global
Poseidon target. Undoing the retired directory permissions is an explicit migration
step, not routine rollback. Avoid deleting source, installed tools, or final artifacts.

## Acceptance criteria

- Normal SASE, ordinary Cargo, and maturin builds succeed and their observed output
  locations match the intended layout. No default build writes raw targets to Poseidon.
- The old shared destination cannot be repopulated by a stale ordinary-user override.
- Athena uses one sccache disk backend configured for 40 GiB, real cache hits are
  observed for eligible code, and missing/pressured storage falls back to correct
  uncached compilation without relocating the compiler cache.
- Existing SASE per-run and explicit recipe isolation still works, including launch
  overrides; ordinary Cargo intermediates participate in existing managed retention.
- Pressure/failure/recovery notifications work without spam or destructive purging, and
  root scratch growth is visible as a separate soft-retention concern.
- Weekly TRIM and existing SMART collection remain operational. No reserved-block
  tuning, broad cache deletion, SSD replacement, or Mac storage changes occur.
- Required checks pass, host deployment is verified, and the final report states any
  remaining operational limits rather than describing soft cleanup as a hard cap.
