---
tier: tale
title: Refresh memory/tui_perf.md with the July 2026 pump-stall learnings
goal: 'memory/tui_perf.md accurately reflects the message-pump freeze class fixed
  by the sase-6c epic and the three related July 2026 tales, stops recommending the
  call_later marshaling anti-pattern, and points freeze diagnosis at the stall watchdog
  — while staying within roughly its current token budget.

  '
create_time: 2026-07-16 13:21:57
status: done
prompt: 202607/prompts/tui_perf_memory_refresh.md
---

# Plan: Refresh memory/tui_perf.md with the July 2026 pump-stall learnings

## Context and evidence

`memory/tui_perf.md` was last substantively updated 2026-06-10..17. Since then, four freeze investigations landed (all
plan files live in the plans sidecar; open it with `sase repo open plans` and read under `202607/`):

- `gh_ref_tui_credential_freeze.md` (07-07) — keystroke-path prompt completion called side-effectful provider
  `resolve_ref` (cloning a partially-typed repo) and git's interactive credential prompt seized the tty.
- `tui_pump_starvation_freeze.md` (07-12) — 5.5-minute total input freeze invisible to the loop-only stall watchdog;
  added the pump-latency beacon + durable recovery rows and established the "spawn slow async work as a free-standing
  loop task" repair (commit `b788ca522`); bounded the prompt-stash flock waits in Rust core.
- `tui_bead_warmup_pump_stall.md` (07-12) — nine 10–40 s pump stalls from `call_later(async warmup)`; coined the key
  insight: "off the event loop is not the same as off the pump".
- `tui_pump_stalls_and_startup.md` (epic `sase-6c`, 07-16, commits `578dad292..b8b7d65e1`) — converted the remaining
  pump-bound async callbacks onto `spawn_pump_free_task()`, time-gated `current_config_token()` and memoized model-alias
  resolution (a render-path glob had frozen the UI for 13 s), took the artifact-index schema rebuild off the
  startup-critical path (cheap staleness metadata check + bounded fallback scan + background rebuild + follow-up
  refresh), and made periodic update checks revalidate-only between full recomputes (the tick interval had equaled the
  cache TTL, so every tick recomputed over the network).
- The `sase-6c` land audit (`sase_6c_landing_gaps.md`) found two pump-bound callbacks missed at phase seams and added
  two durable gotchas: re-sweep all four scheduling APIs after any conversion, and a failed task spawn must release its
  scheduled/loading guard or refreshes stay suppressed for the session.

Against that record the current memory has three problems:

1. **It recommends part of the anti-pattern.** Rule 1 says to "marshal results back with `call_after_refresh()` /
   `call_later()`". Textual awaits async `call_later` / `call_after_refresh` / `set_timer` / `set_interval` callbacks on
   the App's _serial message pump_, so an async callback awaiting `asyncio.to_thread(...)` blocks all key input for its
   full duration — exactly the July freeze class. The codified fix, `spawn_pump_free_task()` /
   `cancel_pump_free_tasks()` (`src/sase/ace/tui/util/pump_tasks.py`), is absent from the memory.
2. **Its diagnosis section misses the primary forensic tool.** The always-on stall watchdog
   (`src/sase/ace/tui/util/stall_watchdog.py`) writes loop- and pump-stall records **with asyncio task stacks** and
   recovery rows to `~/.sase/logs/tui_stalls.jsonl`; every July root cause was found there. The memory never mentions
   it, nor the `SASE_TUI_STALL_*` / `SASE_TUI_PUMP_STALL_*` knobs, nor `docs/perf_runbook.md`.
3. **Several new regression classes have no rule**: render paths doing per-call config stat/glob, startup gating on
   O(archive) index rebuilds, periodic ticks recomputing over the network, and keystroke paths that resolve
   side-effectful refs / spawn prompt-capable subprocesses / take unbounded shared locks.

## Permission note

The user explicitly requested this memory-file update (conversation of 2026-07-16: "improving the memory/tui_perf.md
agent memory file"). Approval of this plan is the explicit user permission that `memory/gotchas.md` requires for editing
`memory/*.md`; the plan changes no other memory file, and `AGENTS.md` / provider shims change only via regeneration.

## Design

Rewrite the body of `memory/tui_perf.md` (keep the frontmatter shape: `type: long`, `parent: AGENTS.md`; extend the
`description` to cover freeze/stall diagnosis, e.g. "... (navigation, refresh, rendering, startup), and before
diagnosing TUI freezes or stalls."). Target ≤ ~75 lines / ~1,400 tokens (currently 58 lines / ~1,029) — every added
token must earn its place; compress surviving rules to pay for the new ones.

Proposed replacement body (implementer may tighten wording but must preserve each rule's substance, and must verify
every cited path/symbol against current source before writing):

```markdown
# TUI Performance Gotchas

Nearly every TUI perf regression has one root cause — slow work reaching the UI thread or Textual's serial app message
pump — and the fixes below are established patterns in this codebase. Reuse them; don't invent new paths.

## Rules

1. **Never block the event loop.** No synchronous disk I/O, JSON parsing, subprocess calls, or `time.sleep` in
   action/message handlers or render paths. Push work off-thread (`asyncio.to_thread()`, `run_worker(..., thread=True)`)
   and marshal thread results back with `call_from_thread`. Do UI mutations (unmount/focus) first, then schedule the
   heavy work.
2. **Off the event loop is NOT off the pump.** Textual awaits async `call_later` / `call_after_refresh` / `set_timer` /
   `set_interval` callbacks on the App's serial message pump: an async callback that awaits anything slow — including
   `await asyncio.to_thread(...)` — blocks every key event until it finishes (July 2026's multi-second-to-multi-minute
   freezes). Timer/pump callbacks must be thin and synchronous; spawn slow async bodies with `spawn_pump_free_task()`
   (`src/sase/ace/tui/util/pump_tasks.py`), cancelled at teardown via `cancel_pump_free_tasks()`. Preserve each site's
   coalescing guards (scheduled/running/pending flags); a failed spawn must release its guard or refreshes stay dead for
   the session. After converting a site, re-sweep all four scheduling APIs for async callbacks awaiting slow work — the
   last epic missed two at phase seams.
3. **Run slow user-initiated operations as tracked background tasks** (agent launches, kill/dismiss persistence,
   ChangeSpec actions): `_submit_tracked_task()` / `_submit_background_task()`
   (`src/sase/ace/tui/actions/task_actions.py`), not ad-hoc fire-and-forget coroutines. Tracked tasks appear in the task
   indicator and Task Queue modal (`t`), dedup duplicate submissions, are counted at quit, and leave inspectable
   records. Shape (see `LaunchTaskMixin` / `CleanupTaskMixin`): optimistic UI stage → sync worker body returning a typed
   outcome → completion effects on the UI thread in `on_complete`.
4. **Re-capture UI state after every `await`.** Selection/tab captured before an await is stale when results land
   (pump-free tasks interleave); re-read the current tab and selected identity before applying, or j/k silently jumps.
5. **Route refreshes through the existing fast path.** Show cached data instantly (`_refilter_agents()`), then schedule
   a background reload (`_schedule_agents_async_refresh()`); coalesce concurrent requests with loading/pending flags
   (last-request-wins). Don't add new refresh code paths.
6. **Prefer selective updates over full rebuilds.** Full agent-list rebuilds are the most expensive UI operation. Use
   `patch_row()` / `try_remove_rows()` (`src/sase/ace/tui/widgets/_agent_list_build.py`); mutate in-memory state
   optimistically and persist off-thread.
7. **Debounce detail panels, never the highlight.** Highlight moves paint immediately; detail-panel updates go through
   `DetailPanelDebouncer` (`src/sase/ace/tui/util/debounce.py`, 150 ms).
8. **Cache disk reads keyed by mtime; render paths never stat/glob.** Don't re-read files or rebuild per-keystroke
   structures on every keypress. Config freshness (`current_config_token()`) is time-gated and model-alias resolution is
   memoized per config token — one render-path glob froze the UI for 13 s; call `clear_config_cache()` in tests that
   edit config. Watch cache keys: too-broad keys serve stale rows.
9. **Keep startup off data-scaled work.** First interactive paint never waits on O(archive) work: a stale artifact-index
   schema is detected by a cheap metadata read, served via the bounded fallback scan, rebuilt in the background, and
   reconciled by a follow-up coalesced refresh. Don't add work before the startup stopwatch ends.
10. **Periodic ticks revalidate; recomputes get their own longer cadence.** Background pollers must not fall through to
    network/full recomputes every tick (update checks did: tick interval == cache TTL). Revalidate the cached snapshot
    on the tick; run the expensive recompute on a separate, much longer interval.
11. **Keystroke paths are read-only and prompt-free.** Completion/typing paths must never call side-effectful resolvers
    (provider `resolve_ref` clones repos and allocates project records), spawn subprocesses that can prompt
    interactively (a git credential prompt seizes the tty and freezes the TUI), or take unbounded shared-store locks
    (bound lock waits in Rust core; degrade with a toast).
12. **Guard programmatic widget updates.** `OptionList` emits `OptionHighlighted` echoes on programmatic
    `highlighted = X` assignments. Set a guard flag and clear it synchronously (`finally:`) — clearing via `call_later`
    races the queued echo and causes cursor jumps/freezes.
13. **Respect activity gates.** Defer non-urgent refresh work while the user is mid-navigation (`NavigationGate`, 250 ms
    window) or typing in the prompt input.

## Measure, don't guess

Perceived causes are usually wrong; profile before and after.

- **Freeze forensics first:** the always-on stall watchdog (`src/sase/ace/tui/util/stall_watchdog.py`) writes loop- and
  pump-stall records with asyncio task stacks, plus recovery rows, to `~/.sase/logs/tui_stalls.jsonl` — it names the
  exact stuck await. Lower the `SASE_TUI_STALL_*` / `SASE_TUI_PUMP_STALL_*` thresholds to make regressions loud in short
  verification runs.
- `sase ace --profile [path]` — pyinstrument profile of the event loop.
- `SASE_TUI_PERF=1` — per-j/k key-to-paint JSONL at `~/.sase/perf/tui_jk.jsonl`; target p95 < 16 ms on every tab.
- `SASE_TUI_TRACE=1` — hot-path span JSONL at `~/.sase/perf/tui_trace.jsonl` (`src/sase/ace/tui/util/trace.py`).
- Benches (p50/p95/max tables): `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` and
  `pytest -s -m slow tests/perf/bench_tui_trace.py`. Full capture/compare recipes: `docs/perf_runbook.md` and
  `tests/perf/README.md`.
```

Rationale for what was kept, added, and dropped:

- Rules 1, 3–7, 12, 13 survive from the current memory (verified still accurate against source on 2026-07-16), lightly
  compressed; rule 1 loses the `call_after_refresh()`/`call_later()` marshaling advice (replaced by `call_from_thread`,
  the pattern actually used in e.g. `actions/update_toast.py`).
- Rule 2 (pump), rule 8's render-path clause, and rules 9–11 are new, each encoding one July 2026 root-cause class.
- The measurement section gains the stall watchdog (primary forensic tool) and defers detail to `docs/perf_runbook.md` /
  `tests/perf/README.md` instead of inlining recipes.

## Implementation steps

1. Read the current `memory/tui_perf.md` and the evidence sources listed above as needed (plans sidecar via
   `sase repo open plans`; note `sase memory read tui_perf.md -r "..."` is the audited read path, but editing the file
   requires reading it directly — that is expected for this explicitly-authorized task).
2. Verify every path/symbol cited in the replacement body against current source (they were all verified on 2026-07-16,
   but re-check at implementation time): `spawn_pump_free_task` / `cancel_pump_free_tasks`
   (`src/sase/ace/tui/util/pump_tasks.py`), `stall_watchdog.py` env-knob prefixes, `~/.sase/logs/tui_stalls.jsonl` (see
   `src/sase/logs/tui_telemetry.py`), `current_config_token` / `clear_config_cache` (`src/sase/config/core.py`),
   `_submit_tracked_task` / `_submit_background_task`, `LaunchTaskMixin`
   (`src/sase/ace/tui/actions/agent_workflow/_launch_tasks.py`) / `CleanupTaskMixin`
   (`src/sase/ace/tui/actions/agents/_cleanup_tasks.py`), `_refilter_agents`, `_schedule_agents_async_refresh`,
   `patch_row` / `try_remove_rows`, `DetailPanelDebouncer`, `NavigationGate`, `sase ace --profile`
   (`src/sase/main/parser_ace.py`), `SASE_TUI_PERF` / `tui_jk.jsonl` (`src/sase/ace/tui/util/perf.py`), `SASE_TUI_TRACE`
   (`src/sase/ace/tui/util/trace.py`), both bench files, `docs/perf_runbook.md`, `tests/perf/README.md`. Fix any drift
   rather than shipping a stale reference.
3. Replace the body of `memory/tui_perf.md` with the content above (frontmatter updated as described). Do not touch any
   other `memory/*.md` file.
4. Run `sase memory init` to regenerate `memory/README.md` and the `AGENTS.md` / provider instruction shims (never
   hand-edit those). Confirm with `sase memory init --check` that there is no drift.
5. Run `just install` then `just check`; both must pass.

## Testing and verification

- `sase plan`-level: no code paths change, so the gate is `just check` (which includes SASE validation of memory and
  generated instruction files) plus `sase memory init --check` showing no drift.
- Sanity-read the regenerated `AGENTS.md` diff: only the `tui_perf.md` description line should change.
- `wc -l memory/tui_perf.md` stays ≤ ~80 lines; eyeball that no rule lost its actionable core in compression.

## Risks

- **Token growth**: the file grows by roughly 300–400 tokens. Accepted deliberately — each new rule encodes a distinct
  multi-hour freeze investigation; the alternative (future agents re-deriving the pump/loop distinction) is far more
  expensive. The implementer should still trim any wording that doesn't change behavior.
- **Reference drift** between proposal and implementation: mitigated by step 2's re-verification.
- **Generated-file mismatch**: mitigated by `sase memory init` + `--check` + `just check`.
