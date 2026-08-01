- **PLAN:** [../202608/artifacts_beads_and_files_subtabs.md](../artifacts_beads_and_files_subtabs.md)

 The "Plans" sub-tab of the "Artifacts" tab is a bit of a misnomer since
it contains information about plans AND beads (but seems to focus on beads more
than plans). Can you help me fix this by re-organizing the "Artifacts" tab's
sub-tabs?

- Let's move the "Plans" and "Chats" sub-tabs to new sub-sub-tabs of the "Files"
  sub-tab.
- The user should be able to cycle through these using the new `(` and `)`
  keymaps (see how the existing `[` and `]` keymaps work for inspiration).
- The current contents of the "Files" sub-tab should be moved to an "Other"
  sub-sub-tab.
- Make sure that you redesign the plans sub-sub-tab so it is completely
  dedicated to plans. Though plan files that link to beads should make a keymap
  available that allows the user to jump to the corresponding bead on the beads
  subtab (see the below bullet).
- We should also add a new "Beads" sub-tab to the "Artifacts" tab that is
  dedicated to beads only, with a keymap to jump to corresponding plan files
  (when applicable). Make sure this tab supports the same functionalities that
  the current "Plans" tab does, but also add excellent support for task beads
  with the ability to close them with a reason, which should also dismiss the
  corresponding bead notification.
- The user should also be able to make arbitrary updates to beads from this
  panel and have a way of filtering for beads based on status and type and other
  things (use your best judgment here).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 