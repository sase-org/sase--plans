---
plan: 202607/bulk_kill_edit_waiting_name_reuse.md
---
 When I kill and retry an agent with the `,x` keymap on the agents tab and that agent was given a name explicitly in its prompt, we correctly seem to add the `!` character after the `%n`/`%name` directive, but we don't seem to do this for the prompts that are added to the prompt input widget after selecting multiple agents using the `m` keymap and then using the `,x` keymap to kill and retry all of them. Can you help me confirm/deny my suspicion and fix this issue? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.


### Additional Requirements

- This might only occur when the agents that I have selected have a waiting status.