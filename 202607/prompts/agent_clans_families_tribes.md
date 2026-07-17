---
plan: 202607/agent_clans_families_tribes.md
---
 We recently added support for parallel agent families and migrated epic
phase agents and the epic lander agent over to use a single parallel agent
family. We did the same for a `research_swarm` xprompt that's defined in my
chezmoi repo. This recent work is mostly going in the right direction but
there's a terminology conflict with agent families that I think has caused a few
significant bugs / undesired functionality. Can you help me clean this up
significantly by separating the concepts of parallel agent families (e.g. used
by epics--will be called clans in the future) and sequential agent families
(e.g. used when an agent proposes a plan that gets approved)?

- See the "Glossary" section below for context on tribes/clans/families. You
  should add a better set of glossary definitions to the memory/glossary.md file
  (using the definition-list format that is already used in that file). Keep
  this concise but make sure these definitions are useful and accurate. Remember
  that every token in context either helps or hurts us.
- Let's start setting `%tribe:epic` in for all epic clans (i.e. add all epic
  clans to the `@epic` tribe).
- We need to start supporting the expandion of agents / agent families in agent
  clans using the `l` keymap (used twice to see hidden steps--the `h` keymap can
  be used to collapse). This means there will be a third level of indentation
  that we need to design.
- Speaking of bugs, the Agent Clan runtime that's shown to the far right of the
  left pane on the Agents tab is incorrect because it does not take into
  consideration which agents are/were running at the same time. That runtime
  should reflect the total wall-clock time it took for all of the agents in the
  clan to complete, not including any time spent waiting for a human (e.g. to
  approve a plan or answer a question).
- Make sure we show an excellent and complete aggregate view (in the agent
  metadata panel) of the agents / families that belong to a clan when that clan
  is selected on the agents tab.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 

### Glossary

**Agent Clan**

- This concept is what will be used to replace parallel agent families but some
  bugs will need to be fixed / modifications will need to be made.
- An agent clan, for example, MUST have a unique name that cannot be used by an
  existing, previously completed, or future sase agent. This contradicts how we
  do things today, where we select one root agent that represents the parallel
  agent family (make sure to clarify / remove these misleading "root" / "child"
  memory/glossary.md entries.
- Migrate the epic lander agent to use the new `<epic_bead_id>.land` naming
  scheme to support this requirement (it should be easy for you to figure out
  how to migrate the `research_swarm` xprompt on your own).
- We should migrate the `%f|%family` directive to `%c|%clan` to support this
  change.
- Clans define a unique agent hood that all of the clan's families are members
  of. We should enforce this and provide a good error message notification when
  this constraint is violated (e.g. `foo.bar` can be in the `foo` clan, but
  `baz.bar` cannot).

**Agent Family**

- An agent family is created when a stopped/complete agent's name is targeted
  (via the `%n|%name` directive when two input arguments are provided) by a
  newly launched sase agent. At this point, what used to be an "agent name"
  becomes an "agent family name" and the original sase agent is renamed used a
  `--<role>` suffix, which all family members must have.
- IMPORTANT: A consequence of the above bullet is that there is no such thing as
  an agent family with one member.
- Agent family members must be launched sequentially (i.e. you can never have
  parallel agents running in an agent family).

**Agent Tribe**

- This should replace the concept of "Agent Groups/Tags" ("tags" is a legacy
  term, but is still used), but will be used in much the same way.
- We should migrate the `%g|%group` directive to `%t|%tribe` to support this
  change.
- We should start using the `@` prefix instead of `#` when showing tribes in the
  TUI (ex: show `@chop`, not `#chop`).