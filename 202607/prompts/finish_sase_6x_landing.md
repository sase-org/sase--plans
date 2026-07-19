---
plan: 202607/finish_sase_6x_landing.md
---






You are the land agent for epic bead sase-6x: verify the epic is truly complete, integrate it with changes
that landed since it started, then close it out.

1. Verify. Run `sase bead show sase-6x` (children, linked plan file), then `sase bead show` on each child
   bead. Confirm every bead note was addressed, and read the actual source code and the epic's commits (bead IDs
   appear in commit messages) to confirm the work previous agents reported complete really is.

2. Integrate. Changes committed since this epic started could not integrate with this epic's feature while it was
   incomplete. Find them (e.g. `git log` since the first commit mentioning sase-6x, excluding the epic's own
   commits; in a PR workflow also review commits on the base branch) and update anything that should now use what
   this epic added or that duplicates or conflicts with it. This integration is part of the epic's work.

3. Land. Close the epic with `sase bead close sase-6x`. AFTER closing, run `just symvision` if available
   (epic-symbol whitelist entries for sase-6x expire at close) and remove the stale entries and unused code
   it reports. Finally, set `status: done` in the frontmatter of the epic's plan file (the PLAN path shown by
   `sase bead show`).

If steps 1-2 uncover remaining work, use your /sase_plan skill to plan it and complete the skill's tier-aware
validate/revalidate/propose loop. Make step 3 the plan's final phase (close, run symvision, mark the plan file done)
so the agent that executes the plan finishes the landing. Otherwise do step 3 now.

%xprompts_enabled:false
### Questions and Answers

#### Q1: Visual gate

> How should I handle the unrelated ACE visual snapshot failures that prevent the approved plan’s just-check gate from passing?

- [x] **Keep epic open (Recommended)** — Preserve the strict gate and resume finalization only after the separate snapshot/rendering defects are fixed.
- [ ] **Waive visual gate** — Authorize closing sase-6x based on the passing lint, validation, generated-skill checks, and relevant feature evidence while documenting the pre-existing failures.
- [ ] **Update goldens here** — Expand this task to regenerate and review the unrelated clan/family panel and renderer-edge snapshot changes before finalizing.

%xprompts_enabled:true
