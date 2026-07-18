---
plan: 202607/big_epic_lander.md
---
 Can you help me add a new built-in `@big_epic_lander` model alias that is used for the epic lander agent when an epic plan file has `<N>` or more phases? `<N>` should default to `5`, but the user should be able to override this via a new sase configuration field (i.e. `5` should be the default value for this new field). The `@big_epic_lander` model alias should be configured to use `@epic_lander` as its default value. Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 