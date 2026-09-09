---
tier: epic
status: done
title: Make epic launches and relaunches scale with the requested work
goal:
  Launching or relaunching one or several epics spends seconds on local preparation
  instead of minutes rescanning agent history, while preserving active workers, exact
  ownership checks, publication barriers, and partial-launch recovery.
phases:
  - id: measure
    title: Establish launch cost and safety baselines
    depends_on: []
    size: small
    description:
      "measure: add nested launch timing and an isolated history-scale benchmark that
      separates discovery, cleanup, reservation, publication, spawning, and admission."
  - id: core-batch
    title: Implement shared batch ownership and cleanup planning
    depends_on:
      - measure
    size: medium
    description:
      "core-batch: add Rust contracts for coherent ownership snapshots, batch cleanup
      closure, expected-owner guards, and bulk reservation decisions with PyO3 coverage."
  - id: registry-batch
    title: Make reservation transactions reuse one fresh view
    depends_on:
      - core-batch
    size: medium
    description:
      "registry-batch: add guarded batch registry operations, short mutation locks, and
      exact planned-owner claims that avoid archive-wide freshness proofs per runner."
  - id: cleanup-batch
    title: Discover and clean replacement owners as one batch
    depends_on:
      - registry-batch
    size: medium
    description:
      "cleanup-batch: replace per-owner archive scans and rebuilds with one catalog and
      deduplicated cleanup plan, preserving fresh destructive checks and recovery."
  - id: launch-batch
    title: Carry bulk reservations through epic fan-out
    depends_on:
      - cleanup-batch
    size: medium
    description:
      "launch-batch: reserve deterministic phase and land names together, consume those
      reservations during spawn, and reuse safe context across ordered CLI targets."
  - id: acceptance
    title: Prove speed, concurrency safety, and recovery
    depends_on:
      - launch-batch
    size: medium
    description:
      "acceptance: verify fresh and repeated launches at real history scale, exercise
      races and failure recovery, document measured gains, and complete landing checks."
proposed_by: bbugyi200.athena.0h7
bead_id: sase-xr
create_time: 2026-09-09 19:52:21
---

- **PROMPT:**
  [prompts/202609/fast_epic_launches.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fast_epic_launches.md)
- **BEAD:**
  [sase-xr](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xr/README.md)

# Plan: Make epic launches and relaunches scale with the requested work

## Evidence and scope

The owner requested this investigation while `sase bead work xe xf xq x7 -Y` was running
on athena on 2026-09-06. No implementation changes or interventions in that command were
made during planning. The following are observations of that run, not benchmark
predictions.

The process was PID 4072584, starting at approximately 22:26:28 UTC. An early `ps`
sample showed 369 CPU seconds after 386 elapsed seconds. Live `py-spy dump` samples then
showed both archive scanning and contention on `agent_name_allocation.lock`. `lslocks`
independently showed another process waiting for that lock while this launcher owned it.
Later samples progressed into subsequent targets, so the evidence supports expensive
repeated work and contention, not a permanent deadlock.

Completed per-target summaries in `~/.sase/logs/tui_launch_timing.jsonl` record:

| Target    |    Total | Forced-reuse cleanup | Agent launch | Publication, inclusive | Prompt render |
| --------- | -------: | -------------------: | -----------: | ---------------------: | ------------: |
| `sase-xe` | 486.08 s |             325.39 s |      88.32 s |                20.94 s |        6.25 s |
| `sase-xf` | 160.54 s |              81.42 s |      24.59 s |                21.15 s |       12.40 s |
| `sase-xq` | 133.91 s |              61.86 s |      30.26 s |                13.76 s |       12.20 s |

The first summary has `ts=1788733588.888199`; the second has `ts=1788734074.9650733`.
All three report `outcome=ok`. Publication contains commit and push; the respective push
stages were 20.08 s, 20.01 s, and 12.88 s. Do not sum nested stages. The first three
targets alone consumed 780.53 seconds (13 minutes). The timer does not currently isolate
initial cleanup preview or repeated bead reads, so there is additional unattributed
time. These numbers measure CLI orchestration; they do not measure provider execution or
time for queued workers to acquire capacity.

A contemporaneous read of `~/.sase/agent_name_registry.json` found 14,080 entries,
18,036,418 bytes, and a source signature covering 40,701 paths. These counts can change.
The inspected Python revision was `fdfb4e238a386b5470a67025da8db1c30bc92e90`; stack
function names and the durable timings corroborate the live path. The inspected core
revision was `0ce37bfdaa0b1b6460a66c83fe779b260776c1a6`. The benchmark must record the
installed Python and core revisions independently.

Relevant findings in the current code:

1. `bead/cli_work_entry.py::handle_bead_work` completes targets sequentially under the
   code-swap reader lock. This is a documented ordering and partial-failure contract,
   not an accidental loop to replace with threads.
2. `cli_work_cleanup_selection.py::select_bead_work_launch` loads a full owner view,
   then calls `lookup_registered_name` per slot. `cli_work_cleanup_targets.py` builds
   that view with `scan_agent_artifacts`; it is a full scan despite the view's
   indexed-in-memory structure.
3. `cli_work_cleanup_apply.py` first revalidates the selection, then verifies each
   destructive target through another selection. Family-member verification can repeat
   the entire selection. Each selection can scan global history again.
4. `agent/names/_wipe.py::_build_wipe_plan` scans all artifacts and dismissed bundles
   for each owner, and repeatedly walks those records to expand its closure.
   `wipe_agent_name_for_reuse` rebuilds the complete name registry after each wipe.
   `_forced_reuse.py` adds member-by-member wipes and additional final rebuilds for
   family and stale-container cases. Dismissed-index synchronization also repeats.
5. `_registry.py::_load_registry_for_reservations` clears scan caches and bypasses the
   display freshness memo on every reservation read. `_registry_store.py` fingerprints
   all sources and checks owner existence. Rebuilds parse historical metadata and
   bundles while holding the global allocation lock.
6. `launch_validation.py` separately reads reserved agent, clan, and family names.
   `multi_prompt_launch_execution.py` validates the batch, then `launch_executor.py`
   validates individual segment plans again. New artifact directories invalidate source
   signatures; post-spawn clan claims and child name claims add contention.
7. `_epic_bead_assignees` calls `proj.show` once per bead on repeated passes. This is
   secondary to archive work, but should become a coherent batched read where the core
   already provides one or can expose a thin batch query.

Audited design context:

- `plan:202608/safe_bead_work_relaunch_2.md` defines the active-worker preservation,
  expected-bead checks, selection shrinking, and partial-retry contracts to retain.
- `plan:202608/async_sidecar_publication.md` already addressed repeated registry reads
  in publication/association rendering. Its parent `sase-ej` is closed. Keep those
  improvements; this epic addresses allocation freshness and destructive cleanup, where
  stale display snapshots are insufficient. Do not reopen or duplicate that publication
  project.
- Active `sase-xe` also changes agent listing and identity. Coordinate through the
  current public core APIs; do not redo its Focus/Fleet implementation. `sase-xq` owns
  bead projection determinism, and `sase-x7` owns canonical-state migration.

This is an epic because the change needs independently verifiable Rust contracts,
registry transactions, destructive host integration, and launch integration. The six
phases are direct implementation work with explicit sequential dependencies. No new
model override, feature switch, daemon, or user-facing CLI option is needed.

## Chosen approach and constraints

Remove repeated whole-history work before considering concurrency. For H historical
sources, K selected owners, and R related records, preparation should perform a constant
number of O(H) discovery/proof passes per target plus O(K + R) indexed lookups and graph
traversal, rather than O(K x H) scans and rebuilds. Warm lookup and runner claim paths
should inspect the reserved owner and relevant markers, without enumerating history.

Start with a single operation-scoped catalog and reverse maps over existing source
records. Keep the current persisted registry representation and its explicit repair
path; do not make a new database migration a prerequisite for the main speedup. Reuse
existing artifact/dismissed indexes where their completeness is proven, but never treat
a best-effort projection as proof that a destructive closure is empty. The current core
`query_related_agent_artifact_dirs` is scoped to project/workflow, returns no bundle
closure, and has traversal caps; it cannot replace `_build_wipe_plan` without additional
completeness semantics. A bounded full discovery remains valid for a cold or untrusted
catalog, shared across the batch.

Shared identity, closure, guard comparisons, and reservation decisions belong in
`sase-core/crates/sase_core` with bindings in `crates/sase_core_py`. Python remains
responsible for process/filesystem effects, existing lock adapters, prompts, and
presentation. Open the core through `/sase_repo` and use its returned checkout path;
never assume an adjacent or particular numbered workspace exists. Release the core
through its normal workflow before requiring its new binding in SASE; do not edit
release-owned version numbers manually or add a Python fallback.

Do not substitute a long TTL or a launch-wide `name_registry_load_session()` for fresh
collision checks. Do not reuse snapshots across user input, mutations, or process
boundaries without an explicit ownership/freshness proof. Keep display reads separate
from allocation decisions. Existing reservation tests intentionally exercise newly
created sources even when directory stat information appears stale.

Do not increase runner capacity or start all four epics in parallel to hide CPU work. Do
not purge history. Keep the required graph checkpoint/publication barrier before
spawning, including detached-store `--no-push` refusal. The measured 20-second
publication stages deserve attribution, but do not authorize moving required publication
to a background queue or changing synchronization semantics in this epic.

## measure: Establish the baseline and expose hidden stages

Extend `agent/launch_timing.py` and its existing consumers, rather than creating a
second telemetry system. Add one correlation ID per command, target index/count,
resolved epic ID, selected/preserved/cleanup owner counts, and child segment identity.
Keep prompt contents out of timing records.

Time initial selection separately, and nest owner discovery, registry source proof,
registry rebuild/read/write, lock wait/hold, closure planning, target revalidation,
process cleanup, index maintenance, bead-assignee reads, and reservation operations.
Count full scans, parsed source files, rebuilds, registry writes, and relevant-row
reads. Make parent versus child durations explicit. Persist useful stage completion
records while the command runs; currently the durable summary only arrives at exit. Add
throttled progress to stderr for human CLI use, with the current target/stage and
completed/total owners. Preserve JSON/JSONL stdout byte shape. Reuse the existing
slow-stage threshold; do not add a progress thread that performs extra scans.

Build an isolated benchmark using generated state in a temporary `SASE_HOME`, a local
bare Git remote, and lightweight test runner processes instead of providers. Use the
real orchestrator and cleanup/registry code. Keep fixtures deterministic, portable, and
independent of live PIDs, production workspace claims, or credentials. Seed sources
directly in linear time, rather than calling the old O(H) persistence helper thousands
of times during setup. Exclude fixture construction from timing.

Cover fresh epic, all-active no-op retry, terminal retry, WAITING retry, mixed family
retry, and an ordered four-target batch. Include 1/12/40 selected slots against
1k/10k/about 40k historical sources, bundle-only owners, and cold/warm registry cases.
Record Python/core revisions, history mix, CPU time, wall time, filesystem work, lock
costs, time to first spawn, and time until all requested runners are registered. Record
capacity admission separately so a quick CLI exit cannot hide delayed startup. Small
structural fixtures belong in ordinary tests; full-scale timings are an explicit
benchmark/monitor lane. Establish the before result before changing behavior.

Acceptance: a single run explains where its time went, instrumentation does not rescan
state, JSON output is unchanged, and benchmark startup cannot launch real work.

## core-batch: Coherent ownership and cleanup contracts

Add focused wire types and pure operations alongside core `agent_cleanup`,
`agent_identity`, and `agent_launch`. Avoid expanding the existing large modules
unnecessarily. The batch request includes logical slots, expected bead/assignee, owner
identity, family/clan generation, marker state, process identity, and source record
identities/signatures. Return selected/preserved/blocked owners, cleanup closures,
per-target reasons, and the exact expected-owner predicates for effects.

Build reverse maps from canonical names, artifact paths, source suffixes, and
relationship pointers. Compute each requested closure with a work queue and visited set;
share indexes across roots and deduplicate overlapping effects while retaining
per-target attribution. Preserve the existing wipe's traversal direction and source
coverage: outgoing retry pointers, incoming parent/retry relations, workflow child
records, and bundle-only records must not disappear. Dotted name prefix alone is not
destructive ownership. Return an explicit incomplete/error outcome if discovery or any
traversal bound cannot prove the selected closure; never report a truncated closure as
safe to wipe. Revalidate source identities before execution.

Provide batch name decision operations over one reservation snapshot, including
same-name duplicates, clan/family containers, current-owner aliases, and existing
planned-owner checks. Keep namespace rules identical. Express expected-generation
comparison and merge decisions in Rust so host callers cannot overwrite claims published
after the snapshot. Include the minimum cleanup-in-progress ownership state needed by
the next phase; it is internal reservation state, not a new bead status or replacement
for agent identity.

Use fixture parity against the existing behavior for valid cases and adversarial
fixtures for collisions, cycles, overlapping roots, shared timestamps across projects,
missing metadata, conflicting family generations, and scope expansion. Provide PyO3
round-trip and public facade tests, not Rust-only tests. Expose a batch bead assignee
read if needed without replicating bead interpretation in Python.

Acceptance: closure and selection policy has one core implementation; Python gets
explicit effect inputs and completeness/guard outcomes through thin adapters.

## registry-batch: Fresh bulk operations and short locks

Add a public reservation-snapshot/bulk mutation API to `agent/names`, backed by the core
decisions above. A snapshot owns one fresh source proof, parsed registry, identity
context, all agent/clan/family views, and source/registry version evidence. Slot lookups
are in-memory. Do not reset enumeration caches for each view in that same proven
snapshot. Keep standalone reservation entry points strict outside this bounded
operation.

Bulk validation and reservation must be atomic with respect to other claimers: collect
expensive source data outside the allocation lock, then acquire the lock, compare the
registry version and relevant source evidence, and merge only guarded deltas into the
latest registry. Retry boundedly on change; fail with an actionable retry rather than
spinning or silently accepting stale ownership. Never write an old whole-registry
snapshot over a concurrent claim. A source change may require a new discovery pass, but
one new unrelated run must not trigger one pass per slot. Preserve fresh detection of
add/remove/rename/bundle rewrite and missing owners. A cheap directory mtime alone is
not sufficient proof under the existing contract.

Use generation-qualified cleanup reservations for the exact owners a cleanup batch will
replace. Persist an operation token and launcher process identity under the allocation
lock; other allocators/reusers cannot steal those names while cleanup runs outside it.
Define and test recovery when the launcher dies: compare process identity, re-read the
original owner/markers, and reconcile what actually remains before releasing or
resuming. Never expire a live launcher solely on elapsed time. Keep reservation states
outside the bead status lifecycle. Inventory all relevant writers, including direct
force reuse, dismissal, and runner publication, so none can accidentally erase this
guard in a rebuild. Exercise these states through the supported code-swap/update
lifecycle; do not leave older registry writers running against a reservation
representation they can discard or misinterpret.

Add an exact planned-owner-to-claimed fast path: a runner may claim its already reserved
name only when artifact path, generation/identity, and reservation token match. Read the
latest registry and targeted metadata, then update that owner; there is no need to
rediscover unrelated free names for this conversion. A missing or conflicting
reservation fails or takes the strict existing validation path, never assumes ownership.
Likewise make a repeated claim of an already established clan an inexpensive no-op after
exact generation/owner checks.

Batch writes and incrementally remove/update records affected by known effects. Reuse
unchanged historical entries and signatures only when justified. Keep full rebuild as
explicit recovery or a bounded cold path, never mandatory after each member. Global
allocation locks cover proof/merge/write only; no network, subprocess wait, full
metadata/bundle parse, or recursive artifact deletion may occur under them. Use a
documented lock ordering for cleanup guards, allocation, and index/store locks.

Acceptance: multiprocess tests prove no duplicate names, lost claims, stale-owner
replacement, or deadlocks. A batch derives its views from one proof, child planned
claims do not walk archives, and interrupted cleanup reservations recover safely.

## cleanup-batch: One discovery and one effect plan

Thread one explicit snapshot/catalog through `cli_work_cleanup_selection.py`,
`cli_work_cleanup_targets.py`, and `cli_work_cleanup_apply.py`. Load the requested bead
associations together. Replace `_verify_cleanup_target_still_selected`'s per-target
whole-view rebuild with one batch revalidation plus targeted fresh checks. After
confirmation, revalidate all targets before the first destructive effect.
Waiting-to-running transitions may shrink the launch set; a new owner, new generation,
new destructive action, or inconsistent bead association aborts without broadening it.

Send all selected roots to the core closure planner once. Route bead-work family members
and stale containers through that plan rather than recursively calling
`wipe_agent_name_for_reuse` and rebuilding per member. Share the low-level batch
primitive with existing explicit reuse callers while preserving their distinct user
authorization rules. Do not make generic explicit force reuse inherit bead-only
restrictions or bypass bead restrictions through the generic helper.

Acquire the guarded cleanup reservations, check authoritative process identity, markers,
generation and bead association again immediately before any kill/delete, then execute
each distinct effect once. A live non-WAITING worker must never reach the termination
helper. Retain the existing protection for RUNNING, STARTING, RETRYING, questions, plan
review, and contradictory live states. Preserve matching families and populated epic
clans as specified by the accepted relaunch plan.

Batch dismissed-index updates, notification dismissal, workspace-claim release, and
artifact-index maintenance over the union of completed effects. Verify removed names
against affected source records and reconcile the registry once per cleanup batch, not
per target. Unrelated records, reservations, history, and notifications survive. A
corrupt unrelated file retains the current best-effort discovery behavior; missing or
ambiguous evidence for a selected owner fails closed. Never reuse a truncated index
answer to justify destructive cleanup.

On failure, report exact completed and remaining targets. Commit no readiness or
preclaim changes until cleanup succeeds. Remove only this operation's guards when safe,
retain sufficient state to resume after interruption, and leave the deterministic rerun
command valid. Filesystem deletion is not transactional; do not claim rollback restores
wiped files. If an owner changes between effects, stop and report partial cleanup rather
than expanding authorization.

Acceptance: preview/confirmation/revalidation perform a constant number of full source
passes regardless of owner/family-member count; cleanup performs at most one initial
catalog scan and one final full reconciliation if needed, with no per-owner rebuild.
Existing safety, family reuse, and unrelated-owner tests continue to pass.

## launch-batch: Reserve once and consume reservations

In `cli_work_handler.py` and `multi_prompt_launch_execution.py`, construct the final
selected phase/land launch specifications and their known artifact identities before
spawning. Reserve their static deterministic names and required clan state in one batch.
Carry typed reservation evidence through `launch_executor.py` and runner metadata
extraction. The child still performs its exact ownership check; the parent and each
segment no longer ask the same global free-name questions independently. Keep public
executor callers without batch evidence on strict validation. Do not introduce an
environment flag that disables name checks.

Retain ordered segments, wait dependencies on preserved work, timestamp uniqueness,
model routing, first-launched-segment clan summary duties, prompt/environment alignment,
and deferred workspace acquisition for waiting runners. Do not wait for an LLM or runner
capacity merely to register the remaining deterministic names. Dynamic/bare name
resolution in general multi-prompts keeps its existing dependency behavior; optimize the
fully known epic path without changing that contract.

Release only unconsumed reservations on failure. Once any process has spawned, preserve
the current preclaim/recovery semantics and roll back only this invocation's spawned
results. Never terminate or release a preserved worker. Check failures before first
spawn, midway through fan-out, and after spawn but before metadata claim. Ensure
all-live retries return `already_running` without cleanup, snapshot, bead writes,
publication, or reservations.

Introduce a command-scoped context for reusable imports, immutable configuration,
identity and source enumeration across multiple targets in `cli_work_entry.py`. Refresh
mutable ownership/bead data at each target boundary; do not preclean, preclaim, or
reserve later targets before earlier ones succeed. Ordered JSON Lines, per-target
results, stop-at-first-error, prior successful side effects, plan-file targets,
standalone tasks, `--wait`, `--dry-run`, `-y`, and `-Y` remain compatible. Dry-run must
not create reservations or mutate agent/bead state.

Keep one required checkpoint/publication operation per processed epic. Add timing inside
the sync worker only if phase-one evidence cannot attribute its time; optimize proven
repeated local preparation there only within the same publication contract. Do not fold
multiple epics into one transaction or weaken remote visibility.

Acceptance: no whole-history proof in the per-slot spawn loop or exact child claim;
preparation scales with selected slots plus a bounded history cost per target. The
parent's faster return must coincide with promptly registered runners, not delayed work
moved into every child.

## acceptance: Performance targets and final verification

Use the identical isolated fixtures and recorded revisions for before/after results.
Report median and range over at least five measured runs per primary scenario, cold and
warm separately; report p95 only with at least twenty samples. Use the full scenario
matrix for structural tests and select representative terminal, mixed-family, and
four-target cases for expensive history-scale timings. Keep network
fetch/integration/push, lock wait, local cleanup, spawn, and capacity admission
distinct. Verify completed timing spans rather than summing overlapping stages. Use
lightweight real child processes to detect moved work and lock contention, plus
deterministic fault injection for correctness.

Required structural gates:

- For unchanged history, increasing K does not increase full archive scans, registry
  rebuilds, or global source proofs proportionally. Small tests assert these bounds.
- Fresh launches and exact reserved child claims never parse historical bundles per
  segment; all-active retries never enter destructive or publication paths.
- Concurrent source changes produce bounded retries/targeted reconciliation, not stale
  reuse. Two concurrent launchers preserve both sets of unrelated claims.
- No global name lock encloses archive parsing, filesystem cleanup, provider work, or
  network operations. Record wait/hold times and test contention explicitly.

Performance acceptance targets, to be measured rather than presumed:

- At approximately 40k historical sources and twelve replacement runners, reduce local
  orchestration CPU/wall time by at least 10x versus the baseline, targeting at most 45
  seconds for local orchestration on athena and at most 5 seconds for an all-active
  no-op retry. Keep cold recovery measurements separate.
- For four representative epics totaling about thirty replacement runners, target at
  most 90 seconds of local orchestration. Also report actual end-to-end duration with
  normal publication enabled. A 20-second publication cost per target can remain visible
  even after local work becomes much faster.
- Growing history from 10k to 40k must affect the bounded discovery portion, not
  multiply per-slot cleanup/claim time. Report fixed-K and fixed-H curves.

If a target is missed, use the new stage/call-count evidence to fix remaining repeated
work in this scope before accepting the epic. Do not declare success based on a tiny
fixture, synthetic sleeps, increased runner capacity, or merely faster CLI return.
Wall-clock thresholds belong in the controlled benchmark; ordinary CI asserts structural
work bounds and correctness to avoid flaky time-based tests.

Extend the relevant existing suites: `tests/test_bead/test_cli_work_cleanup_apply.py`,
`test_cli_work_epic_relaunch.py`, `test_cli_work_multi_target.py`, the epic checkpoint,
publication and rollback tests, `tests/test_agent_name_registry_*.py`,
`tests/test_agent_name_wipe.py`, `tests/test_agent_names_forced_reuse.py`,
`tests/test_agent_launch_executor.py`, and multi-prompt launch tests. Include:

- Waiting-to-running, new generation after preview, mismatched/missing bead, family
  member promotion, duplicate aliases, planned/clan collisions, and PID reuse.
- Overlapping cleanup closures, bundle-only history, incomplete indexes, missing or
  corrupt selected metadata, stale sources, and unrelated records remaining intact.
- Crash after reservation, after some removals, before/after registry publication,
  between spawns, and before child claim; reruns recover without duplicate workers.
- Publication failure before spawn, bead relocation rewriting, zero-spawn versus
  partial-spawn rollback, and later-target failure retaining prior target success.
- Task, plan-file resume, human output, JSON Lines, and genuinely non-mutating dry-run.

Update `docs/beads.md` with the preserved command behavior, progress interpretation,
existing timing switches, and the difference between scheduling and capacity admission.
Publish a concise benchmark report as an audited SASE artifact with reproduction
commands, fixture sizes, versions, before/after results, and residual bottlenecks. Do
not edit memory files or generated instruction shims as part of this epic.

Every phase changing SASE runs `just install` if needed and `just check`. Core phases
run the core repo's `just check` or `./scripts/check.sh`, including PyO3 tests with
Python >=3.12; `cargo test -p sase_core` alone is insufficient. Run the final combined
SASE `just check-full` only through `/sase_monitor`, with TESTING/TESTED statuses, and
run long benchmark lanes through that skill too. Record core release/binding
compatibility and both repositories' results before landing. Production epic retries are
not a benchmark fixture; validate against isolated state and observe subsequent
authorized real launches without manufacturing destructive retries.
