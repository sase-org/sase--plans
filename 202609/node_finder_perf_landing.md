---
tier: epic
title: Finish Node Finder performance budgets
goal:
  The Agents Node Finder meets its approved 2,000-node p95 open, refilter, and highlight
  budgets without losing any jump targets or changing navigation behavior.
parent_bead: sase-19i
phases:
  - id: snapshot-filter
    title: Bound snapshot and broad-query filter work
    size: medium
    description:
      "snapshot-filter: remove data-scaled repeated tree walks in the Node Finder
      snapshot and pure filter while preserving every reachable row, ancestor, hidden
      reason, and hint."
    depends_on: []
  - id: modal-paint
    title: Meet first-paint, broad-query, and highlight budgets
    size: medium
    description:
      "modal-paint: reduce modal list and preview work on the Textual pump, then enforce
      the original 2,000-node p95 budgets with representative benchmarks."
    depends_on:
      - snapshot-filter
proposed_by: bbugyi200.athena.sase-19i.land
create_time: 2026-09-26 06:03:20
status: wip
---

- **PROMPT:**
  [prompts/202609/node_finder_perf_landing.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/node_finder_perf_landing.md)
- **PARENT:**
  [202609/agents_node_finder.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_node_finder.md)

# Finish Node Finder performance budgets

## Why this remains epic work

The six original `sase-19i` phases are closed and the feature is wired. The approved
plan nevertheless requires p95 below 50 ms for `"` to first paint and below 16 ms for
both refiltering and highlight plus Tier 0 at 2,000 nodes. On the current tree,
`.venv/bin/pytest -s -m slow tests/ace/tui/bench_node_finder.py -k open -q` measured
open p50 166.50 ms and p95 436.58 ms. The same bench's highlight test measured p95 19.49
ms. The final narrow-query keystroke test passed at p95 0.40 ms, but phase `sase-19i.5`
measured about 190 ms for a broad query and reported that the current latest-wins
coalescing still blocks the pump on broad results.

This plan covers only the unfinished performance work. Keep the current Node Finder
scope, key behavior, snapshot stability, identity-based reveal, query clear, hidden-by-I
behavior, and two-tier preview intact. Changes after the original plan added CardBlock
and Agent Session Reply presentation in the detail pane; the Node Finder's plain-text
preview remains a separate surface, so reuse shared presentation helpers only where they
reduce real duplication without increasing first-paint work.

## Phase `snapshot-filter`: Bound snapshot and broad-query filter work

Profile the 2,000-node fixture before editing, separating pre-hide roster, tree
projection, omission, header counts, fuzzy scoring, and ancestor retention. The current
snapshot computes each header's counts by scanning all nodes and calling
`_is_descendant` for each pair; `filter_node_finder` similarly scans kept nodes for
every retained header. Replace those repeated ancestry walks with linear or near-linear
ancestor accumulation. Reuse computed ancestry and metadata within one snapshot or
filter call where helpful, without caching live owner state across modal opens.

Keep the original ordering, panel and grouping headers, `◆` here row, hidden reason
precedence, dismissed exclusions, and prefix-free hint allocation. Add focused
regression tests with nested clan/session/workflow rows, multiple headers, folds,
query-hidden rows, and I-hidden rows to prove counts and ancestors remain exact.

Rerun the 2,000-node open benchmark and measure a broad query with many matches as well
as a narrow query. Record the per-stage timings. The pure snapshot and filter must leave
enough of the 50 ms first-paint budget for modal construction and Tier 0.

## Phase `modal-paint`: Meet first-paint, broad-query, and highlight budgets

Profile `NodeFinderModal._rebuild_options`, broad-result `OptionList` churn,
`_move_cursor`, and `_paint_preview` on the same fixture. Preserve immediate Tier 0
highlight, 150 ms off-pump Tier 1, stable hints, click and keyboard behavior, and
Tab/Enter flushing of pending refilters. Reuse rendered row parts or apply a bounded
list update when possible; if a 2,000-row rebuild cannot meet the budget, use a windowed
or virtualized list that retains every row as a reachable hint and jump target. Avoid
synchronous disk access or blocking awaits on the pump.

Improve `bench_node_finder.py` so the open test measures actual modal construction and
first paint, and the keystroke test covers a broad matching query in addition to the
currently narrow converged burst. Keep the narrow and highlight paths covered. Require
the approved p95 budgets: open < 50 ms, broad/narrow refilter < 16 ms, and cursor
highlight plus Tier 0 < 16 ms on the 2,000-node fixture. Repeat measurements enough to
distinguish a stable regression from host contention and include observed numbers in the
phase note. Run touched-area tests, targeted visual snapshots if rendered output changes
(inspect every golden diff), and `just check`. Do not run `just check-full`.

## Landing handoff

The child plan's `parent_bead` points to `sase-19i`. After this child lands, its land
agent resumes the interrupted `sase-19i` review, triages every original
`PROPOSED FOLLOW-UP:` note, confirms later-commit integration, and closes the epic only
when the performance budgets and normal landing checks pass. The parent close, Symvision
pass, and parent plan status update are not phases of this plan.
