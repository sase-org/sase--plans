---
plan: 202607/chops_redesign.md
---
 Can you help me drastically change the way that chops are expected to
be configured by users with the goal of making chops more powerful, structured,
and configurable?

- Sase currently supports two different types of chops: agent chops and script
  chops. We're going to completely remove the agent chop variant and only
  support script chops.
- We should remove the constraint that these chop script names start with
  `sase_chop_`. They should be allowed to name the scripts whatever they want
  and we will reference the full script name in config.
- To support running sase agents we will read a structured JSON output from chop
  scripts, which, if found, will resolve to a list of proposed sase agent
  launches.
- We should no longer run xprompt workflows like `#!sase/refresh_docs` (even
  from structured agent launch proposal output). Instead, the logic from these
  workflows (see which ones are configured in my chezmoi repo) should be either
  (ideally) translated to a shared functionality provided by sase and requested
  via a specific chop configuration or migrated to a chop script that is defined
  in the bugyi-chops repo (see the bullet below).
- We should move all chop scripts (though many will need modifications) that are
  not used by built-in sase chops (some of these are defined in this repo, like
  sase/xprompts/refresh_docs.yml, and others are in my chezmoi repo) to the new
  bbugyi200/bugyi-chops GitHub repo, which I have already created.
- You should make extensive improvements to the `axe.lumberjacks` configuration
  entries. See the lumberjack_chop_config_improvements.md research file for
  context on the following requirements:
  - All of the P0's from that file should be implemented except for P0-3.
  - P1-6 and P2-10 should also be implemented.
  - We need to support something like what P1-5 recommends, but I definitely
    want an easy way to configure a single chop to run against all enabled sase
    projects, so make sure your solution (which should be generic and not
    overfit to this use-case) supports this.
- The bugyi-chops repo should be given an excellent README and should be
  converted into a Python package that gets published to PyPI. Also you should
  add the `sase--plugin` GitHub label to this GitHub repo.
- One of the goals of having Chop scripts produce structured output instead of
  running agents themselves is to make it easy to run Chops manually to debug
  them. Make sure you keep this use case in mind and support a verbose mode for
  chop scripts that allows them to produce good logging/debugging output.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 