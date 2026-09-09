---
tier: tale
title: Remove the redundant toobig queue priority
goal:
  Toobig split agents rely on the configured three-runner admission threshold without
  authoring an explicit priority.
size: small
proposed_by: bbugyi200.athena.0hf
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.sase-xc](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-xc/README.md)
  - [bbugyi200.athena.sase-yz.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-yz.1/README.md)
- **COMMITS:**
  - [e89b9e5](https://github.com/sase-org/sase/commit/e89b9e522a4f89cbe68734861c65b3ba99a0a6c2)
    — fix(test-cost): recalibrate hard CPU budgets
  - [7a4fb21](https://github.com/sase-org/sase/commit/7a4fb2149a1d308299d968f0f3d7c477c72dbaec)
    — feat(usage): accept vendor drift reason

# Plan: Remove the redundant toobig queue priority

## Context and scope

Implement this focused change in the external `bbugyi200/bugyi-chops` repository. The
`toobig_split` chop currently appends `%queue(priority=20)` to every condition-gated
split-file prompt. The Athena `run_every` lumberjack already supplies `wait_runners: 3`,
and SASE scaffolds that setting as `%queue(runners=3)` when it plans the chop proposals.
The explicit priority therefore adds independent queue ranking and bounded deference
that are no longer wanted.

Do not change the SASE queue implementation or the chezmoi-managed lumberjack
configuration: their existing runner-threshold behavior is the mechanism this change
intends to retain. Removing the priority means these agents use the normal default
priority, including on the uncommon machines whose runner cap is below three.

## Implementation

1. In `src/bugyi_chops/toobig_split.py`, remove the now-unused launch-priority constant
   and have `_admission_prompt()` end with `%auto #split_file:<path>`. Preserve the
   `%if` fence, structured `@medium` model, clan metadata, and sequential `wait_on`
   chain exactly as they are.
2. In `tests/test_toobig_split.py`, update the raw admission-prompt contract to require
   `%auto #split_file:<path>` with no authored `%queue` priority. Assert through parsed
   directives that the raw chop prompt has neither an explicit priority nor an explicit
   runner threshold.
3. Strengthen the SASE planning integration coverage by preparing representative toobig
   proposals with `lumberjack_wait_runners=3`. Verify every planned prompt gains the
   host-owned `%queue(runners=3)` threshold, retains the condition/clan/model/wait
   behavior, and has no explicit priority. Keep the surviving-tail promotion coverage
   exercising the same prepared proposals so conditional filtering cannot reintroduce or
   drop the threshold.

## Verification

1. Run the focused toobig test module (`pytest tests/test_toobig_split.py`) to exercise
   raw proposal construction, SASE scaffolding, typed conditional admission, clan
   sequencing, and eligible/skipped launch paths.
2. Run the repository's full `just check` gate (using `just install` first if the
   external checkout needs its development environment). This covers Ruff formatting and
   linting, strict mypy, the complete pytest suite with branch coverage, package builds,
   and Twine artifact validation.
3. Inspect the final diff and search the `bugyi-chops` tree for `LAUNCH_PRIORITY`,
   `%queue(priority=20)`, and equivalent `priority = 20` spellings. The toobig launch
   path and its expectations must contain none, while unrelated queue-priority examples
   outside this external plugin remain untouched.

## Acceptance criteria

- Raw `toobig_split` proposals no longer author a `%queue` directive or priority.
- Under the configured three-runner lumberjack default, planned launch prompts contain
  `%queue(runners=3)` and parse with no explicit priority.
- Conditional admission, structured model selection, sequential clan execution, and
  stale-file skipping behave exactly as before.
- Focused tests and the full `bugyi-chops` verification gate pass.
