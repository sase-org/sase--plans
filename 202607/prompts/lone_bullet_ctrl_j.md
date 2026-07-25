- **PLAN:** [../202607/lone_bullet_ctrl_j.md](../lone_bullet_ctrl_j.md)

 Pressing `<ctrl+j>` in the prompt input widget when there is only a single bullet should not clear the line and add a newline (like it should when there are one or more bullets defined above the current empty bullet). See #sshot for an example of what I'm talking about. Pressing `<ctrl+j>` at that moment should not clear the line, it should insert a new `- ` line below that line. If the user pressed `<ctrl+j>` again at that point, then that new line should be cleared and a newline added. Can you help me fix this?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
