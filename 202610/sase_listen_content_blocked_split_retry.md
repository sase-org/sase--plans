---
tier: tale
title: "sase-listen: split-and-retry Gemini TTS content_blocked chunks"
goal:
  A render whose chunk trips Gemini TTS's context-dependent content_blocked policy
  filter recovers automatically by synthesizing smaller pieces and stitching them; only
  a sentence that is blocked on its own fails, with an error naming the chunk, chapter,
  and sentence and an accurate hint.
size: medium
proposed_by: bbugyi200.athena.0wx
create_time: 2026-10-05 12:00:49
status: wip
---

# Plan: sase-listen: recover from Gemini TTS `content_blocked` by splitting the chunk

All work happens in the linked **sase-listen** repo. Open it with
`sase repo open sase-listen -r "<reason>"`, work only in the printed path, and read its
`AGENTS.md` first. This is not shared SASE domain behavior, so nothing here touches
`sase` or `sase-core` (sase-listen's no-sase-import rule applies).

## Problem

```
sase-listen render https://www.anthropic.com/engineering/harness-design-long-running-apps -e full
...
[7/10] chunk 6 cached
[9/10] chunk 8 cached
sase-listen render: error: Synthesis failed after retries: Gemini request failed (HTTP 400):
  Error code: 400 - {'error': {'message': 'Request blocked for an unspecified policy reason.
  Please modify your input and retry.', 'code': 'content_blocked'}}..
hint: Re-run to resume from the chunk cache, or try another narrator.
```

## Root cause (diagnosed live, 2026-10-05)

Nine of ten chunks were already cached. Only chunk index 7 (`[8/10]`, chapter "Results
from the updated harness", 316 words, 4 paragraphs: heading plus three body paragraphs
about the Digital Audio Workstation build) needed synthesis. Direct
`GeminiEngine.synthesize` probes against `gemini-3.8-flash-tts` (voice Charon) showed:

| Input sent                                                                                                        | Result                             |
| ----------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| Whole chunk, twice                                                                                                | blocked both times (deterministic) |
| Whole chunk, `style` annotation removed                                                                           | blocked (style is not the trigger) |
| Heading, body paragraph 1, body paragraph 2, each alone                                                           | OK                                 |
| Body paragraph 3 (8 sentences: song snippet, melody, drum track, song composition, "Claude cannot actually hear") | blocked                            |
| Each of those 8 sentences alone                                                                                   | all OK                             |
| Sentences 1-4 together                                                                                            | OK                                 |
| Sentences 5-8 together ("song snippet … melody … drum track … song composition … musical taste")                  | blocked                            |

Gemini TTS's input policy classifier (most likely its music/singing-generation
restriction) blocks this text **by context**: a dense run of song-composition wording
gets rejected, but no single sentence does. The API returns HTTP 400 with a structured
body
`{'error': {'code': 'content_blocked', 'message': 'Request blocked for an unspecified policy reason. Please modify your input and retry.'}}`.
The exception is a `google.genai._gaos.lib.compat_errors.BadRequestError` with
`status_code == 400` and that dict on `exc.body`.

This is the second time it has happened. `docs/field-notes.md` § "Gemini TTS
`content_blocked`" records the first occurrence (harness-engineering full edition, fixed
then by hand-editing the script) and names the follow-up this plan implements: _"treat
`content_blocked` as a split-and-retry signal rather than a hard episode failure."_ The
same follow-up appears as notes on beads `sase-1g7.4` and `sase-1e3`.

Today sase-listen gets this wrong in three ways:

1. **No recovery.** `engines/gemini.py::_map_api_error` maps every non-key 400 to a
   generic `PermanentEngineError`. `synthesize_with_retry` never retries those, and
   `pipeline.synthesize_one` turns it into exit 4, which aborts the whole episode even
   though the same words go through when sent in smaller pieces.
2. **Misleading hint.** "Re-run to resume from the chunk cache" cannot help: the block
   is deterministic, so every re-run fails the same way. The message also says "after
   retries" when nothing was retried (permanent errors are never retried).
3. **No location.** The error does not say which chunk or chapter failed (the user can
   only tell from the gap in the `[n/10]` progress lines), and the message ends in `..`
   because the engine message's final period gets a second one appended by the pipeline.

## Design

### 1. Engine taxonomy: `ContentBlockedError`

In `src/sase_listen/engines/base.py`, add
`class ContentBlockedError(PermanentEngineError)`. Docstring: the provider's policy
filter refused this exact input; re-sending it unchanged will fail again, but the same
words often go through in smaller pieces. Subclassing `PermanentEngineError` keeps every
existing `except PermanentEngineError` path and `synthesize_with_retry`'s never-retry
rule correct with no other changes. Export it from `engines/__init__.py` (import plus
`__all__`).

### 2. Gemini classification

In `src/sase_listen/engines/gemini.py`, add a small helper (for example
`_is_content_blocked(exc, code, detail)`). Classify as content-blocked when the HTTP
code is 400 and either:

- the structured body says so: `getattr(exc, "body", None)` is a dict whose
  `["error"]["code"]` equals `"content_blocked"` (read it defensively; any shape
  mismatch means "not blocked"), or
- the secret-free detail text contains `content_blocked` (covers the
  `google.genai.errors.APIError` path and SDK shape drift).

`_map_api_error` checks this after the invalid-API-key check and before the generic 400
→ `PermanentEngineError`. It returns
`ContentBlockedError("Gemini's policy filter blocked the text (HTTP 400 content_blocked)")`,
with no trailing period so callers can punctuate. Leave the 401/403/429/5xx mapping
alone.

The OpenAI adapter is out of scope: nothing has shown an OpenAI equivalent. The pipeline
recovery below is engine-agnostic, so any adapter that later raises
`ContentBlockedError` gets it for free.

### 3. Pipeline: split-and-retry inside `synthesize_one`

`pipeline.synthesize_one` is the single entry point for both first-pass synthesis
(`synthesize_chunks`) and gate re-synthesis (`_resynthesize`), so the recovery goes
there and both paths get it.

Flow:

1. Synthesize the whole text through `synthesize_with_retry`, as today.
2. On `ContentBlockedError`, call a new private helper (for example
   `_synthesize_blocked(...)`) that recursively bisects the text and synthesizes the
   pieces:
   - **Unit selection:** if the text has more than one paragraph (split on blank lines,
     `\n\s*\n`), the units are paragraphs, re-joined with `"\n\n"`. Otherwise the units
     are sentences from the existing `split_sentences`, re-joined with a single space.
     Never split below a sentence; clause-level fragments sound broken.
   - **Bisection:** split the units into two halves at `len(units) // 2`. For each half,
     in order: synthesize its joined text through `synthesize_with_retry` (transient
     errors still retry per piece); if that raises `ContentBlockedError`, recurse on
     that half. A half that still holds several paragraphs bisects again by paragraph; a
     single blocked paragraph drops to sentence units. Output keeps the original text
     order.
   - **Terminal case:** a single sentence that is still blocked cannot be recovered
     automatically. Raise a `ContentBlockedError` carrying that sentence (as an
     attribute such as `.text`, or in the message) so the pipeline can name it.
   - Count every engine call (including the blocked ones) into the returned `attempts`.
     The recursion is bounded: at most about 2 × (number of sentences) calls, roughly 9
     calls for the chunk above.
3. **Stitching:** trim each piece's PCM with `audio.trim_silence`, resample any piece
   whose rate differs from the first piece's using `audio.resample`, and join the pieces
   with `chunk_gap_s` of digital silence (`config.audio.chunk_gap_s` when `config` is
   provided, otherwise the `assemble` default of 0.5 s). The stitched chunk then sounds
   like the planned chunks it sits between: the same gap, no doubled pauses, and far
   below the 4 s internal-silence hard gate. Return the stitched s16le bytes and the
   first piece's sample rate.
4. **Report the split:** extend `synthesize_one`'s return so callers learn how many
   pieces the chunk was rendered in (1 means not split). A 4-tuple or a small
   `NamedTuple` both work; update both call sites. Add `pieces: int = 1` to
   `SynthesizedChunk`.
   - `synthesize_chunks` and `_resynthesize` store `pieces` into the cache entry's
     `usage` dict (next to `engine`/`model`). The cached-hit branch reads
     `hit.usage.get("pieces", 1)`, so the warning still shows on reruns that come
     straight from the cache.
   - In `render`, after `run_chunk_gates`, add one warning per final chunk with
     `pieces > 1`, for example: _"Chunk 8 (Results from the updated harness): Gemini
     blocked the full chunk (content_blocked); synthesized it in 5 pieces."_ Append
     these to `warnings` next to the gate and size warnings, so they reach the CLI
     output, `--json`, and the manifest's warnings.
   - Add `"pieces"` to each chunk's entry in the manifest (next to `"attempts"`, around
     the `made.attempts` serialization in `render`).
   - Splitting already counts toward `retried_chunks` through the existing
     `made.attempts > 1` check in `run_chunk_gates`; keep that.

The cache key stays the chunk's existing content-addressed key. The stitched audio is
the audio for exactly that text, so a rerun is a full cache hit with no new TTS calls.

### 4. Accurate errors and hints

Rework the `except` ladder in `synthesize_one` (keep the `CredentialsError` branch as it
is):

- `ContentBlockedError` (only reaches this point in the terminal single-sentence case):
  message _"Gemini's policy filter blocked a sentence even on its own: "<sentence,
  truncated to ~160 chars>""_. The hint says re-running will not help; the user should
  rephrase that sentence in the narration script and re-render (unchanged chunks come
  from the cache) or use another narrator (`-n openai`).
- Other `PermanentEngineError`: message `Synthesis failed: <exc>` (drop "after
  retries"). The hint says a permanent rejection repeats on re-run, so fix the input or
  try another narrator.
- `TransientEngineError`: keep `Synthesis failed after retries: <exc>` and the current
  "Re-run to resume from the chunk cache, or try another narrator." hint.
- Strip trailing periods from the wrapped exception text before adding the sentence's
  own period, so messages never end in `..`.

**Chunk location:** failures surfaced from `synthesize_chunks` (and the gate
re-synthesis path) must name the 1-based chunk position (matching the `[8/10]` progress
display), the total, and the chapter, for example
`Chunk 8/10 ("Results from the updated harness"): …`. For the content-blocked hint, also
name the narration script path when the plan knows it (`plan.script_path`; for URL
sources this is the cached `full_narration.md`). Either pass a context label and an
optional script path into `synthesize_one`, or catch and re-raise the `SaseListenError`
in `synthesize_chunks` with the prefix added while keeping the exit code and `__cause__`
chain. Choose whichever stays simplest. `run_chunk_gates` does not receive the plan, so
it gets the chunk label without the script path.

## Tests

`tests/test_engines.py`:

- Give the `_CompatError` fake an optional `body` attribute. Add these cases: `400` +
  body `{'error': {'code': 'content_blocked', ...}}` → `ContentBlockedError`; `400` +
  only `content_blocked` in the message → `ContentBlockedError`; `400` + "something else
  broke" → plain `PermanentEngineError`, not `ContentBlockedError` (extend
  `test_gemini_compat_error_taxonomy` or add a sibling test).
- Assert `ContentBlockedError` is a `PermanentEngineError` subclass and that
  `synthesize_with_retry` calls the operation exactly once when it raises.

`tests/test_pipeline.py`, using a new fake such as `PolicyEngine(ToneEngine)` that
raises `ContentBlockedError` whenever `request.text` contains **both** of two marker
words (for example `MELODY` and `DRUMS`). This mirrors the context-dependent block: each
marker alone passes, both together fail.

- **Recovers across paragraphs:** the markers sit in different paragraphs of one chunk.
  The render succeeds, the engine received each paragraph separately, the result warns
  about the split chunk with its 1-based index and chapter, and the manifest records
  `pieces > 1` for it.
- **Recovers across sentences:** the markers sit in different sentences of a single
  paragraph. The render succeeds by dropping to sentence units.
- **Cached on rerun:** a second render with a `CountingEngine` makes zero engine calls,
  and the split warning still appears (it is read from cache usage).
- **Stitch shape:** the stitched PCM's duration is about the sum of the trimmed pieces
  plus `(pieces - 1) × chunk_gap_s`, and the chunk passes the hard gates.
- **Terminal failure:** both markers in one sentence → `SaseListenError` with
  `ExitCode.SYNTHESIS_FAILED`. The message names the chunk position, the chapter, and
  the sentence; the hint mentions rephrasing (plus the script path for file sources) and
  does **not** say "Re-run to resume".
- **Message hygiene:** the existing `MarkerFailEngine` (plain permanent error) path
  produces a message without "after retries" and without `..`, and the
  `test_render_resume_after_crash` expectations still hold.

## Docs

- `docs/troubleshooting.md`: add a section such as "Gemini blocked the text
  (`content_blocked`)". Explain the policy filter, the automatic split-and-retry, the
  render warning it leaves, and what to do when a single sentence is blocked: rephrase
  it in the narration script (path given in the error), or render with another narrator.
  Correct the "Synthesis failed after retries (exit 4)" section so it no longer implies
  every exit-4 failure was retried.
- `docs/reliability.md` (Render pipeline section): add a bullet on content-blocked
  split-and-retry, and adjust the exit-code 4 wording to "synthesis failed".
- `docs/field-notes.md` § "Gemini TTS `content_blocked`": append a short dated note
  recording the second occurrence (harness-design full edition, chunk 8, context block
  on song-composition wording) and that the split-and-retry follow-up is now
  implemented.

## Verification

1. `sase tool run check` in the sase-listen repo. It runs lint (ruff, format, mypy
   `--strict`, codespell) plus pytest; it must pass.
2. Live check without library or feed side effects, using the user's Gemini key (costs a
   few cents): from the sase-listen checkout, run a short `uv run python` snippet that
   builds the real plan for
   `https://www.anthropic.com/engineering/harness-design-long-running-apps` with
   `edition="full"` (`prepare`, then
   `load_source(source, edition="full", config=prep.config)`, then `plan_request`), and
   calls `synthesize_one` on `plan.chunks[7]` with the real engine. Confirm it returns
   audio with `pieces > 1` instead of raising. Do **not** run a full `render` that
   publishes. The user's config sets `feed.auto_publish: true`, and publishing to the
   private podcast feed is the user's call.
3. In the final summary, tell the user that the `sase-listen` on their PATH is an
   editable `uv tool` install pointing at the primary sase-listen checkout. Once that
   checkout has the landed commit, re-running their original command re-synthesizes only
   chunk 8 (nine chunks are cached), splitting it automatically.

## Out of scope

- Rephrasing blocked text with an LLM or silently dropping blocked sentences (both break
  "read the text exactly as written").
- Falling back to a different narrator per chunk (the voice would change mid-episode).
- Writer-side (`generate_content`) safety blocks: a different API and failure shape.
- Closing or editing beads `sase-1g7.4` / `sase-1e3`. Their notes already record this
  follow-up, and the host owns bead status.
