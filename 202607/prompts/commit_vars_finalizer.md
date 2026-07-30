- **PLAN:** [../202607/commit_vars_finalizer.md](../commit_vars_finalizer.md)

 Can you help me make some improvements to the `sase commit` command and
the corresponding /sase_git_commit xprompt skill?

- Instead of accepting file paths as arguments for files that should be added to
  the commit, we should instead have agents specify any files that they want
  excluded from the commit. All agent file changes should be isolated to that
  agent's workspace directory at this point. If there are any files that were
  added, removed, or changed that the agent did not make, there's probably a
  problem that we should be aware of.
- Let's add a new `--vars` option to the `sase commit` command that acts as a
  wrapper around the `sase var` command by setting all of the necessary sase
  variables to support the feature in the next bullet.
- Let's start having the finalizer request that the agent use the `sase commit`
  command's `--vars` option, which should be briefly explained in the
  /sase_git_commit xprompt skill.
- To support this command, we will need to make sure that sase variables support
  list values (i.e. for excluded file paths). They might support these already
  but I'm not sure.
- sase variable values already support multiline strings, which we should use to
  set the commit message variable (make sure we still delete the commit message
  file, if that's how the agent provided the commit message).
- The finalizer should then use these sase variable values to make the
  appropriate commit(s). The agent completion / agent family completion (e.g.
  the agent completion notification and the agent status change) should not
  occur until after the commits have been made.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 