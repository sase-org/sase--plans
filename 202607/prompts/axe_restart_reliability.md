---
plan: 202607/axe_restart_reliability.md
---
 Something seems wrong with lumberjack chops or maybe with `sase axe`
itself. I've been noticing, for example, that I often just stop receiving
Telegram messages from sase (and it stops replying to my Telegram messages).
During this time, agents that are waiting but should have been started by now do
not get started (a chop is supposed to be responsible for starting these waiting
agents I think--you should double check). Can you help me diagnose the root
cause(s) of these issues and fix them?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 