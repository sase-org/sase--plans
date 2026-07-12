---
plan: 202607/max_running_agents.md
---
 I want to add the ability to configure a maximum number of sase agents that are allowed to be running at any given time. Can you help me implement this?

- I want the user to be able to override this both in their personal config but also on a prompt-by-prompt basis using an xprompt directive.
- I think we can accomplish this by using a new `runners` keyword argument to the existing `wait` directive, so users can override this in prompts (a prompt that sets runners to 0, for example, would wait until there are no agents running before it runs). 
- We should have a sase configuration field that defaults to 10 for this.
- While waiting for other agents to finish, we should use the standard waiting agent status for this, but make sure that it is clear at a glance that an agent is waiting because of this new configuration / `wait` key word.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

 