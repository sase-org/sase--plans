---
plan: 202607/custom_notification_gates.md
---
 #fork:sase-6e Can you help me make some improvements related to this
functionality?

- Make sure that user feedback is supported gracefully for every one of these
  custom notification types, even ones we define in the future.
- We should already be representing custom notification type actions like
  approve or reject as commands that are run when the user selects those
  options. These command options are essentially ANDed, but we need to support
  ORed commands to do the user can choose to run more than one command proposed
  by a custom gate notification. This should be used by plan notifications to
  specify whether we commit the plan file to our sidecar repo or not. I'm not
  sure how or if we are supporting this functionality of giving the user an
  option to commit the plan file right now but if we do support it we should
  probably remodel the implementation to fit this conceptualization. If we
  didn't implement it in the last epic then we should implement it now.
- We should expose a new xprompt skill named `/sase_gate` to agents that allows
  them to create beautiful, robust, and powerful custom notification types on
  the fly. Make sure the description of this skill does it justice but also
  mention that it is an easy way for agents to propose commands (ex: dangerous
  commands or commands that the user specifically requested be confirmed before
  use) to be run.
- This is likely already being done but make sure that the commands we run after
  the user submits an option for a custom notification type are run as
  background tasks. This is so the TUI is not blocked and will also allow the
  user to view the status and outcome/output of these commands in the tasks tab
  of the SASE admin center panel.
- The `/sase_gate` skill should instruct agents to select an icon that will be
  displayed in the command notification. I don't think the `sase notify create`
  command supports this currently, so you might need to design/implement support
  for this.
- Make sure that the sase-telegram plugin has excellent support for these new
  custom notification types, all of their custom actions/commands, and any
  improvements you make to them with this new feature.
- Make any other objective improvements (and fix any bugs) that you can find
  which are related to this functionality and that you are confident in.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
