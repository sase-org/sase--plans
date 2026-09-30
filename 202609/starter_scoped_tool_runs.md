---
tier: tale
title: Starter-scoped detached ToolRuns
goal:
  Agent-only detached ToolRuns are durable under their starter, safely stopped when it
  ends, and observable through the CLI.
size: medium
proposed_by: bbugyi200.athena.sase-1cx.3
bead: sase-1cx.3
create_time: 2026-09-30 08:10:53
status: wip
---

- **PARENT:**
  [202609/tool_run_escalation.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_run_escalation.md)
- **BEAD:**
  [sase-1cx.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cx/sase-1cx.3.md)

# Starter-scoped detached ToolRuns (sase-1cx.3)

Implement only phase `detach-run` of `plan:202609/tool_run_escalation.md`. Read that
artifact and `sase bead read sase-1cx.3 -r "Implement the assigned phase"` before
editing. The prerequisite sase-core phase has landed at commit
`cee9f49aa53ae21958280141d21014f7d44a67fe` on its remote master; its Rust bindings and
wire schema are the source of truth. Do not modify sase-core or close the parent epic.

## Implementation

1. Ratchet `sase-core-revision.txt` beyond the core commit with
   `just ratchet-core-revision`, run `just install`, and verify
   `tools/check_sase_core_rs_bindings`. Add thin `require_rust_binding` facades for
   `tool_run_join`, `tool_run_release_join`, and `tool_run_sync_wait_budget` in
   `src/sase/core/tool_run.py`; keep the budget formula in Rust.
2. Create the `tool_run_escalation` beta flag with `sase flag new`, using the three
   `when-enabled`, `when-disabled`, and `remove-when` sentences in the epic design, and
   paste its printed registry entry. The flag defaults off; gate `--detach` now, and
   leave an explicit gate for the later `--join`, bounded wait, and automatic phases.
   The flag bead produced by `sase flag new` is the planned epic-scaffolding exception
   to the prohibition on discovered follow-up beads; do not create any task beads. The
   final epic phase removes the flag.
3. Add a starter resolver using `SASE_AGENT`, `SASE_AGENT_NAME`, and the runner PID from
   `SASE_ARTIFACTS_DIR/agent_meta.json`. Verify the PID's meaning against the
   launch/invoke code. Record boot and `process_identity_token`, and require matching
   identity plus liveness in `starter_alive`; never treat a reused PID as proof.
   Refactor the proc submission of `execute_handoff` into one shared launcher for `-H`,
   `-d`, and the later automatic path. Preserve `-H` output and semantics byte for byte.
   Extend `reserve_handoff_run` with optional `starter` and `continuation_mode` wire
   fields and add the `tool-run-detached` proc tag only for detached runs.
4. Add mutually exclusive `-d/--detach` beside `-H` in `parser_tool.py`, wire the CLI
   request, and implement an agent-only fail-closed detach path. Refuse humans with the
   `-H` alternative, live owners, parent runs, `-v`, `-T`, and `-H` together at exit 2.
   Allow `-k` and `-x` as envelope continuation modes; accept `long` tools by skipping
   the plain inline duration refusal. If starter resolution, reservation, or launch
   fails, exit 1 and start nothing. Keep the `-H` presentation unchanged; detached
   output adds the starter lifetime and `sase monitor start -J <id> ...` hint; `-q`
   prints only the ID. Update help and completion snapshot if needed.
5. In the `_adopt` worker, run an approximately five-second watchdog only for
   starter-bearing claims. While starter identity is live, leave the run alone. After
   starter death, preserve the run only if its join names an active monitor; share the
   existing proc-store owner classification in `sase.tool.owner`. Otherwise request stop
   with `requested_by: sase` and an exact starter- or joining-monitor-ended reason, then
   stop via the shared owner path used by `sase tool stop`. Let the envelope's
   `continuation_mode` override the agent default. Extract a public stop helper from
   `control_stop.py` and use it both in the CLI and in best-effort `invoke_agent` final
   cleanup. Cleanup must match the current runner identity, inspect unsettled candidates
   from `tool_run_briefs` and `tool_run_show`, and preserve active joined runs. Never
   raise from final cleanup.
6. Suppress settlement notifications for any starter-bearing run. Render `starter` and
   `join` in human `sase tool show`; keep JSON core fields intact. Add the
   `docs/tool.md` “Detached runs” subsection covering starter lifetime, watchdog and
   cleanup, recording failure, and the beta flag.

## Verification and closure

- Test both flag states and all detach refusals; unresolvable starter writes no row;
  `long` accepts; `-k`/`-x` reach the worker; `-H` output remains identical. Exercise
  starter death, an active joined monitor, exact-run cleanup, signaled/`stop_requested`
  settlement and reason, and notification suppression. Add hermetic detach and
  starter-death cases to `tools/smoke_sase_tool_runs`, using a sacrificial sleeper in
  temporary `agent_meta.json`, and update `tests/test_sase_tool_runs_smoke.py`.
- Read `sase memory read lint_and_test.md -r "Verify tracked sase changes"`. Run
  targeted tests while iterating, then `just fix` and `sase tool run check`; do not run
  `check-full`. If an identical check failure reproduces on the clean base, record it on
  this phase as `PROPOSED FOLLOW-UP:` with any existing task reference, and continue
  closure.
- Run `sase bead epic-symbols sase-1cx.3` before close. Resolve every remaining phase
  symbol or re-key its Justfile line to a still-open bead. Record discovered
  out-of-scope work only through `sase bead note sase-1cx.3 'PROPOSED FOLLOW-UP: ...'`.
  Close only `sase-1cx.3` with
  `sase bead close sase-1cx.3 --note "<verification evidence>"`; never close an
  ancestor.
