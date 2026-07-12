---
plan: 202607/project_memory_opt_in.md
---
 The sase memory init command currently attempts to initialize a memory directory and overwrite any contents that exist in the AGENTS.md file (and other agent files like CLAUDE.md) for any project directory they are in. This is not correct because only main active sase projects should have their memory and agent files managed like this. I think the problem is that we don't have a good configuration field to control this from the project level. Can you help me add a new project local config field that needs to be set in order for sase to manage the AGENTS.md file? Keep in mind that any directory that has an AGENTS.md file should still have that file copied to all of the other agent files.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 