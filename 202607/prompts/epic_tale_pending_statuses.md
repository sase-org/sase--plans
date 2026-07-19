---
plan: 202607/epic_tale_pending_statuses.md
---
 Can you help me start using a new `EPIC` agent status that should be
shown for agents on the agents tab that proposed an epic plan file instead of
`PLAN`?

- Also, instead of showing `PLAN` for tale plan files, let's start showing
  `TALE` instead.
- Agent statuses are used frequently throughout the codebase so make sure that
  you search for and address all edge cases.
- Also if you see an opportunity to consolidate or factor out any shared agent
  status functionality, do so since it will likely help us in the future.
- Make sure that the new `EPIC` and `TALE` agent statuses each have distinct
  appropriate colors assigned to them.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 