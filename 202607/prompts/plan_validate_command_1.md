---
plan: 202607/plan_validate_command_1.md
---
 

Investigate the current SASE plan formats, proposal flow, epic approval flow, bead creation, hooks, and CI validation, then write and propose a concrete implementation plan for an agent-facing `sase plan validate` command. Do not implement the feature yet.

Treat these as settled requirements:

- The command validates exactly one explicit `PLAN_FILE` argument. Do not infer a plan from agent context and do not add bulk validation in v1.
- The calling agent must explicitly provide the plan tier, and the only supported tiers are `tale` and `epic`. Choose the idiomatic required CLI argument or option spelling after auditing the existing CLI conventions.
- V1 performs deterministic schema validation only: syntax, frontmatter, and tier-specific structural fields. It must not judge prose quality, feasibility, testing quality, risks, or completeness.
- Keep this command independent in purpose and UX from `sase validate` and `sase sdd validate`; do not extend either existing command as the user-facing surface. This validator exists for agents to run before proposing plans.
- The intended agent loop is: run the command with a plan file and tier; when the file is incomplete or invalid, receive the expected tier-specific frontmatter schema plus actionable diagnostics; edit the plan to conform; rerun the same command until it passes; only then propose the plan.
- Every supported plan type must require a top-level `goal` frontmatter property describing the outcome the plan is designed to achieve.
- An epic plan must declare all structured data needed to programmatically create the parent epic bead and every ordered phase bead, including enough dependency information to wire the phase graph correctly. Keep these as schema requirements rather than subjective plan-quality checks.
- Report all discovered validation problems in one run. Each diagnostic must be actionable and identify the file and location when possible. Any validation failure returns a nonzero exit status.
- Enforce successful validation automatically at all three boundaries: before a plan enters the proposal approval queue; during epic approval before notifications, bead creation, or PR setup; and in the appropriate CI or hook checks for committed plan files.

Audit the repository before settling the design. The proposed epic plan should identify the authoritative schema representation, command/API boundaries, integration points, agent-prompt or workflow updates needed to teach agents the tier-aware validate/edit/revalidate loop, migration or compatibility concerns for existing plan files, diagnostic behavior, and focused plus end-to-end tests. Respect the repository's Python/Rust backend boundary and existing CLI rules. Resolve design details from the codebase while preserving every requirement above.

This is a large piece of work that should be split into phases. I'll let you decide how many phases to create, but
keep in mind that each phase will be completed by a distinct agent instance (i.e. a distinct `claude` / `agy` /
`codex` / `qwen` / `opencode` command). Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.



### Additional Requirements

- The goal is for epic plan files to have enough information in frontmatter properties for us to create the epic beads, the phase beads, and all the dependencies and work the epic by running the sase bead work command automatically when the user approves an epic. In other words we don't need the bd/new_epic xprompt anymore. Also each phase entry in the frontmatter should specify an optional model field so agents creating epics can specify which models its phase agents use; however, make sure that the description of this field specifies that an agent should only set this field explicitly when the user's prompt requested it or if some of the phase agents are meant to test the feature's functionality itself (in other words, some of the phase agents do not do real consequential work).