---
tier: epic
title: Clear the agent-session terminology regression in closed-bead landing
goal: The close-attribution epic passes its terminology contract on the integrated
  tree, with no remaining failures caused by sase-19p.
parent_bead: sase-19p
phases:
- id: terminology
  title: Remove the stale family identifier introduced by the close-plumbing phase
  depends_on: []
  size: small
  description: 'terminology: remove the retired family wire-key mention from the runner-slot
    capacity projection docstring and verify the source terminology contract.'
proposed_by: bbugyi200.athena.sase-19p.land
create_time: 2026-09-25 21:50:28
status: wip
bead_id: sase-19p.4
---

- **PROMPT:** [prompts/202609/agent_closed_bead_landing_cleanup.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_closed_bead_landing_cleanup.md)
- **PARENT:** [202609/agent_closed_beads.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_closed_beads.md)
- **BEAD:** [sase-19p.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19p/sase-19p.4.md)

# Finish agent-closed bead landing after terminology regression

## Context

Epic sase-19p has three closed phases and the close-attribution feature is implemented
in sase-core and sase. The land review found one deterministic epic-caused failure on
master:
`tests/test_agent_session_terminology.py::test_current_source_avoids_agent_family_identifiers`.
Commit `ed548d3e2` (phase sase-19p.2) added a `agent_family_parallel` mention to the
docstring in `src/sase/core/runner_slots/_admission_capacity_records.py:154-158`. The
test reports that line as its sole finding. The same line remains on master `4ee966cd5`.

The `sase tool run check` at `013a17072` (ToolRun `0962fffefcd94a0b59720955cda3794c`)
passed lint, SASE validation, and committed-plan validation, then failed in the full
non-visual pytest lane. The other reported failures concern active queue-capacity, Node
Finder, triage, completion, or timing work and are outside this epic. The queue-capacity
`AgentInfo` errors are already documented on `sase-19f`; the Node Finder marker audit is
already documented on `sase-19i`. The new master has subsequent unrelated changes;
recheck the gate on that tree.

## Phase `terminology` — remove the stale identifier and verify

1. Remove the stale identifier from the capacity projection docstring. The docstring
   should explain the current session fields without mentioning the retired family
   spelling. Do not change emitted wire keys, aliases accepted by Rust, or the close
   feature.
2. Run the focused terminology test and focused close-feature tests:
   `tests/test_agent_session_terminology.py`,
   `tests/core/test_bead_touch_index_facade.py`,
   `tests/test_bead/test_cli_close_note.py`, `tests/test_bead/test_cli_touched.py`, and
   `tests/ace/tui/widgets/test_agent_bead_touch_rows.py`.
3. Run `sase tool run check` on the integrated tree. If its full lane remains red,
   establish which failures are unrelated using the existing active-epic notes and
   source provenance. Any failure caused by sase-19p remains this child plan's work.

## Landing handoff

This child plan is directly parented by sase-19p so its land agent can resume the
interrupted epic landing. The parent land agent owns closing sase-19p, the post-close
`just symvision` pass, and setting the parent plan's frontmatter status to done. Those
actions are not child phases.
