---
plan: 202607/placeholder_completion.md
---
  Can you help me add support for placeholder completion to the prompt input widget and to editors (via LSP support--e.g. make sure that users that have the sase-nvim Neovim plugin installed have access to this functionality within Neovim)?

- A placeholder is some string of the form `<placeholder text>`.
- We should start triggering completion whenever the user's cursor is inside of `<>` brackets.
- Make sure this completion is also triggered when the user uses a sase snippet that places the cursor inside these brackets (see the `cbi` snippet defined in my chezmoi repo, for example).
- When this type of completion is triggered, we should include all previously found (in the current prompt) placeholder text (i.e. text that was found inside `<>` brackets). Therefore, this completion should only trigger when placeholders have already been used elsewhere in the prompt.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 