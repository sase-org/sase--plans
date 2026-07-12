---
plan: 202607/vcs_log_command.md
---
 Can you help me add new `sase vcs` command?

- This command will be used as a linked-repo aware version of the `git log` command, but MUST work with any supported sase VCS provider.
- When this command is run we should show a list of commits that were made in chronological order from either the current repo, any repo that is linked with the current repo, or the corresponding `sdd` / `<project>-sdd` repo.
- Make sure that it is clear which repo each commit that is listed belongs to.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  