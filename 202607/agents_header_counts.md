---
tier: tale
title: Consolidate Agents-tab running capacity and status counts
goal: The Agents header reports running capacity and optional queue depth in one accurate,
  compact status strip.
create_time: 2026-07-24 18:06:28
status: done
---

- **PROMPT:** [202607/prompts/agents_header_counts.md](prompts/agents_header_counts.md)

# Consolidate Agents-tab running capacity and status counts

## Goal

Replace the Agents-tab header's separate runner-capacity chip and status chip with one coherent count strip. The strip
must use the same visible running count for both the running status and the capacity numerator, retain the
configured/effective runner limit as the denominator, and show the global runner queue count only when that count is
non-zero.

The example state must change from:

```text
40  [5/10 · 0 queued]  [5 running · 4 waiting · 31 done]
```

to:

```text
40  [5/10 running · 4 waiting · 31 done]
```

A saturated state with queued agents should read like:

```text
40  [10/10 running · 2 queued · 4 waiting · 24 done]
```

## Current behavior and constraints

- `AgentInfoPanel` currently renders `_runner_slots_in_use/_runner_limit` in one always-visible chip, including
  `0 queued`, then renders `_running_count` again in a second metric strip.
- The two numerators intentionally come from different projections today: `RunnerCapacitySnapshot.slots_in_use` counts
  live slot holders, while `_running_count` is the visible, deduplicated agent-lane running metric produced by
  `agent_lane_status_counts()`. The requested header must use the latter value, even when those projections differ.
- Keep `RunnerCapacitySnapshot.slots_in_use` and per-agent slot occupancy intact for admission/queue detail rendering
  elsewhere. Only the Agents header should stop consuming that value.
- Continue using the loader-precomputed effective `max_running_agents` value and `global_cap_queue_count`; do not add
  configuration reads, artifact scans, or other I/O to render/countdown paths.
- A positive queue count should be rendered whenever the cached snapshot says it is positive. Saturation is the normal
  source invariant, but the presentation should not hide a truthful positive count during a transient refresh snapshot.
- Preserve the one-line, cached `layout=False` rendering and countdown-only fast path described by the TUI performance
  guidance.

## Implementation

1. Refactor `src/sase/ace/tui/widgets/agent_info_panel.py` so one helper builds the complete bracketed status strip:
   - Always lead with `<_running_count>/<_runner_limit> running`, including the zero-running state.
   - Use the existing capacity coloring for the numerator (green below the limit, yellow at the limit, red above it),
     the existing blue limit style, and the dim `running` label/separators.
   - Append `<_runner_queue_count> queued` in the existing queue style only when the count is greater than zero.
   - Append the remaining non-zero metrics in their current relative order and styles: stopped, starting, waiting,
     failed, unread, and done.
   - Emit one opening and closing bracket for all of these values, eliminating the duplicate running metric and the
     second chip.

2. Remove runner-slot occupancy from `AgentInfoPanel`'s stable/render state and capacity-update adapter, then update
   `src/sase/ace/tui/actions/agents/_display_detail_info.py` to pass only the effective limit and global queue count to
   the header. Keep the full `RunnerCapacitySnapshot`, its `slots_in_use` computation, and its other consumers
   unchanged. This avoids dead state and prevents an invisible slot-occupancy-only change from causing a header rebuild.

3. Update focused behavior and integration tests:
   - In `tests/ace/tui/widgets/test_agent_info_panel.py`, replace the two-chip expectations with the consolidated form
     and cover: the visible running count being the numerator; zero queue text being absent; a positive queue appearing
     inside the same brackets; zero running still producing `0/M running`; the remaining zero/non-zero metric omission
     rules; count/label styles; and stable-state/countdown fast-path behavior after removing slot occupancy.
   - In `tests/ace/tui/test_agent_neighbor_index_cache.py`, adjust the `update_state()` handoff assertion to the slimmer
     header inputs while retaining checks for the cached limit and queue count.
   - In `tests/ace/tui/visual/test_ace_png_snapshots_agents.py`, update the runner-slot wait header assertion to the
     consolidated conditional-queue syntax. Preserve the deliberately constructed queue-detail fixture; the header
     should report its positive cached queue count even if that visual fixture represents a transient state below the
     limit.

4. Regenerate and inspect the intentionally affected Agents-tab PNG goldens under `tests/ace/tui/visual/snapshots/png/`.
   Because the shared header is present in many Agents-tab snapshots, accept the full affected set only after confirming
   the diffs are limited to the consolidated count strip and resulting horizontal spacing.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run focused tests for the widget, header handoff, and runner-slot visual scenario:

   ```bash
   just test -- tests/ace/tui/widgets/test_agent_info_panel.py tests/ace/tui/test_agent_neighbor_index_cache.py
   just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_agents.py
   ```

3. Regenerate intentional PNG changes with `just update-visual-snapshots`, inspect the generated visual diffs, and rerun
   `just test-visual` without the update flag.
4. Run the mandatory full repository verification:

   ```bash
   just check
   ```

## Acceptance criteria

- The screenshot example renders `[5/10 running · 4 waiting · 31 done]` with no separate capacity brackets and no
  `0 queued`.
- `<N>` is exactly the same visible agent-lane running count formerly shown by `<N> running`, not
  `RunnerCapacitySnapshot.slots_in_use`.
- `<M>` continues to reflect the cached effective maximum runner limit.
- A positive global queue count appears as `· <Q> queued` inside the same brackets; a zero queue count emits no queue
  text or separator.
- Other non-zero status counts retain their labels, styles, and relative ordering, and the strip remains meaningful when
  the running count is zero.
- Rendering remains I/O-free and continues to use the existing stable-state and countdown-only fast paths.
- Focused tests, updated visual snapshots, and `just check` all pass.
