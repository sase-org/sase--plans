---
create_time: 2026-07-10 16:19:55
status: done
prompt: 202607/prompts/codex_at_capacity_retry.md
tier: tale
---
# Plan: Retry Codex "Selected model is at capacity" Failures by Default

## Problem

A Codex agent (CODEX / `gpt-5.6-sol`) failed with a `WorkflowExecutionError` on the `main` step. The captured provider
output was:

```
WorkflowExecutionError: Step 'main' failed: Error running LLM provider command (exit code 1)
stderr: [error] Selected model is at capacity. Please try a different model.
[turn.failed] Selected model is at capacity. Please try a different model.
```

This is a **transient** capacity failure: the selected model is temporarily saturated and the request should succeed
after a cool-down. SASE already has a per-provider retry subsystem that recovers from configured provider errors, but
the current Codex retry patterns do not include any wording matching this failure, so it is treated as terminal and the
agent dies instead of retrying.

## Background: how Codex retry defaults are assembled

SASE assembles per-provider retry config from two layers (see `src/sase/llm_provider/retry_config.py`):

1. **Built-in defaults** — the `llm_default_retry_config()` hook implemented by each provider. The Codex implementation
   lives in `src/sase/llm_provider/codex.py` and currently ships:
   - `error_patterns`: `"exceeded retry limit"`, `"429 Too Many Requests"`, `"Too Many Requests"`, `"rate limit"`,
     `"failed to connect to websocket"`
   - `max_retries: 3`, `wait_times: [60, 300, 1800]`, `preserve_workspace: True`, and a continuation nudge prompt. These
     built-ins are the true "works out of the box with no user config" defaults — the Codex tests confirm this by
     mocking `load_merged_config` to return `{}` and asserting the patterns still apply.

2. **Shipped default config** — `src/sase/default_config.yml` under `llm_provider.retry.codex.error_patterns`, which
   currently **mirrors the same five patterns**. At runtime `get_retry_config("codex")` unions the two lists (deduped),
   so the two layers are intentionally kept in parity.

At failure time, `handle_workflow_error()` (`src/sase/axe/run_agent_exec_retry.py`) calls
`find_retry_config_for_error(str(exc))`. Because `str(exc)` contains the full `[error] Selected model is at capacity...`
text, a case-insensitive substring pattern will match and trigger the retry/cool-down loop.

## Chosen pattern string

Add the pattern **`"Selected model is at capacity"`**.

Rationale:

- It is the exact, distinctive phrase Codex emits on both the `[error]` and `[turn.failed]` lines, so case-insensitive
  substring matching catches it reliably.
- It is specific enough to avoid false positives — it will not accidentally match unrelated "at capacity" wording or the
  existing persistent-auth negative-test case, keeping terminal failures terminal.
- Consistent with the specificity of the existing Codex patterns (e.g. `"429 Too Many Requests"`).

The existing Codex defaults are the right recovery behavior for this failure: `wait_times` of 60s / 5m / 30m give
capacity time to free up, and re-running the **same** model is appropriate since the saturation is transient.

## Changes

Keep all four synced locations consistent (behavior lives in layer 1; the rest preserve parity and documentation):

1. **`src/sase/llm_provider/codex.py`** — add `"Selected model is at capacity"` to the `error_patterns` list returned by
   `llm_default_retry_config()`. This is the change that actually makes retry work by default.

2. **`src/sase/default_config.yml`** — add `- "Selected model is at capacity"` to the
   `llm_provider.retry.codex.error_patterns` list to keep the shipped default in parity with the built-in.

3. **`docs/llms.md`** — update the documented Codex error-pattern list (currently around the "error patterns" bullet and
   the inline pattern array) to include the new string.

4. **`tests/test_llm_provider_retry_config.py`** — in `TestCodexBuiltInDefaults`:
   - Add a fixture constant for the observed capacity failure text (the literal
     `[error] Selected model is at capacity. Please try a different model.` / `[turn.failed] ...` output).
   - Assert `"Selected model is at capacity"` is present in the built-in `error_patterns`.
   - Assert `is_retryable_error(<capacity failure text>, config) is True` and that
     `find_retry_config_for_error(<capacity failure text>)` returns the Codex config.
   - Confirm the existing persistent-auth negative test still passes (no over-matching).

## Out of scope / open question

The Codex message literally says "Please try a different model," which hints a `fallback_model` could be appropriate.
The current Codex built-in defines no fallback (unlike Claude's `"sonnet"`), and choosing a specific Codex fallback
model is a separate product decision. This plan deliberately keeps scope to adding the retry pattern so the failure is
retried on the same model after a cool-down; adding a Codex `fallback_model` can be a follow-up if same-model retries
prove insufficient.

## Verification

- `just install` (ephemeral workspace may have stale deps), then `just check`.
- Targeted run of `tests/test_llm_provider_retry_config.py` (the `TestCodexBuiltInDefaults` class) to confirm the new
  pattern matches the capacity failure and does not regress the negative cases.
