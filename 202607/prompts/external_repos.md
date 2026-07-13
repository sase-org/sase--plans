---
plan: 202607/external_repos.md
---
  Can you help me start allowing agents to use the `sase repo open` command for any sase repo, regardless of
whether it even exists yet?

- We should make some changes to the types of repos sase handles to support this feature. Namely, we should also add a new repo type called external repos. These
    are repos that the agent didn't know about before the request. At the moment external repos are always GitHub repos, but keep in mind that we should try to be VCS agnostic about this so it is easy to add support to, for example, GitLab later.
- Keep in mind that the `sase repo open` command probably doesn't support external repos yet. You will need to add
  support for them. External repos should be cloned to the new workspace-local sase/repos/external/ directory.
- We should create two new xprompt skills to support this feature:
  - `/sase_repo`: This skill will allow agents to work with the `sase repo` command. We should start using this in agent
    instructions instead of telling agents to run the `sase repo open` command directly.
  - `/sase_project`: This skill is not strictly necessary but will teach the agent how to use the `sase project`
    command, which will be useful, for example, when I want one agent to launch one other sase agent for every currently enabled sase
    project.
- We should also make sure that both of these skills have great descriptions and that the `/sase_repo` skill's
  description and the agent instructions we generate in agent instruction files (ex: AGENTS.md) make it clear that
  any time the agent needs to read/modify files in a different sase project (a project that the user told it about in the
  prompt, for example) or a GitHub project that is NOT associated with the current project via a linked repo, they MUST
  use the `/sase_repo` skill to open that repo as an external repo.
- Make sure external repos have all of the same commit finalizer support and diff support / file delta support / support
  for anything else that needs to be done to give this feature full support in the `sase ace` TUI. I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!
- IMPORTANT: When adding/modifying agent skill descriptions or sase memory files (ex: memory/sase.md), try to keep these files concise but useful. Keep in mind that
  every token in context either helps us or hurts us.

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

 