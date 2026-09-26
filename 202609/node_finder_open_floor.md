---
tier: epic
title: Cut the Node Finder open path under the 50 ms budget
goal:
  The 2,000-node Node Finder open benchmark stays under 50 ms p95, with the other
  approved budgets still green and navigation behavior unchanged.
parent_bead: sase-19i.7.3
phases:
  - id: snapshot-floor
    title: Remove open-path stalls and halve snapshot cost
    depends_on: []
    description:
      "snapshot-floor: keep snapshot construction in memory, read each agent role once,
      and bring the warm 2,000-node snapshot under 35 ms p50."
    size: medium
  - id: open-budget
    title: Pass the approved open p95 budget
    depends_on:
      - snapshot-floor
    description:
      "open-budget: make the official 2,000-node open benchmark pass at p95 under 50 ms
      without relaxing the harness or the other budgets."
    size: medium
proposed_by: bbugyi200.athena.sase-19i.7.3.land
create_time: 2026-09-26 12:33:50
status: wip
---

- **PROMPT:**
  [prompts/202609/node_finder_open_floor.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/node_finder_open_floor.md)
- **PARENT:**
  [202609/node_finder_remaining_budgets.md](https://github.com/sase-org/sase--plans/blob/main/202609/node_finder_remaining_budgets.md)

# Cut the Node Finder open path under the 50 ms budget

## Why this is still the epic

`sase-19i.7.3` requires p95 below 50 ms from `"` to first paint and below 16 ms for
broad refilter, narrow refilter, and highlight plus Tier 0 on the 2,000-node fixture.
Phases `sase-19i.7.3.1` (`d1b72cfe5`) and `sase-19i.7.3.2` (`221d72a13`) fused per-open
facets, read each grouping name once, hoisted fold filtering onto those facets, fused
the unmet-fold walk, and reused one roster tree index. That work is in the tree and its
differential tests pass their stated contracts. It does not meet the open budget.

Landing measurement on 2026-09-26,
`.venv/bin/pytest -s -m slow tests/ace/tui/bench_node_finder.py -q`:

- open: n=25 p50 94.75 ms, p95 569.11 ms, max 581.30 ms, budget 50 ms, fail
- narrow keystroke: p95 3.03 ms, pass
- broad keystroke: p95 14.27 ms, p50 12.30 ms, max 14.65 ms, pass
- highlight plus Tier 0: p95 0.63 ms, pass

The same app, timed in stages after startup warmup:

- `build_node_finder_snapshot`: steady samples about 69–110 ms, with separate stalls of
  442 ms, 571 ms, and 603 ms
- `NodeFinderModal` construction, including the empty-query filter: about 1.2 ms
- `push_screen` plus drain until the window exists: best 22.5 ms, p50 about 34 ms, worst
  58 ms, always 28 pump rounds

Inside a steady snapshot of 2,012 rows and 2,010 nodes: `describe_node_finder_row` about
11–16 ms (2,010 calls), fold filter about 8–9 ms, two `build_agent_tree` calls about
5–10 ms together, `_jump_candidate_targets` about 0.3 ms. Roughly 50 ms is still outside
those four buckets.

One cProfile of a 0.69 s snapshot attributed 0.43 s on the profiled thread to
`subprocess._execute_child`. A tracer around later builds saw `git remote -v` under
`build_clan_disk_snapshot` → bead-touch loading → `AgentIdentitySnapshot.current` →
`resolve_sync_targets`, and `git config --get remote.origin.url` on the bead-warmup
thread (`_warm_agent_bead_caches`). One 603 ms sample contained no `Popen`. Predicate
counts on the profiled snapshot: `identity` 12,062, `is_shell_member_role` 22,040,
`child_linkage` 16,120, `is_gate` 12,020, `is_monitor` 10,020, plan-chain suffix parse
22,040, `_ancestor_chain` 4,020.

The median open time is already about twice the budget before any stall. The p95 is the
second-worst of 25 samples, so two stalls fail the suite even if the median is fixed. Do
not relax `_SAMPLES`, `_percentile`, or the budgets, and do not stop the open test
before `build_node_finder_snapshot` returns.

## Already integrated

Commits on this branch after the epic was created, other than the two stitches above,
are axe Services panels, the receipt CLI, the deck panel mixin split, synthetic
model-surface tests, the receipt opportunity report, axe health badges, and proc-rename
test repair. None of them edit the Node Finder snapshot, modal, filter, preview, or
benchmark. No new caller needs the fused facet tables. Leave those commits as they are.

## Phase `snapshot-floor`

Make one warm snapshot cheap and free of synchronous git, disk, and bead-store
resolution.

Extend the per-open tables in `src/sase/ace/tui/actions/agents/_node_finder_snapshot.py`
so each agent pays once for the role and naming facts the row loop, fold filter, and
describer currently recompute: identity, monitor/gate/clan/proc/session/workflow-step
flags, child linkage, presented and display names, jumpable, title, kind label, and kind
accent. `describe_node_finder_row` remains the single-row contract. The snapshot must
consume the table (a batch helper is fine) instead of calling the single-row describer
in a way that re-parses plan-chain suffixes per property. Thread the same booleans into
`filter_agents_by_fold_state` when the caller already has them, and keep the existing
no-table path byte-for-byte for every other caller.

Confirm where the snapshot thread enters `subprocess`. If `build_node_finder_snapshot`
or anything it calls reaches clan disk snapshots, bead-touch loading,
`AgentIdentitySnapshot.current`, `resolve_sync_targets`, or `get_workspace_name`, stop
that read. The finder snapshot is an in-memory projection of the roster already on the
owner. If the git calls are only concurrent warmup, keep them off the snapshot thread
and off the first-paint drain measured by `tests/ace/tui/bench_node_finder.py`
(`_drain_to_first_paint` yields until the window exists, so a warmup joined there counts
against the 50 ms budget). Do not cache facet tables, trees, or owner state across
opens.

Re-profile a warm 2,000-node snapshot. Spend the remaining steady time on the
unaccounted passes: extra `build_agent_tree` walks, header `_evolve` copies, and
ancestor-chain rescans. Share work already computed for this open. Keep ordering,
hidden-reason precedence, dismissed exclusions, fold and query behavior, the `◆` here
row, and prefix-free hints exact.

Add a differential test that the batched description matches `describe_node_finder_row`
for clan, session, workflow-step, hidden-step, proc, gate, and monitor rows, and keep
the existing cross-mode, unmet, fold-filter, and tree-index tests green.

Acceptance for this phase: on the bench app's 2,000-node roster, after at least three
warmup builds, 12 further `build_node_finder_snapshot` calls have p50 under 35 ms. A
cProfile of one typical sample (under 50 ms) shows no `subprocess` frame. Record p50,
max, and the new stage split in the phase note. The open benchmark's p95 under 50 ms is
the next phase's gate; if 35 ms plus the drain still cannot clear it, that phase keeps
cutting this snapshot. Run the focused Node Finder tests. Do not run `just check-full`.

## Phase `open-budget`

Using the snapshot from the previous phase, make the official open benchmark pass. The
measured span stays snapshot, modal construction, mount, windowed list, and Tier 0. The
window is already 64 rows (`_OPTION_WINDOW_SIZE`). The drain's best case on this host is
about 22 ms, so the snapshot has to stay fast enough that snapshot plus drain has p95
under 50 ms. If it does not, profile those 28 pump rounds and remove synchronous work
that first paint does not need. Tier 0 for the highlighted row stays inside the timer.
Do not add a cross-open cache, disk read, or blocking await on the pump.

Run `tests/ace/tui/bench_node_finder.py` unchanged, all four cases, and put p50/p95 in
the phase note. Required: open p95 < 50 ms, broad and narrow refilter p95 < 16 ms,
highlight plus Tier 0 p95 < 16 ms. Broad passed here at 14.27 ms; touch that path only
if this run goes red. A multi-hundred-millisecond open sample is a failure of this phase
even when the median looks fine, because p95 is the second-worst of 25 samples.

Run the focused Node Finder tests and `just check`. Do not run `just check-full`.
Rendered output should not change. If a targeted Node Finder PNG changes, inspect it;
pre-existing PNG failures on the base tree belong to epic `sase-19i` and are not a new
regression. Symvision entries for `sase-19x.4` belong to epic `sase-19x` and are not
this phase.

## Left for the resumed landing

Do not close `sase-19i.7.3`, do not retarget `--epic-symbol` lines, and do not mark
`plan:202609/node_finder_remaining_budgets.md` done. After this plan lands, that epic's
land agent resumes. These notes are context, not extra scope:

- `sase-19i.7.3.1` follow-up on three stale `sase-19x.4` Symvision symbols is already
  owned by active epic `sase-19x`.
- `sase-19i.7.3.1` follow-up that `sase tool run check` rejected an unknown `receipt`
  field was unverified in this landing.
- `sase-19i.7.3.2` follow-up that a cold `just check` exceeds one turn is operational
  guidance for verification, not a product change.
- The broad-query follow-up is not reproduced here (p95 14.27 ms).

## Landing handoff

`parent_bead` is `sase-19i.7.3`. Closing that epic, the Symvision pass, and the parent
plan-file status update are not phases.
