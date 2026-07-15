---
plan: 202607/agent_panel_collapse.md
---
 The agents tab in the TUI currently defines the `H` / `L` keymaps which are supposed to expand/collapse all agent entries. In practice these keymaps have never really worked right and I've never really used them. Can you help me repurpose them to expand/collapse entire agent panels instead?

- The `H` keymap should collapse the entire agent panel that contains the currently selected agent.
- The `L` keymap should reverse this operation for the selected, collapsed agent panel (note that this means that we must support selecting collapsed agent panels using the `J`/`K` keymaps).
- Make sure that collapsed agent panels still show their shortened agent counts, but collapsed panels should NOT contribute to the calculated agent list panel width.
- For example, consider the agents shown in #sshot. If the `refresh_docs.sase.2394f8305458.polish` sase agent were selected and the user presses `H`, I would expect the entire `#chop` agent panel to be collapsed and (since this is the widest panel) for the agent list panel width to be decreased appropriately. The `#chop · 3 [S1 W2]` text should still be visible on the collapsed panel.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
  