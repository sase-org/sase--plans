---
tier: epic
status: done
title:
  Finish the migration kit's macOS rehearsal leg and publish its acceptance evidence
goal:
  Complete the one outstanding leg of `kit-rehearsal` -- the macOS rehearsal and mac's
  deferred G3 probe -- then publish the per-host operation manifests for athena, mac,
  and apollo plus the rehearsal acceptance receipt as artifacts attached to
  `sase-x7.2.1.4`, so `sase-x7.2.1` can close on fleet-wide evidence instead of
  Linux-only evidence.
phases:
  - id: mac-leg
    title: Rehearse the migration kit on protected copies of mac's real data
    depends_on: []
    description:
      "mac-leg: Wait for a `mac` reachability window, build the kit revision in an
      isolated scratch clone and throwaway uv venv on Darwin, run the synthetic matrix
      and the protected real-data rehearsal against copies of mac's `~/.sase` subsets,
      complete mac's deferred census gap G3 probe, and write the mac evidence locally."
    size: medium
  - id: publish-evidence
    title: Fold the mac results in and publish the four kit-rehearsal artifacts
    depends_on:
      - mac-leg
    description:
      "publish-evidence: Fold the mac leg into the rehearsal receipt, finish the athena,
      mac, and apollo per-host operation manifests, and publish all four as artifacts
      attached to `sase-x7.2.1.4`."
    size: small
parent_bead: sase-x7.2.1
proposed_by: bbugyi200.athena.sase-x7.2.1.land
bead_id: sase-x7.2.1.5
create_time: 2026-09-09 19:52:28
---

- **PROMPT:**
  [prompts/202609/mac_rehearsal_leg.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/mac_rehearsal_leg.md)
- **PARENT:** [202609/migration_kit.md](migration_kit.md)
- **BEAD:**
  [sase-x7.2.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-x7/sase-x7.2.1.5.md)

# Finish the macOS rehearsal leg

The migration kit itself is built, landed, and green. Its parent epic `sase-x7.2.1`
cannot close because exactly one of `kit-rehearsal`'s legs was never run and none of its
required artifacts were ever published. This plan is only that remainder. It adds no
scope: every requirement below is already written in the parent plan's `kit-rehearsal`
section and is quoted rather than reinvented.

## Why this work is still open

`sase-x7.2.1.4` shows `CLOSED`, but that status is a `sase stitch create` side effect,
not a completion. Note #1 on that bead is the phase worker explicitly declining to
close; note #3 is the auto-close firing minutes later when commit `16153bf56` landed.
The landing review note on the same bead records what a `sase-x7.2.1.land` audit on
2026-09-06 confirmed:

- **The macOS leg was never run.** `tailscale status` reports `kellys-macbook-pro` as
  offline (last seen ~30 minutes earlier); an SSH attempt and a `tailscale ping` both
  timed out. The parent plan forbids substituting a Linux run.
- **None of the artifacts were published.** `sase artifact list -e` finds no per-host
  operation manifest and no rehearsal receipt. Drafts for athena and apollo plus the
  receipt exist only as local files, and the receipt's own title still reads
  `DRAFT -- mac leg outstanding, not yet published`.
- **mac's G3 probe is still outstanding.** `kit-backup` explicitly deferred mac's
  distribution, entry-point, completion, `launchd`, and cron inventory to this phase's
  reachability window; it was never taken.

Everything else in the epic is verified done and must not be redone: the core
`migration` module and its nine `migration_*` bindings shipped in `sase-core-rs`
`v0.32.25`, the host floor and pin are current, the whole `src/sase/migration_kit/`
surface plus the `sase migrate` command group is landed, and the kit's test lane is
green (72 passed). The athena real-data rehearsal is complete and its measurements
stand.

## Why this is an epic rather than a tale

Two phases, and the split is a safety property rather than scheduling convenience. The
mac leg's cost is unbounded on its input side: it waits on a laptop that is offline
unless its lid is open, and then does a from-source Rust build on Darwin. Publishing is
cheap but irreversible -- `sase artifact create` explicit snapshots are immutable and
permanent, so a receipt must never be published while any leg is still provisional.
Keeping them separate means the expensive evidence is durably on disk before anything is
frozen into the artifact store, and a stalled reachability window cannot strand a
half-published receipt. A tale also cannot carry `parent_bead`, which is the link that
lets `sase-x7.2.1`'s land agent resume its interrupted landing once this work lands.

## Context and evidence

Read through the audited artifact interface, not by opening files directly:

- `file:explicit:5d205f8bf16e8de09e033937` -- the `fleet-census` ledger (per-host data
  locations, findings F1-F6, gaps G1-G5).
- `file:explicit:50875b31c45c9d504c5dce72` -- the `fleet-census` narrative report.
- `file:explicit:a0fd26f83bdbbb9f851b0216` -- the G3 fleet drain inventory published by
  `kit-backup`. athena and apollo are fully probed there; **mac's section is the gap
  this plan closes.**

Read the parent plan `plan:202609/migration_kit.md` in full. Its `kit-rehearsal` section
is the acceptance bar, its "Locked design decisions" section is binding, and its
"Execution rules for every phase" section applies unchanged here.

Facts established by the completed phases:

| Fact                         | Value                                                                                        |
| ---------------------------- | -------------------------------------------------------------------------------------------- |
| Kit revision to rehearse     | master tip carrying `16153bf56`                                                              |
| Core dependency              | `sase-core-rs>=0.32.25,<0.33.0`; pin `050415532b16572fe443fc80c159d516e00baa67`              |
| mac SSH                      | alias `mac`, user `bbugyi`, tailscale name `kellys-macbook-pro` on `tail297af1.ts.net`       |
| mac platform                 | Darwin 26.5, Python 3.12.5, editable install from a git checkout launched via a uv tool venv |
| mac toolchain                | `cargo`, `rustc`, `uv`, `git`, `rsync` all present                                           |
| mac `~/.sase`                | 2.2G, with 62G free on `$HOME`                                                               |
| mac Tier B residue           | none at census time                                                                          |
| mac import-leg purge preview | 9,894 artifacts / 6,501 chats / 9,924 dismissed bundles                                      |
| Kit write root               | `$HOME/cutover-backups`, mode 0700, overridable with `SASE_CUTOVER_BACKUP_DIR`               |

The athena and apollo manifest drafts and the draft receipt were left by `sase-x7.2.1.4`
at `/var/tmp/sase-x7-2-1-4-rehearsal/evidence/` on athena (`athena_manifest.md`,
`apollo_manifest.md`, `rehearsal_receipt.md`), alongside the athena run's JSON evidence
logs. `/var/tmp` is not guaranteed to survive a reboot: the first phase to touch them
must copy them somewhere durable before relying on them, and if they are already gone,
rebuild them from the athena leg's measurements recorded in note #1 on `sase-x7.2.1.4`
rather than re-running athena.

## Execution rules

These inherit from the parent plan and the parent epic and are not negotiable.

- **No production data may be mutated on any host.** The only permitted real-host writes
  are backups (read-only with respect to their sources, written outside every runtime
  root) and writes inside scratch copies the phase created. Every catalog operation runs
  in `--apply` mode only against scratch copies.
- On mac specifically: never run `sase update`, never touch the live editable install,
  and never let an editable refresh pick the kit up. The scratch clone must not be a
  worktree of mac's live checkout.
- Read census and inventory artifacts only with `sase artifact read <ref> "<why>"`. Open
  any repository other than your own workspace checkout only through `/sase_repo`.
- Use `/sase_monitor` for the reachability wait, the Darwin Rust build, and the
  rehearsal runs. Do not promise a later continuation.
- Phase workers create no beads. Record discovered work with
  `sase bead note <phase bead> 'PROPOSED FOLLOW-UP: <summary -- detail>'`.
- Run `sase bead epic-symbols <your own phase bead>` before finishing and clear or
  re-key every leftover entry.
- Completion is host-owned: submit a declaration, never create commits, branches, or PRs
  directly.
- **Close nothing above your own phase bead.** Do not close `sase-x7.2.1.4`,
  `sase-x7.2.1`, `sase-x7.2`, or `sase-x7`. `sase-x7.2.1`'s land agent resumes its
  interrupted landing through this plan's `parent_bead` link once this epic lands.

## Refusals and stop conditions

Stop and report rather than working around any of these:

- mac stays unreachable through the phase's monitored wait. Do not substitute a Linux
  run and do not publish a receipt that claims a mac leg that did not happen.
- mac's toolchain is missing or the from-source `sase_core_rs` build fails on Darwin.
- Free space, checksum verification, or a SQLite integrity check fails on mac.
- An operation's precondition cannot be proved on mac.

A refusal recorded in a manifest is a successful outcome, exactly as it was for athena's
`procs-residue` and `state-residue` runs. Silently downgrading a refusal to a warning is
not.

## Phases

### 1. mac-leg

**Reachability.** `mac` is offline unless it is powered on with its lid open. Probe with
`tailscale status` and a short-timeout SSH attempt. If it is down, wait with
`/sase_monitor` on a bounded polling command rather than substituting a Linux run. If
the window never opens, stop and report; that is a successful stop condition, not a
failure.

**Isolated build.** Once reachable, over SSH:

1. Clone the kit revision into a scratch directory on mac that is **not** a worktree of
   mac's live editable checkout and not inside any SASE runtime root.
2. Create a throwaway `uv` venv and build `sase_core_rs` from source into it at the
   pinned revision. Confirm the built extension exposes all nine `migration_*` bindings
   before running anything, using the repo's own `tools/check_sase_core_rs_bindings` /
   `tools/probe_core_floor`.
3. Confirm the chosen backup root (`$HOME/cutover-backups`, or an explicit
   `SASE_CUTOVER_BACKUP_DIR`) is **not** inside an iCloud-synced directory, and record
   the check. This is an explicit requirement of the parent plan's "Where the kit
   writes" section.

**Synthetic matrix on Darwin.** Run `tests/migration_kit/` plus
`tests/main/test_migrate_parser.py`, `tests/main/test_migrate_startup_isolation.py`, and
`tests/test_check_sase_core_rs_bindings_tool.py` inside the scratch venv and record the
result. The matrix is already written; this phase proves it holds on macOS, where path
semantics, `statvfs`, and signal timing differ from Linux. Record any case that must be
skipped on Darwin and why -- in particular the real-bounded-filesystem ENOSPC variant,
which uses an unprivileged Linux user+mount namespace and has no direct Darwin
equivalent; the injected-`OSError(ENOSPC)` writer-seam variant must still pass.

**Real-data rehearsal on protected copies.** Snapshot mac's import-leg and residue roots
into a scratch root outside every runtime root, then run the full `sase migrate` surface
against the copy only:

- `backup --apply` over the scratch copy: record member count, byte size, per-store
  `PRAGMA integrity_check` results, symlink count, and wall duration.
- `plan` / `run --apply` / `verify` for `import-purge`. This is mac's substantive leg:
  its import-leg backlog is roughly 9,894 artifacts, 6,501 chats, and 9,924 dismissed
  bundles, far larger than athena's. Record counts and durations.
- `plan` / `run --apply` for `procs-residue` and `state-residue`. mac's Tier B residue
  was clean at census time, so `already_done` classifications are the expected outcome;
  record them anyway, because the manifest must state a decided answer for every
  operation. If mac does carry legacy `tasks.jsonl` rows, check whether they hit the
  same rotated-out-canonical-sibling refusal athena hit -- that finding is recorded as a
  note on `sase-x7.6` and mac evidence either way is useful to it.
- `plan lock-residue` (read-only; it has no apply path). Record the classification of
  every lock present under mac's `~/.sase/locks/`.
- `restore` dry run and `--apply` against the scratch copy: verify every checksum,
  report the diff, prove the staged swap, and confirm the backup itself is untouched.

Delete the scratch copies and their rehearsal backups after measurement; keep the small
JSON evidence logs.

**mac's G3 probe.** Read-only, closing the gap `kit-backup` deferred: installed
distributions with exact versions and code directories, console-script entry points,
shell completion registrations, `launchd` user agents, cron and `at` entries, and every
scheduled or automatic updater that could deploy code during a maintenance window. This
defines mac's drain list.

**Acceptance:** the scratch venv builds and exposes all nine `migration_*` bindings; the
kit test lane passes on Darwin with every skip justified; the real-data rehearsal
completes end to end against copies only, with backup, purge, restore, and every refusal
recorded with counts and durations; mac's G3 probe is complete; no production data on
mac was mutated and the live install is untouched; the evidence is written somewhere
durable and its location is recorded in a note on this phase bead.

### 2. publish-evidence

Fold the mac leg into the evidence and freeze it.

**Rehearsal receipt.** Drop the DRAFT marker and record, in full: the kit revision and
checksum, baseline package and config revisions per host, every synthetic matrix case
with its outcome **on both Linux and macOS**, the measured backup and restore durations
and sizes per host, and the `lock-residue` classification of both athena locks --
`code-swap.lock` as `classify_only` and `code-swap-v2.lock` as
`refuse_archive_current_writer`, which is the definitive answer to census finding F3.
Keep athena's recorded refusals stated as successful outcomes, including the
`procs-residue` rotated-out-residue finding and the all-or-nothing `state-residue`
manifest gate.

**Per-host operation manifests** for athena, mac, and apollo. Each names the concrete
ordered operation list, roots, expected counts, backup and secondary-copy locations,
free space required, the estimated duration measured during rehearsal, the drain list
from the G3 inventory, and the verification query for each operation. Two requirements
are easy to miss:

- Every manifest must **name a secondary copy location**. The parent plan makes
  `-s/--secondary` optional for the engine but mandatory for these manifests.
- Every manifest must say **explicitly** that the backups taken during rehearsal prove
  the engine and measure cost and are **not** the cutover backups; `local-state-cutover`
  retakes them inside its own quiesced window.

apollo is not rehearsed -- its manifest is built from the census and the G3 probe, with
its operations marked rehearsed by proxy on athena, which shares its platform. Say so in
the manifest.

**Publish.** Publish all four as artifacts attached to the phase bead that owes them:

```bash
sase artifact create -p <file> -l "<label>" --bead sase-x7.2.1.4
```

If attaching to `sase-x7.2.1.4` is refused because that bead is closed, attach to this
epic's own plan bead instead and add
`sase artifact link add bead:<this epic's bead> related bead:sase-x7.2.1.4 "publishes the artifacts kit-rehearsal owed"`,
then say which route was taken in the phase note. Do not reopen `sase-x7.2.1.4` to work
around it.

**Acceptance:** four artifacts exist and resolve -- athena, mac, and apollo per-host
operation manifests plus the rehearsal receipt; each is attached to a bead and
discoverable with `sase artifact list -e`; the receipt carries no DRAFT marker and
covers both platforms; every manifest names a secondary copy location and disclaims the
rehearsal backups; `just check` passes if any repo file changed; and
`sase bead epic-symbols` is clean.

## Handoff

This epic is done when the four artifacts are published. Its land agent verifies them,
closes this epic, and then resumes the interrupted landing of `sase-x7.2.1` through the
`parent_bead` link -- which means finishing `sase-x7.2.1`'s own close, its symvision
pass, and its plan-file status update. Do not do any of that from inside a phase.
