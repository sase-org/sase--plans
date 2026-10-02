---
tier: epic
title: Make the prompt `<space>` and `<ctrl+n/p>` project-cycling keys instant
goal: 'Opening the prompt bar with `<space>` and cycling the current-project stack
  with `<ctrl+n>` / `<ctrl+p>` become in-memory operations. Neither key path reads
  or writes the VCS MRU, lists project records, spawns a subprocess, or stops, starts,
  or joins a watcher on the event loop. Warm `<ctrl+n/p>` key-to-paint p95 is at most
  16 ms. `<space>` reveals a pre-built bar with key-to-paint p95 at most 60 ms. There
  are no multi-hundred-millisecond first-press or first-visit spikes. Launch, prefill,
  history, and cycling semantics stay unchanged.

  '
phases:
- id: key-perf-harness
  title: Prompt-key perf instrumentation, benchmark, and I/O probes
  depends_on: []
  size: small
  description: 'key-perf-harness: record `SASE_TUI_PERF` key-to-paint samples for
    `<space>`, `ctrl+n`, and `ctrl+p`; add a slow bench plus a non-slow smoke test;
    add a main-thread I/O probe helper that later phases use for zero-I/O tests; record
    a baseline.'
- id: mru-snapshot
  title: App-owned launchable-MRU snapshot for project cycling
  depends_on:
  - key-perf-harness
  size: medium
  description: 'mru-snapshot: add an immutable launchable-MRU snapshot owned by `AceApp`.
    A single-flight worker builds it, and peek-token ticks and launch/set-current
    triggers keep it fresh. `ctrl+n/p` read it only, pin it per prompt session, and
    show a hint instead of editing when it is cold.'
- id: space-prefill
  title: Serve `<space>` and the other MRU-head entry points from the snapshot
  depends_on:
  - mru-snapshot
  size: medium
  description: 'space-prefill: resolve the `<space>` prefill from the snapshot without
    I/O. A cold or launch-pending snapshot opens a blank bar at once and applies a
    late prefill only to an untouched session. Move `,.` and the editor entry point
    to the snapshot, and remove every MRU write from key paths.'
- id: mru-build-efficiency
  title: One project-record pass and memoized provider detection per MRU build
  depends_on:
  - mru-snapshot
  size: small
  description: 'mru-build-efficiency: make one launchable-MRU build list project records
    once and detect each project''s provider once, using per-call (never process-global)
    state, with the pruning semantics unchanged.'
- id: watcher-growth
  title: Pure catalog getters, non-blocking watcher growth, and a wakeable watcher
    stop
  depends_on:
  - key-perf-harness
  size: medium
  description: 'watcher-growth: stop catalog getters from restarting the prompt-source
    watcher. Grow watches off the pump with `ensure_watches` and reconcile once afterward.
    Add a self-pipe so `ArtifactWatcher.stop()` never waits out the 0.5 s `select`.'
- id: mount-dedup
  title: Run each prompt text-area mount, unmount, and worker hook once
  depends_on:
  - key-perf-harness
  size: medium
  description: 'mount-dedup: replace the mixins'' super-chained, Textual-dispatched
    `on_mount`, `on_unmount`, and `on_worker_state_changed` handlers with cooperative
    hooks that are dispatched once. Body order and theme layering stay the same, and
    goldens stay unchanged.'
- id: cycle-edit-coalesce
  title: One highlight build and no pump-side Jinja inspect per cycle edit
  depends_on:
  - mru-snapshot
  - mount-dedup
  size: medium
  description: 'cycle-edit-coalesce: batch highlight-map builds so a cycle edit pays
    for one. Make the queued Changed/SelectionChanged context refreshes no-ops when
    nothing changed, skip the needless arg-hint refresh, and move the Jinja diagnostics
    inspect into a pump-free task.'
- id: post-open-quiet
  title: Quiet the work that follows opening or editing the prompt
  depends_on:
  - watcher-growth
  size: small
  description: 'post-open-quiet: repaint the Agents detail only when a warmed context
    matches the selected agent, and defer that repaint while a prompt is active. Stagger
    non-essential bar warm-ups by one paint, and hoist the first-mount imports.'
- id: gc-policy
  title: Freeze startup objects and log gen-2 GC pauses
  depends_on:
  - key-perf-harness
  size: small
  description: 'gc-policy: once startup loads finish, run `gc.collect(); gc.freeze()`
    at idle. Register an allocation-light, I/O-free `gc.callbacks` hook whose gen-2
    pause records reach the stall/perf logs off the calling thread. Coordinate with
    the separate TUI-freeze investigation.'
- id: prompt-active-state
  title: Explicit prompt-active state and one prompt-bar accessor
  depends_on:
  - space-prefill
  size: medium
  description: 'prompt-active-state: track the active prompt bar explicitly on the
    app so `_prompt_input_active()` no longer queries the DOM. Route every `#prompt-input-bar`
    and `PromptInputBar` lookup through one accessor, which is the prerequisite for
    a hidden spare bar.'
- id: space-hot-spare
  title: Make `<space>` reveal a pre-built hidden prompt bar
  depends_on:
  - space-prefill
  - mru-build-efficiency
  - watcher-growth
  - mount-dedup
  - cycle-edit-coalesce
  - post-open-quiet
  - gc-policy
  - prompt-active-state
  size: large
  description: 'space-hot-spare: after re-measuring, keep one fresh, inert, hidden,
    id-less prompt bar mounted at idle. `<space>` seeds it, reveals it, and calls
    a new `activate()`. Other prompt modes keep fresh mounts, and every session still
    gets a new instance.'
- id: acceptance
  title: Final measurements, regression gates, and docs
  depends_on:
  - space-hot-spare
  size: small
  description: 'acceptance: rerun the bench against the baseline and targets, and
    consolidate the zero-I/O structural tests. Update the perf runbook, leave live-check
    instructions for the user, and record follow-ups (including the `tui_perf` memory
    rules).'
proposed_by: bbugyi200.athena.0vk
create_time: 2026-10-02 14:49:56
status: wip
bead_id: sase-1ew
---

- **PROMPT:** [prompts/202610/prompt_space_and_project_cycle_latency.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/prompt_space_and_project_cycle_latency.md)
- **BEAD:** [sase-1ew](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ew/README.md)

# Plan: Make the prompt `<space>` and `<ctrl+n/p>` project-cycling keys instant

## Context

Two prompt keys are too slow:

- `<ctrl+n>` / `<ctrl+p>` cycle the leading `+project` / `#ref` through the "current
  project" stack, which is the launchable VCS-xprompt MRU stored in
  `~/.sase/vcs_xprompt_mru.json`. They are often painfully slow.
- `<space>` (`start_agent_from_patch`) opens the prompt bar prefilled with the MRU head,
  and is sometimes slow.

A five-model research swarm measured both keys, and this plan implements its
recommendations. Phase workers should read the consolidated report before starting:

```bash
sase artifact read research:202610/prompt_space_and_project_cycle_latency/prompt_space_and_project_cycle_latency.md "<why>"
```

The per-model reports sit next to it:

- `__cld` has headless in-app timings and the prototype numbers.
- `__cdx` has controlled loader and handler measurements.

### Why the keys are slow (measured 2026-10-02 on athena)

1. **Every press re-validates the whole MRU on the UI thread.**
   - `<ctrl+n/p>` reach `_handle_vcs_mru_cycle_key`
     (`src/sase/ace/tui/widgets/_vcs_mru_cycling.py`), which calls
     `load_launchable_vcs_xprompt_mru()`.
   - `<space>` reaches `action_start_agent_from_patch`, which calls
     `_resolve_vcs_xprompt_mru_head()` in
     `src/sase/ace/tui/actions/agent_workflow/_entry_custom.py`, which calls
     `load_launchable_vcs_xprompt_mru_pairs()`.
   - Both calls default to `prune=True`, so they can **write** the MRU file. They run
     the four prune checks in `src/sase/history/vcs_xprompt_mru.py`:
     - about 9 `list_project_records` calls;
     - 6 `git config` subprocesses, from GitHub-provider workflow detection run through
       `is_launchable_project` and `_vcs_prefix_provider_mismatched`;
     - about 107 `stat`s.
   - Cost:
     - warm: 18–26 ms standalone and 30–73 ms in-app;
     - cold: 0.45–2 s;
     - one live incident: an 8.5 s `ctrl+p` freeze inside this loader.
   - With a preloaded list, the `ctrl+p` handler drops from 27.7 ms to 2.4 ms.
2. **The first `<ctrl+n/p>` onto a project restarts an inotify watcher synchronously.**
   - The cycle edit rebuilds the highlight map. That reaches a catalog getter
     (`get_warm_prompt_catalog_assist_entries_exact` and its siblings in
     `src/sase/ace/tui/actions/_startup_prompt_catalog.py`).
   - The getter calls `_ensure_prompt_catalog_project`, which calls
     `_restart_prompt_source_watcher` (`src/sase/ace/tui/actions/_startup_watchers.py`).
   - `ArtifactWatcher.stop()` (`src/sase/ace/tui/util/fs_watcher.py`) closes the fd and
     then joins a thread parked in `select()` with a 0.5 s idle timeout. On Linux,
     closing an fd does not wake that `select`.
   - Cost: 80–470 ms per first visit; one live incident took 2.8 s.
   - `ArtifactWatcher.ensure_watches()` already exists and can add watches without a
     restart.
3. **`<space>` rebuilds the whole `PromptInputBar` on every press.** The rebuild happens
   in `_show_prompt_input_bar_for_home`
   (`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mount.py`).
   - Compose plus CSS take about 45 ms.
   - An O(n²) `on_mount` storm takes about 40 ms:
     - Textual already dispatches each class's own `on_mount` once along the MRO.
     - 15 `PromptTextArea` mixins also chain `getattr(super(), "on_mount")()`.
     - The result is 136 body runs instead of 16.
   - Reflow plus paint take about 45 ms.
   - Steady p50 is about 145 ms; the first press after startup takes 0.4–0.75 s.
   - After the bar opens, the warm-ups repaint the Agents detail through
     `_refresh_visible_prompt_semantic_surfaces` →
     `_schedule_selected_agent_semantic_refresh`. That costs 58–97 ms, or 371 ms the
     first time.
4. **Per-edit amplification on `<ctrl+n/p>`.**
   - One press rebuilds the highlight map 2–3 times at 7–11 ms each:
     - once inside `TextArea.edit()`;
     - again from the glossary and repo-mention context refreshes;
     - again from the queued `Changed` / `SelectionChanged` echoes.
   - The Jinja diagnostics inspect runs synchronously on a pump timer.

### Out of scope: the cross-cutting stall budget

The live TUI spends about 10% of wall time frozen in hitches of 1.5 s or more. The
causes include the Agents-detail debouncer, clan runtime aggregation, GIL starvation,
and RSS growth. **A separate investigation already covers that.** Do not work on those
causes in this epic. This epic makes sure the two keys never _cause_ a stall and never
schedule one right after themselves.

## Target architecture

- **`LaunchableMruSnapshot`.**
  - It is immutable and owned by `AceApp`.
  - A single-flight thread worker builds it by calling the existing
    `load_launchable_vcs_xprompt_mru_pairs(prune=False)`, so the policy stays
    single-sourced.
  - Pure peek-token ticks and explicit launch/set-current triggers keep it fresh.
  - Both keys only _peek_ it.
- **Pure catalog getters.**
  - New projects grow the watch set off the pump through `ensure_watches`.
  - `ArtifactWatcher.stop()` wakes its loop at once.
- **Less work per keystroke and per mount.**
  - One highlight build per cycle edit.
  - The Jinja inspect runs off the pump.
  - Each mount body runs once.
- **No Agents-detail repaints while a prompt is active.** Non-essential bar warm-ups
  wait one paint.
- **GC.** Startup objects are frozen once startup finishes, and gen-2 pauses are logged.
- **Explicit prompt-active state, then a hot-spare hidden bar.** `<space>` becomes a
  reveal instead of a rebuild.

## Guardrails for every phase

- **Behavior parity.**
  - Cycling results stay byte-for-byte identical, including:
    - the four prune classes;
    - alias and display forms;
    - `+tag` vs `#ref` spelling;
    - Patch entries;
    - the ring's empty stop.
  - The prefill text, `display_name`, and `history_sort_key` stay identical.
  - Cancelled-history saves stay identical.
  - Cycling must never promote MRU entries.
  - Edit existing tests only where they reached into a replaced internal, and keep an
    equivalent assertion.
- **Event-loop safety.** Read the `tui.md` and `tui_perf.md` reference memories with
  `/sase_memory_read`.
  - Key handlers, render paths, and getters do no disk I/O and spawn no subprocesses
    (rules 1 and 11).
  - Off-loop work uses `spawn_pump_free_task` with `asyncio.to_thread` or
    `run_worker(thread=True)`. It keeps coalescing guards and generation checks, and it
    is cancelled at teardown (rule 2).
  - Ticks revalidate; recomputes run on a longer cadence (rule 10).
  - Non-urgent work defers while the user is navigating or typing (rule 13).
- **Authority.**
  - A stale snapshot is display data only. Launch keeps revalidating its target through
    the existing guards.
  - Nothing on these paths may call a provider's `resolve_ref`. The bare-git resolve
    _creates_ projects.
  - Patch entries keep using the offline Patch-name index.
- **Boundaries.**
  - No `sase-core` change is needed. The snapshot is TUI-side scheduling and caching
    around an existing Python policy, and provider detection is a Python plugin hook.
  - If a phase finds itself re-implementing MRU validation as a batch resolver, stop and
    record a follow-up instead.
  - No feature flag: each phase lands complete.
  - No keymap changes, so `src/sase/default_config.yml` stays unchanged.
- **Naming drift.** The in-flight rename epic `sase-1eq` (xprompts → macros) may rename
  `sase.history.vcs_xprompt_mru`, `load_launchable_vcs_xprompt_mru*`,
  `_resolve_vcs_xprompt_mru_head`, the MRU file, and nearby TUI identifiers. Use
  whatever spelling master has when you start, and never reintroduce retired names.
- **Coordination.** A separate swarm is investigating the 10% TUI-freeze budget. Before
  touching shared surfaces (GC, the stall watchdog, startup loads), check `git log` and
  active beads for overlapping work, and adapt rather than duplicate it.
- **Verification.**
  - Read the `lint_and_test.md` and `symvision.md` reference memories before finishing.
  - Run `sase tool run check`.
  - Phases that could change rendering run the relevant ACE PNG suites in check mode
    (`just test-visual -- <selectors>`), through `/sase_monitor` if the run would
    outlast the turn. No golden change is expected anywhere. A changed golden is a bug
    to explain, not to accept.
  - Do not run `just check-full` unless the bead says so.
- **Measure, don't guess.**
  - Each phase after `key-perf-harness` reruns its relevant bench cases and records
    before/after numbers in its bead notes.
  - A missed numeric target gets a recorded reason and a proposed follow-up. Never trade
    a guardrail for a number.
- **Follow-ups.** Epic phase workers record discovered work as `PROPOSED FOLLOW-UP:`
  notes on their own phase bead and do not create beads.

## Phase key-perf-harness: Prompt-key perf instrumentation, benchmark, and I/O probes

1. **Instrumentation** in `src/sase/ace/tui/util/perf.py` and its call sites.
   - Reuse `JKPerfTimer` (`app._jk_perf`, enabled by `SASE_TUI_PERF=1`) with these
     actions, each with `tab=current_tab`:
     - `prompt_space`;
     - `prompt_cycle_ctrl_p`;
     - `prompt_cycle_ctrl_n`.
   - Cycle keys: begin when the key reaches `_handle_vcs_mru_cycle_key`, not when the
     xprompt arg-name completion cycle consumes it. Mark model-updated after the edit
     applies, then `call_after_refresh(mark_painted)`.
   - `<space>`: begin at `action_start_agent_from_patch` entry. Mark model-updated once
     the bar is mounted and focused (from the bar's mount path), then mark painted after
     the next refresh.
   - Keep the disabled path a true no-op, and keep the JSONL path and env vars
     unchanged.
2. **Bench** `tests/ace/tui/bench_prompt_bar_keys.py`, marked `slow`.
   - Build on `tests/ace/tui/_bench_tui_jk_helpers.py` (`_perf_jsonl`,
     `_wait_for_startup`, `app.run_test()` + `pilot.press`).
   - Fixture: an isolated SASE home with about 30 MRU entries spanning:
     - four or more launchable projects;
     - Patch refs;
     - one stale entry for each prune class;
     - alias and display forms. When the GitHub workspace plugin is installed, include a
       project it detects, so provider detection runs as in production.
   - Cases. Report p50/p95/max `paint_ms` and the handler time for each:
     - first `<space>` after startup;
     - steady `<space>` (open, `escape`, repeated N times);
     - a single `ctrl+p` in an open bar;
     - a six-key `ctrl+p` burst at a 50 ms cadence, as per-key lag;
     - a first-visit `ctrl+p` onto a project whose prompt catalog this session has not
       requested yet.
   - Also count the stall-watchdog rows written during the run.
   - Print one table. Do not assert budgets: shared-host timing is noisy.
3. **Smoke test.** A non-slow test runs the smallest case end to end (one `<space>`, one
   `ctrl+p`) and asserts that samples were recorded, so the harness cannot rot.
4. **I/O probe helper**, for example `tests/ace/tui/_prompt_key_io_probes.py`. It is a
   fixture or context manager that counts **main-thread-only** calls during a block.
   Off-thread warm-up workers may legitimately do I/O. It counts:
   - MRU file reads and writes;
   - `list_project_records` calls, patched at the facade _and_ at the modules that
     import it by name;
   - `subprocess.Popen` constructions;
   - `ArtifactWatcher.start` / `ArtifactWatcher.stop`;
   - `threading.Thread.join`.

   Later phases assert zeros through it.

5. **Docs.** Add a "Prompt keys" recipe to `docs/perf_runbook.md` and a pointer in
   `tests/perf/README.md`.

**Acceptance:**

- The bench runs end to end on master.
- The smoke test passes under `sase tool run check`.
- The baseline table is recorded in the bead notes.

## Phase mru-snapshot: App-owned launchable-MRU snapshot for project cycling

**Pure model.** Create a Textual-free module, for example
`src/sase/ace/tui/launchable_mru.py`.

- `LaunchableMruSnapshot` is a frozen dataclass with:
  - `state`: `"cold" | "ready" | "error"`;
  - `pairs: tuple[tuple[str, str], ...]` as `(canonical, display)`, in MRU order;
  - `generation: int`;
  - `token`: the peek token observed when the build started;
  - `inputs_signature`;
  - a `refresh_pending` flag (see Triggers).
- Keep "MRU is empty" (`ready` with no pairs) distinct from "cold".
- A build function calls `load_launchable_vcs_xprompt_mru_pairs(prune=False)` and never
  writes. The VCS `+` completion catalog already calls this loader from a thread worker,
  which is the thread-safety precedent.

**App mixin.** For example `src/sase/ace/tui/actions/_launchable_mru.py`; follow the
conventions of `_startup_prompt_catalog.py` and `LaunchContextSource`.

- Register it on `AceApp` and initialize its state alongside the other `_state_init_*`
  mixins.
- `peek_launchable_mru_snapshot()` reads memory only.
- `request_launchable_mru_refresh(*, reason)` is single-flight:
  - one build in flight, one pending, last request wins;
  - the build runs in a pump-free thread task;
  - the result publishes on the main thread only if its generation is current;
  - tasks are cancelled at teardown.
- An optional fast path: the worker first computes a cheap inputs signature:
  - MRU `(mtime_ns, size)`;
  - `current_config_token()`;
  - `vcs_project_catalog_signature(sase_projects_dir())`;
  - the Patch cache signature, if one is exposed.

  If it matches the last published signature, the worker skips validation and publishes
  nothing.

**Triggers:**

- **Startup.** Warm right after first paint, next to the tag-catalog warm in
  `_start_immediate_startup_loads` (`src/sase/ace/tui/actions/_startup_loads_core.py`).
- **Token tick.** An app interval of about 2 s calls
  `peek_current_project_change_token()` (`src/sase/current_project.py`), a time-gated
  MRU stat plus config token that parses nothing. On drift it requests a build. This
  catches writes from other processes, such as CLI `sase run` and
  `sase project set-current`.
- **Recompute.** About every 60 s, request a build, but only when no prompt is active
  and `_nav_gate` is idle. This covers project-spec and Patch drift that the MRU token
  cannot see.
- **Launch.** Synchronously at submit, set `refresh_pending` on the snapshot; this is
  pure memory. After `schedule_submit_time_vcs_replay`
  (`src/sase/ace/tui/actions/agent_workflow/_launch_submit_helpers.py`) finishes its
  off-thread `record_vcs_xprompt_usage`, request a build. Publishing that build clears
  the flag.
- **Set-current from the TUI.** `project_management_actions.py`
  (`_invalidate_current_project_indicator` and its siblings) sets `refresh_pending` and
  requests a build. Do the same for TUI project enable, disable, rename, and alias
  actions.
- Drift detected while a prompt is active builds off-thread and publishes silently. The
  pinned ring (below) protects any in-flight cycle.

**Cycle keys.** In `_handle_vcs_mru_cycle_key`:

- On the first cycle press of a prompt session, peek the snapshot.
  - If it is `ready`, pin its display ring on the text area as
    `self._vcs_mru_ring: tuple[str, ...]`.
  - A burst never touches app state again. A snapshot published mid-burst cannot reorder
    the ring or change `_vcs_mru_index`.
  - Reset the pinned ring wherever `_vcs_mru_index` resets
    (`_prompt_text_area_actions.py`).
- Cold or error snapshot:
  - Leave the text untouched and request a build.
  - Show a brief "loading recent projects" hint, using the lightest existing transient
    prompt-bar hint surface, or a one-second toast as a fallback.
  - Never queue the edit for replay.
- Keep calling `peek_project_tag_catalog()` (memory-only) for display spellings, exactly
  as today.
- **Host fallback.** If the host app has no `peek_launchable_mru_snapshot`, keep the
  synchronous `load_launchable_vcs_xprompt_mru(prune=False)` call so bare test `App`
  hosts keep working. Today every production host is `AceApp`. Existing tests that patch
  the loader keep passing through this fallback.

**Tests:**

- **Parity.**
  - For fixtures covering the default prefix, a stale known project, a gone ref, a
    provider mismatch, alias display forms, and Patch entries, the snapshot ring equals
    the loader's displays.
  - In a real `AceApp` with a seeded snapshot, cycling reproduces the expected text and
    cursor of the existing `tests/ace/tui/widgets/test_prompt_vcs_mru_cycling.py` cases,
    including the empty stop and `ctrl+n` from an empty prompt.
- **Single-flight.**
  - N requests during a build produce exactly one follow-up build.
  - A stale generation is never published.
- **Freshness.** Each of these produces exactly one rebuild:
  - token drift seen by the tick;
  - a launch, which also sets `refresh_pending` immediately;
  - a TUI set-current.
- **Pinning.** A publish mid-burst leaves the ring and index unchanged, and the next
  session sees the new generation.
- **Cold.** `ctrl+p` leaves the text unchanged, shows the hint, and schedules one build.
  The next press after publish cycles normally.
- **Structural.** With a warm snapshot, `ctrl+p` and `ctrl+n` perform zero main-thread
  MRU reads or writes, `list_project_records` calls, and `Popen`s (use the harness
  probe).
- **Bench.** Single `ctrl+p` and burst, before and after.

## Phase space-prefill: Serve `<space>` and the other MRU-head entry points from the snapshot

1. Refactor `_resolve_vcs_xprompt_mru_head` to take the `(canonical, display)` pairs
   instead of loading them. The tag spelling still comes from
   `peek_project_tag_catalog()`.
2. **`<space>`** (`action_start_agent_from_patch`). Keep its legacy override hooks.
   - **Snapshot `ready` and not `refresh_pending`:** compute the head prefill from the
     snapshot with no I/O. Call `_show_prompt_input_bar_for_home(...)` exactly as today;
     an empty MRU opens a blank bar.
   - **Snapshot cold, error, or `refresh_pending`:** open the blank home bar immediately
     and record a pending prefill keyed to the new prompt session id.
     - When the next snapshot publishes, apply the prefill only if all of these hold:
       - `prompt_session_is_live` holds for that id;
       - the bar is still mounted in single-pane prompt mode;
       - the active text area is untouched (empty, no edits since mount, cursor and pane
         unchanged).
     - Applying it means:
       - set the live `PromptContext.display_name` and `history_sort_key`;
       - load the text;
       - put the cursor at the end;
       - refresh the title, subtitle, and dispatch line.
     - Otherwise drop the pending prefill. Dismissal, session invalidation, and a new
       `<space>` also drop it.
3. **`,.`** (`_start_prompt_history_from_last_selection` in `_entry_prompt_history.py`)
   and **`start_last_vcs_xprompt_in_editor`** use the snapshot when it is ready. They
   are not the hot keys and they open a modal or editor anyway, so when the snapshot is
   cold they fall back to the synchronous loader with `prune=False`.
4. **No MRU writes from key paths.**
   - After this phase, nothing in the TUI calls the loader with `prune=True`.
   - Do not add a replacement persisted prune. Read-time filtering already hides stale
     entries, `record_vcs_xprompt_usage` drops a stale recorded prefix, and the file is
     capped at 100 entries.
   - If a maintenance prune still looks warranted, record it as a follow-up. It must be
     concurrency-safe against `record_vcs_xprompt_usage`.

**Tests:**

- **Warm prefill parity.** Initial text, display name, and history key match today for:
  - a project with an alias;
  - a Patch;
  - a `+tag` vs `#ref` spelling;
  - an empty MRU, which opens a blank bar.
- **Structural.** A warm `<space>` performs zero main-thread MRU reads or writes,
  `list_project_records` calls, and `Popen`s before the bar is painted.
- **Cold or pending.**
  - The bar opens without awaiting a build.
  - The prefill lands when the session is untouched.
  - Any of these drops the prefill: typing one character, moving the cursor, switching
    panes, or dismissing the bar.
  - A reopened bar never receives a previous session's prefill.
- **Launch-then-space.** Submit a launch, then press `<space>` before the rebuild lands.
  The bar opens blank and then shows the just-launched project.
- **`,.` and editor parity.** Results match today, warm and cold.
- **Bench.** First and steady `<space>`, before and after.

## Phase mru-build-efficiency: One project-record pass and memoized provider detection per MRU build

`load_launchable_vcs_xprompt_mru_pairs` now runs only in workers and in non-key paths,
but it still does redundant work:

- `is_launchable_project` (`src/sase/ace/tui/modals/project_discovery.py`) lists every
  project record on each call.
- `resolve_project_alias_ref` may list them again.
- `_vcs_prefix_provider_mismatched` calls `detect_workflow_type` a second time per
  entry.
- `_dedupe_mru_pairs` → `humanize_vcs_refs_in_text` may read display names per entry.

The fix:

- Introduce per-call state; never add a process-global cache.
  - It lists the needed project records once.
  - It resolves aliases from that list.
  - It memoizes `detect_workflow_type` by project-file path for the duration of one
    call.
- Keep `is_launchable_project`'s public signature. Add an internal variant that accepts
  pre-listed records.
- `record_vcs_xprompt_usage` and `current_project` may share the helper, but their
  behavior must not change.

**Tests:**

- The existing `tests/test_vcs_xprompt_mru_*.py` suites pass unchanged.
- New counters assert:
  - at most one `list_project_records` call per build, or a small constant independent
    of entry count;
  - at most one `detect_workflow_type` call per distinct project per build.
- Record the worker build time before and after.

## Phase watcher-growth: Pure catalog getters, non-blocking watcher growth, and a wakeable watcher stop

1. **Separate the request from the read.**
   - These getters keep adding the project to `_prompt_catalog_projects` (the desired
     set) and keep scheduling catalog work as today:
     - `get_prompt_catalog_assist_entries`;
     - `get_warm_prompt_catalog_assist_entries_exact`;
     - `warm_prompt_catalog_project`.
   - `_ensure_prompt_catalog_project` must never stop, start, or join anything. It only
     schedules a coalesced watch-growth task.
2. **Watch growth.**
   - A single-flight pump-free task computes `prompt_source_watch_paths(new_projects)`
     (`src/sase/ace/tui/prompt_catalog.py`) off-thread. It then calls
     `ArtifactWatcher.ensure_watches(paths)` there; that call is lock-guarded and
     installs only new watches.
   - Generation-own the desired set: if the watcher was replaced or stopped meanwhile,
     discard the result.
   - On the main thread, update `_prompt_source_watched_projects` and schedule one
     reconciling rebuild (`_schedule_prompt_catalog_rebuild(reason="watch_growth")`).
     That catches edits made before the watch existed.
   - With no running watcher, keep today's polling fallback.
   - Never let the UI thread wait on a join while the watcher thread waits on
     `call_from_thread`.
3. **Wakeable stop** in `src/sase/ace/tui/util/fs_watcher.py`.
   - Create a non-blocking self-pipe (or an `eventfd`) in `start()`, and add its read
     end to the `select` set in `_loop`.
   - `stop()` runs in this order:
     1. set the stop event;
     2. write the wake byte;
     3. join with a short timeout;
     4. _then_ close the inotify fd and the pipe fds, so the worker never selects on a
        closed or reused fd number.
   - Guard `ensure_watches` and `prune_agent_dir_watches` against a concurrent `stop()`.
   - The artifact watcher's quit path benefits as well.

**Tests.** Extend `tests/ace/tui/test_fs_watcher.py`, `test_startup_watchers.py`, and
`test_prompt_catalog.py`:

- With the loop idle, `stop()` returns well under 0.5 s; assert a generous bound, for
  example under 100 ms.
- Requesting a new project's catalog never calls `ArtifactWatcher.stop` or `start`, and
  never joins a thread on the main thread.
- The new project's xprompt and memory directories gain watches. A later edit there
  triggers a rebuild.
- An edit made between the request and the watch install is caught by the reconcile.
- **Structural.** A first-visit `ctrl+p` performs zero main-thread `stop`, `start`, or
  `join` calls.
- **Bench.** The first-visit case, before and after.

## Phase mount-dedup: Run each prompt text-area mount, unmount, and worker hook once

**How dispatch works.** Textual's `MessagePump._get_dispatch_methods` invokes each
class's own `on_mount` / `_on_mount` once along the MRO, most-derived class first. The
`PromptTextArea` mixins also super-chain, so the j-th definer runs j times. Verify this
inventory against the code:

- **`on_mount` definers:**
  - the theme mixins: Yank, Search, Todo, AltSyntax, ArtifactRef, XPromptSyntax, Bullet,
    CodeBlock, Placeholder;
  - `PromptGlossary` and `PromptRepoMention`, which also schedule context warms;
  - `MisspellingHighlight`: repeated calls set a rebuild-pending flag, causing an extra
    rebuild;
  - `JinjaHighlight`: re-registers its theme and sets `self.theme`;
  - `FileCompletionDirectiveInventoryWorker`;
  - `LineRendering` (`_line_rendering.py`), whose chain reaches Textual's
    `ScrollView.on_mount`.
- **`on_unmount`:** `ArtifactRefSync` and `PromptSoftCompletion`. The second runs twice.
- **`on_worker_state_changed`:** `ArtifactRefHighlight` chains into
  `FileCompletionWorker`. The latter runs twice per worker event, so inventory results
  are applied twice.
- `AgentPromptPanel` has the same doubling: `AgentDisplayWorkerMixin` →
  `WorkflowDisplayMixin`. Include it; the fix is the same.

**Order hazard.** Do not simply delete the `super()` calls. Today each dispatch runs its
chain's bodies _base-first_. For example, `YankHighlight.on_mount` documents that it
registers "after the other overlay themes exist". Plain MRO dispatch would run Yank
first.

**Recommended shape:**

- Rename each mixin's dispatched handler to a cooperative, non-dispatched hook, for
  example `_prompt_mount_hook()`.
- Keep the `super()` chaining inside these hooks, so body order stays base-first.
- Dispatch the hook exactly once, from a single handler on the shared base. For
  `on_mount` that is `LineRendering`/`VimTextArea`, because other `VimTextArea`
  subclasses also chain through it.
- Stop chaining into Textual's `ScrollView.on_mount`; Textual dispatches it itself.
- Apply the same shape to `on_unmount` and `on_worker_state_changed`.

**Out of scope.** Record follow-ups for:

- `command_line/input.py`;
- the doubled `_on_mount` in `filter_bar.py`;
- `modals/axe_entry_editor_rendering.py`.

**Tests:**

- **Once, in order.** Each defining class's mount body runs exactly once per mount, in
  the same relative order as today's first chain. Do the same for unmount bodies.
  Worker-state bodies run once per event, and inventory apply functions run once per
  worker success.
- **Theme parity.** After mount and after an app theme switch, the applied text-area
  theme name and syntax styles equal the current behavior. Capture the expectation from
  a pre-change run.
- **Visual.** Run the prompt-bar ACE PNG snapshots in check mode with zero golden
  changes. A diff means the effective theme layering changed.
- **Bench.** `<space>` before and after; the research expects −25 to −35 ms.

## Phase cycle-edit-coalesce: One highlight build and no pump-side Jinja inspect per cycle edit

1. **Highlight batching.**
   - Add a `_highlight_batch()` context manager on the text area. Inside it, any
     `_build_highlight_map` request becomes a single build at exit.
   - Intercept at the most-derived override, so the call inside `TextArea.edit()` is
     deferred too. Verify that Textual routes it through the override; adapt if it does
     not.
   - Wrap the cycle handler's edit, cursor move, hint refresh, and context refresh in
     the batch.
2. **Idempotent context refresh.**
   - Skip the expensive highlight/context parts of
     `_on_prompt_completion_context_changed` when their inputs have not changed since
     the last run. Those parts are the glossary, repo-mention, and Jinja-delimiter
     refreshers. The inputs are the document text or version, the cursor offset, and the
     catalog generations.
   - The queued `Changed` / `SelectionChanged` echoes after a cycle then do no highlight
     work.
   - `PromptInputBar.on_text_area_changed` keeps its cheap bookkeeping: sync-state, todo
     counts, title, dispatch line, height, and readouts.
3. **Arg hint.** A leading-tag swap does not need an immediate
   `_refresh_xprompt_arg_hint_from_cursor`. Skip it, using a cheap check, when the
   cursor is not inside an xprompt argument list. Hints still appear on the next
   relevant move or keystroke.
4. **Jinja diagnostics off the pump.**
   - Run `_fire_jinja_diagnostics_timer`'s `_inspect_with_engine_scope` through
     `spawn_pump_free_task` + `asyncio.to_thread`, on an immutable text copy and catalog
     snapshot.
   - Drop the result if the text changed meanwhile. Apply the overlay and title on the
     main thread.
   - Cancel the task at unmount.
5. **Arg-assist wire conversion.** Confirm that the `xprompt_arg_assist_entries_to_wire`
   memo in `_xprompt_syntax_highlight.py` covers the cycle path. Fix the unmemoized call
   in `src/sase/xprompt/highlight.py` if the cycle path reaches it.

**Tests:**

- **Build count.** At most one full highlight build happens synchronously per `ctrl+p`,
  and none comes from the queued echoes when nothing else changed. A later catalog warm
  may add one.
- **Span parity.** The highlight spans after a cycle equal a fresh unbatched rebuild.
- **Jinja.** Diagnostics still appear after typing `{{ bad`; await the pump-free task in
  the test. Stale inspect results are dropped.
- **Bench.** Single `ctrl+p` and burst, before and after; the research expects −10 to
  −20 ms.

## Phase post-open-quiet: Quiet the work that follows opening or editing the prompt

1. **Scope the Agents-detail repaint.**
   - Today `_refresh_visible_prompt_semantic_surfaces` and
     `_refresh_visible_prompt_catalog_surfaces` (`_startup_prompt_catalog.py`) always
     end in `_schedule_selected_agent_semantic_refresh`.
   - Pass the warmed or invalidated context through. Repaint only in two cases:
     - the context matches the selected agent's highlight context (project and
       workspace, as `agent_prompt_highlight_context` in
       `src/sase/ace/tui/widgets/prompt_panel/_agent_xprompt_highlighting.py` derives
       it);
     - the change is global, such as an invalidation or a config change.
   - While `_prompt_input_active()` holds, set a pending flag instead, and repaint once
     on prompt dismissal through the bar detach path.
   - Skip hidden or not-displayed prompt text areas in the `query(PromptTextArea)`
     loops. This prepares for the hot spare.
2. **Stagger warm-ups** in `PromptInputBarLifecycleMixin.on_mount`
   (`src/sase/ace/tui/widgets/_prompt_input_bar_lifecycle.py`).
   - Keep these synchronous: focus, cursor, title, subtitle, classes, height, and
     whatever the first keystroke needs.
   - Move the non-essential warm-ups behind one `call_after_refresh`, so the bar paints
     first:
     - artifact-ref catalog;
     - model catalog;
     - path inventory;
     - history-word cache;
     - prediction cache;
     - placeholder cache;
     - dispatch-target catalog.
   - Measure the xprompt assist entries and the VCS project completion catalog. Keep
     them synchronous if moving them would delay the first keystroke.
3. **Hoist first-mount imports** in `_refresh_dispatch_context_line` /
   `_dispatch_context_text` (`_prompt_input_bar_dispatch.py`), plus other first-open
   lazy imports that the bench or `python -X importtime` exposes (about 27 ms measured).
   Hoist them to module level, or warm them during deferred startup maintenance. Choose
   whichever leaves TUI cold start no slower.

**Tests:**

- With a prompt active, a warm that lands for a different context schedules no detail
  repaint, and dismissal repaints exactly once.
- With no prompt active, a warm for the selected agent's context repaints.
- First-keystroke completion and arg hints behave the same after the stagger.
- **Bench.** `<space>`, before and after.

## Phase gc-policy: Freeze startup objects and log gen-2 GC pauses

The TUI's interpreter, CPython 3.14.7, uses the classic three-generation collector
(`gc.get_threshold()` returns `(2000, 10, 10)`). In-app full collections measured
445–600 ms, versus 56 ms after `gc.freeze()`. Nothing in `src/sase` tunes or instruments
the collector today.

Before starting, check master and active beads for GC work from the separate TUI-freeze
investigation. If one already landed, keep what exists and fill only the gaps.

1. **Freeze.**
   - Run `gc.collect(); gc.freeze()` once, after `_mount_state_loads_done` and the
     deferred startup release in `_startup_loads_core.py`.
   - Run it at the first idle moment: no prompt active and `_nav_gate` idle. Use a short
     timer that re-arms until idle.
   - Record how long the collect took.
   - Leave idle re-freezing as a follow-up.
2. **Pause logging.**
   - Register a `gc.callbacks` hook that times each collection (`start`/`stop`,
     `info["generation"]`).
   - Append gen-2 pauses at or above a small threshold (for example 20 ms) to a bounded
     in-memory deque.
   - The callback must allocate almost nothing and do no I/O, because it runs on
     whichever thread triggered the collection.
   - Get the records out off the event loop: enrich stall-watchdog rows with the pauses
     that overlap each stall window, and/or flush to a sibling JSONL under
     `~/.sase/logs/` from an existing background thread. Follow the `tui_stalls.jsonl`
     rotation conventions.
3. Unregister the callback at app shutdown.

**Tests:**

- The freeze runs once, only after startup loads finish, and never while a prompt is
  active.
- Calling the callback directly with synthetic long collections records pauses, and the
  calling thread never writes.
- Teardown unregisters the callback.

**Docs.** Explain how to read the GC rows in `docs/perf_runbook.md`.

## Phase prompt-active-state: Explicit prompt-active state and one prompt-bar accessor

1. **Active-bar state.**
   - Add `app._active_prompt_bar`. Every `PromptInputBar` mount site sets it:
     - `_show_prompt_input_bar_for_home`;
     - `_entry_relaunch.py`;
     - `agents/_notification_modals.py`;
     - any others that exist.
   - `_detach_prompt_bar` and the unmount paths clear it.
   - `_prompt_input_active()` (`src/sase/ace/tui/actions/_event_base.py`) returns
     `_prompt_editor_suspended`, or whether that bar is set and still attached. It no
     longer queries the DOM.
   - Preserve today's screen semantics for bars mounted on modal screens.
2. **One accessor.**
   - Generalize the existing `_mounted_prompt_bar` helper in
     `_prompt_bar_stash_store.py`.
   - Route every `query_one("#prompt-input-bar", PromptInputBar)` and
     `query(PromptInputBar)` lookup in `src/` through it. Today those lookups live in:
     - `_prompt_bar_mount.py`;
     - `_prompt_bar_requests.py`;
     - `_prompt_bar_stash_store.py`;
     - `_prompt_bar_memory_panel.py`;
     - `_prompt_bar_snippets_panel.py`;
     - `_launch_prompt_inputs.py`;
     - `src/sase/ace/testing/ace_page_group.py`.
3. **Callers stay as they are.** These keep calling `_prompt_input_active()`, which is
   now cheaper:
   - the countdown tick;
   - auto-refresh;
   - the watcher defer;
   - the detail guards;
   - `check_app_action` (`_app_action_availability.py`);
   - the deck live timers;
   - metadata search;
   - the focus guard.

**Tests:**

- **State transitions.** Cover each of these, and check that a failed mount leaves the
  state consistent:
  - mount;
  - submit;
  - cancel;
  - editor suspend and resume;
  - modal prompt bars.
- **Parity.** Over a scripted session, `_prompt_input_active()` equals the old DOM query
  at every step.
- **No DOM walk.** `check_action` for prompt-gated actions no longer queries the DOM.

## Phase space-hot-spare: Make `<space>` reveal a pre-built hidden prompt bar

**Gate.** First rerun the bench with every Phase 1 change landed. If `<space>` p95 is
already at most 60 ms, record the numbers, skip the hot spare, and close the phase with
a note. The research expects it to be needed, because compose, CSS, and reflow alone
cost about 90 ms. Otherwise:

1. **Spare lifecycle.**
   - After startup loads complete, and after each prompt dismissal, mount one fresh
     `PromptInputBar` with `display: none` once the app is idle (not navigating, no
     prompt active). Mount it in the same container and position as today's bar.
   - Keep at most one spare.
   - Never reuse a used bar. A new instance per session keeps today's undo, completion,
     vim, search, and frontmatter state guarantees.
   - The spare is id-less, because Textual does not allow setting an id after mount;
     verify this. So:
     - convert the bar's TCSS `#prompt-input-bar` selectors to a class that both spare
       and fresh bars carry;
     - rely on the `prompt-active-state` accessor for every lookup.
2. **Inert while hidden.** The spare:
   - cannot take focus;
   - is not counted by `_prompt_input_active()`;
   - is skipped by refresh loops;
   - runs no warm-ups, timers, or workers.

   Move today's `on_mount` focus, cursor, title, subtitle, class, and warm-up logic into
   a new `activate()` method. Fresh, non-spare bars call `activate()` from `on_mount`,
   so other prompt modes are unchanged.

3. **`<space>` with a ready spare** (plain home prompt):
   1. Begin the prompt session (`_setup_home_prompt_context`), which mints a new
      `session_id`.
   2. Seed the spare's stack and text with the existing loaders, using the same prefill
      rules as `space-prefill`, including the late prefill.
   3. Reveal it.
   4. Set the active bar.
   5. Call `activate()`.

   With no spare ready, fall back to today's mount path.

4. **Other entry points keep their explicit remounts.** These are feedback, approve,
   markdown editor, relaunch, `as_xprompt_markdown`, frontmatter inputs, and read-only
   targets.
5. **Dismissal** keeps today's cancelled-history save for the outgoing session and the
   synchronous detach. The used bar is removed and never recycled.

**Tests:**

- The spare mounts once and stays hidden and inert. `_prompt_input_active()` is False;
  countdown, auto-refresh, and watcher callbacks are unaffected;
  `start_agent_from_patch` stays enabled.
- A revealed spare matches the fresh-mount path for text, context, title, subtitle,
  cursor, and classes. Parametrize over a warm prefill, an empty MRU, and a cold
  snapshot with late prefill.
- The session id is new on each activation.
- Cancel saves history once, for the right session. Submit works.
- Rapid `<space>` / `escape` sequences work.
- The spare is replaced after dismissal.
- Quitting with a spare mounted leaves no unfinished workers. See the structural-quiet
  tests such as `test_ace_page_fast_startup_is_structurally_quiet`.
- **Visual.** The prompt-bar and Agents-layout ACE PNG suites pass in check mode with no
  golden changes.
- **Bench.** First and steady `<space>`; expect about 50–60 ms. If it still misses the
  target, the remaining lever is docking the bar as an overlay. Record that as a
  follow-up; it changes the UX.

## Phase acceptance: Final measurements, regression gates, and docs

1. **Final bench.** Rerun the full bench and record a before/after table against the
   `key-perf-harness` baseline, including maxima and stall-row counts. The targets:
   - warm `ctrl+n/p` key→paint p95 at most 16 ms, with the synchronous handler at most 5
     ms;
   - `<space>` p95 at most 60 ms, or at most 110 ms if the hot spare was skipped;
   - the first `<space>` after startup is never blocked by a cold snapshot;
   - no first-visit or burst spikes.
2. **Consolidated structural tests**, cheap enough for `sase tool run check`. Make sure
   all of these exist exactly once, de-duplicating what earlier phases added:
   - With a warm snapshot in a real `AceApp`, `ctrl+n`, `ctrl+p` (including a first
     visit to an unseen project), and `<space>` perform zero main-thread:
     - MRU reads or writes;
     - `list_project_records` calls;
     - `Popen`s;
     - watcher `start`/`stop` calls;
     - thread joins.
   - The `on_mount` body count equals the number of defining classes.
   - A delayed or older snapshot generation cannot overwrite text or reorder an
     in-flight cycle.
   - A cold `<space>` followed by typing is never clobbered by a late prefill.
3. **Optional bench budgets.** Generous budget assertions may go into the slow bench
   only, never into `just check`.
4. **Docs.**
   - `docs/perf_runbook.md`: the prompt-keys section.
   - `tests/perf/README.md`: final pointers.
   - Any docs that describe `<space>` pruning the MRU.
5. **Live-check instructions** go in the bead notes; do not restart the user's TUI
   yourself. The user should:
   1. restart the running TUI, because editable installs keep running already-imported
      code;
   2. run with `SASE_TUI_PERF=1`;
   3. check `~/.sase/perf/tui_jk.jsonl` for `prompt_*` samples;
   4. confirm that no `~/.sase/logs/tui_stalls.jsonl` rows have these handlers on the
      stack.
6. **`PROPOSED FOLLOW-UP:` notes:**
   - A `memory` task to add `tui_perf.md` rules:
     - keymaps read app-owned snapshots and never validate on keystroke;
     - getters never stop, start, or join watchers;
     - mixins must not super-chain Textual-dispatched handlers.
   - The overlay-docked prompt bar experiment.
   - The remaining `on_mount` doubling outside the prompt.
   - A maintenance prune of stale MRU entries, if still wanted.
   - Anything measured but out of scope.

## Non-goals

- The stall-budget track: the Agents-detail debouncer, clan runtime aggregation, GIL
  starvation by workers, and RSS growth. A separate investigation owns these.
- Changing the MRU validation policy, porting it to Rust, or adding a persistent index
  daemon.
- Dropping markdown highlighting, rebinding `<space>`, or retuning debounces.
- Overlay-docking the prompt bar.
- Making cycling promote MRU entries.
