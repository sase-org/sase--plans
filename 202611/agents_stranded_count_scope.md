---
tier: tale
title: Scope the Agents-tab stranded count to what the Agents tab can show
goal:
  The Agents info strip's stranded count matches the stranded leaves flagged on loaded
  rows, every one is reachable with ,j, and the project-wide backlog stays findable.
size: medium
decisions:
  backlog_hint:
    ask:
      Should the Agents tab still point at stranded beads that have no loaded agent row?
    choices:
      toast:
        Only the ,j triple-miss toast names the project-wide count and where to find it
      strip:
        The info strip also appends a dim `+N in Beads` suffix when a backlog exists
      silent:
        The Agents tab never mentions unsurfaced stranded beads; only the Beads pane
        counts them
    default: toast
    why:
      Keeps the always-visible strip honest while still leading to the backlog on demand
    answer: toast
  beads_filter:
    ask:
      Add a `status:stranded` filter to the Beads pane so the stranded backlog can be
      listed?
    default: true
    why: Without it the ~140 legacy stranded beads have no worklist anywhere in the TUI
    answer: true
proposed_by: bbugyi200.athena.1dl
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 12:50:24
status: wip
---

# Scope the Agents-tab stranded count to what the Agents tab can show

## Diagnosis

The Agents info strip renders `⚐ N stranded` next to the agent counts
(`85 [5 running · 29 waiting · 51 done]`), so it reads as "N stranded agents". It
actually counts **stranded beads across each project's whole bead store**:

- `src/sase/ace/tui/actions/agents/_display_detail_info.py` (both the `update_state` and
  the `update_agent_counts` paths) calls `stranded_count_for_agents(roster)`.
- `stranded_count_for_agents` (`src/sase/ace/tui/models/agent_stranded.py`) maps the
  roster to the project keys it represents. It then returns
  `_StrandedReportCache.stranded_count`, which is `sum(len(report.stranded_ids))` over
  those projects' full reports.

The original plan (`plan:202610/stranded_epic_work.md`, epic `sase-1no`) chose a
bead-anchored global count on purpose. But every other Agents-tab surface works only
from **loaded rows**:

- the `STRANDED` status overlay;
- the `,j` stranded fallback (`_jump_to_next_stranded_agent`, predicate
  `stranded_leaf_verdict(agent) is not None`);
- the Enter resolve target.

On 2026-11-17 the live reports held 136 stranded leaves for `gh_sase-org__sase` and 5
for `gh_bobs-org__bob-cli`, which is the 141–142 in the header:

- **Kinds:** 64 epics, 71 phases, and 1 task.
- **Reasons:** 104 `unclosed`, 14 `lost`, 13 `kept`, 5 `followup_failed`.
- **Ages:** 95 are older than 30 days, and the oldest (`sase-j7`) is 98 days old.

Their owning agent rows were dismissed into the Archive long ago. The count therefore
names ~142 things the Agents tab can never show, and `,j` toasts "No unread completed,
stopped, or stranded agents". The classifier is correct; the surface is mis-scoped.

## Goal

The Agents info strip's `⚐ N stranded` counts exactly the distinct stranded leaf beads
that some loaded Agents-tab row flags with a leaf (bold amber) `⚐`. `,j` can then reach
a row for every one of them. The project-wide backlog stays visible and actionable in
the Beads pane (decision `beads_filter`). The Agents tab points at that backlog only as
decision `backlog_hint` allows.

This change is presentation only. Which classified beads the Agents tab can reach
depends on which rows are loaded, so it stays in Python. The Rust stranded-work reducer
and its rules (no age horizon, no new states) are **unchanged**.

## Before you start

Read `tui.md` and `tui_perf.md` with `/sase_memory_read` before touching TUI code. Read
`docs_writing.md` before editing `docs/`. Read `lint_and_test.md` before finishing.

The in-flight phase `sase-1t3.11.1.5` reworks how stranded facts are collected (the
`sase.bead.stranded*` collector path). This tale touches only the TUI projection in
`agent_stranded.py` and its callers. If the two land close together, rebase onto it
rather than editing the collector.

## Changes

### 1. One "surfaced stranded" projection (`src/sase/ace/tui/models/agent_stranded.py`)

Add a per-row helper and a roster-level count, both memory-only (no disk I/O,
render-safe):

- `surfaced_stranded_leaves(agent) -> tuple[tuple[str, str], ...]` returns the
  `(project_key, bead_id)` pairs this row flags as stranded **leaves**. Derive them from
  `agent_stranded_view(agent)` plus the verdict index:
  - The row's own verdict, when its state is `stranded`. This is exactly
    `stranded_leaf_verdict`.
  - Every `waiting_on` / `waiting_on_agents` entry whose state is `stranded`. Use the
    wait project that `agent_stranded_view` already resolves.
  - For clan containers, every id in `clan_stranded_ids`. Use the clan's project, which
    `agent_stranded_view` already resolves.
  - Blocked (breadcrumb) states never count.

  If the view does not expose the project for wait/clan entries, extend
  `_AgentStrandedView` minimally. Keep its `signature` stable for unchanged inputs.

- `surfaced_stranded_count(roster) -> int` returns the size of the union of
  `surfaced_stranded_leaves` over the roster. Memoize it the way
  `src/sase/ace/tui/_agent_wait_cache.py` memoizes wait maps:
  - Key: roster generation (`get_roster_generation`), `id(roster)`, `len(roster)`, and
    the identity of each represented project's cached report snapshot.
  - It must stay O(1) on a cache hit, because the info strip refreshes often.
- Confirm that clan container rows appear in the roster that `_display_detail_info.py`
  passes (`_agents_with_children or _agents`). If containers are synthesized only at
  list-build time, add them through the same clan grouping that `clan_members` and the
  wait cache use, so a clan-only leaf still counts. One example is an epic phase whose
  own row was never loaded but whose clan shows `⚐1`.
- Keep `stranded_count_for_agents` (project-wide) only if decision `backlog_hint` needs
  it (`toast`/`strip`). Under `silent`, delete it and its
  `_StrandedReportCache.stranded_count` memo. Symvision flags unused symbols.

### 2. Info strip (`_display_detail_info.py`, `widgets/agent_info_panel.py`)

- Both call sites pass `surfaced_stranded_count(...)` as `stranded_count`. Keep the
  `⚐ N stranded` grammar and zero-suppression.
- The existing warmup path already refreshes the strip after a report changes. Also make
  sure a roster change (rows loaded or dismissed) refreshes the count. The memo key
  covers this, but confirm that the strip repaints on that path.

> [!decision] backlog_hint = strip Add `stranded_backlog_count` to `update_state` /
> `update_agent_counts`, computed as project-wide minus surfaced and floored at 0.
> Render ` · ⚐ N stranded` (surfaced, bold amber) followed by a dim ` +M in Beads`. When
> surfaced is 0 and the backlog is positive, render only a dim ` · ⚐ +M in Beads`.
> Zero-suppress both parts. Cover both shapes in
> `tests/ace/tui/widgets/test_agent_info_panel_grammar.py`.

### 3. `,j` stranded fallback (`actions/agents/_unread_navigation.py`)

- `_jump_to_next_stranded_agent` uses the predicate
  `bool(surfaced_stranded_leaves(agent))`, so every surfaced leaf has a reachable row.
  That includes clan containers and WAITING rows. Keep
  `include_collapsed_clan_members=True`. Make sure clan container rows are jump
  candidates in `_timed_agent_jump_candidates`.
- `_stranded_agent_jump_time` uses the most recent `since` among the row's surfaced
  leaves, falling back to the agent's stop or start time as today. This keeps the "most
  recently stranded first" order that the `default_config.yml` comment for
  `jump_to_next_unread_done_agent` describes.
- Enter on any such row already resolves through `_agent_enter_stranded.py`. Verify that
  this still holds for WAITING and clan rows, and do not change its priority order.
- Leave the agents-live `stranded:true` query field (owner-row semantics) unchanged.

### 4. Triple-miss toast (`actions/agent_workflow/_leader_mode.py`)

> [!decision] backlog_hint = toast When `,j` misses all three jumps and the project-wide
> count is positive, extend the toast. If `beads_filter` is true, it reads
> `No unread completed, stopped, or stranded agents · ⚐ N stranded beads have no agent row here (Beads: status:stranded)`.
> If it is false, it reads `… (see the Beads pane)`. Use the memoized project-wide
> count, with no I/O on the key path. Otherwise the toast text is unchanged.

Under `strip` or `silent`, leave the toast text unchanged.

### 5. Beads pane `status:stranded` (only when decision `beads_filter` is true)

- Add `"stranded"` to `DERIVED_BEAD_STATUS_VALUES` (`src/sase/bead/filter_query.py`).
  Update the status hint in `src/sase/ace/query_profile/profiles/_beads.py`.
- In `widgets/artifacts/beads_filtering.py` `_record`, add the `stranded` status label
  when the bead's cached verdict state is `stranded`. Read the verdict through an O(1)
  verdict-index lookup, for example a small public accessor over
  `_StrandedReportCache.verdict_index`. Do not use `cached_bead_verdict`, which scans
  every verdict per bead.
- The filter index is cached by `snapshot.source_key` (`query_rows.py`,
  `beads_filter_session.py`). It must also rebuild when the stranded reports change, or
  `status:stranded` serves stale rows. Fold a stranded-report token into its cache key,
  for example the `beads_stranded_signature` already computed in
  `beads_stranded_refresh.py`, or the report snapshot identities.
- The Beads status bar's `⚐ N stranded` (`beads_stranded_count`) stays project-wide. It
  should equal the leaf count that `status:stranded` lists across tasks, epics, and
  phases.

### 6. Docs and help

- `docs/reference/tui/agent-row-glyphs.md`, "Stranded work": rewrite the info-strip
  paragraph. The strip counts stranded leaves flagged on loaded rows (own row, WAITING
  target, or clan subtree), and `,j` visits them. Describe where the project-wide count
  lives according to the decisions.
- Search for a help-modal legend that describes the info-strip count (start with
  `modals/help_modal/agents_reference_sections.py` and `agents_mode_sections.py`) and
  align it.
- If `beads_filter` is true:
  - Document `status:stranded` wherever the Beads filter's derived statuses (`blocked`,
    `launched`, `triage`) are documented.
  - Mention it as the worklist in `docs/reference/work-units/bead-lifecycles.md`
    (`#stranded-beads`).

## Tests

- `tests/ace/tui/models/test_agent_stranded.py`:
  - A report has three stranded leaves. One has a loaded owner row, one is visible only
    through a loaded clan's subtree, and one has no row. The surfaced count is 2.
  - Blocked verdicts never count.
  - The memo invalidates on a roster generation bump, a roster replacement, and a report
    replacement, and it hits otherwise.
  - Screenshot regression: 140 old stranded leaves, none flagged by a loaded row, give a
    surfaced count of 0.
- `tests/ace/tui/test_agent_stranded_navigation.py`:
  - `,j` lands on a clan container whose only stranded leaf has no row of its own.
  - `,j` lands on a WAITING row whose target is stranded.
  - The order is by most recent `since`.
- `tests/ace/tui/test_leader_keymap_dispatch.py` (and the helpers): cover the toast text
  for the chosen `backlog_hint` and `beads_filter` branch.
- `tests/ace/tui/widgets/test_agent_info_panel_grammar.py`: the strip uses the surfaced
  count, plus the `strip` shapes if that branch is chosen.
- Beads filtering (`status:stranded`, if chosen): matching, negation, and index
  invalidation after a report change.
- PNG golden `agents_stranded_rows_and_clan_120x40.png`: rerun it. Regenerate only if
  the scenario's strip count legitimately changes, and inspect the diff.
- Assert no new disk I/O on the render or keystroke paths. The new helpers read only
  `_STRANDED_REPORT_CACHE`.

## Out of scope

- Reducer or classification changes in `sase-core`, including any age horizon. The ~140
  legacy beads are genuinely stranded by the current rules. Clearing them is a user
  action: Beads pane `w` resolve, or `sase bead close`.
- Changing the agents-live `stranded:true` field or the per-row `STRANDED` overlay.

## Verification

- Run `just check` as described in `lint_and_test.md`.
- Live: restart the TUI (a running TUI keeps its imported code) and open the Agents tab.
  - The strip shows `⚐ N stranded` only when that many distinct leaves are flagged on
    visible or collapsed rows, and `,j` cycles through them.
  - The Beads pane still shows the project-wide count, and `status:stranded` lists it if
    that branch was chosen.
