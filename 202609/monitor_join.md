---
tier: tale
title: Join detached ToolRuns with monitors
goal: A monitor can join an existing detached ToolRun, stream and settle its outcome,
  and stop it safely without starting another run.
size: medium
proposed_by: bbugyi200.athena.sase-1cx.5
bead: sase-1cx.5
status: done
---

- **PARENT:**
  [202609/tool_run_escalation.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_run_escalation.md)
- **BEAD:**
  [sase-1cx.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cx/sase-1cx.5.md)

# Monitor joins for detached ToolRuns

Implement phase `sase-1cx.5` of `plan:202609/tool_run_escalation.md` in this sase
checkout. Do not close the parent epic or any ancestor. The prerequisite detached-run
and bounded-follow phases are already closed. Preserve one ToolRun id and its proc
ownership from detach through join and settlement. The `tool_run_escalation` feature
flag gates the public join path until the later guidance phase removes the flag. The
Rust core already exposes `tool_run_join` and `tool_run_release_join`; use these
bindings rather than duplicating join state in Python.

## Implementation

1. Add public `-J/--join RUN` to `sase monitor start` with clear help and an example.
   Validate the agent caller and reject a command remainder, `-c`, `-f`, and `-a` with
   exit 2; for `-f`, direct the caller to `-n`. Reject when the flag is off. Before slow
   lane work or an in-flight handoff marker, inspect the ToolRun and reject ineligible
   shape/ownership with exit 2, or settled, stopped, and joined-elsewhere races with
   exit 1, showing state and a `sase tool show` pointer. Keep parser options sorted and
   refresh the completion snapshot if needed.
2. Extend `StartMonitorRequest` with `join_run_id` and include it in request identity.
   Build the display command from the original ToolRun's canonical `sase tool run` words
   (including ad hoc argv), with default label `tool:<name> (joined)` and reason
   `finish <tool> (joined run)`. In the monitor start transaction, skip wrapper
   resolution and ToolRun reservation for joins. After creating the member and before
   submitting its proc, atomically record the Rust join with monitor id and caller
   agent. Tear down the member on refusal. Release the matching join on submit/start
   failure. Preserve ordinary lane, claim, continuation, handoff, and duplicate-start
   behavior.
3. Launch the monitor proc as `[sys.executable, -m, sase, tool, _join, RUN]`, with
   `monitor_tool_run_id` and `monitor_tool_run_joined: true` in member metadata. Expose
   `tool_run_joined` in start/show JSON and a joined label in human monitor output.
   Never settle a joined ToolRun as monitor-owned in `proc_adapter`; its executing proc
   remains the owner.
4. Add hidden `sase tool _join RUN` using the existing internal worker conventions.
   Reassert the same join from `SASE_MONITOR_ID` as an idempotent replay, then follow
   the ToolRun without a deadline from output offset zero, sending output and stage
   lines into the monitor log. On settlement, render a compact summary and existing
   triage footer, and mirror the run's mapped exit code. On SIGTERM/SIGINT, request a
   stop with a reason naming the joining monitor (and timeout when the monitor
   termination intent says timeout), route it through the proc owner, wait at most 15
   seconds, then exit 143/130.
5. Route `sase tool stop` on a run joined by an active monitor through `stop_monitor` so
   the monitor follow-up is suppressed and its joiner stops the run. If the monitor is
   terminal, stop through the existing proc-owner path. Ensure monitor stop, timeout,
   and lost joiner converge with the detached-run watchdog and preserve `stop_requested`
   settlement reason.
6. Add focused tests for every refusal/exit code, flag-off behavior, join-before-submit
   and rollback, settle-between-validation-and-join, replay, success/failure output and
   codes, pre-join output, stop/timeout/lost-joiner paths, and no-follow-up tool stop.
   Extend the hermetic tool-run smoke harness with detach→join→settlement and one id.
   Document join usage in `docs/monitors.md` and `docs/tool.md`.

## Verification and close

Run focused join/monitor/tool tests and the smoke case, then `just fix` and
`sase tool run check` (install first if bindings are stale). Follow `lint_and_test.md`
before finishing after tracked sase edits. Inspect `sase bead epic-symbols sase-1cx.5`;
resolve each remaining symbol or re-key its Justfile entry to a still-open bead. Record
the required `PROPOSED FOLLOW-UP:` note on `sase-1cx.5` for `-f` with `--join`,
including stage diagnostics mirroring and the join-time workspace fingerprint guard;
record any other discovered follow-up there, without creating beads. Close only
`sase-1cx.5` with a note naming the tests and checks actually verified. A check failure
identical on the clean base is a proposed follow-up and does not hold this phase open.
