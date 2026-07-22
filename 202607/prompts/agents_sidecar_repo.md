- **PLAN:** [../202607/agents_sidecar_repo.md](../agents_sidecar_repo.md)

 I want to add support for a new, hidden (i.e. not included in sase
memory files or auto-cloned to the project repo's sase/repos/ directory) sidecar
repo named `--agents`. Can you help me design and implement this?

- The point of this sidecar repo will be to publicly store and host all of the
  data necessary to reconstruct the sase agents that we store in that repo. We
  should store all sase agents that created commits for the given sase-managed
  (see the `is_sase_managed` config field for context) project.
- We should start updating this repo by pulling in any changes that may have
  been pushed by other machines every time we do a full sase update (e.g. every
  time we use the `,U` keymap). We currently periodically check for sase
  updates, sase plugin updates, and agent CLI updates. We should start
  periodically checking if this repo needs to be synced as well and use a
  distinct visual indicator in the top right-hand corner to relay this to the
  user. Make sure this check, which should be performed for every enabled sase
  project, does not hurt performance.
- The source of truth for sase agents should still live locally in the same
  locations. When we sync this new agents sidecar repo (e.g. run `git pull` on
  it to bring in external changes), we should also integrate any new agents that
  are defined in that repo into our local representation on disk.
- This means that anyone who works on a sase-managed repo using sase should have
  access to a set of the same shared historical agent chats (the ones we commit
  to this repo at least; any sase agent run that is not associated with a commit
  for a given project should not have its data committed to this repo).
- Make sure that the `sase repo init` command automatically initializes this
  repo after confirming with the user. Make sure that we are very clear with the
  user that all of their sase agent chats that are associated with commits will
  be pushed to this repo.
- We should add support for a project local sase.yml configuration field that
  can completely disable this repo or configure it to be private. In which case
  the associated GitHub repo will still be created and still be committed to but
  it will be a private repo instead of public.
- Machine Agent Hoods
  - We will need to add a new sase agent name syntax for referencing sase agents
    that ran on a different machine than the current one. This way sase agent
    names are still always unique.
  - Let's solve this by requiring a distinct, top-level agent hood be associated
    with every agent.
  - This hood should be configured on every machine via a new required
    `machine_name` config field.
  - A new `sase config init` command, which the `sase init` command should wrap
    like it does with other `init` commands, should prompt the user for a unique
    (all lowercase letters and/or underscores) machine name if this field is not
    already configured. When the user submits the machine name, it should be
    written to a new (if it doesn't already exist; otherwise, make sure we make
    the minimal required file change) `sase_<machine_name>.yml` file in the
    appropriate location (e.g. depending on if the `use_chezmoi` config field is
    set or not).
  - IMPORTANT: Machine agent hoods MUST be stripped from any sase prompt on the
    machine associated with that hood, but they MUST be included when written to
    disk (ex: an agent named `foo` on this machine should become
    `athena.foo`--once I configure `machine_name`--on this machine when written
    to disk, but should always be converted back to `foo` when displayed to the
    user on this machine).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 