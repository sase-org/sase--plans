---
tier: tale
title: Project parallel family members into the Agents headline total
goal: 'The Agents-tab headline total counts loaded parallel-family members in place
  of each aggregate root, matching the adjacent status summary without changing rendered-row,
  panel, or navigation semantics.

  '
create_time: 2026-07-17 12:25:39
status: done
prompt: 202607/prompts/agent_family_headline_total.md
---

# Plan: Project parallel family members into the Agents headline total

## Context and intended behavior

The Agents-tab summary now projects a loaded parallel family's member statuses in place of its aggregate root status. A
family with two running members and one done member therefore renders `[2 running · 1 done]`, but the numeric headline
beside it still says `1` because it comes from `AgentPanelIndex.top_level_total`. That combines two incompatible
cardinalities in one summary.

Make the headline use the same effective-agent projection as its status buckets: each ordinary top-level row contributes
one, while a parallel root with loaded parallel members contributes the number of those members instead of one for the
root. Thus the example becomes `3 [2 running · 1 done]`, not `1 [...]` and not `4 [...]`. A root whose parallel members
are not loaded continues to contribute one, and serial runtime children never contribute through this projection.

This changes only the global headline count. The selectable position denominator and `j`/`k` navigation still count
rendered top-level rows, each panel border's `· N` remains a row count, and expanded workflow children retain their
existing visibility behavior. Hidden top-level `STARTING` rows remain included once in the headline and in the separate
starting bucket.

## Implementation

1. Extend the existing pure `agent_summary_status_counts()` result in
   `src/sase/ace/tui/models/_agent_parallel_family.py` with the projected total. Increment it from the same
   `projected_agents` sequence already used for all status categories so headline and buckets share one definition and
   cannot drift. Preserve loaded-family replacement, unloaded-family fallback, member identity handling for unread
   terminal agents, family `STARTING`-as-running, and serial-child exclusion. Keep the projection purely in memory and
   avoid a second scan, disk access, subprocess work, or new refresh paths.

2. In the cached global metrics path in `src/sase/ace/tui/actions/agents/_display_detail.py`, derive the headline from
   the projection's total plus the existing hidden top-level `STARTING` count. Continue using
   `AgentPanelIndex.non_child_total` and `non_child_position()` for selectable row position. Do not change
   `AgentPanelIndex.top_level_total` or repurpose it as an effective-family count, because it remains a useful row
   cardinality and is shared with navigation-oriented behavior. Retain the current cache key and invalidation flow;
   loaded agent lists and unread-state changes already invalidate the cached metrics.

3. Leave panel title construction unchanged. It may consume the richer shared projection for status buckets, but its
   explicit `agent_count` stays the panel slice's rendered row count. This keeps the visual distinction deliberate: the
   global headline answers how many effective agents the global status strip describes, while panel borders and position
   text describe selectable rows.

## Tests and verification

1. Update the focused global-info tests in `tests/ace/tui/test_agent_panel_index_integration.py` so mixed ordinary rows
   and multiple loaded families assert the projected headline total alongside member status counts. Cover one family
   member per effective count, no root double-counting, serial-child exclusion, and the no-loaded-member fallback.
   Preserve explicit assertions that the position denominator remains the rendered top-level row count and that ordinary
   and hidden top-level `STARTING` cases retain their current totals.

2. Keep or strengthen panel-title coverage in `tests/ace/tui/test_agent_panel_titles.py` to prove exposing a projected
   total does not change per-panel `· N` row totals while family status chips remain member-based and panel-scoped.

3. Update the existing parallel-family scenario in `tests/ace/tui/visual/test_ace_png_snapshots_agents.py` to assert an
   info headline total of `3` with `R2/D1`, while the harness's `agent_count` remains the loaded/rendered list-state
   count. Regenerate and inspect only the intended `agents_parallel_family_counts_120x40.png` golden, then rerun the
   scenario in comparison-only mode.

4. Run `just install` before repository checks, execute the focused projection, panel-title, and info-panel tests,
   perform the dedicated visual update and comparison runs, and finish with the repository-required `just check`.
