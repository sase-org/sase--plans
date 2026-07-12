---
create_time: 2026-07-12 09:54:25
status: done
prompt: 202607/prompts/tui_pump_starvation_freeze.md
tier: tale
---
# Fix multi-minute ace TUI freeze: message-pump starvation invisible to the stall watchdog

## Problem

On 2026-07-12 the ace TUI froze (no response to any input) for ~5.5 minutes (~09:24:48 → 09:30:19 EDT), and the
always-on stall watchdog recorded nothing in `~/.sase/logs/tui_stalls.jsonl`.

## Diagnosis (from logs; reconstruct before trusting)

Timeline evidence (`~/.sase/logs/`):

- 09:24:44 — user submits an agent launch (`tui_launch_timing.jsonl`: `tui_agent_launch` total 3.48s;
  `agent_launch_spawn` ok at 09:24:48.04, 294ms). The launch worker thread returned ~09:24:48.35.
- The launch completion effects (`Worker.StateChanged` → `_on_task_worker_completed` → `_on_launch_task_complete` →
  "Agent started for sase" toast) only ran at **09:30:19** — 5.5 minutes later (`tui_toasts.jsonl`).
- In between, notification-poll toasts still appeared (09:27:07 "Question from @…", 09:27:27 "Axe: … failed"). Those are
  emitted from a Textual **timer callback**, which runs in the timer's own asyncio task
  (`textual/timer.py: _tick → await invoke(callback)`), so the asyncio **event loop was healthy** the whole time.
- `tui_stalls.jsonl` recorded nothing; the watchdog thread was confirmed alive in the running process; `tui.log` has no
  stall/recovery warnings. No kernel OOM or system-level stall in the journal.
- The user was locked out for the whole window: they launched a diagnostic agent through the Telegram bridge at 09:27:29
  (axe spawn with no `tui_agent_launch` wrapper) and opened a second ace instance at 09:28:06 because the primary TUI
  would not respond.

Interpretation: the **App-level message pump** was stuck awaiting a single message handler from ≤09:24:48 until
09:30:19. Textual's pump processes messages serially and awaits coroutine handlers to completion
(`textual/message_pump.py: _process_messages_loop → await invoke(...)`). Key events, worker state-changes, and
`call_later` callbacks all queue behind a stuck handler — total input freeze — while timer tasks and the watchdog's
`call_soon_threadsafe` beacon keep running. The current watchdog measures only raw loop latency, so pump starvation is
structurally invisible to it.

Pump-starvation vectors identified in the current code (all violations of tui_perf rule 1 at the pump level; any of them
can reproduce this freeze):

1. **Agents refresh runs ON the pump** — `src/sase/ace/tui/actions/agents/_loading_refresh.py`:
   `_schedule_agents_async_refresh` schedules `_run_agents_async_refresh` via `self.call_later(...)`. `call_later` posts
   an `events.Callback` that the pump awaits, and the coroutine awaits the entire agents load
   (`await load_agents_async(...)`). Any hung/slow read inside the load (agent-name registry JSON, artifact-index
   sqlite, workflow/chat scans — all contended by concurrently running agent runners; three runners plus axe hook runs
   like `just test` were active during the freeze window) starves all input for the full duration. A launch triggers
   exactly this refresh, and the freeze began seconds after a launch.
2. **Prompt-stash pump handlers await IO in-handler** — `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_stash.py`:
   - `async def on_app_focus` awaits `asyncio.to_thread(read stash counts)`; fires every time the terminal/tmux pane
     regains focus (the user hops panes constantly).
   - `on_prompt_input_bar_restore_requested`, `action_restore_prompt_stash`, and
     `on_prompt_input_bar_update_pinned_requested` await stash reads in-handler.
   - `on_prompt_input_bar_stashed` (sync) calls `append_prompt_stash` directly on the loop. The prompt-stash store
     (`sase-core` `crates/sase_core/src/prompt_stash/store.rs`) takes **blocking flocks with no timeout**
     (`FileExt::lock_shared` / `lock_exclusive`) on the per-user pile `~/.sase/prompt_stash.jsonl.lock`, which is shared
     across concurrent ace instances and CLI processes. One wedged exclusive holder blocks every reader indefinitely.

Which vector fired on 2026-07-12 cannot be proven post-hoc — precisely because the watchdog cannot see pump starvation
and records no recovery rows. So the fix has two halves: make the pump un-starvable on the identified paths, and make
any recurrence self-diagnosing.

## Phase 1 — Pump-starvation observability (stall watchdog extension)

- Extend `src/sase/ace/tui/util/stall_watchdog.py` with a second beacon that measures **message-pump latency**: from the
  watchdog thread, periodically deliver a pump-routed mark (e.g. via `app.call_later`) and measure delivery lag. Re-arm
  only after the previous mark is delivered so a stuck pump cannot accumulate a callback flood.
- On pump lag ≥ threshold (reuse the 5s default; add `SASE_TUI_PUMP_STALL_*` env knobs mirroring the loop ones), append
  a durable `tui_pump_stall` record to `tui_stalls.jsonl` including **asyncio task stacks** (`asyncio.all_tasks()` +
  `Task.get_stack()` captured via a loop-side ping or `sys._current_frames` fallback) so the record names the exact
  stuck handler/await — the datum that was missing today. Include `pause_depth` and the existing trace context fields.
- Write durable **recovery** rows (`tui_stall_recovered` / `tui_pump_stall_recovered` with duration) for both beacon
  types instead of only `log.warning` — today the freeze window had to be reconstructed from toast timestamps.
- Respect the existing suspend pause/resume wiring for both beacons.
- Tests: a fake app/pump fixture with a deliberately stuck async handler → pump-stall + recovery records written with a
  captured stack; loop-beacon behavior unchanged; suspend pause still quiets both beacons.

## Phase 2 — Un-starve the pump in the identified handlers

- `_prompt_bar_stash.py`:
  - `on_app_focus`: replace the in-handler await with the existing `_spawn_prompt_stash_task(...)` fire-and-forget
    pattern (already used for pin persistence and badge refresh in the same file).
  - `on_prompt_input_bar_restore_requested`, `action_restore_prompt_stash`,
    `on_prompt_input_bar_update_pinned_requested`: return immediately after spawning the same task shape; UI effects
    (`push_screen`, toasts) run from the task on the UI thread.
  - `on_prompt_input_bar_stashed`: move `_persist_stashed_panes` (exclusive flock) off the loop using the
    tracked/background-task pattern (tui_perf rule 2), with optimistic toast/badge and an error toast on failure. Keep
    `_stash_prompt_bar_before_restart` synchronous (deliberate restart-path exception); Phase 3's bounded lock makes it
    safe.
- `_loading_refresh.py`: stop running the refresh coroutine on the pump — schedule `_run_agents_async_refresh` as an
  app-held asyncio task (small helper holding strong task refs) instead of `call_later`, preserving the existing
  coalescing flags (`_agents_refresh_scheduled`, `_agents_loading`, pending flags) and NavigationGate deferral
  semantics. This keeps the single existing refresh fast path (tui_perf rule 4) — only the execution context changes —
  so keys keep flowing while a load is slow or stuck; the apply/finalize leg still runs on the UI thread.
- Audit the remaining app-level async pump handlers (8 exist today) for in-handler IO awaits and apply the same
  hop-to-task shape where needed (e.g. `_prompt_bar_save_xprompt.py`).
- Tests: for each converted handler, assert the handler returns without awaiting a mocked slow store read and that pump
  message processing continues (reuse the Phase 1 fake-pump fixture).

## Phase 3 — Bounded lock waits in the shared prompt-stash store (Rust core)

Crosses the backend boundary → implement in the sibling `sase-core` repo (`crates/sase_core/src/prompt_stash/store.rs`)
plus bindings, then adapt Python callers here.

- Replace unbounded `lock_shared` / `lock_exclusive` with try-lock + bounded retry (total wait on the order of a couple
  seconds, jittered), returning a typed lock-timeout error through the binding.
- Python facade (`src/sase/core/prompt_stash_facade.py`): surface the timeout as a distinct exception; callers degrade
  gracefully (badge read → keep last count; stash/restore/pin → error toast "prompt stash is busy — retry").
- Behavior identical when uncontended. Rust unit test: a held lock makes a reader/writer time out with the typed error
  instead of hanging.

## Phase 4 — Verification

- `just install`, then `just check` (known pre-existing failures to not chase: llm_provider `default_effort` failures;
  pyvision `ChangeSpecProjectFile`; the full test phase may be killed by the sandbox — fall back to targeted pytest
  subsets for touched areas).
- Targeted tests: stall watchdog, prompt-stash handlers, agents loading/refresh suites under `tests/ace/tui/`.
- Manual repro: hold `flock -x ~/.sase/prompt_stash.jsonl.lock` in a shell, then focus-cycle the TUI and stash/restore
  prompts — the UI must stay responsive with degraded toasts; with an artificially slowed refresh load, a
  `tui_pump_stall` record naming the stuck task must appear in `tui_stalls.jsonl`, followed by a recovery row.

## Out of scope / follow-ups

- Lock hygiene of the agent loaders' shared inputs (agent-name registry JSON, artifact-index sqlite) across runner
  processes: Phase 1 telemetry will name the exact stuck await if a freeze recurs, turning that from speculation into a
  targeted fix.
- The axe orchestrator restart observed at 09:33 (noted, likely unrelated).
