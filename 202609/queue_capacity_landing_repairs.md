---
tier: epic
title: Finish queue capacity persistence, authoring and display
goal: Preserve authored queue budgets through real scans, history and continuations,
  present them truthfully, and complete the missing acceptance evidence for sase-zt.
parent_bead: sase-zt
phases:
- id: core-contracts
  title: Complete canonical capacity records and editor semantics in Rust
  depends_on: []
  description: 'core-contracts: preserve canonical and legacy capacity through metadata,
    waiting markers, launch wires and indexed scans; share persisted-zero handling
    and flag-aware editor suggestions in Rust.'
  size: medium
- id: adapters-continuations
  title: Adopt the complete capacity wire and preserve continuation budgets
  depends_on:
  - core-contracts
  description: 'adapters-continuations: advance the core pin without losing later
    contracts, finish Python canonical-field adoption, remove duplicate admission
    policy, and preserve exact capacity through monitor delivery.'
  size: medium
- id: presentation
  title: Complete capacity metadata, colors and both-state presentation
  depends_on:
  - adapters-continuations
  description: 'presentation: show authored capacity in the detail header and every
    eligible row, fix large-budget accents, preserve the legacy flag branch, and prove
    real scan-to-render behavior.'
  size: medium
- id: acceptance
  title: Complete visual, live and combined-tree acceptance
  depends_on:
  - presentation
  description: 'acceptance: inspect the required PNG changes, observe the real admission
    and TUI color scenarios, verify newer index and monitor integrations, and run
    the repaired combined-tree landing gate.'
  size: medium
proposed_by: bbugyi200.athena.sase-zt.land
create_time: 2026-09-13 07:09:15
status: wip
bead_id: sase-zt.6
---

- **PROMPT:** [prompts/202609/queue_capacity_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_capacity_landing_repairs.md)
- **PARENT:** [202609/queue_capacity_budget.md](https://github.com/sase-org/sase--plans/blob/main/202609/queue_capacity_budget.md)
- **BEAD:** [sase-zt.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zt/sase-zt.6.md)

# Remaining work for sase-zt

The original approved plan is `plan:202609/queue_capacity_budget.md`. All five original
phases are closed, but the landing audit at main f3a39fa835 found the concrete gaps
below. This plan repairs only those gaps. The directly linked parent is sase-zt; its
land agent resumes after this child completes.

The detailed audit and all original note dispositions are preserved in
`file:explicit:49e1cedbfb0962c856bb2e22`; consume it through `sase artifact read`. Its
automatic bead attachment failed in the existing hidden-plans publication incident
reported to sase-yy.8; the immutable snapshot itself was created.

## Evidence and preserved behavior

Rust c55326f implements the correct schema-4 inequality:
`occupied_capacity + effective_weight <= admission_limit`. Explicit capacity replaces
the global limit only for its launch. With queue_capacity_budget on, new zero capacity
and weight greater than authored capacity are rejected; persisted explicit zero means an
exact effective-weight drain budget. Off keeps the old occupied-load threshold and
independent global ceiling. The sunset flag and its separate removal bead sase-zv must
remain.

Main commits 89d51301fa, 3c89591db8 and dd1ed61a4a implement substantial adapter,
display and documentation work. The land audit passed 92 focused tests, but reproduced
these gaps using actual source and bindings:

- A real `scan_agent_artifacts` of isolated agent_meta.json and waiting.json containing
  only queue_capacity=100/queue_capacity_explicit=true returns None for both capacities.
  Rust AgentMetaWire has no capacity field, while WaitingMarkerWire/scanner still read
  only wait_runners. New running and indexed history rows therefore lose the authored
  badge when waiting.json disappears.
- f3a39fa835 prevents parked head waiters from blocking the whole queue by writing both
  spellings. Its real-scanner test passes in both flag states. This is a compatibility
  bridge to preserve until the scanner and pinned extension agree.
- A named RUNNING agent with weight=2 and explicit capacity=100 renders Weight: but no
  Capacity: in `build_header_text`. Legal capacity 4294967295 renders in scientific
  notation with a quiet color under global limit 1, because the gold accent compares
  formatted strings using isdigit().
- `queue_launch_prefix({wait_runners: 0, queue_capacity: 3, queue_weight: 1})` emits
  capacity=0, which the actual On parser rejects. Its current test expects the obsolete
  spelling to win. Legacy-only zero also cannot simply be reauthored into a new prompt
  after the semantics change.
- Rust editor/directive.rs still suggests 0 as a drain and describes 1/capacity as an
  occupied-load threshold. The user can complete syntax that launch rejects.
- Neither original .3 nor .5 documents the required inspected visual evidence. Phase .5
  confirms live admission and parking, but not the red global header or gold/quiet row
  colors.

Later work to preserve includes schema-28 full-history filtering (core 7949496/main
3c1185c281), indexed loading/session history reuse (f609668b72/2e08f0842d), per-segment
epic budget flooring (9429544b53), monitor continuation capture/delivery (638647694b),
and directive doctor diagnostics (2170f422e9). At audit time the installed extension was
0.34.24 but the main source pin remained c55326f. A passing installed wheel does not
prove rebuilding that old pin preserves schema-28 integrations.

## core-contracts

Open core through `sase repo open sase-core`, or the sanctioned external
`sase repo open gh:sase-org/sase-core` when configured lookup fails. Use only the
returned checkout. Follow its release and verification instructions.

1. Complete the queue_capacity/queue_capacity_explicit contract in
   `crates/sase_core/src/agent_scan/{wire,scanner}.rs` and
   `agent_launch/{mod,admission}.rs`. Readers must accept canonical-only, legacy-only
   and the dual-written transition records without duplicate-field deserialization
   errors. Canonical values win when both spellings are present; omission, explicit
   false and explicit zero must remain distinguishable. Writers emit canonical names
   after the consuming cohort is ready.
2. Preserve authored fields in both metadata and waiting-marker projections, filesystem
   scans, index serialization/rebuild/query, and minimal/list record shapes. Advance
   wire/index schemas where existing rows would otherwise cache missing fields
   indefinitely. Preserve machine provenance and full-history candidate filtering from
   the newer core changes.
3. Keep persisted-zero normalization in shared Rust policy. Provide or reuse the narrow
   contract the continuation adapter needs so existing zero-capacity records resume with
   an exact effective-weight drain budget. Do not translate fractional weight to a
   rounded integer budget that permits extra occupied load, or silently drop capacity
   and fall back to the global limit. Newly authored zero remains invalid with the flag
   on.
4. Make queue editor metadata/suggestions respect queue_capacity_budget, including
   positional/keyword/colon forms. On suggests valid positive budgets with correct help;
   Off retains its zero drain and threshold semantics. Propagate the same contract to
   ACE and the LSP using their existing flag request path. Preserve duplicate-key,
   migration, u32 upper-bound and weight validation behavior.

Add Rust and PyO3 contract coverage for canonical-only metadata/waiting/index records,
legacy and dual aliases, explicit markers, both flag states, and editor
completion-to-parser parity. Run the whole core `just check`, including binding tests,
through a monitor if lengthy. Existing sase-xv describes this host's libpython loader
workaround; arrange the selected interpreter's LIBDIR or use a known working explicit
interpreter. Record the verified core SHA.

## adapters-continuations

1. Advance `sase-core-revision.txt` to the verified core containing this repair and the
   later schema/index/retention contracts, refresh validation probes and schema
   constants, and run `just install`. Follow the existing development/published
   dependency conventions; do not manually edit release versions or substitute an old
   core pin for a currently passing wheel.
2. Finish canonical capacity fields through agent scan/launch Python mirrors, directive
   metadata, slot markers/polling, ACE model/enrichment fields, integration/listing
   serializers and CLI/ops adapters. Keep explicit read aliases at the boundary; use
   canonical names internally and in writers. Include any necessary coupled consumers
   opened through sase_repo. Remove f3a39fa835's dual-write bridge only once
   real-scanner regression tests pass without it.
3. Update monitor continuation delivery to prefer canonical capacity and preserve exact
   authored or persisted legacy semantics through the existing durable delivery path.
   Positive budget inheritance, legacy zero, simultaneous old/new fields, fractional
   weight, and explicit priority zero must all survive. Do not pass historical zero
   through the new-authored parser or build a second delivery journal. Preserve the
   newer capture rollout and exactly-once admission behavior. Coordinate via notes with
   active sase-zl.13.11 if it still edits these seams.
4. Resolve the original plan's duplicate-policy helpers. `may_start` currently has only
   tests and a package re-export as consumers; delete it and dead tests if the
   external-consumer check confirms that. Re-express any necessary display fallback from
   shared snapshot facts, without implementing admission a second time from integer lane
   count/default weight. Preserve the Off semantics and avoid claiming eligibility when
   the global limit is unknown.

Test the actual writer -> Rust scanner -> snapshot -> admission path, including a
capacity-blocked head waiter followed by an admissible high-budget launch in both
states. Test metadata-only running/completed rows and index refresh/rebuild after
waiting-marker removal. Run the existing per-segment epic-capacity tests to preserve the
newer weight flooring, and real continuation prefix/admission tests rather than only
string formatting assertions. Run `just check`.

## presentation

1. Add the planned authored Capacity: field beside Weight: in
   `widgets/prompt_panel/_agent_display_header_metadata.py`. Show authored capacity even
   with default weight and in every status. Match the row's suppression rules for clan
   containers, proc/gate/monitor shells and serial child rows; preserve family-container
   and parallel-member behavior.
2. Reuse the existing badge styling: dim c, quiet #87AFD7, gold #FFD700 when the
   authored integer exceeds the known global limit. Compare validated numeric values,
   not formatted digit strings. Preserve the exact authored integer through the u32
   bound; unknown limits stay quiet. Include zero only when presenting an existing
   legacy record, with its meaning explained honestly.
3. Make row, detail and ladder facts survive real scanner/index loads and
   full/selective/history-reuse refreshes. Keep authored fields distinct from per-waiter
   live admission_limit/free-capacity facts and global header pressure. Ensure any
   detail occupancy denominator uses the waiter's admission limit. Rendering must remain
   free of filesystem scans or synchronous I/O.
4. Audit wait/approval labels in both flag states: the existing Off behavior must not be
   described as the On budget. Only describe capacity=1 as running alone when effective
   weight makes that true; fractional-weight launches may share a budget of 1. Preserve
   newer per-segment epic flooring and existing controls. Keep relevant product
   docs/help consistent without redoing the already-correct xprompts memory paragraph.

Use behavioral render tests for running/queued/done rows, family and parallel members,
suppressed rows, unknown/over-limit values and flag transitions. Include real
scan/enrichment-to-render regression coverage so preconstructed Agent fixtures cannot
hide field loss. Run `just check`.

## acceptance

1. Run the affected Agents and Models panel PNG visual suites. Inspect
   actual/expected/diff images before accepting intentional goldens; explicitly cover a
   quiet c1 and a gold c100, authored Capacity: metadata and the global header pressure
   state. Record inspected results, not just snapshot generation.
2. Complete the approved live smoke through authorized SASE launch workflows. Preserve
   and restore the preexisting runner override in a finally/cleanup path. Under
   temporary global limit 1, observe an ordinary occupied claim plus an admitted
   capacity=100 launch, the red global C/L header and gold c100 row. Observe a
   default-weight capacity=1 waiter parking until the others drain, then actually
   admitting, with quiet c1. Record admission, marker and rendered evidence, and stop
   only the smoke agents. Never bypass the launch approval mechanism to work around an
   unrelated handoff error.
3. Verify the two invalid authoring cases, positive capacity inheritance through a real
   serial continuation, and legacy-zero upgrade behavior. Include a live or
   production-path indexed refresh after a waiting marker disappears. Exercise both
   states of the sunset flag in isolated tests without changing the operator's
   persistent flag setting.
4. Review post-audit drift in the core/main repos, run the complete relevant contract
   suites, and run `just check-full` only via `sase_monitor` with TESTING/TESTED. Verify
   the rebuilt pinned extension, not an unrelated installed version. Investigate
   failures causally; don't relax test budgets or suppress failures to make this landing
   pass. Distinct unrelated failures go through the established task/epic triage
   workflow.

The original phase follow-ups have a separate audit disposition: libpython is
corroborated on sase-xv; schema and oracle mismatches now pass and require no tasks; the
historical settle-count overage was fixed in 638647694b and remaining CPU-budget
evidence is on sase-xc. The unrelated creator-handoff ImportError is tracked as medium
bug sase-106, separately from this repair. Preserve those outcomes for the eventual
parent landing note; do not turn unrelated fixes into child phases here.
