- **PLAN:** [../202607/model_alias_completion.md](../model_alias_completion.md)

 Can you help me add model aliases (ex: `@default`) to the completion
menu that is shown from the prompt input widget (and from editors via LSP
support, I believe) when `%m:` or `%model:` is typed?

- Make sure it is very clear that these are model aliases in the menu that is
  shown. Make sure we show what that model alias resolves to with rich context
  information about how it was configured. See how the "Models" panel in the TUI
  handles this for inspiration.
- Prioritize these beneath normal model names but if the user types `@` after
  `:`, then we should just how model aliases.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 