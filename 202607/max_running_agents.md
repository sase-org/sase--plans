---
create_time: 2026-07-12 18:12:53
status: wip
prompt: 202607/prompts/max_running_agents.md
tier: epic
---
# Plan: Global Cap on Concurrently Running Agents (`max_running_agents` + `%wait(runners=N)`)

## Goal

Add the ability to limit how many sase agents may be RUNNING at any given time:

1. A new user-config field, **`max_running_agents`** (top-level in `sase.yml`, int, default **10**, minimum 1).
2. A new **`runners=` keyword argument on the existing `%wait` directive** for per-prompt overrides (e.g.
   `%wait(runners=0)` waits until _no_ agents are running before starting).
3. Agents held back by the cap use the **standard `WAITING` status**, but the display makes it clear at a glance that
   they are waiting for a _runner slot_ (not a dependency or time floor), including a live "running / allowed" readout
   and queue position.

The feature must be **intuitive** (one mental model shared by config and directive), **reliable** (race-free admission,
no deadlocks, no starvation, works even when the axe daemon is down), and **beautiful** (polished TUI treatment with PNG
snapshot coverage).

## Background (from codebase exploration)

Two separate "agent worlds" exist:

- **Interactive/background user agents** (what `sase run` launches, what the ACE Agents tab and `sase agent list` show).
  State is on-disk markers per artifact dir (`agent_meta.json`, `waiting.json`, `ready.json`, `done.json`) + PID
  liveness. **These have no concurrency cap today** — `src/sase/agent/launch_spawn.py` spawns immediately. _This is the
  world we are capping._
- **Axe ChangeSpec runners** (hooks/mentors/CRS), already capped by `axe.max_hook_runners` / `axe.max_agent_runners`
  (default 3) via `RunnerPool`/`SharedRunnerPool` (`src/sase/axe/runner_pool.py`). **Unchanged by this feature** and
  excluded from the new count.

The existing `%wait` machinery we build on:

- Parsing: `src/sase/xprompt/_directive_collect.py` whitelists `%wait` kwargs (only `time=` today); parsed values land
  in `PromptDirectives` (`src/sase/xprompt/_directive_types.py`), are resolved in `_directive_values.py`, and assembled
  in `_directive_extract.py`.
- Runtime: the spawned agent process (`src/sase/axe/run_agent_runner.py`) calls `wait_for_dependencies()`
  (`src/sase/axe/run_agent_wait.py`), which writes a `waiting.json` marker (→ `WAITING` status) and polls every 2s; the
  axe `wait_checks` chop (`src/sase/scripts/sase_chop_wait_checks.py`, 10s interval) writes `ready.json` for resolved
  named deps. Time floors are resolved in-process (no axe needed). After the wait, `record_run_started_at()`
  (`src/sase/axe/run_agent_markers.py`) flips the agent to RUNNING.
- Display: TUI loaders (`src/sase/ace/tui/models/_loaders/_meta_enrichment_filesystem.py` and
  `_meta_enrichment_wire.py`) map `waiting.json` → `WAITING`; row rendering in
  `src/sase/ace/tui/widgets/_agent_list_render_agent.py` (Amethyst `#AF87FF`, inline countdown suffixes); wait reason
  detail in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`; interactive editing via the `w` wait modal
  (`src/sase/ace/tui/modals/wait_modal.py`). The Rust core's artifact scan projects `waiting.json` as
  `WaitingMarkerWire` (`src/sase/core/agent_scan_wire_markers.py`) for the fast wire path — the only Rust surface this
  feature touches.
- Counting: `list_running_agents()` (`src/sase/agent/running.py`) already computes "live root agents" globally (artifact
  scan + no `done.json` + PID alive, children skipped). This predicate is our definition of "running".

Config fields are defined in `src/sase/config/sase.schema.json` + `src/sase/default_config.yml` (kept in sync by
`tests/test_config_schema.py`), loaded via `load_merged_config()` (`src/sase/config/core.py`). Adding a field requires
**no Rust changes** (the schema is data).

## Semantics (one mental model)

Every root agent, as the **final stage** of its pre-run wait (after named deps and any `time=` floor), must pass the
**runner-slot gate**:

> An agent may start when the number of currently-running root agents is **≤ its threshold**.

- Threshold defaults to **`max_running_agents − 1`** (so total running, including this agent, never exceeds the cap:
  default 10).
- **`%wait(runners=N)`** overrides the threshold to exactly **N** for that launch: "start only when at most N agents are
  already running." `runners=0` → start only when nothing is running. `runners=25` → this launch tolerates up to 25
  concurrent runners (a per-prompt raise).
- "Running" = the `list_running_agents()` predicate: root user agents across **all projects**, past the gate, not done,
  PID alive. Excluded from both counting and gating: **child agents** (prevents parent-holds-slot-waits-on-child
  deadlocks), workflow python/bash steps, and axe ChangeSpec runners (governed separately by `axe.max_*_runners`).
  `%hide` agents are ordinary agents: counted and gated.
- **Admission order is strictly first-come-first-served** by slot-queue entry time among live waiters. This makes
  behavior predictable and starvation-free, and gives `runners=0` clean **drain-barrier semantics**: agents launched
  after it queue behind it until it has started. Fast path: an agent whose threshold is satisfied at its first check
  _and_ with no live earlier waiter queued starts immediately (uncontended launches stay instant).
- The gate re-reads the merged config and its own `waiting.json` on every poll (~2s), so **raising `max_running_agents`
  mid-flight releases queued agents within seconds**, and wait-modal edits to a parked agent's `runners` value apply
  live.

### Directive grammar

```
%wait(runners=3)              # start when ≤3 agents already running
%wait(runners=0)              # drain barrier: start when nothing is running
%w(runners=2)                 # alias works as usual
%wait(agent1, time=5m, runners=1)  # stages: deps → time floor → slot gate
```

Validation: `runners` must be a non-negative integer; reject duplicates across multiple `%wait` occurrences, negatives,
and non-numeric values with a `DirectiveError` (mirroring the existing `time=` conflict handling). No shorthand xprompt
is added.

## Key design decisions

1. **Consumer-side admission under a global file lock (not the axe chop).** The waiting agent's own poll loop performs
   check-and-claim inside a `flock` critical section on a well-known lock file under `~/.sase/` (pattern:
   `SharedRunnerPool` in `src/sase/axe/runner_pool.py`). **Invariant: the marker that makes an agent count as running is
   written inside the same critical section as the count check** — this closes the check-then-claim race with zero
   daemon dependency. The axe `wait_checks` chop is untouched; the gate works even when axe is down (unlike named-dep
   waits, which already require axe). Sase is single-host, so `flock` is sufficient.
2. **Reuse the WAITING flow rather than blocking at spawn.** Spawn stays immediate; over-cap agents park exactly like
   `%wait` agents do (`waiting.json` → `WAITING` → poll). This reuses status buckets, glyphs, TUI display, kill
   handling, and the wait modal for free, and keeps launch UX snappy.
3. **Pure-logic core module.** Counting predicate, FIFO ordering, and admission decisions live in a new
   `src/sase/core/runner_slots/` package (structured like `src/sase/core/wait_dependency_resolution/`): pure functions
   over scanned records, unit-testable without processes. Runtime glue (lock, marker writes, polling) stays in
   `src/sase/axe/`. Consistent with the existing boundary: wait _logic_ is Python; the Rust core only projects on-disk
   markers for the fast scan path.
4. **Naming.** Config: top-level `max_running_agents` — mirrors the RUNNING status, and deliberately avoids the
   `axe.max_agent_runners` namespace (different world). Directive kwarg: `runners`, per the feature request. Marker
   fields (see below) use the `wait_runners` / `slot_*` prefix.
5. **New marker fields** (all optional, backward compatible):
   - `agent_meta.json`: `wait_runners` (int) — present only when the directive supplied it.
   - `waiting.json`: `wait_runners` (int, effective threshold), `wait_runners_explicit` (bool, directive vs config
     default — drives display form), `slot_requested_at` (ISO timestamp — the FIFO queue key). The set of live
     `waiting.json` markers with `slot_requested_at` _is_ the queue; no separate queue store.

## Display design (Phase 4 spec)

- **Row (Agents tab):** keep `WAITING` in the standard bold Amethyst, then a compact slot suffix in the same dimmed
  style as today's countdown suffixes, using the Running glyph `▶` as the visual key that this wait is about _running
  agents_:
  - Config-gated: `WAITING ▶10/10` (live running count / `max_running_agents`).
  - Explicit `runners=N`: `WAITING ▶7→0` (live count → target threshold), since `runners=0` rendered as a fraction
    (`7/1`) would read as a bug.
  - The live count is computed once per TUI refresh from the same scan snapshot the loaders already hold (no extra
    scan), and must participate in the row render-cache key.
- **Detail/header pane:** extend the existing `Wait:` line with a runners entry in plain words, e.g.
  `runners: 10/10 in use · queue #2 of 3` or `runners ≤ 0 (drain barrier) · 3 still running`. Dependency badges and
  time-floor rendering are unchanged; the slot stage renders after them, matching the actual stage order.
- **Wait modal (`w`):** add a `runners` field alongside deps and time; saving rewrites `waiting.json`, which the parked
  agent honors on its next poll.
- **Other surfaces:** `sase agent list -j` / `src/sase/integrations/agent_list_entries.py` gain the slot fields so
  mobile/Telegram surfaces and the `sase_agents_status` skill can state the wait reason.

Exact glyph/spacing may be refined during implementation, but the information content (slot-wait is visually distinct
from dep/time waits; live count vs. allowed; queue position in the detail pane) is required, and PNG snapshots lock in
the final look.

## Phases

Each phase is a self-contained unit of work for a separate implementing agent, lands with its own tests and docs, and
leaves the repo green (`just check`). Later phases depend on earlier ones as noted. Phase agents touching xprompt
parsing must first read `memory/xprompts.md`; agents touching TUI loaders/rendering must first read `memory/tui_perf.md`
(via the long-memory read procedure).

### Phase 1 — Config field + directive parsing + metadata plumbing (inert)

Foundation only; parsed and persisted but not yet enforced.

- Config: add top-level `max_running_agents` (int, default 10, `minimum: 1`) to `src/sase/config/sase.schema.json` and
  `src/sase/default_config.yml`; add a cached accessor `get_max_running_agents()` following the `get_timezone()` pattern
  (`src/sase/core/time.py`).
- Directive: extend the `%wait` kwarg whitelist in `src/sase/xprompt/_directive_collect.py` (currently `time=` only)
  with `runners=`; collect raw values; add a resolver in `src/sase/xprompt/_directive_values.py` validating non-negative
  int / no duplicates; add `wait_runners: int | None` to `PromptDirectives` (`src/sase/xprompt/_directive_types.py`);
  assemble in `src/sase/xprompt/_directive_extract.py`. Update the directive-arg completion candidates so `%wait(`
  suggests `runners=` in the prompt widget.
- Plumbing: thread `wait_runners` through `AgentInfo` and `agent_meta.json` in `src/sase/axe/run_agent_directives.py`.
- Tests: `tests/test_config_schema.py` (accept + reject-below-minimum), directive parsing suites
  (`tests/test_directives_wait.py`, `tests/test_directives_time.py` patterns), meta-write coverage, completion-candidate
  tests.
- Docs: `docs/configuration.md` (new field section/table) and `docs/xprompt.md` (directive table row, syntax examples,
  and a subsection explaining runner-slot semantics incl. the drain barrier).

### Phase 2 — Admission engine + runtime enforcement (depends on Phase 1)

The heart of the feature.

- New `src/sase/core/runner_slots/` package: running-count predicate (reusing/refactoring the `list_running_agents()`
  predicate over a scan snapshot), live-waiter queue derivation from `waiting.json` markers (PID-alive filtered, ordered
  by `slot_requested_at` with a deterministic tiebreak), and the pure admission decision
  (`may_start(count, threshold, queue, me)`).
- Runtime gate in `src/sase/axe/run_agent_runner.py` / `run_agent_wait.py`: an **unconditional** final wait stage for
  root agents (children and workflow steps exempt), running after named deps and time floors. On contention it
  writes/refreshes `waiting.json` with the new slot fields and polls (~2s, honoring `was_killed()` exactly like the
  existing loop). Check-and-claim executes under the global `flock`; the running marker is written inside the critical
  section (the invariant above). Effective threshold resolved per poll: directive `wait_runners` if present, else
  `get_max_running_agents() − 1`.
- Robustness: dead/stale waiters must never block the queue (PID checks); crashed runners free slots via the existing
  liveness predicate; the gate must fast-path to zero parking when uncontended; behavior with axe down must be
  identical.
- Tests: pure-logic unit tests (thresholds, FIFO, barrier, stale PIDs, tiebreaks); runner integration tests with fake
  artifact dirs (patterns: `tests/test_run_agent_wait.py`, `tests/test_run_agent_runner_wait_queue.py`); a concurrency
  test where multiple claimants race one lock; live config-change release test.
- Docs: `docs/troubleshooting/` entry for "agent WAITING on runner slots".

### Phase 3 — Rust scan projection of the new marker fields (independent of Phase 2)

Small, surgical change in the sase-core repo (per the `rust_core_backend_boundary` instructions — update the Rust
wire/API and tests there first, then the Python callers here).

- sase-core: extend the agent-scan waiting-marker projection (`agent_scan/{scanner,wire}.rs`) with `wait_runners`,
  `wait_runners_explicit`, and `slot_requested_at`; Rust tests; follow the repo's release/versioning conventions for the
  `sase_core_rs` binding.
- This repo: extend `WaitingMarkerWire` (`src/sase/core/agent_scan_wire_markers.py`) and the
  `src/sase/core/agent_scan_facade.py` rehydration; parity tests asserting the wire path exposes exactly what the
  filesystem path reads.

### Phase 4 — Display & surfaces (depends on Phases 2 and 3)

Implements the display design above. Read `memory/tui_perf.md` first.

- Agent model fields (`src/sase/ace/tui/models/agent.py`), both meta-enrichment loaders, row render
  (`_agent_list_render_agent.py`) with render-cache key updates (`_agent_list_render_cache.py`), detail header
  (`_agent_display_header.py`), wait modal `runners` field (`wait_modal.py`),
  `src/sase/integrations/agent_list_entries.py` + agent-list JSON.
- Live running count derived from the existing refresh snapshot (no additional scans; verify refresh cost is unchanged).
- Tests: loader/render unit tests, wait-modal tests (`tests/ace/tui/test_wait_modal.py` pattern), and **PNG visual
  snapshots** (`just test-visual`) for: config-gated row, `runners=0` barrier row, and the detail pane with queue
  position.
- Docs: `docs/ace.md` status-display notes.

### Phase 5 — Hardening, end-to-end verification, doc sweep (depends on all)

- End-to-end scenarios using the fake agent provider (`src/sase/fakey/`): launch cap+K agents and assert at most cap run
  concurrently and release in FIFO order; `runners=0` drains and blocks later launches until it starts; raising
  `max_running_agents` mid-flight releases waiters; killing a parked agent leaves the queue healthy; a crashed RUNNING
  agent frees its slot; everything works with axe stopped; child agents and `%repeat` chains behave.
- Perf validation: parked-agent poll cost and TUI refresh cost within existing budgets.
- Doc sweep for consistency across `docs/configuration.md`, `docs/xprompt.md`, `docs/ace.md`, troubleshooting; fix any
  drift introduced by earlier phases. (No edits to `memory/*.md`, `AGENTS.md`, or provider shims — those require
  explicit user permission.)

## Risks & mitigations

- **Deadlock via gated children:** children are exempt from gate and count (decision above).
- **Barrier starvation / queue jumping:** strict FIFO admission; documented drain-barrier behavior.
- **Overshoot races:** single-flock check-and-claim invariant; concurrency test in Phase 2.
- **Dead waiters wedging the queue:** PID-alive filtering everywhere the queue is derived.
- **TUI perf regressions:** count from the existing snapshot only; render-cache keys updated; `memory/tui_perf.md`
  review mandated; perf validation in Phase 5.
- **Semantic confusion `runners=0` vs cap:** docs explain the shared threshold model explicitly; display renders the two
  forms differently (`▶10/10` vs `▶7→0`).

## Out of scope

- Capping axe ChangeSpec runners (already covered by `axe.max_*_runners`).
- Per-project or per-provider caps, priority classes, or preemption of running agents.
- An environment-variable override for the cap.
- Notifications when an agent parks for a slot (status display is the surface).
