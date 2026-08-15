---
tier: tale
title: Unify variable snapshots under sase var get
goal:
  Replace sase var show with unambiguous snapshot modes in sase var get and give agents
  a concise, complete get/list/set skill reference.
size: medium
proposed_by: bbugyi200.athena.sase-mg.land.w1
create_time: 2026-08-15 19:03:37
status: wip
---

- **PROMPT:**
  [prompts/202608/unify_var_get.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/unify_var_get.md)

# Plan: Unify variable snapshots under `sase var get`

## Context and scope

The existing `sase var get` selector language treats a bare token such as `status` as an
unscoped variable key, while `sase var show` reads either the current agent's live
snapshot or the newest visible snapshot for an exact historical agent name. Consolidate
those behaviors without making agent names ambiguous with variable keys. This is a
Python CLI, documentation, skill-source, and test change in the `sase` repository; the
Rust selector/query contract already provides everything selector mode needs and must
not be broadened for this work.

The resulting public command group has only `get`, `list`, and `set`. Bare `sase var`
continues to delegate to `sase var list` through the central default-list convention.

## Behavioral contract

Give `sase var get` three explicit modes:

1. With no positional target, read `SASE_ARTIFACTS_DIR` directly and render the current
   agent's canonical snapshot. This must retain `show`'s stale-index-safe behavior,
   empty-success output, compact sorted JSON, pretty block rendering, and clear missing
   `SASE_ARTIFACTS_DIR` error.
2. With exactly one fully wrapped `<agent_name>` target, strip only the wrapper and
   render the newest exact-name historical snapshot. Document shell-safe examples with
   quotes, such as `sase var get '<build>' --format json`. Preserve exact names,
   including dotted, hyphenated, and digit-leading names; project filters narrow the
   candidate history, `--hidden` controls hidden-agent visibility, an agent with no
   variables is an empty success, and an unknown name is a clear error.
3. With one or more ordinary targets, preserve the existing selector grammar and all
   existing exact/global/hood/key-wildcard/JSON-path semantics and output envelopes.
   `sase var get build.*` remains selector mode and is distinct from snapshot-mode
   `sase var get '<build>'`.

Treat wrapped-agent mode as a distinct invocation shape: reject empty or malformed
wrappers, multiple wrapped agents, and mixtures of a wrapped agent with selectors as
usage errors before querying history. Snapshot mode supports the former `show` formats
(`pretty` and `json`) and color behavior; reject selector-only `raw`/`jsonl` formats and
an explicitly supplied selector match limit with an actionable hint to use a selector
(for example `build.*`) when per-value output is wanted. Keep selector mode's current
exit codes and query diagnostics intact. Make help text describe all three modes, option
applicability, quoting, and examples while keeping subcommands and options alphabetized.

## Implementation

- In `src/sase/main/parser_var.py`, remove the `show` parser and teach the `get` parser
  to accept zero or more typed targets. Separate wrapped-agent targets from parsed Rust
  selectors without weakening argparse's current invalid-selector diagnostics. Retain
  the selector limit default in selector execution while representing whether the user
  explicitly supplied `--limit`, so snapshot-mode option validation is deterministic.
- In `src/sase/main/var_handler.py`, fold the current and named snapshot branches into
  the `get` dispatch before the existing selector-query branch. Share one snapshot
  renderer/read path, preserve direct current-artifact reads, validate mode-specific
  combinations, and remove the `show` dispatch and `show`-specific error wording.
- Adapt `src/sase/main/var_cli.py` only as needed so named snapshot resolution accepts
  `get`'s repeatable project filters and hidden visibility while still selecting the
  newest exact-name artifact deterministically. Update `src/sase/main/var_render.py`
  module/API documentation to describe `get`, `list`, and snapshot rendering; do not
  change selector JSON/JSONL schemas.
- Delete obsolete `show`-only parser/handler code and migrate its focused tests rather
  than leaving a hidden compatibility subcommand. Ensure `sase var show` is rejected by
  the public parser after the migration.

## Skill and documentation

Rewrite the canonical generated-skill source at `src/sase/xprompts/skills/sase_var.md`
as a compact, command-oriented reference:

- `get`: show the current snapshot, show a quoted `<agent_name>` snapshot, retrieve
  selectors, and choose `pretty`, `raw`, `json`, or `jsonl` appropriately.
- `list`: discover historical keys and typed values with the most useful agent, key,
  project, date, value, visibility, ordering, and limit filters.
- `set`: cover assignment, literal text, file/stdin, and `--json` forms, plus the fact
  that writes merge and later values replace earlier ones.
- Keep only the cross-agent facts agents need to act correctly: stable producer names,
  `%wait`, the `agents[...]` Jinja namespace, structured values, small-value limits,
  artifact-path guidance for reports, persistence/secret visibility, and concise `STOP`
  semantics for repeat chains. Remove duplicated exposition and every `show` example.

Update `docs/configuration.md`, `docs/cli.md`, and the cross-agent variable section in
`docs/xprompt.md` to expose only `get`, `list`, and `set`, document the
snapshot/selector distinction and option applicability, and replace all current
`sase var show` examples. Update the landing contract so these docs cannot drift back to
a four-subcommand model. Do not edit generated provider `SKILL.md` files and do not
deploy skills from the working tree; use `sase skill init --diff` only as the pre-merge
rendering preview.

## Tests and verification

- Move the scenarios in `tests/main/test_var_show.py` and the snapshot assertions in
  `tests/main/test_var_handler.py` into `get` coverage, then remove the obsolete test
  module. Cover live current reads, newest named selection, repeatable project filters,
  hidden visibility, empty snapshots, unknown agents, missing current context, pretty
  and JSON output, and color behavior.
- Extend `tests/main/test_var_parser.py` for zero-target, wrapped-agent, and selector
  parsing; malformed/mixed targets; removal of `show`; three-command alphabetical help;
  and concise mode-aware `get --help` examples.
- Preserve all existing selector tests in `tests/main/test_var_get.py` and add failures
  for snapshot-only/selector-only option combinations so the two output contracts cannot
  silently blur together.
- Update `tests/main/test_var_integration.py` to exercise `set` -> current snapshot via
  `get` -> historical `list` -> selector `get`, and update
  `tests/main/test_init_skills_sources.py` plus
  `tests/test_powerful_variables_landing.py` to assert the concise skill and the new
  documented command inventory.
- Run `just install`, the focused variable/parser/skill/landing tests, and
  `sase skill init --diff`. Search tracked source, tests, and current docs for obsolete
  `sase var show` references, then run `just check`. Escalate to the monitored
  `just check-full` lane only if scoped selection broadens or reports an unusual
  selection, as required by the repository verification policy.
