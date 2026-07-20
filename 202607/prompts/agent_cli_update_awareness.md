---
plan: 202607/agent_cli_update_awareness.md
---
 Can you help me make sure that when we check for updates (every 10m or at startup) that we also check for any agent CLI providers that need updates?

- The `,U` keymap should update these agent CLI providers as well, if and only if, a previous background update check has determined that one or more agent CLIs need to be updated.
- Make sure we are smart about this and don't hurt performance in any way.
- Also, make sure we make the visual indicator that shows whether updates are available (shown on the top right of the TUI) look distinct somehow when one or more agent CLIs need updates.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
