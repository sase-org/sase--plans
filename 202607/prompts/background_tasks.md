- **PLAN:** [../202607/background_tasks.md](../background_tasks.md)

 Can you help me add a new `sase task` command that can be used to list,
monitor, and run new sase background tasks (these are the commands that sase
runs in the background and makes available from the "Tasks" tab of the "SASE
Admin Center" panel)?

- Make sure this new command supports the following subcommands:
  - `list`: Lists all stored background tasks. We should start storing all
    background task up to some configurable (you should add a new sase config
    field for this that defaults to 100) limit. Make sure this command supports
    a `-n|--limit` option and that we show which TUI session (figure out how to
    display this) each task originated from.
  - `show`: Should take a single task ID and show useful information about that
    task.
  - `run`: Should be used to run a new background task in a particular TUI
    instance/session.
- Make sure all of these sub-commands have CLI options that make them more
  flexible and useful for both humans and agents.
- For our first use-case, I would like to start adding transparency back to the
  `sase bead work` command (and other commands to create the proper beads) that
  is run when a user approves an epic (e.g. from Telegram or from a TUI sase
  gate notification panel). Let's start running this command as a background
  command using the new `sase task run` command.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
