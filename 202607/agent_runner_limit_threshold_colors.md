---
tier: tale
title: Make the Agents-header runner limit reflect capacity pressure
goal: The runner-limit denominator uses rounded capacity thresholds to progress from
  neutral grey through yellow and orange to red.
create_time: 2026-07-24 19:11:48
status: wip
---

- **PROMPT:** [202607/prompts/agent_runner_limit_threshold_colors.md](prompts/agent_runner_limit_threshold_colors.md)

# Make the Agents-header runner limit reflect capacity pressure

## Goal

Change only the configured runner-limit denominator (`M`) in the Agents-tab `N/M running` header so it communicates how
close the visible running-agent count is to capacity:

- below 50%: use the same dim grey style as the adjacent `running` and `waiting` labels;
- at or above 50%: use the current bold gold/yellow (`#FFD700`);
- at or above 75%: use bold orange (`#FF8700`);
- at or above 100%: use bold red (`#FF5F5F`).

Convert each percentage to an integer running-count boundary by rounding up. For example, a limit of 10 yields
boundaries of 5, 8, and 10, while a limit of 7 yields boundaries of 4, 6, and 7. Counts above the configured limit
remain red.

The visible numerator `N` must retain its existing behavior: green below the limit, yellow at the limit, and red above
the limit. Preserve the consolidated header text, queue visibility rules, all other metric colors, and the existing
cached countdown/render paths.

## Implementation

1. In `src/sase/ace/tui/widgets/agent_info_panel.py`, replace the denominator's unconditional gold style with a small,
   deterministic style-selection helper driven solely by the already-cached `_running_count` and `_runner_limit`.
   Evaluate thresholds from highest severity to lowest:
   - return dim grey when the limit is zero or otherwise non-positive;
   - return red when `running >= limit`;
   - return orange when `running >= ceil(3 * limit / 4)`;
   - return yellow when `running >= ceil(limit / 2)`;
   - otherwise return dim grey.

   Compute the ceiling boundaries with integer arithmetic (for example, `(3 * limit + 3) // 4` and `(limit + 1) // 2`)
   so odd limits follow the requested round-up rule without floating-point edge cases. Reuse the established palette
   values already present in the TUI, and apply the result only to every glyph of the rendered denominator. Do not add
   state, I/O, refreshes, awaits, or cache-key inputs: both values already participate in the panel's stable render
   state.

2. In `tests/ace/tui/widgets/test_agent_info_panel.py`, expand the focused Rich span assertions for the capacity
   segment. Cover both sides of every boundary for a limit of 10 (`4/5`, `7/8`, and `9/10`), an over-capacity count, and
   an odd limit such as 7 to prove that 50% rounds to 4 and 75% rounds to 6. Include the zero/non-positive-limit
   fallback if the helper accepts it. Assert that:
   - all denominator digits receive the expected dim, yellow, orange, or red style;
   - the numerator keeps its independent existing style;
   - the slash and status labels remain dim;
   - queued and done counts retain their current styles; and
   - zero queues remain omitted.

3. Regenerate only PNG visual goldens whose shared Agents header renders a denominator with a newly selected color.
   Compare every modified golden directly with its prior version and accept it only when changed pixels are confined to
   the denominator glyphs in the one-line Agents header. No text, geometry, list/detail content, modal, footer, or other
   metric pixels should change.

## Validation

1. Run the focused `AgentInfoPanel` widget tests, including the new threshold and rounding cases.
2. Run the complete ACE PNG visual snapshot suite after updating and auditing the affected goldens.
3. Run `just install` first as required for an ephemeral SASE workspace, then run the mandatory repository-wide
   `just check`. If an unrelated pre-existing repository or external generated-state failure blocks that aggregate
   command, record it precisely and run the remaining in-repository formatting, lint, type-check, unit, and visual gates
   independently without changing unrelated files.

## Non-goals

- Do not change how running agents, runner limits, or queues are counted.
- Do not change the numerator's existing colors or make the denominator bold while it is in the neutral dim-grey state.
- Do not change status-strip wording, spacing, ordering, conditional queue display, or any other Agents-tab palette
  entry.
- Do not change runner scheduling, configured-limit resolution, Statistics-pane colors, refresh cadence, or TUI cache
  behavior.
