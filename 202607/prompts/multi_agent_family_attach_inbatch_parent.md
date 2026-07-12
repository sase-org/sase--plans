---
plan: 202607/multi_agent_family_attach_inbatch_parent.md
---
 If I attempt to use the `%n(foo, bar)` directive on the 2nd agent in a multi-agent prompt where the first agent specified `%n:foo` for its name, this fails since the "foo" sase agent doesn't exist at the time we try to launch the 2nd agent. It's important that we be able to add custom agent family members like this from multi-agent prompts. Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
  