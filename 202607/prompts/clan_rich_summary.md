---
plan: 202607/clan_rich_summary.md
---
 I want to start supporting the ability to initialize an agent clan with a set of rich enabled text (figure out some way to serialize Rich's/Textual's syntax so users can effectively format text however they want) that gets displayed for that clan's summary in the agent metadata panel. Can you help me implement this as an argument to the `%clan` xprompt directive?

- We should support either a multiline string argument, which xprompt directive's should support a double colon syntax shorthand for, or the path to an executable script that outputs the rich text.
- We should write a built-in script that we use to give to the epic clans that we create. The script should output text that renders beautifully in the TUI.
- We should also use the other syntax of a plain string to just show the research prompt (prefixed with a bold RESEARCH PROMPT: ) that was provided as an input argument to the research_swarm xprompt, which is defined in my chezmoi repo.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 