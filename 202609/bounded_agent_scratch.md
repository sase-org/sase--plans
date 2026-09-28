---
tier: epic
title: Bound agent scratch by ownership, not by environment luck
goal: 'Per-launch agent scratch (cargo targets, agent TMPDIRs) is removed when its
  launch is dead, on every managed temp root any writer actually used, regardless
  of which environment the service host was started with; cleanup refusals are visible;
  and `sase disk list` / disk-pressure notifications account for where the bytes really
  are, so a SASE host can no longer silently fill its disk.

  '
phases:
- id: root-registry
  title: Managed temp root registry the reaper follows
  depends_on: []
  size: medium
  description: 'root-registry: every root `get_sase_managed_tmpdir()` writes into
    is recorded in a Rust-owned registry under SASE_HOME, and every managed-tmp reaper
    entry point reaps all registered roots instead of only the root its own environment
    resolves.'
- id: scratch-liveness
  title: Rust-owned launch scratch liveness that works under systemd
  depends_on: []
  size: medium
  description: 'scratch-liveness: move the procfs liveness probe into sase-core, stop
    treating pre-launch non-dumpable processes (systemd --user, sd-pam, ssh-agent)
    as incomplete observations, and make runner-exit cleanup log every outcome.'
- id: dead-launch-reap
  title: Dead-launch backstop pass and liveness-aware pressure
  depends_on:
  - root-registry
  - scratch-liveness
  size: medium
  description: 'dead-launch-reap: the housekeeping reaper removes launch-keyed scratch
    that no live process holds after a short grace, pressure pruning becomes liveness-aware
    with a realistic minimum entry size, and defaults and docs move with it.'
- id: disk-attribution
  title: Truthful disk attribution under pressure
  depends_on:
  - root-registry
  size: medium
  description: 'disk-attribution: `sase disk list` and the disk_pressure job cover
    every registered root and every workspace checkout before the stray walk, report
    unattributed bytes, and workspace compaction stops over-reporting hardlinked bytes.'
- id: visual-run-retention
  title: Retention for visual snapshot run reports
  depends_on: []
  size: small
  description: 'visual-run-retention: the visual maintenance tooling prunes old `.pytest_cache/sase-visual/runs/`
    directories while preserving the latest report, recent runs, and any unfinished
    apply journal.'
- id: host-acceptance
  title: Integrated acceptance on apollo and athena
  depends_on:
  - root-registry
  - scratch-liveness
  - dead-launch-reap
  - disk-attribution
  - visual-run-retention
  size: small
  description: 'host-acceptance: after deployment, prove on apollo and athena that
    the registry finds the agent root, runner exit removes scratch, the backstop reaps
    dead launches, and disk attribution matches df.'
proposed_by: bbugyi200.kellys_mbp.1d
create_time: 2026-09-27 14:23:26
status: done
bead_id: sase-1bf
---

- **BEAD:** [sase-1bf](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bf/README.md)

# Plan: Bound agent scratch by ownership, not by environment luck

## Incident Evidence (2026-09-27)

apollo (DigitalOcean, 193 GiB root disk) hit 100% full (442 MiB free). Investigation
found:

- `~/.cache/sase/tmp/cargo-targets` held **90 GiB in 279 per-launch directories** (one
  per agent launch, 0.8–4.4 GiB each for sase/sase-core builds, dating back 6 days
  despite the 3-day build horizon). `agent-tmp/` held another 876 MiB in 388
  directories.
- **Root split.** `~/.profile` exports `SASE_TMPDIR=~/.cache/sase/tmp`, so the
  tmux/TUI-launched agents write there. The systemd user unit `sase.service` loads
  `~/.sase/service/env`, which was captured on 2026-09-20 from a shell without
  `SASE_TMPDIR` and contains only `PATH`. The housekeeping `managed_tmp_reap` job
  therefore scanned `~/.sase/tmp` (21–51 entries per pass, `selected=0`) and never saw
  the real root. A long-lived tmux server still carries a third value
  (`SASE_TMPDIR=~/tmp/sase`), and that root also holds bucket directories. This is the
  same failure mode as sase-15q, whose fix (capture `SASE_TMPDIR` into the service env
  plus an init-time warning) is a point-in-time capture that silently went stale. apollo
  also needed an emergency reclaim on 2026-09-14 (sase-10r).
- **Runner-exit cleanup never succeeds on Linux systemd hosts.**
  `src/sase/axe/run_agent_runner_scratch.py` scans every same-uid `/proc/<pid>`, and
  `/proc/<pid>/environ` and `cwd` are `EACCES` for the user's own non-dumpable
  processes: `systemd --user`, `(sd-pam)` and `ssh-agent`. Every probe returns
  `complete=False`, and `sase_core::managed_tmp::reap_launch_scratch` then preserves
  both candidates ("preserved because liveness was incomplete").
  `cleanup_launch_scratch` discards that result and swallows all exceptions, so nothing
  records the refusal. A live reproduction on apollo with a synthetic scratch key
  confirmed `live=False complete=False`, and both directories survived cleanup.
- **The age and pressure policy cannot bound the inflow even with the right root.**
  athena's service env _does_ capture `SASE_TMPDIR`, and its reaper removes ~57 entries
  per pass. Its `~/.cache/sase/tmp/cargo-targets` still sits at **92 GiB across 943
  entries** (~300 launches/day). The pressure pass only considers entries of at least
  `managed_tmp.pressure.min_entry_bytes` (1 GiB), and the typical sase-core build target
  is 810–890 MiB, so most entries are never pressure candidates.
- **Attribution lied.** The disk_pressure job named `rust_prebuild_cache 3.4 GiB` as the
  top owner while ~90 GiB sat in the unscanned root. `sase disk list` reported
  `fallback walk truncated; visited=0` for the 47 GiB workspace tree and skipped the
  primary checkout because its shared scan budget ran out.
- **Unbounded visual run reports.** `.pytest_cache/sase-visual/runs/` had no retention:
  556 runs (8.3 GiB) in one workspace and 14.3 GiB in total across four workspaces.
- `sase workspace compact` reported 13.2 GiB reclaimed on apollo while `df` moved by ~3
  GiB, because locally cloned workspace objects are hardlinks to the primary's object
  store.

The immediate cleanup used SASE's own owners, not ad-hoc deletion. It ran
`reap_managed_tmpdir` against the real root, a pressure pass with a lowered entry floor,
and `sase disk reap --apply`. It also pruned visual runs older than 48h, keeping each
latest-report run and any unfinished journals. apollo now has 106 GiB free, but nothing
prevents a repeat.

## Design Principles

1. **The reaper follows writers, not environment variables.** Where scratch lands is
   decided by whichever process wrote it, so the set of roots to reap is whatever
   writers actually used, recorded durably at write time.
2. **Scratch lifetime follows its owner.** A per-launch directory is garbage the moment
   no live process holds it, and age horizons stay only as a fallback for hosts where
   liveness cannot be observed.
3. **A refusal must be visible.** Every preserved-for-safety outcome is counted and
   logged, so a probe that is always incomplete is caught in hours, not after the disk
   fills.
4. **Attribution must account for the disk.** When the owner table cannot explain `df`,
   it says so with an unattributed row instead of pointing at a small owner.

Alternatives considered and rejected:

- **Keep the env-capture approach (sase-15q) and re-run `sase service init`.** This
  already failed once. A captured snapshot rots whenever the profile, tmux server, or
  init shell differs.
- **Make config the single source of truth and ignore `SASE_TMPDIR`.** That breaks
  explicit per-shell overrides and still diverges whenever a process resolves its root
  differently. The registry tolerates divergence instead of forbidding it.
- **Shared per-workspace `CARGO_TARGET_DIR`.** This bounds bytes by workspace count
  rather than time: athena has 44+ workspaces with 0.9–7.6 GiB targets. It gives up
  per-launch isolation and still needs the same retention machinery. It is a possible
  later build-speed optimization, not the disk fix.
- **Only shorten horizons.** athena proves that age plus pressure cannot keep up with
  ~300 launches/day on its own, so ownership-based removal is required.

## Repository Boundaries

Retention, liveness, and registry decisions are shared backend behavior. They live in
the `sase_core` crate of the linked sase-core repo; open it with
`sase repo open sase-core` and work in the printed path. Python in this repo stays a
thin adapter over `sase_core_rs`. Every phase that adds or changes a binding must move
`sase-core-revision.txt` past its sase-core commit as described in
`docs/rust_backend.md`. The managed-tmp wire lives in
`sase_core::managed_tmp::MANAGED_TMP_REAP_WIRE_SCHEMA_VERSION` (currently 3) and must
match `src/sase/core/managed_tmp_reaper.py`.

Related beads: epic sase-zw (Bound SASE's disk footprint on a long-running host) is
still open. sase-15q, sase-10r, sase-zn.6 and sase-zw.8.7.1 (current Python liveness
probe) are the prior work this plan corrects.

## Phase root-registry: Managed temp root registry the reaper follows

Add a Rust-owned registry of managed temp roots, for example a `managed_tmp_roots`
module in sase_core with a binding.

- **File.** Store it at `$SASE_HOME/managed_tmp/roots.json` with a schema version and
  entries `{path, first_seen_epoch, last_seen_epoch}`. Writes are atomic (temp file plus
  rename) under a lock file. Read-modify-write merges concurrent registrations.
- **Validation (Rust).** Only absolute, non-symlink directory paths are registered.
  Reuse the reaper's existing broad-root refusal (`ManagedTmpReapError::UnsafeRoot`, the
  sase-157.1 guard refusing `TMPDIR`/`HOME`-shaped roots), so a transient bad
  `SASE_TMPDIR` can never enroll `/tmp`, `$HOME` or `/`. Entries whose path no longer
  exists are dropped on the next write.
- **Writer hook.** `get_sase_managed_tmpdir()` in `src/sase/core/paths.py` registers its
  resolved unsandboxed root once per process per root, using a module-level set.
  `last_seen` is rewritten only when it is more than an hour stale, to avoid write
  amplification. Registration is fail-open: a failure logs one warning and never breaks
  a launch. Under pytest the sandbox root is never registered, and the
  `state_write_guard` sandbox rules stay intact.
- **Readers.** Every managed-tmp reaper entry point enumerates _effective root ∪
  registered existing roots_, de-duplicated by resolved path, and reaps each with the
  same Rust reaper and horizons:
  - the housekeeping chop `src/sase/scripts/sase_chop_managed_tmp_reap.py`, with
    per-root lines plus totals in the structured summary;
  - the disk_pressure owner-safe cleanup
    (`src/sase/scripts/sase_chop_disk_pressure.py`);
  - the managed-tmp step of `sase disk reap`.

  Each pass also registers the effective root and `$SASE_HOME/tmp` if it exists. The
  existing `_unmanaged_default_root_warning` becomes an informational root list, because
  a mismatch is now harmless. Runner-exit cleanup keeps using its own environment's
  root, since that is where its scratch was created.

- **Docs.** Update the `managed_tmp_reap` paragraph in `docs/axe.md` (including the
  "reverse mismatch is not detected" sentence, which becomes false) and the
  `managed_tmp` section of `docs/configuration.md`.
- **Tests.** Cover the incident shape: a writer process with `SASE_TMPDIR=X` creates
  `cargo-targets/<key>`, then a reaper process with no `SASE_TMPDIR` reaps X. Also cover
  unsafe-root refusal, the fail-open writer, concurrent registration merge, and pruning
  of missing roots.

## Phase scratch-liveness: Rust-owned launch scratch liveness that works under systemd

Move launch-scratch liveness observation from `src/sase/axe/run_agent_runner_scratch.py`
into sase_core, next to `reap_launch_scratch`, and fix the systemd false-incomplete.

- **Semantics preserved.** Only same-uid processes are considered, and the observer
  skips its own pid plus caller-supplied exempt pids. A process holds a candidate when:
  - its environ has `TMPDIR`, `TMP`, `TEMP`, `CARGO_TARGET_DIR` or
    `CARGO_BUILD_BUILD_DIR` at or under the candidate path, or
    `SASE_LAUNCH_SCRATCH_KEY=<key>`; or
  - its cwd resolves at or under the candidate.
- **New pre-launch exemption.** When a process's `environ` or `cwd` is unreadable,
  compare its start time with the candidate directory's birth time. The start time comes
  from world-readable `/proc/<pid>/stat` field 22 plus `btime` in `/proc/stat`. The
  birth time comes from statx via `Metadata::created()`. A process that started strictly
  before the scratch directory existed cannot have inherited its environment, so it is
  neither a holder nor an incomplete observation. If birth time is unavailable, or the
  process started later, keep today's fail-closed behavior. Record counts of exempted
  and still-unreadable pids in diagnostics.
- **API shape.** Provide a batch observer that scans `/proc` once and answers for many
  candidates; the dead-launch-reap phase uses the batch form. On hosts without a usable
  procfs (macOS), return an explicit `unobservable` status rather than `complete=false`.
- **Runner adoption.** `cleanup_launch_scratch` calls the binding instead of the Python
  probe; delete `_candidate_liveness` and its helpers. Keep candidate derivation (the
  bucket path must equal the exported `TMPDIR`/`CARGO_TARGET_DIR`). The function must
  still never raise. It must log one structured line per exit to the runner log:
  `launch_scratch_cleanup` with removed, skipped, bytes, and the skip reasons or
  exception with traceback. Operators and the acceptance phase can then see refusals.
- **Handoff outcomes.** Keep the `SHELL_HANDOFF_OUTCOMES` (monitor/gate) skip. Verify,
  and document in the phase notes, whether a gate or monitor follow-up turn gets a fresh
  scratch key from `launch_spawn._managed_agent_scratch_env` or reuses the old one. The
  dead-launch-reap phase depends on that answer.
- **Tests.** Include a fake proc root with an unreadable-environ process started before
  the candidate, which must be exempt, and one started after it, which must stay
  incomplete. Keep the existing live-holder and cwd tests.

## Phase dead-launch-reap: Dead-launch backstop pass and liveness-aware pressure

Add a backstop so scratch from crashed, SIGKILLed, OOM-killed or handoff-ended launches
does not wait days for an age horizon.

- **Dead-launch pass.** Add a Rust reaper pass over the children of the launch-keyed
  buckets `agent-tmp`, `cargo-targets` and legacy `build-targets`. An entry is removed,
  largest first and within `max_removals`, when all of these hold:
  - its newest descendant mtime is older than `managed_tmp.dead_launch.grace_seconds`
    (default 7200);
  - the batch observer reports no holder, with a complete observation.

  Report `dead_launch_scanned`, `_selected`, `_removed`, `_reclaimable_bytes`,
  `_reclaimed_bytes`, `_preserved_live`, `_preserved_incomplete` and
  `dead_launch_observer` (`procfs` or `unobservable`). When the observer is
  unobservable, the pass skips and the age horizon remains the fallback.

- **Handoff keys.** If the scratch-liveness phase found that gate or monitor follow-ups
  reuse a scratch key, treat keys of agents with a pending gate or monitor as held.
  Agent metadata already records `launch_scratch_key` in
  `src/sase/axe/run_agent_markers.py`. Otherwise no special casing is needed.
- **Liveness-aware pressure.** When the observer is available, pressure never removes a
  held entry, and an unheld entry needs only the dead-launch grace rather than
  `pressure.min_age_seconds`. Lower `managed_tmp.pressure.min_entry_bytes` from 1 GiB to
  64 MiB. Largest-first ordering already prefers big entries, and the 1 GiB floor
  excluded the typical 0.8–0.9 GiB target. Lower
  `managed_tmp.horizons.build_scratch_seconds` from 3 days to 1 day, since it is now
  only the non-procfs fallback.
- **Config.** Add `managed_tmp.dead_launch.enabled` (default true) and `grace_seconds`
  with getters in `sase.config`. Update `src/sase/config/_settings_system.py` defaults,
  `src/sase/default_config.yml` (per the gotcha: keep it in sync), and
  `src/sase/config/sase.schema.json`.
- **Wiring.** Bump the managed-tmp wire schema version on both sides. Wire the new
  fields through `src/sase/core/managed_tmp_reaper.py`, the housekeeping chop summary,
  the disk_pressure owner-safe cleanup, and the `sase disk reap` summary.
- **Docs.** Update `docs/axe.md` (managed_tmp_reap and disk_pressure paragraphs),
  `docs/configuration.md` and `docs/rust_backend.md` where they describe launch scratch.
- **Tests.** Rust: a dead entry past grace is removed; a held entry is kept; an
  incomplete observation is kept; the unobservable status skips; the budget is
  respected. Python: the adapter field mapping and the chop summary.

## Phase disk-attribution: Truthful disk attribution under pressure

Make `sase disk list`, `sase disk reap` and the disk_pressure notification explain where
the disk went. Follow where each piece of logic already lives: if classification is in
sase-core (the inventory decodes a Rust wire in
`src/sase/core/disk_footprint_inventory.py`), change it there.

- **Every registered root.** Managed-tmp rows cover every registered root from the
  root-registry phase, not only the effective one.
- **Scan ordering.** Size SASE-known heavy locations before the generic stray walk, so
  budget exhaustion can only clip the stray walk. Known locations are the managed roots,
  each project's workspace root under the workspace provider's state directory, and the
  primary checkouts. Workspace checkouts get their own rows per project with sub-rows
  for `.git/objects`, `.pytest_cache` (including `sase-visual`), `sase/repos`, `.venv`
  and in-tree `target/`. Today this tree reports `fallback walk truncated; visited=0`.
- **Unattributed row.** When coverage is partial or the filesystem is under pressure,
  emit an `unattributed` row equal to filesystem used minus attributed physical bytes on
  the filesystem that holds SASE_HOME. The disk_pressure notification must lead with
  that row when it exceeds the largest attributed owner.
- **Compaction accounting.** `sase workspace compact` must report only bytes it actually
  frees: count files with `st_nlink == 1`, or de-duplicate inodes shared with the
  primary object store. Start at `reclaimed_bytes` in
  `src/sase/workspace_provider/_git_objects_model.py`.
- **Tests.** Cover budget exhaustion that still reports workspace rows, the
  unattributed-row math with a fake statvfs, and hardlinked-object accounting.

## Phase visual-run-retention: Retention for visual snapshot run reports

Bound `.pytest_cache/sase-visual/runs/` in the visual maintenance tooling under
`tests/ace/tui/visual/`, which owns `maintenance.lock`, run directories,
`latest-report.json` and `apply-journal.json`.

- **When.** At the end of each maintenance or update run, while still holding
  `maintenance.lock`, prune run directories. This covers check mode and every update
  outcome.
- **What survives.** Always keep:
  - the run `latest-report.json` points at, and the current run;
  - any run whose `apply-journal.json` has status `planned` or `applying`
    (`UNFINISHED_STATUSES`), or is unreadable;
  - the 10 most recent runs;
  - any run with a descendant modified in the last 24 hours.

  Delete the rest. Symlinks are never followed.

- **Docs.** Document the retention in `docs/development.md` next to the paragraph that
  says every run retains a reviewable report. No memory note changes:
  `sase/memory/lint_and_test.md` stays accurate.
- **Tests.** Cover retention selection with synthetic run directories, including an
  unfinished-journal run, a latest pointer and a recent run.

## Phase host-acceptance: Integrated acceptance on apollo and athena

After all phases land and apollo and athena run the new code (both run editable installs
from their primary checkouts; use their normal update path), verify on both hosts over
SSH. Read the `tailnet.md` reference memory for access.

1. **Registry.** On apollo, before touching the stale service capture, confirm
   `~/.sase/managed_tmp/roots.json` lists `~/.cache/sase/tmp` while
   `~/.sase/service/env` still lacks `SASE_TMPDIR`. Confirm the next housekeeping
   `managed_tmp_reap` line reports that root.
2. **Runner exit.** Launch a trivial agent that creates files under `$CARGO_TARGET_DIR`
   and `$TMPDIR`. Confirm its runner log shows `launch_scratch_cleanup` with both
   directories removed.
3. **Backstop.** Confirm `dead_launch_removed > 0` or an explained zero in the
   housekeeping summary, and that no directory held by a running agent was removed.
4. **Attribution.** Confirm `sase disk list` shows workspace rows and that attributed
   plus unattributed bytes are within 10% of `df` used.
5. **Record.** Log the `cargo-targets` bucket size and entry count on both hosts in the
   phase notes.
6. **Stale capture.** Report whether apollo's service env capture is still stale. Do not
   change host service configuration in this phase.
