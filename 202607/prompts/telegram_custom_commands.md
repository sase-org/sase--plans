---
plan: 202607/telegram_custom_commands.md
---
  Can you help me add support for custom, user-defined telegram slash commands that can be defined in sase's config?

- As a first use case define a  /tasks command that sends a PDF (with a nice telegram message that describes the contents of the PDF) of the tasks query results from the tasks queries in the ~/bob/dash.md file. See the /bob_query skill for inspiration on how this might work. 
- Any new scripts that you define in order to implement this should be defined in my chezmoi repo. The necessary configuration changes to enable this new slash command should also be made to the appropriate files in my chezmoi repo. 
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 