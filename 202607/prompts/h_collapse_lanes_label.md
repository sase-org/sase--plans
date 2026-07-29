- **PLAN:** [../202607/h_collapse_lanes_label.md](../h_collapse_lanes_label.md)

 The `H` (collapse houses) keymap seems to have the wrong description (see #sshot). Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 

%xprompts_enabled:false
### Questions and Answers

#### Q1: H label

> In the screenshot, H is labeled "collapse houses" while a family member (np--plan) is selected inside the one open np family. Internally that label is consistent with the code, help modal, and docs/ace.md — so which part reads wrong to you?

- [x] **Wrong word: houses** — "House" is only a metaphor in the glossary; the real user-facing terms are lane / hood / family / clan. Footer (and help modal + docs) should use the real term instead.
- [ ] **Wrong number: plural** — Only one house is open here, so it should read "collapse house" (singular) and only pluralize when 2+ open houses would be collapsed.
- [ ] **Wrong scope: should name the row** — With a family member selected, H should advertise what it does to that row (e.g. "collapse family"), not the group-wide sweep label.
- [ ] **Wrong behavior: H does something else** — Pressing H here actually does something other than fully collapsing open houses (e.g. collapses the group/clan, or does nothing). The label is fine; the availability logic is buggy.
- [ ] **Something else** — None of the above — I will describe what I expected H to say/do.

%xprompts_enabled:true