---
tier: epic
title: Indexed agent name, session, clan, and dismissed-bundle resolution
goal:
  Preserve agent lookup results while removing archive-wide work from authoritative
  hijack resolution and achieving warm resolution below one second on athena.
parent_bead: sase-1t3.6
phases:
  - id: lookup-core
    title: Targeted Rust artifact lookup queries and bounded recovery contract
    size: medium
    depends_on: []
    description:
      "lookup-core: add read-only, target-selective artifact queries, compatible lookup
      projections, bounded unavailable-index recovery, PyO3 contracts, and focused
      Rust/binding tests. Keep maintenance off the lookup path."
  - id: named-bundles
    title: Indexed named-agent and dismissed-bundle resolution
    size: medium
    depends_on:
      - lookup-core
    description:
      "named-bundles: replace find_named_agent's full artifact scan and recursive
      dismissed-bundle scan with Rust-backed exact name/workflow queries; maintain the
      archive projection on writes and prove ranking and recovery parity."
  - id: session-clans
    title: Indexed session and clan resolution with hijack parity
    size: medium
    depends_on:
      - lookup-core
      - named-bundles
    description:
      "session-clans: replace session and clan archive enumeration with targeted Rust
      queries and group resolution, preserve generation and archived-completion
      semantics, and verify authoritative hijack resolution without archive scans."
proposed_by: bbugyi200.athena.sase-1t3.6
proposed_with: codex/gpt-6-astra
create_time: 2026-11-16 08:12:14
status: wip
---

- **PROMPT:**
  [prompts/202611/indexed_agent_lookups.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/indexed_agent_lookups.md)
- **PARENT:**
  [202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/tui_storm_trust_heap_recovery.md)

# Indexed agent lookups

This is a child implementation epic for `sase-1t3.6`, phase `indexed-name-lookups` of
`plan:202611/tui_storm_trust_heap_recovery.md`. Read that artifact with
`sase artifact read` and the assigned phase with `sase bead read`; the ancestor's
decisions are final. No memory edits are authorized. In particular, do not edit
`tui_perf.md`. This work does not propose a global reinstall or change trust behavior.

## Evidence and size

At sase `e27b302028`, the assigned phase was in progress and had no prior implementation
or remaining-work notes. Inspection found the original paths still present:

- `src/sase/agent/names/_lookup_named.py::_find_named_agent_candidates` calls
  `scan_agent_artifacts` with no record bound. `_find_named_dismissed_bundle` runs
  `rglob("*.json")` and parses every bundle on a miss.
- `_lookup_groups.py::_iter_agent_session_members` and `_iter_clan_members` enumerate
  `iter_ace_run_artifact_dirs()` and read each `agent_meta.json`, then hydrate matching
  members' terminal state. Exact-root and exact-member lookups also use those iterators.
- `src/sase/agent/hijack/facts.py::resolve_hijack_subject` uses these APIs for the CLI
  and persistence worker. The ancestor records 17–38 seconds per resolution and 65,875
  dismissed JSON files. These are inherited measurements, not new timings.
- At inspected sase-core `50a6d972`, artifact scalar columns already include
  `agent_name`, `workflow_name`, `agent_session`, and `agent_clan`. Session/clan and
  workflow indexes exist; the agent-name index uses NOCASE. Generic candidate filters do
  not express all required exact lookup semantics.
- The generic `query_agent_artifact_index` may repair active rows and, with full history
  plus revalidation, reconcile source directories. Even its cached opener migrates
  missing/stale databases. Calling this generic API with a filter is not a sufficient
  fix.
- The archive already has `dismissed_bundle_summaries` and an agent-name index. Its
  existing query matches agent names/source run IDs, lacks a distinct `workflow_name`
  lookup, and defaults to hidden visibility. Reuse this store.

This is an epic because query/storage contracts, named/bundle resolution, and group
resolution each have separate compatibility and verification obligations. Each phase is
bounded medium work. Dependencies serialize integration and pin changes; there is no
extra planning phase or separate speculative indexing service.

## Shared contract

1. Open `sase-core` using `/sase_repo` and work only in its printed checkout. Read its
   `AGENTS.md`. Rust owns new query, projection, identity matching, ranking, and group
   selection logic. Python supplies current-owner identity and OS observations where
   existing adapters require them, then translates results into the existing public
   dataclasses. Do not add a parallel Python implementation of the new backend logic.
2. Healthy lookups are proportional to matching candidates/members, not total history.
   Match in SQL before decoding record JSON. Do not query all rows and filter in Python
   or Rust. Include historical/hidden rows needed for explicit references; inbox
   visibility and recent-history windows must not truncate identity resolution.
3. Preserve existing current-owner candidate order, local/qualified relationship
   equivalence, foreign-owner isolation, and case behavior. Use existing identity
   helpers; NOCASE indexes alone do not define name semantics. Restrict source artifact
   matches to `ace-run` records with actual `agent_meta`, as the old implementation
   does.
4. The index discovers candidates; current marker/process observations still determine
   actionable state. Revalidate only matching artifact directories and directly supplied
   targets. Confirm lifecycle upserts cover creation, rename/promotion, dismissal, and
   marker mutation. A new successor must become discoverable without a full-history
   refresh. Do not infer liveness from cached `status` or `has_running_marker` alone.
5. Return explicit availability/completeness alongside query results: a healthy empty
   result differs from absent, stale-schema, corrupt, busy, or incomplete data. Use
   no-create/no-migrate read-only opens and bounded lock acquisition (both SQLite and
   Python's process-local lock). Never silently invoke full rebuild or reconciliation
   from the lookup path.
6. For unavailable indexes, follow the existing cheap-detect/bounded-fallback pattern:
   direct artifact/member pointers first, then a bounded source scan using the
   established 200-record fallback budget with prompt/step reads disabled. Bound
   directory traversal as well as JSON hydration; avoid collecting or sorting an entire
   archive before applying a result limit. Share this recovery implementation rather
   than recreating it for each public lookup. Bundle fallback may use a known direct
   bundle pointer or a similarly bounded scan; never the old recursive walk. Preserve
   incompleteness through the internal API. A partial group cannot authorize stopping a
   possibly different session occupant, and an unavailable lookup cannot become an
   authoritative historical miss or completion. Surface the existing failure/diagnostic
   channel when completeness cannot be established. Recovery uses existing index
   maintenance outside the lookup, after which the same request must converge to the
   full result; do not add synchronous auto-repair or a new daemon.
7. Existing public lookup signatures and returned dataclass shapes stay compatible. The
   scan implementation is an oracle on controlled test fixtures only, not a production
   fallback. Keep unrelated name allocation, most-recent-name discovery, wipe/migration
   scans, TUI worker/cache changes, and general loader refresh out of scope.

## Phase lookup-core

Primary locations in sase-core are `crates/sase_core/src/agent_scan/index/`:
`index_wire.rs`, `storage.rs`, `record_summary.rs`, `maintenance.rs`, and a focused new
lookup module exported by `mod.rs`. Follow neighbouring bindings in
`crates/sase_core_py/src/` and add round-trip registration tests. Python mirrors and
thin facades belong under `src/sase/core/`.

- Define one typed target-query contract for exact ordered name/workflow candidates,
  session relation candidates, clan relation candidates, and exact artifact pointers.
  Return matching records or compact lookup data plus availability/completeness; do not
  construct unrelated clan context or TUI projections.
- Reuse existing scalar indexes where they satisfy exact comparisons. Add an exact
  agent-name index if required. For legacy session inference, either store a dedicated
  effective lookup relation or union selective existing session/workflow predicates and
  apply the legacy rule only to that candidate set. Do not redefine the existing
  `agent_session` column's documented meaning and break TUI query pushdown.
- Cover `_agent_session_base_from_meta`'s explicit session, plan-chain workflow, and
  member-name/workflow forms, plus `_clan_identity_from_meta`'s legacy parallel-session
  form. Reuse Rust's existing identity/plan-chain helpers. A matching workflow may be an
  SQL candidate without being a valid session member.
- Introduce any required schema/projection upgrade through existing maintenance. Healthy
  reads never build indexes. Detect an old database lacking lookup support before use,
  maintain required columns/indexes on rebuild/upsert, and update all Rust/Python schema
  mirrors and validation fixtures if their version changes.
- Implement the shared bounded recovery contract above. Test missing, old, newer,
  corrupt, and busy indexes without mutating the database or recursively walking the
  archive. Test that an explicit maintenance run restores complete indexed queries.
- Test selective reads with many unrelated rows, using query plans and decoded-record or
  file-access counts rather than brittle elapsed-time assertions. Include exact pointer
  aliases and symlinked paths; canonicalize both sides of comparisons.

Acceptance: the bindings select only requested identities, expose completeness, and have
bounded failure behavior. Normal lookup does no source-wide refresh. Existing TUI query
behavior and schema tests remain green. This phase provides infrastructure; it does not
claim the public lookup cutover is complete.

## Phase named-bundles

Change `_lookup_named.py` and its core adapter to use the new targeted artifact API.
Move changed ranking into a Rust helper with explicit observations as needed; retain the
existing process-identity/liveness adapter rather than inventing a PID-only test.

- Lock down the actual legacy ordering: outer current-owner candidate spelling wins;
  within a spelling, live/nonterminal versus historical priority is considered before
  exact-name versus workflow-name priority, then timestamp. `include_dead=True`
  distinguishes alive, dead nonterminal, and done; `only_done=True` excludes nonterminal
  rows. Non-root workflow matches are excluded for nonterminal candidates. Do not
  accidentally make a completed exact match outrank a running workflow match.
- Preserve dismissed-prefix handling even when `done.json` has been removed. For each
  owner spelling, artifact matches precede that spelling's bundle fallback; the next
  spelling is considered only afterwards.
- Extend the existing archive store in `crates/sase_core/src/agent_archive/`, not a
  second database. Add the source bundle's `workflow_name` projection and index, plus an
  exact name-or-workflow lookup binding. Do not substitute the archive's `workflow`
  display field. Preserve raw-suffix/filename-stem ordering and artifacts-dir versus
  bundle-parent fallback. Search all visibilities and workflow children, as the old scan
  did. Avoid unrelated bundle payload reads; hydrate a winning exact bundle only when a
  field absent from the summary is required.
- Maintain this projection through `upsert_bundle_summary`, rebuild, deletion, and
  existing dismissal/revive/rename flows. Update Rust wire/Python summary mirrors and
  schema checks together. An old bundle index takes bounded recovery, never a full
  foreground upgrade. Exclude stale deleted bundle paths and test repaired indexes.
- Add fixture-oracle parity for duplicate names across projects, exact versus workflow
  collisions, alive/dead/done, malformed markers, `only_done`, `include_dead`,
  qualified/local/foreign spellings, dismissed prefixes, bundle-only agents, child
  bundles, missing artifact directories, and all archive visibility states.

Relevant suites include `tests/test_agent_names_lookup.py`,
`tests/test_agent_names_forced_reuse.py`, `tests/test_agent_names_resume.py`,
`tests/test_dismissed_agent_names.py`, and `tests/test_dismissed_bundle_index*.py`.
Create focused additional files instead of further enlarging already large modules.
Patch the unbounded scanner and bundle `rglob` to fail in healthy-index regression
tests, including healthy misses and CLI lookup failures.

Acceptance: named lookups and bundle fallbacks preserve fixture results and no longer
read unrelated archive records. Dismissal immediately updates the lookup projection.

## Phase session-clans

Replace `_lookup_groups.py` archive iterators and delegate new group-selection behavior
to Rust. Preserve `AgentSession`, `AgentSessionMember`, `AgentClan`, and
`AgentClanMember` at the public Python boundary.

- Preserve newest root generation, transitive descendants, rootless legacy recovery,
  member sorting, dotted numeric root names, exclusion of canonical member names as
  session containers, and exclusion of plain same-named agents without session metadata.
  Explicit-root and explicit-member queries must select their own generation, even when
  a newer generation with the same session name exists.
- Preserve cycle protection and parent traversal behavior. Clan resolution picks the
  latest generation, including the legacy generation fallback to parent timestamp or
  artifact timestamp. Match local/qualified relation spellings through the same owner
  identity normalization as before.
- Keep archived-completion recovery for members missing done markers and its outcome
  classification. Feed only selected members to
  `src/sase/core/dismissed_agent_completion.py::load_archived_agent_completions` (or its
  thin replacement) and reuse the archive index. Preserve `archived_completion`
  provenance; do not equate dismissal with successful completion in group logic.
- Add parity fixtures for all these cases, multi-project histories, empty/corrupt
  metadata, old generations, monitor/gate members, and missing terminal markers. Reuse
  `tests/test_agent_names_lookup.py`,
  `tests/test_axe_chop_wait_checks_plan_agent_sessions*`,
  `tests/test_agent_chat_from_name_agent_session*`, and targeted hijack facts/stop
  tests.
- Exercise `resolve_and_decide`/`sase agent hijack --explain` and the persist-hijack
  resolution seam against indexed fixtures. Stub destructive process effects. Assert
  unchanged mode/code/selected occupant with unbounded scanners patched to fail, and
  safe refusal when a bounded group result is incomplete. Cover late successor creation
  and selected-target marker changes after the index was populated.

Coordinate with the active `sase-1t2` hijack epic and `sase-1t3` storm-hotfix phase:
read their current state and incorporate landed commits without waiting on unfinished
work. Preserve their StopDeps seams, row-local TUI resolver, latches, and stop custody;
the indexed backend must not weaken authoritative pre-stop revalidation.

Acceptance: all session/root/member/clan entry points avoid archive enumeration on a
healthy index; group membership and hijack decisions match the fixture oracle.

## Verification, pins, and landing

Each phase reads current `lint_and_test.md` and runs focused semantic tests plus
`sase tool run check` in every repository it changes. In sase-core, use the `just`
wrappers for targeted tests and formatting; never bare cargo. Use `/sase_monitor` for
long verification commands and wait for staging commands to exit. Do not run
`just check-full` unless explicitly requested separately. Do not globally install or
restart the live TUI. Build the local test environment through approved workspace
recipes when a new binding is needed.

Every new consumed binding needs a pin covering its core commit. Consult
`docs/contribute/rust-backend.md`: when both repositories are declared for commit, the
host commits core first and updates `sase-core-revision.txt` before the sase commit. For
an already-landed core dependency use `just ratchet-core-revision`; preserve any newer
concurrent pin. Never invent a SHA or alter the published version window.

The child epic's land agent verifies the combined scope and records measurements on
`sase-1t3.6`. Re-measure the baseline after incorporating any landed storm fix; report
the installed/core versions, fixture or archive size, target names, and cold versus warm
state. Measure representative hit, miss, session, clan, and dismissed-only lookups with
a synthetic archive at the research scale and read-only real athena `--explain` probes.
Report at least five warm samples as p50/p95/max, with internal resolution and whole CLI
timing separated. Target warm resolution below one second; record residual CLI startup
costs separately instead of attributing them to lookup. Do not stop or hijack live
agents to benchmark. Scaling tests must show unrelated history does not increase
hydrated records or bundle file reads.

If the target remains slow, profile before expanding scope. Correct lookup regressions
here; record independent residual costs as `PROPOSED FOLLOW-UP:` notes. Follow the
parent instruction for failures reproduced identically on the clean base: record the
evidence and any existing tracking bead, then close completed scope rather than leaving
an unrelaunchable phase open. Workers create no task beads.

Each child phase resolves or re-keys its own epic symbols before closing only its
assigned phase. The child epic land agent, after all child phases are complete, runs
`sase bead epic-symbols sase-1t3.6`, resolves remaining entries, and closes `sase-1t3.6`
with a note naming tests, parity evidence, and measurements. This is the designated
completion responsibility for the parent phase, not authorization for a child phase
worker to close ancestors. Never close `sase-1t3` or any higher ancestor.
