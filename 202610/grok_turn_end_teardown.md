---
tier: tale
title: Reap grok providers that hang after their turn ends
goal:
  A grok run that ends its turn without a declaration is reaped and completes instead of
  stalling.
size: medium
decisions:
  recover_final_reply:
    ask:
      Recover grok's final message from its session chat history after a turn-end reap?
    choices:
      recover: The reaped turn's final summary reaches done.json and the saved chat
      skip:
        The reaped turn ends at its last streamed narration; the final summary stays
        lost
    default: recover
    why:
      grok only flushes its final message at session end, which is the step that hangs
    answer: recover
proposed_by: bbugyi200.apollo.9r
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 17:21:08
status: wip
---

# Reap grok providers that hang after their turn ends without a declaration

## Problem

The `9a` agent session (project `gh_bobs-org__bob-cli`) has been stuck for about five
hours. Its `9a--code` member (grok-4.6) finished its work, landed its stitch on
`origin/master`, and ran `sase final context -f json`. Because the commit requirement
was `not_triggered`, that call returned `submission_required: false`. As `sase_final`
step 2 instructs, the agent then ended its turn without submitting a declaration. Its
reasoning reads: "if `submission_required` is false, the process should stop and return
immediately."

Grok then finished the turn but never exited:

- `~/.grok/sessions/<quoted cwd>/<session id>/events.jsonl` ends with
  `{"type":"turn_ended","outcome":"completed"}` at 16:21:25Z.
- `~/.grok/logs/unified.jsonl` shows `shell.handle_prompt.done ok=true` for that pid,
  but never shows the `session_end.worker_join` line that every other grok run logs. The
  process is idle in `futex_wait`: no children, no pending stdout bytes, 32s of total
  CPU.
- The runner (`run_agent_runner.py`) is still blocked reading grok's stdout.

This is a recurring grok 1.0.62 teardown hang, not a one-off. On 2026-10-20 alone, at
least two other grok runs hung at the same point. Since 2026-10-10, eight grok runs have
written a `provider_teardown_stall.json`. Those runs were reaped only because they had
already submitted a declaration. The completion watchdog in
`src/sase/llm_provider/_subprocess_plain.py` (`start_completion_watchdog`) arms only
when a new `final_submission.json` appears. `docs/contribute/llm-invocation.md`
("Provider Teardown Watchdog") says: "A provider that wedges _before_ submitting its
declaration is not covered." A legitimate no-submission turn is exactly that case, so it
is never reaped.

A second, related loss: grok writes its final assistant message to stdout only during
session end. When teardown hangs, the final message never reaches sase. This happened in
the 9a run and in the reaped run
`~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202610/20/20261020155300`,
whose `done.json` `step_output._raw` stops at intermediate narration. The message is
still persisted in grok's `chat_history.jsonl`.

The fix follows the decision `decisions:adapters-normalize-harnesses`: the grok adapter
detects its harness's turn end and makes the CLI conform to the single-turn contract
with a bounded guard. The host does not model grok's private wait semantics.

## Changes

### 1. Generic: let the completion watchdog arm on a provider-reported turn end

In `src/sase/llm_provider/_subprocess_plain.py`:

- Add an optional keyword parameter to `start_completion_watchdog`, for example
  `turn_end_probe: Callable[[], ProviderTurnState | None] | None = None`. On each poll
  the probe reports the provider's latest turn-lifecycle state: ended (with an outcome
  string), started/running, or unknown/`None`.
- Keep the existing declaration trigger unchanged. Add a second, independent trigger:
  when the probe reports "ended", start the grace period. If the probe later reports a
  new turn started, disarm it. grok sometimes re-enters a turn about 40ms after
  `turn_ended`; see "Grok probe" below. When either trigger's grace expires while the
  process is alive, call the existing `_tear_down_stalled_provider`.
- A probe that raises must not kill the watchdog thread. Treat it as "unknown" and keep
  polling, so the declaration trigger still works.
- Reuse `SASE_PROVIDER_TEARDOWN_GRACE_SECONDS` (default 120s, `0` disables) for both
  triggers. Healthy grok runs exit within about 0.4s of `turn_ended`.

In `src/sase/llm_provider/_subprocess_reap.py` (`TeardownStall`):

- Add `trigger: Literal["declaration", "provider_turn_end"]` and
  `turn_outcome: str | None`, and serialize both in `to_json()`. Keep the existing
  `provider_teardown_stall.json` keys, which have no other in-repo consumers. Set
  `declared_at` to the moment the grace was armed (a declaration or a turn end) and
  document that meaning, or rename it to `armed_at` and keep `declared_at` as an alias.
  Choose one and test it.
- Make `note` trigger-aware. The provider-turn-end wording should say that the provider
  reported its turn ended (no declaration) and was still running N s later.

In `src/sase/llm_provider/_subprocess_stream.py` (`settle_teardown_stall`):

- Keep returning `0` for `trigger == "declaration"`, and for `provider_turn_end` when
  `turn_outcome == "completed"`. For any other outcome (grok writes `error` and
  `cancelled`), keep the provider's own non-zero return code and still append the note,
  so the normal failure and retry path runs. Update the docstring, which currently says
  the watchdog only fires after an accepted declaration.

Other adapters keep calling `start_completion_watchdog(process, runtime=...)` with no
probe, so their behavior is unchanged.

### 2. Grok probe

In `src/sase/llm_provider/grok.py`:

- `_invoke_loop` already creates `session_id` and passes `--session-id` and
  `--cwd os.getcwd()`. Pass both through to `_run_subprocess`. It can take them as
  kwargs, or `_run_subprocess` can receive a prebuilt probe.
- Resolve the session directory as
  `<grok_home>/sessions/<quote(cwd, safe="")>/<session_id>/`, where `grok_home` is
  `$GROK_HOME` if set and `~/.grok` otherwise. If the exact directory is missing, fall
  back to globbing `<grok_home>/sessions/*/<session_id>` (session ids are fresh UUIDs).
  Observed encoding: `%2Fhome%2Fbryan%2F.local%2Fstate%2F...`, which is
  `urllib.parse.quote(path, safe="")`.
- Implement the probe incrementally. Keep a byte offset into `events.jsonl` and read
  only appended bytes on each poll. Ignore a trailing partial line until it completes.
  Track the last `turn_started`/`turn_ended` event, where `turn_ended` carries
  `outcome`. A missing file or directory means "unknown". Never raise out of the probe.
- Real event shapes seen in recent sessions:
  - `{"type":"turn_started","session_id":...,"turn_number":0,...}`
  - `{"type":"turn_ended","outcome":"completed"}`
  - `{"type":"turn_ended","outcome":"cancelled","cancellation_context":{"reason":"max_turns_reached","limit":1}}`
  - `{"type":"turn_ended","outcome":"error"}`
  - Some sase sessions contain 2–3 `turn_ended`/`turn_started` pairs. The probe must
    report the latest state, which is why a later `turn_started` disarms.
- Call `start_completion_watchdog(process, runtime="grok", turn_end_probe=probe)`.

> [!decision] recover_final_reply = recover After
> `stream_and_parse_messages_json_output` returns, check `teardown_stall_for(process)`.
> If the stall's trigger is `provider_turn_end`, read `chat_history.jsonl` from the same
> session directory. Take the last `{"type":"assistant"}` entry that comes after the
> last `tool_result` entry. If its `content` is not already the tail of the streamed
> `content`, append it (and its text to `live_reply.md` when that file is available), so
> `InvokeResult.content` and the saved chat get grok's final summary. Best-effort: any
> read or parse failure leaves the streamed content unchanged. For the 9a session, this
> recovers "Newly added references now reset Bob Refs onto that row ...".

> [!decision] recover_final_reply = skip Do only the probe and reap. Leave
> `InvokeResult.content` as streamed.

### 3. Docs

Update the "Provider Teardown Watchdog" section of `docs/contribute/llm-invocation.md`:

- Describe both triggers.
- Replace "A provider that wedges _before_ submitting its declaration is not covered"
  with the precise remaining gap: a provider that wedges without a declaration and gives
  no turn-end signal. Today that is every adapter except grok.
- Note the outcome rule (non-`completed` turn ends keep a failure exit).
- Note the grok final-reply recovery, if that decision is chosen.

Read the `docs_writing` reference memory before editing `docs/`.

## Tests

- `tests/llm_provider/test_completion_watchdog.py`:
  - A probe that reports "ended" arms the grace and reaps a still-alive dummy process.
    The stall has `trigger == "provider_turn_end"` and the outcome.
  - A later "started" disarms, so there is no reap.
  - A raising probe neither kills the thread nor blocks the declaration trigger.
  - `0` grace disables both triggers.
  - The existing declaration-trigger tests still pass.
- `tests/llm_provider/test_completion_watchdog_streams.py`: `settle_teardown_stall`
  returns `0` for declaration and `completed` turn-end stalls. It keeps the non-zero
  code for `error`/`cancelled`, with the note appended to stderr in both cases.
- `tests/llm_provider/test_subprocess_reap.py`: `to_json()` and `note` cover both
  triggers.
- Grok tests (`test_grok_provider_stream.py` / `test_grok_provider_invocation.py`, using
  `_grok_provider_core_helpers.py`), with a temporary `GROK_HOME`:
  - The session-dir resolution, including the glob fallback.
  - Incremental parsing across partial lines and multiple turn pairs.
  - `start_completion_watchdog` receives a probe.
  - If `recover_final_reply = recover`: chat-history recovery appends the final
    assistant entry, skips it when already streamed, and tolerates missing or garbled
    files.

Run `just check` through `sase tool run` as the lint/test memory requires. Do not run
`just check-full`.

## Out of scope

- The currently stuck `9a--code` process. Its runner loaded the old code, so this change
  cannot rescue it. The user handles it separately: `kill 132632` (the hung grok child,
  not runner pid 105143) unblocks the runner. Without a registered stall, the runner
  sees the SIGTERM exit as a provider failure and will likely mark `9a--code` failed.
  Its work is already safe: the `feat(refs)` stitch is on bob-mac-capture
  `origin/master`.
- A provider-agnostic heuristic, such as arming on `final_context.json` with
  `submission_required: false` plus stdout silence. Only grok has shown this failure
  mode (`decisions:corpus-before-mechanism`). Other adapters can supply a probe later.
- Changing `sase_final`'s "stop and return when `submission_required` is false" rule.
  The agent followed it correctly.
- The grok CLI teardown hang itself, which is upstream.
