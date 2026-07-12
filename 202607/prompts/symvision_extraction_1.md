---
plan: 202607/symvision_extraction_1.md
---
 Can you help me migrate the pyvision script that is defined in my chezmoi repo to a new bbugyi200/symvision repo that publishes a new symvision PyPI package using release-please? See the sase-5r epic bead and related work for context. Make sure that this new repo has just as good of documentation, linting, and testing as the toobig repo (renamed from toolong), if not better. Keep in mind that you'll need to create this GitHub repo yourself, which should be public. You'll also need to initialize a git directory in the same parent directory as the other repo and link that directory with our new GitHub repo. 

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

  