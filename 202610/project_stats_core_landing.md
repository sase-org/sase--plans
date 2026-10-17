---
tier: tale
title: Finish the project statistics core landing
size: medium
goal: Repair project statistics source aggregation and unavailable values, prove the
  sase history budget, and finish the sase-1m3.4.1 landing.
proposed_by: bbugyi200.apollo.sase-1m3.4.1.land
bead: sase-1m3.4.1
status: done
---

- **PARENT:**
  [202610/project_stats_core_engine.md](https://github.com/sase-org/sase--plans/blob/main/202610/project_stats_core_engine.md)
- **BEAD:**
  [sase-1m3.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1m3/sase-1m3.4.1.md)

# Finish the project statistics core landing

This is the remaining work from the land audit of `sase-1m3.4.1`, whose parent is phase
`sase-1m3.4`. Implement this bounded repair directly, then perform the complete closeout
below in the same turn. This tale has no separate land agent. Do not wait for this
work's commit SHA, push, or CI: host finalization commits after the turn.

## Context and decisions already settled

Read `sase-1m3.4.1` and its notes with `sase bead read`, and read
`plan:202610/project_stats_core_engine.md` and the relevant sections 4, 6, and 12 of
`plan:202610/sase_stats_project.md` with `sase artifact read`. The original four phases
are closed. Their implementation commits in `sase-core` are `ac370734` (runner),
`b0cd62b9` (contracts), `855ed4c3` (history/cache), and `be2bfbee`
(aggregation/binding). The audit examined those commits and the actual sources.

Open `sase-core` through `/sase_repo`, read its `AGENTS.md`, and work only in the
printed checkout. All production changes belong there. Open the plans repository through
`/sase_repo` before updating the linked plan at closeout. The parent's final decision is
`stats_decision_record = no`: make no memory changes. Retain unimplemented Code, Runs,
directory-loading, and plan facets as explicit null/empty seams for sibling phases; add
no Python caller, CLI, UI, dependency, or manual pin change. A later caller phase owns
the pin unless the host advances it automatically.

Integration was reviewed through sase `315092afeb` and core `be2bfbee`, both on master
with no additional local origin/master commits at audit time. Core `305b7b80` adds plan
`proposed_with` validation, which remains an input for the later plans facet. Sase
`7545e85abf` stamps normalized AGENT/MODEL footer values; the engine already consumes
these via `parse_commit_footer`. Sase `7c6b32457b` adds target-project inventory
resolution and declared-by provenance, which the later loader consumes. Subsequent
release integration `315092afeb` fixes the earlier provenance/inventory fixtures; do not
repeat those repairs. The remaining intervening changes are documentation, verification,
release, or unrelated UI work. Recheck new drift before closing, without absorbing the
other parent phases.

## 1. Separate source metadata, history reuse, and aggregate deduplication

Repair `project_stats/query.rs` and `output.rs` together:

- `collect_portfolio_code` currently deduplicates by identity/common-dir and continues
  before appending the later history rows. Two specs selecting divergent refs in one
  repository therefore yield a project SHA union larger than the portfolio. Always union
  every selected code-source history by SHA, regardless of coverage deduplication.
  Preserve unique history walks by identity/common-dir/tip.
- Coverage currently counts the same declared identity twice when it comes from separate
  clones. Use the repository identity for deduplicated coverage and measurement as
  required by the parent contract; keep common-dir in history/cache keys to avoid
  conflating distinct clone state. Reduce repeated source health deterministically so
  changing spec order cannot conceal a partial/unavailable selected source. Unresolved
  paths without a resolved identity remain explicit.
- `load_unique_sources` reuses an entire `WalkedSource`, including `ResolvedRepo` and
  issues. Two ref aliases resolving to the same tip inherit the first alias's
  `ref.name`; repeated missing paths inherit the first spec's issue.repo. Share raw
  history, but retain each spec's selected ref, configured identity, and issue
  attribution. Keep the query cache key faithful to inputs affecting resolution.

Add hermetic regression tests using the existing fixture helpers. Create a shared root
plus divergent main/side commits and select both tips, first within one project and then
across two projects. Assert project/portfolio SHA unions, model invariants, lifetime
counts, coverage, and invariance under spec order. Add same-tip different-ref aliases
and repeated unavailable paths with different display names; assert each repo row
retains its own ref/issue attribution. Exercise two clones with one declared identity so
coverage deduplication is separate from history isolation. Existing shared-history and
cold/warm equality tests must keep passing.

## 2. Preserve unavailable measurements instead of manufacturing zero

The contract says a requested unavailable facet is present with nullable values and an
issue; an available empty repository is measured zero. Currently an all-missing project
produces `Some(output)` and `Some(lifetime)` with `commits: 0` because `output::measure`
aggregates an empty vector, and the portfolio does the same. `ProjectStatsOutputWire`
and `ProjectStatsLifetimeWire` use nonnullable integer counts, so the required
distinction cannot be represented. A failed history walk also carries an empty
`HistoryOutcome` and is indiscriminately marked partial and measured.

Make the Output/lifetime measurement counts nullable where absence is meaningful, and
update aggregation, invariants, fixtures, and the binding round-trip accordingly. Keep
the enclosing requested facet present with coverage and source issues. Emit numeric zero
for successfully measured empty repos/empty intervals, numeric measured subtotals for
mixed success and failure, and null for wholly unavailable measures. Do not emit a zero
model distribution as if unavailable history had been measured. Distinguish a failed
source from a truncated walk with usable complete records; retain explicit partial
coverage and issues for bounded lower-bound results. Keep unrequested facets null and
preserve the promised null seams for later facets.

Test all missing sources, a history-command failure after successful resolution, mixed
success/failure, an available empty repo, an empty range on nonempty history, and
partial history. Assert serialized JSON values through the binding as well as domain
values, including portfolio behavior. Read the core compatibility rules and describe any
nullable-wire compatibility impact in the host commit declaration; do not edit release
versions/changelogs by hand. Search for consumers that may have landed since the audit
and adapt any affected callers within this repair.

## 3. Finish the extension contract and real-repository acceptance

`project_stats/mod.rs` currently defines `MeasuredRepo`, `FacetContext`, and
`FacetResult` plus its impl. Move those bodies to a sibling module and re-export them
with appropriate visibility; the facade must contain only module declarations and
re-exports, as required by the original plan and core conventions.

The ignored `measured_history_acceptance_on_sase_checkout` test currently falls back to
the core checkout. Its first candidate (`../../../../..` from the crate manifest) does
not reach the primary sase checkout in a managed linked layout, and `../..` is
sase-core. The phase's recorded 1,568 commits and 93.2 ms warm median therefore do not
satisfy the required sase measurement.

Make the manual harness accept an explicit authorized sase repository path (a
non-SASE-prefixed environment variable is compatible with the hermetic runner), validate
that it is the intended repo, and fail clearly for a wrong/missing explicit path instead
of silently benchmarking core or returning success without measuring. Do not encode a
numbered workspace path in source. Run it from the supported `just test` recipe against
the current primary sase checkout, with fixed range and now values within that run,
fresh temporary cache, and at least five warm samples. Record repository identity, tip,
total/range history counts, timeout/output budgets, cold time, warm median, cache state,
and whether extension startup is included on both `sase-1m3.4.1` and `sase-1m3.4`.
Confirm equal cold/warm data and fingerprints. Target the existing warm one-project
budget of 150 ms. If missed, measure and fix avoidable history/cache overhead in this
engine before declaring completion; for example, inspect unconditional equal-tip cache
rewrites and memo eviction. Do not claim the later combined history/directory/plans
target has been measured.

Run focused domain and binding tests and the shared file_history runner tests, then
`just fmt` and the required `sase tool run check` (the guarded `just check` gate) in
sase-core. Never use bare cargo or `just check-full`. Use `/sase_monitor` before a long
gate, with a follow-up that finishes this tale's remaining closeout; wait for the
monitor-start command itself to exit. If any tracked sase files need changes, read
`lint_and_test.md` and run that repository's `just check` too. A targeted test is not a
replacement for the gate.

All four original phases reported the same clean-base Clippy 1.95 nonminimal_bool deny
in `project_onboarding/classify.rs:500-502`, tracked by `sase-1an`. The land audit
corroborated unchanged source at core `be2bfbee`; it did not rerun a compiler. Run the
gate on the repaired tree and fix any failures caused by this work. A failure proven
identical on the clean base is handled under the original plan's explicit
baseline-failure policy, with exact evidence, rather than being relabeled as a pass or
treated as remaining project-statistics work.

## 4. Finish this tale, the child epic, and its parent phase

Before closing, reread `sase-1m3.4.1`, all descendants and their notes, and the two
linked plans through the audited commands. Verify the four original phases remain
closed, this repair covers every finding, and no new descendant or integration work
remains. The landing note on `sase-1m3.4.1` records all proposed-follow-up outcomes:
reuse that record in the close note instead of filing them again. Specifically, the four
Clippy proposals from `.1` note 1, `.2` note 2, `.3` note 1, and `.4` note 1 were
consolidated into a corroborating +1 on `sase-1an`. The legacy-reader proposal from `.2`
note 1 became ready medium bug `sase-1mb`, with independent pipe-deadlock reproduction
and two durable source snapshots. That pre-existing migration is outside this tale. No
other child proposal was declined; the parent phase's skipped decision record remains
intentionally skipped under `stats_decision_record = no`, without a new memory task.

Complete this tale's implementation and verification first. Inspect the actual assigned
bead and descendant tree: if the host created a distinct child bead for this tale,
resolve its exemptions and close that child normally with
`sase bead close <this-tale-child> --note "<implemented repairs and verification>"`
before closing its parent. If the coder inherits `sase-1m3.4.1`, do not pre-close it:
use the ordered closeout below. Never invent a child bead merely to execute these
instructions. Mark this tale's own linked plan done as its lifecycle requires.

Then perform the following concrete closeout in order:

1. Run `sase bead epic-symbols sase-1m3.4.1`. Resolve every listed symbol through a real
   consumer, privatization, justified non-test pragma, or deletion. Re-key a Justfile
   exemption only when a specifically identified still-open later bead still needs it.
   The audit found no entries, but check again on the final tree.
2. Close normally with
   `sase bead close sase-1m3.4.1 --note "<repairs; all four child-note checks; integration commits reviewed; domain/binding/runner and gate results; actual sase measurements; every follow-up outcome; symbol cleanup>"`.
   If refused for leftover symbols or descendants, finish the named work and retry.
   Never use `--force` to advance this successful nested landing.
3. From the primary sase checkout run `just symvision` and confirm no stale whitelist
   remains. In the opened plans repository set `status: done` in
   `202610/project_stats_core_engine.md` (artifact
   `plan:202610/project_stats_core_engine.md`). Read reference memory before any
   tracked-code cleanup and satisfy the corresponding verification gate.
4. Reread `sase-1m3.4.1` for its parent link and `sase-1m3.4` for phase readiness.
   Confirm the completed child and this repair satisfy the parent's core-engine scope
   and that every descendant is closed. Run `sase bead epic-symbols sase-1m3.4` and
   retire/re-key any remaining entries using the same rules. Close only that parent
   phase normally with
   `sase bead close sase-1m3.4 --note "<integrated core-engine verification, actual sase performance, baseline disposition, and completed child epic>"`.
   Recheck `just symvision` if whitelist changes or this close affect exemptions. Leave
   containing epic `sase-1m3` and `plan:202610/sase_stats_project.md` open for its
   already-waiting land agent. If phase readiness is genuinely incomplete or ambiguous,
   record the concrete blocker on `sase-1m3.4` and report it instead of forcing closure.

Submit `/sase_final` last for every repository changed, including the plans repository.
Choose completion from verified work, never from whether the host's not-yet-created
commit is visible.
