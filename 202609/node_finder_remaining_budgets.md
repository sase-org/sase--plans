---
tier: epic
title: Finish the remaining Node Finder open and broad-query budgets
goal: The 2,000-node Node Finder meets the approved first-paint and refilter p95 budgets
  without changing navigation behavior.
parent_bead: sase-19i.7
phases:
- id: snapshot-facets
  title: Bound snapshot construction for first paint
  depends_on: []
  description: 'snapshot-facets: profile and fuse repeated per-agent snapshot work
    while preserving every reachable row and hidden reason.'
  size: medium
- id: modal-budget
  title: Finish open and broad-query p95 budgets
  depends_on:
  - snapshot-facets
  description: 'modal-budget: optimize any remaining modal and broad-filter costs,
    verify every approved budget, and protect behavior with focused tests.'
  size: medium
proposed_by: bbugyi200.athena.sase-19i.7.land
create_time: 2026-09-26 10:26:41
status: wip
bead_id: sase-19i.7.3
---

- **PROMPT:** [prompts/202609/node_finder_remaining_budgets.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/node_finder_remaining_budgets.md)
- **PARENT:** [202609/node_finder_perf_landing.md](https://github.com/sase-org/sase--plans/blob/main/202609/node_finder_perf_landing.md)
- **BEAD:** [sase-19i.7.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19i/sase-19i.7.3.md)

# Finish the remaining Node Finder budgets

## Current evidence

The approved plan `plan:202609/node_finder_perf_landing.md` requires p95 below 50 ms
from `"` to first paint, and below 16 ms for both refiltering and highlight plus Tier 0
on a 2,000-node fixture. The closed `sase-19i.7.1` stitch `f62604e71` replaced repeated
ancestry scans; the closed `sase-19i.7.2` stitch `0b55415cd` windowed the modal list and
improved the production-path benchmark. Those changes are present, and 59 focused Node
Finder model, snapshot, modal, and preview tests pass at `7cdde2b32`.

Fresh measurements on that tree, with 25 post-warmup samples, still fail open (p50
101.07 ms, p95 444.49 ms against 50 ms) and narrowly fail a broad matching refilter (p95
16.16 ms against 16 ms). Narrow refilter (p95 1.39 ms) and cursor highlight plus Tier 0
(p95 0.98 ms) pass. The open result has a long tail, so profile individual stages and
distinguish application work from host contention; do not relax the approved budget or
change the benchmark to measure a less complete path.

The later CardBlock and session presentation commits affect the Agents detail pane; the
Node Finder retains a separate plain-text preview. The `0b55415cd` stitch already
integrates the intervening agent-tree and panel code. Two commits after it affect tool
receipts and finalization, not Node Finder code. Keep later navigation and presentation
behavior intact.

## Phase `snapshot-facets`

Profile the current open benchmark end to end: roster and fold projection, grouping,
panel tree, hidden-reason and unmet-fold classification, row description, hint
allocation, modal construction, Textual mount, window materialization, and Tier 0.
Record per-stage distributions, not only one sample. The previous phase estimated about
68 ms of snapshot work and a 20–25 ms mount floor; verify those estimates on the current
tree.

Remove repeated data-scaled work in `build_node_finder_snapshot` by computing per-agent,
per-open facets once where that preserves semantics: identity, parent/depth, role/panel
keys, fold keys, group membership, and hidden reasons. Reuse a coherent lookup across
grouping/fold/unmet and row construction rather than re-deriving plan-chain predicates
for each pass. Keep all caches scoped to one snapshot; live owner state must be read
afresh on every open. Keep panel and group ordering, the `◆` here row, dismissed
exclusions, fold/query/I-hidden precedence, and prefix-free hints exact. Add a focused
cross-mode regression where the fused computation could change counts, ancestors, or
hidden reasons. Measure the remaining pre-mount cost and report it for the next phase.

## Phase `modal-budget`

Use the first phase's profile to remove any remaining synchronous first-paint work in
the modal, list window, and Tier 0 path. Keep every node reachable through its hint and
jump target; preserve Tab/Enter flushing, immediate Tier 0 highlight, 150 ms off-pump
Tier 1, and identity-based reveal. Optimize broad-query scoring or window updates enough
to leave a stable margin below 16 ms without changing fuzzy ordering, context rows, or
query-clear behavior. Avoid cross-open snapshot caching, disk access, and blocking
awaits on the Textual pump.

Run `tests/ace/tui/bench_node_finder.py` with all four budget cases on the 2,000-node
fixture and enough repeated samples to distinguish stable cost from loaded-host
outliers. Require open p95 < 50 ms, broad and narrow refilter p95 < 16 ms, and highlight
plus Tier 0 p95 < 16 ms. Record p50/p95 and per-stage findings in the phase note. Run
focused Node Finder tests and `just check`; do not run `just check-full`. Run targeted
Node Finder visual snapshots if rendered output changes, inspect generated diffs, and
approve only intentional changes. Five of seven targeted PNG tests already fail on the
pre-phase tree; that issue is recorded on active parent epic `sase-19i` and must not be
mistaken for a new regression from this plan. `just symvision` currently fails on three
stale `sase-19x.4` entries owned by active epic `sase-19x`; that unrelated failure is
documented on its bead.

## Landing handoff

The child epic's `parent_bead` points to `sase-19i.7`. After this plan lands, its land
agent resumes the budget review and the original epic landing. Do not add a phase for
closing either epic, running Symvision after close, or marking either plan file done.
