---
tier: tale
goal: Slow-tool call reports truncate oversized command output from the beginning,
  clearly mark the omission, and preserve at least the final 50 lines across transcript-backed
  and preview-only runtimes.
create_time: 2026-07-15 13:48:14
status: wip
prompt: 202607/prompts/tail_preserving_tool_call_output.md
---

# Plan: Preserve command-output tails in slow-tool reports

## Context

The Agents tab registers deferred Markdown report hints for completed slow tool calls and materializes a selected report
when the user presses `v` and chooses its hint. A report currently has two possible output sources:

- `Recorded Output` renders normalized `*_preview` fields from `tool_calls.jsonl`. The shared provider normalizer limits
  these strings to a 512-character prefix. This is the only available source for runtimes such as the Codex example
  whose tool-call record has no transcript path, so the report keeps the start of a long test run and loses the final
  failure summary.
- `Full Output (transcript)` best-effort recovers matching result text from a transcript. Recovery caps output at 64 KiB
  by retaining the prefix, and it stops collecting matching text after roughly twice that budget. It can therefore omit
  both the end of a large result and later matching result fragments.

Both paths are head-biased even though the end of command output normally contains the actionable failure, summary, exit
status, or final successful result. The report writer is also invoked synchronously from the Textual view-input handler
even though recovery may stat, scan, and JSON-decode a transcript as large as 16 MiB; project TUI performance rules
require this work to run off the event loop.

This is a `tale`: the provider-neutral capture contract, report formatting, view action, documentation, and tests form
one atomic behavior change and do not need independently landable phases or multiple coding agents.

## Desired behavior

- Treat the existing character limits as soft output budgets. When output exceeds a budget, retain a suffix rather than
  a prefix and move an explicit omission marker to the beginning of the displayed text.
- Preserve at least the final 50 logical lines of command output. If the nominal character budget begins inside those
  lines, expand the retained suffix back to the start of the fiftieth line. Normalize CRLF/CR before counting lines,
  preserve the original final-newline behavior, and avoid off-by-one changes for output that is already within budget.
- Make the marker unambiguous about direction and size, for example by reporting that characters (and, when cheaply
  known, lines) were truncated from the beginning. The marker must be outside the retained command text semantically but
  remain safe inside the existing Markdown fence.
- Keep non-command previews head-oriented: command input, paths, errors, web/read content, and subagent final messages
  have different semantics and should not silently change. Preserve `SASE_TOOL_LOG_FULL=1` as the explicit raw logging
  escape hatch.
- Apply the behavior uniformly through shared normalization rather than adding Codex-, Claude-, Qwen-, or agy-specific
  branches.
- Do not mutate raw provider transcripts or source artifacts. Previously recorded head-truncated previews cannot have
  their missing tails reconstructed; continue to render them with an honest recorder-truncation note. New reports over
  new artifacts get the tail guarantee, while old transcript-backed artifacts benefit from tail-oriented recovery.

## Design

### 1. Introduce one tail-oriented command-output truncation policy

Add a small shared helper with explicit constants for the minimum retained line count and the applicable character
budget. The helper should return unchanged normalized text when no truncation is necessary; otherwise it should choose
the earlier of the character-budget boundary and the start of the final 50 lines, prepend a directional omission marker,
and return the retained suffix. Keep the policy deterministic and independently unit-testable, including empty output,
trailing newlines, exactly-at-limit values, fewer/exactly/more than 50 lines, and a fiftieth line that pushes the result
beyond the nominal character budget.

Use this policy only for textual command-result fields (`stdout`, `stderr`, and combined `output`) in
`src/sase/llm_provider/_tool_call_common.py`. Keep the normalized `stdout_preview`, `stderr_preview`, and
`output_preview` field names so the artifact schema, readers, pair-collapsing logic, and report renderer remain backward
compatible. This makes future preview-only records retain the end of a command without globally expanding or reversing
metadata and content previews. Unknown tool-result envelopes should use the command-output policy only when the field
itself is one of these output streams; content/result/subagent fields retain their current policy.

Document that command-output summaries remain bounded by a soft character budget that may grow enough to include 50
complete trailing lines. Call out the deliberate size tradeoff: typical records stay compact, while unusually wide
trailing lines may make a summary larger than the nominal budget to honor the diagnostic-line guarantee.

### 2. Tail-cap transcript recovery without losing later fragments

Update `src/sase/ace/tui/tools/report.py` so recovered output uses the same suffix semantics at its existing 64 KiB
report budget. Recovery must inspect all eligible mappings in every transcript line within the existing 16 MiB scan cap;
remove the prefix-oriented early stop that prevents later matching fragments from being considered. Preserve source
order and existing duplicate suppression, but accumulate or compact data so retained state is bounded to the useful tail
plus the minimum-line guarantee rather than concatenating arbitrary output indefinitely.

When recovery truncates, put the omission marker before the retained text and update the recovery note to say the
beginning was omitted while the output tail was preserved. Keep all current degradation messages for a missing tool use
ID, unavailable/missing/oversized/unreadable transcript, or no matching result. Continue using the transcript scan cap
as a safety boundary; this change is about the selected output direction, not unbounded file scanning.

Defensively apply the report-tail formatter to textual recorded-output values before fencing them. New normalized
command previews should already satisfy the policy, but this protects reports built from full/debug, legacy, or
externally produced records containing an oversized `*_preview`. Do not reinterpret an existing marker that says the end
was truncated: the report can identify that historical limitation but cannot manufacture the missing suffix.

### 3. Keep report materialization off the Textual event loop

Adapt the hint-view flow in `src/sase/ace/tui/actions/hints/_processing.py` so deferred report writes—including
transcript stat/read/JSON work, atomic write/fsync, and report pruning—run through the established off-thread mechanism
before the selected pager, editor, artifact viewer, or clipboard action is dispatched. Preserve multi-selection order,
ordinary file hints, commit hints, per-report failure notifications, and the rule that failed materializations are
removed from the destination list.

The interactive operation is bounded and directly gates opening the requested view, so it does not need to become a
long-running tracked task. Any async boundary must re-check the relevant UI/application state before applying UI
effects, while the destination intent and selected file list remain the immutable request that the user submitted.

### 4. Preserve report structure and compatibility

Keep deterministic report paths, Markdown section ordering, fenced-output safety, atomic replacement, retention of the
newest 50 reports, successful/failed call eligibility, and subagent report behavior unchanged. No keymap or default
configuration changes are needed. This is presentation/capture glue local to Python, not shared backend domain behavior,
so it does not cross the Rust core boundary.

## Test plan

1. Add focused unit coverage for the tail helper:
   - short and exactly-at-budget text is byte-for-byte unchanged after newline normalization;
   - oversized output loses a recognizable prefix, retains a recognizable final sentinel, and starts with a
     directionally correct omission marker;
   - the complete final 50 lines survive even when they exceed the nominal character budget;
   - 49, 50, and 51-line cases plus trailing/no-trailing newline and CRLF inputs establish line-count boundaries;
   - repeated application does not double-wrap an already tail-truncated value if that path can occur.
2. Extend shared provider and runtime fixture tests so long Bash/command output produces tail-oriented
   `stdout_preview`/`stderr_preview`/`output_preview` records. Include the Codex preview-only shape from the supplied
   example and at least one structured/hook path to prove the shared implementation is runtime-neutral. Assert that
   command input, ordinary content previews, and subagent `content_full` retain their existing prefix behavior.
3. Extend `tests/ace/tui/tools/test_report.py` with transcript results larger than 64 KiB and more than 50 lines. Assert
   the early sentinel is absent, the last 50 complete lines and final sentinel are present, the marker precedes them,
   and the note describes beginning truncation. Add a transcript with an early large matching fragment and a later
   matching fragment to ensure recovery no longer stops before the true tail.
4. Cover recorded-output fallback for no-transcript entries, including a future tail-oriented preview and a historical
   head-truncation marker. Verify the former exposes the final failure lines and the latter remains clearly identified
   as irrecoverable rather than being mislabeled as tail-complete.
5. Update hint-action tests for asynchronous/off-thread materialization across pager and editor routing, multiple mixed
   files, and write failure. Add a regression assertion that transcript recovery/report writing is not executed on the
   Textual event-loop thread.
6. Run the focused provider-normalization, tool-report, and hint-view suites. Then run `just install` for the ephemeral
   workspace and `just check`, including the visual snapshot suite, to catch cross-runtime, formatting, typing, and TUI
   regressions.

## Risks and mitigations

- **Artifact growth:** retaining 50 complete trailing lines can exceed the nominal character budget when lines are
  unusually wide. Keep the expansion isolated to command-output fields, document the soft cap, and retain all existing
  transcript/file scan caps.
- **Historical incompleteness:** old preview-only artifacts already discarded their tails. Make this limitation explicit
  in the report and test it; the guarantee applies prospectively unless a historical transcript can recover the output.
- **Mixed stdout/stderr ordering:** structured providers may expose the streams separately. Preserve each field's
  current section and ordering rather than inventing an interleaving that the source data cannot support; apply the
  final-50-line guarantee independently to each available command-output stream.
- **Duplicate transcript fields:** nested provider envelopes can expose the same text more than once. Retain existing
  deduplication and source ordering while changing only the bounded accumulator/truncation direction.
- **Async view races:** materialization may finish after the user navigates elsewhere. Treat the submitted selection as
  the view request, re-check app state before notifications/viewer dispatch, and avoid refreshing or moving Agents-tab
  selection as a side effect.

## Expected outcome

For newly captured long commands, selecting a slow-tool hint from the Agents tab opens a compact report whose omission
marker appears before the output and whose visible suffix includes at least the command's final 50 lines. The supplied
Codex-style no-transcript case will show the end of `just check`—including the actual pytest failure summary—instead of
ending near pytest startup. Transcript-backed reports will likewise retain their true final output even when later
matching fragments occur after an initially large result. Short output and non-command report content remain unchanged,
report generation no longer performs transcript I/O/JSON parsing on the Textual event loop, and raw source artifacts
remain available for deeper investigation.
