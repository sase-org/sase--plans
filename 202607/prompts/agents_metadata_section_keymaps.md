---
plan: 202607/agents_metadata_section_keymaps.md
---
 Can you help me add new `<ctrl+j>` and `<ctrl+k>` keymaps to the "Agents" tab of the `sase ace` TUI?

- The `<ctrl+j>` keymap should jump to the first (if this is the first time the user pressed `<ctrl+j>`) or next (otherwise) section in the agent metadata panel and scroll the viewport such that the section title is the top line shown in the right pane of the TUI.
- The `<ctrl+k>` keymap should do the same thing but in reverse (i.e. go to the last or previous section in the agent metadata panel).
- Make sure both of these keymaps support cycling to the beginning/end of the file (e.g. if the `<ctrl+j>` keymap is used when we are already at the last section, then we should jump to the first section).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 