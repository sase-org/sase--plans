---
plan: 202607/vcs_log_github_remote_sourcing.md
---
 It seems like the `sase vcs log` command currently uses the git commit history from the local directories associated with the corresponding repos. This is not correct. We should construct our list of commits from the latest state of the GitHub repo. Can you help me fix this? You can use the primary repo directory locations for this and just fetch the right commits without changing any files or the working state of those directories I think, right? Make sure it is made clear which commits exist in the current primary checkout of the corresponding repo as compared to only existing on the GitHub remote.

I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  