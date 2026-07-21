---
plan: 202607/warn_custom_builtin_aliases.md
---
 When loading the "Models" panel in the TUI, can we start showing a warning message / indicator when the user has defined a custom alias that overrides a builtin alias? Advice the user that they should override the model aliases value using the `llm_provider.model_aliases.builtin` config field instead of `llm_provider.model_aliases.custom`. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
