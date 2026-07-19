---
plan: 202607/tribe_panel_display_config.md
---
 Can you help me add some new sase configuration fields that allow me to control what icon, if any, is shown before an agent tribe's name on the agents tab when that panel is visible?

- We should also add a configuration field that allows us to control whether an agent tribe panel of a particular name is rendered as initially expanded or not. This should default to true.

- In sase's default config we should configure the `@default` and `@epic` tribes (if `@default` has not yet replaced the "untagged" panel, it will soon) to use this new functionality.
- Both of these tribes should be given good default icons that make it easy to distinguish them at a glance.
- Also I frequently use tribes that are named `@research` and `@chop`. These should be given excellent icons as well.
- The chop tribe should be collapsed by default when initially loaded (any user change to this expansion state should be remembered and prioritized over the initial expansion state configuration).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 