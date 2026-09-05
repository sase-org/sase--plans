---
tier: tale
title: Retry grok max_tokens_truncation CLI aborts
goal:
  A grok agent run that hits the Grok Build CLI's max_tokens_truncation internal error
  retries with the continuation nudge instead of failing the workflow.
size: small
proposed_by: bbugyi200.kellys_mbp.sase-ws.1.f1
create_time: 2026-09-05 06:06:03
status: wip
---

# Retry grok `max_tokens_truncation` CLI aborts instead of failing the run

## Goal

Classify the Grok Build CLI's `max_tokens_truncation` internal error as retryable in the
grok provider's built-in retry config, so an agent run that hits it resumes with the
standard continuation nudge instead of failing the whole workflow and holding its
workspace hostage.

## Background / Root Cause

On 2026-09-04 the `sase-ws.1` coder run (`ace(run)-260904_135714`, grok-4.6, effort
xhigh) died at turn 55 when the `grok` CLI aborted the entire session with exit code 1
and this stderr payload:

```
Error: Internal error: {
  "message": "response truncated by max_tokens",
  "error_kind": "max_tokens_truncation",
  ...
}
```

The coder had already completed most of its work (the user later hand-committed it as
`61d72860a` and closed the bead with "I had to manually commit but I think the agent
completed its work"). The failure chain:

1. `GrokProvider._invoke_loop` (`src/sase/llm_provider/grok.py`) correctly raised
   `CalledProcessError` on the nonzero exit; `invoke_agent` wrapped it as
   `LLMInvocationError`.
2. `handle_workflow_error` (`src/sase/axe/run_agent_exec_retry.py`) consulted
   `is_retryable_error` / `find_retry_config_for_error`, but grok's built-in
   `error_patterns` (`"xAI API error"`, `"xAI rate limit"`, `"xAI server error"`,
   `"xAI upstream request failed"`) match none of the error text, and no other
   provider's patterns match either. The handler returned `"raise"` and the run failed
   terminally.

This is the same failure class the claude provider already retries via its built-in
`"Prompt is too long"` pattern: a mid-session output/context truncation where the retry
engine's `preserve_workspace=True` + `_RETRY_CONTINUATION_NUDGE` design is exactly the
intended recovery (fresh session, `git status`/`git diff`, continue). The gap is purely
that grok's pattern list predates observing this error family live.

## Changes

### 1. `src/sase/llm_provider/grok.py` — extend built-in retry patterns

In `GrokProvider.llm_default_retry_config`, add two patterns to `error_patterns`:

- `"max_tokens_truncation"` — the stable machine `error_kind` key in the CLI's JSON
  error payload.
- `"response truncated by max_tokens"` — the prose `message`, kept as a second anchor in
  case a future CLI build drops the JSON structure but keeps the sentence.

Keep the existing four xAI patterns, `max_retries=3`, `wait_times=[60, 300, 1800]`,
`continuation_prompt=_RETRY_CONTINUATION_NUDGE`, and `preserve_workspace=True`
unchanged. Add a short comment citing the captured live failure (2026-09-04,
`ace(run)-260904_135714`, grok-4.6 xhigh, turn 55) the way the usage-limit config
comments in the same file cite their captured evidence.

### 2. `tests/llm_provider/test_grok_provider_core.py` — regression coverage

`test_grok_retry_config_uses_xai_specific_patterns` currently asserts
`all("xai" in pattern.lower() for pattern in config.error_patterns)`, which the new
patterns intentionally break. Rework that test (and add coverage) so that:

- The four xAI transport patterns are still present (assert on the explicit list or a
  subset check rather than the blanket `all(...)` predicate).
- A new test feeds the captured live stderr text (the
  `Error running LLM provider command (exit code 1)` + `max_tokens_truncation` JSON
  snippet) through `is_retryable_error` with `GrokProvider().llm_default_retry_config()`
  and asserts it is retryable, and through `find_retry_config_for_error` asserting a
  config is found (guarding the cross-provider lookup path `handle_workflow_error`
  actually uses).

## Constraints

- Do not change `wait_times` or add per-pattern wait handling; the 60s first-retry wait
  is acceptable for this error family and per-pattern scheduling would be new framework
  surface this fix does not need.
- Do not touch the usage-limit config or its precedence over retry classification.
- No CLI surface, config schema, or docs changes.

## Verification

- Run the touched test module plus the retry-defaults suite:
  `tests/llm_provider/test_grok_provider_core.py`,
  `tests/test_llm_provider_retry_defaults.py`.
- Run `just check` per the two-speed verification rule before finishing.
