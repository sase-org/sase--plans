---
plan: 202607/wait_priority_directive.md
---
 Can you help me add a new priority keyword argument to the wait xprompt directive?

- This argument should have an integer value that defaults to 10.
- This argument will be used to control which agents are selected from the max runners (defaults to 10 but it is configurable) queue, if there are agents waiting because there are too many agents running right now and launching would therefore violate the max runner's limit.
- Agents that were launched with a lower priority will be selected and run from the queue first, regardless of when their prompts were submitted.
- If all agents in the queue have the same priority, we should select the agent that's been waiting the longest, as we do now.
- As our first use case let's start setting the priority to 20 for all of the agents that the toobig_split chop (defined in my bugyi-chops Github repo) launches.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 