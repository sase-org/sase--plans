---
plan: 202607/toolong_extraction.md
---
 Can you help me factor the pylimit script out of my chezmoi repo (leave the old copy behind) into a new, dedicated bbugyi200/toolong Python repo (you will need to create this repo with the gh command before proposing your plan file)?

- Make sure the script has perfect parity with the existing pylimit script, but this script should be language-agnostic (I should be able to run it on a rust project for example).
- Make sure this repo has an excellent README and an automated release process powered by release-. See how this repo (sase) controls its release process for inspiration.
- Also make sure this new repo has great GitHub Actions, CI tests, linting, and formatting. See how this repo (sase) handles all of that for inspiration.
- Migrate this repo (sase) over to using the first published version of this new PyPI package instead of pylimit.

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.

 