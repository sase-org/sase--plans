- **PLAN:** [../202607/wait_bead_statuses.md](../wait_bead_statuses.md)

Your previous attempt hit a model context limit or transient provider failure. Any file edits, new tests, and other on-disk changes you made are preserved. Before making additional changes, run `git status` and `git diff` to see what is already in place, then continue implementing the plan from wherever you left off. Do not re-apply edits that are already present.

 Can you help me start showing all bead statuses for the beads an agent is waiting for in the `Wait:` field of the agent metadata panel? For example, in #sshot, we should show a checkmark next to the right of `beads: sase-9r.2` since the `sase-9r.2` bead is closed. Make sure this doesn't hurt the TUI's performance (beads can be slow). Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 