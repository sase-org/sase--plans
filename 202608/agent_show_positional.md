---
tier: tale
title: Make agent show take the agent name positionally
goal:
  The agent-show CLI uses a required positional name and the CLI memory forbids required
  options.
size: medium
proposed_by: bbugyi200.athena.01t
create_time: 2026-08-14 16:43:26
status: done
---

- **PROMPT:**
  [prompts/202608/agent_show_positional.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_show_positional.md)

# Plan: Make `sase agent show` take the agent name positionally

## Goal

Change the public invocation from `sase agent show -n NAME` / `--name NAME` to
`sase agent show NAME`, and record the general CLI design rule that required inputs
belong in positional arguments rather than required options. Keep the parser destination
as `args.name` so the existing show handler and its rendering behavior remain unchanged.

## Scope and decisions

- Remove `-n` / `--name` from `sase agent show`; do not retain a compatibility alias.
  The accepted command contract is the required `NAME` positional, and omitting it or
  using the old option syntax is a normal argparse usage error.
- Limit the command migration to `sase agent show`. Other commands that currently have
  required options are not silently redesigned as part of this focused request.
- Synchronize every in-repo public example and reference discovered for this command,
  including generated-skill source guidance. Do not edit deployed/generated skill files
  directly.
- The user explicitly authorized the canonical `sase/memory/cli_rules.md` edit in this
  conversation, so regenerate the managed memory outputs afterward without asking for
  separate permission or creating a commit.

## Implementation

1. In `src/sase/main/parser_agent.py`, replace the required `-n` / `--name` option on
   the `show` subparser with a required `name` positional argument, update the adjacent
   invocation comment, and make the positional help text clear. Preserve the `name`
   destination expected by `src/sase/agents/cli_show.py`; no handler change should be
   necessary.
2. Add parser-contract coverage in `tests/main/test_parser_command_help.py` (or the
   nearest focused agent-parser test module): assert that `sase agent show brisk-otter`
   parses to `args.name == "brisk-otter"`, help renders `NAME` as a positional without
   `-n` / `--name`, a missing name exits with a usage error, and the removed option form
   is rejected. Keep handler-rendering tests intact because the namespace field is
   unchanged.
3. Update public command documentation in `docs/configuration.md` so the `agent show`
   row advertises `<name>` instead of `-n/--name`. Search the repository again for the
   old `sase agent show -n` / `--name` forms and update any other user-facing
   occurrences while avoiding unrelated commands such as `agent kill` and `agent tribe`.
4. Update all `sase agent show` examples in
   `src/sase/xprompts/skills/sase_agents_status.md` to use the positional form. Extend
   the `sase_agents_status` expectations in `tests/main/test_init_skills_sources.py`
   with the new invocation so the source and every provider-rendered skill remain
   contract-tested. Use `sase skill init --diff` only as a read-only preview; do not
   deploy skill output from an uncommitted tree.
5. Edit the canonical `sase/memory/cli_rules.md` through the authorized project-memory
   workflow. Add concise guidance alongside the existing CLI rules: options must not be
   required; values required for command execution should be positional arguments, while
   options represent optional controls or modifiers. Run `sase memory init --no-commit`
   to regenerate `AGENTS.md`, provider instruction shims, and the memory README, then
   inspect the diff and retain only the expected canonical and generated memory changes.
   Run `sase memory init --check` afterward to confirm the regenerated state has no
   drift.

## Validation

1. Run `just install` before repository checks so the workspace environment matches the
   current checkout.
2. Run the focused parser and generated-skill tests, for example
   `just test tests/main/test_parser_command_help.py tests/main/test_init_skills_sources.py`.
3. Inspect `sase agent show --help` to confirm the usage is `sase agent show [-h] NAME`,
   the positional is documented clearly, and the removed option is absent. Exercise
   parser failure coverage for both a missing name and the old `--name` form through the
   automated tests rather than depending only on help text.
4. Re-run repository searches for stale old-form examples and inspect the memory and
   skill-generation previews for unintended output.
5. Run the required `just check` whole-repo lint and diff-scoped test gate. If it
   escalates or reports unusual test selection, follow project guidance and run
   `just check-full` through `/sase_monitor` with a follow-up action.
