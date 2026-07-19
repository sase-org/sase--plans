---
plan: 202607/agent_cli_updates.md
---
 I want to add the ability to SASE to update all currently installed supported agent CLIs like Claude and Codex, for example. Can you help me implement this?

- We should be realistic but ambitious about our ability to do this reliably and make sure to point to the provider's canonical install/update documentation in error messages and on the new updates subtab that we add.
- We should add command line support for this functionality and support in the TUI.
- I will let you figure out how to implement command line support on your own. Make sure that the code that handles these updates is shared by the command line interface and the TUI.
- In the case of the TUI we should split the current updates tab on the SASE admin center panel into three sub-tabs, which remain on a tab called Updates.
- These tabs should be named Core, Plugins, and Agent CLIs.
- We should add a new `A` (update agent CLIs) keymap to support this functionality which should be available from all three of these sub-tabs.
- The new Agent CLIs sub-tab should give the user more granular control over which Agent CLIs they update. See how we do this for SASE plugins, which should be moved to the Plugins sub-tab, for inspiration.
- The existing `u` keymap should continue to work the same way it always has although we may need to update its documentation.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 