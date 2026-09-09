---
status: done
tier: epic
title: Fix ACE TUI freezes and prompt-input lag
goal: "The ACE TUI stays responsive while typing in the prompt input and while
  navigating the Agents tab: the main thread stops burning CPU on redundant render and
  measurement passes, the one-second countdown tick stops doing unbounded synchronous
  work on the Textual pump, keystroke paths stop doing per-project disk I/O, and
  background workers stop starving the event loop. Key-to-paint p95 on the Agents tab
  returns under the 16 ms budget and the stall watchdog stops recording multi-second
  hitches during ordinary use.

  "
phases:
  - id: section_visual
    title: Stop the prompt panel double-render and cache its section anchors
    depends_on: []
    size: medium
    description: "section_visual: memoize prompt-panel section anchors per (generation,
      width) and stop SectionTrackingVisual.get_height from running a second full Rich
      console render on every measurement pass.

      "
  - id: countdown_gate
    title: Gate the countdown tick on prompt typing, not just j/k
    depends_on: []
    size: small
    description: "countdown_gate: extend the activity gate so the one-second countdown
      tick defers its Agents-tab repaint work while the user is typing in the prompt
      input, matching the documented tui_perf activity-gate rule.

      "
  - id: config_token
    title: Stop per-tick config-token thread churn and per-key token lookups
    depends_on: []
    size: small
    description: "config_token: move the refresh-thread spawn out of the config-token
      cache lock, raise the revalidation interval above the tick cadence, and resolve
      tribe displays once per call instead of once per panel key.

      "
  - id: prompt_completion
    title: Take per-project disk I/O off the prompt completion keystroke path
    depends_on: []
    size: medium
    description: "prompt_completion: cache project workflow-type and changespec-name
      lookups across calls and keep the debounced soft-completion timer callback free of
      synchronous per-project file reads.

      "
  - id: artifact_index
    title: Index artifact link targets instead of scanning them per ref
    depends_on: []
    size: small
    description: "artifact_index: replace the linear known-target scan in
      _known_target_for_ref with a prebuilt index so patch loading stops burning worker
      CPU and stealing the GIL from the event loop.

      "
  - id: perf_guards
    title: Regression guards for the repaired hot paths
    depends_on:
      - section_visual
      - countdown_gate
      - config_token
      - prompt_completion
      - artifact_index
    size: medium
    description:
      "perf_guards: add benches and unit guards that fail if the prompt panel
      double-renders, the countdown tick ignores typing, the config token spawns a
      thread per tick, the completion path reads project files, or the artifact link
      lookup returns to a linear scan."
proposed_by: bbugyi200.athena.0fe
bead_id: sase-v2
create_time: 2026-09-09 19:51:58
---

- **PROMPT:**
  [prompts/202608/tui_freeze_regression.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/tui_freeze_regression.md)
- **BEAD:**
  [sase-v2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-v2/README.md)

# Plan: Fix ACE TUI freezes and prompt-input lag

## Problem

The ACE TUI (`sase ace`) intermittently becomes unresponsive, with severe lag while
typing in the prompt input widget. This plan is written from forensics on a live,
reproducing TUI process, not from inspection alone.

## Evidence

All measurements below come from one live `sase ace` session on the reporter's machine
(64 cores, load average ~8, so the host was **not** saturated) plus its always-on
instrumentation logs.

### The TUI process is CPU-bound on its own main thread

`ps` showed the TUI at **179% CPU** and **2.3-2.5 GB RSS** with 24 threads while sitting
on the Agents tab. A 25-second `py-spy record --threads` sample of the main thread
(18.71 s of attributed samples) breaks down as:

| Share           | Frame                                                                                                                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 68.3% (12.78 s) | `textual/_compositor.py render_update` -> `_render_chops` -> `widget.render_lines` -> `_render_content` -> `visual.to_strips` -> `strip._apply_link_style` -> `Strip.__init__` -> `FIFOCache.__init__` |
| 18.3% (3.42 s)  | `src/sase/ace/tui/widgets/prompt_panel/_section_navigation.py:85` `SectionTrackingVisual.render_strips`                                                                                                |
| 8.6% (1.60 s)   | `src/sase/ace/tui/widgets/prompt_panel/_section_navigation.py:125` `SectionTrackingVisual.get_height`                                                                                                  |

`FIFOCache.__init__` alone was **70.7% of main-thread self time** — the compositor is
repainting full widget lines continuously rather than serving cached strips.

### Key-to-paint is far over budget

`~/.sase/perf/tui_jk.jsonl` (`model_ms + paint_ms`), last 400 samples:

| tab/action    | n   | p50     | p95      | max       |
| ------------- | --- | ------- | -------- | --------- |
| agents / next | 281 | 23.9 ms | 206.3 ms | 5140.8 ms |
| agents / prev | 119 | 21.8 ms | 208.5 ms | 425.1 ms  |

`sase/memory/tui_perf.md` sets the target at **p95 < 16 ms on every tab**. p95 is 13x
over budget and even p50 misses it.

### The stall watchdog recorded 12 multi-second hitches in 6.5 minutes

`~/.sase/logs/tui_stalls.jsonl` for the reproducing process, window 08:38:36 to 08:45:09
— 12 `tui_hitch` records totalling **39.7 s of stall**, with six over 4.8 s. Grouped by
`last_action`: `text_area_changed` 5, `j` 3, `k` 2, `ctrl+l` 1, `left_square_bracket` 1.
The `text_area_changed` majority is the reported prompt-input lag.

Four distinct blocking call chains were captured (paths repo-relative):

1. **Keystroke path doing per-project file reads** (5.43 s,
   `last_action=text_area_changed`): `widgets/_prompt_soft_completion.py:129`
   `set_timer` lambda -> `_fire_prompt_completion_timer:165` ->
   `widgets/prompt_completion_root.py:29` `resolve_prompt_completion_base_dir` ->
   `_normalize_prompt_refs:58` -> `project_aliases.py:282` ->
   `project_alias_prompts.py:140` -> `replace:119` -> `workflow_type_for:71` ->
   `project_aliases.py:266 _project_workflow_type` ->
   `workspace_provider/_registry.py:187 detect_workflow_type` -> pluggy hook ->
   `workspace_provider/utils.py:133 parse_bare_repo_dir` -> `open(project_file)`.

2. **Countdown tick resolving the config token per panel key** (5.19 s and 4.88 s):
   `actions/_event_countdown.py:44` -> `actions/agents/_display_detail_info.py:40,140`
   -> `actions/agents/_selection.py:238 _get_selected_agent` ->
   `_selection.py:50 _resolve_focused_panel` ->
   `actions/agents/_panel_fold_intent.py:30 panel_is_collapsed` ->
   `models/tribe_display.py:200 effective_collapsed_panel_keys` ->
   `tribe_display.py:115 tribe_display_for` -> `tribe_display.py:105 _tribe_displays` ->
   `config/core.py:262 current_config_token` -> `refresh_thread.start()`.

3. **Countdown tick walking the agent tree per row** (5.05 s):
   `actions/_event_countdown.py:45 _patch_agent_runtime_rows` ->
   `actions/agents/_display_panel_patches.py:215` ->
   `widgets/agent_list.py:470 patch_active_runtime_rows` ->
   `models/agent_time.py:657 row_runtime_or_wait_ticks` (recursive) ->
   `agent_time.py:376 _is_family_shell` -> `models/agent.py:278 is_gate` ->
   `gate_shell/state.py:57` -> `shells/state.py:42` ->
   `plan_chain.py:326 agent_family_role_for_suffix`.

4. **Live `py-spy dump`** caught the main thread holding the GIL in
   `actions/agents/_display_detail_info.py:93 _selected_agent_neighbor_count`, again
   under `_on_countdown_tick`.

### Background workers steal the GIL

In the same 25 s window, worker thread 377332 burned **4.17 s** entirely inside
`ace/tui/relations/artifact_links.py _known_target_for_ref`, entered from
`actions/patch/_loading.py:182 _prepare_patch_load_from_disk`. Under CPython's GIL that
CPU-bound worker directly delays the event loop.

### Direct micro-benchmarks

Run against the same interpreter the TUI uses, on an otherwise idle process:

- `resolve_prompt_completion_base_dir("... #gh:sase ...")`: p50 **3.63 ms**, p95 **5.27
  ms**, cold **293 ms**. The reporter has **32** projects under `~/.sase/projects/`.
- `resolve_prompt_completion_base_dir("hello world")` (no `#`): ~0 ms — the early return
  at `prompt_completion_root.py:26` is the only reason typing is ever cheap.
- `current_config_token()`: ~0 ms warm. Its cost in production is contention and
  thread-spawn latency, not computation.

## Root causes

1. **`AgentPromptPanel.render()` returns a fresh `SectionTrackingVisual` on every call**
   (`widgets/prompt_panel/__init__.py:89-95`). A new wrapper object each repaint defeats
   Textual's visual-level caching, so every paint re-runs `render_strips` (a full
   segment scan of every strip) and every layout re-runs `get_height`.
2. **`SectionTrackingVisual.get_height()` runs a second full render**
   (`_section_navigation.py:113-131`): it calls `app.console.render(...)` over the
   entire renderable purely to count rows and harvest anchors. Textual calls
   `get_height` on every arrange pass, so the panel is rendered twice per layout and the
   result is never memoized on `(generation, width)`.
3. **The countdown tick's activity gate does not cover typing.**
   `actions/_event_countdown.py:42` checks only `self._nav_gate.is_navigating(...)`, and
   `util/nav_gate.py` records **only j/k** (`NavigationGate.record` is called from the
   navigation actions). `sase/memory/tui_perf.md` rule 13 requires deferring non-urgent
   refresh work while the user is _either_ mid-navigation _or_ typing in the prompt
   input; the typing half is missing. That is why `text_area_changed` is the most common
   `last_action` on recorded hitches.
4. **`current_config_token()` spawns a daemon thread while holding its cache lock**
   (`config/core.py:250-262`), and `_CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS` is **0.75
   s** (`config/core.py:125`) — shorter than the 1 s countdown tick, so _every_ tick
   finds the token expired and spawns a fresh thread. `Thread.start()` blocks until the
   new thread bootstraps, on the UI thread, inside the global lock. This is exactly the
   failure mode `tui_perf.md` rule 10 describes ("update checks did when tick interval
   equaled cache TTL"). Compounding it, `effective_collapsed_panel_keys` calls
   `tribe_display_for(key)` per candidate key and each call re-enters
   `_tribe_displays()` -> `current_config_token()`.
5. **The debounced prompt-completion timer callback does synchronous per-project disk
   I/O.** `_fire_prompt_completion_timer` is a `set_timer` callback, i.e. it runs on the
   Textual pump (`tui_perf.md` rule 2), and it calls
   `resolve_prompt_completion_base_dir` synchronously. Inside,
   `canonicalize_project_aliases_in_prompt` memoizes `workflow_type_for` and
   `changespec_names` only in **per-call** dicts (`project_alias_prompts.py:58-73`), so
   every keystroke re-reads project files for up to 32 projects. This violates
   `tui_perf.md` rule 11 (keystroke paths are read-only and prompt-free) and rule 8
   (cache disk reads keyed by mtime).
6. **`_known_target_for_ref` linearly scans every known target for every ref**
   (`relations/artifact_links.py:246-280`), making patch loading O(refs x targets) when
   the lookups it performs are exact-match on `(pane_id, part)` and could be served from
   a dict.

## Phases

Phases `section_visual`, `countdown_gate`, `config_token`, `prompt_completion`, and
`artifact_index` touch disjoint files and may run in parallel. `perf_guards` depends on
all five.

### Stop the prompt panel double-render and cache its section anchors

Target: `src/sase/ace/tui/widgets/prompt_panel/_section_navigation.py` and
`src/sase/ace/tui/widgets/prompt_panel/__init__.py`.

This is the single largest main-thread cost (~27% of samples across `render_strips` and
`get_height`, plus most of the compositor repaint above it).

- Give `AgentPromptPanel` a cached visual: `render()` must return the _same_
  `SectionTrackingVisual` instance while `_section_generation` is unchanged, and build a
  new one only when the generation advances. Keep the existing generation bump at
  `__init__.py:77` as the sole invalidation point.
- Memoize anchor collection on `(generation, width)` inside `SectionTrackingVisual`.
  Once anchors have been published for a width, neither `render_strips` nor `get_height`
  may recompute them.
- Remove the second full render in `get_height`. The current implementation calls
  `app.console.render(self._visual._renderable, options)` and walks every segment to
  count `\n`. Replace it with the wrapped visual's own
  `self._visual.get_height(rules, width)` for the height, and harvest Rich anchors from
  the paint pass in `render_strips` instead of from a separate measurement render. If
  anchors are genuinely required before first paint, compute them once per
  `(generation, width)` and reuse that result for both calls rather than re-rendering
  per measurement.
- Preserve current behavior for the `RichVisual` and non-`RichVisual` branches and keep
  `_publish_section_layout`'s generation guard (`__init__.py:138-147`) intact, including
  the `_preserve_missing_section_generation` case.

Verify with `pytest tests/ace/tui/` for the prompt panel and section-navigation suites,
and confirm section navigation (jumping between metadata sections) and fold behavior
still work.

### Gate the countdown tick on prompt typing, not just j/k

Target: `src/sase/ace/tui/actions/_event_countdown.py` and
`src/sase/ace/tui/util/nav_gate.py`.

- Introduce a typing-activity signal alongside `NavigationGate`. The app already records
  `_last_input_mono` in `_record_input_event` (`_event_countdown.py:52-58`); either
  extend `NavigationGate` with a separate typing window or add a sibling gate, but keep
  j/k and typing windows distinguishable so navigation tuning does not silently change
  typing behavior.
- Feed the gate from the prompt input's text-changed path — the same event the stall
  records label `text_area_changed` — so the gate is hot while the user types.
- In `_on_countdown_tick`, defer `_update_agents_info_panel()`,
  `_patch_agent_runtime_rows()`, and `_poll_starting_agent_transitions()` while either
  gate is active. The logical countdown decrement above must keep running, and the next
  quiet tick must catch all three surfaces up — preserve the existing comment's contract
  at `_event_countdown.py:36-41`.
- Do not silently extend the deferral to the `artifacts` or `axe` branches; those are
  out of scope for this phase.

### Stop per-tick config-token thread churn and per-key token lookups

Target: `src/sase/config/core.py` and `src/sase/ace/tui/models/tribe_display.py`.

- Move `refresh_thread.start()` out of the `_current_config_token_cache_lock` critical
  section in `current_config_token()` (`config/core.py:250-262`). Publish the thread
  handle under the lock, then start it after releasing, so a slow thread bootstrap never
  blocks another caller holding the UI thread.
- Raise `_CONFIG_TOKEN_REFRESH_INTERVAL_SECONDS` (`config/core.py:125`) above the TUI's
  1 s tick cadence so a periodic tick does not force a revalidation every time. Pick the
  new value deliberately and comment why it must exceed the tick interval; `tui_perf.md`
  rule 10 is the governing rule.
- In `tribe_display.py`, resolve the token and the display map **once per call** in
  `effective_collapsed_panel_keys` (`tribe_display.py:178-201`) and look up each
  candidate key against that single snapshot, instead of calling
  `tribe_display_for(key)` — and therefore `current_config_token()` — per key.
  `tribe_identity_colors` (`tribe_display.py:120-131`) already demonstrates the intended
  shape; follow it.
- Keep `clear_config_cache()` and the cache-generation/`CONFIG_DIR` rebinding semantics
  unchanged; tests depend on them.

### Take per-project disk I/O off the prompt completion keystroke path

Target: `src/sase/ace/tui/widgets/prompt_completion_root.py`,
`src/sase/project_alias_prompts.py`, and
`src/sase/ace/tui/widgets/_prompt_soft_completion.py`.

- Replace the per-call memo dicts in `canonicalize_project_aliases_in_prompt`
  (`project_alias_prompts.py:58-73`) with a process-level cache for
  `project_workflow_type` and `load_changespec_names`, invalidated by mtime per
  `tui_perf.md` rule 8. Project workflow type changes rarely; re-reading up to 32
  project files per keystroke is the defect.
- Confirm the cached lookup is safe for the non-TUI callers of
  `canonicalize_project_aliases_in_prompt` before landing; if any caller needs strictly
  fresh reads, give it an explicit bypass rather than weakening the cache.
- Keep `_fire_prompt_completion_timer` thin. After caching, re-measure
  `resolve_prompt_completion_base_dir` on a warm process; if the p95 for a prompt
  containing `#` is still above roughly 1 ms, move the base-dir resolution off the pump
  with `spawn_pump_free_task()` (`src/sase/ace/tui/util/pump_tasks.py`) and apply the
  result under the existing `_prompt_completion_generation` guard, cancelling at
  teardown with `cancel_pump_free_tasks()` per `tui_perf.md` rule 2.
- `resolve_prompt_completion_base_dir` must stay read-only: no CWD changes, no workspace
  claims, no project activation (its docstring at `prompt_completion_root.py:19-25`
  states this contract) and no subprocess that could prompt, per `tui_perf.md` rule 11.

### Index artifact link targets instead of scanning them per ref

Target: `src/sase/ace/tui/relations/artifact_links.py`.

- Build an index of `known_targets` keyed by the fields `_known_target_for_ref` actually
  matches on — `(pane_id, parts[-1])` for `patch`, `bead`, and `ref:<kind>`;
  `(pane_id, parts[0])` for `file`; `(pane_id, parts[0])` plus a sha-prefix structure
  for `stitch` — and reuse it across the refs resolved in one load, instead of scanning
  `known_targets` per ref (`artifact_links.py:246-280`).
- The `stitch` branch matches `target.parts[1] == sha or startswith(sha)`, so it needs a
  per-repo bucket rather than a flat dict; keep prefix semantics exactly as they are
  today.
- The `agent` branch calls `current_owner_agent_name_lookup_candidates` per target
  inside the loop; hoist that so it is computed per ref, not per target.
- Behavior must be identical, including match precedence: the exact `file` and `agent`
  fast paths at `artifact_links.py:238-245` win before the general scan. Add a test that
  a ref matching multiple targets still resolves to the same target as before.

### Regression guards for the repaired hot paths

Target: `tests/ace/tui/` and `tests/perf/`.

Add guards that would have caught each regression, following the existing bench
conventions in `tests/perf/README.md` and `docs/perf_runbook.md`:

- A prompt-panel test asserting `render()` returns the same visual instance across
  repaints within one generation, and a counter-based assertion that a layout plus paint
  pass performs exactly one anchor-collection pass per `(generation, width)` — not two.
- A countdown-tick test asserting that with the typing gate hot,
  `_update_agents_info_panel`, `_patch_agent_runtime_rows`, and
  `_poll_starting_agent_transitions` are not called, and that the following quiet tick
  calls all three.
- A config-token test asserting that repeated `current_config_token()` calls at the TUI
  tick cadence do not spawn a refresh thread per call, and that no thread is started
  while the cache lock is held.
- A completion test asserting `resolve_prompt_completion_base_dir` performs no
  project-file reads on a warm cache — assert on read counts, not wall time, so the
  guard is not flaky.
- An artifact-links test asserting resolution is not linear in target count (e.g.
  resolution work stays flat as `known_targets` grows), plus the equivalence test named
  in the `artifact_index` phase.
- Extend the existing j/k bench (`tests/ace/tui/bench_tui_jk.py`) coverage if it does
  not already report Agents-tab p95, so the 16 ms budget is measurable.

Prefer deterministic counter/instance assertions over timing assertions throughout;
`tests/selection` already carries flake baseline debt and this plan must not add more.

## Verification

- `just check` during development; `just check-full` through `/sase_monitor` before
  landing the combined tree.
- Re-run the live forensics on a real TUI after landing: with the TUI idle on the Agents
  tab, `py-spy record --pid <tui> --duration 25 --threads` must no longer show the
  compositor repaint path dominating, and process CPU must fall well below the observed
  179%.
- `~/.sase/perf/tui_jk.jsonl` with `SASE_TUI_PERF=1`: Agents-tab p95 under 16 ms.
- `~/.sase/logs/tui_stalls.jsonl`: no multi-second `tui_hitch` records during a few
  minutes of ordinary typing and navigation.

## Out of scope

- The ~2.5 GB RSS observed in the reproducing process. It is suspicious and worth its
  own investigation, but nothing gathered here ties it causally to the freezes, and this
  plan should not grow to chase it. File it as a separate task bead.
- The `sdd/_git.py` subprocess seen in the `agents_sync` worker thread and the
  `notification_store_facade` read on another worker. Both were idle in `select`/native
  code rather than burning CPU, so they are not implicated.
- Migrating any of this logic into the Rust core. These are presentation-layer Textual
  render, timer, and cache concerns, which `sase/memory/rust_core_backend_boundary`
  keeps in this repo.
