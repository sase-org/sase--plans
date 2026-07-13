---
plan: 202607/finish_sase_5v_epic.md
---






Can you help me verify that all the work associated with the bead with ID sase-5v is complete?

Actually read through the source code and the git commits that are associated with that bead's work (they should have
the bead ID in the commit message) and ensure all of the work that the previous agents say is complete, is actually
complete. Also, run `sase bead show` on every child bead and ensure that any notes on those beads have been
addressed.

If not, plan out the remaining work using your /sase_plan skill (make sure to include closing the bead as the
final step of the plan) and complete it. Otherwise, close the bead using the `sase bead close` command. If
available, run the `just pyvision` command AFTER closing the epic bead (some symbols can be ignored while an epic
is open) to make sure we didn't leave any unused code behind.

Finally, find the plan file associated with this work (which should be in the sdd/plans/ directory with
`tier: epic`, in a YYYYMM
subdirectory). If found, a `status` field should be added (or updated if it already exists) to the frontmatter of
the plan file with a value of `done`.

%xprompts_enabled:false
### Questions and Answers

#### Q1: Permission

> May I make the plan’s narrowly scoped protected-file edits to tools/AGENTS.md, tools/CLAUDE.md, tools/GEMINI.md, tools/OPENCODE.md, tools/QWEN.md, and memory/pyvision.md?

- [x] **Approve edits** — Make only the pyvendor-to-basher documentation updates specified by the approved plan.
- [ ] **Deny edits** — Leave protected files unchanged and keep the sase-5v bead open with a recorded follow-up.

%xprompts_enabled:true
