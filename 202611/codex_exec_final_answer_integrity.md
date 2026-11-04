---
tier: tale
title: Stop false Codex turn-integrity failures on codex exec JSON streams
goal:
  A Codex turn that ends with a final agent message is never failed as a turn-integrity
  error because of earlier mid-turn command failures, so staged handoffs (like
  bob-cli-h0.8's monitor) are adopted, while real aborted turns are still caught.
size: small
proposed_by: bbugyi200.apollo.cd
proposed_with: claude/opus
create_time: 2026-11-04 15:09:37
status: wip
---

# Stop false Codex turn-integrity failures on `codex exec --json` streams

## Problem

Phase agent `bob-cli-h0.8` (Codex provider, `gpt-6-luna`, codex-cli 0.166.0) did 53
minutes of real work. It staged a `sase monitor start` handoff, emitted a proper final
agent message, and Codex reported `turn.completed`. The SASE Codex stream parser still
raised:

```
Codex turn integrity failure: task completed with no final agent message and command
execution was killed at teardown (exit_code -1): item_143 (...), item_208 (...), item_223 (...)
```

That `LLMInvocationError` fired before `_invoke.py` persisted the ordinary provider
response for the staged handoff. The host therefore held the handoff ("ordinary provider
response was not persisted for this staged handoff"), marked the run FAILED, and never
launched the monitor. The phase bead stays IN_PROGRESS with its work uncommitted, and
the epic's land agent (`bob-cli-h0.land`) waits on that bead with nothing left to wake
it. Evidence lives in the origin artifacts dir
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202611/04/20261104085126/`
(`.sase_provider_failure.json`, `tool_calls.jsonl`, `live_reply.md`,
`.sase_handoff_adopted.json`).

This is a false positive with two independent root causes, both in
`src/sase/llm_provider/_subprocess_codex.py` (`_CodexTurnIntegrityState`):

1. **The final answer is never recognized in the real stdout format.**
   `has_nonempty_final_answer` is set only from an agent message whose `phase` is
   `final_answer`, or from a non-empty `last_agent_message` on a `task_complete` event.
   Those fields exist in Codex's rollout/`event_msg` format (the format of every
   integrity fixture), but `codex exec --json` stdout carries neither. A live probe of
   codex-cli 0.166.0 printed exactly:

   ```json
   {"type":"thread.started","thread_id":"01a1a828-..."}
   {"type":"turn.started"}
   {"type":"item.completed","item":{"id":"item_0","type":"agent_message","text":"OK"}}
   {"type":"turn.completed","usage":{"input_tokens":15571,"cached_input_tokens":11264,"cache_write_input_tokens":0,"output_tokens":5,"reasoning_output_tokens":0}}
   ```

   So for real Codex runs, the "no final agent message" half of the check is always
   true. The check reduces to "any `exit_code: -1` failed command anywhere in the turn
   fails the turn."

2. **Mid-turn command failures are classified as teardown kills.**
   `_is_killed_teardown_command` matches any `exit_code == -1` completion with a failed
   status, no matter when it happened. In h0.8, `item_143`, `item_208`, and `item_223`
   were spawn failures 25 minutes before the turn ended. Each was
   `Failed to create unified exec process: No such file or directory (os error 2)`, with
   a result about 5 ms after start, and the agent ran about 100 more commands after
   them. The same misclassification also feeds `stranded_commands()`. A mid-turn spawn
   failure of a `sase monitor start` that the agent later retried successfully would
   wrongly trip the stranded-handoff re-drive in `CodexProvider.invoke`
   (`src/sase/llm_provider/codex.py`).

The check was added by sase-11t.2 (commit `06a53a0e51`) to catch real aborted turns. It
must keep catching those: task complete, no final answer, and a command killed at
teardown or left without a result.

## Changes

### 1. Recognize the exec-format final answer (`_subprocess_codex.py`)

In `_CodexTurnIntegrityState.observe`:

- Keep the existing rules. An agent message with `phase == "final_answer"`, or a
  non-empty `last_agent_message` on `task_complete` / `turn.completed`, sets
  `has_nonempty_final_answer`.
- Add the exec-format rule. A completed agent message with non-empty text and **no
  `phase` field** becomes a _candidate_ final answer. Any later `item.started` before
  turn completion clears the candidate, because new work began, so it was commentary.
  Completions of commands already in flight when the message was recorded do **not**
  clear it: those are teardown kills of work the final answer already reported. When the
  turn completes with a standing candidate, set `has_nonempty_final_answer`.
- A message with an explicit non-final phase (for example `commentary`) never becomes a
  candidate. This keeps `codex-cli-stranded-monitor-*.jsonl` and the rollout-format
  fixtures behaving exactly as today.

Note the ordering difference: in rollout fixtures the teardown completions arrive after
`task_complete`. Keep the state machine order-tolerant, and make "turn complete" a flag
rather than a cutoff, as it is today.

### 2. Teardown kills are only kills the turn never moved past

- An `exit_code == -1` failed command completion is recorded as a _provisional_ kill.
- When a later `item.started` arrives, or a later agent message completes, discard every
  provisional kill recorded before it. The turn demonstrably continued, so those were
  mid-turn failures (spawn errors and similar) that the agent already saw as tool
  results. They are neither integrity failures nor stranded handoffs.
- Kills with no subsequent new item or agent message stay in `killed_commands` and keep
  their current `stranded_commands()` / `integrity_error()` treatment.
- Leave `pending_commands` (started-without-result) semantics unchanged.

Keep both rules small and documented in the class docstring. Do not key on the
`Failed to create unified exec process` text; the positional rule covers it and any
other mid-turn `-1`.

### 3. Fixtures and tests

Add `tests/fixtures/codex_stream/codex-cli-0.166.0-midturn-spawn-failure.jsonl`. It is a
hand-distilled, exec-format reproduction of the h0.8 stream; do not copy real
transcripts:

- `thread.started`, `turn.started`.
- `item.started` / `item.completed` for `item_143`: `type: command_execution`, a
  `/usr/bin/zsh -lc "rg ..."` command, completed with `exit_code: -1`,
  `status: "failed"`, and `aggregated_output` set to the spawn-failure text.
- A later successful command (`item_144`, `exit_code: 0`, `status: "completed"`).
- A phaseless commentary `agent_message`.
- A successful `sase monitor start ...` command (started and completed, exit 0).
- A phaseless final `agent_message`.
- `turn.completed` with usage only.

Add tests next to `tests/test_llm_provider_codex_parser_integrity.py`; split into a new
file if that one would grow past the repo's size lint:

- The new fixture yields `integrity_error is None`, `stranded_commands == ()`, and
  content containing the final message text.
- The exec-format real-abort cases still fail:
  - A phaseless commentary message, then a non-handoff command that starts and completes
    with `-1`, then `turn.completed`, gives an integrity error containing
    `killed at teardown`.
  - A command started without a result and no message gives `started without a result`.
- A phaseless final message emitted while a non-handoff command is in flight, followed
  by that command's `-1` completion and `turn.completed`, gives no integrity error. The
  command is still reported in `stranded_commands` as `killed_at_teardown`.
- A mid-turn `sase monitor start` completed with `-1`, followed by a successful retry of
  the same command and a phaseless final message, gives `stranded_commands == ()`. Also
  add a provider-level assertion in the style of
  `tests/llm_provider/test_codex_handoff_guard.py` that `CodexProvider` does not run a
  stranded-handoff continuation for it.
- All existing Codex parser, integrity, and handoff-guard tests stay green unchanged,
  including `codex-cli-aborted-empty-final.jsonl` and both
  `codex-cli-stranded-monitor-*.jsonl` fixtures.

### 4. Docs

Update the "Turn Integrity Check" section of `docs/reference/providers/codex.md` in two
ways. First, a final answer is recognized either from Codex's explicit final-answer
signals or, on `codex exec --json` streams, from the last agent message that no new item
follows. Second, an `exit_code` `-1` command counts as killed at teardown only when the
turn did not continue past it. Follow the docs-writing memory note. Check whether the
matching sentence in `docs/reference/providers/retry-and-fallback.md` still reads
correctly; change it only if it now states something false.

## Non-goals

- Do not change host staged-handoff adoption. The fail-closed hold when a provider
  raises after staging stays as designed (`explicit-handoff-fails-closed`,
  `handoffs-end-through-host-completion`). This plan removes the false raise only.
- No Rust-core change. Normalizing a provider CLI's stream is the provider adapter's job
  (`adapters-normalize-harnesses`).
- Do not recover the stalled `bob-cli-h0.8` run or its held workspace; that is an
  operator action outside this repo change.
- Do not change the Codex retry pattern list or the integrity error message prefix.

## Verification

- Targeted:
  `pytest tests/test_llm_provider_codex_parser*.py tests/llm_provider/test_codex_handoff_guard.py tests/test_llm_provider_retry_defaults.py`.
- Then the repo's standard gate per the lint-and-test memory note (`just check`, run
  through `sase tool run`).
