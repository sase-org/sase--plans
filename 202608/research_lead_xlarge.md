---
tier: tale
title: Default the research lead to @xlarge
goal:
  The packaged research_lead alias uses the valid @xlarge built-in model alias by
  default.
size: small
proposed_by: bbugyi200.athena.0a7
create_time: 2026-08-21 16:09:46
status: wip
---

# Default the research lead to `@xlarge`

## Goal

Restore the packaged `@research_lead` model alias to a valid built-in default by
replacing the removed `@smartest` alias with `@xlarge`, while preserving the alias's
description, bucket membership, and normal user/project override behavior.

## Context

The `sase-research-artifacts` plugin contributes its model aliases through
`src/sase_research_artifacts/default_config.yml`. The `research_lead` entry currently
points at the removed `@smartest` alias. Its exact packaged value is covered by
`tests/test_default_config.py`; the xprompt itself already launches through
`@research_lead` and therefore does not need to change.

## Implementation

1. In `src/sase_research_artifacts/default_config.yml`, change only the
   `research_lead.model` default from `@smartest` to the built-in `@xlarge` alias. Keep
   the `research_lead` name, description, and `researchers` bucket unchanged.
2. Update the corresponding exact-value assertion in `tests/test_default_config.py` to
   expect `@xlarge`. Retain the existing bucket and schema-validation coverage so the
   test continues to exercise the config through plugin discovery rather than by parsing
   the YAML independently.
3. Confirm the obsolete `@smartest` spelling no longer occurs in the repository. No
   README or xprompt documentation changes are needed because they refer to the stable
   `@research_lead` alias rather than its overridable default value.

## Verification

1. Run `just test tests/test_default_config.py` to verify the packaged configuration
   loads with the new exact model value and remains schema-valid.
2. Run `just check` from the `sase-research-artifacts` repository to satisfy its full
   lint and non-wheel test gate.

## Acceptance Criteria

- A fresh install's packaged `research_lead` alias resolves by default through
  `@xlarge`, with its existing description and `researchers` bucket intact.
- No repository source or test expectation still references the removed `@smartest`
  alias.
- The focused default-config tests and the repository's `just check` gate pass.
