---
plan: 202607/phase_plan_parent_links.md
---
 We recently added support for epics that have parents, but we forgot to add similar support for phase plans. Namely, can you help me start automatically adding the bead ID that the phase agent is working (add a new `bead` property) and the epic plan file's path (relative if possible, using a new `parent` property) to the plan file's frontmatter when that plan is proposed (this should be done automatically by the `sase plan propose` command)? Make sure that we also link the parent epic plan file's path to the new epic plan file when creating child epic plans as well. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 