- **PLAN:** [../202607/model_override_effort.md](../model_override_effort.md)

 Can you help me make some improvements to the model override/edit panel
(see #sshot for what this looks like right now)?

- Let's start showing all model aliases below builtin model names.
- Let's start prompting the user for an effort level after they select a model /
  model alias. Make sure that the currently configured default effort level is
  selected by default. We should already have an existing panel that presents
  effort levels to users so you should reuse that.
- Also, make sure that specifying an effort level (via the
  `<model>@<effort_level>` syntax) works when inputing the model name manually
  via the `Custom` option (I don't think this is working currently).
- As a part of this change, we will need to start support for the
  `<model>@<effort_level>` syntax when `<model>` is a model alias instead of
  just a model name / `<provider>/<model_name>` spec. For example, since
  `@default` has the value `codex/gpt-5.6-sol` on this machine,
  `@default@medium` should resolve to `codex/gpt-5.6-sol@medium`. If the model
  alias already defines an effort level explicitly, using the
  `@<model_alias>@<effort_level>` syntax should override that effort level to
  `<effort_level>`.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
