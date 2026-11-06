---
tier: tale
title: Show running and newest slow tool calls first
goal:
  The per-agent SLOW TOOL CALLS section lists running calls first, then newest-first,
  dropping the oldest when capped.
size: small
decisions:
  triage_views:
    ask:
      Should the clan and tribe SLOW TOOL CALLS triage lists also switch to
      running-first, newest-first order?
    choices:
      keep:
        Clan and tribe lists stay slowest-first; only the per-agent/session section
        changes
      match:
        Clan and tribe top-5 lists show running and newest calls instead of the slowest
    default: keep
    why:
      Those lists are deliberate cross-member slowest-call rankings, not the
      chronological list
    answer: keep
proposed_by: bbugyi200.athena.197
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-06 11:34:38
status: wip
---

# SLOW TOOL CALLS: running calls and newest calls first

## Problem

The per-agent and session `SLOW TOOL CALLS` section in the Agents metadata header lists
rows oldest-first. In the reported screenshot, a `research.3i.final` agent shows five
completed `Bash` calls ordered `11:02:07, 11:02:08, 11:03:06, 11:03:07, 11:03:08`, so
the most recent call is at the bottom. The intended order puts calls that are still
running at the top, then the most recently started calls. When the section is capped,
the oldest calls are the ones dropped.

## Root cause

`src/sase/ace/tui/widgets/prompt_panel/_agent_slow_tools.py`:

- `_select_sourced_slow_tool_calls()` returns every qualifying call sorted ascending by
  `_sourced_slow_tool_call_sort_key` (`started_at`, `source_index`, `line_number`).
- `_select_visible_slow_tool_calls()` already picks the right _set_ when capped at
  `MAX_VISIBLE_SLOW_TOOL_CALLS` (8): all running calls (newest 8 if there are more),
  then the newest non-running calls. It then re-sorts that set ascending
  (`sorted(visible, key=_sourced_slow_tool_call_sort_key)`). When the section is not
  capped it returns the ascending tuple unchanged.

So the selection already drops the oldest calls, but both code paths display
oldest-first. The fix only changes display order. It does not change which calls qualify
or which are kept.

This is presentation ordering of a Textual section, so it stays in this repo under the
Rust-core boundary. `select_slow_tool_calls()` in `src/sase/ace/tui/llm_calls/slow.py`
and the LLM Calls panel timeline keep their chronological ascending contract and are not
changed.

## Changes

### 1. Display order in `_agent_slow_tools.py`

- Add a display sort key helper, for example `_slow_tool_call_display_sort_key(item)`,
  that orders:
  1. running calls (`slow_call.is_running`) before everything else. Calls marked
     `did_not_complete` count as non-running.
  2. within each group, newer `started_at` first. `started_at` is a UTC-aware datetime,
     so sort on `-started_at.timestamp()` or use a two-pass stable sort.
  3. on equal `started_at`, ascending `source_index`, which keeps
     `test_slow_tools_section_ties_use_source_order` valid.
  4. then descending `entry.line_number`, so a later line in the same log counts as
     newer.
- Rewrite `_select_visible_slow_tool_calls()` so that **both** the uncapped and capped
  paths return `tuple(sorted(visible, key=<display key>))`. Keep the existing capped
  selection logic (`running[-MAX:]`, else running plus
  `non_running[-remaining_slots:]`), because it already keeps the newest calls.
- Leave `_select_sourced_slow_tool_calls()` sorted ascending. `slow_calls` feeds
  `prepare_slow_tool_call_details()`, whose duration rank uses the input index as a tie
  breaker, and `detail_by_item_id` maps by `id(item)`. Neither depends on display order,
  so rank and share output stay the same.
- Update any docstring or comment that says "ascending" or "start-time order" for the
  rendered rows.
- The overflow tail (`+ N more · … full timeline in LLM Calls`) and the "full commands
  hidden" tail still render after the rows. Because the oldest rows are the hidden ones,
  the overflow line now reads correctly at the bottom.
- Hint markers (`register_tool_call_report_hint`) are assigned in row order, so the
  newest reportable completed call now gets the lowest hint number. This is intended.

### 2. Clan and tribe triage lists (decision `triage_views`)

> [!decision] triage_views = keep Make no code change in
> `_agent_clan_disk_aggregation.py`, `_agent_display_clan_sections_slow.py`,
> `_agent_tribe_aggregation.py`, or `_agent_display_tribe_sections.py`. Those lists stay
> ranked slowest-first.

> [!decision] triage_views = match
>
> - In `_aggregate_clan_slow_tool_calls()` (`_agent_clan_disk_aggregation.py`), replace
>   the `-effective_duration_ms` ranking with the same order as change 1: running first,
>   then `started_at` descending, then `member_order` ascending, then `line_number`
>   descending. Update its docstring ("rank") to match.
> - In `_agent_tribe_aggregation.py`, sort the concatenated `slow_calls` globally with
>   the same key (via `TribeSlowToolEntry.entry.call`) before building
>   `_TribeDiskSnapshot`, so the tribe top-`TRIAGE_LIMIT` rows reflect the newest calls
>   across clans.
> - Update the assertions in `tests/ace/tui/widgets/test_agent_clan_aggregation*.py`,
>   `test_agent_tribe_aggregation.py`, `test_agent_display_clan_sections.py`, and
>   `test_agent_display_tribe_sections.py` that depend on slowest-first order.
> - Update the clan and tribe docs sentences that describe the triage ranking, if any.

### 3. Tests: `tests/ace/tui/widgets/test_agent_slow_tools.py`

- Rename `test_slow_tools_section_orders_by_start_time_across_sources` to something like
  `test_slow_tools_section_orders_running_then_newest_first_across_sources`. With the
  same fixture, assert
  `index("running middle") < index("completed late") < index("completed early")`.
- `test_slow_tools_section_overflow_points_to_llm_calls_timeline`: keep the membership
  assertions and add an order assertion that `call 9` renders before `call 2` (newest
  first), with the overflow line after the last row.
- `test_slow_tools_section_keeps_running_call_when_capped`: add order assertions that
  `running long` is first even though it started earliest, then `completed 8` comes
  before `completed 2`.
- Add an uncapped test that covers the screenshot case: five completed calls at
  increasing `recorded_at` render newest-first. Add a running call that started _before_
  a newer completed call and assert it still renders on top. Add a `did_not_complete`
  call and assert it sorts with the non-running rows by start time.
- Keep `test_slow_tools_section_ties_use_source_order` passing without changes.

### 4. Tests: `tests/ace/tui/widgets/test_agent_slow_tools_hints.py`

- `test_completed_slow_tools_get_hint_markers_and_report_specs`: `success` (14:00:01)
  now renders above `failed` (14:00:00), so `success` gets hint `[12]` and `failed` gets
  `[13]`. Swap the expected markers and mappings. The other assertions about report
  specs and alignment stay as they are.
- Check that `test_hint_markers_and_report_specs_are_identical_across_detail_tiers` and
  `test_slow_tool_hints_ignore_non_reportable_rows` still pass. They compare tiers with
  each other or use a single reportable row, so they should not need edits.

Run the slow-tool test modules locally and fix any other order-dependent assertion they
expose, for example in `test_agents_panel_fold_slow_tools.py`,
`test_summary_fold_slow_tools.py`, `test_agent_slow_tools_detail.py`,
`test_agent_slow_tool_tick.py`, `test_prompt_panel_header.py`, or
`test_tool_runs_run_links.py`:

```bash
.venv/bin/pytest -q tests/ace/tui/widgets/test_agent_slow_tools.py \
  tests/ace/tui/widgets/test_agent_slow_tools_hints.py \
  tests/ace/tui/widgets/test_agent_slow_tools_detail.py \
  tests/ace/tui/widgets/test_agent_slow_tool_tick.py \
  tests/ace/tui/widgets/test_summary_fold_slow_tools.py \
  tests/ace/tui/test_agents_panel_fold_slow_tools.py \
  tests/ace/tui/test_tool_runs_run_links.py \
  tests/ace/tui/widgets/test_prompt_panel_header.py
```

### 5. Docs

Read the `docs_writing.md` reference memory before editing docs. In
`docs/reference/tui/agents-main-deck.md` (the "Slow tool calls" bullet), replace
"ordered by start time and capped at 8 rows" with wording that says running calls come
first, then the rest newest-first by start time, and that the 8-row cap drops the oldest
calls. The overflow line still points to the LLM Calls timeline. Grep `docs/` for any
other sentence that states the per-agent slow-call order, for example
`agent-sessions.md` or `agents-tab-panels.md`, and update it the same way.

### 6. PNG goldens

`tests/ace/tui/visual/test_ace_png_snapshots_agents_slow_tools.py` renders three calls:
a completed check at 12:02, a failed read at 12:04, and a running test. Its goldens
`agents_slow_tool_calls_level_1_120x40.png` and
`agents_slow_tool_calls_level_2_120x40.png` will change, because the running row moves
to the top and the 12:04 row moves above the 12:02 row. Follow the `tui_screenshot.md`
reference memory's golden-maintenance contract:

- Run the targeted update through `/sase_monitor`:
  `just fix-tui-screenshots -- tests/ace/tui/visual/test_ace_png_snapshots_agents_slow_tools.py`.
  Then inspect the retained report and every changed golden. The only expected
  difference is the reordered rows.
- Prove host-independence with the targeted `--check` form run twice, once with `TZ=UTC`
  exported, and confirm that neither run reports an `unstable` node.
- If another visual suite that shows a multi-row SLOW TOOL CALLS section changes,
  inspect and accept it the same way. Accept no unrelated golden diffs.

### 7. Verification

- Run the targeted pytest command above, and also the clan and tribe test modules if
  `triage_views = match`.
- Run `sase tool run check` (per `lint_and_test.md`). Do not run `just check-full`.

## Out of scope

- The LLM Calls panel timeline and the slow markers in its rows stay chronological.
- `select_slow_tool_calls()` keeps its ascending return contract, which other callers
  such as clan aggregation and tool-run links rely on.
- `MAX_VISIBLE_SLOW_TOOL_CALLS` and the slow threshold do not change.
