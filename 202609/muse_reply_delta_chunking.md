---
tier: tale
title: Coalesce Muse output deltas into one live-reply chunk per run stream
goal:
  A Muse Code reply renders in the agent metadata panel as one timestamped chunk of
  intact prose instead of one chunk per streamed token delta.
size: medium
proposed_by: bbugyi200.apollo.15
create_time: 2026-09-20 11:34:12
status: wip
---

# Plan

## Symptom

In the Agents tab metadata panel, a `muse-spark-1.3` reply is shredded into ~25 separate
`AGENT CHAT` entries, each with its own `─── 11:25:35 ───` divider and all stamped with
the same second. Breaks land mid-word (`It doesn` / `'t replace coding agents —`) and
mid-inline-code (`is \`` / `` sase` — Structured
``), so the markdown renders wrong on top of being fragmented. The screenshot at `~/tmp/screenshots/20260920_112710.png`
captures a DONE muse agent in this state.

## Root Cause

`_stream_output_delta()` in `src/sase/llm_provider/_subprocess_muse.py` routes every
`run.output.delta` payload through the shared `append_stream_text()` helper in
`src/sase/llm_provider/_subprocess_artifacts.py`. That helper is written for providers
whose text events are **whole assistant messages** — Claude's `assistant` event, Codex's
`item.completed`/`agent_message`, Qwen's `assistant`, OpenCode's `text`. For every call
it does two things that are correct per message and wrong per token:

1. `write_reply_timestamp()` appends an entry to `live_reply_timestamps.jsonl` at the
   current `live_reply.md` byte offset.
2. It writes a `"\n\n"` separator before the text whenever the file is non-empty.

Muse is the only provider whose text events are _incremental deltas of a single
message_. So each delta becomes its own timestamped chunk with a hard paragraph break
injected into the artifact bytes.

The TUI then faithfully renders what the artifacts say. `render_agent_reply_content()`
in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_content.py` calls
`agent.get_timestamped_reply_chunks()`, which resolves through
`ArtifactFilesCache.read_reply_chunks()` in `src/sase/agent/artifact_files_cache.py`:
that splits `live_reply.md` at each recorded byte offset. One divider is emitted per
chunk and each chunk is `.strip()`ed before markdown rendering — which is also why the
leading spaces of deltas like `" is \`"` vanish and words fuse or split oddly.

So the panel is not splitting the reply; the muse stream parser writes it pre-split.

## Why It Was Never Caught

Both recorded Muse captures —
`tests/llm_provider/fixtures/muse_exec_read_tool_R708.1.jsonl` and
`tests/llm_provider/fixtures/muse_exec_write_bash_tools_R708.1.jsonl` — contain exactly
**one** `run.output.delta`, carrying a one-word reply (`"bravo"`, `"DONE"`). With a
single delta the separator branch never fires and the file is a single chunk, so
`test_muse_stream_streams_deltas_into_the_live_reply_artifact` asserts
`live_reply == "bravo"` and passes.

That same one-delta capture seeded a factually wrong claim that must be corrected as
part of this work: the module docstring rule 2 in `_subprocess_muse.py` and rule 2 under
"The Event Stream" in `docs/llms.md` both state that `run.output.delta` "repeats the
same text the terminal event later returns". In a one-token reply the single delta does
equal the terminal text; in a real reply the deltas **concatenate** to it. The
consequential half of the rule — deltas are display-only and are never appended to the
returned content — is correct and must stay.

## Secondary Defects From The Same Cause

Both are real and in scope; both disappear with the same fix:

1. `_resolve_muse_content()` joins the salvage fallback `state.streamed_texts` with
   `"\n\n"`. When no `run.terminal.*` event arrives (schema drift), the returned reply
   is mangled exactly like the panel is.
2. With `suppress_output=False`, `append_stream_text()` runs `print(text, flush=True)`
   per delta, so console output gets one line per token.

## The Fix

Keep streaming deltas live — dropping them and writing only the terminal text would
leave the panel blank for the entire run — but coalesce them into one chunk per run
stream.

### 1. `src/sase/llm_provider/_subprocess_artifacts.py`

Add a delta-shaped sibling to `append_stream_text()` (name it `append_stream_delta()` or
similar) that takes an explicit "open a new chunk" flag:

- Opening a chunk: write the timestamp entry and, if the file is non-empty, the `"\n\n"`
  separator, then the text — i.e. exactly today's behavior. Preserve the existing
  ordering where `write_reply_timestamp()` runs **before** the separator is written, so
  a chunk's recorded offset points at the separator and the renderer's `.strip()` still
  cleans it; `read_reply_chunks()` and its test in
  `tests/agent/test_artifact_files_cache.py` depend on that convention.
- Continuing a chunk: append the text only — no timestamp entry, no separator — and
  print with `end=""` so the console mirrors the artifact.

Leave `append_stream_text()` itself untouched. Its per-call timestamp is correct for
every message-shaped provider, and changing it would collapse their legitimate
per-message dividers.

Follow the module's existing convention of publishing an underscore-prefixed alias, and
re-export the new symbol from `src/sase/llm_provider/_subprocess.py` alongside the other
`_subprocess_artifacts` re-exports if symvision or the compatibility surface wants it.

### 2. `src/sase/llm_provider/_subprocess_muse.py`

Teach `_MuseStreamState` which run stream currently owns the open chunk:

- Derive a stream key from the delta payload's `command_id`, falling back to
  `run_stream.id`, then to a fixed sentinel when neither is present. Both captured
  fixtures carry `command_id` and `run_stream.id`, and the `run.terminal.*` payload
  carries the matching `command_id`.
- The first delta for a key opens a chunk; later deltas with the same key continue it.
- A `run.terminal.*` event for the open key closes the chunk, so a subsequent delta from
  a different run stream opens a fresh timestamped chunk. This mirrors the boundary
  `_resolve_muse_content()` already honors when it joins multiple `terminal_texts` with
  `"\n\n"`.
- Accumulate streamed text per stream key so the `_resolve_muse_content()` fallback
  concatenates within a stream and joins across streams with `"\n\n"`. Keep the existing
  `muse_missing_run_terminal_event` diagnostic; its `streamed_chunks` count should now
  mean delta count, or be renamed if the implementer prefers clarity — either is fine as
  long as the diagnostic still fires.
- Correct docstring rule 2 to say deltas are incremental fragments that concatenate to
  the terminal text and are streamed for live display only.

### 3. `docs/llms.md`

Correct rule 2 under "The Event Stream" the same way, and note that SASE coalesces the
deltas of one run stream into a single timestamped `live_reply.md` chunk.

## Alternatives Considered And Rejected

- **Stop streaming deltas; write the terminal text once at the end.** Simplest, but the
  metadata panel would show nothing at all while a muse agent runs. Live reply is the
  point of the artifact.
- **Coalesce in `read_reply_chunks()` or in `render_agent_reply_content()`.** Wrong
  layer. The `"\n\n"` is already baked into the artifact bytes, so the un-chunked
  `get_live_reply_content()` fallback path and the salvage reply stay corrupt; and any
  heuristic there (merge chunks sharing a second, merge chunks not ending in
  punctuation) would misfire on the message-shaped providers that share the reader.
- **Strip and rejoin chunks at render time.** Hides the corruption instead of fixing it
  and still loses the intended paragraph structure of a real multi-paragraph reply.

## Scope Boundary

Muse only. Claude, Codex, Qwen, and OpenCode emit whole assistant messages per text
event, so their `append_stream_text()` path is correct as it stands and must not change
behavior. This is not Rust-core work: every provider's CLI stdout parsing lives in
Python under `src/sase/llm_provider/`, and `sase_core_rs` is not on this path.

If the implementer finds another provider streaming token-level deltas through
`append_stream_text()`, do not widen this plan — record it as discovered follow-up.

## Tests

In `tests/llm_provider/test_muse_provider_core.py` (synthetic envelopes via the existing
`_envelope()` helper are sufficient; a recorded multi-delta fixture is welcome but not
required):

1. **Multi-delta coalescing.** Several deltas sharing one `command_id` plus a matching
   `run.terminal.completed`. Assert `live_reply.md` equals the plain concatenation of
   the delta texts with no injected `"\n\n"`, that `live_reply_timestamps.jsonl` holds
   exactly one entry at `byte_offset` 0, and that the returned content is still the
   terminal text. Use a reply whose deltas split mid-word and mid-inline-code, so the
   screenshot's exact failure is the assertion.
2. **Run-stream boundary.** Two delta groups with distinct `command_id`s, each followed
   by its terminal event: two timestamp entries, `"\n\n"` between the two groups, and
   the returned content still the `"\n\n"`-joined terminal texts.
3. **Salvage fallback.** Deltas with no terminal event return the concatenated text, not
   the `"\n\n"`-joined text, and still record the `muse_missing_run_terminal_event`
   diagnostic.
4. **Existing single-delta assertions keep passing** unchanged
   (`test_muse_stream_streams_deltas_into_the_live_reply_artifact`,
   `test_muse_stream_returns_the_terminal_text_without_delta_duplication`,
   `test_muse_stream_parses_the_write_and_bash_capture`).

Panel-level regression guard: write a muse-shaped `live_reply.md` /
`live_reply_timestamps.jsonl` pair produced by the parser and assert
`render_agent_reply_content()` emits exactly one timestamp divider for it (colocate with
the existing prompt-panel widget tests under `tests/ace/tui/widgets/`).

No PNG snapshot change is expected, since the fix removes dividers rather than restyling
them. If rendered TUI output does move, read `tui.md` and `tui_screenshot.md` with
`/sase_memory_read` first and run `just fix-tui-screenshots`.

## Verification

Read `lint_and_test.md` with `/sase_memory_read`, then run `just fix` inline followed by
`sase tool run check`. Do not run `just check-full` — it is not requested here.

## Out Of Scope

Existing muse artifacts from past runs stay fragmented. `live_reply.md` is ephemeral
display state; no migration or backfill.
