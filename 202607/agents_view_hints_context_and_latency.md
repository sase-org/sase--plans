---
tier: tale
title: Agents-tab view hints for SASE CONTEXT memory files + non-blocking hint rendering
goal: 'Pressing `v` on the Agents tab renders selectable hints next to SASE CONTEXT
  memory-file rows, and hint rendering no longer blocks the UI thread for ~1s+ on
  a detail-header summary cache miss.

  '
create_time: 2026-07-16 17:40:40
status: wip
prompt: 202607/prompts/agents_view_hints_context_and_latency.md
---

# Plan: SASE CONTEXT memory-file view hints + fast, non-blocking hint rendering

## Product context

On the Agents tab of `sase ace`, the `v` (view) keymap re-renders the agent detail panel with numbered `[N]` hints next
to viewable files and mounts a `HintInputBar` so the user can type a hint number to view/edit/copy a file.

Two problems:

1. **Feature gap:** Memory files listed in the `SASE CONTEXT → MEMORY` lane (e.g. `tui_perf.md` with its `↩ frontmatter`
   marker) get no hint, so they cannot be opened through the `v` flow even though sibling header sections (PLAN, DELTAS,
   ARTIFACTS, SLOW TOOL CALLS, COMMITS) all render hints.
2. **Latency:** Users sometimes wait ~1s or more after pressing `v` before hints render and they can type a selection.
   The whole TUI is frozen during that window.

## Root-cause findings (measured, not guessed)

Profiled on real agent state (308 agents, replies up to ~100KB / ~220 timestamped chunks):

- `AgentPromptPanel.update_display_with_hints` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py`) calls
  `build_detail_header_summary(agent)` **synchronously on the UI thread** whenever `get_cached_detail_header_summary`
  misses. Measured cost of that build on real agents: **up to ~1390ms** (many agents 100–800ms), dominated by
  `build_slow_tool_sources` (400–860ms) plus the SASE CONTEXT lane loaders
  (`load_memory_reads/skill_uses/opened_workspaces_for_agent_context`, ~100ms combined). This directly violates tui_perf
  rule 1 ("never block the event loop") and is the ~1s freeze the user reports.
- The miss window is real and common: the normal display path only _reads_ the summary cache and fills it via a
  background enrichment worker (`start_agent_detail_header_enrichment` in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_async.py`). Pressing `v` within ~1.5s of selecting an agent
  lands before the worker finishes.
- Worse, the `v` press itself advances `AgentDetail._agent_detail_generation`
  (`src/sase/ace/tui/widgets/agent_detail.py`), so when the in-flight enrichment worker completes,
  `_apply_agent_detail_header_enrichment_result` fails its `is_current` check and **discards the freshly built summary
  without caching it** — the hint path then rebuilds the exact same data synchronously. The expensive work is done twice
  and the blocking rebuild is guaranteed in precisely the case where a worker was already running.
- The warm path (summary cached) is fine: regex hint scanning on a 97KB reply is ~7ms, `humanize_vcs_refs_in_text` ~16ms
  (uncached in the hint path; the plain path caches it via `_humanize_display_text`), Rich wrap ~40ms, warm summary
  rebuild 20–40ms. No restructuring of the text rendering is needed — the fix is about never doing the summary build on
  the UI thread and never discarding a built summary.

## Design

### Part A — hints for SASE CONTEXT memory files

Follow the established `HeaderHintState` pattern used by the ARTIFACTS section (`_agent_artifacts.py`): marker `[N] `
styled `bold #FFFF00` inserted before the row's primary text, and `hint_state.hint_mappings[N] = <abs path>`.

- `build_header_text` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`): pass the existing `hint_state`
  through to `append_agent_context_section` (today it is the only hint-capable section called without it).
- `append_agent_context_section` (`src/sase/ace/tui/widgets/prompt_panel/_agent_context.py`): accept
  `hint_state: HeaderHintState | None = None` and forward it to the MEMORY lane only.
- `append_agent_memory_reads_section` (`src/sase/ace/tui/widgets/prompt_panel/_agent_memory_reads.py`): when
  `hint_state` is provided, render a hint marker for each visible read row and map it to `event.resolved_path` (the
  absolute path recorded by the `sase memory read` audit log — no disk access needed at render time). Each visible row
  gets its own hint even when the same file was read twice (rows are events; at most `MAX_VISIBLE_READS = 5` rows
  render). The overflow ("+ N more") line gets no hint.
- `append_lane_row` (`src/sase/ace/tui/widgets/prompt_panel/_agent_context_common.py`): extend with an optional
  hint-label argument rendered between the glyph and the primary text, and include its width in the returned reason
  indent so `↳ reason` lines stay aligned under hinted rows.
- Selection plumbing needs no changes: `v`-mode mappings are plain `int -> absolute path`, and the existing
  view/edit/copy processing (`src/sase/ace/tui/actions/hints/_files.py`, `_processing.py`) handles `.md` files already.

Non-goals: hints for the SKILLS lane (skill names, not single files) and the WORKSPACES lane (directories, not viewable
files); ChangeSpecs-tab hint flow.

Rule compliance: no `stat`/`exists` calls in the render path (tui_perf rule 8) — `resolved_path` comes straight from the
audit event already held in the cached `DetailHeaderSummary`.

### Part B — non-blocking hint rendering

Never build the detail-header summary on the UI thread; reuse the existing background enrichment worker and its caches
(tui_perf rules 1 and 5 — no new refresh paths).

1. **Stop discarding built summaries.** In `_apply_agent_detail_header_enrichment_result`
   (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_async.py`), call `cache_detail_header_summary` on `SUCCESS`
   _before_ the `is_current` check. The cache is keyed by agent identity + associated-plan key, so caching a
   stale-generation result is safe; only the re-render stays gated on currency. This alone removes the duplicated
   0.4–1.4s rebuild.
2. **Remove the synchronous fallback from the hint path.** In `update_display_with_hints`
   (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py`), on a summary-cache miss: render immediately with
   `summary=None` (hints for timestamps, commits, xprompt, prompt, and chat content appear instantly) and request
   enrichment through the existing worker with a render context captured _after_ the hint-path generation bump so its
   completion passes `is_current`. Coalescing with an already in-flight worker for the same agent must keep working (the
   existing same-identity request-replacement logic in `start_agent_detail_header_enrichment`).
3. **Hint-aware worker completion.** When enrichment completes and the app is in Agents-tab hint mode for the same agent
   (`_should_render_agent_detail_with_hints` in `src/sase/ace/tui/actions/agents/_display_detail.py`), the completion
   must route through the app's hint-preserving refresh (`_render_agent_detail_with_hints`) instead of the plain
   `_update_display_impl`, so the annotated view is not clobbered and the app's `_hint_mappings` / `_hint_commit_views`
   / `_hint_tool_call_reports` are refreshed together with the display. Mirror the existing message-based pattern
   (`LinkedDeltasRefreshed` in `src/sase/ace/tui/widgets/file_panel/_messages.py`) rather than having the widget reach
   into app internals.
4. **Never renumber under the user's fingers.** Skip the hint-mode completion re-render when the mounted
   `HintInputBar`'s input (`#hint-input`) already contains typed text: the summary-backed sections would insert hints
   ahead of content hints and shift every number the user is currently reading. The enriched summary stays cached; the
   on-screen numbering and stored mappings stay mutually consistent. When the input is still empty (the overwhelmingly
   common case within the sub-second completion window), re-render with the full section set.
5. **Warm-path polish + observability.**
   - Use the panel's cached `_humanize_display_text` helper (from `AgentDisplayRenderMixin`, same class via the
     `AgentPromptPanel` MRO) in the hint path instead of raw `humanize_vcs_refs_in_text` calls (~16ms/press on large
     replies, plus per-chunk savings).
   - Add `tui_trace` spans (e.g. `widget.prompt_panel.update_display_with_hints`) so this path shows up in
     `~/.sase/perf/tui_trace.jsonl` and regressions are measurable; today it has no span at all.

Behavioral tradeoff to accept: on the (now rare) cold path, summary-derived sections (PLAN, DELTAS, ARTIFACTS, SASE
CONTEXT, SLOW TOOL CALLS) are absent from the annotated view for the sub-second until the worker lands, and hint numbers
shift once when it does (unless the user already typed, in which case the partial view stays stable). This replaces a
full UI freeze in the same scenario.

Rust-core boundary check: everything here is presentation-only Textual rendering, widget state, and worker plumbing — no
`sase-core` changes.

## Testing

- Extend `tests/ace/tui/widgets/test_agent_memory_reads.py` and `tests/ace/tui/widgets/test_agent_context.py`: hint
  markers render for visible MEMORY rows, mappings point at `resolved_path`, reason-line indent accounts for the marker,
  no hints without `hint_state`, no hint on the overflow line, SKILLS/WORKSPACES lanes unchanged.
- Hint-path render test: `update_display_with_hints` with a cached summary containing memory reads includes them in
  `AgentHintRender.file_hints`.
- Non-blocking guarantees:
  - `update_display_with_hints` never calls `build_detail_header_summary` (monkeypatch it to raise; assert the hint
    render still succeeds on a cold cache and that an enrichment request was started).
  - `_apply_agent_detail_header_enrichment_result` caches the summary even when `is_current` fails.
  - Completion while hint mode is active re-renders with hints and refreshes the app-side mappings; completion with
    typed hint input skips the re-render; completion without hint mode keeps today's plain re-render.
- Existing regressions must keep passing, especially `tests/ace/tui/test_agents_view_hint_survives_refresh.py`
  (auto-refresh re-renders preserve hint mode) and the PNG visual suite (`just test-visual`) — update goldens only for
  intentional hint-marker additions.
- Manual/perf verification: with `SASE_TUI_TRACE=1`, press `v` immediately after selecting a cold agent; the new span
  should show the hint render completing in tens of milliseconds (previously the freeze was up to ~1.4s), and the stall
  watchdog log should stay quiet.

## Risks

- **Completion routing regressions:** the enrichment completion currently always plain-renders when current; the
  hint-mode branch must not change plain-path behavior. Covered by the completion tests above.
- **Number-shift UX on the cold path:** mitigated by the typed-input guard; strictly better than the current freeze.
- **Coalesced worker requests:** replacing the request context on a same-agent in-flight worker must carry the post-`v`
  generation, or the completion would be discarded again. Add a test for pressing `v` while enrichment is in flight.
- `build_slow_tool_sources` itself remains slow (0.4–0.9s off-thread); making it cheaper is out of scope here but worth
  a follow-up if the cold-path section fill-in feels laggy.
