---
plan: 202607/retire_legacy_plan_layout.md
---
 #fork:5v Can you now help me move the final few tales/ markdown files in the .sase/sdd/ directory (the sdd companion repo) and then remove all the migration logic we added to the `sase sdd init` command and any backward compatibility code we have related to this (all projects on all machines have been updated)? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.


%xprompts_enabled:false
### Questions and Answers

#### Q1: SDD cleanup

> Should I expand the scope to deduplicate or relink the 1,998 pre-existing collision-suffixed companion plans so `sase sdd validate` and `just check` can pass?

- [x] **Repair companion corpus** — Analyze and safely deduplicate/relink the historical `_1` plan collisions, then rerun all verification.
- [ ] **Leave corpus unchanged** — Keep the approved implementation as-is and hand off with the known validation failure documented.

%xprompts_enabled:true
