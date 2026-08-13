---
tier: epic
title: A monitor an agent starts must actually run
goal: 'A monitor started from inside an agent survives that agent''s own runner teardown,
  and when it cannot, the failure is loud and immediate: `sase monitor start` refuses
  to return success for a supervisor that is not provably alive, a monitor''s workspace
  is never harvested out from under it, and a monitor''s `--next` action is never
  silently dropped.

  '
phases:
- id: detach
  title: Supervisor survives its starter's teardown
  depends_on: []
  size: medium
  description: 'detach: reparent the supervisor to PID 1 before `start_monitor` returns
    and set its signal dispositions in the first statements it executes, so the starter''s
    runner teardown cannot kill it during its startup window.

    '
- id: ack
  title: Monitor start is not reported until the supervisor proves it is alive
  depends_on:
  - detach
  size: medium
  description: 'ack: have the supervisor publish a startup acknowledgement, block
    `start_monitor` on it, and turn a missing acknowledgement into a torn-down member
    and a hard `MonitorError` the still-live starter agent can act on.

    '
- id: claim
  title: A monitor's workspace claim cannot be harvested behind its back
  depends_on: []
  size: small
  description: 'claim: make the stale-RUNNING sweeper reconcile a monitor before releasing
    its `ace-monitor` claim, and leave the claim alone when reconciliation fails,
    so a live lane''s workspace is never handed to another agent.

    '
- id: followup
  title: The --next action survives a failed claim transfer
  depends_on:
  - claim
  size: medium
  description: 'followup: stop coupling the follow-up launch to a workspace-claim
    transfer that can no longer succeed, and give settlement an explicit degraded-launch
    outcome instead of dropping the follow-up.

    '
- id: visibility
  title: A stalled monitor lane is visible without reading done.json
  depends_on:
  - followup
  size: small
  description: 'visibility: surface dead-on-arrival supervisors and follow-up launch
    failures in the ACE Agents tree, `sase monitor list`, and notifications, and document
    the new guarantees.

    '
- id: exercises
  title: End-to-end exercises for the agent-started monitor path
  depends_on:
  - detach
  - ack
  - claim
  - followup
  - visibility
  size: xsmall
  description: 'exercises: drive a real agent-started monitor on every supported runtime
    and report what the CLI, the tree, and the follow-up agent actually did.'
proposed_by: bbugyi200.athena.zo
create_time: 2026-08-13 13:37:23
status: wip
bead_id: sase-l1
---

- **PROMPT:** [prompts/202608/monitor_supervisor_survival.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/monitor_supervisor_survival.md)
- **BEAD:** [sase-l1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-l1/README.md)

# Plan: A monitor an agent starts must actually run

## Problem

On 2026-08-13 at 12:53:44 agent `zl.w0--code` ran

```
sase monitor start --command 'just test-visual' --timeout 20m --next '<long continuation>'
```

per this repo's Two-Speed Verification convention. No monitor ever ran. The lane went
silent for 17m33s, then reconciled to `failed`, and the `--next` continuation — which
carried the entire remaining plan for `artifacts_split_modes.md` — was never launched.
The project owner noticed only because the ACE Agents tab showed no monitor.

This is not a one-off. Every monitor started from inside a Claude-runtime agent since
08:31 that day died the same way.

## Evidence

Monitor `18tx7ty20h8w` (`zl.w0--mon`, artifacts
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/13/20260813125344`):

- `agent_meta.json` has **no** `monitor_output_path` and **no** `monitor_pgid`.
  `run_supervisor()` writes `monitor_output_path` at `src/sase/monitor/supervise.py:137`
  as its first action after reading meta, and `monitor_pgid` at line 192 immediately
  after spawning the command. Neither ran.
- `live_reply.md` is exactly 73 bytes — only the reconciler's own line. The supervised
  command produced zero bytes.
- `supervisor.log` exists and is **0 bytes**. Since `a54aec6ab`/the
  `monitor_wait_resolution` work the supervisor is spawned with
  `stdout=supervisor.log, stderr=STDOUT` (`src/sase/monitor/start.py:197-217`), so any
  Python traceback would have landed there. Verified live:
  `python -m sase monitor _supervise --artifacts-dir <missing>` prints a full traceback
  to stderr and the whole startup costs ~0.83s.
- `done.json` records `monitor_state: failed`, `monitor_exit_code: null`, and
  `error: "monitor supervisor died without reporting; no process group was recorded"`.

So the supervisor was spawned (its pid and `monitor_supervisor_identity` were recorded
from `/proc`, so it existed), then died **during its own ~0.8s Python startup**, before
installing signal handlers, before reading meta, before spawning the command, and
without raising — i.e. it was killed by a signal.

### What kills it

`sase monitor start` run inside an agent ends in `maybe_handoff_monitor_from_agent()` →
`kill_agent_runner_group()` (`src/sase/main/utils.py:60-90`), which SIGTERMs the agent
runner's process group. From the transcript (`tool_calls.jsonl`) the sequence was:

| time (EDT)   | event                                                      |
| ------------ | ---------------------------------------------------------- |
| 12:53:35     | starter's Bash tool invokes `sase monitor start`           |
| 12:53:44.162 | monitor member created, supervisor Popen'd                 |
| 12:53:44.241 | `.monitor_go` written, `start_monitor` returns             |
| ~12:53:44.3  | `kill_agent_runner_group()` SIGTERMs the runner group      |
| 12:53:58.350 | starter runner finishes its remaining `gh` steps and exits |

The kill lands roughly 10% into the supervisor's 0.8s startup.

`killpg` alone does not explain it. The supervisor is spawned with
`start_new_session=True` (`src/sase/monitor/start.py:214`), and a reproduction of the
exact topology — runner in its own process group, an isolated-pgid child spawning a
`start_new_session=True` grandchild, then `killpg(runner_pgid, SIGTERM)` — confirms the
grandchild **survives**.

What does explain it is a perfect correlation with the starter's agent runtime, because
`setsid()` escapes a `killpg` but not a PPID walk, and at kill time the supervisor is
still a descendant of the runtime's Bash tool:

| starter              | runtime | supervised command                 | outcome         |
| -------------------- | ------- | ---------------------------------- | --------------- |
| `smoke-fail--0`      | codex   | `bash -lc 'echo boom >&2; exit 3'` | ran, exit 3     |
| `z2--code`           | codex   | `just check-full`                  | ran 30s, exit 1 |
| `sase-kp.land--code` | claude  | `just check-full`                  | died, 0 output  |
| `sase-kv.5.w1--code` | claude  | `just test-visual …`               | died, 0 output  |
| `zl.w0--code`        | claude  | `just test-visual`                 | died, 0 output  |

Host-started monitors (`starter_agent: null`, cwd = the main repo: `zl.f1--mon`,
`zm--mon`, `z6.f2--mon`, `sase-kp.land.w1--mon`) are unaffected — nothing kills a
process group they live in.

Two runtimes agreeing and one disagreeing is a _diagnosis_, not a licence to
special-case a runtime. Per this repo's Uniform Agent Runtimes rule the fix must make
the supervisor survive a process-tree teardown from **any** runtime.

### Why nothing noticed

Four independent gaps turned a dead supervisor into a silent 17-minute stall:

1. **`start_monitor` never confirms the supervisor is alive.** It writes `.monitor_go`
   and returns a `MonitorRecord` with `monitor_state="running"`. The starter agent then
   deliberately kills itself. Nothing is left in the lane to notice a dead-on-arrival
   supervisor.
2. **The supervisor's signal handlers are installed too late.**
   `signal.signal(SIGTERM, …)` is at `supervise.py:133`, after `_read_meta()` and after
   the whole `sase` import. Every millisecond before that is a window in which the
   default SIGTERM disposition kills it silently.
3. **The workspace claim was harvested.** `cleanup_stale_running_entries`
   (`src/sase/ace/scheduler/stale_running_cleanup.py:30`) released workspace #10's claim
   because the supervisor's pid was dead. `run_stale_running_cleanup`
   (`src/sase/axe/hook_jobs.py:375-399`) _does_ reconcile monitors first, but wraps that
   call in `except Exception` and proceeds to release regardless. Workspace #10 was then
   handed to `sase-ku.9--plan` at **12:54:35** — 51 seconds after the monitor was
   supposed to have claimed it for a 20-minute test run. Workspace prep stashed
   `zl.w0--code`'s 23-file, 756-line implementation of `artifacts_split_modes.md` into
   `stash@{0}` of `sase_10` (dated 12:54:36). It is recoverable, but only by luck of the
   stash: a hard reset would have destroyed 50 minutes of agent work.
4. **The follow-up was dropped because of that harvest.** `settle_claim_and_followup` →
   `launch_followup_agent` (`src/sase/monitor/followup.py:116-133`) passes
   `retry_transfer_from_pid=<dead supervisor pid>` to `spawn_agent_subprocess`. With the
   claim already gone it failed with
   `Failed to transfer workspace #10 from pid 3333672: workspace #10 with pid 3333672 was not found`,
   and the entire `--next` continuation was discarded into a `monitor_followup_error`
   string in `done.json`.

## Relationship to epic `sase-ku`

`sase-ku` ("sase monitor hardening — a supervisor that cannot silently orphan, wedge, or
lie") shipped the machinery that _detected_ this: `sase-ku.5`'s reconciler (`29cb7924a`)
is what finally moved the phantom to `failed`, and `sase-ku.3`'s durable identity is
what let it decide the supervisor was dead. Its phases `sase-ku.1`–`sase-ku.9` are
closed and `sase-ku.10` (end-to-end exercises, `xsmall`) is in progress.

That epic hardened the supervisor _once it is running_. Nothing in it covers the
**dead-on-arrival** case, because every path it exercised started the monitor from the
host or from a codex-runtime agent, where the starter never tears down a process tree
containing the supervisor. This plan covers that gap. It touches
`src/sase/monitor/start.py`, `supervise.py`, `followup.py`, `settlement.py`,
`transaction.py`, `src/sase/ace/scheduler/stale_running_cleanup.py`, and
`src/sase/axe/hook_jobs.py`; `sase-ku.10` runs exercises and commits no source, so the
two do not conflict, but `sase-ku.10`'s matrix should be re-run after `detach` and `ack`
land because both change what a healthy start looks like.

---

## Phase `detach`: Supervisor survives its starter's teardown

Close the window in which the supervisor can be killed, and remove it from the process
tree that gets torn down.

1. **Reparent before returning.** Spawn the supervisor through a double-fork bootstrap:
   an intermediate process that `setsid()`s, forks the real supervisor, writes the
   grandchild's pid where `start_monitor` can read it, and exits immediately, so the
   supervisor is reparented to PID 1 _before_ `start_monitor` returns. A PPID walk
   performed after that point cannot reach it. Keep `stdin=DEVNULL`, `close_fds=True`,
   and the existing `supervisor.log` capture (`SUPERVISOR_LOG_NAME`) intact across both
   forks.
   - `start_monitor` currently records `process.pid` and `process_identity(process.pid)`
     (`start.py:231-234`). Both must now describe the **grandchild**, not the
     intermediate, or `supervisor_is_alive()` and the reconciler will chase a pid that
     exited by design. Do not widen the gap the existing comment warns about: write the
     pid first, then the identity.
   - `_terminate_supervisor()` (`start.py:509-521`) must signal the grandchild.
2. **Set signal dispositions first.** The supervisor must set `SIGHUP` to `SIG_IGN` and
   install its `SIGTERM`/`SIGINT` handling in the _first_ statements it executes, before
   importing anything expensive and before `_read_meta()`. Today that happens at
   `supervise.py:133`, ~0.8s into startup. The bootstrap is the natural place: set the
   dispositions in the pre-`exec` child, or in a bootstrap entry module whose first
   lines run before the `sase` import graph loads. A `SIGTERM` that arrives during
   startup must result in an orderly `stopped` monitor with a recorded reason, never a
   silent death.
3. **Tests.**
   - A supervisor spawned by `start_monitor` is not a descendant of the spawning process
     (assert its PPID is 1, or that no PPID walk from the spawner reaches it).
   - A PPID-walk sweep that SIGTERMs every descendant of the spawner leaves the
     supervisor running and the monitored command producing output. This is the
     regression test for the reported failure; it must fail against today's code.
   - A `SIGTERM` delivered inside the startup window settles the member as `stopped`
     with an explanatory line in `live_reply.md`, rather than leaving `monitor_state` at
     `running`.
   - `SIGHUP` to the supervisor does not kill it.

## Phase `ack`: Monitor start is not reported until the supervisor proves it is alive

`start_monitor` must never hand back `monitor_state="running"` for a supervisor that is
not provably running, because its caller's next act is to kill itself.

1. **Startup acknowledgement.** After the supervisor has taken ownership — dispositions
   set, meta read, output log opened — it writes a `.monitor_started` marker (alongside
   the existing `.monitor_go` in `src/sase/monitor/transaction.py`) carrying its real
   pid, pgid, `monitor_supervisor_identity`, and the `monitor_id` it is serving. Reuse
   the atomic write-and-`os.replace` pattern already in `_write_monitor_go_marker`.
2. **Block the start on it.** After releasing the launch barrier, `start_monitor` waits
   a bounded interval (a new `MONITOR_START_ACK_TIMEOUT_SECONDS` in `transaction.py`;
   ~20s is comfortably above the measured 0.83s startup) for `.monitor_started`, polling
   the supervisor's liveness too so a dead pid fails fast instead of waiting out the
   budget.
3. **Fail loudly on no acknowledgement.** On timeout or an already-dead supervisor:
   `_terminate_supervisor()`, return the workspace claim to the starter (the reverse of
   the `transfer_workspace_claim` at `start.py:238-246`; it must not be left orphaned or
   released into the free pool), `_teardown_failed_member()` with a specific error, and
   raise `MonitorError`.
4. **Make the starter see it.** `will_handoff_monitor_to_agent_runner()` /
   `maybe_handoff_monitor_from_agent()` must not run when the start raised, so the
   starter agent stays alive, `sase monitor start` exits non-zero, and the agent reads a
   message that names the failure and its recovery options (retry, or run the command
   inline). The `--json` envelope gets the same error. Check the ordering note in
   `will_handoff_monitor_to_agent_runner`'s docstring: output must still be emitted
   before any kill.
5. **Idempotence.** `_start_monitor_locked`'s existing same-fingerprint replay
   (`start.py:112-113`) must not return a torn-down member as if it were live; a member
   torn down for a missing acknowledgement must not block a later start on that lane.
6. **Tests.** Acknowledgement written → start succeeds and the record carries the real
   pid/pgid. Supervisor killed before acknowledging → start raises `MonitorError`, the
   member is terminal `failed`, the starter's workspace claim is intact, no handoff
   marker is written, and the CLI exits non-zero. Acknowledgement arriving late (past
   the budget) does not leave a live supervisor behind.

## Phase `claim`: A monitor's workspace claim cannot be harvested behind its back

A running monitor owns its workspace for the length of its command. Nothing may hand
that workspace to another agent while the monitor's own state still says `running`.

1. **Reconcile before releasing.** In `run_stale_running_cleanup`
   (`src/sase/axe/hook_jobs.py:375-399`), `_reconcile_dead_monitor_supervisors()`
   already runs first but swallows every exception and lets
   `cleanup_stale_running_entries` proceed. Make a reconciliation failure block the
   release of `ace-monitor`-workflow claims for that sweep, and log the reason. Other
   claims may still be swept.
2. **Teach the sweeper about monitors.** `cleanup_stale_running_entries`
   (`src/sase/ace/scheduler/stale_running_cleanup.py:30`) must treat a claim whose
   workflow is `MONITOR_WORKSPACE_CLAIM_WORKFLOW` as releasable only once the owning
   monitor member is terminal. A dead pid alone is not sufficient evidence — that is
   exactly the state a not-yet-reconciled dead supervisor is in.
3. **Reconcile promptly.** The observed phantom lived 17m33s. Confirm what actually
   drives `reconcile_dead_supervisors()` — the ACE loading path
   (`src/sase/ace/tui/actions/agents/_loading_helpers.py:448`) and the
   `stale_running_cleanup` hook job are the only callers — and tighten the sweep
   interval, or reconcile monitors on their own cadence, so a dead supervisor is
   detected in a small number of minutes and the lane's workspace is not
   idle-but-claimed for long.
4. **Tests.** A dead supervisor with a live `ace-monitor` claim: one sweep reconciles
   the monitor and then releases the claim, in that order. A reconciliation that raises
   leaves the `ace-monitor` claim in place and the workspace unavailable to other
   agents. A monitor still `running` with a live supervisor is never swept.

## Phase `followup`: The --next action survives a failed claim transfer

The `--next` action is the whole point of a monitor handoff. It must not be collateral
damage of a claim bookkeeping failure.

1. **Decouple the launch from the transfer.** `launch_followup_agent`
   (`src/sase/monitor/followup.py:116-133`) passes
   `retry_transfer_from_pid=<supervisor pid>` to `spawn_agent_subprocess`. When that pid
   no longer holds the claim, fall back to a fresh claim on the same workspace, and to
   workspace `0` if the workspace has since been taken by another agent — then launch
   anyway. The follow-up prompt must say which happened, because a follow-up that lands
   in a different workspace than the monitor ran in cannot assume the command's
   artifacts are present.
2. **Make settlement's outcome explicit.** `settle_claim_and_followup`
   (`src/sase/monitor/settlement.py:26-71`) currently collapses to launched / not
   launched. Give it three outcomes — launched, launched-degraded (with the reason), and
   not-launchable — and record the degraded reason on the member and in `done.json`
   alongside `monitor_followup_error`.
3. **Never lose the payload.** When the follow-up genuinely cannot be launched, the
   composed prompt must be persisted to the member's artifacts as a durable artifact
   file so the instruction can be replayed by hand, instead of surviving only as an
   error string.
4. **Tests.** Claim transfer impossible → follow-up still launches, degraded reason
   recorded, prompt states the workspace situation. Workspace taken by another agent →
   follow-up launches against workspace 0. Follow-up genuinely unlaunchable → the prompt
   is persisted and the error names its path. Existing behaviour for `stopped` and
   `lost` monitors (`LOST_FOLLOWUP_ERROR`) is unchanged.

## Phase `visibility`: A stalled monitor lane is visible without reading done.json

The project owner found this by noticing an absence. Absences are hard to see.

1. **ACE Agents tree.** A monitor member that is terminal with `monitor_exit_code: null`
   (supervisor died without reporting) and a member carrying a `monitor_followup_error`
   must render distinctly in the Agents tab — the lane is stalled and needs a human, not
   merely finished. Follow the row conventions already in
   `src/sase/ace/tui/widgets/_agent_list_render_agent.py`: the row shows
   `⏱ <monitor_label> (<STATUS>)` and the command belongs only in the detail panel
   (`_agent_monitor_section.py`). Update the help modal and the keybinding footer if a
   new binding or glyph is introduced, per this repo's ACE guidelines.
2. **CLI.** `sase monitor list` gains a signal for "follow-up did not launch" so a
   stalled lane is visible without `--json` plumbing; `sase monitor show` already prints
   the error line and should keep doing so.
3. **Notifications.** `notify_monitor_complete`
   (`src/sase/monitor/settlement.py:97-120`) already appends a follow-up-failure note
   but sends it through the ordinary workflow-complete channel. A dropped `--next`
   strands a lane, so it should raise as an alarm the owner will actually see, not a
   routine completion note.
4. **Docs.** Extend `docs/monitors.md` and the `/sase_monitor` skill source
   (`src/sase/xprompts/skills/sase_monitor.md`) with the startup-acknowledgement
   contract, the new failure mode and its message, and the rule that a monitor owns its
   workspace until it is reconciled. `sase-ku.9` rewrote both of these on the same day —
   extend that text, do not replace it. Run `sase skill init --diff` after editing the
   skill source.

## Phase `exercises`: End-to-end exercises for the agent-started monitor path

Automated tests cannot express "a real runtime tore down a real process tree".

1. Launch a real agent on **each** supported runtime (claude, codex, and any other
   configured runtime), have it start a real monitor over a command that runs for
   minutes, and confirm the supervisor survives the handoff, the command's output
   streams into `live_reply.md`, and the `--next` follow-up agent launches into the
   lane. The claude-runtime row is the reported regression and is the one that must be
   demonstrated green.
2. Exercise the new failure path deliberately: kill the supervisor inside its startup
   window and confirm `sase monitor start` exits non-zero, the starter agent survives
   and can report the failure, and the lane is left clean.
3. Confirm the workspace is not reassigned to another agent while the monitor runs.
4. Report what the CLI, the ACE tree, and the follow-up agent actually did, and note the
   result on `sase-ku` since that epic's `sase-ku.10` covers the adjacent matrix.

## Risks

- **Double-forking loses `Popen.wait()`.** `start_monitor` can no longer reap the
  supervisor, and `_terminate_supervisor`'s `process.wait()` must become a poll on the
  grandchild's liveness. Getting this wrong turns a clean teardown into a hang.
- **The acknowledgement adds latency to every start.** It is bounded by the measured
  ~0.83s startup, but a loaded machine will be slower; the timeout must be generous
  enough that a healthy-but-slow start is never torn down.
- **Blocking claim release on reconciliation can wedge a workspace.** If reconciliation
  is persistently broken, workspaces stay claimed. The blocked release must be logged
  loudly and must not apply to non-monitor claims.
- **Degraded follow-ups in workspace 0** cannot see the monitored command's working
  tree. The prompt must be explicit so the follow-up agent does not act on absent
  artifacts.

## Out of scope

- The zl.w0 lane's own recovery. Its work is intact in `stash@{0}` of workspace
  `sase_10` (23 files, 756 insertions, dated 2026-08-13 12:54:36) and restoring it is
  the owner's call, not this plan's.
- `sase-kr` (a smoke harness for the approved-epic launch monitor). Adjacent, separately
  filed, and unaffected: epic-launch monitors are host-started and never hit this bug.
- The `sase-ku.10--mon-0` start failure
  (`could not claim workspace: workspace #0 with pid 3700673 was not found`), a monitor
  started by another monitor. Phase `claim` may incidentally fix it; if it does not, it
  should be filed separately.
