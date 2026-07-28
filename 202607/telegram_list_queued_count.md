---
tier: tale
title: Show queued-agent counts in the Telegram list summary
goal: Telegram list overviews account for queued agents in both their status rollup and canonical section order.
create_time: 2026-07-28 18:43:09
status: wip
---

- **PROMPT:** [202607/prompts/telegram_list_queued_count.md](prompts/telegram_list_queued_count.md)

# Plan: Show queued-agent counts in the Telegram `/list` summary

## Objective

Update the `sase-telegram` plugin so every non-terminal status represented by the `/list` overview is also represented
in the status rollup directly below its active-agent total. In particular, queued agents must contribute a
`… <count> queued` segment between the running and waiting segments. Keep the overview's detailed status sections in the
same canonical order so the summary and body agree.

The screenshot demonstrates the current mismatch: the title reports 15 active agents, while the rollup reports only 5
running and 6 waiting. The remaining four agents are in SASE's `Queued` bucket, but Telegram omits that bucket from the
rollup.

## Root cause

Work in the linked `sase-telegram` repository; open it first with `sase repo open sase-telegram -r "<reason>"` and use
only the path that command returns.

SASE's `sase.agent.status_buckets.AGENT_STATUS_BUCKETS` now defines this canonical order: `Stopped`, `Failed`,
`Starting`, `Running`, `Queued`, `Waiting`, `Done`. Its agent-list entries already expose queued agents with
`status_bucket == "Queued"` and `status_glyph == "…"`.

The plugin predates that addition and duplicates the old order in two places:

- `src/sase_telegram/agent_format.py::format_header_status_counts()` hard-codes the recognized rollup buckets without
  `Queued`, so it counts queued entries internally but never emits them.
- `src/sase_telegram/scripts/sase_tg_inbound.py::_LIST_STATUS_BUCKET_ORDER` also omits `Queued`, so
  `_group_list_entries()` treats the queued detail group as an unknown fallback instead of placing it between Running
  and Waiting.

The active-entry filtering and `agent_list_entries()` projection are already correct; no SASE core/backend change is
needed.

## Implementation

1. Import and reuse `AGENT_STATUS_BUCKETS` from `sase.agent.status_buckets` in the Telegram formatting path instead of
   maintaining a private status-order tuple.
   - In `format_header_status_counts()`, derive the ordered, nonzero rollup buckets from the shared tuple.
   - Preserve the existing glyph lookup, lowercase labels, omission of zero-count buckets, and `None` behavior for
     empty/no-recognized-status input.
   - This should render queued entries using the glyph already supplied by each `AgentListEntry`, producing text such as
     `▶ 5 running · … 4 queued · ⏳ 6 waiting`.
2. Remove the duplicated `_LIST_STATUS_BUCKET_ORDER` in `sase_tg_inbound.py` and have `_group_list_entries()` use the
   same shared `AGENT_STATUS_BUCKETS` order. Preserve the current fallback for genuinely unknown future buckets after
   all canonical groups.
3. Add a focused `/list` regression test in `tests/test_inbound.py` using active Running, Queued, and Waiting entries.
   Assert:
   - the title's active total includes all three buckets;
   - the next line contains the exact nonzero rollup counts and canonical `Running → Queued → Waiting` order;
   - the detailed group headers appear in that same order. Include more than one queued entry so the reported count, not
     merely the label's presence, is verified. Existing tests for empty lists, recent/terminal entries, project
     filtering, refresh callbacks, and HTML escaping should remain unchanged.

## Scope boundaries

- Do not change how runner-slot waiters are classified; that logic already lives in SASE and correctly yields `Queued`.
- Do not infer queued count as `active - running - waiting`; aggregate the entry buckets directly so Starting, Stopped,
  or future statuses remain accurate.
- Do not redesign individual queued-agent detail tokens, queue-position metadata, `/show` layouts, or Telegram command
  behavior. Reusing the shared formatter may make existing `/show` rollups correctly include Queued as an incidental
  consistency improvement.
- No documentation update is required because the command documentation does not enumerate the rollup's status segments.

## Validation

From the opened `sase-telegram` checkout:

1. Run the focused regression test: `just test tests/test_inbound.py -k 'list and queued'`.
2. Run `just install`, then the repository-required full validation with `just check`.
3. Re-read the rendered assertions to confirm the active count equals the sum of the non-terminal test buckets and both
   the rollup and section headings place `Queued` between `Running` and `Waiting`.
