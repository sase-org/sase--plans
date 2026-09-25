---
tier: tale
title: Repair bead-work preflight integration before sase-19o landing
goal:
  Bead-work preflight works with ownerless compatibility callers and landing
  verification can complete.
size: medium
proposed_by: bbugyi200.athena.sase-19o.land--3
bead: sase-19o
status: done
---

- **BEAD:**
  [sase-19o](https://github.com/sase-org/sase--beads/blob/main/pages/sase-19o/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-19o.land--3--code](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-19o.land--3--code/README.md)
  - [bbugyi200.athena.sase-19o.land--3--plan](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-19o.land--3--plan/README.md)
- **COMMITS:**
  - [566c96b](https://github.com/sase-org/sase/commit/566c96bcdfb98b9d60621ff66ddaa45a66a5ea01)
    — fix(bead): skip launch-name preflight for ownerless compatibility callers

# Finish sase-19o landing after bead-work preflight drift

## Context

The epic's three phases are closed and its source and post-start commits were audited. A
subsequent commit, `29f18be2b`, added `preflight_bead_work_launch_names` to task and
epic launch. `just check` now reports about 60 bead-work test failures with
`RuntimeError: agent-name reservation batches require an agent owner`. Those older tests
intentionally use ownerless compatibility fixtures; the new preflight calls
`plan_registered_name_reservations`, whose Rust request requires a configured owner. The
latest run also reported four distinct failures in TUI link following, tool-run store
contention, zsh completion timing, and global-state leak detection. The run is monitor
`0a60ecg0vh3w` / ToolRun `a7db83afefcb303150959890d344f6cb`. Preserve the land agent's
existing uncommitted Symvision repairs in `Justfile`,
`src/sase/bead/cli_work_cleanup_selection.py`,
`src/sase/bead/cli_work_cleanup_targets.py`, `src/sase/tool/triage_stage.py`, and
`tests/tool/test_known_continuation.py`.

## Work

1. Resolve the ownerless preflight integration while preserving the existing
   owner-configured collision check and the rule that preflight never mutates bead
   state. Prefer a small Python boundary correction; if supporting ownerless planning
   requires shared backend behavior, make that change in `sase-core` through its linked
   checkout and update the binding and pin as required.
2. Add focused coverage for the ownerless compatibility path and run the affected
   bead-work tests. Investigate the four distinct failures from the latest full
   escalation; fix any regression attributable to this landing. Route genuinely
   unrelated persistent defects through `/sase_new_task`, recording their disposition in
   the epic close note.
3. Run `just fix` and `just check` through the governed tool/monitor path. Do not run
   `just check-full`. Leave a precise verification and follow-up summary for the
   sase-19o land agent to resume its close procedure.

## Acceptance

- Owner-configured name collisions still fail before any bead-store mutation.
- Ownerless compatibility callers do not crash in preflight.
- `just check` passes, or each remaining unrelated failure has a documented disposition
  consistent with the landing instructions.
- The land agent has enough evidence to complete its separate close procedure.
