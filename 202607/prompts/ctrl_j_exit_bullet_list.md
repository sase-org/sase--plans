- **PLAN:** [../202607/ctrl_j_exit_bullet_list.md](../ctrl_j_exit_bullet_list.md)

 The `<ctrl+j>` keymap in the prompt input widget currently auto-inserts
a (properly indented) bullet if the current line belongs to a bullet. This is
normally what we want, but can sometimes be annoying since, when the user wants
to end the bullet list, they are likely to just hit `<ctrl+j>` and expect to be
able to do it that way. Can you help me make it so hitting `<ctrl+j>` twice
(i.e. `<ctrl+j><ctrl+j>`) works for this use-case by making it so the 2nd time
the user presses `<ctrl+j>` the current line (this line should have been created
by the first `<ctrl+j>` and should contain `- ` with some optional leading space
and no bullet contents) is cleared and a new newline is added (so the cursor
should be on a new line 2 lines below the line they were on before pressing
`<ctrl+j>` for the first time?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
