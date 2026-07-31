- **PLAN:** [../202607/admin_center_alternate_tab.md](../admin_center_alternate_tab.md)

 Can you help me add support for a new `#` (last tab) keymap to the
"SASE Admin Center" panel when a tab is selected (i.e. not when the initial
landing page is shown--in which case, the `#` keymap has a different purpose)?

- When this keymap is triggered, we should focus the last tab that the user had
  focused on this panel before the tab that is currently selected.
- We should only remember the very last tab that was selected (really, we need
  to remember the last two, since the landing page uses the first one), so
  pressing `#` repeatedly should just jump back and forth from the same two
  "SASE Admin Center" tabs.
- Make sure this keymap has an excellent visually distinct footer description in this panel.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 