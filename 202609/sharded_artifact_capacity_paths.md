---
tier: tale
size: medium
title: Fix capacity path derivations broken by day-sharded artifact dirs
goal:
  ACE agent nodes show QUEUED with real queue context for runner-slot waiters, and the
  locked-admission candidate carries its real project name, under the day-sharded
  artifact layout.
proposed_by: bbugyi200.athena.0iu
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0iu](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0iu.md)
- **COMMITS:**
  - [140c8e4](https://github.com/sase-org/sase/commit/140c8e42f268e8b67728be1a2f93cf16b85af776)
    — fix(ace): derive capacity paths from the sharded artifact layout parser

# Fix capacity path derivations broken by day-sharded artifact dirs

## Problem

Every live runner-slot waiter renders as `WAITING` in the ACE Agents tab while
`sase agent list -j` correctly reports the same agents as `QUEUED` with queue positions.
The ACE capacity snapshot is also empty: occupied lanes 0, occupied capacity 0.0, no
waiters, so the capacity header and queue ranks/blockers show nothing even with agents
visibly parked at the admission gate.

Verified live (2026-09-10): six waiters (`0ii`, `0ij`, `0ik`, `0il`,
`chop.refresh_docs.sase.9_665097.1`, `toobig-54.machines_pane.0`) report `QUEUED`
positions 1-6 from the CLI, while replaying the exact TUI compute path
(`compute_apply_loaded_agents` → `refresh_runner_slot_context` with `effective_limit=8`)
yields `WAITING`, `qpos=None` for all six and `occupied_lanes=0`.

## Root cause

Agent artifact dirs are now day-sharded:
`~/.sase/projects/<project>/artifacts/ace-run/<YYYYMM>/<DD>/<timestamp>`
(`DAY_SHARDED_LAYOUT_VERSION` in `src/sase/core/agent_artifact_paths.py`). Several
capacity-path helpers still assume the legacy `.../artifacts/ace-run/<timestamp>` shape
and derive names positionally:

1. `_workflow_dir_name` in `src/sase/ace/tui/models/agent_runner_slots.py` returns
   `Path(artifacts_dir).parent.name`, which is now the day shard (`"10"`), not
   `"ace-run"`. The Rust capacity projection only treats
   `workflow_dir_name == "ace-run"` records as runner-slot participants, so it drops
   every TUI capacity record: zero occupancy, zero waiters, and
   `runner_slot_display_status` never derives `QUEUED` for any agent node. Rewriting the
   record's `workflow_dir_name` to `"ace-run"` in the live repro restored
   `occupied_lanes=9`, `waiters=6`, matching the CLI exactly.
2. `_project_name` in the same file falls back to `path.parents[2].name`, which under
   the sharded layout is `"ace-run"` rather than the project directory. It is usually
   masked by the `project_file` branch but is wrong whenever that attribute is missing.
3. `_project_name_from_artifact_dir` in `src/sase/core/runner_slots/_admission.py` uses
   the same `parents[2].name` heuristic. It feeds `_synthetic_capacity_record` /
   `runner_slot_candidate_record`, so the real locked-admission candidate built in
   `src/sase/axe/run_agent_wait_slots.py` now carries `project_name="ace-run"`. Owner
   keys are `project:family`-scoped, so this poisons candidate identity in the
   weighted-capacity admission path.

The CLI is unaffected because it builds capacity records from Rust scan-wire records
whose `project_name`/`workflow_dir_name` come from the scanner itself. Existing tests
pass because their fixtures use legacy-layout paths (for example
`tests/ace/tui/test_agent_runner_slots.py` builds
`/tmp/project/artifacts/ace-run/<name>`), where `parent.name` happens to be `"ace-run"`.

The regression came in with `81064c144` (`feat(tui): show weighted runner capacity`,
bead sase-z4.4) landing on top of the earlier sharded-layout migration (`6cfa3b171`).
The in-flight sase-z4.6.5 epic does not name this defect: its phases target Rust
candidate-lineage authority, integrated lifecycle acceptance, and release floors. Its
`integrated-acceptance` phase's runtime/CLI/TUI parity comparison would only surface
this incidentally, and only if its fixtures use the sharded layout.

## Fix

Use the canonical layout-aware parser instead of positional path arithmetic.
`parse_agent_artifact_path` (`src/sase/core/agent_artifact_paths.py`, Rust binding)
already handles both legacy and day-sharded layouts and returns `project_name`,
`workflow_dir_name`, and `timestamp`. `src/sase/agent/_restart_planning.py` and
`src/sase/agent/names/_lookup_groups.py` already follow this pattern; mirror it.

1. In `src/sase/ace/tui/models/agent_runner_slots.py`, derive `workflow_dir_name` and
   the non-`project_file` project fallback from
   `parse_agent_artifact_path(artifacts_dir)`.
   - Only call the parser for rows with a real `artifacts_dir`; synthetic `memory://`
     capacity dirs (clan containers and rows without artifacts) must keep their current
     derivations and must not be fed to the parser.
   - Keep the existing `agent.workflow` / ace-run-root fallbacks for parse misses so
     non-standard dirs (for example `.../artifacts/crs/axe`) behave as before.
   - This code runs per row on every Agents load in the worker-thread boundary. Parse
     each distinct `artifacts_dir` at most once per `refresh_runner_slot_context` call
     (a per-call dict keyed by artifact dir is enough; project-alias resolution is not
     involved in parsing). Do not add stat/glob work, per the TUI perf rules.
   - `_capacity_timestamp` (`Path(artifacts_dir).name`) stays correct in both layouts;
     leave it, or route it through the same parsed info when the parser already ran for
     that row.
2. In `src/sase/core/runner_slots/_admission.py`, fix `_project_name_from_artifact_dir`
   to use `parse_agent_artifact_path`, keeping the current `parents[2].name` arithmetic
   only as the fallback for unparseable paths. This corrects the synthetic candidate's
   `project_name` at real admission.
3. Audit the remaining runner-slot/capacity surfaces for positional artifact path
   arithmetic (`parent.name` / `parents[N]` on artifact dirs) and fix any other spot
   that feeds capacity records, owner keys, or queue identity. Known-good already:
   `_restart_planning.py`, `_lookup_groups.py` (both parse first, fall back second). Do
   not sweep unrelated `project_file` derivations; `Path(project_file).parent.name` is a
   different, correct convention.

## Tests

1. Add day-sharded-layout regression coverage to
   `tests/ace/tui/test_agent_runner_slots.py`: an agent with
   `artifacts_dir=".../artifacts/ace-run/<YYYYMM>/<DD>/<timestamp>"` and a live slot
   request must produce a capacity record with `workflow_dir_name="ace-run"` and the
   correct project name, must be counted in occupancy, and must surface as `QUEUED` with
   a queue position after `refresh_runner_slot_context` (snapshot path with
   `effective_limit`, and the no-limit fallback path).
2. Add a sharded-layout case to the admission candidate coverage for
   `runner_slot_candidate_record` asserting the candidate's `project_name` is the
   project directory, not `"ace-run"`.
3. Keep at least one legacy-layout case per touched code path so the fallback stays
   exercised.

## Verification

- Run the focused suites for the touched areas:
  `tests/ace/tui/test_agent_runner_slots.py`, `tests/test_agents_tab_apply_boundary.py`,
  `tests/test_agent_list_runner_slots.py`, and the runner-slot admission tests under
  `tests/` that cover `sase/core/runner_slots`.
- Run the repo's required check recipe before finishing (read
  `sase/memory/lint_and_test.md` first, as it requires).
- Manual sanity check on a host with queued agents: `sase agent list -j` reports
  `QUEUED` rows, and the Agents-tab compute path derives the same statuses, positions,
  and a non-zero capacity snapshot from the same live data.

## Coordination with sase-z4.6.5

The sase-z4.6.5 epic (in progress) is rewiring the locked-admission candidate path
(`run_agent_wait_slots.py`, capacity adapter) in its `admission-authority` phase and
adding runtime/CLI/TUI parity acceptance in `integrated-acceptance`. This tale must not
restructure that candidate wiring: keep the `_project_name_from_artifact_dir` fix a
minimal, behavior-preserving correction of the derived value so it survives (or
trivially rebases onto) the phase work. If the epic's phase commits have already
replaced one of the helpers named above by implementation time, verify the replacement
is sharded-layout-correct instead of re-fixing it, and add the missing sharded
regression tests only.
