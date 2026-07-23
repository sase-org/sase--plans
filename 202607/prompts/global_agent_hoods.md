- **PLAN:** [../202607/global_agent_hoods.md](../global_agent_hoods.md)

 We recently added support for a new `--agents` hidden sidecar repo (see
the sase-8k epic bead for more context), but I made several msitakes when
describing the desired outcome. Also, I've thought of several improvements we
should make. Can you help me make these changes?

- I'm pretty sure we are writing the machine agent hood to disk on the machine
  that is assocaited with that hood. That is incorrect. We should only be
  persisting the machine agent hood when saving agent data to the new `--agents`
  sidecar repo.
- `--agents` Sidecar Repo Needs More Agent Data
  - We are currently writing (and commiting and pushing) only the data
    corresponding with the sase agent that actually made commits to the
    `--agents` repo, but we should be storing all agents in that agent's family
    (if applicable) and all agents in that agent's top-level hood, recursively.
  - We also need to start saving enough data so that remote machines can sync
    and then reconstruct entire agent families (e.g. by reviving them using the
    `R` keymap on the agents tab).
  - For example, if sase agent `foo.bar.baz--code` made a commit, then the
    following agents (assuming they exist--or agent families, in the case of
    `foo.bar.baz`) would be committed to the `--agents` repo:
    - `foo.bar.baz--plan`
    - `foo.bar.baz`
    - `foo.bar`
    - `foo`
    - `foo.boom`
    - `foo.boom.bam`
    - `foo.bar.kazam`
    - etc...
  - We should start also pushing a beautiful agent family overview markdown file
    whenever a new agent family is pushed. This will be linked to from sase
    commits (see below bullet).
- We should completely remove support for (i.e. stop adding) the `SASE_MACHINE`
  sase commit tag in favor of using the full agent name (including the
  `<username>.<machine_name>.` prefix mentioned in a below bullet) for the
  `SASE_AGENT` commit tag's value. This value should also be turned into a
  hyperlink that points to the appropriate GitHub agent / agent family markdown
  file (see how we do this for plan files with the `SASE_PLAN` commit tag for
  inspiration and context--also, see a previous bullet for the agent family
  markdown file that I'm referring to).
- Unique Machine Identities Per-Project
  - We need to start disambiguating different machines belonging to different
    users (or the same user, but probably not) that may have the same name when
    they both commit to the same project.
  - We can accomplish this by requiring that the user have an `id.username` sase
    config field defined.
  - The `sase config init` (and the `sase init` command, by extension) command
    should prompt the user for a value for this field if it is not defined and
    it should be added to the same file that the `machine_name` config field
    (which we should start grouping under the `id` field--i.e. rename to
    `id.machine_name`) is added to. Make sure we make the user aware that
    `id.username` MUST be unique across all sase users but SHOULD be the same
    across all of this user's machines (recommend that they use their GitHub
    username).
  - We can then start defining a new "Username Agent Hood" that contains all
    machine hoods defined by that user.
  - This username hood, like the machine hood, should only be written (and
    committed and pushed) to the `--agents` sidecar repo.
  - For example, all of the agents that are created by this machine should have
    `bbugyi200.athena.` prepended to their names. Make sure to edit my
    sase_athena.yml file (in my chezmoi repo) appropriately to migrate the
    `machine_name` field to `id.machine_name` and add the `id.username` field,
    which should have `bbugyi200` as its value.
  - Make sure we remove the `<username>.` prefix when sase writes to disk (for
    its internal representation of agents) from any machine configured to use
    that same `id.username`. Likewise, make sure that we strip the entire
    `<username>.<machine_name>` when both `id.username` and `id.machine_name`
    match.
- `--agents` Sidecar Repo Updates (push/pull)
  - We should start pushing all necessary new agent data automatically when an
    agent runs the `sase commit` command.
  - We should only show the visual update indicator for `--agents` repos when
    other machines have made updates to one of our enabled sase projects.
  - When we update sase (e.g. via the `,U` keymap), we should only sync in
    changes that have already been detected by a previous periodic check (so
    updates are not slowed down by needing to fetch from GitHub for every
    enabled sase project).
  - We should compensate for this (lack of ability to sync our `--agents` repos
    when we want to) by adding a new `a` (sync agents) keymap to the "Updates"
    tab of the "SASE Admin Center" panel, that just updates the current
    machine's agent state appropriately (by syncing all `--agents` repos
    corresponding with enabled sase projects on that machine and then updating
    that machine's state on disk accordingly).
- IMPORTANT: Two agents before you were tasked with this same prompt (except for
  this bullet) and produced the
  ~/.sase/plans/202607/agents_sidecar_identity_and_families.md and
  ~/.sase/plans/202607/agents_sidecar_identity_and_publish.md epic plan files.
  Review these for inspiration and make sure that the plan you produce is at
  least as good as the best of these two, with fewer bugs (dig for bugs in any
  parts of either of these plans that you integrate into your plan) than either.
  Do NOT reference either of these plan files from your new plan (i.e. this plan
  file should be standalone).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
