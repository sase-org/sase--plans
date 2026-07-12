---
create_time: 2026-07-09 10:53:30
status: done
prompt: 202607/prompts/ci_pyvision_stale_telegram_pragma.md
tier: tale
---
# CI Pyvision Stale Telegram Pragma Plan

## Diagnosis

`actstat` shows the failing `sase-org/sase` workflow is the `CI` run on `master` for commit
`848aa07fe280fccaf1dabb089305df6690b3290c`. The only failed job is `lint`, and the failed step is `Lint`.

The GitHub Actions log shows that `just lint` succeeds through keep-sorted, ruff, mypy, and script structure validation,
then fails in the pyvision unused-definition pass:

```text
Error: pyvision pragma in src/sase/integrations/agent_status_groups.py:46: external repository 'https://github.com/sase-org/sase-telegram.git' does not reference symbol 'group_agent_statuses'
```

The current `sase-telegram` code still imports `status_bucket_header` from `sase.integrations.agent_status_groups`, but
no longer imports or references `group_agent_statuses`. Its agent list renderer now groups Telegram list-entry records
locally. This means the `group_agent_statuses` external-repository pragma is stale, not a flaky Actions failure.

Local `just _lint-pyvision` can pass if pyvision resolves the URI pragma through an older local checkout/cache.
Re-running with `PYVISION_EXTERNAL_REPO_PATHS` pointed at current `sase-telegram` reproduces the CI failure, matching
GitHub Actions' fresh clone behavior.

## Proposed Fix

1. Remove the obsolete `group_agent_statuses` public API from `src/sase/integrations/agent_status_groups.py`.
   - Remove its `# pyvision: https://github.com/sase-org/sase-telegram.git` pragma.
   - Remove the now-unused `AgentStatusGroup` dataclass and `_agent_status_bucket` helper if they are only supporting
     `group_agent_statuses`.
   - Keep `agent_status_bucket_glyph` and `status_bucket_header`, because `sase-telegram` still consumes
     `status_bucket_header`.

2. Update the focused tests in `tests/test_agent_status_groups.py`.
   - Drop tests that only exercise the removed grouping API.
   - Keep or add tests for the surviving header/glyph behavior so the integration-facing formatting contract remains
     covered.

3. Verify the fix locally with the same failing path.
   - Run `PYVISION_EXTERNAL_REPO_PATHS=<current-sase-telegram-checkout> just _lint-pyvision` to confirm the stale pragma
     failure is gone against current `sase-telegram`.
   - Run `just install` if needed for this ephemeral workspace, then run `just check` as required after code changes.

## Expected Outcome

The `lint` job should stop failing because pyvision will no longer try to prove a stale external reference for
`group_agent_statuses`. The remaining external pragma for `status_bucket_header` should continue to pass because current
`sase-telegram` still imports that symbol.

## Risks

Removing `group_agent_statuses` is a public API change within the integration helper module. The available evidence
indicates it has no non-test consumers in this repository or current `sase-telegram`; if another unpublished consumer
exists, it would need to migrate to its own grouping logic or request a real supported API.
