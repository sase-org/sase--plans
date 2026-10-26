---
tier: epic
title: Stream Muse commentary into the live Reply card
goal: "While a Muse agent runs, each commentary message Muse commits to its session log
  appears in live_reply.md, and so in the main deck's Reply card, within about a second.
  The returned reply, token usage, final-answer streaming, and host finalizers are
  unchanged, and tests exercise the real Muse 1.4.13 stream shape so this gap cannot
  hide again.

  "
phases:
  - id: commentary-tail
    title: Commentary tail in the Muse adapter
    depends_on: []
    size: medium
    description:
      "commentary-tail: add a fail-soft session-log tail driven by a new on_tick hook in
      stream_json_lines, write each root-run commentary message as its own live_reply.md
      chunk, record muse_session_id at launch, add the terminal-text fallback, cover it
      with unit and temporal tests, and update the Muse provider docs."
  - id: mounted-fixture
    title: Production-shaped mounted TUI fixture
    depends_on:
      - commentary-tail
    size: medium
    description:
      "mounted-fixture: rewrite gated_muse_script so stdout carries no mid-run deltas
      and commentary arrives only through a session log, then update the four mounted
      live-reply follow tests to assert commentary reaches the Reply card before the
      terminal event."
  - id: live-capture
    title: Muse 1.4.13 capture and end-to-end check
    depends_on:
      - commentary-tail
    size: medium
    description:
      "live-capture: record sanitized 1.4.13 stdout and session-log fixtures from a real
      tool-using run, add a replay test through the real parser and tail, run one live
      end-to-end check through MuseProvider, and update the fixture-release note in the
      docs."
decisions:
  decision_record:
    ask:
      Record a decision that private provider logs may feed display-only surfaces, never
      control flow?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.2i.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-26 15:05:42
status: wip
---

- **PROMPT:**
  [prompts/202610/muse_commentary_live_reply.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/muse_commentary_live_reply.md)

# Plan: Stream Muse commentary into the live Reply card

## Context

The research report
`research:202610/muse_commentary_missing_from_exec_stream/muse_commentary_missing_from_exec_stream.md`
is the source for this plan. Read it with `sase artifact read <ref> "<why>"` before
starting any phase. Its findings, which this plan accepts in full:

- Since Muse `1.4.2`–`1.4.4`, Muse writes a short **commentary** message before most
  tool calls. It records each one in its private session log
  (`$XDG_DATA_HOME/muse/sessions/YYYY/MM/DD/<session-id>/session.jsonl`) as a
  `runtime.session` record whose `payload.event.kind` is `assistant_message_committed`
  and whose `payload.event.phase` is `"commentary"`.
- `muse exec --json` stdout never carries those messages. It carries tool lifecycle
  events all run long and `run.output.delta` fragments of the **final answer** only,
  after the last tool call. No current Muse release or flag changes this
  (`1.4.13-R6996.1` is current stable).
- `_stream_output_delta` in `src/sase/llm_provider/_subprocess_muse.py` is the only
  writer of `live_reply.md`, so the file stays at 0 bytes for a median 84% of each Muse
  run. The Reply card is therefore correct to show `Waiting for agent response...`. The
  stream loop, the TUI follow, and the render path all work. The earlier fixes
  (`2307212bcd`, `392983091d`, `8150a76d46`) made the TUI follow an empty file.
- The test fixture `gated_muse_script()` emits `run.output.delta` mid-run, which real
  Muse never does. That is why the tests missed this.

Example commentary record from a local 1.4.13 log (compact JSON, one line, abridged):

```json
{
  "schema_version": 1,
  "id": "01a1…",
  "stream": { "kind": "session", "id": "<session-id>" },
  "sequence": 53,
  "recorded_at": 1792463279757454,
  "record_type": "event",
  "durability": "durable",
  "payload_type": "runtime.session",
  "payload_schema_version": 1,
  "payload": {
    "kind": "run",
    "run_id": "01a1…",
    "event": {
      "kind": "assistant_message_committed",
      "message_id": "7268…",
      "phase": "commentary",
      "provider_item_id": "rs_…",
      "response_id": "resp_…",
      "text": "Approved — implementing your docs theme plan now."
    }
  }
}
```

Facts about the log that the design relies on, checked against local 1.4.13 logs:

- Logs range from about 0.4 MB to 24 MB per run, mostly tool output, so the tail must
  only ever read new bytes.
- In every log checked, all commentary records are top-level lines with this session's
  `stream.id` and one `run_id`.
- The first line of each log is a `retained_frame` container. Its
  `children[].record_json` values are **JSON-encoded strings**, so inside a child the
  quotes are escaped (`\"phase\":\"commentary\"`). A byte prefilter that requires the
  literal `"phase":"commentary"` would skip children. The prefilter below avoids that.
- The final answer is also committed as `assistant_message_committed`, but **without** a
  `phase`. Stdout deltas already stream it.

Constraints that apply to every phase:

- This is provider-adapter glue in Python. The `live_reply.md` +
  `live_reply_timestamps.jsonl` artifact contract does not change. That means **no TUI
  change and no `sase_core` change**. Provider stream parsing does not live in
  `sase-core`, and every frontend already reads the artifact.
- The tail is **display-only and fail-soft**. Commentary never enters
  `InvokeResult.content`, `terminal_texts`, `streamed_texts`, delta salvage, the
  wait-claim guard, or anything host finalizers see. SASE must never pass
  `--no-session-log`.
- This does not conflict with `decisions:adapters-normalize-harnesses`. That decision
  rejected reading the private session log for **wait semantics** (control flow). This
  tail only adds display text, and SASE already reads the same log for token usage.
- No feature flag. Phase `commentary-tail` lands a complete, user-ready fix. The other
  two phases only add tests, fixtures, and evidence.
- Verify every phase with `sase tool run check`. Read `lint_and_test.md` with
  `/sase_memory_read` first. Do not run `just check-full`.

## Phase `commentary-tail`: commentary tail in the Muse adapter

Read `docs_writing.md` with `/sase_memory_read` before editing docs.

### 1. New module `src/sase/llm_provider/_muse_commentary_tail.py`

A small class, for example `MuseCommentaryTail`, built with:

- the session id;
- an `on_commentary(text: str)` callback;
- optional `sessions_root` and clock overrides, for tests.

It holds the session id, the open log handle and byte offset, a partial-line buffer, a
set of seen `message_id`s, and a disabled flag.

**Locate.** Call the public `find_muse_session_log(session_id)` from
`_muse_session_usage.py` until it returns a path. Rate-limit this to about once a
second, because it globs every date directory. The existing glob already handles runs
that cross midnight. Once found, keep the file open in binary mode.

**Read only new bytes.** Never rescan from the start.

- A normal poll reads at most a bounded amount per call (about 1 MiB), so a burst of
  tool output cannot stall the stdout loop. Later polls read the rest.
- A forced or final drain reads to EOF.
- Keep a trailing unterminated line in the buffer until its newline arrives.
- If one buffered line exceeds a size cap (about 4 MiB), drop it and skip to the next
  newline. Commentary records are a few hundred bytes, so this only bounds memory on
  giant tool-output lines.

**Prefilter before parsing.** Only `json.loads` a line when it contains both
`b"assistant_message_committed"` and `b"commentary"`. That matches top-level records and
the escaped `retained_frame` children alike.

**Accept a record only when all of these hold:**

- `payload_type == "runtime.session"` (reuse `MUSE_SESSION_PAYLOAD_TYPE`);
- `payload.kind == "run"`;
- `payload.event.kind == "assistant_message_committed"`;
- `payload.event.phase == "commentary"`;
- `stream.id` equals the session id;
- `text` is a non-empty string after stripping surrounding whitespace;
- its `message_id` has not been seen. A record without a `message_id` is still accepted.

If a line is a `retained_frame` container, apply the same filter to each child's
`record_json`, decoded as JSON. Never emit the unphased final message or
`reasoning_summary_delta` records.

**Public surface:**

- a rate-limited `tick()` (poll at most every ~0.25 s; locate at most every ~1 s);
- a forced `drain()`;
- a final `finish()` that drains to EOF and closes the handle.

None of them may raise. Keep every record-vocabulary string in one constant block at the
top of the module, as `_subprocess_muse.py` does.

**Fail soft.** Record at most **one** diagnostic per tail instance. Use
`append_tool_call_collector_diagnostic(artifacts_dir, reason="muse_commentary_tail_unavailable", extra={...})`
with `session_id`, a `cause`, and an `error` where one exists. The causes:

- `session_log_not_found`: only reported from `finish()`, if the log never appeared;
- `read_error`: an `OSError`, after which the tail disables itself;
- `malformed_record`: a line passed the prefilter but would not decode;
- `unexpected_error`: any other exception, after which the tail disables itself.

At worst the card looks exactly as it does today.

### 2. `on_tick` hook in `stream_json_lines`

In `src/sase/llm_provider/_subprocess_stream.py`, add a keyword-only
`on_tick: Callable[[], None] | None = None`. Call it once per pass of the main loop,
after the select/read block. The loop already wakes at least every 0.1 s even when
stdout is quiet, so polling happens without a thread or a lock, and commentary and delta
writes stay deterministically ordered. Document the hook in the docstring. Other
providers pass nothing and do not change.

### 3. Wiring in `src/sase/llm_provider/_subprocess_muse.py`

- **Record the session id at launch.** At the start of
  `stream_and_parse_muse_json_output`, when `session_id` is set, write
  `{"muse_session_id": session_id, "runtime": "muse"}` with
  `write_run_metadata_artifact`, so other tools can find the live log mid-run. The
  post-exit write stays and merges as it does now.
- **Create the tail.** Create it only when `session_id` is set and
  `state.live_reply_file` is open. Store it on `_MuseStreamState` and pass its `tick` as
  `on_tick`.
- **Final drain.** After `stream_json_lines` returns normally, call the tail's
  `finish()` **before** the `finally` block closes the reply files. Do not drain when
  the stream raised.
- **Write commentary.** Add `_append_commentary(text, state)`:
  1. call `_close_delta_chunk(state)`;
  2. call `append_stream_delta(text, …, new_chunk=True)`;
  3. end that chunk too, so the console line ends and the next delta of any stream opens
     a fresh chunk. One way is to set `open_stream_key` to a private commentary
     sentinel, then call `_close_delta_chunk`.

  The result is one timestamp divider per message, the same shape Claude and Codex
  produce. Do not touch `terminal_texts` or `streamed_texts`.

- **Order commentary before the final answer.** In `_stream_output_delta`, when a delta
  is about to open a new chunk, call the tail's forced `drain()` first. Then compute
  `new_chunk` from the state as it stands after the drain. This way commentary committed
  before the final answer always lands above it.
- **Terminal fallback (hardening).** In `_capture_run_terminal`, when the terminal
  carries text and no delta was streamed for that run-stream key (the key is not in
  `state.streamed_texts`), append the terminal text to `live_reply.md` as its own closed
  chunk. Do not add it to `streamed_texts`. The returned content is unchanged. This has
  never been observed; it only guards against a future build that drops deltas.
- Update the module docstring's parser rules to describe the commentary source and the
  terminal fallback.

### 4. Tests

Add these under `tests/llm_provider/`. Build session-log lines in the real 1.4.13 shape
shown above.

**Unit tests for the tail**, for example `test_muse_commentary_tail.py`. Feed it an
excerpt containing:

- commentary records;
- an unphased final message;
- a `retained_frame` line whose child is a commentary record with an escaped
  `record_json`;
- marker and `reasoning_summary_delta` records;
- a duplicate `message_id`;
- a foreign `stream.id`;
- a partial last line that a later append completes.

Assert that only the commentary comes out, in order, once each. Then cover these cases:

- **Appends between ticks:** they are picked up with no rescan.
- **Late log:** the log appears after several ticks and is still found.
- **Missing log:** exactly one diagnostic from `finish()`, and no exception.
- **Read error and malformed line:** one diagnostic each, and no exception.
- **Bounded reads:** commentary behind a large tool-output line still arrives across
  several ticks.

**Stream-level tests**, for example `test_muse_commentary_stream.py`, using a session
log installed under a temporary `XDG_DATA_HOME`:

- Replay stdout made of `task.lifecycle.*` events, then final-answer deltas, then
  `run.terminal.completed`. Assert:
  - `live_reply.md` holds the commentary chunks in order, then the final answer once;
  - `live_reply_timestamps.jsonl` has one row per chunk;
  - the returned content equals the terminal text exactly.
- With no `session_id`, the output is byte-identical to today's.
- **Terminal fallback:** a terminal with text and no deltas writes that text exactly
  once. With deltas present, nothing extra is written.
- **Temporal integration test.** A fake `muse` process writes commentary into the
  session log, puts only `task.lifecycle.*` on stdout, then blocks on a handshake file.
  Run the parser in a thread. Before releasing the handshake, assert that
  `live_reply.md` already holds the commentary and that `run_metadata.json` already
  holds `muse_session_id`. After release, assert that the final deltas follow, the final
  answer is not duplicated, and the returned content equals the terminal text alone. Put
  the fake script builder in a small shared helper module so phase `mounted-fixture` can
  reuse it.
- **`on_tick`:** a `stream_json_lines` test showing the hook fires while stdout is quiet
  and the child is still alive.

Existing Muse stream, artifact, wait-guard, and mounted tests must keep passing
unchanged. Phase `mounted-fixture` owns making the mounted fixture realistic.

### 5. Docs

- **`docs/reference/providers/muse.md`:**
  - In **The Event Stream**, add a rule that live commentary comes from the session log
    SASE named, not from stdout. Cover the record filter, one chunk per message, the
    display-only guarantee (never part of the returned reply), and the
    `muse_commentary_tail_unavailable` diagnostic.
  - Revise rule 2 so it no longer implies stdout deltas are the only live-reply source.
  - Revise the terminal-text sentence to describe the fallback.
  - Extend the `--session-id` bullet: the id now also locates live commentary.
  - Extend **Token Usage and Model Identity**: the same log is tailed live, and
    `muse_session_id` is recorded at launch.
- **`docs/contribute/llm-invocation.md`:**
  - Replace the claim that Muse "is usually quiet through tool work" (around the Live
    Reply File section) with the commentary behavior.
  - Add a module-table row for `_muse_commentary_tail.py`.
- Run `just fix` so the docs formatting and budget ratchets pass.

### 6. Decision record

> [!decision] decision_record

When accepted, use `/sase_memory_write` to add one strand to the `decisions` web, for
example `sase/memory/decisions/private-logs-display-only.md`, in the existing records'
format:

- **Claim:** a provider's private session log may feed display-only, fail-soft surfaces
  such as live commentary. It never feeds control flow, returned content, or completion.
- **Why:** the alternatives the research rejected or deferred: synthetic tool
  breadcrumbs, prompting Muse to narrate, migrating to `muse serve`/MSP now, and moving
  this to `sase_core`.
- **Cost:** one more consumer of a format that changes between releases.
- **Reopens when:** `muse exec --json` exports assistant message phase, or provider
  stream parsing moves to `sase_core`.

Link `[[decisions/adapters-normalize-harnesses]]`, then run `sase memory init`.

> [!decision] decision_record = no

Write no memory.

### 7. Follow-up to record

Record this as a `PROPOSED FOLLOW-UP:` note on this phase's bead:

- **Retire the commentary tail once `muse exec --json` exports message phase.** Parse
  the new event in `_process_muse_json_line`, dedupe by `message_id` against the tail
  during the transition, then delete the tail module and the `on_tick` call site.

## Phase `mounted-fixture`: production-shaped mounted TUI fixture

Read `tui.md` with `/sase_memory_read` first. The TUI itself does not change in this
phase. Only the test fixture and the assertions change.

- **Rewrite `gated_muse_script()`** in
  `tests/ace/tui/_live_reply_follow_mounted_helpers.py` to match real Muse 1.4.13. Reuse
  phase `commentary-tail`'s fake-Muse helper where it fits.
  - During tool phases, stdout emits only `task.lifecycle.*`, and optionally
    `run.model.configured`. It never emits `run.output.delta` mid-run.
  - The script creates the session log at
    `<XDG_DATA_HOME>/muse/sessions/YYYY/MM/DD/<session-id>/session.jsonl` itself, after
    it starts. It appends one commentary record per stage, flushing each.
  - Keep the gate-file choreography.
  - The last stage emits the final-answer deltas plus `run.terminal.completed` on
    stdout. It also commits the unphased final message to the log, to prove that message
    is not duplicated.
- **Update callers.** Change `exercise_gated_stream` and
  `tests/ace/tui/test_live_reply_follow_mounted_reply.py`, which inlines the same flow,
  to:
  - set `XDG_DATA_HOME` to a temp directory with `monkeypatch`;
  - pass `session_id=` to `stream_and_parse_muse_json_output`.

  Consolidate the reply test onto `exercise_gated_stream` if that is cleaner.

- **Assertions.** In all four mounted tests (single-agent reply, session current turn,
  workflow root, workflow step):
  - after the first gate, the Reply card shows commentary 1 with one divider, while the
    process is alive and before any terminal or `done.json`;
  - after the second gate, commentary 2 appears through the watcher route or the polling
    backstop, with two dividers;
  - after release, the final answer appears exactly once, below the commentary;
  - the parser returns only the terminal text.

  Keep the existing checks: no roster refresh (`refresh_calls == []`), the
  live-reply-source identity, and `old_text` appearing exactly once in the session test.

## Phase `live-capture`: Muse 1.4.13 capture and end-to-end check

- **Capture.** Use the installed `muse` and record `muse --version` (expect
  `1.4.13-R6996.1`). Name fixtures after the release actually captured.
  - Run one short, tool-using `muse exec --json` in a throwaway temp directory outside
    the repo. Use the argv shape documented in `docs/reference/providers/muse.md`, a
    SASE-generated `--session-id`, and a **non-Contributor** model such as
    `muse-spark-1.3`.
  - Use a prompt that runs two or three harmless shell commands and narrates between
    them. Save stdout and copy the session log.
- **Sanitize.** Drop encrypted reasoning and credentials. Replace home and absolute
  paths. Trim the session log to:
  - the first `retained_frame` line;
  - the commentary records;
  - the unphased final message;
  - the `model_completed` usage records;
  - a few representative neighbors, such as a tool-output record and a marker.

  Keep the original record order and byte shape. Commit them, for example as
  `tests/llm_provider/fixtures/muse_exec_commentary_R6996.1.jsonl` and
  `tests/llm_provider/fixtures/muse_session_log_commentary_R6996.1.jsonl`. Keep the
  existing R708.1/R3401.1 fixtures and their tests.

- **Replay test.** Install the session-log fixture under a temporary `XDG_DATA_HOME`,
  replay the stdout fixture through `stream_and_parse_muse_json_output` with that
  session id, and assert:
  - `live_reply.md` holds every captured commentary text, in order, as its own chunk,
    then the final answer once;
  - the returned content equals the terminal text;
  - usage matches the `model_completed` sums;
  - the tool-call records still parse from the 1.4.13 stdout.

  If replay shows real 1.4.13 tool-call drift, record a `PROPOSED FOLLOW-UP:` with
  evidence rather than widening this phase.

- **End-to-end check.** From this checkout's venv, call `MuseProvider().invoke(...)`
  directly with `SASE_ARTIFACTS_DIR` pointing at a temp directory. Use a tool-heavy
  prompt of about a minute on a non-Contributor model.
  - Confirm `live_reply_timestamps.jsonl` shows several commentary chunks timestamped
    well before the final chunk, spread through the run.
  - Confirm the returned content equals the final answer only.
  - Record the evidence (chunk count, first and last chunk offsets from start, and run
    duration) in the phase bead's notes. Leave no capture files outside the committed
    fixtures.
- **Docs.** Update the "Fixture coverage is keyed to Muse release" sentence in
  `docs/reference/providers/muse.md` and the fixture docstrings in the Muse test modules
  to name the new commentary fixtures' release.

## Upstream request (human-owned, in parallel)

No phase can file this with Muse Code. Here is a draft for Bryan to send:

> `muse exec --json` drops assistant commentary. Since 1.4.4, models commit
> `assistant_message_committed` messages with `phase: "commentary"` before tool calls,
> and they appear in `session.jsonl` and the interactive surfaces. Headless exec stdout
> carries only `run.output.delta` for the final answer, so exec consumers cannot show
> progress narration. Please export message phase on exec: either a `phase` field on
> `run.output.delta`, or a durable `run.output.message.committed` event with
> `message_id`, `phase`, and `text`.

## After landing

New agent launches pick up the adapter change only after a global reinstall, which is
human-only. Then launch a tool-heavy Muse agent and select it in the Agents tab. A
one-line chunk with a timestamp divider should appear within about a second of each
commentary message, while the Tools count keeps climbing. Muse's median chunk count per
run should move from 1 toward the 6–9 that Claude, Codex, and Grok show.

## Out of scope

- TUI follow or render changes, and synthetic tool breadcrumbs in the Reply card.
- Showing reasoning summaries; that would belong in a thinking pane, if anywhere.
- Migrating to `muse serve`/MSP.
- Prompting Muse to narrate on stdout.
- Any `sase-core` change.
