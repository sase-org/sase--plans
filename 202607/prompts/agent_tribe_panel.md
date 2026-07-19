---
plan: 202607/agent_tribe_panel.md
---
  Can you now take inspiration from the agent family
and agent clan summaries on the agent metadata panel to create a similar summary
that should be shown in the agent metadata panel when an Agent tribe is
selected?

- Make sure that tribe panels support all of the advanced functionality that is
  supported by family panels and clan panels (e.g. numbered keymaps to jump to
  members, section folding, etc...).
- Agent tribe panel summaries should support 4 levels of folding. Currently,
  both agent families and agent clans support just 3 levels of folding. This
  works well for agent clans, but the 1st level of folding for agent families is
  unnecessary, so let's start just using 2 levels of folding for agent families.
  Agent tribe panels, however, probably need even more folding than agent clans.
  Get creative here but also make sure that each fold level provides real value.
- Currently it is only possible to select an agent tribe panel when that panel
  is collapsed.
- I want to make it possible to select these panels when they are expanded as
  well.
- We can accomplish this by allowing the keymap to select the current tribe
  panel when there is nothing else to collapse (i.e. when it would
  normally/currently do nothing).
- Make sure that the j/k/l keymaps behave as you would expect when an expanded
  Agent Tribe Panel is selected. In other words, we should navigate to the
  next/previous agent tribe panels and select that panel, for the j/k keymaps,
  or select the previously selected (or top, if we don't know the previously
  selected) agent/agent family/agent clan in the currently selected agent tribe
  panel.
- Let's start using `h` to collapse expanded agent tribe panels now too, since
  we can now select these panels. Therefore, we don't need the `H` keymap
  anymore; however, there is a problem with this plan: The `h` keymap currently
  also collapses groups (ex: `Running`--these are dependent on the current
  grouping strategy). Let's solve this problem by using a new `H` keymap that
  handles this type of fold/collapse (the `l` keymap can still be used to expand
  these groups).
- With regards to the related, `,H` keymap, let's start rendering hints for ALL
  collapsable/expandable elements that are currently visible. This should
  include all agent tribe panels, agent clans, agent families, and agents. Try
  not to display hints for multiple entries that would result in the same action
  (e.g. two hints for child steps that would collapse the containing agent
  should just not be shown; show a hint next to the parent agent instead).
- Also, make sure we show hints for expanded agent tribe panels (the panels
  themselves) when the special apostrophe keymap is used.
- Make sure that we make a selected agent tribe panel that is expanded visually
  distinct since otherwise it would be difficult to tell that it is the tribe
  that is selected and not an agent within that tribe.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 