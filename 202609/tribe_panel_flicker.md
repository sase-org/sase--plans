---
tier: tale
title: Stop selected-tribe Main deck flicker and typing lag
goal:
  With a tribe panel selected, background refreshes never blank or re-scroll the Main
  deck or jump panel, no-op refreshes cost at most one cheap UI-thread pass, and tribe
  repaints stay out of the way while the prompt input is active.
size: medium
proposed_by: bbugyi200.athena.0tq
create_time: 2026-09-28 15:39:08
status: wip
---

# Stop selected-tribe Main deck flicker and the typing lag it causes

## Problem

With an agent tribe panel selected on the Agents tab, the Main deck's Summary card
periodically disappears and reappears even though nothing about the tribe changed. The
user also reported lag while typing in the prompt input widget.

## Diagnosis (measured, not guessed)

These results come from a headless `AceApp.run_test()` harness built on the existing
selected-tribe benchmark fixture (`tests/ace/tui/_bench_tui_jk_helpers.py`:
`_install_agents_fixture`, `_wait_for_startup`). It hooked
`AgentPromptPanel.update_tribe_display`, the Main-deck
`CardDocumentView._apply_section_content`, and `AgentJumpPanel.show_jump_map`.

### Root cause 1 (primary flicker): every background refresh repaints a cheap placeholder over the complete tribe document

- Background agent-list refreshes reach the display with `defer_detail=True`. These
  include the finalize path (`_loading_finalize.py` →
  `_refresh_agents_display_after_finalize`), the tool-runs loader, unread fallbacks,
  proc completion, and others. Both `_refresh_agents_display_impl` and
  `_try_refresh_agents_display_incremental_impl` in
  `src/sase/ace/tui/actions/agents/_display.py` then call
  `_apply_tribe_summary(cheap=True)` whenever a tribe panel is selected. They do this
  even when the selected tribe is the one already on screen and no agent changed.
- In deck mode (detached identity), `build_tribe_detail_text(cheap=True)` renders a body
  of only the description, or `⋯ loading…`. It also publishes **no member jump map**.
- Measured: an idle finalize refresh with zero agent changes changed the Main deck from
  21 lines to `⋯ loading…` (1 line), then back to 21 lines about 170 ms later when the
  150 ms debounced full paint landed.
- Measured: the jump panel (TRIBE MEMBERS roster) received `jump_map=None`. Its targets
  went from 12 to 0 and it was hidden, then re-shown about 180 ms later. The whole deck
  area re-lays out twice per refresh.
- Measured: the Main deck scroll position was reset every time, going from 5 to 0 and
  from 40 to 0. The placeholder shrinks the content, so the scroll offset clamps to 0,
  and the full document comes back scrolled to the top.
- The cheap tribe path exists for selection changes, where a placeholder beats showing
  the wrong subject. It is wrong for same-subject refreshes, where the complete document
  already on screen is the better placeholder.

### Root cause 2 (typing lag): redundant full rebuilds and relayouts of a potentially huge document on the UI thread, which also drive gen-2 GC pauses

- With a heavy synthetic tribe document of about 5,000 rows, each idle refresh blocked
  the event loop for 25–60 ms per paint. There are two paints per refresh, because the
  placeholder and the full document each force a Textual relayout and re-render of the
  whole document plus the jump-panel show/hide.
- cProfile: most of the time is spent in Textual `_refresh_layout` / `render_update` /
  `Strip` construction. `update_tribe_display` costs about 32 ms per call, prompt source
  `update` (digest + flatten) about 22 ms, and `_apply_agent_footer_update` about 14 ms
  per call.
- The 300–540 ms freezes that appeared during the run were **gen-2 garbage collections**
  (a `gc.callbacks` probe logged `GC gen2 298–408 ms` exactly at each loop-lag spike).
  They are driven by the allocation churn of repeatedly rebuilding and re-rendering the
  document. Removing the redundant rebuilds reduces how often they trigger.
- Other sources force extra full rebuilds even when nothing visible changed:
  - `_apply_tribe_section_enrichment_result` posts `TribeSectionSnapshotLoaded` for
    every successful worker result, including identical ones. The disk TTL is 10 s and
    the stats TTL is 60 s, so this triggers a full rebuild plus digest.
  - The debounced full paint rebuilds the entire document on every refresh, even when
    its inputs are unchanged.
- While the prompt input bar is mounted, auto-refresh and the watcher are already gated
  (`_prompt_input_active()`). However, tribe repaints triggered by enrichment
  completions and notification unread changes (`_refresh_tribe_summary_only`) still land
  mid-typing. `_fire_debounced_detail_update` only honors the navigation gate. This
  violates the TUI perf rule "respect activity gates… or typing in the prompt input".

### Root cause 3 (secondary flicker): membership churn blanks the disk-backed sections

- `prepare_tribe_section_snapshot` (`_agent_tribe_aggregation.py`) drops the cached
  `disk` snapshot (PROMPTS, REPLIES, SLOW TOOL CALLS) whenever the tribe's source
  signature changes. The signature includes every concrete agent-session turn row, so
  any member starting a new turn, spawning, or being dismissed makes those sections
  vanish (showing `⋯ scanning member data…`) until the worker reloads them. The CLAN
  SUMMARIES section already avoids this ("Membership churn never blanks the section").

### User's suspicion: confirmed

The flicker is real and happens on no-op refreshes. The associated lag is real: each
refresh does two UI-thread paint and relayout passes instead of zero or one, and
background repaints are not gated while the prompt input is open.

## Scope

Everything here is presentation-only Textual refresh and paint behavior. It stays in
this repo (no sase-core change). No keymap or `default_config.yml` change. No memory
note edits.

## Implementation

### 1. Never repaint a cheap tribe placeholder over the same complete document (fixes RC1)

- Add explicit state to `AgentDetail` recording whether the currently shown tribe
  document is complete:
  - In `_deck_show_tribe_summary`
    (`src/sase/ace/tui/widgets/_agent_detail_deck_refresh.py`), set a
    `_tribe_document_complete` flag to `not cheap`. Initialize it to `False` in
    `AgentDetail.__init__`.
  - Every path that clears `_current_tribe_identity` (`_deck_show_empty`, the agent
    display paths in `_agent_detail_display.py`) implicitly invalidates it. Also reset
    the flag there for clarity.
  - Expose a public predicate, for example
    `AgentDetail.shows_complete_tribe_document(identity) -> bool`. It returns
    `self._current_tribe_identity == identity and self._tribe_document_complete`.
- In `AgentDetailRenderMixin._apply_tribe_summary`
  (`src/sase/ace/tui/actions/agents/_display_detail_render.py`), when `cheap=True`,
  first resolve `self._focused_tribe_panel_context()`. If it is not `None` and
  `agent_detail.shows_complete_tribe_document(focus.container_identity)` is true, return
  `True` without building the snapshot, touching the document, or updating the footer.
  Resolve the predicate via `getattr(..., None)` + `callable` so the existing test fakes
  that lack it keep working. The callers already schedule
  `_fire_debounced_detail_update`, and that full paint updates the footer.
- This covers all cheap call sites:
  - the two `defer_detail` branches in `_display.py`;
  - `_apply_agent_detail_immediate`, used by `_refresh_agent_focus_detail`, startup tag
    and prompt-catalog loads, and unread projection.
- Selection changes to a _different_ tribe, or from an agent row to a tribe, still get
  the cheap placeholder, because the identity differs. Keep
  `test_collapsed_panel_tribe_uses_cheap_then_debounced_document`,
  `test_selected_tribe_navigation_defers_even_the_cheap_document`, and
  `test_debounced_document_waits_for_navigation_gate_to_quiesce` passing.
- Result: same-subject refreshes keep the Main card, the jump panel, and the scroll
  offset stable. The debounced full paint then applies real changes only (the digest
  dedups identical content).

### 2. Skip rebuilding an unchanged complete tribe document (RC2)

- In `AgentPromptPanel.update_tribe_display`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display.py`), for `cheap=False`, run
  `prepare_tribe_section_snapshot(...)` as today, then build a render-input key from:
  - the `AgentTribeSummarySnapshot` (a frozen value dataclass; compare with `==`);
  - the cached `TribeSectionSnapshot` (`==`; unchanged objects short-circuit on
    identity);
  - the fold level and a frozen view of the fold overrides;
  - `detaches_identity_header`;
  - `publish_member_jump_map`;
  - the app theme name.
- If the key equals the last complete render's key **and**
  `self._section_content_digest` still equals the digest that render produced (so
  nothing else has been shown since), skip `build_tribe_detail_text`, the digest, and
  `self.update(...)`.
- Still run the enrichment scheduling tail (`start_tribe_section_enrichment`) on a skip,
  so TTL refreshes keep working.
- Store the key and digest only after a complete (non-cheap) update is applied. Any
  cheap tribe update or non-tribe update must not match. Rely on the digest check and
  clear the memo in `update_display` / `update_header_only` / empty paths.
- `watch_theme` must still force a rebuild. The theme in the key covers this; add a
  test.

### 3. Do not repaint for no-op enrichment results (RC2)

- In `_apply_tribe_section_enrichment_result`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_async_groups.py`), capture the
  cached snapshot before `cache_tribe_enrichment`. Post `TribeSectionSnapshotLoaded`
  only when a renderer-facing field changed: `disk`, `runtime_statistics_loaded`,
  `runtime_statistics`, or `clan_summaries`. Timestamps and clan updates must still be
  cached, and the pending-request relaunch logic must stay unchanged.

### 4. Defer background tribe repaints while the prompt input is active (RC2)

- In `_fire_debounced_detail_update`, after the existing navigation-gate check, add:
  when a tribe panel is focused (`_focused_tribe_panel_context() is not None`) and
  `getattr(self, "_prompt_input_active", None)` reports active, reschedule through
  `self._agent_detail_debouncer.schedule(self._fire_debounced_detail_update)` and
  return. This mirrors the nav-gate retry and keeps the pump callback thin.
- In `_refresh_tribe_summary_only`, when the prompt input is active and a tribe summary
  would render, schedule the debounced update instead of rendering immediately. Return
  `True` so callers do not fall through to other paths.
- Scope the deferral to tribe focus only. Agent-row detail behavior is unchanged.

### 5. Keep disk sections through membership churn (fixes RC3)

- In `prepare_tribe_section_snapshot`, when a cached entry exists but the source
  signature changed:
  - keep `cached.snapshot.disk` as stale content instead of `None`;
  - set `disk_enriched_monotonic=None` so `tribe_sections_to_refresh` requests a reload;
  - reset `loading_sections` as today.
- Worker results are still merged only when their `source_signature` matches the current
  entry (unchanged), so the stale disk is replaced on the next worker result.
- To avoid showing departed units, have the REPLIES and SLOW TOOL CALLS renderers
  (`_agent_display_tribe_sections.py`) and PROMPTS (`_agent_display_tribe_prompts.py`)
  drop entries or group members whose `unit_identity` is not in the current
  `snapshot.units`. Drop prompt groups left with no members. Mirror the `present_units`
  argument already used by `append_clan_summaries` in `_append_tribe_body`.

### 6. Tests

Add them alongside the existing files.

- `tests/ace/tui/test_agent_detail_two_phase.py`, using the `_SummaryApp` fakes:
  - when the fake detail reports a complete document for the focused identity,
    `_refresh_agents_display_debounced()` and the `defer_detail` refresh branches issue
    no cheap `show_tribe_summary` call, and the debouncer is pending;
  - a different identity still receives the cheap call;
  - `_fire_debounced_detail_update` defers while `_prompt_input_active()` is true and
    renders once it is false;
  - `_refresh_tribe_summary_only` defers while the prompt input is active.
- `AgentDetail` / deck tests:
  - `shows_complete_tribe_document` is true only after a non-cheap `show_tribe_summary`
    for that identity;
  - it is false after `show_empty` or an agent display.
- Aggregation tests next to the existing tribe aggregation / summary tests (for example
  `tests/ace/tui/widgets/test_agent_tribe_summary.py`):
  - a source-signature change retains the previous `disk` and forces a refresh request;
  - a matching worker result replaces it;
  - departed units' replies, slow calls, and prompt members are not rendered;
  - an identical enrichment result posts no `TribeSectionSnapshotLoaded`;
  - a changed one does.
- `update_tribe_display` memo:
  - an unchanged key skips `update()`;
  - a changed section snapshot, fold level, theme, or intervening non-tribe update
    forces a rebuild.
- One Pilot regression test with
  `AceApp(query="!!!", auto_start_axe=False, refresh_interval=0)`,
  `_install_agents_fixture(app, count=48)`, focusing a tribe panel with attention
  entries (press `J` once, then `app._activate_focused_panel()`), raising folds (`z`,
  `z`) so the Main document scrolls, and scrolling the Main view down. Then call
  `app._refresh_agents_display_after_finalize(previous_agents=list(app._agents), defer_detail=True)`
  and pause about 0.5 s. Assert that:
  - no Main-deck paint in that window contains `⋯ loading…` (hook or inspect
    `CardDocumentView` content);
  - the jump panel never lost its targets or gained `hidden`;
  - the Main scroll offset is unchanged. Repeat once with
    `_refresh_agents_display(list_changed=True, defer_detail=True)`. Keep it out of the
    `slow` marker if it runs in a few seconds, like other Pilot tests.

### 7. Measure before and after

Use the same approach as the diagnosis: a scratch Pilot harness, not committed.

- Monkeypatch `AgentPromptPanel._build_tribe_enrichment` to add about 60 replies of 40
  lines each.
- Raise folds to EXHAUSTIVE.
- Run a 1 ms `asyncio.sleep` loop-lag probe plus a `gc.callbacks` probe around 3 idle
  finalize refreshes.
- Report per-refresh max loop lag, paints per refresh, and gen-2 GC count, before and
  after, in the final summary.
- Expected result: one or zero Main paints per no-op refresh (was two), no placeholder,
  preserved scroll, and fewer allocation-driven GCs.

### 8. File follow-ups (not part of this change)

Use `/sase_new_task` for each; it checks for duplicates.

- App-wide gen-2 GC pauses. The harness showed 300–540 ms pauses. The user's live
  `~/.sase/logs/tui_stalls.jsonl` shows about 150 `tui_hitch` events today on the Agents
  and Artifacts tabs (p50 about 2.9 s) with scattered, allocation-site-looking innermost
  frames, on a host with load average around 18. Investigate GC tuning (for example
  `gc.freeze()` after startup, or thresholds) with measurements.
- Uncached per-member `git` subprocesses in the tribe/clan enrichment worker:
  `clan_member_source_token` → `_audit_log_paths` → `project_memory_name` →
  `_run_git_stdout` runs up to 2 `git` processes per member on every disk refresh (at
  least every 10 s while a tribe is selected) when no checkout marker exists. Verify on
  real workspaces, then cache it.
- UI-thread project-alias `is_file()` scans seen in live hitch stacks
  (`rearm_live_agent_watch_coverage` / `hydrate_agent_attempt_history` →
  `get_artifacts_dir` → `load_project_alias_map` → `_project_record_has_spec`).

## Verification

- `just check` (the agent default). Do not run `just check-full` unless explicitly
  instructed.
- Run the new and touched tests directly with pytest first, for example
  `tests/ace/tui/test_agent_detail_two_phase.py` and the tribe summary / aggregation
  tests.
- If any deck or Agents-tab visual snapshot is affected, follow the documented visual
  snapshot workflow. None is expected, since the settled document is unchanged.
