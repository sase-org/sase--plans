---
plan: 202607/family_plan_root_rename.md
---
 We recently added the distinction between agent clans and agent
families but The main goal was essentially to leave agent families the way they
were and add support for this new concept of agent clans (see the sase-6n epic
bead for context).

- This seems to mostly work as planned but we've started to use the `--plan-0`
  suffix instead of `--plan` for agents that proposed a plan that the user
  approved and the root / top-level agent row no longer keeps the original agent
  name (see ~/tmp/screenshots/20260718_063757.png). This is not correct.
- When agent families are created they should take on the name of the first
  agent that was in that family and the first agent should be renamed (in this
  case, we should use the `--plan` suffix).
- Agent family names must be supported as input arguments for the `#fork`
  xprompt and the `%wait` directive (the previous bullet allows for this, but
  make sure it actually works).
- Make sure that all other previously supported standard ways that we create
  agent families (e.g. when the user answers a question, gives feedback on a
  plan, etc...) are supported / working as they were from the user's standpoint.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  