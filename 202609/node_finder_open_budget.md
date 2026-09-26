---
tier: epic
title: Pass the Node Finder open p95 budget
goal: The unchanged 2,000-node Node Finder open benchmark stays under 50 ms p95, with
  the other approved budgets still green and navigation unchanged.
parent_bead: sase-19i.7.3.3.3
phases:
- id: snapshot-body
  title: Cut the warm snapshot body
  depends_on: []
  size: medium
  description: 'snapshot-body: remove the dominant in-snapshot cost until a same-process
    warm build is at least 15 ms cheaper or its p95 is under 25 ms.'
- id: open-budget
  title: Pass the official open benchmark
  depends_on:
  - snapshot-body
  size: medium
  description: 'open-budget: finish the first-paint drain and any leftover snapshot
    cost until the unchanged four-case benchmark passes.'
proposed_by: bbugyi200.athena.sase-19i.7.3.3.3.land
create_time: 2026-09-26 18:21:07
status: wip
bead_id: sase-19i.7.3.3.3.3
---

- **PROMPT:** [prompts/202609/node_finder_open_budget.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/node_finder_open_budget.md)
- **PARENT:** [202609/node_finder_open_and_broad_tail.md](https://github.com/sase-org/sase--plans/blob/main/202609/node_finder_open_and_broad_tail.md)
- **BEAD:** [sase-19i.7.3.3.3.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19i/sase-19i.7.3.3.3.3.md)

# Pass the Node Finder open p95 budget

## Measured gap

Phases `sase-19i.7.3.3.3.1` (`4be908bb66`) and `sase-19i.7.3.3.3.2` (`0a74b4f25c`) are
in the tree. They added per-open facet tables, the plain-row describer, the keep-all
fold skip, shared walk order, skipped banner member indexes, the `filter_tree_rows`
all-match early-out, the per-modal view memo (cap 8), and the identical-rebuild skip
with its live-highlight guard. Focused snapshot, model, and modal tests cover those
paths. Keep that behavior exact.

The epic acceptance is still open. On `acbd5999ad`, the unchanged command
`.venv/bin/pytest -s -m slow tests/ace/tui/bench_node_finder.py -q` reported 25 measured
samples:

- open: p50 73.42 ms, p95 113.02 ms, max 427.48 ms, over the 50 ms budget
- keystroke-dispatch: p95 0.01 ms, under 16 ms
- keystroke: p95 0.96 ms, under 16 ms
- keystroke-broad: p95 5.46 ms, under 16 ms
- highlight: p95 0.55 ms, under 16 ms

The open p50 is itself over 50 ms. The bench's p95 is the second-worst of 25 samples, so
a single GC spike is not what fails the gate. The body of the distribution does. Closing
because a clean base tree also misses 50 ms is how the previous phases stopped short.
This plan's second phase closes only when the official benchmark passes.

A GC-paused stage split on the same harness (noisier host, 10 warm samples, 2,204 rows)
was: snapshot p50 about 90 ms, of which facets were about 11 ms, `build_agent_tree`
about 10 ms, `AgentPanelGroup.from_agents` about 0.7 ms, and the unwrapped roster, row,
and header remainder about 64 ms; modal init about 2.7 ms; drain p50 about 28 ms across
exactly 28 `asyncio.sleep(0)` rounds. `on_mount` was about 10 ms of that drain and
`_rebuild_options` about 2.4 ms. Quieter notes from the previous phases put a warm
snapshot near 30–40 ms and the drain floor near 23–25 ms, with open p50 near 63 ms.
Re-measure on an idle page. Use the split as a map, and use a same-process before/after
as the quota.

A cProfile taken while the Agents page was alive charged seconds to prompt-panel clan
aggregation, bead-touch scans, and glossary loads. Those frames were not shown as
callees of `build_node_finder_snapshot`. Charge the snapshot only for frames
`print_callees` attributes to it. If that list includes disk, subprocess, or bead-store
work, remove it: one open reads current in-memory owner state and does not scan those
stores.

## Integration already reviewed

Commits after `4be908bb66`, excluding this epic's own stitches, are `4e7262d675` (stash
trash), `74c89a6385` (memory docs), `e95241543d` (scrollbar helper), `f7886b1a64`
(receipt report), and `acbd5999ad` (agent clan neighbors). None edit the node-finder
snapshot, modal, filter, or benchmark. Clan neighbors adds hood indexing for the prompt
panel and member jump. The finder has no new caller to retarget. Confirm only that the
open path did not start calling that loader.

## Phase `snapshot-body`

Profile at least 12 warm `build_node_finder_snapshot` calls on the 2,000-node bench
roster with GC enabled, after startup work has gone idle. Time facets,
`build_agent_tree`, `from_agents`, the row loop, and header counts separately. Save one
cProfile and `print_callees` for `build_node_finder_snapshot`.

Cut the dominant frames inside
`src/sase/ace/tui/actions/agents/_node_finder_snapshot.py` and the presentation helpers
that profile shows it actually calls (`agent_groups` tree and keys, row construction,
header counts). The earlier 1–2 ms ideas (a banner-skeleton emitter, fact-threaded
grouping keys) are optional only when the profile still ranks them. Prefer removing
repeated per-row walks and allocations.

Preserve tree order, hidden-reason precedence, dismissed exclusions, panel, banner,
fold, query, and `I`-hidden behavior, the `◆` here row, hint assignment, counts, and
navigation identities. Share data only within one open. Keep the normal grouping and
fold paths exact for other callers. Keep `NAMED_PROC`, gate, and monitor classification
exact. Leave `tests/ace/tui/bench_node_finder.py` byte-for-byte unchanged, including
`_SAMPLES`, `_percentile`, the fixture size, the timers, and the budgets.

Acceptance: on one process, at least 12 warm snapshots after the change have a p50 at
least 15 ms below the same harness measured before the change, or a warm snapshot p95
under 25 ms. Record both distributions and the cProfile on the phase bead. Run the
focused snapshot, grouping, and fold tests. The official open p95 belongs to the next
phase. A phase note that cites diminishing returns without one of those two measurements
is not acceptance.

## Phase `open-budget`

Run the unchanged four-case benchmark. When open p95 is still at or above 50 ms, cut the
first-paint drain. It is 28 pump rounds with a floor near 23–28 ms. `on_mount` is about
10 ms of that, and the 64-row rebuild is only about 2 ms, so the rest is Textual
compose, mount, and layout before the window exists. Remove synchronous work on that
path in `NodeFinderModal` and the widgets it composes. Keep the work inside the measured
span: the timer still covers snapshot, modal construction, `push_screen`, and the pump
until the first window and Tier 0 are visible. Tier 1 stays behind its 150 ms debounce.
Keep the real first-paint predicate (`is_mounted`, option count equal to the window,
Tier 0 painted).

Keep the per-modal view memo and the identical-rebuild skip, including the guard that a
cursor move followed by a refilter still recenters. Keep broad, narrow, and highlight
behavior, scoring, order, context rows, and hints.

Acceptance: `.venv/bin/pytest -s -m slow tests/ace/tui/bench_node_finder.py -q` passes
all four cases on 25 measured samples (open p95 under 50 ms, keystroke-dispatch,
keystroke, keystroke-broad, and highlight p95 under 16 ms). Record p50, p95, and max on
the phase bead. Run the focused Node Finder model, modal, snapshot, and navigation
tests, and `just check` through `sase tool run check` when that wrapper is available.
Leave `just check-full` unrun. Close this phase only after that benchmark command
exits 0. A miss that also appears on a tree without the phase diff stays in this phase
until the benchmark passes. Leave the budget numbers in the test file unchanged.

## Handoff

`parent_bead` returns this work to `sase-19i.7.3.3.3`'s land agent. That agent repeats
integration review, follow-up reconciliation, symbol cleanup, the epic close, and the
plan-file status update. Those steps are not phases here.

Follow-ups already triaged on that landing, so this epic should not refile them:

- The open miss is this plan.
- The 1–2 ms snapshot ideas are in `snapshot-body` when a profile still shows them.
- `AgentSessionShellGateWire` and `AgentSessionShellMonitorWire` collection failures
  still reproduce at `acbd5999ad` in `tests/ace/tui/models/test_gate_rows.py` and
  `test_monitor_rows.py`. Active epic `sase-1ab` already owns them (notes #2 and #4).
- `sase init memory --check` exits 0 on this tree, so the earlier memory-drift proposal
  is not reproduced.
