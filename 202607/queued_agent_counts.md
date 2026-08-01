---
tier: tale
title: Distinguish and scope queued-agent counts in the Agents tab
goal: Queued agents use a distinct color and appear as scoped Q counts in every tribe-panel
  and clan aggregate.
create_time: 2026-07-24 18:57:39
status: done
---

- **PROMPT:** [prompts/202607/queued_agent_counts.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/queued_agent_counts.md)
- **AGENTS:**
  - [bbugyi200.athena.jq.f0.f1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.jq.f0.f1/README.md)
  - [bbugyi200.athena.jq.f0.f1--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.jq.f0.f1.md#member-code)
- **COMMITS:**
  - [ca348d7](https://github.com/sase-org/sase/commit/ca348d7034c1887b600464b913d8b29cba304ef9) — feat(ace): show globally queued agent counts

# Distinguish and scope queued-agent counts in the Agents tab

## Goal

Make runner-capacity queue pressure immediately distinguishable from ordinary waiting in the `sase ace` Agents tab, and
expose that pressure in every tribe-panel and clan aggregate that already uses the compact agent-count chip.

After this change:

- The global header continues to render a positive queue as `<Q> queued`, but its number uses a dedicated bright-pink
  queue style (`bold #FF87D7`) instead of the waiting metric's purple.
- Compact aggregate chips use `Q` for the same queue concept and place it between running and waiting, for example
  `[R3 Q2 W4 D5]`.
- `Q0` is suppressed, like every other zero-valued compact metric.
- A globally queued agent remains part of the normal `W` status total. Queue admission is orthogonal to the agent's
  `WAITING` lifecycle status, matching the global header's existing simultaneous `queued` and `waiting` counts.

## Queue semantics and invariants

Use one public, pure predicate in the existing runner-slot presentation model as the authoritative definition of a
globally queued agent. It must preserve the definition already used by `RunnerCapacitySnapshot.global_cap_queue_count`:

- the row participates in user-agent runner slots;
- the row is live (`pid` is present), has requested a slot, and is currently `WAITING`; and
- the wait comes from the configured global cap (`wait_runners_explicit` is false), not an explicit per-agent runner
  barrier.

Dependency-only waits, time waits, explicit `runners <= N` / drain barriers, dead rows, yielded questions, serial child
rows that do not participate in runner slots, and synthetic clan containers must not contribute to `Q`. Reuse the
predicate in the global capacity calculation and all scoped aggregates so their definitions cannot drift.

Scoped counts must operate only on the already-loaded in-memory rows represented by that aggregate, deduplicate the same
concrete agent when it is reachable through both a container and a flat row, and preserve existing status, unread, lane,
family, and clan projection semantics. No renderer may infer queue membership from display text, queue position, panel
selection, or the runner-limit value.

## Implementation

### 1. Establish shared queue semantics and presentation style

- In `src/sase/ace/tui/models/agent_runner_slots.py`, promote/refactor the current implicit-global-wait test into a
  reusable public predicate and make `refresh_runner_slot_context()` use it for `global_cap_queue_count`. Keep FIFO
  eligibility/position behavior unchanged; an explicit barrier may participate in the eligible queue detail without
  contributing to the global `Q` count.
- In `src/sase/ace/tui/agent_count_chip.py`, add a named canonical queued-count style with value `bold #FF87D7`, add the
  `queued`/`Q` metric to the shared formatter between `running`/`R` and `waiting`/`W`, and retain the formatter's zero
  suppression and selected-panel chrome override behavior. The queue digit keeps its semantic color even when the panel
  is selected; the `Q`, brackets, and separators follow the same neutral/selected chrome rules as the other ordinary
  metrics.
- Reuse that named style for the positive global queue number in `src/sase/ace/tui/widgets/agent_info_panel.py`. Do not
  alter the global strip's wording, queue omission rule, denominator style, running-capacity thresholds, other metric
  colors, or stable/countdown cache behavior.

### 2. Carry `queued` through existing aggregate snapshots

- Extend the existing status-count value objects with a default-zero `queued` field: `ClanStatusCounts`, the internal
  agent summary/lane count projection, `AgentPanelCounts`, `TribeStatusCounts`, and `MemberRosterStatusCounts`.
- Count queue membership with the shared runner-slot predicate while retaining the existing projection/deduplication
  boundaries:
  - global agent summary/lane projections count each represented concrete queued agent once;
  - clan aggregates include queued members represented by the clan, including the concrete rows behind family
    containers, without counting the synthetic clan row;
  - tribe snapshots expose the scoped queue count in the panel header and in aggregate clan/family roster-unit chips;
    and
  - panel border-title counts expose the queue count for that panel slice only.
- Keep `queued` independent of `waiting`: incrementing `Q` must not decrement or reclassify `W`, change aggregate status
  priority, change lane totals, or affect unread/done replacement rules.
- Preserve the current pure, O(rows), in-memory paths. Do not add file reads, config reads, polling, awaits, a new
  refresh route, or a full-list rebuild trigger.

### 3. Render `Q` consistently on tribe and clan surfaces

- Pass `queued` into `format_agent_count_chip()` at every scoped aggregate call site so the shared format appears
  consistently in:
  - expanded and collapsed tribe-panel border titles, including selected-panel chrome;
  - whole-tribe detail headers and aggregate clan/family member rows;
  - synthetic clan rows in the Agents list and clan detail headers/roster aggregates; and
  - other existing clan-count consumers such as the clan cleanup chooser, so the same clan cannot show conflicting chips
    on different Agents-tab surfaces.
- Preserve all existing status tokens and canonical ordering around the inserted token. The full order becomes
  `[S… R… Q… W… F… U… D…]`, with absent metrics omitted.
- Do not add `Q` to ordinary leaf-agent rows, change the verbose per-agent runner-slot detail (`eligible #N of M`,
  `runners <= N`, drain barrier), or display explicit runner waits as globally queued.

## Tests

Add or strengthen focused coverage at each boundary:

- Runner-slot model tests prove the public queue predicate and global snapshot agree for implicit live waiters, and
  reject explicit barriers, dependency-only waits, dead/yielded rows, non-slot rows, and duplicate/container
  projections.
- Shared count-chip tests cover the new canonical `[S… R… Q… W… F… U… D…]` order, multi-digit `Q`, zero suppression,
  bright-pink queue-digit style, neutral chrome, and selected-panel chrome override.
- Global header tests assert the positive queued number uses `bold #FF87D7`, differs from waiting purple and done cyan,
  remains conditional on a positive count, and leaves runner numerator/limit styling unchanged.
- Aggregate model tests cover a mixed scope containing an implicit global-cap waiter, an explicit runner barrier, and an
  ordinary dependency waiter. Assert that only the implicit waiter contributes to `queued`, that it still contributes to
  `waiting`, that container/flat representations are deduplicated, and that tribe/clan/panel totals remain scoped.
- Rendering tests assert `Q` appears in the correct position and color in tribe-panel titles (selected and unselected),
  tribe detail/unit chips, clan list/detail chips, and relevant clan-family/cleanup consumers; also assert zero-queue
  and explicit-barrier cases omit `Q`.
- Update or add a representative Agents PNG fixture with a real implicit global-cap waiter inside a clan/tribe scope so
  one end-to-end state visibly exercises the distinct global queue color plus scoped `Q` in the panel title, clan row,
  and detail header. Regenerate only affected goldens and audit pixel/SVG changes for the intended queue token, color,
  and resulting horizontal reflow.

## Validation

Because this is an ephemeral SASE workspace, run:

1. `just install`
2. Focused runner-slot, count-chip, info-panel, panel-title, tribe-summary/rendering, clan-summary/rendering,
   parallel-family, cleanup-clan, render-cache, and representative visual tests.
3. The complete ACE PNG visual snapshot suite, auditing changed goldens before accepting them.
4. `just check`

If a repository-wide gate fails for a demonstrably unrelated pre-existing condition, record the exact failure and still
run all remaining in-repository formatting, lint/type, unit, and visual gates independently. Do not broaden this tale
into an unrelated repair.

## Acceptance criteria

- A positive global queue count is bright pink and visually distinct from every other count on the header.
- Every affected tribe-panel and clan aggregate shows `Q<n>` for its own implicit global-cap waiters, in
  running/queued/waiting order and with the same bright-pink queue digit.
- Explicit runner barriers and non-runner waits do not contribute to `Q`.
- `Q` and `W` may both include the same queued agent; all pre-existing status totals, lane totals, aggregate statuses,
  queue-detail annotations, and conditional omission behavior remain correct.
- No new I/O or asynchronous work reaches a render/navigation path, and the full required validation is clean apart from
  any explicitly documented unrelated baseline failure.
