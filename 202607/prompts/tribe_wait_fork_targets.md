---
plan: 202607/tribe_wait_fork_targets.md
---
 Can you help me add support to the wait/fork directive/xprompt for targeting the very next agent/agent clan that is launched in a given tribe?

- You will likely need to add support for waiting for / forking agent clans, which I don't think is supported yet.
-  Make sure that we do not include the entire chat transcripts from every clan member when we fork an agent clan. We should include the prompts from every clan member but only statistics about the reply and other useful metadata. Remember that every token in context either helps or hurts us. 
- Tribe input arguments should just be the tribe name prefixed with `@`.
- For example, the @epic target would specify that an agent should wait for /fork the next Epic clan (these get created each time a user approves an epic plan file) that joins the @epic tribe.

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 