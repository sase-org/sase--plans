- **PLAN:** [../202607/wait_field_lanes.md](../wait_field_lanes.md)

 Can you help me make the `Wait:` field in the agent metadata panel look
much better when we are waiting for agents and beads?

- See #sshot for what this can look like now.
- Let's start prepending `[agents] ` before the first agent (so `sase-ae.1` in
  the screenshot).
- We should also remove the `+` and put the rest of the content (starting with
  `[beads]`, which should replace `beads:`) on the next line.
- Make sure that this 2nd `[beads]` line is properly indented/aligned.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 