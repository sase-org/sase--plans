---
plan: 202607/phase_bead_context_lane.md
---
 We currently show a phase agent's phase bead using the `Bead:` field
which is rendered at the top of the agent metadata panel on the "Agents" tab of
the `sase ace` TUI. Can you help me start rendering this information in a new
`BEAD` lane in the existing `SASE CONTEXT` section in this panel instead?

- This lane should render above all other lanes.
- In addition to the phase bead's ID and description (shown in the `ID:` and
  `Description:` sub-fields under the new `BEAD` lane), we should also render an
  `Epic Plan:` field, which should have a value of the epic plan file path
  (relative--e.g. sase/repos/plans/202607/some_epic_plan_file.md), and
  `Epic Title:`, which should have a value equal to the value of the `title`
  property in the epic plan file's frontmatter.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
