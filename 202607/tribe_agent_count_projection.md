---
tier: tale
title: Correct concrete-agent counts and settled-member status projection in Agents-tab
  summaries
goal: 'Tribe, clan, family, panel-title, and group-banner summaries in `sase ace`
  all count concrete agents with one shared projection, and sequential family members
  that finished an approved plan/tale handoff read as done instead of running.

  '
create_time: 2026-07-19 10:53:33
status: wip
prompt: 202607/prompts/tribe_agent_count_projection.md
---

# Plan: Concrete-agent counting and settled family-member status projection

## Problem

The Agents-tab tribe detail document (whole-panel focus) reports wrong agent counts and wrong agent status counts.
Reference scenario (observed live): an untagged tribe containing 4 sequential agent families (each a `--plan` planner +
`--code` coder) and 2 single-agent workflow units — 10 concrete agents total. The TUI currently shows:

1. **`Composition: … 12 agents` instead of 10.** `tribe_unit_real_rows()`
   (`src/sase/ace/tui/models/agent_tribe_summary.py`) returns, for a non-container unit, the workflow aggregate root
   **plus** all descendant step rows. A single-agent workflow (root + `main` agent step) therefore counts as 2 agents,
   and `agent_count = len(real_rows)` inherits the double count.
2. **`… 10 nested` instead of 8.** `nested_count = sum(len(unit.children))` counts workflow step rows as nested agents.
   A workflow's `main` step is that unit's own execution, not a nested agent; only agents living inside containers
   (family members, clan members) are nested.
3. **Family status chips over-report running members.** `TALE APPROVED` / `PLAN APPROVED` bucket globally as `Running`
   (`ACTIVE_PLAN_HANDOFF_STATUSES` in `src/sase/agent/status_buckets.py`). That is correct for a standalone planner
   mid-handoff, but a sequential family member with a successor member has finished: its process stopped and the next
   member owns the work. Today a fully finished family (planner `TALE APPROVED`, coder `TALE DONE`) shows `[R1 D1]`
   instead of `[D2]`, an active family (planner `TALE APPROVED`, coder `WORKING TALE`) shows `[R2]` instead of
   `[R1 D1]`, and roster child rows render `▶ TALE APPROVED` (running glyph/color) instead of a done glyph.
4. **The tribe header chip counts top-level units, not agents.** `Status: RUNNING [R2 D4]` sums to 6 units while the
   Composition line talks about agents. The shared counter `agent_summary_status_counts()`
   (`src/sase/ace/tui/models/_agent_clan.py`) already projects _clan_ containers into their members but counts _family_
   containers and workflow roots as single opaque units. The same helper drives the left-panel title chips
   (`src/sase/ace/tui/actions/agents/_display_panel_titles.py`) and the info-panel "effective-agent" metrics
   (`src/sase/ace/tui/actions/agents/_display_detail_info.py`), so the inconsistency is visible on several surfaces at
   once.
5. **Status-group banners have the same family gap.** The `"N agents · M running"` banner summary
   (`src/sase/ace/tui/models/agent_groups/_tree.py`, `_BannerSummary` projection) substitutes clan containers with their
   children but counts a family container as 1 agent.

Root cause across all five: there is no single definition of "concrete agent", and no family-aware status projection —
each surface re-derives counts slightly differently.

## Semantics contract (the core of this change)

Define these rules once in the models layer and route every summary surface through them:

- **Concrete agent.** Clan containers and family containers are never agents. A workflow aggregate root whose concrete
  agent-type step rows are loaded is _represented by those steps_ (one agent per `step_type == "agent"` step, excluding
  synthetic planners); with no loaded agent steps the root itself counts as one agent. Python/bash step rows are never
  agents. Sequential family containers project to `concrete_family_member_rows()` (which already resolves the planner
  step vs rename-on-attach root). Clans project to their members, recursively applying the family rule to members that
  are family containers.
- **Settled family member.** Within one sequential family's ordered concrete member list, every member _before the last_
  whose status is in `APPROVED_PLAN_STATUSES` (`PLAN APPROVED`, `TALE APPROVED`) projects to the **Done** bucket for
  counting, chips, and glyphs. The raw status text stays visible (e.g. `✓ TALE APPROVED` in done styling). The _last_
  member keeps global bucketing, so a planner-only family mid-handoff still reads as running.
- **Nested agent.** A concrete agent is nested when it lives inside a container unit and is not the unit row itself
  (rename-on-attach family roots that double as first member are top-level, not nested). Workflow steps are never nested
  agents.
- **Untouched:** `status_bucket_for_values()` and the query/filter taxonomy in `src/sase/agent/status_buckets.py` must
  not change — the remap is a presentation-side projection for family sequences only. Parallel families
  (`agent_family_parallel`) keep their existing counting paths.

## Design

All work is pure in-memory presentation projection in Python (Textual models/widgets). No Rust core (`sase_core`)
changes: no wire/API semantics move, matching how the existing projections (`agent_family_members.py`, `_agent_clan.py`)
are already layered. Per the TUI performance rules, no disk I/O, no new refresh paths, and no per-render stat/glob — the
new helpers are O(loaded rows) transformations of rows the snapshot builders already walk.

1. **Shared projection helpers** — extend `src/sase/ace/tui/models/agent_family_members.py` (or a sibling models module
   if it reads better):
   - a helper resolving any non-container row to its concrete agent rows (workflow aggregate → loaded agent-type steps,
     else the row itself), generalizing the existing `_concrete_planner_child()` logic;
   - a helper producing effective status buckets for an ordered family member list, applying the settled-member rule
     above (returns per-row bucket, or the set of settled identities).

2. **Tribe summary model** (`src/sase/ace/tui/models/agent_tribe_summary.py`):
   - `agent_count`: count deduped concrete agents across units via the shared helpers (clan → members' concrete agents;
     family → concrete members; agent/workflow → concrete agent rows).
   - `nested_count`: concrete agents inside containers, excluding a family root row that is itself the unit row.
     (Existing mixed-unit test scenario must still yield 4 agents / 2 nested; the reference scenario yields 10 agents /
     8 nested.)
   - header `counts` chip: project the same concrete agents with effective buckets (reference scenario: `[R2 D8]`).
   - family unit `status_counts` chips: use effective buckets (active family `[R1 D1]`, finished family `[D2]`).
   - `_TribeUnitChild` carries the member's effective bucket for the roster renderer.
   - Keep `real_rows` (unit + descendants) as-is for errors / output variables / workflow variables / runtime-span
     aggregation so python-step outputs and root-attached data keep rendering.

3. **Shared status counters**:
   - `agent_summary_status_counts()` (`src/sase/ace/tui/models/_agent_clan.py`): project family containers into concrete
     members with effective buckets, mirroring the existing clan projection; recurse one level for clan members that are
     family containers. Preserve the asking/dismissable/starting special cases. Preserve unread semantics: a container
     identity in `unread_ids` must still count exactly one unread (attribute it to a projected member — e.g. the final
     one — when no member identity is itself unread).
   - Group banner summary (`src/sase/ace/tui/models/agent_groups/_tree.py`): apply the same family projection so a
     status group over 2 active families reads `4 agents · 2 running`.
   - Panel titles (`src/sase/ace/tui/actions/agents/_display_panel_titles.py`): extend the existing clan total
     substitution to family containers so the title total equals the chip sum (reference scenario:
     `(untagged) · 10 [R2 D8]`) and stays stable across fold levels.

4. **Roster rendering** (`src/sase/ace/tui/widgets/prompt_panel/_member_roster.py`):
   - Add an optional effective-bucket field to `MemberRosterEntry` / `MemberRosterChild`; `_append_member_fields()`
     prefers it over `status_bucket_for_values(status)` for the glyph and status style. Raw status text unchanged.
   - Pass effective buckets from the tribe children (`_agent_display_tribe.py` roster adapter), the family detail roster
     (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_family.py`), and the clan detail roster's family-member
     children (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_clan_roster.py`).

## Expected behavior on the reference scenario

- Tribe document: `Composition: 4 families · 10 agents · 8 nested`; `Status: RUNNING [R2 D8]`.
- TRIBE MEMBERS: active families `[R1 D1]` with `✓ TALE APPROVED` planner + `▶ WORKING TALE` coder; finished families
  `[D2]`; workflow units unchanged visually but counted once.
- Left panel: `(untagged) · 10 [R2 D8]`; running group banner `4 agents · 2 running`.
- Top strip / info panel effective-agent totals include family members (e.g. 28 → 32 in the reference session). This is
  an intentional, user-visible consequence of unifying "agent" semantics — flagging explicitly for review.

## Testing

- Update `tests/ace/tui/models/test_agent_tribe_summary.py`: flip the concrete-planner family chip expectation to
  `running == 1 / done == 1`; add a workflow-unit scenario (root + agent step
  - python step → 1 agent, 0 nested); add a finished-family scenario (`[D2]`, header done counts); keep the mixed-root
    scenario asserting 4 agents / 2 nested.
- Extend `tests/ace/tui/models/test_agent_family_members.py` for the new helpers (settled-member buckets incl.
  last-member exemption; workflow agent-step resolution; no-steps fallback).
- Update counter tests for `agent_summary_status_counts` (family projection, unread-on-container attribution) and
  group-banner tests for `agent_groups/_tree.py`.
- Update widget tests: `tests/ace/tui/widgets/test_agent_display_family.py`, `test_agent_display_clan.py`, tribe display
  tests, and panel-title tests for the new chip and glyph output.
- Visual PNG snapshots: run `just test-visual`; accept intentional diffs with `--sase-update-visual-snapshots` after
  inspecting `.pytest_cache/sase-visual/` artifacts (family/tribe/clan panel goldens will legitimately change).
- Full gate: `just install` then `just check` before completion.

## Risks and edge cases

- **Planner-only family mid-handoff** (approved status, successor not yet attached) must stay running — covered by the
  last-member exemption; add a regression test.
- **Rename-on-attach families** where the root row is the first concrete member: root stays a top-level agent (not
  nested); existing mixed-root test pins this.
- **Workflows with multiple agent steps** count one agent per agent step — an honest count even though rare;
  python/bash-only workflows fall back to 1 (the root).
- **Unread counting** must not lose container-level unread marks nor double-count when both the container and a member
  are marked.
- **Parallel families** and `clan_member_counts()` consumers (left-list chips, cleanup modal) keep direct-member
  semantics; do not reroute them.
- **Scope of the running→done remap** is limited to `APPROVED_PLAN_STATUSES` on non-final family members; `WORKING PLAN`
  / `WORKING TALE` and `EPIC APPROVED` are left bucketing as running everywhere.
- Out of scope: the left-list `×N` fold annotations (they intentionally count collapsible rows, not agents) and any
  change to `status_bucket_for_values()` query semantics.
