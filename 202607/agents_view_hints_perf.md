---
tier: epic
title: Make Agents-tab `v` view hints load fast
goal: 'Pressing `v` on the Agents tab paints the hint input bar immediately and renders
  numbered hints in bounded, cached, off-pump work, so hint latency stops scaling
  with transcript size, family member count, or auto-refresh cadence.

  '
phases:
- id: measure
  title: Instrument and baseline the view-hints keypath
  depends_on: []
  size: small
  description: 'measure: add end-to-end trace spans for the `v` keypath, add a view-hints
    scenario to the TUI trace bench, capture a committed baseline, and document the
    new spans in the perf runbook. No behavior change.

    '
- id: bound
  title: Bound hint-mode content and remove per-fragment disk work
  depends_on:
  - measure
  size: medium
  description: 'bound: apply the normal render path''s byte/line caps to hint-mode
    body content (including family member fragments), hoist per-chunk workspace resolution
    out of the family hint loop, and memoize hint path resolution.

    '
- id: offpump
  title: Paint the hint bar before rendering the annotated document
  depends_on:
  - measure
  size: medium
  description: 'offpump: restructure the view-files action so the hint input bar mounts
    first and the annotated render runs as a pump-free task, with a readiness guard
    so an early submission still resolves against complete hint mappings.

    '
- id: cache
  title: Cache the annotated hint document
  depends_on:
  - bound
  size: medium
  description: 'cache: memoize the hint render result and wrap the annotated document
    in a width-keyed cached renderable so repeat presses, refresh repaints, and post-enrichment
    repaints reuse rendered segments.

    '
- id: dedupe
  title: Stop redundant hint re-renders on refresh and enrichment
  depends_on:
  - cache
  - offpump
  size: medium
  description: 'dedupe: skip the full hint document rebuild on Agents-tab refreshes
    and header-enrichment messages whose inputs did not change, and stop the one-second
    summary TTL from forcing repeated enrichment while hint mode is active.

    '
- id: verify
  title: Verify the speedup and guard it
  depends_on:
  - dedupe
  size: small
  description: 'verify: re-run the view-hints bench against the committed baseline,
    record the after numbers, add a regression floor, and sync the help popup and
    perf runbook with any user-visible behavior change.

    '
create_time: 2026-07-27 14:21:10
status: wip
bead_id: sase-a5
---

- **PROMPT:** [202607/prompts/agents_view_hints_perf.md](prompts/agents_view_hints_perf.md)
- **BEAD:** [sase-a5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-a5/README.md)

# Plan: Make Agents-tab `v` view hints load fast

## Problem

`v` on the Agents tab is bound to `action_view_files` (`src/sase/ace/tui/bindings.py:21`,
`src/sase/default_config.yml:235`). On the Agents tab it routes to `_view_agent_files`
(`src/sase/ace/tui/actions/hints/_files.py:65`), which calls `AgentDetail.update_display_with_hints` →
`AgentHintsDisplayMixin._update_display_with_hints_impl`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_hints.py:94`) and only mounts the `HintInputBar` **after** that
render returns (`_files.py:99-104`).

That render is a completely separate code path from the normal detail render (`_agent_display_render.py:160`), and it is
the only prompt-panel render path in the widget that is simultaneously **uncapped, uncached, and synchronous on the
Textual event loop**. Everything the normal path learned about staying fast was applied to `_update_display_impl` and
not to its hint twin.

### Confirmed defects

1. **No content caps in hint mode.** `_update_display_with_hints_impl` appends the whole xprompt, the whole prompt, and
   every reply chunk into one `rich.text.Text` via `append_text_with_file_hints`
   (`src/sase/ace/tui/widgets/prompt_panel/_file_path_hints.py:108`). The normal path routes the same content through
   `lazy_renderable` (`src/sase/ace/tui/util/lazy_syntax.py:351`), which caps markdown highlighting at
   `MARKDOWN_SYNTAX_HIGHLIGHT_MAX_BYTES` (24 KB) / `MARKDOWN_SYNTAX_HIGHLIGHT_MAX_LINES` (600) and caps the plain
   fallback at `PLAIN_RENDER_MAX_BYTES` (128 KB) / `PLAIN_RENDER_MAX_LINES` (5000). Hint mode has no cap at any size.

2. **No render caching in hint mode.** The normal path returns a `_CachedRenderable`
   (`src/sase/ace/tui/util/lazy_syntax.py:82`) that caches rendered segments per console width, so a repaint at an
   unchanged width is nearly free. Hint mode constructs a brand-new raw `Text` and hands it to `self.update(...)` every
   time, so every repaint re-renders and re-wraps the entire document.

3. **Synchronous on the event loop, with the bar painted last.** `action_view_files` is an ordinary synchronous Textual
   action. The user gets zero feedback until the whole annotated document has been built and the container has laid out.
   This inverts the shape `sase/memory/tui_perf.md` prescribes ("Do UI mutations (unmount/focus) first, then schedule
   the heavy work") and violates its rule that keystroke paths never do synchronous disk work.

4. **The document is rebuilt on every refresh while hint mode is active.** `_should_render_agent_detail_with_hints` /
   `_render_agent_detail_with_hints` (`src/sase/ace/tui/actions/agents/_display_detail_render.py:236-251`) re-run the
   full hint render on each Agents-tab detail repaint, and `on_agent_detail_header_enriched` (same file, `:253`) re-runs
   it again when the threaded header-enrichment worker lands.

5. **First press on a cold summary renders twice and shows incomplete hints.** `get_cached_detail_header_summary`
   (`_agent_display_header_summary.py:53`) returns `None` until the enrichment worker has populated the panel-local
   cache for that agent identity. On a miss, the first hint render omits every SASE CONTEXT hint — artifact files,
   associated plan, phase bead, memory reads, skill uses, deltas, commit views, slow-tool reports — because
   `build_header_text` only emits those sections when `summary is not None` (`_agent_display_header.py:238`). Those
   hints appear only after the worker returns and the whole document is rebuilt. `should_refresh_detail_header_summary`
   treats the summary as stale after `DIFF_CACHE_TTL_SECONDS = 1.0` (`src/sase/ace/tui/widgets/file_panel/_diff.py:53`),
   so the worker re-arms about once a second.

6. **Per-fragment disk work in the family hint path.** For a family container row,
   `_family_reply_renderables_with_hints` (`_agent_display_family_render.py:308`) calls `_family_member_hint_workspace`
   (`:297`) **per reply chunk per member**, and that calls `resolve_agent_workspace_dir` (`_file_path_hints.py:42`) →
   `parse_workspace_dir` (`src/sase/workspace_provider/utils.py:86`), which `os.path.exists` and then opens and reads
   the project spec file, plus `detect_workflow_type` through pluggy and an `os.path.isdir`. Family members' replies are
   also rendered in full with no cap, so cost scales with member count.

7. **Per-match path resolution.** `resolve_file_path` (`_file_path_hints.py:94`) runs `os.path.expanduser` +
   `os.path.isabs` + `os.path.join` for every regex match with no memoization.

8. **No end-to-end trace coverage.** `widget.prompt_panel.update_display_with_hints` exists
   (`_agent_display_hints.py:91`) but is not listed in `docs/perf_runbook.md`, and there is no span at the action level,
   so the keypress → bar-visible interval cannot be read out of `tui_trace.jsonl` today.

### Measurements taken while planning

Measured with the repo virtualenv against real agent transcripts under the local artifacts tree, at console width 120.
`HINT` is `append_text_with_file_hints` into a fresh `Text` plus a full Rich render of that `Text`; `NORMAL` is
`lazy_renderable(..., "markdown", render_cache=...)` rendered once cold and once warm.

| content                   | HINT build | HINT render | NORMAL first | NORMAL cached |
| ------------------------- | ---------: | ----------: | -----------: | ------------: |
| real reply, 102.6 KiB     |     7.7 ms |     44.9 ms |      44.1 ms |        1.8 ms |
| real reply, 101.5 KiB     |     7.7 ms |     43.7 ms |      88.2 ms |        1.9 ms |
| real reply, 4.2 KiB (p50) |     0.4 ms |      2.1 ms |      16.5 ms |        0.9 ms |
| synthetic, 500 KiB        |    83.9 ms |    451.4 ms |            — |             — |
| synthetic, 1 MiB          |   161.4 ms |    855.8 ms |            — |             — |
| synthetic, 2 MiB          |   325.7 ms |   1779.1 ms |            — |             — |

Local `live_reply.md` size distribution across 3252 files: p50 4.3 KB, p90 11.4 KB, p99 57.7 KB, max 105 KB.

Read this as: a single hint render of the largest single agent transcript costs roughly 50 ms, the same repaint on the
normal path costs under 2 ms because it is cached, and the uncapped path has no ceiling — family containers, follow-up
chains, and consolidated reply views concatenate several member transcripts into one uncapped document, which is where
the synthetic 500 KiB–2 MiB rows become reachable. Multiply the per-render cost by the rebuild sources in defects 4 and
5 and the observed "hints load slowly" follows.

**These numbers are a planning baseline from isolated component benchmarks, not an end-to-end keypress measurement, and
they do not yet apportion cost between the annotated render and `build_header_text`.** The `measure` phase exists to
establish the real end-to-end split before any optimization lands, and every later phase reports its own before/after
against that baseline rather than against this table.

## Non-goals

- Changing which paths are recognized as hints (`FILE_PATH_RE` semantics stay as-is).
- Changing hint numbering order, the `@`/`%` suffix grammar, or any `HintInputBar` interaction.
- Touching the ChangeSpecs-tab branch of `action_view_files` beyond what shared code requires.
- Reworking `build_header_text` or the detail-header enrichment worker themselves; both are shared with the normal path
  and are out of scope except where hint mode uses them differently.
- Moving anything into the Rust core. This is presentation-layer Textual/Rich state, which `rust_core_backend_boundary`
  explicitly leaves in this repo.

## Phase 1 — `measure`: Instrument and baseline the view-hints keypath

Establish where the time actually goes before changing behavior, per `sase/memory/tui_perf.md` ("Measure, don't guess").

Work:

- Add `tui_trace` spans covering the whole keypath, so the keypress → bar-visible interval is readable:
  - `agents.view_files` around `action_view_files` (`src/sase/ace/tui/actions/hints/_files.py:17`),
  - `agents.view_agent_files` around `_view_agent_files` (`:65`), with a separate inner span around the `HintInputBar`
    mount so bar latency is distinguishable from render latency,
  - `agents.view_hints_refresh` around `_render_agent_detail_with_hints`
    (`src/sase/ace/tui/actions/agents/_display_detail_render.py:245`).
- Add counters to the existing `widget.prompt_panel.update_display_with_hints` span (`_agent_display_hints.py:91`):
  annotated character count, hint count, commit-view count, whether the detail-header summary was warm or cold, and
  whether the row is a family container. The warm/cold counter is what confirms or refutes defect 5.
- Add a `view_hints` scenario to `tests/perf/bench_tui_trace.py` driving the app through `Pilot` the way the existing
  scenarios do, covering: (a) `v` on a plain agent with a large reply, (b) `v` on a family container row, (c) a second
  `v` press on the same row, (d) an auto-refresh tick with hint mode active. Extend `tests/perf/fixtures.py` only as
  needed for the family-container case.
- Write the baseline numbers JSON into `tests/perf/baselines/` alongside the existing baselines so later phases diff
  against a committed artifact rather than a transient run.
- Document the new spans, and the pre-existing `widget.prompt_panel.update_display_with_hints` span, in the span list in
  `docs/perf_runbook.md`, plus a short subsection explaining how to read a view-hints capture.

Constraints:

- No behavior change. Spans are near-zero-cost no-ops when `SASE_TUI_TRACE` is unset, and the new bench is
  `pytest.mark.slow` so `just test` stays fast.
- Record in the phase's commit message which defect numbers above the baseline confirms and which it refutes. Later
  phases must be free to drop a fix whose premise the data kills; say so explicitly rather than implementing it anyway.

## Phase 2 — `bound`: Bound hint-mode content and remove per-fragment disk work

Give the hint path the same ceiling the normal path already has, so worst-case cost stops being unbounded.

Work:

- Add a small shared helper (a new `src/sase/ace/tui/widgets/prompt_panel/_hint_caps.py`, or an addition to
  `_file_path_hints.py`) that bounds one content fragment before it is scanned and appended, reusing the caps in
  `src/sase/ace/tui/util/lazy_syntax.py` (`PLAIN_RENDER_MAX_BYTES`, `PLAIN_RENDER_MAX_LINES`) rather than inventing new
  constants. Reuse `_truncate_plain_content` if it can be exported cleanly; do not copy it.
- Route every `append_text_with_file_hints` call site in `_update_display_with_hints_impl` (`_agent_display_hints.py`)
  and in `_render_reply_with_hints` (`:38`) through that helper, and give the truncated tail an explicit dim-italic
  notice in the same voice as `lazy_renderable`'s ("… N more lines — hints not generated past this point"), so an elided
  hint reads as elided rather than missing.
- Apply the same bound to the family hint fragments in `_agent_display_family_render.py` (`_family_text_with_hints`,
  `_family_reply_renderables_with_hints`), and additionally bound the **total** annotated budget across family members
  so a wide family cannot multiply the per-fragment cap.
- Hoist `_family_member_hint_workspace` (`_agent_display_family_render.py:297`) out of the per-chunk loop: resolve at
  most once per family member, not once per reply chunk.
- Memoize `resolve_agent_workspace_dir` (`_file_path_hints.py:42`) on `(workspace_num, project_file, workspace_dir)`
  with a bounded cache, and memoize `resolve_file_path` (`:94`) on `(path, workspace_dir)`. Both are pure given their
  inputs within a render.

Constraints:

- The cap must apply to the _hint-generating scan_, not just to what is displayed; a hint number the user can see must
  always resolve to a path.
- Numbering stays contiguous and stable: truncation removes hints from the tail, never renumbers the head.
- Keep `tests/ace/tui/widgets/test_agent_display_family.py`, `tests/ace/tui/widgets/test_agent_display_xprompt.py`, and
  `tests/ace/tui/widgets/test_agent_display_artifact_file_metadata.py` green; add coverage for the truncation notice and
  for hint-number stability across the cap boundary.
- Report the bench delta for the family-container and large-reply scenarios against the `measure` baseline.

## Phase 3 — `offpump`: Paint the hint bar before rendering the annotated document

Make the keypress feel instant regardless of what the render costs, per rules 1–3 of `sase/memory/tui_perf.md`.

Work:

- Restructure `_view_agent_files` (`src/sase/ace/tui/actions/hints/_files.py:65`) into two steps:
  1. **Synchronous, on the keypress:** resolve the selected agent, set `_hint_mode_active`, and mount the `HintInputBar`
     into `#agent-detail-container`. This is the only work the user waits on.
  2. **Deferred:** run the annotated render through `spawn_pump_free_task` (`src/sase/ace/tui/util/pump_tasks.py:64`)
     with a named registry attribute, and publish `_hint_mappings` / `_hint_commit_views` / `_hint_tool_call_reports`
     when it completes. Cancel the task via `cancel_pump_free_tasks` on teardown, on tab switch, and when the hint bar
     is removed.
- Add a readiness guard so an early submission is still correct: `on_hint_input_bar_submitted`
  (`src/sase/ace/tui/actions/hints/_processing.py:97`) must not evaluate a selection against half-built mappings. Await
  the pending render's completion (or re-dispatch the submission once it lands) instead of validating against a partial
  `_hint_mappings`. Re-capture the selected agent and current tab after the await, per rule 4 of
  `sase/memory/tui_perf.md`.
- Preserve the existing "no files or commits found" warning (`_files.py:80-89`): with the bar mounted first, that
  determination now happens after the render, so the empty case must unmount the bar and notify rather than silently
  leaving an unusable bar mounted. Do the same for the `header_enrichment_pending` case, which today deliberately keeps
  the bar open while enrichment is in flight.
- Keep `_refocus_existing_hint_bar` (`_types.py:64`) working: a second `v` while a bar is mounted must still refocus
  rather than mount a duplicate, including while the deferred render is still running.

Constraints:

- The `ChangeSpecs`-tab branch of `action_view_files` keeps its current synchronous shape unless shared code forces
  otherwise; this phase is Agents-tab scoped.
- `tests/ace/tui/test_hint_input_bar_duplicate_guard.py`, `tests/ace/tui/test_agents_view_hint_survives_refresh.py`, and
  `tests/ace/tui/actions/test_view_files_image.py` must stay green. Add a test that submits a hint selection immediately
  after `v`, before the deferred render resolves, and asserts the correct file is opened.
- Report the bench delta for keypress → bar-visible specifically; that is this phase's headline number.

## Phase 4 — `cache`: Cache the annotated hint document

Make every repaint after the first one cheap, matching what `_CachedRenderable` already does for the normal path.

Work:

- Add a bounded, panel-local cache for the hint render, keyed on everything that can change the document: agent
  identity, the fold level and section fold overrides from `panel_fold_state_from_widget`, digests of the annotated
  source content (xprompt / prompt / reply chunks), the detail-header summary's cache key, the cap parameters introduced
  in `bound`, and the attempt view mode / pinned attempt number. Follow the shape of `_detail_header_summary_cache`
  (`_agent_display_header_summary.py:40`) — `OrderedDict`, bounded entries, LRU eviction — rather than inventing a new
  caching idiom.
- On a hit, reuse both the `AgentHintRender` result and the built renderable.
- Wrap the annotated document in the same width-keyed segment cache the normal path gets, so a repaint at an unchanged
  console width reuses rendered segments. Prefer reusing `_CachedRenderable` (`src/sase/ace/tui/util/lazy_syntax.py:82`)
  — promote it out of module-private if needed — over a parallel implementation.
- Clear the cache where the other panel caches are cleared: `show_empty` (`_agent_display_render.py:447`) and on
  agent-identity change in `_reset_markdown_render_cache_for_agent` (`:77`).

Constraints:

- An over-broad key serves stale hint numbers, which is worse than being slow — a hint that resolves to the wrong file
  is a correctness bug. Prefer a key that is too narrow (extra misses) over one that is too broad, and add a test that a
  changed reply chunk produces a changed document.
- Do not cache across attempt-pinned views; `attempt_pinned_number` is part of the key.
- Report cold-vs-warm `v` timings from the bench's repeat-press scenario.

## Phase 5 — `dedupe`: Stop redundant hint re-renders on refresh and enrichment

Remove the rebuild sources rather than only making each rebuild cheaper.

Work:

- In `_render_agent_detail_with_hints` (`src/sase/ace/tui/actions/agents/_display_detail_render.py:245`), skip the
  rebuild when the cache key from `cache` is unchanged, so an Agents-tab refresh with no relevant change does not touch
  the hint document at all.
- In `on_agent_detail_header_enriched` (`:253`), repaint only when the newly cached summary actually changes the header
  — that is, when it adds or changes a SASE CONTEXT section that carries hints. An enrichment result identical to the
  one already rendered must not rebuild the document.
- Stop the one-second `DIFF_CACHE_TTL_SECONDS` staleness window (`src/sase/ace/tui/widgets/file_panel/_diff.py:53`) from
  re-arming the enrichment worker on every hint repaint while hint mode is active. Either gate
  `_start_agent_detail_header_enrichment_from_context` (`_agent_display_async_agent.py:64`) on a longer cadence while a
  hint bar is mounted, or suppress re-enrichment entirely for the lifetime of one hint session. Prefer the former if a
  running agent's artifacts can grow mid-session; state which you chose and why, per rule 10 of
  `sase/memory/tui_perf.md` (ticks revalidate, recomputes get a longer cadence).
- Consider warming the detail-header summary when a row is selected rather than when `v` is pressed, so the common case
  enters hint mode with a warm summary and renders complete hints on the first pass. Only do this if the `measure`
  baseline showed cold-summary presses are actually common; drop it otherwise and say so.

Constraints:

- `tests/ace/tui/test_agents_view_hint_survives_refresh.py` encodes the existing guarantee that hints survive a refresh
  — that guarantee must hold. Suppressing a rebuild is only correct if the already-rendered document is still right.
- A running agent whose reply grows while the hint bar is open must still pick up new hints; the content digests in the
  `cache` key are what make that work, so do not gate on identity alone.

## Phase 6 — `verify`: Verify the speedup and guard it

Work:

- Re-run `pytest -s -m slow tests/perf/bench_tui_trace.py` for the `view_hints` scenarios, diff against the committed
  `measure` baseline, and record the before/after table in the commit message.
- Add a regression floor for the view-hints scenarios in the same style as the existing perf regression checks
  (`tests/perf/check_agent_launch_regression.py`, `tests/perf/phase7_check_regression.py`), with a variance allowance
  appropriate for a Pilot-driven run.
- Run a short interactive soak with `SASE_TUI_TRACE=1` and lowered `SASE_TUI_HITCH_THRESHOLD_SECONDS`, exercising `v` on
  a large agent and on a family container, and confirm `~/.sase/logs/tui_stalls.jsonl` records no new hitch attributable
  to the hint path.
- If `bound` made truncation user-visible, update the `?` help popup entry for `v`
  (`src/sase/ace/tui/modals/help_modal/agents_bindings.py:95`) — required by `src/sase/ace/CLAUDE.md` — and the perf
  runbook subsection added in `measure`.
- Update `CHANGELOG.md` if the repo's release tooling expects an entry for a user-visible change.

## Success criteria

- Keypress → hint bar visible is bounded by widget mount cost and no longer scales with transcript size, family member
  count, or agent count.
- A second `v` press on an unchanged row, and an auto-refresh tick with hint mode active, do not rebuild the annotated
  document.
- Worst-case annotated render is bounded by explicit caps shared with the normal render path, with elided hints shown as
  elided.
- Every hint number the user can see resolves to the same path it resolves to today.
- The `view_hints` bench scenarios show a measured improvement against the committed baseline, and a regression floor
  guards it.

## Repo conventions the phase agents must follow

- Run `just install` before `just check` in a fresh workspace, and run `just check` before reporting done — both are
  required by the repo's `CLAUDE.md`.
- Read `sase/memory/tui_perf.md` through the `sase_memory_read` skill before touching any render or keystroke path; do
  not read the canonical memory file directly.
- Read `sase/memory/symvision.md` through the same skill if Symvision reports unused or private-misuse symbols after
  promoting `_CachedRenderable` or adding new helper modules.
- Do not edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims. This plan does not grant
  that permission.
