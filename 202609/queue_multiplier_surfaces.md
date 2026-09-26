---
tier: epic
title: Finish queue multiplier display and editing surfaces
goal:
  The already implemented %queue multiplier reaches the TUI, agent-list JSON, and live
  editing surfaces. An authored 1.5x budget remains 1.5x through display and edits while
  resolving to 7.5 units when the effective machine budget is 5.
parent_bead: sase-19f
phases:
  - id: model-projection
    title: Carry multiplier through TUI agent models and loaders
    size: medium
    depends_on: []
    description:
      "model-projection: Add queue_capacity_multiplier to Agent state and every
      metadata, filesystem, identity, fleet, dedup, clan, and roster projection that
      carries queue capacity. Keep integer and multiplier mutually exclusive in setters
      and copies. Add focused projection and fleet tests."
  - id: display-list
    title: Render multiplier capacity and expose agent-list JSON
    size: medium
    depends_on:
      - model-projection
    description:
      "display-list: Render c1.5x badges, 1.5x budget and resolved units in the detail
      header, wait lane, queue ladder, and agent rows. Include the multiplier in display
      cache keys and runner-slot capacity models. Expose queue_capacity_multiplier in
      agent-list entries and JSON. Add focused widget and CLI/list tests."
  - id: edit-capacity
    title: Accept and preserve multiplier capacity in wait and directive editors
    size: medium
    depends_on:
      - display-list
    description:
      "edit-capacity: Teach the wait modal, wait actions, directive persistence, agent
      directive command, and prompt queue editing to accept <M>x through the Rust-backed
      parser. Prefill authored 1.5x, clear the opposite capacity form on edits, and
      preserve the multiplier on priority-only or weight-only changes. Add focused
      modal, persistence, and prompt-edit tests; run just check in sase."
proposed_by: bbugyi200.apollo.sase-19f.land
create_time: 2026-09-26 04:40:35
status: wip
---

- **PROMPT:**
  [prompts/202609/queue_multiplier_surfaces.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/queue_multiplier_surfaces.md)

# Finish queue multiplier display and editing surfaces

This is the missing work from phase sase-19f.4 of parent epic sase-19f. The phase bead
is closed and its notes report implementation, but no phase-4 commit is present in the
sase checkout. Current source has no `queue_capacity_multiplier` field in the TUI or
agent-list integration modules. The parent land audit recorded this on sase-19f and will
resume through `parent_bead` after this child epic lands.

Read the parent epic bead, every phase note, and the accepted
`plan:202609/queue_capacity_multiplier.md` before editing. The Rust parser, format and
resolution bindings were implemented in sase-core commits f55c63b and 96e42d9; Python
launch plumbing landed in 6beedbc11. Use those wrappers and the effective
`max_running_agents` limit rather than duplicating parsing or resolution in Python. Read
`tui.md`, `tui_perf.md`, `tui_screenshot.md`, and `lint_and_test.md` via audited
`sase memory read` before changing TUI behavior. Run `sase tool run check` for file
changes. Do not run `just check-full`.

## Model and projection

Follow the original phase-4 plan's model and loader paths:
`src/sase/ace/tui/models/_agent_state.py`, the three `_meta_enrichment_*` loaders,
`_fleet_agents_rows.py`, `_dedup.py`, `_agent_clan_sections.py`, and
`_member_roster_digest.py`. Review all copy/replace sites that carry `queue_capacity` so
the multiplier survives refresh, deduplication, and fleet rows. Read persisted
integer/multiplier precedence through the existing queue adapter.

## Display and JSON

Update `widgets/_queue_weight_badge.py`, the prompt-panel header, wait and queue
sections, agent-list row rendering/cache, and runner-slot capacity models. A 1.5x record
with effective budget 5 must display `c1.5x` and `1.5x budget (7.5 capacity units)`
where the surface has room; the wait lane shows `capacity budget 1.5x (7.5)`. Use
over-limit styling for M > 1. Add `queue_capacity_multiplier` to the agent-list entry
builder/model and `src/sase/agents/cli_list.py` JSON without breaking existing
integer-capacity output. Include multiplier in cache keys so a changed multiplier cannot
leave a stale row.

## Editing and verification

Update `modals/wait_modal_values.py`, `wait_modal_completion.py`,
`actions/agents/_wait_actions.py`, `_directive_persistence.py`,
`src/sase/ops/commands/_agent_directive.py`, and
`src/sase/xprompt/_directive_edit_wait.py`. An edit to integer capacity clears the
multiplier and an edit to multiplier clears the integer. Edits to only priority or
weight preserve authored multiplier text. Use `parse_queue_capacity_value`,
`format_queue_capacity_multiplier`, and `resolve_queue_capacity_multiplier` in
`src/sase/xprompt/queue_directive.py`; their parent epic Symvision exemptions remain
until this work consumes them. Add focused tests named in the parent phase-4 plan,
including 1.5x rendering, the 5 -> 7.5 resolution, JSON output, and editing round trips.
For changed TUI goldens, follow the screenshot memory's capture and inspection
procedure. Do not add parent close, parent Symvision cleanup, or parent plan status
changes as a phase of this child epic.
