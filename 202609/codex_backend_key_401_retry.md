---
tier: tale
title: Retry Codex's OpenAI-side backend-key 401
goal:
  A Codex run that hits the ChatGPT Codex backend's own-key 401 during an OpenAI outage
  retries with the existing backoff, while API-key-mode and expired-login 401s stay
  terminal.
size: small
proposed_by: bbugyi200.athena.0sk
create_time: 2026-09-25 19:07:25
status: wip
---

# Retry Codex's OpenAI-side "backend key" 401 instead of failing the agent

## Background

On 2026-09-25 the follow-up agent `0sg--3` (Codex, `gpt-5.6-terra`) failed about 40
seconds into its turn. A second Codex agent (`sase-17x.13.10.6--4`) failed the same way
nine minutes later. The Codex CLI stderr, captured in `done.json["error"]`, was:

```text
[error] Reconnecting... 2/5 (stream disconnected before completion: websocket closed by server before response.completed)
...
[error] Reconnecting... 5/5 (stream disconnected before completion: websocket closed by server before response.completed)
[error] Reconnecting... 1/5 (unexpected status 401 Unauthorized: Incorrect API key provided: sk-svcac****fvMA. You can find your API key at https://platform.openai.com/account/api-keys., url: https://chatgpt.com/backend-api/codex/responses, cf-ray: a40d8c967a8d8c87-EWR, request id: e8dd28c2-a3ab-4f90-8613-b16d975bfee2)
...
[error] unexpected status 401 Unauthorized: Incorrect API key provided: sk-svcac****fvMA. You can find your API key at https://platform.openai.com/account/api-keys., url: https://chatgpt.com/backend-api/codex/responses, cf-ray: a40d8d0b4aa9aa2a-EWR, request id: 2e5d0c49-2088-48cb-8468-3cd2cff06132
[turn.failed] unexpected status 401 Unauthorized: Incorrect API key provided: sk-svcac****fvMA. You can find your API key at https://platform.openai.com/account/api-keys., url: https://chatgpt.com/backend-api/codex/responses, cf-ray: a40d8d0b4aa9aa2a-EWR, request id: 2e5d0c49-2088-48cb-8468-3cd2cff06132
```

This was an OpenAI-side incident: status.openai.com reported "Codex Outage (401 Backend
Key Error)". Every Codex model failed with the same error. The local login was healthy:
`~/.codex/auth.json` uses `auth_mode: chatgpt` with OAuth JWTs, holds no API key, and
Codex had just refreshed it. The `sk-svcac…` key in the message is the ChatGPT Codex
backend's own service-account key, not a user credential.

SASE's Codex retry policy (`llm_default_retry_config()` in
`src/sase/llm_provider/codex.py`, plus the bundled `llm_provider.retry.codex` entry)
only matches rate-limit, capacity, websocket-connect, and turn-integrity text. This
error matched none of those, so the run failed immediately instead of taking the
existing `[60, 300, 1800]` backoff. The session then needed a manual retry.

## Goal

Treat the ChatGPT Codex backend's "Incorrect API key provided" 401 as transient and
retryable. Keep real, persistent user-auth failures terminal, as documented and tested
today.

## Design

Retry matching (`is_retryable_error`) is a case-insensitive substring match over the
whole error text, so one pattern cannot express "A and B". Pick a single literal that
spans both facts:

1. the OpenAI invalid-API-key message, and
2. the ChatGPT-auth Codex endpoint (`https://chatgpt.com/backend-api/codex/responses`).

In ChatGPT-login mode, SASE-launched Codex never sends an `sk-…` key. So an
invalid-API-key response from that endpoint can only mean the backend's own key is bad.
The masked key sits between "Incorrect API key provided:" and the rest of the message,
so anchor on the fixed tail:

```text
You can find your API key at https://platform.openai.com/account/api-keys., url: https://chatgpt.com/backend-api/codex/responses
```

The implementer may shorten this if it stays unambiguous. For example,
`api-keys., url: https://chatgpt.com/backend-api/codex/responses` works. Whatever
literal is chosen must:

- match the observed `0sg--3` output above;
- not match an API-key-mode invalid key. In that mode Codex calls
  `https://api.openai.com/v1/responses` with the user's own key, and the failure must
  stay terminal;
- not match the ChatGPT login-invalidation 401 observed in July 2026:
  `unexpected status 401 Unauthorized: Encountered invalidated oauth token for user, failing request, url: https://chatgpt.com/backend-api/codex/responses`.
  That is a real expired or revoked login and must stay terminal.

Do not add a bare `401 Unauthorized`, `Incorrect API key provided`, or
`stream disconnected before completion` pattern. They are too broad and would retry
persistent auth failures or unrelated errors.

## Changes

1. **`src/sase/llm_provider/codex.py`**: in `llm_default_retry_config()`, append the new
   literal to `error_patterns`. Add a short comment in the style of the Claude
   OAuth-refresh-lock comment in `claude.py`. It should say this is the ChatGPT Codex
   backend's own service-account key failing (an OpenAI-side outage) and that
   user-credential failures stay terminal. Leave `max_retries`, `wait_times`,
   `continuation_prompt`, and `preserve_workspace` unchanged. The existing
   `[60, 300, 1800]` backoff is the right cool-down for a provider outage.
   - Do not edit `src/sase/default_config.yml`. Provider-hook patterns are always merged
     ahead of configured patterns (`_merge_with_built_in` in
     `src/sase/llm_provider/retry_config.py`). The Claude precedent (commit "fix(llm):
     retry Claude OAuth refresh lock contention") also changed only the hook.
2. **`tests/test_llm_provider_retry_defaults.py`**, in `TestCodexBuiltInDefaults`:
   - Add a `_CODEX_BACKEND_KEY_OUTAGE_FAILURE` fixture built from the observed output
     above. Keep the websocket-closed reconnect lines and the final `[turn.failed]`
     line, with the masked key shortened as shown.
   - Assert `is_retryable_error(...)` is `True` against the Codex built-in config, and
     that `find_retry_config_for_error(...)` finds it (mirror the existing
     `test_codex_capacity_failure_*` pair).
   - Add negative tests asserting both of these are not retried by the Codex config or
     the finder: the API-key-mode invalid key against
     `https://api.openai.com/v1/responses`, and the ChatGPT "Encountered invalidated
     oauth token for user" 401. Mirror `test_codex_persistent_auth_failure_not_retried`.
   - If any existing test asserts the exact Codex built-in `error_patterns` list or its
     length, update it to include the new literal.
3. **`docs/llms.md`**: in "Provider-Supplied Retry Defaults" → "Codex", add the new
   pattern to the error-pattern list. Explain in one clause that it covers the ChatGPT
   Codex backend's own-key 401 during an OpenAI-side outage, while API-key-mode and
   expired-login 401s stay terminal. Leave the bundled-config list (the "Codex:" block
   near "A bare `403 Forbidden` is deliberately excluded…") unchanged unless the bundled
   config is also changed. It should not be.

## Verification

- `just test tests/test_llm_provider_retry_defaults.py` (through `sase tool run`, as the
  lint/test memory requires) passes, including the new positive and negative cases.
- `just check` passes.
- Spot check: `find_retry_config_for_error()` returns the Codex config for the `0sg--3`
  `done.json["error"]` text. It returns `None` for both negative fixtures.

## Out of scope

- The "state db returned stale rollout path" `ERROR` noise that fills Codex stderr. Task
  bead `sase-11s` already tracks it.
- Changing retry backoff timing or adding provider-outage detection beyond this one
  pattern.
