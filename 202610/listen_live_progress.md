---
tier: tale
title: Live progress for sase-listen render (stage checklist, chunk board, safe Ctrl-C)
goal:
  "`sase-listen render` (and the other network-bound commands) always show what the user
  is waiting for: a beautiful live stage checklist with sub-steps, chunk progress, ETA
  and retry countdowns on a TTY; clean plain lines elsewhere; unchanged --json; and a
  Ctrl-C that stops promptly while keeping every paid chunk cached."
size: medium
proposed_by: bbugyi200.athena.0ws.f0.f0
create_time: 2026-10-05 10:49:01
status: wip
---

# Plan: Live progress for `sase-listen` (render checklist, chunk board, safe Ctrl-C)

All work happens in the **sase-listen** linked repo
(`sase repo open sase-listen -r "<why>"`; work only in the path it prints). Read that
repo's `AGENTS.md` first. Run verification with `sase tool run check` from the
sase-listen checkout (never bare `just check`). Do not edit `CHANGELOG.md`
(release-please owns it). Do not import `sase` or `sase_core_rs` (no-sase-import rule).

## Problem

`sase-listen render` is the slowest thing the tool does, and its output today
(`src/sase_listen/cli/render.py::_ProgressEvents`) is a stub left for a "cli phase" that
never landed:

```
stage: synthesize
[1/10] chunk 0 synthesized
[4/10] chunk 3 synthesized
...
stage: gates
stage: master
```

What's wrong with it:

1. **The longest waits print nothing at all.** For an article URL, fetching, extraction
   and the Gemini script writer (up to `writer.max_attempts` = 3 calls, each up to 300
   s, plus lint-repair loops) run inside `pipeline.load_source` before the first event.
   Resolving credentials (`api_key_command`, for example `pass show …`), planning, cover
   art and **publishing over SSH to the feed host** are silent too.
2. **Backoff sleeps are invisible.** `engines/retry.py::synthesize_with_retry` sleeps up
   to 60 s (or a server `retry-after`) with no signal. The writer and every TTS chunk
   use it.
3. **The chunk lines are wrong.** All pending chunks fire `on_chunk_started` in the main
   thread _before_ submission, so the CLI can't tell queued from in flight. Numbering
   mixes 1-based (`[4/10]`) with 0-based (`chunk 3`).
4. **Ctrl-C hangs and wastes money (verified).** `synthesize_chunks` caches results in
   the main-thread `as_completed` loop. On Ctrl-C the `KeyboardInterrupt` leaves that
   loop, and `ThreadPoolExecutor.__exit__` → `shutdown(wait=True)` (no `cancel_futures`)
   keeps running **every queued chunk** before the interrupt surfaces. A probe with 9
   jobs, 3 workers and SIGINT at 0.2 s finished all 9 jobs before the
   `KeyboardInterrupt` surfaced. Then the results are thrown away uncached. With real
   TTS that is minutes of silent waiting, paying for audio that is then discarded. A
   second Ctrl-C doesn't help either: the interpreter's atexit hook joins the workers
   again.
5. Unexpected errors and `KeyboardInterrupt` print a bare message or traceback, with no
   hint about what is cached or how to resume.

## Design goals (the bar for "done")

- **Always know what you're waiting for.** Every wait longer than about a second has a
  named row, a live sub-step in plain English (for example "attempt 2 of 3 · fixing 2
  lint findings · waiting on gemini-3.1-pro-preview"), an elapsed timer, and progress or
  an ETA when it can be measured. Backoff waits show a live countdown. Upcoming stages
  are visible as dim pending rows, so the user knows what's left.
- **Reliable.** `--json` output is byte-for-byte unchanged. Non-TTY output is plain
  lines with no ANSI escapes. A progress-display bug can never fail, slow or corrupt a
  paid render. User content (titles, errors, paths) is never interpreted as Rich markup
  or emoji codes. The terminal is always restored on error or Ctrl-C. Ctrl-C stops
  promptly, keeps everything already paid for in the cache, and says exactly how to
  resume.
- **Beautiful.** One accent color (`ui.ACCENT`, cyan), one glyph set, aligned columns,
  terse `·`-separated facts, and a compact view (at most about 16 lines). The finished
  checklist stays in scrollback as a record of the run. Completed rows collapse into a
  `✓` line with a summary and duration.

## Visual specification

These mockups were prototyped with the repo's own `rich` 15 and screenshotted; match
them closely. Columns: glyph (1), label (width = longest visible label), detail
(ratio=1, `no_wrap`, `overflow="ellipsis"`), time (right-aligned, min width 6). Stage
rows are indented 2 spaces under the header. Build everything from `rich.text.Text` and
`Table.grid` objects. Never pass dynamic strings through markup.

### Live (TTY), mid-synthesis, 80+ columns

```
♪ An open-source spec for Codex orchestration: Symphony (Full)          2m 14s
  openai.com · full edition · Charon · gemini-3.8-flash-tts

  ✓ Fetch article   openai.com · 2,364 words · cached                     0.1s
  ✓ Write script    2,198 words · 8 chapters · 2 attempts               1m 42s
  ✓ Plan episode    10 chunks · 2 cached · ≈14 min · ≈$0.19               0.3s
  ⠹ Synthesize      ━━━━━━━━━━━━━━╸━━━━━━━━━  6/10 · ~40s left              31s
    ♪ chunk 7       Why specs beat prompts                                 12s
    ♪ chunk 8       Running Symphony locally                                9s
    ↻ chunk 9       retry 1 of 4 in 14s · rate-limited (HTTP 429)
                    1 queued
  · Quality gates
  · Master audio
  · Save episode
  · Publish
```

- **Header line 1:** `♪` (bold accent), the title (bold, ellipsized), and total elapsed
  right-aligned (dim). Before any title is known it shows the source (URL without
  scheme, or file name).
- **Header line 2 (dim):** `·`-joined non-empty parts: source site/host or file name,
  `<edition> edition`, narrator voice (or name), narrator model.
- **Active row:** accent `dots` spinner and a bold accent label. The detail is the
  current step text (or, for Synthesize, the progress bar plus `done/total` plus the
  ETA). The time column shows elapsed in whole seconds.
- **Synthesize sub-rows:** these are rows of the **same grid** so their times align with
  the stage times (an earlier nested-table prototype misaligned them). Show one row per
  in-flight chunk (`♪`, label `chunk N`, the chapter title, or "Intro"/"Outro" for those
  kinds, and its elapsed time). Retry-waiting chunks get a yellow `↻` and
  `retry A of M in Ns · <reason>`. Put the countdown **before** the reason so
  ellipsizing never hides it. Show at most 4 chunk rows, then `+K more in flight`. A
  final dim `N queued` row appears when chunks are queued.
- **Progress bar:** `rich.progress_bar.ProgressBar`, `complete_style=ACCENT`,
  `finished_style="green"`, width `clamp(console.width - 52, 10, 28)`. Cached chunks
  count as done (they pre-fill the bar).
- **ETA:** hidden until the first _synthesized_ chunk finishes. Let D be the mean wall
  time of finished synthesized chunks and P the maximum simultaneous in-flight count
  seen (at least 1). Then `eta = queued*D/P + max(0, D - mean in-flight elapsed)`. Round
  to 5 s below a minute (`~40s left`) and to whole minutes above (`~3m left`) to avoid
  jitter.
- **Pending rows:** dim `·` and a dim label. **Done:** green `✓`, normal label, dim
  summary, dim duration. **Warning:** yellow `⚠` with the summary in yellow. **Failed:**
  red `✗`, bold label, red first line of the error. **Interrupted:** yellow `■`.
- **Durations** (one shared formatter): `0.3s`/`6.4s` below 10 s, `41s` below 60 s,
  `1m 42s`, `1h 02m`. Active-row elapsed time uses whole seconds (`0s`, `41s`) so it
  doesn't flicker. **Numbers:** `2,364 words`, `≈14 min` (`<1 min` below 0.5), `≈$0.19`,
  `6.8 MB`, `-16.0 LUFS`.

### Final frame plus summary (success)

The live region stops with `transient=False`, so the finished checklist persists. Each
row shows its outcome, sub-rows disappear, and the header drops its elapsed time. Then
the summary goes to **stdout**:

```
♪ An open-source spec for Codex orchestration: Symphony (Full)
  openai.com · full edition · Charon · gemini-3.8-flash-tts

  ✓ Fetch article   openai.com · 2,364 words · cached                     0.1s
  ✓ Write script    2,198 words · 8 chapters · 2 attempts               1m 42s
  ✓ Plan episode    10 chunks · 2 cached · ≈14 min · ≈$0.19               0.3s
  ✓ Synthesize      8 synthesized · 2 cached · 1 retry                  1m 12s
  ✓ Quality gates   all 10 chunks in range                                0.0s
  ✓ Master audio    14m 12s · -16.0 LUFS · 6.8 MB                         6.4s
  ✓ Save episode    verified · 8 chapters                                 0.2s
  ✓ Publish         apollo (via apollo)                                   2.1s

♪ Ready in 3m 12s · 14m 12s of audio · ≈$0.19
  /home/<user>/.local/share/sase-listen/library/<id>/<slug>.mp3
  Published to apollo — refresh the feed in AntennaPod to download it.
  ⚠ 1 warning
    · Chunk 4 of 10 (content): pace 238 wpm …
```

- "Ready in" is bold green. Print the MP3 path as its **own line with `soft_wrap=True`**
  (the absolute path, not `~`), so Rich never hard-wraps it and it stays copy-pasteable.
  The prototype showed Rich folding it mid-word without this.
- When the live view was not used (`plain`/`off` modes, or stderr not a TTY), the
  summary also carries the title: `♪ Ready in 3m 12s — <title>`.
- Publish line variants:
  `Published to <host> — refresh the feed in AntennaPod to download it.` /
  `Published to the local feed.` /
  `⚠ Publish queued: <reason> — run sase-listen publish --pending` / nothing when not
  publishing.
- With `-o`, add a line `Copied to <path>`.
- The summary uses a stdout `Console(highlight=False, emoji=False, markup=False)`, so it
  degrades to plain text when stdout is not a terminal.

### Failure and interrupt

The failing row turns `✗` with the error's first line. Rows after it stay pending. Then
this goes to **stderr**:

```
✗ sase-listen render failed during Master audio (exit 1)
  [Errno 18] Invalid cross-device link: '…' -> '…'
  hint: Re-run; a killed render resumes from the cache.
  10 of 10 chunks are cached, so re-running will not synthesize them again.
```

The cache line comes from the renderer's chunk counts. Add it only when the failure or
interrupt happened at or after Synthesize and at least one chunk is cached. Keep the
`hint:` prefix (existing tests grep for it). Omit "during <stage>" when no stage started
(for example, config errors). Interrupt (exit **130**) uses a yellow `■` with a hint
that depends on the stage:

- **Fetch article / Read … / Write script / Plan episode:** "Nothing was rendered yet;
  re-run to start again." For Write script, add "the fetched article is cached".
- **Synthesize / Quality gates:**
  "`N of M chunks are cached — re-run the same command to resume without paying for them again.`"
- **Master audio / Save episode:** "All chunks are cached; re-run to finish without new
  synthesis."
- **Publish:** "The episode is saved in your library; publish it with
  `sase-listen publish <episode_id>`."

### Plain mode (non-TTY stderr, `TERM=dumb`, or `--progress plain`)

Plain mode writes line-oriented text to stderr with no ANSI escapes and no carriage
returns. It is thread-safe (one lock). Cached chunks get one aggregate line, not one
line each:

```
♪ An open-source spec for Codex orchestration: Symphony
→ Fetch article
  · fetching openai.com
  · extracting the article text
✓ Fetch article (1.2s): openai.com · 2,364 words
→ Write script
  · attempt 1 of 3 · waiting on gemini-3.1-pro-preview
  ↻ retry 1 of 4 in 14s · rate-limited (HTTP 429)
  · attempt 1 of 3 · checking the draft against the article
✓ Write script (1m 42s): 2,198 words · 8 chapters · 1 attempt
→ Plan episode
✓ Plan episode (0.3s): 10 chunks · 2 cached · ≈14 min · ≈$0.19
→ Synthesize
  · 2 chunks cached
  · 8 to synthesize · 3 at a time
  · chunk 3 of 10 synthesized (12s) · 3/10 done
  ↻ chunk 9 of 10: retry 1 of 4 in 14s · rate-limited (HTTP 429)
✓ Synthesize (1m 12s): 8 synthesized · 2 cached · 1 retry
...
⚠ Publish (2.1s): queued · feed host unreachable (tried apollo, apollo-do)
```

Glyph fallback: if `"♪→·↻✓⚠✗■━".encode(stream.encoding or "ascii")` fails, use ASCII
(`*`, `->`, `-`, `~`, `ok`, `!!`, `x`, `##`). Plain mode uses no spinners and no ETA.

## Architecture

### 1. Event protocol: new `src/sase_listen/events.py`

This is a leaf module (stdlib-only imports; `RenderPlan`/`RenderResult` only under
`TYPE_CHECKING`), so `web/`, `writer/` and `pipeline` can all use it without cycles.
Move `RenderEvents` here. `pipeline.py` keeps it importable with an explicit re-export
(`from sase_listen.events import RenderEvents as RenderEvents`, because mypy strict
disables implicit re-export). CLI modules import from `sase_listen.events`.

```python
class Stage(StrEnum):
    SOURCE = "source"; WRITE = "write"; PLAN = "plan"; SYNTHESIZE = "synthesize"
    GATES = "gates"; MASTER = "master"; SAVE = "save"; PUBLISH = "publish"

class RenderEvents:
    """No-op progress callbacks. Frontends override what they show.

    Order: on_stages, then per stage on_stage -> (on_step | on_retry_wait |
    chunk events)* -> on_stage_done; a failing stage simply raises.
    on_chunk_started, on_chunk_finished and on_retry_wait may be called from
    worker threads; implementations must be thread-safe.
    """
    def on_stages(self, stages: Sequence[str]) -> None: ...      # may be re-sent, refined
    def on_stage(self, stage: str) -> None: ...                   # stage started
    def on_step(self, stage: str, text: str) -> None: ...         # live sub-step text
    def on_stage_done(self, stage: str, summary: str = "", *, warning: bool = False) -> None: ...
    def on_title(self, title: str) -> None: ...                   # source title known
    def on_plan(self, plan: RenderPlan) -> None: ...              # existing
    def on_chunk_started(self, index: int, total: int) -> None: ...  # request really began
    def on_chunk_finished(self, index: int, total: int, *, cached: bool) -> None: ...
    def on_chunk_retried(self, index: int, attempt: int, reason: str) -> None: ...  # gate re-synth
    def on_retry_wait(self, stage: str, chunk: int | None, wait: RetryWait) -> None: ...
    def on_done(self, result: RenderResult) -> None: ...          # existing
```

The old stage names `tag` and `commit` retire. Tagging becomes a Master sub-step, and
verify, commit, copy and prune become the Save stage. Only the CLI and tests consume
these names (SASE integrations use `--json`). Record the change in
`docs/architecture.md`.

`RetryWait` is a frozen dataclass in `engines/retry.py` (`attempt` 1-based retry number,
`max_retries`, `delay_s`, `reason`), re-exported by `events.py`.

### 2. Pipeline and library instrumentation (no presentation logic beyond short text)

The pipeline owns **facts and short plain-English step and summary strings**. The CLI
owns all layout and styling. Put the shared formatters in `src/sase_listen/ui.py` (no
deps): `format_duration`, `format_words`, `format_bytes`, `plural(n, one, many=None)`,
`approx_minutes`, `approx_cost`. Add the new glyph constants there too (`GLYPH_PENDING`
`·`, `GLYPH_RETRY` `↻`, `GLYPH_STOP` `■`, `GLYPH_ARROW` `→`, `SPINNER = "dots"`).
Summary composition goes in small pure helpers in `pipeline.py`, unit-tested.

- **`engines/retry.py`:**
  `synthesize_with_retry(..., on_retry: Callable[[RetryWait], None] | None = None)`.
  Call it right before each sleep with the computed delay.
- **`writer/gemini.py` + `writer/__init__.py`:** `GeminiWriter(..., on_retry=None)`
  passes it through. `create_writer(config, *, on_retry=None)`. Do **not** change the
  `Writer` protocol; test fakes implement `write(system, user)`.
- **`writer/author.py::author_script(..., events: RenderEvents | None = None)`:** before
  each call, emit step `attempt {n} of {max} · waiting on {model}`. When repairing, emit
  `attempt {n} of {max} · fixing {k} lint finding(s) · waiting on {model}`. After the
  reply, emit `attempt {n} of {max} · checking the draft against the article`.
- **`web/store.py`:** add `reused: bool = False` to `AcquiredSource` (set True on a
  cache hit). `acquire(..., on_step: Callable[[str], None] | None = None)` emits
  `fetching {host}` (or `reading saved HTML from {file name}`), then
  `extracting the article text`.
- **`audio/mastering.py::master_to_mp3(..., on_step=None)`:** emit
  `measuring loudness (pass 1 of 2)` before pass 1 and `encoding the MP3 (pass 2 of 2)`
  before pass 2. The audio package stays CLI-agnostic.
- **`feedhost.py`:**
  - `run_remote(..., on_attempt: Callable[[str], None] | None = None)` is called with
    each SSH destination before trying it.
  - `publish_any(..., on_step=None)` and `flush_pending(cfg, *, on_step=None)` emit
    `sending {n} queued episode(s) first` (only when the outbox is non-empty),
    `packing the episode`, `sending to {host} via {dest}` (and
    `{prev} unreachable · trying {dest}` on fallback), or `updating the feed` locally.
- **`pipeline.py`:**
  - Add pure helpers `source_stages(source, edition) -> list[Stage]` (SOURCE, plus WRITE
    for URL brief/full), `should_publish(request, cfg, kind: str | None) -> bool` (the
    existing `want_publish` rule, **reused by `render`** so they cannot drift; unknown
    kind means tentatively True when `auto_publish` is on), and
    `expected_stages(request, cfg, kind)`. A dry run stops after PLAN.
  - `load_source(..., events=None)` emits SOURCE start, steps, `on_title` and done
    (summaries below). For URL brief/full it emits the WRITE stage. A cached script gets
    an immediate done with `… · cached`. Before `create_writer`, emit step
    `reading the Gemini API key`. Pass `on_retry` into `create_writer` so writer backoff
    surfaces as `on_retry_wait(Stage.WRITE, None, wait)`. For refs, emit step
    `reading {ref} via sase artifact read`.
  - `render()`:
    - Emit
      `on_stages(expected_stages(request, cfg, kind=<"article" for URLs, else None>))`
      first, then again with the real kind after `load_source`.
    - PLAN stage steps: `resolving the narrator and API key` (around `prepare`),
      `splitting {words} words into chunks` (around `plan_request`),
      `preparing cover art` (around `resolve_cover_bytes`). The dry run finishes PLAN
      and returns.
    - Emit `on_stage(SYNTHESIZE)` **before** taking the episode lock, so a lock conflict
      fails on a named row.
    - MASTER steps: `assembling {n} chapter(s)`, the two mastering steps,
      `writing chapters and cover art` (`write_tags`).
    - SAVE steps: `verifying the MP3`, `saving to the library`, `copying to {path}`,
      `pruning the chunk cache`.
    - PUBLISH runs only when `should_publish`. Success is done with
      `{host} (via {dest})` or `local feed`. A queued remote failure is done with
      `warning=True`, `queued · {error}`. A non-explicit local failure is done with
      `warning=True`, `skipped · {error}`. An explicit `--publish` failure raises as
      today.
  - **Stage summaries** (omit zero or empty parts):
    - **SOURCE:** `{site or host} · {words} words[ · cached]` (URL),
      `{ref} · {words} words` (ref), or
      `{file name} · {words} words[ · normalized from Markdown]` (file).
    - **WRITE:** `{words} words · {n} chapters · {k} attempt(s)` or `… · cached`.
    - **PLAN:** `{n} chunks · {c} cached · ≈{m} min · ≈${cost}`. Use `all {n} cached`
      when everything is cached, and drop the cost when it is 0 (tone).
    - **SYNTHESIZE:** `{s} synthesized · {c} cached · {r} retry/retries`, where `r` is
      the sum of `attempts-1` over synthesized chunks. Use `all {n} chunks cached` when
      nothing was synthesized.
    - **GATES:** `all {n} chunks in range`, or `{k} re-synthesized` with
      `warning=bool(gate_warnings)`.
    - **MASTER:** `{audio duration} · {lufs:.1f} LUFS · {size}` (size measured after
      tagging).
    - **SAVE:** `verified · {n} chapters[ · copied to {name}]`.
  - **`synthesize_chunks` rewrite (fixes problem 4):**
    - Each worker task calls `events.on_chunk_started` when its request actually begins.
      It runs `synthesize_one(..., on_retry=…)` (new param, threaded to
      `synthesize_with_retry`, emitting `on_retry_wait(Stage.SYNTHESIZE, index, wait)`).
    - The worker writes the result to the cache **itself** (`ChunkCache.put` is already
      atomic and concurrency-safe), then emits `on_chunk_finished(..., cached=False)`.
    - After the cache-hit scan, emit step
      `{pending} to synthesize · {concurrency} at a time`.
    - Manage the pool explicitly. Collect with a loop of
      `concurrent.futures.wait(pending, timeout=0.25, return_when=FIRST_COMPLETED)`
      instead of a bare `as_completed`. CPython does not block SIGINT in worker threads,
      so the kernel may deliver a terminal Ctrl-C to a worker. A main thread parked in
      an untimed wait then does not act on it until the next chunk completes, which can
      be a minute. The timed loop wakes the main thread about 4 times a second.
    - On any `BaseException` in the collect loop: `cancel()` every not-yet-started
      future and count the running ones. If interrupted and some are running, emit step
      `stopping · finishing {n} in-flight chunk(s) so they stay cached · Ctrl-C again to quit now`.
      Then `shutdown(wait=True)` (running chunks finish and cache themselves) and
      re-raise.
    - `SaseListenError` takes the same path, so a permanent failure no longer burns the
      queue either.
    - `_resynthesize` (gates) passes `on_retry` too.
  - **Human chunk numbering:** gate warnings and the hard-gate error become 1-based
    `Chunk {i+1} of {n} ({kind})`. Manifest and JSON `index` fields stay 0-based (data).

### 3. CLI renderers: new `src/sase_listen/cli/progress.py`

- **`ProgressState`:** the model. It holds stage rows (id, label, status, started/ended,
  step, summary), header (title, subtitle), and a chunk board (total, done, cached,
  in-flight `{index: started}`, waits `{index: (deadline, RetryWait)}`,
  chunk→chapter/kind from `on_plan`, finished durations, max parallel). Every mutation
  goes under one `threading.Lock`, and the clock is injectable (`clock=time.monotonic`).
  - `on_stages` replaces only the _pending_ tail; completed and active rows never move.
  - `on_stage` for an unannounced stage appends it.
  - Labels come from the CLI's knowledge of the source (`looks_like_url` → "Fetch
    article", `looks_like_ref` → "Read ref", else "Read file"), then "Write script",
    "Plan episode", "Synthesize", "Quality gates", "Master audio", "Save episode",
    "Publish".
  - `on_chunk_retried` during GATES sets the step
    `re-synthesizing chunk {i+1} of {n} · {reason}`.
- **`build_view(snapshot, now, width)`:** pure, returns the Rich renderable per the
  visual spec. Spinner frames come from `Spinner("dots").render(now)`, so they are
  deterministic under a fake clock.
- **`LiveProgress(RenderEvents)`:**
  `rich.live.Live(self, console=Console(stderr=True, highlight=False, emoji=False, markup=False), refresh_per_second=10, transient=False)`
  with `__rich__` → `build_view`. It is a context manager. On exit it marks the active
  row `failed` (exception), `interrupted` (`KeyboardInterrupt`) or leaves it done,
  renders a final frame without spinner or sub-rows, and stops Live (cursor restored on
  every path).
- **`PlainProgress(RenderEvents)`:** the line format above, written to stderr under a
  lock.
- **Safety guard:** every event handler and the view build run inside
  `try/except Exception`. On the first internal error the display disables itself (and
  stops Live), remembers the error, and afterwards prints one dim line:
  `progress display error: <exc> (the render was not affected)`. The pipeline must never
  see an exception from a progress sink.
- **`resolve_mode(flag, *, as_json, console)` → `live | plain | off`:**
  - `auto` is `off` with `--json`. Otherwise it is `live` if the stderr console
    `is_terminal` and `TERM != "dumb"`, else `plain`.
  - `live` forces a terminal console (useful under `script(1)`).
  - `NO_COLOR` removes color only; Rich handles it and the view still animates. This
    replaces the docs' old "NO_COLOR ⇒ plain" claim.
- **`interrupt_guard(on_force_quit)`:** a context manager (main thread only; restores
  the previous SIGINT handler). The first SIGINT raises `KeyboardInterrupt`. The second
  calls `on_force_quit()` (stop Live, print
  `■ Stopped immediately; in-flight chunks were not cached.`) and then `os._exit(130)`.
- **`activity(text, *, enabled)`:** a context manager with `.update(text)` for the short
  network waits in other commands.
  - TTY: a transient `console.status(..., spinner="dots", spinner_style=ACCENT)` on
    stderr.
  - Non-TTY: each text printed once as a plain stderr line.
  - Disabled under `--json`.

### 4. `render` and `script` integration

- **`cli/render.py`:**
  - Add `--progress {auto,live,plain,off}` (default `auto`). Delete `_ProgressEvents`.
  - Time the run. Wrap `render(...)` in `interrupt_guard` plus the progress context.
  - Add `ExitCode.INTERRUPTED = 130` to `errors.py`. Interrupt → exit 130. Human mode
    prints the interrupt block. JSON mode prints
    `{"ok": false, "error": {"code": 130, "message": "Interrupted.", "hint": "Re-run the same command; finished chunks are cached."}}`.
  - Replace `_print_result` with the summary spec. Replace `_print_plan` with a Rich
    layout that keeps the section words `Chapters:`, `Omissions (N):` and `Warnings:`:
    - a header,
    - a facts line,
    - a writer-usage line,
    - the script path,
    - a borderless chapters table (number, title, words, ≈min, chunks),
    - omissions and warnings.
  - Errors use the failure block (stderr) in every human mode.
- **`cli/script_cmd.py`:** for URL sources, add the same `--progress` flag and run
  `load_source(..., events=progress)` under it. The CLI sends
  `on_stages(source_stages(...))`. stdout (the script text or the `Wrote …` line) is
  unchanged. Apply the same interrupt handling (exit 130).

### 5. Short waits in other commands

Wrap these remote and SSH calls in `activity(...)`, feeding `on_attempt` / `on_step`
updates through `.update()`. Keep their existing stdout and `--json` output unchanged.

- `publish` (`Publishing {id} to {host}…`, and per-episode
  `sending queued episode {i} of {n}` for `--pending`)
- `unpublish` (remote: `Removing {id} from {host}…`)
- `feed` remote status and actions (`Asking {host} for the feed status…`)
- the `doctor` feed-host probe (`Checking feed host {host}…`)

## Tests (all offline; CI runs Linux and macOS, so no real PTYs)

- **`tests/test_engines.py`:** the `on_retry` callback receives the correct
  attempt/max/delay (with injected `rand`/`sleep`), and `retry_after` is reflected in
  `delay_s`.
- **New `tests/test_events.py`:**
  - A `RecordingEvents` (thread-safe) over a tone render of the tiny script:
    - `on_stages` arrives first; stage order is SOURCE, PLAN, SYNTHESIZE, GATES, MASTER,
      SAVE; every started stage gets exactly one done with a non-empty summary;
      `on_title` fires;
    - synthesized `on_chunk_started` calls come from non-main threads;
    - MASTER steps include both loudness passes;
    - no PUBLISH without publishing, and PUBLISH present with a local feed config;
    - the dry run stops at PLAN.
  - URL brief path with a fake writer (reuse `tests/test_writer.py` patterns plus saved
    HTML via `--html`): WRITE steps include `attempt 1 of`; the cached re-run summary
    ends `· cached`.
  - `should_publish` / `expected_stages` truth table.
- **Interrupt regression (problem 4):** make it race-free. Raising `KeyboardInterrupt`
  _inside_ a worker races the freed worker against the main thread's `cancel()`.
  - Use the tiny script (4 chunks) with concurrency 1. The fake engine's 1st call
    returns audio. Its 2nd call (chunk index 1) calls `_thread.interrupt_main()` (a
    SIGINT as seen by the main thread), then blocks on a `threading.Event` (timeout 5 s)
    before returning audio.
  - A `RecordingEvents.on_step` sets that event when it sees the `stopping ·` step. So
    chunk 1 is provably in flight while chunks 2 and 3 are queued.
  - Assert:
    - `render` raises `KeyboardInterrupt`;
    - the engine saw exactly **2** calls (queued chunks never reached it);
    - chunks 0 **and 1** are in the cache (the in-flight chunk drained and cached
      itself).
  - On the current code this test sees 4 calls and no cached chunk 1.
  - CLI: `main([... "render", src, "-n", "tone", "--progress", "plain"])` with a fake
    engine patched in via `pipeline.default_engine` (its 2nd call interrupts the main
    thread, then sleeps about 0.5 s) returns **130**, and stderr mentions `cached` and
    `re-run`.
- **New `tests/test_progress.py`:**
  - `ProgressState` and `build_view` under a fake clock, rendered through
    `Console(file=StringIO(), width=80, force_terminal=False, color_system=None)`, with
    golden text snapshots for: writing (step and elapsed), mid-synthesis (bar, ETA,
    in-flight rows, a retry countdown, `+K more`, `queued`), failure, interrupt, and the
    final success frame.
  - Narrow width (50) doesn't crash and keeps the time column.
  - Title `"[bold]x[/bold] :smile:"` renders literally.
  - `on_stages` refinement keeps done rows.
  - ETA rounding.
  - `PlainProgress`: exact lines for a scripted event sequence, no `\x1b`, and ASCII
    fallback with an ascii-encoded stream.
  - The guard: a handler that raises internally never propagates, and the notice prints
    once.
  - `resolve_mode` table, including `--json` → off and `TERM=dumb` → plain.
  - `interrupt_guard`: call the installed handler directly. The 1st call raises
    `KeyboardInterrupt`; the 2nd calls the force-quit callback and a monkeypatched
    `os._exit(130)`. The previous handler is restored.
- **CLI tests:**
  - `render … --progress plain` (tone, `--no-publish`): stderr has `✓ Synthesize`;
    stdout has `Ready in` and the MP3 path **on one unbroken line** even with
    `COLUMNS=40` and a long `XDG_DATA_HOME`.
  - `--progress live` on a captured stream (forced terminal): the final frame text
    contains every done row.
  - `--json`: stdout is exactly one JSON object and stderr is empty.
  - `script <url> --html … --progress plain` with a fake writer: script text on stdout,
    progress on stderr.
  - `publish` activity: a plain stderr line when not a TTY; nothing extra with `--json`
    (reuse the fake-ssh fixtures in `tests/test_feedhost.py`).
- Update `test_human_output_paths` and any assertions that depend on the old `stage:` /
  `[i/n] chunk` lines or the 0-based gate text.

## Visual QA (required; this is what "beautiful" is checked against)

1. Add `tools/progress_demo.py`. It replays a scripted, realistic event timeline (URL
   source → writer with one repair and one 429 backoff → 10 chunks at concurrency 3 with
   a retry → master → save → publish) through `LiveProgress` at real speed (`--speed`
   factor). With `--svg-dir DIR` it instead renders key frames (writing, mid-synthesis,
   final, failure) via
   `Console(record=True, width=84, force_terminal=True, color_system="truecolor").save_svg(...)`
   under a fake clock.
2. Screenshot the SVGs with headless Chrome and **look at the PNGs** (Read tool), then
   iterate on spacing, alignment and colors until they match the spec and look polished.
   Gotcha: under the agent `TMPDIR`, Chrome dies with "Socket path too long" (singleton
   socket). Use a short temp dir and a profile:
   `P=$(mktemp -d /tmp/chr.XXXX); TMPDIR=$P google-chrome --headless=new --disable-gpu --no-sandbox --hide-scrollbars --user-data-dir=$P/ud --window-size=1100,640 --screenshot=$PWD/live.png file://$PWD/live.svg; rm -rf $P`.
3. Commit the final-frame SVG as `docs/assets/render-progress.svg` and embed it in
   `docs/cli.md`.
4. Real-terminal smoke test, free and offline (tone engine) through a PTY:
   `script -qec "uv run sase-listen render src/sase_listen/data/demo.md -n tone --no-publish" /dev/null`.
   Also run with `--progress plain`, with `2>&1 | cat`, and with `--json`. Confirm a
   clean final frame, no leftover cursor or ANSI garbage in the plain/pipe cases, and an
   unchanged JSON shape. Run `tools/progress_demo.py` once animated in a PTY and press
   Ctrl-C mid-synthesis to check the interrupt frame and block.

## Docs

- **`docs/cli.md`:**
  - a new "Live progress" section (the SVG, what each row means, the ETA caveat);
  - `--progress` on `render` and `script`;
  - exit code `130`;
  - Ctrl-C semantics (first press finishes in-flight chunks so they stay cached, second
    press quits immediately);
  - rewrite "Non-TTY and NO_COLOR behavior" to match `resolve_mode`.
- **`docs/reliability.md`:** interrupt semantics, and "progress output can never fail a
  render".
- **`docs/architecture.md`:** the events protocol (stages, steps, thread rules, retired
  `tag`/`commit`) replaces the current one-sentence description.
- **`docs/troubleshooting.md`:** add "Progress output is garbled or too chatty → use
  `--progress plain` / `off`". In the EXDEV section, note that the stage now shows as
  "Master audio".
- **`AGENTS.md` architecture map:** add `events.py` and `cli/progress.py`.

## Out of scope

- The `audition`/`ls`/`cache` stubs.
- ffmpeg percent-progress parsing.
- A config-file or env progress preference.
- Any `--json` schema change.
- `content_blocked` split-and-retry.
- Telegram and sase-research-artifacts changes.
- Reinstalling the user's `uv tool` install. Report the upgrade command instead:
  `uv tool install --force --reinstall git+https://github.com/sase-org/sase-listen`.

## Acceptance

- `sase tool run check` passes (ruff, format, mypy --strict, codespell, pytest).
- The interrupt regression test fails on the pre-change `synthesize_chunks` (4 engine
  calls) and passes after.
- The SVG screenshots were reviewed and match the visual spec.
- The PTY smoke test shows the live checklist and the stdout summary.
- `--json` output is unchanged and stderr is empty in `--json` mode.
- Report back:
  - before/after output for the tone demo, in plain mode;
  - the PNG paths reviewed;
  - the exact exit codes observed for success, failure (missing file) and interrupt.
