---
plan: 202607/automate_sdd_companion_repo.md
---
 We recently added support for storing SDD files in a separate GitHub repo. Can you help me make sure that the creation of this GitHub repository is automated by the `sase sdd init` and then also by the `sase init` command, which wraps the former command?

- This GitHub repo should be in the same organization as the current repo (determined by your working directory) and should be named `<project>--sdd` (ex: `foo-org/foo--sdd` if your in a directory corresponding with the `foo-org/foo` GitHub project).
- We currently search for the `<project>-sdd` and `sdd` repo names by default I believe, so you should change that so we search for the `<project>--sdd` and `sdd` repo names instead.
- Make sure we display good output to the user while creating the repo and handle errors gracefully and intuitively.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  