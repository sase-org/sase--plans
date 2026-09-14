---
tier: epic
parent_bead: sase-zw.8
title: Finish the remaining disk-retention safety and integration gaps
goal: Retention refuses incomplete protection evidence, preserves protected paths
  and Git dependencies, and reports bounded owner results on the correct filesystem.
phases:
- id: scratch
  title: Share scratch liveness and report every cleanup outcome
  size: medium
  depends_on: []
  description: 'scratch: integrate launch-exit cleanup with the Rust scratch owner
    and expose ordinary-age bytes, failed removals and incomplete liveness checks.'
- id: procs
  title: Refuse proc cleanup when durable protection coverage is incomplete
  size: medium
  depends_on:
  - scratch
  description: 'procs: reject missing or malformed proc stores, preserve unreadable
    trees, and bound scanning while retaining reservation synchronization.'
- id: runs
  title: Preserve run protections through deletion and empty-shard cleanup
  size: medium
  depends_on:
  - procs
  description: 'runs: fix symlink-ancestor validation, protect empty referenced runs,
    refresh protection coverage safely, and expose deduplicated actionable previews.'
- id: objects
  title: Apply dependency-preserving repair rules to normal borrower reuse
  size: medium
  depends_on:
  - runs
  description: 'objects: prevent healthy checkout reuse from rewriting alternates
    without eligibility, source connectivity and rollback guarantees.'
- id: inventory
  title: Make inventory bounded and accurate about ownership and coverage
  size: medium
  depends_on:
  - objects
  description: 'inventory: move shared inventory classification to Rust, apply a whole-pass
    budget, resolve actual owned roots and configured horizons, and account for overlap
    and incomplete scans.'
- id: pressure
  title: Use owner filesystem observations and structured cleanup results
  size: medium
  depends_on:
  - inventory
  description: 'pressure: align doctor, housekeeping and manual cleanup thresholds
    per filesystem and propagate owner errors, skips and measured bytes to CLI results.'
- id: acceptance
  title: Prove the repaired combined tree and refresh host acceptance
  size: medium
  depends_on:
  - pressure
  description: 'acceptance: run installed binding and complete verification, add regression
    evidence for every landing reproduction, and finish bounded inventory and build-timing
    acceptance.'
proposed_by: bbugyi200.athena.sase-zw.8.land
create_time: 2026-09-14 16:48:08
status: wip
bead_id: sase-zw.8.7
---

- **PROMPT:** [prompts/202609/disk_retention_final_safety.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/disk_retention_final_safety.md)
- **PARENT:** [202609/disk_footprint_remaining_work.md](https://github.com/sase-org/sase--plans/blob/main/202609/disk_footprint_remaining_work.md)
- **BEAD:** [sase-zw.8.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zw/sase-zw.8.7.md)

# Finish disk-retention safety before landing sase-zw.8

This child repairs only unfinished requirements from
`plan:202609/disk_footprint_remaining_work.md`. Its parent is **sase-zw.8**, whose six
phases closed but whose landing audit reproduced unsafe deletion and incomplete
integration at main `00acd607f` and Rust core `3566872` / v0.34.28.

Read the current landing audit `file:explicit:1b136cd32d914b18447bed0a` through
`sase artifact read`, together with acceptance evidence
`file:explicit:a0a0691996ca110fe07614a4`. The audit contains exact reproductions, source
locations, commit review, every note disposition, and passing-test evidence. The older
parent audit is `file:explicit:b165eb0a7616cb5051633c9c`. Preserve work already proved
complete; do not repeat operational deletion or redesign the feature.

The audit snapshot was created successfully, but automatic bead-link attachment failed
because the hidden plans clone was dirty (existing task sase-10y). Preserve that clone;
use the explicit reference and the parent landing note instead of assuming derived links
were published.

Use `sase repo open gh:sase-org/sase-core` before accessing core and use its printed
path. Shared policy and ownership behavior belong in Rust and PyO3 bindings. Keep Python
as host observation, provider effects, and presentation adapters. Open any other needed
repo through the same skill. Honor each repo's instructions and maintain the published
dependency floor and schema validation when adding wire fields.

Phases are serial because they share the binding registration, configuration and
owner-result contracts. Keep each phase directly implementable, with focused tests for
its failures and the repository's required check. The final phase proves the combined
tree; green legacy tests alone do not establish the missing cases.

## scratch

Already complete: Python calls the Rust reaper; config horizons exist; muse-prompts
defaults to 3d; core dev-update has incremental=false; target/build directories remain
isolated. Keep those changes.

Remaining source gaps:

- Rust `managed_tmp.rs` discards ordinary-age removal sizes and suppresses removal
  errors. A real 512-byte aged entry previews selected=1 but disk reports zero bytes
  because the adapter uses pressure-only totals.
- Later main `59dde523c` added `axe/run_agent_runner_scratch.py`, a second Python policy
  that scans live processes and removes launch scratch at runner exit. It treats
  unreadable process details as no reference. Integrate this caller with the Rust
  scratch owner instead of maintaining two retention/liveness implementations.

Extend the owner contract for ordinary-age and pressure selections, successful bytes,
skips, failures and incomplete observations. Keep the distinction between preview
estimates and actual successful removal; deindex only removed artifact directories. Use
bounded scans that preserve candidates when freshness or liveness cannot be established.
The launch-exit mode must retain its exact assignment identity, handoff-outcome
suppression and protection for surviving child processes; it must never become an
arbitrary recursive-delete API. Preserve macOS's safe behavior when Linux process
evidence is unavailable.

Preserve core `5ea49f5`'s configurable `pressure_low_free_space_min_age_seconds`
(default 1h). A breached free-space floor shortens the base age for both free_space and
size_and_free_space; size-only keeps the base age. Retain its Python/Rust regression
cases. Keep later `tools/run_pytest` sibling cleanup as a separate test-owned lifecycle;
do not claim arbitrary /var/tmp directories. Add binding tests for age-only bytes,
failed deletion, incomplete tree reads, assigned/unassigned paths and live/handoff
protection.

## procs

Keep canonical proc IDs, the exclusive store lock, protection for every retained row,
pruned-row immediate cleanup, and hourly orphan sweeping outside append/reserve scans.

Fix the confirmed failure: `procs/runtime.rs` calls `read_rows_unlocked` and discards
its statistics. Missing stores are interpreted as empty; malformed rows are skipped.
Both a missing store and a store containing `not-json\n` delete an aged
`runtime/0123456789ab` with budget 1. Fail closed whenever a preexisting runtime tree
lacks a trustworthy complete store snapshot. An initialized, valid empty store must
remain distinguishable from missing or invalid data; do not weaken normal store reader
compatibility to implement retention's stronger requirement.

Preserve unreadable trees and metadata failures instead of treating them as empty or
old. Bound enumeration and descendant inspection across the sweep, including the
post-budget capped calculation, so holding the store lock cannot turn into an unbounded
recursive scan. Return explicit diagnostics and retryable partial results. Recheck root
containment and symlink components at mutation time under the existing synchronization
protocol.

Prove missing, malformed and mixed-validity stores, permission failures, fresh/invalid
IDs, symlink roots and ancestors, 4,000-directory convergence, and an actual concurrent
reserve/sweep race. Retain pruned-row/log/runtime parity. Historical rowless proc logs
are separately tracked by **sase-115**, proposed by sase-zw.8.2; do not broaden this
phase into that task or fleet payload task sase-104.

## runs

Fix three independent protection holes in the existing Rust run owner:

1. `path_safety_violation` canonicalizes before examining ancestors, erasing symlink
   evidence. A terminal run submitted through `projects/alias -> projects/demo` is
   actually removed. Validate the original path components and the canonical
   project/workflow/run relationship, then revalidate at mutation. Refuse symlinked
   roots, ancestors, leaves, invalid run paths and ambiguous coverage.
2. `prune_empty_tree` only knows watched paths. An empty run marked referenced_dir is
   counted protected and still deleted with its day/month during the empty pass. Feed it
   the same protected/live run coverage; preserve protected runs and their descendants
   and parents as necessary. Restrict traversal to eligible shard/run structure,
   preserve unknown directories, and make preview obey the apply budget.
3. Python refreshes protections once before scanning the whole corpus, while Rust
   rechecks only done/running/waiting/question markers. Use authoritative reference
   coverage and synchronization or generation validation so newly referenced runs,
   reopened beads/gates/plans and revived continuation ancestry cannot be deleted
   between collection and mutation. The apply API itself must refuse unavailable or
   incomplete protection sources; caller-supplied empty reason arrays are not proof.

Preserve recent-month protection, continuation retention limits, partial-source refusal,
bottom-up cleanup, future nonempty runs, and artifact-index deletion invalidation for
full-history consumers. Add focused Rust and real binding tests for each reproduction,
reference changes during apply, source failures, protected empty runs, revived
continuations, invalid paths and accurate removal budgets. Confirm retained chat and
prompt consumers still resolve while removed runs are deindexed.

Finish the approved hourly user surface: `sase_chop_artifact_run_prune.py` only logs a
preview. Add an actionable notification or existing command-backed gate with stable
deduplication for unchanged selections and protection problems. Preview remains the
default; no unattended artifact deletion. Use the existing gate/notification mechanisms
and preserve subsequent durable-answer/admission behavior. Update documentation and
configuration/schema only where behavior requires it.

## objects

Keep the Rust alternate ownership planner, preservation of foreign/relative entries,
explicit repair rollback, `gc.pruneExpire=never`, and disabled borrower maintenance.

Normal reuse remains unsafe: the successful git-status branch of `ensure_git_clone_at`
calls `ensure_sase_alternate` without the repair path's guards or connectivity proof. In
the audit a dirty healthy borrower was reused against a newly initialized primary
missing its objects: the call returned success, changed alternates and left git status
exiting 128. Fix every path that installs/repoints or dissociates an existing borrower,
not only `sase workspace repair`.

Put shared eligibility and mutation planning in Rust, with Python gathering claims,
occupants, status and performing provider effects under the project lock. Recheck
eligibility at mutation time and distinguish new materialization from existing checkout
mutation. Preserve claimed, occupied and dirty borrowers. Verify source connectivity and
borrower-required objects before abandoning a dependency; make configuration and
alternate rollback cover all failed postconditions, including normal reuse.
Missing-primary recovery must leave an irrecoverable borrower intact with an actionable
error; it must never reach recursive rematerialization. Preserve opt-out dissociation
after safe repoint and foreign alternates.

Use real-Git regression fixtures for the healthy-but-dirty reuse reproduction, source
missing required objects, moved/deleted primaries, unique local history, foreign and
relative alternates, claim/occupant changes, and failed rollback. Existing 22-case
workspace coverage is useful but omitted the reproduced successful-status branch. Do not
compact live workspaces for this phase's development.

## inventory

The current Python inventory has a fresh 5s/50,000-node fallback budget per root, eager
child lists, silently omitted owners on discovery failure, and no whole-pass budget. Its
target resolver returns the first core checkout and can omit a distinct shared primary
target. Horizon rows use shipped constants, so a configured 7d handoff still renders 3d.
Totals simply sum rows, including nested roots.

Complete the approved Rust inventory/classification contract using host-resolved owned
root observations. Enumerate actual configured managed roots, shared primary and recipe
target/build paths, honor profile/target overrides and configured retention horizons,
and avoid inferred target ownership. Reuse the same owner policy data the reapers
consume. Do not add a separate horizon table or fallback backend.

Apply one bounded pass across discovery, enumeration, measurement, subprocesses and
fallbacks. Avoid unbounded eager listings inside a nominally bounded loop. Return
partial/error diagnostics and unresolved owner coverage instead of silently dropping
rows. State what the stray depth limit excludes; a clipped scan cannot prove no unowned
content exists. Preserve never-follow-symlinks behavior.

Represent nested/overlapping roots without double-counting physical usage or falsely
equating logical sizes with reclaimable bytes. Keep per-owner rows useful while making
aggregate accounting explicit. Retain compatible JSON fields where practical and make
incomplete coverage prominent in CLI and doctor output. Test whole-pass exhaustion, wide
directories, failed workspace discovery, distinct primary/recipe roots, overrides,
overlaps and truncated/depth-limited scans through the real binding.

## pressure

Use the existing Rust pressure classifier as the single threshold policy. The chop
currently samples only sase_home and forwards those free bytes to managed_tmp, which may
be on another filesystem. Manual/ordinary cleanup also uses different configured floors
than doctor. Observe each actual owner filesystem, deduplicate observations by
filesystem identity, and forward the effective trigger/recovery policy for that owner.
Keep absolute and proportional floors, low-space age semantics and safe age/live-build
limits consistent for doctor, housekeeping and manual preview/apply.

Finish structured owner orchestration in Rust with thin host effect adapters:

- Carry scratch phase's ordinary-age/pressure counts, bytes, skips and errors.
- Include proc row-prune effects in the operation result, rather than silently
  discarding their runtime/log removals before a second orphan sweep.
- Carry artifact apply errors and blocked protection sources into the aggregate.
- Replace `workspace_compact_steps`' final-line and `compacted #` text parsing with
  structured compact/repair results from the owner interface. Bound subprocesses.
- Make CLI status fail for errors, blocked apply and nonzero owner exit codes, while
  distinguishing ordinary protective skips. The audit's artifact permission error
  currently exits 0; a proc step with exit_code=1 also exits 0 because `_exit_code` only
  examines mode.

Do not auto-delete artifact runs, backups or unowned strays under pressure. Test
separate filesystems, the original 875GiB/40GiB mismatch, small-volume absolute floors,
custom thresholds, both low-space triggers, each owner failure, partial success,
accurate selected/reclaimed bytes, and workspace output wording independence. Use real
shared owners in CLI/chop/doctor integration tests with only external effects isolated.

## acceptance

Refresh the supported installation and record main/core revisions, published versions
and each changed wire schema. Preserve later continuation, forced-reuse barrier, sudo,
gate admission and Git-identity harness changes. Incorporate any post-plan drift that
touches these owners; other active features are not permission to weaken retention.

Run core's prescribed complete check including binding tests (with the interpreter
library path if required), per-phase checks, and main **just check-full through
/sase_monitor with TESTING/TESTED** on the final combined tree. Fix epic-caused failures
before claiming completion. The prior .6 full run had 41665 passing bodies but failed
afterward; its post-fix focused success does not substitute for the full gate. Record
every final command outcome and regression case in a durable artifact.

Finish missing direct-profile and recipe evidence without inventing target roots: verify
nonincremental output, record no-op and one-crate-changed timings, and retain already
documented cold-build evidence or measure it safely if none exists. Do not delete shared
dependency output merely to produce a cold-build timing.

Remeasure host inventory with the repaired bounded coverage contract and doctor checks.
The prior report records root available 203,009,458,176 bytes, successful compaction of
22 borrowers, and preserved backups costing about 72.3 billion bytes, but an inventory
walk clipped at 50,000 nodes. Report partial coverage honestly and perform targeted
bounded follow-up measurements if needed to establish the approved no-unowned
row-above-1GiB outcome. Recheck retained chats/prompts and proc state parity.

Preserve prior backup declines and all live/recent/reference protections. Prior cleanup
gate approval is not new permission for new candidates. This phase normally needs
read-only remeasurement; if new artifact/backup/unowned deletion is necessary, prepare
exact candidates and obtain the explicit authorization required by artifact policy. Do
not repeat already completed fleet compaction without new evidence of eligibility and
need. Record why the free-space target is or is not met, including retained groups.

Every historical PROPOSED FOLLOW-UP has a disposition in the landing audit: orphan logs
are sase-115; the old generic check bundles are resolved by later fixes/focused tests;
the monitored full rerun is this acceptance work. Recheck new reports through the task
policy, preserving original proposing bead IDs.

The child plan's parent_bead link is the landing handoff. No phase closes sase-zw.8 or
sase-zw, runs their post-close Symvision, or changes their plan status. After this child
lands, its land agent resumes those directly parented plans, rechecks every
descendant/note and post-child drift, removes stale epic-symbol exemptions, and closes
normally only when complete. Never force a successful nested landing.
