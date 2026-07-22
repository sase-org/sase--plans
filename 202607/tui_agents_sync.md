---
tier: tale
title: TUI agents-sidecar sync status and comprehensive update
goal: ACE shows actionable agents-sidecar sync state without TUI performance regressions
  and synchronizes every enabled agents repo through tracked manual and comprehensive
  update paths.
bead: sase-8k.7
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 16:05:58
status: wip
---

- **PROMPT:** [202607/prompts/tui_agents_sync.md](prompts/tui_agents_sync.md)

# TUI agents-sidecar sync status, indicator, and comprehensive update leg

## Goal

Complete bead `sase-8k.7` by surfacing agents-sidecar synchronization state in ACE without adding work to render or
keystroke paths, making the distinct top-right indicator actionable through tracked tasks, and extending the existing
`,U` comprehensive update so its confirmed worker synchronizes every enabled project's agents sidecar before any
restart. Preserve per-project failure isolation and do not change or close the parent epic.

## Implementation plan

1. Tighten the agents-sync status boundary for TUI callers.
   - Extend the status facade with an explicit no-network/revalidate-only mode so periodic ticks and previews can
     recompute local `HEAD...@{upstream}` and unexported-agent facts even when the cache is absent or stale without ever
     fetching.
   - Keep forced network recomputation explicit and test the distinction. Base the TUI's longer cadence on its own
     last-recompute state rather than the snapshot's `checked_at`, because local revalidation rewrites that timestamp.
   - Reuse `ProjectSyncStatus`, `SyncStatusSnapshot`, `SyncOutcome`, `get_agents_sync_status`, and `sync_agents` rather
     than duplicating target discovery or sync logic in the presentation layer.

2. Add pump-safe periodic checks and tracked manual sync actions.
   - Add an ACE agents-sync action mixin with a thin synchronous timer callback, a thread worker, an in-flight guard,
     and UI-thread application through `call_from_thread`. Ordinary ticks use revalidate-only status; only the longer
     recompute cadence requests network refresh. Register one timer after first paint and initialize its timer, guard,
     cadence, last-status, and last-recompute state in the startup state initializer.
   - Load new `ace.agents_sync` settings (`check_interval_minutes: 10`, `recompute_interval_minutes: 30`,
     `indicator: true`) from merged config, adding both defaults and schema validation. No config, disk, subprocess, or
     network access may occur in the timer callback, widget setter, render path, click handler, or keystroke path.
   - Add an `action_sync_agents` submission path that runs `sync_agents` as a deduplicated tracked task with an
     `agents-sync` exclusive scope, logs truthful per-project outcomes, and schedules the existing status revalidation
     path on completion. The indicator click invokes this action; the existing `,U` leader/command remains the
     comprehensive manual entry rather than adding another key binding.

3. Add the distinct agents-sync indicator and top-bar integration.
   - Implement `AgentsSyncIndicator` as a pure `Static` widget with an idempotent `set_status` projection over immutable
     statuses. Hide it when no enabled project needs attention; otherwise render green `⇅ N` using `#5FD787` and a
     tooltip that lists each pending project and its behind/ahead/unexported/error reason plus click and `,U` guidance.
   - Compose and export it next to the existing updates badge, add matching auto-width CSS, and update the top-bar order
     assertion. Keep state formatting deterministic and free of I/O.

4. Add agents repositories as the third `,U` comprehensive-update leg.
   - Capture a no-network agents-sync status snapshot while building the confirmation preview and render a third “Agents
     repos” section with per-project current/pending/skipped/error detail. Treat enabled targets as runnable so `,U` can
     perform agents sync even when SASE and agent CLIs are already current; disabled-only or empty inventories remain
     no-ops.
   - After the agent-CLI and SASE legs, call `sync_agents` inside the same tracked worker. Add its ordered outcomes to
     `ComprehensiveUpdateResult`, include per-project errors in aggregate failure semantics without discarding
     successful legs, render a concise `Agents repos: …` summary, and claim the `agents-sync` exclusive scope so
     indicator-click sync and comprehensive sync cannot overlap. Finish this leg before restart handling and refresh the
     indicator from the shared status path on all completions.
   - Update the existing leader/command/footer wording from “SASE + CLIs” to comprehensive wording that includes agents
     repositories; do not add a new keymap value.

5. Verify behavior and performance.
   - Add focused tests for config parsing, timer registration and overlap guard, ordinary no-network ticks versus
     long-cadence recompute, UI-thread application, indicator show/clear/idempotence/tooltip, click-to-tracked-task
     wiring and dedup scopes, status facade revalidate-only semantics, top-bar order, preview truthfulness, third-leg
     execution order, partial failures, aggregate summary, and indicator refresh after either manual path.
   - Add an ACE PNG snapshot for the green pending-sync indicator and intentionally accept only its new golden. Run the
     focused unit and visual tests while iterating.
   - Run `just install`, then the mandatory `just check`. Run the dedicated visual suite if needed to confirm exact PNG
     stability, and exercise the TUI performance benchmark/profile path with `SASE_TUI_PERF=1` to confirm cached j/k p95
     remains below 16 ms and that no periodic work reaches the message pump.

## Completion

Review the final diff for unrelated changes, record concise implementation and validation notes on `sase-8k.7`, close
only `sase-8k.7`, and verify parent `sase-8k` remains open/in progress. Do not create any beads.
