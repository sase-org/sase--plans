---
tier: epic
title: Restore visible Muse reply streaming without fragmenting replies
goal: 'Muse reply deltas reach the selected agent''s visible Reply card during generation,
  before terminal completion, with intact text, responsive navigation, and correctly
  framed interactive console output.

  '
phases:
- id: live-reply-follow
  title: Refresh the selected live Reply card
  depends_on: []
  size: medium
  description: 'live-reply-follow: route reply-file events to a throttled background
    snapshot and Reply-card update, with a selected-source polling backstop and lifecycle,
    scroll, attempt, and navigation guards; keep the Agents loader uninvolved.

    '
- id: jsonl-reader
  title: Drain provider JSONL promptly and preserve UTF-8
  depends_on: []
  size: medium
  description: 'jsonl-reader: replace buffered text reads in the shared JSONL transport
    with bounded nonblocking byte reads and incremental decoding, preserving stderr,
    partial records, EOF, and completion-watchdog semantics across providers.

    '
- id: console-framing
  title: Preserve complete console lines under the provider timer
  depends_on: []
  size: small
  description: 'console-framing: avoid flushing individual fragments through Rich
    FileProxy while retaining prompt artifact writes, plain-stream flushing, and final
    chunk closure; verify the real provider timer on a terminal.

    '
- id: streaming-validation
  title: Verify the complete Muse streaming path and document its behavior
  depends_on:
  - live-reply-follow
  - jsonl-reader
  - console-framing
  size: medium
  description: 'streaming-validation: exercise a gated fake Muse through the mounted
    TUI, capture a real Muse reply with transport-to-paint timing and visual evidence,
    verify performance and targeted goldens, and document the supported behavior.'
proposed_by: bbugyi200.athena.0vt
create_time: 2026-10-03 15:03:48
status: done
bead_id: sase-1fu
---

- **PROMPT:** [prompts/202610/muse_reply_streaming.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/muse_reply_streaming.md)
- **BEAD:** [sase-1fu](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1fu/README.md)

# Restore visible Muse reply streaming

## Outcome, scope, and sizing

As Muse emits reply text, the selected agent's Reply card should grow while the provider
remains running. The reader must deliver every complete JSONL record promptly even when
Muse pauses after a burst. Words, Unicode, inline code, and Markdown must remain intact,
with one timestamp divider per contiguous run stream. Interactive console output under
the provider timer should appear as complete lines; ordinary stdout and agent logs
retain fragment-by-fragment streaming.

This is an epic because it combines independently reviewable TUI scheduling, shared
subprocess transport, console framing, and integration evidence. The first three phases
have no dependencies and may run independently; the final phase depends on all three.
Medium phases are substantial but bounded implementation work with a specified design,
rather than additional planning assignments.

All implementation belongs in the primary `sase` checkout. Textual refresh and rendering
are presentation, and descriptor reads and console output are Python process glue. Reuse
the existing artifact cache and Rust-backed path resolution; this plan introduces no
backend domain, wire contract, or binding, and needs no `sase-core` changes or
revision-pin update. If implementation discovers a new shared domain requirement, stop
that scope expansion and replan the boundary.

Each phase delivers a complete correction with regression coverage. No feature flag is
planned: there is no deprecated API migration or partially exposed new product flow. The
optional sunset flag in the research is not required by `sase_flags.md`; do not retain
the broken refresh or transport as a permanent choice. If phase boundaries change and
expose unfinished behavior, follow that memory's beta scaffolding policy and remove the
scaffolding before epic landing.

## Evidence and constraints

Read the audited artifact
`research:202610/muse_live_reply_streaming/muse_live_reply_streaming.md` before
implementing. It consolidates live Muse, transport, terminal, and corpus probes. Its
baseline is Muse `1.4.2-R4684.1` and Rich `14.3.3`; verify the actual installed versions
when collecting new evidence. The plan author also inspected the current source and ran
short local transport and Rich `FileProxy` probes.

The September 20 fix, `a0244d7599`, remains correct. In
`src/sase/llm_provider/_subprocess_muse.py`, `_stream_output_delta` calls
`append_stream_delta` for every fragment. The helper flushes `live_reply.md` on every
write and timestamps only a new chunk. Terminal text remains authoritative for
`InvokeResult.content`; deltas are salvage when terminal text is unavailable. The
existing coalescing, stream-boundary, diagnostics, usage, retry, and wait-guard
contracts must survive.

Three gaps explain the visible problem:

1. `EventWatcherRefreshMixin._on_artifact_change` returns after reply paths produce no
   dirty surfaces. `artifact_path_affects_agents` intentionally ignores both reply files
   to protect loader performance. Existing countdown and auto-refresh paths refresh
   markers, rows, and files rather than the Reply card. The conditional slow-tool tick
   does not cover Muse's reply after tool completion.
2. `stream_json_lines` reads one line per `select` wakeup. A local child flushed three
   lines together: line one was dispatched at 13 ms, while lines two and three waited
   until the next write at 450 ms. Existing fixture tests generally inspect artifacts
   only after exit and cannot catch this stall.
3. Rich `FileProxy.flush` prints its current fragment with a newline. The local probe
   turned three flushed fragments into three broken lines; withholding the proxy flush
   until the final newline produced one intact line.

The local transport probe also split the valid UTF-8 bytes of `café` across two writes
and got `caf��`. Therefore adopt the research's incremental byte-reader alternative now,
rather than merely draining the existing replacement-decoding text wrapper. Valid split
characters must survive; genuinely invalid bytes still decode leniently.

Two research recommendations need adaptation to the current source. The TUI uses a
hidden prompt panel as the source for Main deck card documents, so changing that hidden
widget alone is insufficient: publish through its document sink. Also
`Agent.get_artifacts_dir()` calls path resolution and directory checks; it is not a
disk-free accessor. Resolve reply-source paths off-thread and store them for watcher
comparisons rather than calling it in the event handler.

Muse controls emission cadence and usually emits reply text near the end of its tool
work. This change displays available reply bytes promptly; it cannot create text that
Muse has not emitted. Short answers may still arrive in one burst, and mechanical
plan/monitor handoffs may legitimately have no reply.

## Phase live-reply-follow

### Implementation

Primary entry points are `src/sase/ace/tui/actions/event_refresh/_watcher.py`,
`src/sase/ace/tui/actions/_event_countdown.py`, and
`src/sase/ace/tui/widgets/prompt_panel/`. Reuse `AgentDetailRenderContext` in
`_agent_display_async_types.py`, the lifecycle patterns in `util/pump_tasks.py`, and
card-document publication in `widgets/_agent_detail_deck_source.py`. Read `tui.md` and
`tui_perf.md` first.

1. Add a small presentation controller, preferably a focused prompt-panel mixin, with
   request, collect, apply, and cancel responsibilities. Prepare a descriptor for the
   selected concrete live agent's reply files in a background worker. Resolve artifact
   paths there, retain the selected identity and render generation, and use those stored
   paths for event matching. Avoid filesystem resolution, stat, glob, JSON reads, or
   path discovery on the UI thread.
2. In `_on_artifact_change`, route matching `live_reply.md` and
   `live_reply_timestamps.jsonl` events to this controller before the empty-target
   return. Preserve ordinary marker handling for mixed batches. Reply events must
   continue to produce no Agents dirty flag or roster delta. Use the existing rearmed
   live-directory watcher coverage rather than installing a second watcher or watching
   every archived directory.
3. Enable following only for the selected detail on the Agents tab in a live status,
   using the existing live-status vocabulary and excluding bash/python workflow output.
   Skip hint mode, a pinned historical attempt, and views without a Reply card. For a
   consolidated agent-session/follow-up Reply card, use its already loaded live turn
   sources and stable block IDs; refresh the relevant block without replacing historical
   replies. Bound polling to the currently displayed live source, rather than the whole
   session or roster. Summary-only clan/tribe views stay on their existing refresh
   routes.
4. Use a 300 ms leading-and-trailing throttle, one active collection, and
   scheduled/running/pending guards. Coalesce duplicate events while retaining the final
   pending update. Timer callbacks only schedule work through `spawn_pump_free_task`;
   the task performs stat and reads in a worker thread. Defer during navigation and
   prompt typing through the established activity gates, then converge on the latest
   bytes when the user is idle.
5. Collect an immutable reply presentation snapshot using the existing
   `ArtifactFileCache.read_reply_chunks` and `read_live_reply` behavior. The joint
   timestamp/reply signatures already handle a growing final chunk. Include the
   reply-source identity, generation, attempt mode/pin, signatures, and content. Prepare
   expensive Markdown/humanization work from that snapshot off-thread where it is pure;
   the UI application should only compose/apply presentation objects. Do not access
   mutable widget state from the collection thread. Handle a timestamp-only creation,
   missing file, truncation, or replacement as recoverable state. Do not mark a
   signature applied until its snapshot has been accepted; a changed signature during
   collection must cause another request.
6. On the UI thread, recheck current tab, selected identity, render generation, attempt
   state, hint mode, and activity gates after every await. Replace only the Reply card
   or active turn block in the current source document, retaining the latest
   Context/Summary cards and their enrichment. Publish through the existing Main
   document sink so every visible Main pane receives the update. Reuse the reply
   rendering conventions without rereading files during render. Do not invoke the whole
   `_update_display_impl` pipeline, restart enrichment, rediscover files, or reload the
   roster for every refresh.
7. Piggyback on `_on_countdown_tick` with a pump-free signature probe, at most once per
   second for the selected displayed live source, even when inotify is absent. Compare
   `(mtime_ns, size)` for the two reply files and issue the same refresh request only on
   drift. Skip inactive tabs, non-live sources, navigation, and typing. Probe overlap
   must coalesce. Selection/visibility changes reset the source descriptor and require
   an initial collection, so ignored events while hints or an attempt pin were active
   cannot leave a stale view afterward.
8. Cancel timers/tasks and invalidate generations on selection, pin, hint,
   terminal-state, tab, and teardown changes. Clear guards on collection failure or
   failure to spawn. A trailing refresh must not be lost when terminal markers race
   reply writes: terminal display should take the latest reply through the normal
   terminal-state route.
9. Preserve folds, focus, card/block selection, split layouts, and scroll position. Use
   the established bottom-pin/follow behavior in `_section_view.py` and
   `decks/final/live.py`: follow growth when the visible reader is already at the
   bottom, preserve position when reading above it, and resume after returning to the
   bottom. Hidden source-widget scrolling must not control visible panes.

### Acceptance and tests

Add controller tests and mounted `AcePage`/Textual tests. Start with a selected RUNNING
Muse agent with one timestamp and no pending tool. Append fragments and prove the
visible Reply card grows before any terminal/done marker, without a selection change,
Agents loader invocation, header-enrichment restart, or file rediscovery. Keep
`tests/ace/tui/widgets/test_agent_reply_muse_chunks.py` and
`test_artifact_change_ignores_non_loader_artifact_content` passing.

Cover nonselected reply events, mixed reply/marker events, unavailable watchers,
unchanged signature probes, burst coalescing and its last fragment, descriptor
initialization races, navigation/typing deferral, pin/hint transitions, stale workers
after selection or generation changes, terminal races, and cancellation. Prove slow file
reads do not block a keypress in the mounted app. Check a live session turn, single and
split Main panes, folded replies, and bottom-follow versus scroll-up behavior. Trace
collection/application counts so a continuous burst does not produce more than one
application per throttle interval, apart from the initial leading update. A quiet
selected probe performs at most two stats, and reply bytes never increase loader or
broad-surface reload counts.

## Phase jsonl-reader

### Implementation

Keep the public `stream_json_lines` callback and return tuple unchanged. Confine new
transport helpers to `_subprocess_stream.py` or a focused private sibling. The same loop
serves Muse, Claude, Codex, Qwen, and OpenCode. Plain-text/PTY providers also use
`prepare_nonblocking_text_stream` and `drain_reaped_streams`; preserve those callers'
existing helper contracts.

1. For the JSONL route, read stdout and stderr descriptors nonblockingly with `os.read`.
   Give each stream one incremental UTF-8 decoder with replacement for malformed input,
   and assemble stdout records across read boundaries. Do not mix raw descriptor reads
   with wrapper `readline`, iterator, or `read` calls during normal reading or exit
   draining. Preserve current newline semantics, including CRLF, large records, and the
   final record without a newline.
2. Drain each ready descriptor until it would block, actual EOF, or a bounded per-turn
   byte/record budget. Distinguish `BlockingIOError` from a zero-byte EOF, retry
   interrupted reads, remove EOF descriptors from `select`, and flush decoder/record
   tails exactly once. Valid multi-byte characters wait for their continuation bytes
   instead of becoming replacement characters.
3. Bound work per stream, for example 64 KiB reads and 256 KiB or 256 records per
   iteration, so continuous stdout cannot starve stderr, process-exit checks, or
   watchdog settlement. If decoded complete records remain after reaching the dispatch
   budget, service that backlog on subsequent iterations without waiting for another OS
   readiness signal. Budgeting must never strand buffered records.
4. Accumulate stderr completely and respect `suppress_output`. Decode split characters
   incrementally there too; console fragments without a newline should remain
   observable. Poll for process completion and the teardown watchdog between bounded
   work batches, including while pending records are draining.
5. Reuse this byte-reader state for post-exit and watchdog draining on the JSONL route.
   Preserve the bounded reaped-pipe settle deadline, handling of shielded descendants
   that retain pipe handles, `settle_teardown_stall`, stderr notes, and return-code
   normalization only when the accepted-declaration watchdog owns termination. A normal
   nonzero exit must remain nonzero. Leave the legacy text helper available to plain/PTY
   callers; add no second reader of their pipes.

### Acceptance and tests

Use real subprocesses and explicit parent/child handshakes. A child writes model
metadata and several Muse deltas in one flush, then waits for parent release before
emitting a terminal record. All deltas must be dispatched and present in `live_reply.md`
while it is still blocked. Test a burst exceeding the dispatch budget, fair stderr
delivery during continuous stdout, stdout EOF while stderr continues, absent streams,
and empty success/failure exits.

Split JSON records and raw UTF-8 characters across writes, including emoji and multibyte
text on stderr. Assert exact valid text, lenient invalid-byte handling, and one final
tail dispatch. Retain the existing 400,000-character dribbled-record regression. Update
transport-specific mocked `readline` tests to the descriptor contract; preserve
behavioral assertions rather than preserving the implementation detail or weakening
Unicode expectations.

Run `tests/llm_provider/test_subprocess_utf8_decode.py`,
`test_completion_watchdog_streams.py`, Muse stream/invocation/artifact tests, and the
affected Claude, Codex, Qwen, and OpenCode parser tests. Also retain plain stream
watchdog coverage because of the shared helper seam. A process killed by the
accepted-declaration watchdog must return its complete reply promptly even with a
shielded descendant holding a pipe open.

## Phase console-framing

In `_subprocess_artifacts.append_stream_delta`, detect Rich's actual
`rich.file_proxy.FileProxy` on `sys.stdout`. Emit with `end=""` and suppress the
per-fragment console flush only for that proxy. Keep per-fragment artifact writes and
file flushes unconditional, retain console flushing for ordinary stdout/log streams, and
respect `suppress_output`. Use the existing newline at chunk closure to flush the
remaining proxy text, including when the parser exits abnormally.

Verify the real `provider_timer` path in a bounded PTY subprocess, plus a direct proxy
test for precise framing. A fragment sequence split mid-word/inline-code must render
intact; an embedded newline should become visible before a handshake allows the terminal
record, and the final unterminated line should appear on chunk closure. Strip terminal
control sequences only in the test assertion. Cover plain stdout flushing before exit,
suppressed console output with growing artifacts, multiple run streams, and cleanup on
exception. Existing Muse console and timestamp-offset tests must stay green. No
provider-timer redesign or new console option is needed.

## Phase streaming-validation

1. Add an end-to-end regression using a gated fake Muse executable and the production
   parser, while a mounted ACE app displays the selected agent. Write multiple reply
   deltas, pause before terminal/done, and assert intermediate visible content and one
   intact timestamp divider. Exercise the watcher route and independently its polling
   backstop. Then release the child and prove terminal-authoritative content appears
   once without duplicating deltas. Include an interval after the last tool finishes, so
   the slow-tool tick cannot make the test pass accidentally. Bound waits and clean up
   every test-owned process.
2. Capture a real local Muse long reply with the installed release recorded. Timestamp
   raw JSONL arrival, parser dispatch, artifact growth, and visible mounted-panel
   application. Register a concise sanitized evidence artifact and retain a
   release-keyed, multi-delta fixture where useful; exclude session paths and
   credentials. Inspect an actual midstream `sase screenshot` PNG using the current
   checkout's TUI. The success criterion is multiple partial replies visible while the
   provider is running, before `run.terminal.*` or `done.json`. Separate upstream
   emission timing from transport-to-paint latency. If the real reply arrives as one
   burst, try one sufficiently long answer and report the measured cadence; do not
   insert artificial delays into production output.
3. Under normal idle input, target event-driven transport-to-paint latency of about 500
   ms, and polling convergence within about 1.5 s. These are local measurement targets;
   deterministic tests assert handshakes and lifecycle ordering with generous bounded
   timeouts instead of tight wall-clock limits. Use `SASE_TUI_TRACE=1` to show the
   throttle and no extra broad Agents reloads; use `SASE_TUI_PERF=1` while navigating
   past a streaming agent to assess the established j/k p95 target below 16 ms. Compare
   before/after under the same fixture/load and investigate regressions before claiming
   completion.
4. Add deterministic visual coverage for an incomplete live reply and its grown content,
   covering the visible Main Reply card in representative single/split layouts. Run
   targeted `just fix-tui-screenshots -- <selectors>` including the affected Agents
   deck/context snapshots. Inspect every changed PNG and the retained report; a
   `partial` report does not prove skipped goldens current.
5. Update `docs/llms.md` to explain continuous artifact writes, selected Reply-card
   following, terminal-authoritative return content, and intact line streaming under
   Rich's timer. Describe Muse's quiet tool phase and bursty short replies accurately.
   No new CLI flags, keymaps, or persistent refresh preferences are required; update the
   help popup only if implementation changes an exposed TUI option's description or
   behavior.

## Verification and completion requirements

Every coding phase reads `lint_and_test.md`, runs focused meaningful regressions,
formats its changes, and passes the guarded `sase tool run check` recipe before
completion. The final phase runs it again after integration tests/documentation changes.
Do not run `just check-full`; it is explicit-only. Use `/sase_monitor` for commands that
can outlast the provider's synchronous ceiling, choosing the handoff before starting. A
mutating visual-check monitor needs a follow-up that inspects the report and PNG changes
before finalization. Agents leave commits, branches, and PRs to the host-owned
finalizer.

The epic is complete when a Muse reply grows visibly during generation, the shared
reader drains bursts without damaging Unicode or watchdog handling, Rich console output
keeps words and lines intact, and regression tests plus live visual/performance evidence
support those outcomes. Keep the existing chunking fix and terminal-result contract
throughout. Reasoning-summary progress, session log tailing, extra narration directives,
PTY/stdbuf production wrappers, and an MSP migration are outside this plan. If timing
shows an upstream emission limit, document the evidence and replan that separate scope
rather than expanding this epic during implementation.
