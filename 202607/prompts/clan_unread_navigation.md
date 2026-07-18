---
plan: 202607/clan_unread_navigation.md
---
 Can you help me add support for the unread indicator to agent clans (see the sase-6n epic bead for context)?

- Agents/agent families that live under clans show their unread notification when the clan is expanded but when the clan is collapsed the clan has no unread indicator.
- We should add an unread indicator to the clan row that always has a count associated with it when that clan has members that are unread.
- The `,j` keymap should auto expand a clan and select the relevant agent/agent family when triggered (and the most recent unread agent/agent family lives in that clan).
- Selecting a clan row should not mark any of the agents/agent families contained within that clan as read.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 