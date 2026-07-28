---
tier: tale
title: Restore sase-telegram CI after the SASE identity-directive migration
goal: Telegram emits current SASE identity directives and its full lint and test suite
  passes.
create_time: 2026-07-25 06:46:23
status: done
---

- **PROMPT:** [202607/prompts/fix_ci_identity_directives.md](prompts/fix_ci_identity_directives.md)
- **AGENTS:**
  - [bbugyi200.athena.k5--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.k5.md#member-code)

# Restore CI after the SASE identity-directive migration

## Context and root cause

The latest `sase-org/sase-telegram` GitHub Actions run fails in the Python 3.12 `Lint` step. Ruff passes, but mypy
reports that `src/sase_telegram/scripts/sase_tg_inbound.py` passes `directive_alias="n"` to
`sase.agent.retry_prompt.rewrite_retry_prompt_name`, whose contract now accepts only `"id"` or `"i"`.

This is an integration break caused by SASE's intentional launch-directive migration from `%name`/`%n` to `%id`/`%i`.
The CI workflow checks out the current SASE source, so it detected that Telegram still generates the removed `%n`
syntax. The Python 3.13 job did not expose a separate install failure: it was canceled during dependency compilation
after the Python 3.12 matrix entry failed because the matrix uses the default fail-fast behavior.

The repository's tests also contain active launch prompts and expected Telegram retry payloads using `%n` or `%name`.
Leaving those fixtures unchanged would either exercise obsolete behavior or fail once the suite gets past linting.

## Implementation

1. Update Telegram retry-prompt generation in `src/sase_telegram/scripts/sase_tg_inbound.py` to request the supported
   short identity alias, producing `%i:<retry-name>` payloads while preserving the existing prompt-rewrite,
   name-allocation, long-prompt callback, and error fallback behavior.
2. Migrate active identity-directive prompts, assertions, pending-action payloads, and explanatory comments in
   `tests/test_inbound.py` and `tests/test_custom_gates.py` from `%n`/`%name` to `%i`/`%id`. Use the short alias where
   the test previously exercised short syntax and the canonical long spelling where it previously exercised long syntax.
3. Keep non-identity uses of the letter `n`, plain-language references to agent names, and compatibility behavior
   outside this repository unchanged. Do not weaken mypy, pin SASE to an older revision, or disable matrix fail-fast;
   the correct fix is to follow the current SASE API and launch syntax.

## Validation

1. Search production code and tests for remaining `%n`, `%name`, and `directive_alias="n"` occurrences, and review any
   survivor to ensure no active launch or retry path still emits retired syntax.
2. Install the repository's development environment with `just install`, as required for a fresh SASE workspace.
3. Run focused retry, launch, normalization, project-extraction, and custom-gate tests that cover the migrated fixtures.
4. Run `just check` to execute the complete ruff, mypy, and pytest validation required by the repository.
5. Re-run `just check` after reviewing the final diff and repository status, so the handoff records a clean second full
   validation of the exact final source state.

## Expected result

Mypy accepts Telegram's use of `rewrite_retry_prompt_name`, retry buttons and callbacks emit valid `%i` identity
directives, all tests describe the current SASE launch contract, and both Python 3.12 and Python 3.13 CI matrix entries
can complete instead of failing or being canceled.
