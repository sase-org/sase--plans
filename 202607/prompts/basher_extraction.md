---
plan: 202607/basher_extraction.md
---
 We recently migrated the pylimit script to a new toobig GitHub repo and we are currently in the process of  doing the same thing for the pyvision script (search for and review the related sase epic beads and the corresponding commits for these migrations for context). Can you now help me do the same thing for the pyvendor script and the bugyi.sh script?

- These should both be contained in a new Python package called basher, which I already own and actually already have another repo associated with.
- Rename that old GitHub repo to pybasher (or something else if that's taken) and create a brand new bbugyi200/basher repo in GitHub before creating the plan file (PyPI should already be registered to that repo).
- One significant improvement we're going to make here is that the bugyi.sh file will be generated using the Python package's version number to produce the file name suffix instead of the current date.
- Also, make sure the basher script is user-friendly and configurable.
- Also, add some useful CLI options.
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! 

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

 