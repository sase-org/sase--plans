---
tier: tale
title: Surface active monitors through their agent-family root
goal:
  Agent families display their monitor lifecycle accurately without losing monitor rows
  or inflating agent counts.
size: medium
proposed_by: bbugyi200.athena.00p
create_time: 2026-08-14 08:11:39
status: wip
---

# Plan: Surface active monitors through their agent-family root

## Objective

Make an agent family remain visibly active while a command launched through
`sase monitor` is running. In the collapsed Agents view, the family root must mirror the
monitor member's existing start status (normally `MONITORING`) and use the `Running`
bucket instead of appearing terminal as `TALE DONE` or `PLAN DONE`. The monitor must
also remain a visible family member when the family is expanded.

Do not introduce a second monitor status vocabulary. Monitor artifacts already carry the
authoritative `monitor_state`, configurable `monitor_start_status` / stop status, and
effective status bucket. Preserve the established distinction between the LLM agent that
started a monitor and the non-LLM monitor member itself.

## Confirmed diagnosis

The `00i.f0` artifact shape from the reported screenshot reproduces the bug:

- The completed code member belongs to family `00i.f0` and points at the plan-family
  root through its `parent_timestamp`.
- The monitor member also belongs to family `00i.f0`, but its `parent_timestamp`
  deliberately points at the concrete code member that launched it. Monitor follow-up
  settlement uses that direct relationship to wait for and fork the starter safely.
- TUI status normalization and final ordering treat `parent_timestamp` as the display
  container link and only roll direct children into a family root.
- Consequently, a monitor started by a family continuation is nested under that
  already-child row. It is absent from the root's `followup_agents`, cannot participate
  in root status mirroring, and can fall out of the final visible row ordering. The
  newest direct child remains the completed coder, so the collapsed root shows
  `TALE DONE` even though the monitor is running.
- A direct-root monitor does not expose the defect, which is why existing monitor-row
  and family-member tests pass: they construct the monitor with the family root as its
  parent.

The durable direct-starter relationship is useful monitor lifecycle data and should not
be rewritten or migrated. The defect is the TUI's assumption that causal artifact
parentage and family display parentage are always identical.

## Implementation

1. Add a pure, idempotent family-projection normalization in the TUI agent model before
   `children_by_parent_timestamp` and root status overrides are computed.
   - Build an O(n), project-scoped lookup of loaded family-root rows keyed by durable
     family identity. Include enough identity context to prevent equal family names in
     different projects (or distinct clan generations, where applicable) from colliding.
   - For monitor-member rows only, resolve the loaded family root from `agent_family`.
     When the monitor's persisted parent is another member of that same family, use the
     root timestamp as the monitor's in-memory display parent.
   - Leave direct-root monitors unchanged. If no unique loaded root can be resolved,
     preserve the persisted relationship and fail safely rather than attaching the
     monitor to an unrelated row.
   - Keep this normalization free of filesystem reads, subprocesses, and render-time
     work. It must operate only on the already-loaded snapshot so refresh cost remains
     linear and off the Textual event loop's render path.

2. Reuse the existing family status and ordering semantics after normalization.
   - A running monitor must be treated as the newest active family member, causing the
     root to mirror its configured start label (default `MONITORING`) and explicit
     `Running` bucket.
   - The monitor row must appear as a sibling in the family roster/order, retain
     `is_monitor == True`, and keep monitor-specific rendering and stop actions. The
     root remains an LLM agent/container (`is_monitor == False`) and keeps normal root
     actions.
   - A terminal monitor must continue to project its configured stop label and
     monitor-state-derived `Done` or `Failed` bucket until a later follow-up agent
     becomes the newest active member. Existing custom start/stop labels must remain
     authoritative.
   - Preserve established counting rules: a monitor is a real lane member but not an
     additional LLM agent. The lane/root can be Running because it is monitoring while
     concrete-agent totals do not gain a phantom agent.

3. Add regression coverage based on the real nested-monitor topology.
   - Add a status-normalization test with a plan/tale root, a completed code child, and
     a running monitor whose persisted parent is the code child's timestamp. Assert that
     the in-memory display relationship is rooted, the root displays `MONITORING`, and
     its effective bucket is `Running`.
   - Add loader/ordering coverage proving the monitor remains in the visible family row
     sequence instead of being stranded beneath an already-child code row.
   - Cover a terminal successful monitor, a terminal failed monitor, and a later active
     follow-up so the root transitions through monitor status without masking failure or
     overriding genuine resumed agent work.
   - Cover identical family names in separate projects (and repeated normalization) to
     prove the projection is scoped and idempotent.
   - Assert summary/lane counts preserve the non-agent monitor rule while classifying
     the active family lane as Running.

## Expected files

- `src/sase/ace/tui/models/_agent_status_family_core.py` (or a focused adjacent
  family-projection module) for the pure monitor display-parent normalization.
- `src/sase/ace/tui/models/_agent_status_apply.py` to invoke normalization before family
  relationship indexes and root mirroring are built.
- Focused tests under `tests/test_agent_loader_status_override_*.py` and
  `tests/ace/tui/models/` for status, ordering, visibility, isolation, and counts.

Do not change monitor supervisor/follow-up persistence unless implementation proves the
TUI cannot safely derive display parentage from the already-present `agent_family`; the
direct starter timestamp is intentionally retained for monitor settlement and chat
inheritance.

## Validation

1. Run the focused status-override, monitor-row, ordering/grouping, and summary-count
   tests changed by this tale.
2. Run `just install` before repository-wide commands, as required for an ephemeral
   workspace.
3. Run `just check` and resolve every caused lint, type, or scoped-test failure.
4. Re-run the focused regression tests after any correction made during `just check`.

No PNG golden update is expected: the fix reuses the existing `MONITORING` label,
styles, and row rendering rather than adding layout or visual vocabulary. If a visual
snapshot changes beyond the intended status/group placement, investigate it as a
regression instead of accepting it automatically.

## Acceptance criteria

- Reproducing the `00i.f0` topology while its monitor is running places the collapsed
  root in the Running group with status `MONITORING`, not in Done as `TALE DONE`.
- Expanding the family exposes the actual monitor member with monitor styling and
  controls.
- Monitor completion and follow-up launch advance the root deterministically through the
  monitor's terminal state to the resumed agent's active state.
- Existing and historical nested-monitor artifacts work without an on-disk migration.
- No monitor is counted as an extra LLM agent, no cross-project family is attached, and
  no new I/O or data-scaled render work is introduced.
