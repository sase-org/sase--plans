---
tier: tale
title: Guard the claude provider against single-turn background waits
goal:
  Claude-provider SASE agents never fail by ending their turn to wait for a background
  notification; backgrounding is disabled at the source, and any residual wait-state
  reply is continued in-session or fails loudly.
size: medium
proposed_by: bbugyi200.athena.0lg
create_time: 2026-09-15 14:42:08
status: wip
---

# Claude Provider: Stop Single-Turn Agents From Ending Their Turn To Wait

## Problem

Claude-provider SASE agents fail when the model ends its provider turn waiting for a
background notification that can never arrive. SASE agents are single-turn (see the
`single-turn-agents` decision): `claude -p` exits when the model gives its final
response, so no background-task notification or scheduled wake-up is ever delivered.

Diagnosed failure (`sase-10w.3`, corroborated by `0iw--code` and `sase-xr.land--2`,
across both sonnet and claude-fable-5): during a commit-finalizer conflict-repair turn,
the agent ran `just test-visual` with the Bash tool's maximum 10-minute timeout. Claude
Code's Bash tool auto-backgrounded the command when it hit the timeout ("Command did not
complete within its ...s timeout and was moved to the background (ID: ...). ... You will
be notified when it completes."). The model then ended its turn with "I'll wait", the
`claude -p` process exited with a rebase still paused, and the run failed with
`second_unresolved_conflict`. Evidence: artifacts dir
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/14/20260914090801/`
(`finalizers/commit/conflict_repair_response.main.md`, `tool_calls.jsonl` tail).

The agy provider already guards against this class of failure
(`src/sase/llm_provider/agy.py`: print-mode directive, no-progress classification,
bounded continuations, then `LLMInvocationError`). The claude provider
(`src/sase/llm_provider/claude.py`, `src/sase/llm_provider/_subprocess_claude.py`) has
no equivalent.

## Verified CLI facts (Claude Code 2.1.272, inspected on this host)

These make prevention possible, not just detection:

1. `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` (env var) gates every backgrounding path in
   the Bash tool: it removes the `run_in_background` capability, makes the
   background-all action throw "Background tasks are disabled in this session.", and —
   critically — disables auto-backgrounding on timeout and on turn-abort (the CLI
   computes `disabled = backgroundTasksDisabled || CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`
   and derives `canAutoBackground`/`turnAbortBackgrounds` from it). With it set, a
   command that hits its timeout is killed and reported as a synchronous timeout failure
   the model can react to in the same turn, instead of detaching.
2. `BASH_MAX_TIMEOUT_MS` raises the cap on the Bash tool's per-command timeout, so long
   verification suites can complete in the foreground instead of being killed at the
   default 10-minute maximum.
3. Print mode supports `--resume <session-id>`, and the provider already passes
   `--session-id <uuid>`, so a wait-state reply can be continued in the same session
   with full context (unlike agy, which must rebuild context in a fresh prompt).
4. Bash tool results for backgrounded commands carry a structured `backgroundTaskId` in
   the stream-json user event's top-level `tool_use_result` envelope, plus recognizable
   text ("moved to the background (ID: ...)", "Command running in background with ID:
   ...", "Output is being written to: ..."). Mid-turn completion arrives as a user
   message containing a `<task-notification>` block with `<task-id>...</task-id>` and
   `<status>completed</status>`.
5. `--append-system-prompt <text>` and `--disallowedTools <names>` work in print mode.

## Fix design: three layers

Layer 1 prevents the failure at the source; layer 2 tells the model the ground rules;
layer 3 is a backstop that turns any residual wait-state reply into a bounded in-session
continuation and, ultimately, a loud failure instead of a silent bad success.

### Layer 1 — Prevention (subprocess env + args)

In `ClaudeCodeProvider._run_subprocess` (`src/sase/llm_provider/claude.py`):

- Build an explicit env for `subprocess.Popen` (`env = os.environ.copy()`) and apply
  `env.setdefault(...)` (setdefault so an operator-set value always wins; no feature
  flag — this is a reliability bug fix, and per the SASE flag rules a permanent operator
  choice is an env/config knob, not a flag):
  - `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS = "1"`
  - `BASH_MAX_TIMEOUT_MS = "14400000"` (4h; module constant with a brief comment —
    generous like agy's 24h `--print-timeout`, so real verification suites can finish in
    the foreground). Do NOT set `BASH_DEFAULT_TIMEOUT_MS`: commands without an explicit
    timeout should still fail fast, and with backgrounding disabled a timeout is now a
    synchronous, recoverable error the model can retry with a larger explicit timeout.
- Add `--disallowedTools` `ScheduleWakeup` to the base args in `_invoke_loop`. A denied
  call surfaces an error to the model while its turn is still alive (it can adapt
  immediately), which beats detecting a successfully-scheduled wake-up that will never
  fire. `SASE_CLAUDE_LARGE_ARGS`/`SASE_CLAUDE_SMALL_ARGS` never set `--disallowedTools`
  today, so there is no collision.

### Layer 2 — Single-turn directive (system prompt)

Add a module-level directive constant in `claude.py` and pass it via
`--append-system-prompt` in the base args. Wording along these lines (final wording up
to the implementer, must cover all four points):

> SASE single-turn print mode: this session runs exactly one provider turn with no
> follow-up events. Background-task notifications, scheduled wake-ups, and "you will be
> notified" promises can never reach you; anything still running when you give your
> final response is lost. Run commands synchronously in the foreground, and if a command
> is killed by its timeout, rerun it with a larger explicit timeout. Never end your turn
> to wait.

This also covers paths layer 1 cannot block, e.g. the model manually detaching work with
`nohup ... &` and waiting for it.

### Layer 3 — Wait-state detection + bounded in-session continuation

**Tracking (`src/sase/llm_provider/_subprocess_claude.py`):**

Add a small mutable collector (e.g. `@dataclass ClaudeTurnWaitState`) passed as a new
optional keyword through `stream_and_parse_json_output` /
`_stream_and_parse_messages_json_output` into `_process_json_line` (an out-parameter, so
the existing 4-tuple return shape and the grok/muse callers of
`stream_and_parse_messages_json_output` are untouched; when the keyword is omitted,
behavior is unchanged). It records:

- **Outstanding background tasks**: on user events, a `backgroundTaskId` in the
  top-level `tool_use_result` envelope (primary), falling back to a text regex on
  `tool_result` content for the CLI's backgrounding messages
  (`"moved to the background (ID: <id>)"`, `"backgrounded by user with ID: <id>"`,
  `"running in background with ID: <id>"`). Store the ids in a set.
- **Resolved tasks**: on user events whose text content contains a `<task-notification>`
  block, extract `<task-id>` and remove it from the outstanding set (any terminal
  `<status>` counts as resolved).
- **ScheduleWakeup use**: on assistant `tool_use` blocks named `ScheduleWakeup` whose
  input does not set `stop: true` (belt in case the disallow is bypassed or renamed).
- **Final assistant text tail**: the last non-empty assistant text block, for the
  classifier below.

**Classification (`claude.py`, module-level helper mirroring agy's, with its own
claude-tuned regexes — do not reuse agy's private patterns across modules):**

A completed turn (return code 0, no pending interrupt) is a _wait-state reply_ when:

- the outstanding-background-task set is non-empty AND the final text tail (last ~700
  chars) matches a wait-signal regex ("I'll wait", "will be notified", "notify me",
  "when it completes/finishes", "waiting for", "still running", "in the background"); OR
- a non-stop `ScheduleWakeup` tool_use was recorded.

Requiring text corroboration for the background-task case keeps precision high: a turn
that leaves an incidental background task behind but delivers a real final answer is
accepted (the task is lost either way; that is today's benign behavior).

**Continuation loop (`claude.py` `_invoke_loop`):**

- Track `wait_continuations` alongside `cycle`, capped by a new
  `SASE_CLAUDE_MAX_WAIT_CONTINUATIONS` env knob (default 2, parsed with the same
  `max(0, int(...))`-with-fallback shape as agy's `_agy_max_no_progress_continuations`).
- On a wait-state reply under the cap: append the turn's text to `response_content`,
  then rerun with the base args but `--resume <session_uuid>` replacing
  `--session-id <session_uuid>` (same session, full context preserved), sending a nudge
  prompt on stdin, e.g.: "Your previous reply ended the turn waiting for a background
  notification or wake-up that will never arrive; this session is single-turn. Finish
  the work now: read the background task's output file directly or rerun the command
  synchronously in the foreground, then give your final answer. Do not end your turn
  waiting." The loop currently mints a fresh `session_uuid` every iteration; restructure
  so a continuation reuses the prior turn's uuid while interrupt restarts keep today's
  fresh-session behavior.
- On a wait-state reply at the cap: raise `LLMInvocationError` mirroring agy's wording,
  e.g. "Claude produced a wait-for-background reply after N continuation(s); refusing to
  report it as success. Reason: <reason>. The model is waiting on a notification that
  `claude -p` cannot deliver. Artifacts: <SASE_ARTIFACTS_DIR>." A loud failure is
  strictly better than today's silent bad success (the `sase-10w.3` run "succeeded" from
  the provider's point of view while the rebase sat paused).
- Interrupt handling keeps priority: the pending-interrupt branch runs before wait-state
  classification, unchanged.

## Files to change

- `src/sase/llm_provider/claude.py` — env prevention, `--disallowedTools`,
  `--append-system-prompt` directive, wait-state classifier + regexes, continuation loop
  with `--resume`, `SASE_CLAUDE_MAX_WAIT_CONTINUATIONS`, `LLMInvocationError` on cap
  exhaustion.
- `src/sase/llm_provider/_subprocess_claude.py` — `ClaudeTurnWaitState` collector and
  the optional keyword plumbing into `_process_json_line`.
- `tests/llm_provider/test_claude_provider_core.py` — extend, plus a new
  `tests/llm_provider/test_claude_wait_guard.py` (mirror the structure of
  `tests/llm_provider/test_agy_no_progress.py`).

## Tests

Unit tests, no real CLI invocations (mock `subprocess.Popen` +
`stream_and_parse_json_output` exactly as `test_claude_provider_core.py` does today):

1. `_run_subprocess` passes an env to `Popen` containing
   `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` and `BASH_MAX_TIMEOUT_MS`, and a pre-set
   value in `os.environ` wins over the default.
2. Base args include `--append-system-prompt` (with the directive) and
   `--disallowedTools ScheduleWakeup`.
3. Collector: feeding synthetic stream-json lines through `_process_json_line` records a
   `backgroundTaskId` from the `tool_use_result` envelope, records one from text
   fallback, resolves it via a `<task-notification>` user message, records a non-stop
   `ScheduleWakeup` tool_use, and captures the final text tail.
4. Classifier fixtures: wait-y tails ("...so I'll wait.", "I'll be notified
   automatically when it completes") classify as wait-state when a task is outstanding;
   a complete answer with an outstanding task does not; a wait-y tail with no
   outstanding task and no ScheduleWakeup does not.
5. Continuation: first invoke returns a wait-state reply, second returns a clean answer
   → exactly two `Popen` calls, the second using `--resume` with the first call's
   session uuid and the nudge on stdin; combined content returned.
6. Cap exhaustion: wait-state replies past `SASE_CLAUDE_MAX_WAIT_CONTINUATIONS=1` raise
   `LLMInvocationError` naming the reason.
7. Regression: clean single-turn answer → one `Popen` call, unchanged result and usage
   accumulation; interrupt path still restarts with a fresh session id.

## Verification

Run `just check` before finishing (the two-speed-verification default). Also run the
existing claude/agy provider test modules explicitly:
`tests/llm_provider/test_claude_provider_core.py`,
`tests/llm_provider/test_claude_hooks.py`, `tests/llm_provider/test_agy_no_progress.py`
(guards against accidental shared-helper drift), and the new test module.

## Risks and mitigations

- **CLI wording/env-var drift across Claude Code versions**: layer 1 uses a documented
  env var; layer 3's primary signal is the structured `backgroundTaskId` field, with
  text regexes only as fallback. If every layer silently stops working we regress to
  today's behavior, never worse.
- **False-positive continuations**: bounded (default 2), in-session (cheap — resume
  reuses context), and only triggered by structural-plus-text agreement or an explicit
  ScheduleWakeup.
- **Legitimate background use**: SASE has no sanctioned in-provider background path —
  `/sase_monitor` is the sanctioned mechanism and runs outside the provider process.
  Operators can re-enable backgrounding per-run by pre-setting
  `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=` (empty) in the environment.

## Non-goals

- No change to the commit-finalizer conflict-repair prompt (the provider-level directive
  covers all call sites; tighten the prompt separately if failures persist).
- No changes to other providers (codex/qwen/opencode use different CLIs; agy already has
  its guard).
- No Rust core changes: this is provider-invocation glue, which lives with the other
  Python providers in `src/sase/llm_provider/`.
- No feature flag (see layer 1 rationale).
