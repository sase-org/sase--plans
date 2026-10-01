---
tier: tale
title: Retry Muse transient model-service failures
goal:
  A Muse agent whose run dies because Muse's own model-stream retries ran out (504s or
  first-event timeouts) is retried in a fresh Muse session with its workspace preserved,
  instead of failing terminally and stranding its epic.
size: small
proposed_by: bbugyi200.apollo.3r
create_time: 2026-09-30 21:32:46
status: wip
---

# Retry Muse transient model-service failures

## Why

Agent `bob-cli-31.4` was a Muse (`muse-spark-1.3-contributor`) epic phase worker. It
FAILED after about 37 minutes of productive work. The Muse session log shows that every
model request for that session got an HTTP 504 (about 60 s each) or a first-event
timeout for 6m9s. Muse then exhausted its own turn retry budget and exited 1 with:

```
run ended with Failed: no data is reaching this machine from the model service — likely
a local network issue; check connectivity and retry (3 failed attempts (9 provider
requests) over 6m9s, all [model_stream_first_event_timeout])
```

A second Muse agent on the same model kept working through the same window. The failure
was therefore transient and scoped to that session, and a fresh Muse session would very
likely have survived it.

SASE never retried. `handle_workflow_error` in `src/sase/axe/run_agent_exec_retry.py`
retries only when some provider retry config pattern matches the error. Claude, Codex,
Grok, Agy, and Fakey ship `llm_default_retry_config()`. Muse ships none, and neither the
bundled `default_config.yml` nor user config has an `llm_provider.retry.muse` entry.
`find_retry_config_for_error(<that error>)` returns `None` today (verified), so the
error was raised immediately. As a result:

- The phase bead stayed `in_progress`.
- The workspace stayed held.
- Every downstream phase and the epic's land agent now wait indefinitely for a human.

`docs/llms.md` (Muse section, "Interrupts and Retries") also says a nonzero Muse exit
"falls into SASE's generic retry path". No such path exists: an error that matches no
pattern is terminal.

A smaller, related defect showed up in the same run. When Muse emits a
`run.terminal.failed` event with no `text`, `_resolve_muse_content` in
`src/sase/llm_provider/_subprocess_muse.py` still records the schema-drift diagnostic
`muse_missing_run_terminal_event` into `tool_calls_writer_errors.jsonl`. The terminal
event did arrive; it just carried no reply text. The diagnostic falsely signals schema
drift on every failed Muse run.

## Changes

### 1. Muse provider retry defaults (`src/sase/llm_provider/muse.py`)

Add an `@hookimpl def llm_default_retry_config(self) -> ProviderRetryConfig` to
`MuseProvider`, shaped like the Grok and Codex hooks. Import `_RETRY_CONTINUATION_NUDGE`
and `ProviderRetryConfig` lazily inside the hook, as they do. Add `ProviderRetryConfig`
to the existing `TYPE_CHECKING` import block.

- `max_retries=3`
- `wait_times=[60, 300, 1800]`
- `continuation_prompt=_RETRY_CONTINUATION_NUDGE`
- `preserve_workspace=True`
- Leave `spawn_new_agent` at its default `False`: in-process retry, like the other
  providers.
- `error_patterns`, each with a provenance comment:
  - `"no data is reaching this machine from the model service"`: captured live from the
    2026-10-01 `bob-cli-31.4` failure (Muse 1.4.2-R4684.1). It is Muse's zero-byte chain
    terminal, printed after its own turn retry budget is exhausted.
  - `"model_stream_first_event_timeout"` and `"model_stream_idle_timeout"`: the machine
    error kinds that Muse lists in that give-up summary's `all [...]` suffix. They are a
    second anchor in case a later build rewords the prose, the same dual-anchor
    rationale as Grok's `max_tokens_truncation`.
  - `"kept failing until the whole turn retry budget was exhausted"`: the sibling
    give-up prose for the transport, service, router, and stream-ended failure classes.
    It comes from scanning the shipped binary and has not been observed live; say so in
    the comment, as Grok's usage-limit comment does.
- Deliberately exclude broad wording such as `"server error"`, `"rate limited"`,
  `"connection failed"`, or bare `"504"`. These are Muse per-attempt status vocabulary,
  and `find_retry_config_for_error` applies every provider's patterns to every
  provider's errors. Exclude anything that would match auth or configuration failures,
  or `"usage limit reached"`, so usage-limit precedence stays in charge.
- Before committing, confirm that each pattern string is present in the installed Muse
  binary. The `muse` launcher on `PATH` execs `muse-bin-<version>` from its own
  directory, and the version is in `.muse-version` there. Run `strings` on that binary
  and use a fixed-string search. If a binary-scanned pattern is missing from the current
  build, drop that pattern; keep the live-captured one regardless.
- Add a short comment: Muse has no headless resume, so a SASE retry starts a fresh
  `muse exec` session (new `--session-id`) with the nudge prepended. On-disk edits
  survive because `preserve_workspace=True`. A fresh session is also what escapes the
  session-scoped 504s seen here.

Do not add a `muse` entry to `src/sase/default_config.yml`. Grok's defaults are
hook-only, and Muse should follow that precedent.

### 2. Stop the false missing-terminal diagnostic (`src/sase/llm_provider/_subprocess_muse.py`)

- Add `saw_run_terminal: bool = False` to `_MuseStreamState`.
- Set it in `_capture_run_terminal`.
- In `_resolve_muse_content`, keep the existing delta-salvage return. Record the
  `muse_missing_run_terminal_event` diagnostic only when `saw_run_terminal` is false.
  Update the docstring or comment to match.

### 3. Docs (`docs/llms.md`)

- Muse "Interrupts and Retries": replace the false "generic retry path" sentence. The
  new text should say:
  - Muse retries its model stream internally.
  - When that budget is exhausted on a transient model-service failure, it exits 1.
  - Muse's provider-supplied retry defaults then re-run the agent in a fresh Muse
    session, with the workspace preserved and the resume nudge prepended.
- "Provider-Supplied Retry Defaults": add Muse to the sentence that lists the providers
  declaring a recovery entry. Add a `Muse:` block after `Grok:` in the same format:
  patterns with provenance, `max_retries`, `wait_times`, `continuation_prompt`, and
  `preserve_workspace`.

### 4. Tests

- New `tests/llm_provider/test_muse_retry_config.py`. Use a module constant holding the
  captured error text, with a provenance comment. Trim it to the
  `Error running LLM provider command (exit code 1)` header, the `stderr:` CLI line
  `run ended with Failed: no data is reaching ...`, one
  `[muse] task rejected: skip_if_running` noise line, and the
  `[muse] run terminal failed: no data is reaching ...` diagnostic. Cover:
  - The hook's shape: `max_retries == 3`, `wait_times == [60, 300, 1800]`,
    `preserve_workspace is True`, `continuation_prompt == _RETRY_CONTINUATION_NUDGE`,
    `spawn_new_agent is False`.
  - `is_retryable_error(captured, config)` is true.
  - `get_retry_config("muse")` is not `None`, and
    `find_retry_config_for_error(captured)` returns a config that matches. This guards
    the lookup path that `handle_workflow_error` actually uses, mirroring
    `test_grok_max_tokens_truncation_error_found_via_cross_provider_lookup`.
  - The Muse config does not match non-transient Muse failures:
    - the exit-2 usage-error note (`MUSE_USAGE_ERROR_NOTE` from `_subprocess_muse`)
    - the stranded wait-claim `LLMInvocationError` text,
      `Muse produced a wait-claim reply after ...`
    - a usage-limit line containing `usage limit reached`
- `tests/llm_provider/test_muse_provider_stream.py`: in
  `test_muse_stream_reports_a_non_completed_terminal_outcome`, or in a sibling test, set
  `SASE_ARTIFACTS_DIR` to `tmp_path`. Feed a textless `run.terminal.failed` envelope and
  assert that no `muse_missing_run_terminal_event` entry is written. The existing
  truly-missing-terminal tests must keep passing unchanged.

## Verification

- Run the new and touched tests directly with pytest.
- Run `just check` the way the `lint_and_test` reference memory prescribes.

## Out of scope

- Recovering the stranded `bob-cli-31.4` run and its held workspace. That is an operator
  action, not a code change.
- Muse's own misleading "likely a local network issue" wording for what were server-side
  504s. That belongs to the upstream CLI.
- Any Muse headless-resume work. `muse exec` has no resume flag.
