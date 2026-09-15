---
tier: epic
title: Complete the remaining retention landing contracts
goal:
  Retention preserves changing protections and active borrowers, stays bounded, reports
  complete owner outcomes, and passes supported-build and host acceptance.
parent_bead: sase-zw.8.7
phases:
  - id: scratch_bounds
    title: Bound scratch observation before choosing removals
    description:
      "scratch_bounds: add shared work bounds to scratch traversal and launch liveness
      while preserving complete outcome reporting."
    size: medium
    depends_on: []
  - id: run_protection
    title: Revalidate authoritative run protections at mutation
    description:
      "run_protection: refuse absent protection proof and preserve references or
      continuation state that change during collection and deletion."
    size: medium
    depends_on:
      - scratch_bounds
  - id: borrower_eligibility
    title: Guard every existing borrower dependency mutation
    description:
      "borrower_eligibility: check claims and occupants under project synchronization
      for reuse, recovery, repoint and dissociation."
    size: medium
    depends_on:
      - run_protection
  - id: inventory_bounds
    title: Bound configured owner discovery as part of inventory
    description:
      "inventory_bounds: cover workspace discovery and target resolution with the shared
      deadline and report every unresolved owner."
    size: medium
    depends_on:
      - borrower_eligibility
  - id: cleanup_results
    title: Preserve every owner error and partial cleanup effect
    description:
      "cleanup_results: put aggregate result policy in Rust and propagate discovery
      failures, structured workspace errors and proc log effects."
    size: medium
    depends_on:
      - inventory_bounds
  - id: binding_compatibility
    title: Verify the supported core revision and wheel contracts
    description:
      "binding_compatibility: advance the CI core pin to the completed APIs and prove
      all required bindings and schema versions through supported installs."
    size: medium
    depends_on:
      - cleanup_results
  - id: measured_acceptance
    title: Complete host measurements and combined acceptance
    description:
      "measured_acceptance: prove the repaired safety cases, host inventory and build
      timing requirements, resolve unsupported baseline allowances, and pass both
      repository gates."
    size: medium
    depends_on:
      - binding_compatibility
proposed_by: bbugyi200.athena.sase-zw.8.7.land
create_time: 2026-09-15 18:33:17
status: wip
---

- **PROMPT:**
  [prompts/202609/retention_landing_contracts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/retention_landing_contracts.md)
- **PARENT:**
  [202609/disk_retention_final_safety.md](https://github.com/sase-org/sase--plans/blob/main/202609/disk_retention_final_safety.md)

# Complete the remaining retention landing contracts

This child contains only unfinished requirements of
`plan:202609/disk_retention_final_safety.md`. Its parent is **sase-zw.8.7**. The landing
audit found material gaps despite all seven phases being closed. Read the current audit
through `sase artifact read`: **file:explicit:b1aa712c5de680aad6436f99**. It records
exact isolated reproductions, source locations, commit review, every note disposition
and current passing tests. Snapshot creation succeeded but automatic bead attachment
failed on the known dirty hidden plans clone; use the explicit reference and parent
note, without cleaning or replacing that clone.

At audit time main was 449e1a8ec and core was a68ee7d. Open sase-core using
`sase repo open sase-core` and use its printed path. Open other repositories through the
same skill. Shared retention, eligibility, classification and aggregation policy belongs
in Rust and its PyO3 API; Python observes host state, performs provider effects, and
renders results. Honor repository-specific checks and managed release rules. The serial
dependencies are intentional: these phases share binding registration, tests and the
installed core.

Preserve already working behavior: proc missing/malformed-store refusal and exclusive
locking, 4,000-directory convergence, ordinary scratch byte/error reporting, launch
assignment and handoff guards, configurable low-space age, submitted-path symlink
checks, protected empty runs, deduplicated actionable hourly previews, foreign/relative
Git alternates and rollback, configured horizons, overlap accounting and
owner-filesystem pressure classification. No phase repeats fleet compaction or
operational deletion just to demonstrate progress. Do not turn remaining safety defects
into unrelated task beads.

## scratch_bounds

`managed_tmp.rs` now returns useful age/launch/pressure outcomes, but `tree_snapshot`
recursively traverses the entire candidate and `iter_children` eagerly collects all
entries. Python `run_agent_runner_scratch.py` iterates all PIDs and reads whole
environment files with no shared work bound. A removal count is not a bound on
observation or lock hold time.

Introduce finite node/depth/work limits shared across one scratch invocation, including
pressure discovery, candidate sizing, final freshness checks and launch-exit liveness
observation. Use incremental listings and bounded reads. Budget exhaustion, unreadable
process details, filesystem errors and unknown freshness must preserve affected
candidates and produce explicit incomplete results. Do not silently drop an unavailable
root or fabricate zero-age metadata. Keep host-specific process observation thin and the
deletion decision in Rust.

Preserve exact launch scratch key/bucket identity, surviving descendants, handoff
suppression and safe behavior where Linux process evidence is unavailable. Preserve both
low-space triggers and size-only base age, using the existing configurable
pressure_low_free_space_min_age_seconds contract.

Prove with deterministic fixtures that wide/deep trees and oversized or incomplete
process observations stop at their limits without deletion; assert selected, removed,
skipped, failed and incomplete counts/bytes through the real binding. Use small explicit
test budgets rather than expensive huge trees. Include age-only, pressure and
launch-exit callers and the existing reporting regressions.

## run_protection

Two current reproductions must fail before the repair and pass afterward:

1. A schema-2 binding request with one terminal old run and **no coverage fields**
   deletes it, because protection arrays and sources_unavailable all default empty.
2. Python apply reads protections once; candidate collection then adds a reference
   before Rust apply. The run is still deleted. Rust refreshes paths and active markers
   only; newly referenced runs, reopened work and continuation revival are not covered
   by that refresh.

Replace caller-asserted empty arrays as proof with an authoritative coverage and
mutation protocol. Implement the shared protocol in Rust over the relevant existing
owners. A compatible solution must either synchronize protection-source publication with
deletion or validate authoritative generations with a safe mutation barrier. An extra
unlocked scan, a caller-authored complete=true boolean, or generation checks that leave
an unguarded check/delete interval do not meet the requirement. If a source cannot
provide complete trustworthy coverage, preserve the candidate and report why.

Cover artifact references/consumption, open bead/plan/gate state, and continuation
ancestry/revival. Prove writer-versus-delete behavior with controlled barriers and real
owner operations, not only monkeypatched arrays. All apply entry points, including
direct binding requests, must reject omitted/stale/incomplete proof. Keep preview
best-effort and visibly partial. Keep caller policy (scope, horizon, limit) separate
from mutable eligibility. Update schema probes with any wire change.

Apply the same protections to empty-shard removal. Bound traversal and lookahead when a
removal/work budget is exhausted; retain unknown directories and unreadable paths.
Preserve original-path symlink checks at mutation and accurate preview/apply budgets.
Verify retained chat/prompt consumers and removed-run deindexing. Keep the notification
timestamp-awareness fix from 53035c967 and stable deduplication.

## borrower_eligibility

Dirty reuse and failed connectivity rollback work. Claims/occupants are absent from the
normal reuse contract. The audit created a healthy borrower and a replacement source
containing the same objects, started a live process with cwd in the borrower, and called
ensure_git_clone_at(..., share_git_objects=true). Alternates changed while the occupant
remained alive. This violates the approved eligibility requirement even though
connectivity survived.

Use the existing workspace/project synchronization and authoritative claim/occupant
observations for **every existing-checkout dependency mutation**: healthy reuse,
broken-borrower recovery, repair, repoint, compaction and opt-out dissociation.
Represent shared eligibility in Rust. Recheck at the mutation boundary so a claim or
occupant appearing after initial inspection prevents mutation. Distinguish new
materialization from existing reuse; a new clone's own reservation must not block
legitimate creation. Do not invent a second claim store or bypass the forced-reuse
barrier and existing caller lock ordering.

Preserve dirty/claimed/occupied borrowers, unique local objects, foreign and relative
alternates, source and borrower connectivity proof, and configuration/alternate rollback
on every failure. Preserve irrecoverable borrowers with actionable errors; never fall
through to recursive rematerialization when dependency recovery fails.

Add real-Git cases for the occupied healthy reproduction, claimed clean borrowers,
claim/occupant changes between observation and mutation, recovery/dissociation, unique
local history and failed rollback. Assert both alternates/config preservation and caller
outcome. Exercise provider entry points as well as the policy binding.

## inventory_bounds

Keep the Rust inventory classifier and configured-horizon fixes. The existing
InventoryScanBudget reaches only some listings and sizing; `workspace_rows` still calls
full collect_workspace_inventory with no deadline, even if max_nodes=0. Project,
registry, claim and occupancy discovery remains unbounded. Root discovery still relies
on environment hints and cwd guesses. With no discovered core checkout, an explicit
CARGO_TARGET_DIR produces zero target rows.

Carry one deadline/work budget across discovery, configured-root resolution,
enumeration, measurement, subprocesses and fallbacks. Avoid starting an unbounded owner
discovery call after budget exhaustion; use incremental bounded owner observations or a
safely bounded host operation that can actually stop. Return unresolved/partial owner
rows for failures or omitted work. Do not leave late rows marked complete with a zero
size solely because measurement was skipped.

Resolve authoritative configured primary and actual recipe target/build paths, including
distinct shared primary targets, invocation outside a source checkout, CARGO_TARGET_DIR,
CARGO_BUILD_BUILD_DIR and profile overrides. Relative target/build paths must be
interpreted in the corresponding recipe's cwd. Do not infer a directory is owned merely
from its spelling. Preserve no-follow behavior and overlap/physical versus logical
accounting. Keep explicit limits on stray-scan depth and excluded scope.

Tests must cover exhausted budget before workspace discovery, a slow/failing owner, wide
directories, primary versus recipe roots, override-only roots, partial row sizes, nested
roots and depth clipping through the real classifier. Demonstrate bounded latency/work
for ordinary CLI and doctor callers, not just injected fast size functions.

## cleanup_results

Complete the approved Rust-owned structured orchestration/result policy. Python
currently implements owner sequencing and result aggregation itself. Keep Python
effects, but make selection of owner operations and interpretation of owner outcomes
consistent for CLI, doctor/housekeeping and future frontends through a typed core API.

Fix these explicit remaining result holes:

- Workspace discovery exceptions currently become an empty project list and a successful
  `no workspace projects found` step. Preserve the failure and coverage.
- A workspace subprocess returning JSON errors=1 with process exit 0 currently produces
  DiskReapResult.failed=false and CLI exit 0. Honor structured owner errors, nonzero
  exits, malformed results and blocked apply independently of wording.
- ProcPruneOutcome reports runtime effects, but delete_proc_logs returns no counts or
  bytes, and a log exception prevents returning partial runtime effects. Account for
  row-pruned current/rotated log effects, runtime effects and later orphan sweep without
  double counting or erasing successful earlier mutations on a later error. Historical
  rowless log reconciliation is separate task sase-115 and stays out of scope.
- Scratch owner exceptions currently escape the group, and incomplete observations are
  not distinguished from ordinary protective skips in aggregate success. Isolate owner
  failures and report partial results and retryability consistently.

Preserve correct per-filesystem free bytes, effective thresholds and the configured
low-space age. Do not let pressure auto-delete artifact runs, backups or unknown strays.
Bound provider subprocesses and preserve structured compaction/repair output. Prove each
failure and mixed success through the Rust contract and real CLI/chop adapters,
including successful effects followed by a later exception.

## binding_compatibility

Main's sase-core-revision.txt is 3566872, older than all five current child core
commits. Both CI and master-gate build this exact pin. Local source builds hide the
skew: metadata says 0.34.35 while installed APIs include unreleased changes. Tag
v0.34.35 has run schema1, sharing schema1 and no disk inventory, whereas main requires
run2, sharing2 and inventory1 even before these repairs.

After preceding core changes are available in the sanctioned repository, advance the
main CI pin using the repository's supported pin workflow. Probe every changed
binding/schema and required safety behavior on a wheel built from that exact pin. Do not
validate only against whichever development extension is already installed. Update the
installed-binding validator for all new contracts, including rejection of missing
run-protection proof and the typed cleanup result contract.

Respect release-plz ownership of core versions and main's release-floor reconciler.
Determine the actual released core version containing these APIs and either verify the
supported published dependency floor or carry the exact required reconciliation through
the normal release path. Do not manually bump core versions or assert that a locally
built package with the same version proves the published wheel. Record exact revisions,
distribution origins, versions and schemas for both CI and supported installation paths;
an unresolved compatibility path is not acceptance complete.

Preserve intervening bead-routing and routine/job binding requirements while updating
the pin. Check all required bindings, not only retention's subset. Run core's complete
prescribed check, including PyO3, and the main compatibility tests. Long commands use
the SASE monitor workflow.

## measured_acceptance

The previous .7 evidence proves monitored check-full yvzyta519jke exited 0. It does not
contain required refreshed host inventory, direct-profile/recipe build timings,
nonincremental output proof or the final complete core check. Its code commit also added
fourteen broad flake-baseline exceptions under a now-closed phase. Complete these
missing requirements after the repairs; do not repeat completed host cleanup.

1. Run the isolated failures in the audit as meaningful regressions through installed
   APIs, plus real concurrency tests for reference publication/run deletion and proc
   reservation/orphan sweeping. A proc test that merely inserts a row before calling
   sweep is not the required concurrent race proof. Preserve 4,000-dir convergence,
   missing/malformed/mixed store refusal, protected empty shards, symlink guards and
   real-Git dependency tests. Discover current split test filenames.
2. Refresh the supported installation; record main/core revisions, package origins,
   versions and wire probes. Run core's full prescribed check including bindings and
   main **just check-full through sase_monitor with TESTING/TESTED**, after final
   changes. Preserve monitor/settlement, forced-reuse, sudo, gate admission, bead
   routing, routine/job config and public tui rename contracts.
3. Repair the fourteen-node blanket baseline introduced by 05be391f7. Read the audit's
   per-node dispositions and evidence. Known owners are sase-ni (the +1 reopened it),
   sase-10g, sase-10p and sase-10v. Claimed-show was fixed by 981004d5e. Git-identity
   failure/pass used different LD_LIBRARY_PATH environments. Do not infer flakiness
   merely from historical failure/current success, or write fixed-at without a verified
   fixing commit. Keep any justified allowance under its actual task owner; remove
   unsupported allowances or retire evidence with accurate provenance. New independent
   recurrences get PROPOSED FOLLOW-UP notes with exact source record, tree/environment
   and same-tree rerun evidence for the resumed land agent. Do not grow the baseline
   just to obtain a green gate.
4. Measure direct dev-update and actual recipe no-op and one-crate-changed build
   timings, verify incremental output does not regrow, and record separate target and
   build roots. Recover valid prior cold-build evidence or measure safely without
   deleting shared dependency output. Revert any temporary benchmark edit and verify no
   unintended tracked source changes remain.
5. Remeasure host inventory and doctor pressure on the corrected bounded contract.
   Report partial/unresolved coverage honestly. Use targeted bounded measurements to
   establish the approved no-unowned-row-above-1GiB criterion, or document the exact
   remaining owner/scope blocker. Recheck retained chats/prompts and proc state parity.
   Explain the free-space target with retained groups. Preserve earlier backup declines;
   earlier gate approval is not permission for new candidates. If new operational
   deletion is necessary, prepare exact candidates and obtain the explicit authorization
   required by artifact policy.
6. Save a durable acceptance artifact covering **each** requirement, exact commands,
   outcomes and limitations. A passing legacy test suite alone cannot close missing
   safety, compatibility or measurement work.

The child plan's parent_bead link is the handoff back to interrupted landing. No
implementation phase closes sase-zw.8.7 or an ancestor, runs their post-close Symvision,
or marks their linked plan files done. After this child lands, its land agent resumes
sase-zw.8.7, then directly parented plans sase-zw.8 and sase-zw only after rechecking
all descendant notes, linked-plan readiness and intervening drift. Retire epic-symbol
entries before normal closes. Never force a successful nested landing; record and stop
at any incomplete or ambiguous ancestor.
