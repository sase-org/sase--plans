---
tier: tale
title: Aggregate agent-family status counts in Agents tab summaries
goal: 'Agents-tab panel and global status summaries replace each parallel family root''s
  aggregate status with its loaded member-status counts while preserving visible-row
  totals, panel scoping, unread semantics, and responsive refreshes.

  '
create_time: 2026-07-17 11:45:34
status: done
prompt: 202607/prompts/agent_family_status_counts.md
---

# Plan: Aggregate agent-family status counts in Agents tab summaries

## Context and intended behavior

Parallel agent families render as one top-level root row whose own status is a priority aggregate, while the root's
loaded `runtime_children` retain the real member statuses and already produce compact chips such as `[R3 W1]`. The panel
title and global Agents-tab count strip currently count only top-level rows, so each family contributes the root's
single aggregate status instead of the member chip. In the reported view, two family roots with `[R3 W1]` and `[R2 W5]`,
two ordinary running rows, and one ordinary waiting row therefore incorrectly render `[R4 W1 D28]`; their effective
status contribution should be `[R7 W7 D28]`, with the same delta reflected globally as
`[7 running · 23 waiting · 42 done]`.

Status metrics and row totals have different meanings and must remain separate. The global `63` and per-panel `· 33` are
visible/selectable top-level row totals; they should not grow when hidden family members are projected into the status
metrics. Serial workflow/follow-up children must remain excluded, and a root with no loaded parallel members must
continue contributing its own status.

## Implementation

1. Add one pure, in-memory status-count projection for the Agents-tab summary surfaces, building on the existing
   parallel-family helpers. For each visible top-level row, use loaded parallel member statuses when the row is a family
   root with members; otherwise use the row itself. Map family `STARTING` members to running, matching the existing
   family-row chip, and preserve the canonical stopped, running, waiting, failed, unread, and done categories. Split
   terminal family members between unread and done using their own identities, and never count the aggregate root in
   addition to its members.

2. Update `agent_panel_counts()` in `src/sase/ace/tui/actions/agents/_display_panel_titles.py` to consume that
   projection. Keep its top-level filtering and panel-local input so expanded children are not double-counted,
   non-family rows behave exactly as before, and each tag panel remains isolated.

3. Reuse the same projection from the cached global metrics path in `src/sase/ace/tui/actions/agents/_display_detail.py`
   so panel titles and the top strip cannot diverge. Preserve `AgentPanelIndex` totals and position semantics, retain
   the separate count for hidden top-level `STARTING` rows, and keep the existing cache/refresh invalidation path. The
   projection must inspect only already-loaded objects: no new disk access, subprocesses, full list rebuilds, or
   event-loop work.

## Tests and verification

1. Extend `tests/ace/tui/test_agent_panel_titles.py` with mixed ordinary rows and multiple parallel families. Assert
   that member buckets replace each root's single bucket, serial/family child rows are not double-counted, unread/done
   members retain their distinction, counts remain panel-scoped, and the `· N` row total is unchanged.

2. Extend `tests/ace/tui/test_agent_panel_index_integration.py` to exercise the global info metrics with the same family
   projection. Assert the requested running/waiting aggregation, family starting/terminal edge cases, unchanged
   top-level headline and navigation totals, and ordinary/no-loaded-member fallback behavior.

3. Strengthen the existing parallel-family visual scenario in `tests/ace/tui/visual/test_ace_png_snapshots_agents.py`:
   its one visible family root with two running and one done member should keep a visible-agent total of `1` while both
   the panel chip and global strip report the member totals. Regenerate and inspect only the intentional
   `agents_parallel_family_counts_120x40.png` golden change.

4. Run `just install` before repository checks, execute the focused unit tests, run the dedicated visual test/update
   flow and re-run it without snapshot updates, then finish with the required `just check` suite.
