---
tier: epic
parent_bead: sase-zu
title: Finish indexed agent-history correctness, reuse and measured acceptance
goal: Repair the confirmed row-loss and freshness gaps, preserve later queue and Refresh
  panel changes, and prove the remaining sase-zu acceptance criteria.
phases:
- id: production-oracle
  title: Make the parity oracle exercise production history and refresh paths
  size: medium
  depends_on: []
  description: 'production-oracle: extend the harness to reproduce unseen artifacts,
    conflicting machine provenance, settled bounded queries and query-keyed refresh
    invalidation through production entry points.'
- id: index-freshness
  title: Make indexed history authoritative without archive-wide marker repair
  size: medium
  depends_on:
  - production-oracle
  description: 'index-freshness: repair the Rust-owned completeness and discovery
    contract, bound steady-state revalidation, and adopt its wire through the existing
    Python loader.'
- id: machine-parity
  title: Repair machine candidate parity across provenance and tree projection
  size: medium
  depends_on:
  - index-freshness
  description: 'machine-parity: make candidate selection preserve every live row across
    source-owner conflicts, tree descendants and both query dialects, proving exactness
    before retaining negated pushdown.'
- id: refresh-integration
  title: Finish query-keyed delta reuse and integrate completion with Refresh
  size: medium
  depends_on:
  - machine-parity
  description: 'refresh-integration: preserve exact deltas under committed queries,
    invalidate overflow and stale-query work, ensure settled completeness, and stamp
    successful Agents history completion in Refresh.'
- id: acceptance
  title: Verify the pinned cohort and complete measured acceptance
  size: medium
  depends_on:
  - refresh-integration
  description: 'acceptance: verify the selected Rust revision and supported install,
    prove production-path parity and performance on synthetic and real archives, publish
    evidence, and finish the three flag retirements.'
proposed_by: bbugyi200.athena.sase-zu.land
create_time: 2026-09-13 10:21:10
status: done
bead_id: sase-zu.8
---

- **PROMPT:** [prompts/202609/agent_query_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_query_landing_repairs.md)
- **PARENT:** [202609/agent_query_load_tiering.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_query_load_tiering.md)
- **BEAD:** [sase-zu.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zu/sase-zu.8.md)

# Finish the remaining sase-zu work

## Scope and evidence

This is the remaining work from the landing audit of sase-zu, not a replacement for its
completed implementation. Read these artifacts through `sase artifact read`:

- `plan:202609/agent_query_load_tiering.md` — original requirements.
- `file:explicit:8c69223181c49846b294f84a` — every original child-note disposition,
  source/commit review, integration inventory and two independent reproductions.

The main audit tree was `654335d555`, identical to freshly fetched origin/master. The
Rust checkout opened through `sase repo open gh:sase-org/sase-core` was `ba651fe`.
Reopen Rust through sase_repo and use the returned path; never infer a sibling checkout
location. Shared discovery, candidate selection and freshness contracts belong in Rust,
with Python limited to adapters and Textual state.

Preserve the shipped bounded first paint, quiet-time worker path, partial-history
notice, query/profile cache key, explicit pushdown field classification and removed flag
branches. The existing synthetic fixture and impossible-filter oracle test are useful
and should be extended.

Confirmed remaining defects:

1. After an index rebuild, a newly created artifact is absent from production
   full-history loading, which nevertheless returns complete_history=true and
   needs_full_history_reconcile=false. Revalidate only repairs existing SQL rows.
2. A row with source_machine=athena and imported_source_owner.machine_name=apollo
   matches the live query machine:apollo but is dropped by the indexed candidate. The
   index stores one scalar; the live adapter evaluates both values.
3. The oracle measures cached full history, while production forces revalidation. The
   fast fixture is smaller than its requested window and cannot prove settling.
   Published cached-history p50 is 7672.78 ms versus 8621.35 ms for source scanning; the
   required substantial full-history speedup is not demonstrated.
4. Rust select_records refreshes every row in where_sql before applying a candidate
   filter, including capped Tier 1 revalidate queries. Returning few records does not
   prove bounded disk work.
5. Active searches still force exact artifact deltas into broad Tier 1 fallback. Delta
   overflow does not invalidate complete-history reuse or request the required upgrade.
   Full-history async results bypass the bounded-prefix stale-query guard.
6. The main Rust pin c55326f has scan wire 8/index schema 27, while main requires 9/29.
   The pin predates this epic's Rust implementation and later queue changes.
7. Flag definitions were removed but retirement beads sase-zx, sase-101 and sase-107
   remain open. Fresh real-archive settled-session evidence is missing.

## Phase production-oracle

Extend tests/perf/agent_load_tiering_harness.py and its existing fixture/tests so the
primary acceptance oracle invokes the actual TUI loader and applies its final query/tree
semantics. Keep an explicit authoritative scan reference. If direct core cached reads
remain useful, label them separately from production full history.

Cover artifacts added after index build, mutations that move rows into or out of a
candidate filter, hidden/unhidden and deleted artifacts, and old completed rows beyond
the Tier 1 cap. Compare settled visible identities and descendants for the whole
existing battery, both query dialects, machine provenance conflicts and family/clan
projection. Distinguish temporarily absent bounded history from rows that never arrive;
exercise the existing quiet/prefix reconcile scheduling and manual-full-history path
without long wall-clock sleeps.

Add regressions for repeated unchanged-query refreshes, exact deltas affecting old rows,
overflow, query/profile changes while disk or preparation work is in flight, and
coalescing/spawn failures. Demonstrate the current defects with diagnostic oracle runs
and passing oracle-contract tests that prove detection of missing rows and false
completeness. Following repair phases must add regression assertions requiring zero
differences; do not leave the default suite knowingly red between phases, skip the
broken cases or invent a second loader in tests. Use deterministic read/repair/decode
counters to complement p50/p95/max timings. Keep archive-scale runs out of the default
fast test lane.

Acceptance: diagnostic runs detect the two audit failures against the current
implementation; an intentionally under-selecting candidate still fails the oracle; real
production freshness parameters and settle behavior are visibly exercised. Record
failing nodes as expected remaining epic work for the following phases.

## Phase index-freshness

Repair the completeness decision in the existing Rust artifact-index subsystem and its
Python loader adapter. A snapshot may claim complete history only after accounting for
source changes through a defined reconciliation boundary, including previously unindexed
directories. Reuse existing index lifecycle, targeted upsert, watcher/change tracking
and rebuild mechanisms. Establish what happens after an unobserved write, event
overflow, process restart and index rebuild; do not assume that marker revalidation
discovers new records.

Keep first paint bounded and do repair off the UI pump. A required initial
reconciliation may be deferred, but once it settles, unchanged-query refreshes must not
pay an archive-wide marker-signature or JSON-read cost again. Do not simply switch
freshness to cached and retain a false completeness claim. Bound Tier 1 revalidation and
eliminate redundant prefilter/full-selection marker work; preserve repair of rows whose
changed scalar fields would otherwise exclude them. Update wire metadata/versioning and
bindings when the contract requires it.

Preserve missing-index, schema-stale, operation-lock-busy and exception recovery through
the existing scheduler. Keep absent/deleted rows from being served forever as stale
record_json; preserve dismissal/retention semantics. Update misleading
full-history/source-scan comments in callers as the finalized contract warrants.

Acceptance: production oracle discovers post-build artifacts and converges after
mutations/deletions without false complete_history; unchanged full-history and periodic
Tier 1 revalidation have bounded marker work with measured counters. Run Rust repository
checks including PyO3 and Python focused integration tests.

## Phase machine-parity

Audit the complete values used by live and legacy machine evaluation after the
phase-seven imported-owner and clan changes. Cover source_machine versus owner, meta
versus done marker precedence, absent values, case/whitespace handling, workflow
children and mixed-provenance family/clan descendants. The current fixture only uses
agreeing source/owner values, so it cannot justify exactness.

Repair candidate projection in Rust and the thin Python projection/adapters so positive
filtering never under-selects and negated filtering is used only when exactness holds
for the evaluated dialect and final tree. If an expression cannot be proven exact,
deliberately leave it on the now-bounded fallback path, as the original plan permits. Do
not redefine user query semantics to fit the index or assume every descendant shares its
container's provenance. Retain explicit field classification; a newly added unclassified
field must fail its coverage test.

Acceptance: production/source parity for both machine polarities and nested boolean
compounds, including the audit's conflicting-provenance example, tree containers,
bare-machine behavior in each dialect and records added or changed after indexing. Run
Rust/PyO3 and both Python query-adapter suites for any shared-wire change.

## Phase refresh-integration

Finish the existing canonical-query/profile-digest reuse mechanism. Committed queries
must not disable exact artifact updates: apply changed/deleted artifact rows to retained
history, then evaluate the current query and tree. Carry enough unfiltered state to
remove rows that cease matching. Overflow or loss of reliable delta coverage must
invalidate the relevant completeness watermark and schedule one coalesced
reconciliation. Repeated ordinary refreshes for unchanged covered history must not
restart it.

Guard stale results at the disk and final apply boundaries for all load tiers, including
query changes during worker preparation. Do not attach old-query rows or completeness to
the new query. Ensure both fallback and pushable bounded queries eventually satisfy the
original settled-row invariant, even when older matches lie outside the recent prefix.
Preserve callbacks and scheduled/running/ pending cleanup when worker spawn or loading
fails.

Integrate the newer Refresh panel's Agents history surface with successful completion.
It currently stamps manual_full_history at request time and misses automatic
completed-history upgrades. Preserve R, ,y, Full history, Everything and refresh_panel
On/Off behavior, while ensuring failed/stale history requests do not masquerade as
successful fresh history. Update relevant help/docs only where their user-visible
behavior changes.

Acceptance: an unchanged-query session with N ordinary refreshes performs exactly one
completed full-history upgrade; query change, explicit manual refresh and overflow
trigger the appropriate new upgrade; mutations to old rows remain exact; stale responses
never corrupt newer state; Refresh tests pass in both flag states.

## Phase acceptance

Select a compatible Rust source cohort containing the fixes and the later canonical
queue scan/editor fields (ba651fe lineage) and pressure reaping. Update
sase-core-revision.txt through the repository's supported pin workflow and verify the
exact selected source, Python mirrors and validation fixtures. Do not manually edit
release-managed crate versions. Verify required runtime support is represented by the
published dependency policy as well as the local development build.

Run a fresh supported just install using the sanctioned checkout and confirm the
resulting binding imports and implements the needed wire. The historical installer
downgrade proposal from sase-zu.4 was not independently reproduced; do not treat the
stale pin as proof of that different bug. If it recurs, capture exact installer output
and versions and route the new evidence through sase_new_task.

Run the complete production-path oracle and 13,000-artifact benchmark with source,
bounded first paint, actual full history, periodic revalidate and an N-refresh session.
Publish p50/p95/max, read/repair/decode counts, actual speedup ratios and zero settled
missing/extra rows in docs/perf_runbook.md. Attribute expensive stages and finish
epic-caused costs rather than calling a ~1.12x ratio the promised result.

Exercise the repaired current tree against the real Athena archive using the saved not
machine:apollo query. Capture fresh startup and load logs tied to the tested revision
and process, then observe a settled session with ordinary refreshes, an old-row delta
and explicit Full history. Prove indexed first paint in the baseline's range and no
repeated expensive full loads for unchanged covered history. Keep broad loop/pump/heap
work with sase-zn.9; ownership notes on that epic and sase-100 describe the narrow
integration seams this child owns.

After the corresponding exact-parity and winning-behavior evidence passes, close the
already-removed flags' retirement beads sase-zx, sase-101 and sase-107 with normal
evidence notes, and ensure their names are absent from executable flag
branches/registry/schema. Do not restore obsolete disabled branches to satisfy a flag
linter. The explicitly requested memory follow-up is already task sase-109; carry its
disposition and the installer proposal's disposition into acceptance notes without
creating duplicates or directly editing memory.

Run required verification, including the Rust repository's complete checks when changed
and main just check-full through sase_monitor with TESTING/TESTED. The combined gate,
real-archive result and provenance must be recorded; historical phase checks are not
substitutes. The child plan's parent_bead link hands the verified result back to the
interrupted sase-zu landing.
