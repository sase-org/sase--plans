- **PLAN:** [../202607/validate_explain_option.md](../validate_explain_option.md)

 One of the goals of the `sase plan validate` command was to make sure
that agents using the /sase_plan skill, which is used very frequently by agents,
do not load too much information about the expected frontmatter properties of
plan files up front. Instead they should only get that information when they run
the `sase plan validate` command. Can you help me implement this using the
recommended solution below?

- Let's add a new `--explain` option to the `sase plan validate` command that
  runs the same validation, but also outputs useful information (base on the
  tier) that used to be the /sase_plan skill.
- The `--explain` option should cause the command to output a preamble before
  its validation results. The ~/tmp/tale_desc.md file contains what this
  preamble should contain if the plan is a tale. The ~/tmp/epic_desc.md file
  contains what the preamble should contain if the plan is an epic.
- The agent will only be told to add one property to the plan file frontmatter,
  `tier: <tier>`, where `<tier>` is either `tale` or `epic`. This means we can
  remove the `--tier` option from the `sase plan validate` command, since that
  command should be able to figure that out from this property (make sure we
  output a useful error message for agents in case they forget to add this
  property).
- If/when the finalizer asks you (make sure to include this in your plan for the
  coder agent) to commit, you should re-apply commit e0aa2327bb14 (by reverting
  the revert: 8634bbc6670d) after making your new commit to change the
  /sase_plan xprompt skill the way I want. After doing that, run the appropriate
  init command to initialize the changes to the skill and then terminate. Make
  sure to review git commit e0aa2327bb14 before deciding on your solution so you
  are sure that we are aligned. Do NOT modify the /sase_plan skill yourself.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 