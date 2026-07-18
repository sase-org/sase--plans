---
plan: 202607/member_roster_sections.md
---
 I want to start adding a new section to the very top of the agent metadata section list for agent families and for agent clans. 

- This section should be named either FAMILY MEMBERS or CLAN MEMBERS (the presence of this section will also serve as an extra indication to the user that they are looking at an agent family/clan, so make sure you make these sections stand out and make sure they are visually distinct).
- Every family member or clan member belonging to that family/clan should be listed here with some details for each that should be foldable using the new fold keymaps.
- Each entry in this section should have a number associated with it. Ideally if there are 10 or fewer agents, we should associate a single number starting at 0 and ending at 9. If there are more numbers than that, the number should start at 00 and end at 99. No more than 100 members should be supported in this section (show how many members we truncated).
- Each of these numbers should be associated with a keymap that, if used, navigates to the corresponding Agent/Agent Family, expanding the containing clan or family if necessary.
- As a part of this change since we're already adding the first foldable section for agent families, we should add good support for each level of folding to each section shown when agent families are selected.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 