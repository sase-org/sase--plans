---
plan: 202607/dynamic_research_xprompt_paths.md
---
 I'm pretty sure we have some xprompts that still instruct agents to write to the sdd/research/ directory unconditionally. That is not correct. I think we have a `sase sdd path` command that we should be using instead (use the xprompt shell expansion syntax to run this command). Can you help me find the xprompts that need to be updated and update them? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 