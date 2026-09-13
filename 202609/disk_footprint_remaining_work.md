---
tier: epic
title: Finish disk retention safety and integrated footprint acceptance
goal:
  SASE disk owners enforce bounded, safe retention through Rust core, disk pressure
  reports and invokes the same policies, and the remaining host acceptance is measured.
parent_bead: sase-zw
phases:
  - id: scratch
    title: Adopt the Rust scratch owner and finish Cargo leak prevention
    size: medium
    depends_on: []
    description:
      "scratch: replace the duplicate temp reaper with the existing Rust API, complete
      configurable horizons and the non-incremental core profile, and preserve isolated
      Cargo build directories."
  - id: procs
    title: Make proc runtime retention bounded and safe against concurrent launches
    size: medium
    depends_on:
      - scratch
    description:
      "procs: implement validated, age-bounded runtime retention in Rust, protect
      concurrent reservations, and run orphan cleanup on the hourly lane."
  - id: runs
    title: Complete protected run retention and empty-shard cleanup
    size: medium
    depends_on:
      - procs
    description:
      "runs: move shared run-retention decisions into Rust, revalidate protection before
      apply, remove eligible empty descendants safely, and surface actionable previews."
  - id: objects
    title: Preserve shared-object dependencies throughout repair and reuse
    size: medium
    depends_on:
      - runs
    description:
      "objects: complete Rust-owned sharing and repair safety, retain foreign
      alternates, and prevent failed borrower recovery from deleting local work."
  - id: pressure
    title: Unify disk inventory, pressure decisions and owner delegation
    size: medium
    depends_on:
      - scratch
      - procs
      - runs
      - objects
    description:
      "pressure: make disk inventory bounded and complete about partial scans, share
      pressure thresholds across surfaces, and return truthful preview/apply outcomes
      from owner APIs."
  - id: acceptance
    title: Complete host reclamation and combined verification evidence
    size: medium
    depends_on:
      - pressure
    description:
      "acceptance: verify the installed cohort, account for the prior cleanup decision,
      measure remaining owned reclamation and safe workspace compaction, and publish
      full acceptance evidence."
proposed_by: bbugyi200.athena.sase-zw.land
create_time: 2026-09-13 18:40:34
status: wip
---

- **PROMPT:**
  [prompts/202609/disk_footprint_remaining_work.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/disk_footprint_remaining_work.md)
- **PARENT:**
  [202609/bound_sase_disk_footprint.md](https://github.com/sase-org/sase--plans/blob/main/202609/bound_sase_disk_footprint.md)

# Finish the remaining work for sase-zw

This is the remaining-work child of `sase-zw`, not a redesign of the entire disk
feature. The land audit at main `8f7dad695b` reviewed every phase and note, the epic's
six production commits, its linked plan, and subsequent core/provider/main changes. All
phases are closed, but the source and isolated reproductions below show unfinished
acceptance. Preserve the working CLI, config, retention protections, shared-clone
mechanism, and subsequent continuation/query work.

Read audit `file:explicit:b165eb0a7616cb5051633c9c`, plus
`plan:202609/bound_sase_disk_footprint.md` and
`plan:202609/shared_workspace_git_objects.md`, through `sase artifact read`. The audit
records every proposed follow-up disposition, the read-only inventory, and which
reported verification failures came from a stale installed extension. Artifact creation
succeeded but automatic link publication encountered a dirty hidden plans clone.
Preserve that clone; use the explicit audited reference and the parent bead's landing
note rather than assuming every derived link exists.

Open core using `sase repo open gh:sase-org/sase-core`; use the printed path. Open the
GitHub provider with the same sanctioned workflow only if needed. At audit time
configured linked slugs failed with existing task `sase-zo`, while external opens
succeeded. Do not guess a sibling checkout path. Shared backend policy belongs in
`sase-core/crates/sase_core` and its PyO3 bindings; Python should retain presentation,
provider integration, and thin adapters. Do not add a Python fallback or a new backend
switch.

The serial dependencies intentionally avoid concurrent edits to the shared Rust binding
registration, pin and configuration while these owners are corrected.

## scratch

Confirmed gaps:

- `src/sase/core/managed_tmp_reaper.py` duplicates the Rust reaper added by core
  `a64c40d` / main `70b018b91a` (sase-zn.9.2); its public function never calls the
  existing `sase_core_rs.reap_managed_tmpdir` binding.
- `muse-prompts` uses the 12h command horizon rather than the parent plan's 3d handoff
  horizon. Bucket ages/pressure inputs are not configurable through the shipped
  configuration as the parent plan requires.
- Core Cargo.toml still has `[profile.dev-update] incremental = true`. The Justfile
  override does not fix a direct build with that profile.
- Later main `eea8af0421` sets `CARGO_BUILD_BUILD_DIR=<target>/build`. Cleanup and
  inventory still assume the older profile/incremental layout.

Replace the Python implementation with a schema-checked call to the existing Rust wire,
retaining the Python result interface and index-deletion integration. Reuse the Rust
age/pressure selection and fresh-descendant checks; preserve unknown stable buckets,
symlink exclusion, generic handoff protection, bounded removal, low-free-space pressure,
and the later macOS fix. Configure horizons through the existing config pattern and use
3d for muse handoffs. Avoid a second pressure policy in Python.

Set `incremental = false` in core's dev-update profile. Preserve recipe and launch
overrides, explicit caller override semantics, managed target/build-dir isolation, and
the bounded rust-prebuild cache. Determine Cargo's actual intermediate layout with
`CARGO_BUILD_BUILD_DIR`; clean only owned incremental output, never deps or a whole
shared target. Do not invent target roots.

Add real binding coverage for the existing reaper cases and configuration overrides.
Verify direct profile and recipe builds do not produce incremental cache, record no-op
and one-crate-changed build timings, and update the Rust backend docs/config/schema.
Coordinate pin changes with current master; retain schema-30/wire-9 and continuation
support, including later retention-cap fixes.

## procs

`procs/runtime.py::sweep_orphan_proc_runtime_dirs` removes all rowless directories
without checking their name or age and without a budget. Audit reproduction: 4,000 fresh
`not-a-proc-*` directories left zero survivors. It runs in `append_proc`,
`reserve_proc`, and `prune_procs` after the core snapshot has already been returned.
Another reservation can appear between that snapshot and deletion. Disk apply takes an
even older snapshot and sweeps again.

Implement the runtime owner in Rust alongside the proc store, with explicit
preview/apply results. Validate the canonical proc-ID alphabet and length, direct-child
containment, and non-symlink roots/entries. Protect every active or newly reserved proc
under the store's synchronization protocol. Revalidate eligibility before removal;
unavailable store/liveness information must preserve data. Delete only runtime state of
terminal rows actually pruned. Use an age horizon and a per-pass budget for historical
orphans, tolerate filesystem races, and report removed/skipped/errors and actual
reclaimed bytes.

Keep immediate cleanup for pruned rows but remove whole orphan scans from interactive
append/reserve paths. Wire a bounded ordinary sweep onto hourly housekeeping even when
free space is ample, with documented config/schema and chop output. Inspect orphan logs
and record their disposition; do not broaden into separate fleet-payload minimization
task `sase-104`.

Prove 4,000-directory convergence, fresh/invalid/symlink preservation, active rows,
reserve-versus-sweep races, failed store reads, and row/log/runtime retention parity
through Rust and Python callers. Disk preview must use this owner instead of recounting
candidates with a separate Python policy.

## runs

Preserve the existing recent-month, referenced-dir/name/timestamp, open-bead,
terminal-marker, and continuation ancestry protections. Later main `a6f6ae5c66` and core
`23f19f0` introduced portable continuation locators and retention; `52c80c9528` handles
unavailable continuation plans and core `3fa0a54` separates the retention corpus cap
from MAX_NODES. Use these contracts, including their failure behavior, while moving
shared planning/protection decisions to Rust.

The apply API itself must refuse missing protection sources and revalidate current
artifact, bead, plan, gate and continuation references before deletion. Do not rely only
on CLI checks or the original preview snapshot. Validate canonical project-root
containment and symlink ancestors at mutation time. Retain data whenever ownership or
protection coverage cannot be established.

Fix empty-shard application: an empty `202704/01/20270401000000` currently produces two
rmdir errors and survives. Reuse the shared shard-range definition, walk eligible empty
descendants bottom-up with no symlink traversal, and preserve any newly nonempty or live
run. Do not remove nonempty future runs. Use a removal budget and surface partial
failures as unsuccessful outcomes.

Hourly retention currently only logs a preview. Add the parent plan's actionable
notification or command-backed gate through existing mechanisms, with stable
deduplication so unchanged candidates do not spam the user. Preview remains default;
artifact deletion requires explicit apply authorization. Keep this separate from
unattended pressure cleanup.

Test protection changes after preview, source failure during apply, live markers,
continuation revival, nested empty directories, invalid roots, and real artifact index
invalidation. Verify full-history/index consumers from sase-zu observe deindexing and
retained chat/prompt paths still resolve. The unrelated future timestamp
creator/prevention proposal was corroborated on existing `sase-v4`; do not file it
again.

## objects

Keep persistent alternates, `git clone --shared`, `gc.pruneExpire=never` on the source,
disabled borrower auto-maintenance, and local-only compaction. The GitHub provider
already uses `ensure_workspace_checkout`; do not convert external/SDD clones with
distinct lifetimes indiscriminately.

Complete the shared lifecycle/ownership policy in Rust, leaving provider calls and CLI
presentation thin. `handle_compact` already checks claims, occupants and dirty trees
under the project lock. Apply equivalent eligibility and locked rechecks to alternate
rewrite/dissociation, including missing-primary cases. The current repair path ignores
these guards, and disabling sharing with an already broken alternate attempts repack
before the planned repoint.

Preserve every alternate not owned by SASE. Git-relative alternate paths are relative to
the object database, not its info directory. Do not treat a file containing the primary
among other entries as wholly SASE-owned. Prove source connectivity before replacing an
old dependency and restore consistent config and alternate state on failure. Recover a
missing/moved source by repointing or dissociating only when all required objects can be
preserved; otherwise leave the borrower intact with an actionable failure.

Integrate recovery with `ensure_git_clone_at`: a failed `git status` on a borrower must
not take the existing recursive-delete/rematerialize path and discard unique local
objects or dirty files. Respect the sharing config for normal materialization and
document explicit compaction/opt-out behavior.

Use real Git fixtures for multiple/relative alternates, moved sources, source deletion,
locally unique objects, failed repair rollback, dirty/claimed/ occupied checkouts, and
dissociation after repoint. Keep origin rewrite/fetch, post-repack fsck, and
disabled-sharing tests. Do not manually compact active workspaces while developing these
fixes.

## pressure

Move shared inventory classification, pressure decisions, and owner orchestration to
core APIs rather than growing `core/disk_footprint.py`'s independent policy. CLI tables
and notifications remain Python presentation.

Use one pressure contract for doctor, housekeeping and owner invocation, with both
absolute floors and percentages on the actual owning filesystem. Audit reproduction: on
an 875 GiB volume with 40 GiB free, doctor returns WARN but managed-temp pressure does
not trigger because its independent floor is 32 GiB. Small filesystems can also hit
doctor's 3 GiB floor while the percent-only chop returns space_ok. Forward the effective
trigger/recovery policy to the owner; preserve minimum age and live-build protections.
Verify separate filesystems and threshold overrides.

Bound the complete inventory pass, including subprocesses and fallback walks. The
audit's live list took minutes; du's 5s timeout falls back to an unbounded recursive
walk. Return partial/error diagnostics rather than hanging or silently dropping an
owner. Enumerate the actual shared primary and recipe core targets, their
build-dir/intermediate layouts, and managed workspace roots. Report overlaps accurately;
do not sum overlapping roots as independently reclaimable physical bytes. Show
incomplete stray coverage prominently; a truncated scan cannot prove absence of unowned
paths. Keep never-follow-symlinks behavior.

Make preview/apply consume owners' structured results. Current managed-temp preview is
just a command description; proc preview ignores eligibility and apply sweeps from stale
IDs; artifact errors may still return success; workspace results are scraped from text.
Return actual selections, bounded counts/bytes, skips and errors consistently, with
nonzero CLI status for failed apply. Do not auto-delete artifacts, backups or unowned
strays in housekeeping.

Retain bare-list delegation, JSON shape compatibility where practical, and document any
necessary wire changes. Update default config, schema, help, completion and docs
together. Add integrated CLI/chop/doctor tests that exercise the real shared owners with
only external effects isolated.

## acceptance

First refresh the workspace with the supported install. At audit time its extension
reported scan wire 8/index 27 and lacked continuation_plan_retention, despite pin
7f43a996 containing the source. Do not mistake stale-wheel failures for product flakes.
Run core's prescribed complete check including bindings, and SASE `just check-full`
through `/sase_monitor` on the combined tree. Repair any failure caused by this work;
use the task policy for unrelated failures. Preserve the published-cohort work tracked
by `sase-10d` rather than silently claiming an old permitted wheel satisfies the new
API.

Recover the original cleanup outcome before proposing any deletion. Gate
`custom-68f1bf05-931f-4b34-b58e-036db7d1c9aa` selected only `remove_leaked_state`: 821
runtime directories, the August ace-run shard, and managed build-targets were removed.
Backups were not selected. `.stignore` already has the required exclusions. Never replay
old broad deletion commands or treat that response as authority to delete newly created
content.

Remeasure the parent's inventory and doctor health with the repaired owners. Use
existing explicit authorization for still-applicable operations; if new
artifact/backup/unowned deletion is necessary, prepare exact safe candidates and
commands before presenting the required gate. Preserve backups not selected and state
guards. Use owned policies for authorized cleanup, retaining active, recent, referenced
and in-flight state. Retrofit eligible quiescent managed checkouts, recording every skip
and before/after connectivity/bytes. The original phase tested only one checkout; do not
equate that with fleet convergence.

At audit time root available space was about 85.6 GiB; managed cargo-targets were about
136.4 GiB and build-targets 34.6 GiB. Distinguish newly active build output, safely
reclaimable stale bytes, and deliberately retained backups. Record an honest
before/after report, the declined groups' measured cost, and any remaining bounded
active usage. Verify no unowned row above 1 GiB remains without an explicit disposition
and complete enough scan evidence to support it. Do not manufacture the 250 GiB
acceptance target by deleting protected data.

Publish the audit/measurement artifact, passing verification results, install and
retained-run smoke results, and all follow-up dispositions on this phase. The parent's
audit already corroborates sase-106, sase-v4, sase-xk, sase-zo, and sase-10d; it
forwards the historical tiering report to sase-zu. The monitor alias bug is fixed by
638647694b; stale-refresh/suffix-merge failures are fixed by ef254fd6dc. The historical
installed research_swarm runners=0 failure did not reproduce in current focused tests;
verify the installed cohort and report fresh evidence rather than reopening unrelated
name-registry task sase-qs.

The child land agent resumes the parent through `parent_bead: sase-zw` after this work
lands. Closing sase-zw, post-close Symvision and marking its original plan done are
landing actions, not phases in this child plan.
