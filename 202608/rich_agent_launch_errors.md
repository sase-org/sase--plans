---
tier: tale
title: Preserve actionable agent-launch diagnostics in Admin Center Logs
goal:
  The error entry targeted by ACE's last-error keymap preserves the causal launch
  failure, bounded process output, and debugging context without blocking the TUI.
size: medium
proposed_by: bbugyi200.athena.0a4
create_time: 2026-08-21 19:45:46
status: wip
---

# Preserve actionable agent-launch diagnostics in the Admin Center Logs tab

## Summary

Make the launch-failure entry targeted by the ACE `,L` keymap contain the real cause and
enough bounded execution context to debug it. Keep the existing toast, error-id anchor,
Logs-tab source, and jump behavior; repair the diagnostic data that reaches that entry.

This is a medium tale because the defect crosses the durable `sase run` result boundary
and ACE's failure-log adapter, but the contracts and affected call paths are already
identifiable and one implementation agent can complete the change and tests together.

## Problem and root cause

The reported failed proc demonstrates a split-brain diagnostic:

- Its durable proc result has `status: error`, `exit_code: 1`, and only the generic
  message `exited with code 1`.
- Its captured combined stdout/stderr log contains the useful `XPromptArgumentError`,
  the invalid `priority` value, and the complete exception chain.
- `src/sase/main/query_handler/_launch.py::launch_query` emits typed failure results for
  several validation branches, but its general launch exception paths either exit
  without emitting a `run.launch` result or allow an unexpected exception to escape.
  Proc settlement therefore has to synthesize the generic result.
- `src/sase/ace/tui/actions/agent_workflow/_launch_procs.py` records a payloadless
  failed launch using `completion.output`, but the durable proc observer only reads
  output for the proc selected in the Procs detail pane. Normal launch completions
  therefore arrive with empty output even though the supervisor has already flushed a
  useful proc log.
- The submitted prompt is already retained as launch-recovery metadata, but it is popped
  before failure logging and is not supplied as the existing bounded `prompt_preview`
  field.
- Multiline context values are stringified after a single `output:` prefix in the human
  log, which makes a recovered traceback harder to scan in the Logs tab.

## Goals

- Ensure every ordinary agent-launch failure produces the most specific available typed
  `run.launch` error message instead of relying on the supervisor's exit-code fallback.
- Include a bounded tail of the failed launch proc's combined output in the durable
  launch-failure record even when the observer completion did not preload output.
- Include the existing bounded submitted-prompt preview and useful proc metadata so the
  focused entry identifies what ran and where its remaining full log lives.
- Render multiline process output as a readable indented section in
  `launch_failures.log`, which is the source displayed by the Admin Center Logs tab.
- Preserve error-id correlation: the toast, registered session error, JSONL record,
  human-log header, and `,L` focus target must continue to share one ID.
- Keep all new disk I/O off the Textual event loop and keep copied output explicitly
  bounded by both lines and characters.

## Non-goals

- Do not change the `,L` keymap, toast wording, error registry, Logs-pane selection or
  anchor-scanning behavior.
- Do not make the proc observer read log tails for every watched or visible proc; the
  failure adapter can recover one terminal proc's tail off-thread without adding
  periodic observer work.
- Do not change proc wire schemas, Rust core behavior, retention policy, or the Procs
  tab.
- Do not expose request sidecars or full prompts. Continue to use the launch logger's
  existing 200-character prompt-preview contract.

## Implementation

### 1. Publish causal typed failures from `sase run`

Update `src/sase/main/query_handler/_launch.py` so every terminal failure around
`launch_agents_from_cwd()` writes a failed `run.launch` result before the process exits
or re-raises:

- Preserve the `MultiPromptPartialLaunchError` rollback and its detailed counts, but
  build that text once, print it, emit it through `emit_run_launch_result`, and exit
  nonzero.
- For the currently handled general `RuntimeError`, emit the original message before
  exiting nonzero.
- Add an `Exception` fallback for non-`RuntimeError` launch failures such as
  `XPromptArgumentError`. Emit a concise type-plus-message summary, then preserve the
  current unhandled-exception traceback behavior so the combined proc log still contains
  the complete causal chain. Do not catch `KeyboardInterrupt` or `SystemExit`.
- Reuse a small local failure-emission helper if needed so stderr text, typed result
  text, and tests cannot drift across branches.

The result payload should remain absent on failure. ACE currently treats a failed result
with no payload as the payloadless branch that writes and registers the linked
launch-failure entry; attaching an arbitrary dict would incorrectly route it through the
normal `_LaunchProcOutcome` decoder.

### 2. Recover and bound the failed proc's diagnostic context off-thread

Update `src/sase/ace/tui/actions/agent_workflow/_launch_procs.py` at the existing
payloadless failure path:

- Thread the popped `submitted_prompt` through
  `_schedule_payloadless_launch_failure_log()` into `_log_payloadless_launch_failure()`
  and pass it as `prompt_preview`.
- Prefer `completion.output` when present. When it is empty, call
  `sase.procs.read_proc_log_tail()` with the completion's durable proc ID and
  authoritative `log_path`. This read belongs inside the existing `asyncio.to_thread()`
  logging body, after terminal settlement has closed/flushed the supervisor pipe, not in
  `_on_launch_proc_complete()` or the observer poll loop.
- Use a named failed-launch diagnostic limit (200 tail lines) and then enforce a
  character ceiling (64 KiB) while retaining the newest text and adding an explicit
  truncation marker. Apply the same character bound to preloaded `completion.output`, so
  the launch-failure files remain bounded regardless of how the text arrived.
- Add nonempty proc context that is already present on `ObservedProc`: `status`,
  `phase`, `exit_code`, `command`, `log_path`, and the proc's terminal message. Keep
  `proc_id` and `proc_type`; normalize empty strings to `None` so blank fields such as
  the current `project_file:` line are omitted.
- Continue choosing the top-level failure exception text from `completion.error`, then
  `completion.message`, then the existing fallback. With step 1, handled launch failures
  will use the causal typed message; the captured output remains the crash/traceback
  fallback and deeper context.

Do not move the log write onto the UI thread. Preserve the tracked task set and task
cleanup used by `_launch_failure_log_tasks`, and keep the synchronous no-running-loop
branch for non-TUI/test callers.

### 3. Make multiline launch context readable in the Logs tab

Update `src/sase/logs/launch_log.py`'s human-block formatter without changing the JSONL
field structure or error-anchor header:

- Keep scalar context as the existing `key: value` rows.
- Render multiline values, especially `output`, under a section label with every payload
  line indented. Use a clear human label such as `process output` while retaining the
  stable JSONL key `output`.
- Preserve the logger's never-raise guarantee and existing exception traceback section.
  The captured process-output section supplements that synthetic logging traceback; it
  does not replace or overload the `traceback` JSONL field.

The existing Logs pane already reads, styles, focuses, and scrolls this human log on a
background worker. No changes should be needed in `logs_pane.py` or
`logs_pane_render.py`; add a renderer test only if implementation evidence finds a
formatting assumption that must be adjusted.

## Tests

Add focused regression coverage at the existing boundaries:

- In `tests/test_launch_query_feedback.py` (and the existing partial-launch suite if
  that is the closer fixture), assert that a normal `RuntimeError`, a partial
  multi-prompt failure, and a representative non-`RuntimeError` launch exception all
  publish a failed typed result carrying the useful cause. Assert that the unexpected
  exception path still raises so its traceback reaches combined output.
- In `tests/ace/tui/test_launch_failure_logging.py`, cover an empty `completion.output`
  backed by a real or mocked proc log. Assert that the JSONL and human entry contain the
  recovered causal output, the prompt preview, proc status, exit code, command/log
  locator, and the caller-provided error ID. Also cover the nonempty-output fast path so
  it does not unnecessarily reread the proc log.
- In `tests/ace/tui/test_failed_launch_stash.py`, extend the existing async failure test
  to prove the same submitted prompt is both consumed once for recovery and forwarded to
  failure logging, without changing stash behavior or blocking the event loop.
- In `tests/logs/test_launch_log.py`, pin multiline indentation/human labels, JSONL
  field stability, output truncation behavior at the chosen adapter boundary, and the
  logger's never-raise/error-anchor guarantees.
- Keep the existing `tests/ace/tui/test_registered_error_toasts.py`, focused Logs-pane
  tests, and error-registry tests passing to prove the toast-to-`,L` navigation contract
  is unchanged.

## Verification

Run installation first because the workspace environment may be stale, then exercise the
narrow suites and the repository gate:

```bash
just install
pytest tests/test_launch_query_feedback.py \
  tests/ace/tui/test_launch_failure_logging.py \
  tests/ace/tui/test_failed_launch_stash.py \
  tests/ace/tui/test_registered_error_toasts.py \
  tests/logs/test_launch_log.py
just check
```

If implementation changes a visual fixture or Logs-pane rendering code after all, also
run `just test-visual` and inspect/accept only the intentional Logs snapshot delta. The
expected implementation does not require a visual golden update.

## Acceptance criteria

- Reproducing the reported XPrompt argument failure and pressing `,L` opens the same
  anchored launch-failure entry, whose top-level error names the `XPromptArgumentError`
  rather than only `exited with code 1`.
- That entry contains a readable, indented tail with the invalid argument and causal
  traceback, a 200-character-or-shorter prompt preview, proc ID/status/exit code,
  command, and full-log locator.
- A launch child that fails before writing a typed result still gets useful bounded
  output through the proc-log fallback; missing/unreadable proc logs degrade to the best
  available completion message without masking the original failure.
- No launch diagnostic copies more than 200 lines or 64 KiB of proc output, and the
  retained text clearly says when older content was truncated.
- Launch completion and `,L` handling perform no new synchronous disk I/O on the Textual
  event loop.
- Existing successful launch behavior, failed-prompt recovery, toast wording, configured
  leader-key display, and error-anchor focus behavior remain unchanged.
- `just check` passes.

## Rollback and compatibility

The change is additive to existing JSONL records and human logs: older entries remain
readable, and consumers that ignore new context keys continue to work. If recovery
causes an unforeseen issue, the proc-log fallback and extra metadata can be removed
without reverting typed failure emission or changing the `,L` navigation contract. This
is a corrective diagnostics change intended to ship complete, so it does not need a
feature flag or a persistent user configuration choice.
