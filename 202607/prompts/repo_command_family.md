---
plan: 202607/repo_command_family.md
---

#fork:sase-5w Can you now help me add a new sase repo command?

- Add a list subcommand. Make sure this command lists all repos for the current project or for all projects (use a CLI option to make this configurable) and that every linked repo and sidecar repo is shown in the output, along with whether or not a particular project workspace has that repo cloned.
- The sase workspace open command should be migrated to a new sase repo open command. Make sure you update all agent instructions as necessary.
- We should also add a new sase repo log command that shows a log of which agents opened which repos and for what reasons. See how the sase memory log command handles this for inspiration.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

 