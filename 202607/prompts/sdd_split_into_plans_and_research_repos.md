---
plan: 202607/sdd_split_into_plans_and_research_repos.md
---
 #fork:6a Can you now help me migrate the sdd repo to two different linked repos (sase--plans and sase--research)? This will require us to improve and generalize the concept of a linked repo a little bit.

- We should add a new linked_repos.<repo>.auto_clone config field that specifies whether or not we should automatically clone a linked repo while preparing the workspace directory before launching the agent. This field should default to false but we should set it to true for the new sase--plans linked repo.
- Agents that set the previously mentioned config field should not have their descriptions added to any agent files (ex: AGENTS.md).
- The new plans linked repo will contain the flattened contents of the current plans/ directory and will also contain the beads/ directory.
- The new research linked repo will contain a flattened version of our research directory. Make sure that you update all of our current research xprompts accordingly, which I believe are defined in the chezmoi repo.
- One or more of the phase agents you assign should use GPT image to generate great infographics for these repos. These infographics should be included in the readme files that are generated when we generate the GitHub repos, which is something that the sase init commands should do.
- Both of these linked repos should be added by default for any repo that is sase managed, which is something that is controlled by a project local configuration field.
- This might be a large, tricky migration so make sure you think this through thoroughly.

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

 

### Additional Requirements

- See the previous agent's attempt for context on the following requirements: 1) The repos that get created by the sase init commands should be public, not private. 2) The tales/, legends/, and myths/ directories have been deleted from all projects at this point so you shouldn't need to worry about them. 3) When I said "flattened", I mean there will be no sdd/ top-level directory anymore, NOT that we should get rid of the `<YYmmdd>/` directories; these should go in the top-level companion repo directory (e.g. next to the assets/ directory and, for the `--plans` companion, the beads/ directory).