---
tier: tale
title: Fix CI model-alias availability isolation
goal:
  Make the epic work-rendering alias test deterministic across CI and developer hosts
  without changing production availability-aware selection.
size: small
proposed_by: bbugyi200.athena.y3
create_time: 2026-08-11 14:45:22
status: wip
---

# Fix CI model-alias availability isolation

## Context

GitHub Actions CI run 31517337366 for `master` fails only
`tests/test_bead/test_work_rendering_models.py::TestModelDirective::test_no_model_only_adds_role_alias_directives_over_baseline`
on Python 3.14 after 28,905 tests pass. The test expects `@epic_lander` to resolve
through `@default` and `@smarter` to frozen pool member 0, `codex/gpt-5.6-sol@high`, but
CI resolves frozen pool member 1, `claude/opus@high`.

The production resolver intentionally filters load-balanced alias pools by provider
availability even when `consume=False`. The test patches the configured provider to
Claude but does not pin target availability. The suite's CI-isolation fixture supplies a
Claude CLI stub, so Codex is unavailable and the resolver correctly skips member 0.
Other tests that assert a specific frozen pool member explicitly patch
`_resolved_target_is_available` to return true. The failure is therefore missing test
isolation, not incorrect production selection behavior.

The preceding run, 31516685140, also contained external-mirror import and type failures;
current `master` commit 48617f3ee removes that abandoned implementation, and the current
run confirms its lint, performance, and visual lanes pass. The remaining actionable
failure is this model-alias assertion.

## Implementation

1. In the failing work-rendering test, patch
   `sase.llm_provider.config._resolved_target_is_available` to return true before
   resolving the role aliases. Keep the existing frozen member-0 assertions so the test
   deterministically verifies the intended alias graph and effort propagation
   independent of CLIs installed on the host.
2. Do not change production alias resolution or weaken availability-aware pool
   selection; that behavior is covered separately by load-balancing tests.

## Validation

1. Run the failing test in isolation with a PATH that does not provide a Codex CLI,
   proving it no longer depends on ambient provider executables.
2. Run the complete `tests/test_bead/test_work_rendering_models.py` module to catch
   interactions with the neighboring rendering cases.
3. Run `just install`, then the repository-required `just check`. If scoped selection
   escalates or reports an unusual selection, run `just check-full`.
4. Re-run `actstat` and inspect the newest GitHub Actions state, clearly distinguishing
   local validation from any run that still targets a pre-fix commit.

## Acceptance criteria

- The failing alias assertion passes without relying on an installed Codex CLI.
- Existing provider-availability and load-balancing semantics remain unchanged.
- The work-rendering test module and `just check` pass.
- The final report identifies any GitHub Actions status that cannot yet include the
  local, uncommitted fix.
