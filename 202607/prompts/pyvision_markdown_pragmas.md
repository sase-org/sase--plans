---
plan: 202607/pyvision_markdown_pragmas.md
---
 GitHub Actions is failing for the sase repo. Can you run the `actstat` command to get more information about
the failing jobs, diagnose the root cause of these failures, and then fix them? We should have never been using `pyvision` pragmas like this anyway. These are equivalent to the agent cheating the get lens to pass. The appropriate thing to do is to remove dead code if it's not used anymore or to make public functions private if they are only referenced in the file they're defined in and possibly by tests. As a part of this fix, can you help me update the `pyvision` script in my chezmoi repo to not accept markdown files as valid file references (the point of listing a file reference in the pragmas is to list another piece of code or configuration somewhere that needs that symbol to be public). Revender the pyvision script into this repo using the pyvendor script when you're done.

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

