- **PLAN:** [../202607/plan_header_provenance.md](../plan_header_provenance.md)

 Can you help me add some new bullets to the top of plan files?

- We do this to link to (and from) prompt files currently.
- Let's start adding a new bullet that links to any commits, as sub-bullets,
  that are associated with that plan file (this can be multiple commits for a
  single coder agent that worked a tale plan and is almost certainly multiple
  commits for an epic plan file).
- Let's start adding a new bullet that links to any agents, as sub-bullets, that
  are associated with that plan file (again, this can/will include multiple
  agents sometimes--especially when the plan file has `tier: epic`).
- We currently add a file reference to the parent plan file using a `parent`
  property in the child plan file's frontmatter. Let's stop doing this in favor
  of adding a new bullet that links to the parent (since GitHub does not render
  hyperlinks in frontmatter properties).
- Make sure every bullet/sub-bullet that references to a commit/agent/plan links
  to the relevant GitHub URL.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 