---
plan: 202607/telemetry_inhouse_graphs.md
---
 This project is supposed to have great telemetry with good support for Grafana, for example, but it has never worked right. Can you help me migrate the telemetry graphs, in particular around sase agents, to an in-house solution?

- We can use a low-level framework but should remove all Grafana support and references.
- These new, in-house Telemetry graphs should be viewable from the CLI using a sase command (we might even already have a command for this but it doesn't currently work well) but they should also be visible via a new Telemetry tab that we add to the SASE Admin Center panel in the TUI.
- Take extra care to make sure this change does not hurt performance too much.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 