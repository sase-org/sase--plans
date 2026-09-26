---
tier: epic
title: Finish Node Finder open and broad-query budgets
goal:
  The 2,000-node Node Finder passes the unchanged first-paint and refilter p95 budgets
  with exact navigation behavior.
parent_bead: sase-19i.7.3.3
phases:
  - id: snapshot-tail
    title: Bound the 2,000-node Node Finder snapshot tail
    depends_on: []
    size: medium
    description:
      "snapshot-tail: profile and remove repeated grouping and row passes until a warm
      snapshot leaves room for first paint."
  - id: paint-and-broad
    title: Pass first-paint and broad-refilter budgets
    size: medium
    depends_on:
      - snapshot-tail
    description:
      "paint-and-broad: trim first-paint pump work and broad refilter while preserving
      navigation, then pass every official budget."
proposed_by: bbugyi200.athena.sase-19i.7.3.3.land
create_time: 2026-09-26 15:55:40
status: wip
---

- **PROMPT:**
  [prompts/202609/node_finder_open_and_broad_tail.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/node_finder_open_and_broad_tail.md)
- **PARENT:**
  [202609/node_finder_open_floor.md](https://github.com/sase-org/sase--plans/blob/main/202609/node_finder_open_floor.md)

# Finish the Node Finder open and broad-query budgets

## Why this remains open

This is the unfinished acceptance from epic `sase-19i.7.3.3`, not a new feature. Its two
closed phases landed in `f079c0af5d` and `64b15bdf52`. The current snapshot uses
per-open facet tables, a plain-row description path, keep-all fold optimization,
memoized unmet chains, and reason-set interning; the tests cover the row descriptions
and fold behavior. Those cuts do not yet pass the approved performance gates.

On HEAD `f583cd5097`,
`.venv/bin/pytest -s -m slow tests/ace/tui/bench_node_finder.py -q` measured 25
post-warmup samples per case:

- open: p50 77.47 ms, p95 102.19 ms, max 407.68 ms, **fails** the <50 ms budget;
- broad refilter: p50 12.29 ms, p95 17.81 ms, max 17.83 ms, **fails** the <16 ms budget;
- narrow refilter: p95 1.03 ms, passes <16 ms;
- highlight plus Tier 0: p95 0.57 ms, passes <16 ms.

The open test times `build_node_finder_snapshot`, `NodeFinderModal` construction,
`push_screen`, and pump drain until the first window and Tier 0 appear. It does not wait
for Tier 1. The bench fixture has 2,000 agents, and the test's 30 samples include five
warmups. Keep `_SAMPLES`, `_percentile`, fixture size, timers, and budgets unchanged.
The child phase notes report prior steady snapshot around 50 ms and a further 25 ms
drain, plus GC stalls; measure again because the current tree and load have changed.

Commits after this epic started do not replace the finder path. `d5fc75864f` changed
gate-turn imports in `node_finder_rendering.py` and `Agent.is_proc_shell` to the
`NAMED_PROC` agent type. The remaining work must retain correct gate, monitor, proc,
session, and workflow-step classification. No cross-open owner or facet cache is
allowed: opening must reflect current state. The parent `sase-19i` already owns five
Node Finder PNG failures noted on its bead, and `sase-1ab` owns the current unrelated
private-import Symvision failure. Neither is acceptance for this child.

## Phase `snapshot-tail`

Profile 12 or more warm 2,000-node `build_node_finder_snapshot` calls with timings for
roster/facets, `AgentPanelGroup.from_agents`, `agents_for_panel`, `build_agent_tree`,
row creation, omission/remap, header counts, and total. Capture one representative
`cProfile` and a stall sample if present. Verify that the snapshot thread does not run
subprocess, synchronous disk, or bead-store resolution. Do not attribute an unrelated
concurrent warmup thread to the snapshot.

Use the profile to remove the dominant repeat work in
`src/sase/ace/tui/actions/agents/_node_finder_snapshot.py` and its presentation-only
grouping helpers. In particular, inspect the per-panel tree/grouping work, the
`enclosing` map built from group agent indices, facet dictionaries keyed by `id(agent)`,
row allocations and replacement passes, and ancestor walks. Share data only within one
open; keep the normal grouping and fold paths exact for other callers. Do not move
shared domain semantics into a new Python implementation when they belong in the Rust
core.

Preserve tree order, all hidden-reason precedence, dismissed exclusions,
panel/banner/fold/query/I-hidden behavior, the `◆` here row, hint assignment, counts,
and navigation identities. Keep or expand differential tests for clan, session,
workflow-step, hidden-step, named proc, gate, and monitor rows and for collapsed/grouped
versus expanded views. The output must be equivalent to the existing snapshot; a fast
path may not silently skip rows or delay the snapshot past the benchmark timer.

Acceptance: record the exact stage split, p50/p95/max across at least 12 warm builds,
and the cProfile evidence on the phase bead. Aim for snapshot p95 below 25 ms to leave
room for first-paint drain; if it remains higher, continue reducing the measured hotspot
before closing this phase. Run focused snapshot/model tests. The final <50 ms open p95
is the next phase's gate.

## Phase `paint-and-broad`

After `snapshot-tail`, run the unchanged four-case benchmark and profile the remaining
open span. If snapshot plus modal construction is fast but open is over 50 ms, time the
existing 28-ish pump rounds and remove synchronous first-paint work in
`NodeFinderModal`/Textual integration. Keep the first 64-row window and Tier 0 in the
measured span; Tier 1 remains debounced and outside it. Avoid cross-open caches,
blocking awaits, disk reads, or hiding work behind an early timer stop. Preserve the
real first-paint predicate.

Profile `_apply_refilter` separately into `filter_node_finder`, `_rebuild_options`
(including guides and OptionList mutations), chrome, and Tier 0. The broad query `bench`
matches the whole tree. Cut the measured >16 ms p95 path without weakening scoring,
order, context rows, hints, or highlight behavior. Reuse immutable per-modal/per-view
data only when its invalidation is proven; preserve live query changes and the
latest-wins callback behavior. Recheck the `NAMED_PROC`, gate, and monitor shapes
touched by later work.

Acceptance: the unchanged `tests/ace/tui/bench_node_finder.py` passes all four cases
with 25 measured samples: open p95 <50 ms, narrow and broad refilter p95 <16 ms,
highlight plus Tier 0 p95 <16 ms. Record p50/p95/max and stage timings on the phase
bead. Run focused Node Finder model/modal/snapshot/navigation tests and `just check`
(through `sase tool run check` where available). Do not run `just check-full`. If the
existing unrelated `sase-1ab` Symvision error still blocks `just check`, name the run
and failure rather than claiming a pass. Inspect any targeted Node Finder PNG changes;
the parent epic owns its previously recorded golden failures.

## Landing handoff

The `parent_bead` link returns control to `sase-19i.7.3.3`'s land agent for a fresh
official benchmark, post-child drift review, follow-up reconciliation, symbol cleanup,
close, and plan status update. These are not child phases. The same parent-chain landing
rules then apply to `sase-19i.7.3`, `sase-19i.7`, and `sase-19i` only when each one's
acceptance is actually complete.
