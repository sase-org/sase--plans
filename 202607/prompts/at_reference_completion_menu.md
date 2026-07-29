- **PLAN:** [../202607/at_reference_completion_menu.md](../at_reference_completion_menu.md)

 We currently trigger artifact completion in the prompt input widget and
LSP completion menu (for external editors) after the user types two characters
after `@` (i.e. if they type `@pl`, they will see `@plan` in the prompt input
widget completion menu). Can you help me change this so `@` immediately triggers
this completion menu?

- We should also show local (to the current working directory) file paths, but
  these should be clearly (in a visually appealing way) separated from artifact
  prefixes (like `@plan` or `@chat`, for example).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 