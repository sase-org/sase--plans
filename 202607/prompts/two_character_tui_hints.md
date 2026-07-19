---
plan: 202607/two_character_tui_hints.md
---
 Can you help me add support for two-character hint values in the TUI?

- We should use `00` to start, then `01`, up to `09` and then `0a`, all the way up to `ZZ`.
- We should only use two-character hints in the TUI when we need to. For example
  we should show two-character hints when using the special apostrophe keymap if there are more than 62 agents
  shown on the agents tab. Otherwise we should continue to showing one-character
  hints.
- Let's start making the first hint shown for one-character hints `0` instead of `1`, just to be consistent.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
