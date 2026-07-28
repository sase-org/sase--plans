---
tier: tale
title: Record the real approval-surface origin on detached epic launches, then land
  epic sase-9s
goal: 'Detached epic-launch task rows record which surface approved them (ace, telegram,
  cli, or axe) instead of always "api", the last CLAUDE_PROJECT_DIR direct read uses
  the shared runtime-neutral helper, and epic sase-9s is closed, symvision-clean,
  and its plan file marked done.

  '
bead: sase-9s
create_time: 2026-07-26 10:43:48
status: done
---

- **PROMPT:** [202607/prompts/epic_launch_origin_and_land_sase9s.md](prompts/epic_launch_origin_and_land_sase9s.md)
- **PARENT:** [202607/detached_epic_launch.md](https://github.com/sase-org/sase--plans/blob/main/202607/detached_epic_launch.md)
- **BEAD:** [sase-9s](https://github.com/sase-org/sase--beads/blob/main/pages/sase-9s/README.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-9s.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-9s.land/README.md)
  - [bbugyi200.athena.sase-9s.land--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-9s.land.md#member-code)
- **COMMITS:**
  - [f499ca1](https://github.com/sase-org/sase/commit/f499ca1db61a7f2ccef55e6d8d59f57048617d49) — feat(tasks): record detached epic launch origins (sase-9s)
  - [1cb134f](https://github.com/sase-org/sase/commit/1cb134fd1cd8e76f8427aad797844a8c681060b8) — test: stabilize post-rebase suite checks (sase-9s)

# Plan: Record the real approval-surface origin on detached epic launches, then land epic sase-9s

## Context

Epic `sase-9s` made every epic approval submit one detached background task whose command is literally
`sase bead work <plan> --yes-to-all`. All eight phase beads are closed, and the end-to-end verification run (recorded in
the research sidecar at `202607/detached_epic_launch_verification.md`) confirmed every flow works. It found exactly one
gap, and a landing sweep found one small integration leftover. This tale fixes both, then lands the epic.

**Gap 1 — every epic-launch row records `origin: "api"`.** `submit_detached_task()` requires an `origin` because a
detached row has no `session_id`, so `origin` is the only record of where the work came from. The epic plan's `launch`
phase specified that origin as `"ace" | "telegram" | "cli" | "axe"`, and `submit_epic_launch_task()` in
`src/sase/bead/epic_launch.py` accepts an `origin` keyword — but no caller ever threads one through, so
`prepare_epic_launch()` in `src/sase/_plan_approval_epic.py` always lets it default to `"api"`. Today the task surfaces
cannot tell a Telegram approval from an ACE one.

**Gap 2 — `get_tmux_prefix()` still reads `CLAUDE_PROJECT_DIR` directly.** `src/sase/main/plan_approve_handler.py`
(`get_tmux_prefix()`) reads `os.environ.get("CLAUDE_PROJECT_DIR", ".")`. Epic phase `cwd` added the runtime-neutral
helper `provider_project_dir_from_env()` in `src/sase/env_contracts.py` for exactly this lookup; the direct read is the
anti-pattern the "Uniform Agent Runtimes" convention forbids.

## Task 1: Thread the approval-surface origin through detached epic launches

Two seams cover every approval surface. Do not touch the `sase-telegram` repo — it already applies gate responses with
`source="telegram"`, which the gate seam below consumes.

**Declare the origin vocabulary once.** Add a small helper next to the launch code (in `src/sase/bead/epic_launch.py` or
`src/sase/_plan_approval_epic.py`) that maps a gate-response `source` string to a task origin:

- `"tui"` → `"ace"`
- `"telegram"` → `"telegram"`
- `"auto_resolution"` → `"axe"` (auto approval driven by the agent runner)
- anything else (including the executor's `"host"` default) → `"api"`

**Seam A — the shared plan-approval executor.**

- `prepare_epic_launch()` in `src/sase/_plan_approval_epic.py` gains `origin: str = "api"` and forwards it to
  `submit_epic_launch_task()`.
- `execute_plan_approval_response()` in `src/sase/plan_approval_actions.py` gains an `epic_launch_origin` keyword
  (default `"api"`) and forwards it to `prepare_epic_launch()`.
- Callers pass their surface:
  - `src/sase/ace/tui/actions/agents/_notification_plan_gate.py` (the `execute_plan_approval_response` call) → `"ace"`.
  - `src/sase/ace/tui/actions/agents/_notification_modals.py` (the direct `prepare_epic_launch` call) → `"ace"`.
  - `src/sase/main/plan_approve_handler.py` (`_approve_plan_from_cli`) → `"cli"`.
  - `src/sase/main/plan_reject_handler.py` never launches an epic; leave it alone unless a signature change forces a
    mechanical update.

**Seam B — the neutral gate executor.** In `PlanGateAdapter.apply_side_effects` / `on_response`
(`src/sase/notification_gates/adapters.py`), the epic-launch branch already has the applied response in scope and
`run_plan_side_effects` already reads `response.get("source")`. Map that same `source` through the new helper and pass
the result as `origin=` on the `prepare_epic_launch()` call. This covers Telegram, `sase gate`, and auto-approval
without touching those surfaces.

Keep `submit_epic_launch_task()`'s `origin` default of `"api"` as the fallback for direct API callers.

## Task 2: Use the runtime-neutral project-dir helper in get_tmux_prefix

In `src/sase/main/plan_approve_handler.py`, replace the direct read in `get_tmux_prefix()`:

```python
project_dir = os.environ.get("CLAUDE_PROJECT_DIR", ".")
```

with

```python
project_dir = provider_project_dir_from_env() or "."
```

importing `provider_project_dir_from_env` from `sase.env_contracts` at module level.

## Task 3: Tests

- `tests/test_plan_approval_actions.py`: `execute_plan_approval_response(..., epic_launch_origin="cli")` records
  `origin == "cli"` on the submitted detached task; the default still records `"api"`.
- `tests/test_bead/test_epic_launch.py`: `submit_epic_launch_task(..., origin="telegram")` stores that origin on the row
  (extend an existing submit test rather than duplicating the harness).
- The gate seam: extend the epic-launch cases that exercise `PlanGateAdapter` (see `tests/test_plan_gates.py` and the
  gate/executor tests that drive `apply_side_effects`) so a response applied with `source="telegram"` produces a task
  with `origin == "telegram"`, and `source="tui"` produces `origin == "ace"`.
- The source→origin mapping helper: one direct table-driven test including the unknown-source fallback to `"api"`.
- TUI path: extend an existing epic-approval test in `tests/ace/tui/` (for example the notification plan-gate approval
  coverage) to assert the submitted task's origin is `"ace"`.

Check `docs/ace.md` and `docs/cli.md` where they describe detached-task ownership and `origin`; if either implies the
origin distinguishes approval surfaces, make sure the prose matches the now-true behavior (it should need no change or
only a light touch).

## Task 4: Land epic sase-9s

Do this only after Tasks 1–3 are complete and `just check` passes.

1. `sase bead close sase-9s`.
2. After closing, run `just symvision` — epic-symbol whitelist entries for `sase-9s` expire at close. Remove the stale
   whitelist entries and any unused code it reports.
3. Set `status: done` in the YAML frontmatter of the epic's plan file — the PLAN path printed by
   `sase bead show sase-9s` (currently `sase/repos/plans/202607/detached_epic_launch.md` relative to the repo checkout).

## Verification

- `just install`, then `just check` before finishing (workspace directories are ephemeral, so `just install` is not
  optional).
- Focused suites while iterating: `tests/test_plan_approval_actions.py`, `tests/test_bead/test_epic_launch.py`,
  `tests/test_plan_gates.py`, `tests/ace/tui/test_notification_epic_launch.py`, `tests/main/test_task_handler.py`.
- Read `sase/memory/cli_rules.md` via `/sase_memory_read` only if you end up adding or changing CLI options (none are
  expected — `epic_launch_origin` is a Python keyword, not a CLI flag).
