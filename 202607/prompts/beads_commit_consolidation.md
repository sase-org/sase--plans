- **PLAN:** [../202607/beads_commit_consolidation.md](../beads_commit_consolidation.md)

 We make way too many commits to sase's `beads` sidecar repos currently.
Can you help me start committing to this repo less?

- It is important that we commit and push by default for all of the `sase bead`
  sub-commands but for commonly performed operations we should start
  consolidating into a single commit when possible.
- For example, when we run the `sase bead work` command, we know that we are
  going to mark all of the ready phase beads in that epic as in-progress and the
  rest of them (including the epic bead itself) as in-progress. So why not just
  do that (and commit and push once before launching any agents) and then let
  the other logic that we have elsewhere fall back to doing nothing in this
  case?
- Do some research / an audit to attempt to find other instances where we can
  reduce the number of commits we're making to this repo.
- Make sure that you have a good understanding of how sase beads are used before
  planning your implementation so you don't break any important behavior.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 