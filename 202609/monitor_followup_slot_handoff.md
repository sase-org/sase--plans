---
tier: tale
title: Keep the runner slot held across the monitor-to-follow-up handoff
goal:
  Queued agents are never admitted into the slot a settling monitor is handing to its
  guaranteed --next follow-up, so max_running_agents and %wait(runners=N) thresholds are
  never exceeded by monitor successions.
size: medium
proposed_by: bbugyi200.athena.0gz
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0gz](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0gz.md)
- **COMMITS:**
  - [56754a1](https://github.com/sase-org/sase/commit/56754a17ae5e99b3193d4d8860ae07d64e1c0d35)
    — fix(monitor): hold slot across follow-up handoff

# Keep The Runner Slot Held Across The Monitor-To-Follow-Up Handoff

## Problem

When a sase monitor with a `--next` follow-up settles, other queued agents can be
admitted into the slot the monitor's family still logically owns, so the configured
`max_running_agents` cap (or a `%wait(runners=N)` threshold) is exceeded once the
guaranteed follow-up agent starts. The user-reported suspicion is confirmed; the root
cause is a slot-accounting gap during the monitor-to-follow-up handoff.

### Confirmed root cause

Runner-slot admission works like this today:

1. Parked agents poll every 2 seconds under the host-wide `~/.sase/runner_slots.lock`
   (`src/sase/axe/run_agent_wait_slots.py::wait_for_runner_slot` →
   `_try_claim_runner_slot`) and admit themselves when
   `may_start(running_count, threshold, queue, me)` passes.
2. `running_count` comes from
   `src/sase/core/runner_slots/_admission.py::running_agent_slot_count`, which counts
   one slot per family that has any _occupying_ non-parallel member.
   `is_runner_slot_occupying_record` requires the record to be live; a real monitor
   member occupies from a recorded `pid`, while every other shell kind additionally
   needs `run_started_at`.
3. `src/sase/agent/names/_common.py::is_process_alive` returns `False` the moment
   `stopped_at` appears in the member's persisted `agent_meta.json`, regardless of
   whether the process is actually still running.
4. Both monitor settlement paths persist `stopped_at` **before** spawning the follow-up:
   - Supervisor path: `src/sase/monitor/supervise.py::_finish_monitor` sets
     `meta["stopped_at"]` (line ~427) and persists it via `write_agent_meta_atomic`
     (line ~438), then saves chat history, and only then calls
     `settle_claim_and_followup` (line ~460), which launches the follow-up agent.
   - Proc path: `src/sase/monitor/proc_adapter.py::settle_monitor_artifacts` sets and
     persists `stopped_at` (lines ~170-179); the follow-up launches later in
     `settle_monitor_followup`, driven by the checkpoint order in
     `src/sase/procs/settlement.py::settle_proc_shell` (`artifacts_settled` →
     `followup_settled`).
5. The follow-up is spawned as a detached child
   (`src/sase/monitor/followup.py::launch_followup_agent` →
   `src/sase/shells/followup.py::launch_shell_followup` → `spawn_shell_family_successor`
   → `spawn_detached_child` → `spawn_agent_subprocess`). The child's runner
   (`src/sase/axe/run_agent_runner.py`) boots Python, writes bootstrap meta with its
   pid, expands `#fork` xprompts, and only then — being a serial family member exempt
   from the gate (`wait_for_runner_slot` returns `claim()` immediately when
   `parent_timestamp` is set and the member is not parallel) — records `run_started_at`.

So from the instant `stopped_at` is persisted until the follow-up child records
`run_started_at`, the family occupies **zero** slots: the monitor member is "dead" per
`is_process_alive` and the follow-up does not yet satisfy the occupancy test. That
window spans chat-history saving, workspace-claim transfer, subprocess spawn, Python
boot, and fork-transcript expansion — easily longer than the 2-second waiter poll. A
parked agent slips into the freed slot, then the follow-up starts unconditionally, and
the cap is exceeded.

Note the inverse handoff (starter → monitor) was already fixed by making monitor
occupancy pid-based — see the docstring on `is_runner_slot_occupying_record`. This plan
closes the missing symmetric half (monitor → follow-up). No change to the counting
semantics in `_admission.py` is needed: during the intended overlap the family still
counts exactly once because `running_agent_slot_count` reduces all serial occupying
members of a family to a single boolean.

## Fix Design

Invariant to establish: a monitor member whose settlement will launch a follow-up stays
slot-occupying until the follow-up member itself occupies the family's slot. Concretely:
never persist `stopped_at` (or the done marker) on the monitor member before the
follow-up launch, and after a successful launch, wait (bounded) for the follow-up's
`run_started_at` before persisting the monitor's terminal state. The supervisor process
is alive and pid-recorded throughout the wait, so the monitor keeps occupying; the
exempt follow-up claims `run_started_at` shortly after boot; the two overlap instead of
leaving a gap.

### 1. Plumb the successor's identity through `FollowupLaunchResult`

In `src/sase/shells/followup.py`:

- Extend `FollowupLaunchResult` with the successor's `artifacts_dir: str | None` and
  `pid: int | None` (default `None`).
- `launch_shell_followup` already holds the spawn's `AgentLaunchResult` (which has
  `artifacts_dir` and `pid`); thread those into every `record_launched(...)` call via
  new optional keyword arguments, and have `record_followup_launched` copy them onto the
  returned `FollowupLaunchResult`. Keep the change additive so the gate-shell and
  plan-shell callers of the same helpers keep working unchanged.

### 2. Add a bounded successor-start wait helper

Add to `src/sase/shells/followup.py` (next to `wait_for_starter`) something like:

```python
def wait_for_followup_started(
    artifacts_dir: str,
    pid: int | None,
    *,
    timeout_seconds: float = 60.0,
    poll_seconds: float = 0.5,
) -> bool: ...
```

Semantics:

- Poll the successor's `agent_meta.json` for a non-empty `run_started_at`; return `True`
  as soon as it appears.
- Return `False` early when the successor evidently died before claiming: its
  `done.json` exists, or `pid` is known-dead **and** a re-read of the meta still shows
  no `run_started_at` (re-read after the liveness check to avoid racing a successful
  claim).
- Return `False` on timeout. The caller proceeds regardless of the return value (fail
  open: worst case degrades to today's behavior, bounded by the timeout). 60s/0.5s
  mirrors `DEFAULT_STARTER_SETTLE_TIMEOUT_SECONDS` / `STARTER_SETTLE_POLL_SECONDS`.

### 3. Reorder the supervisor settlement path

In `src/sase/monitor/supervise.py::_finish_monitor`:

- First terminal meta write: persist all terminal fields **except** `stopped_at`. Note
  `write_agent_meta_atomic` writes the whole dict, so either delay setting
  `meta["stopped_at"]` until after this write (then set it in-memory so
  `launch_followup_agent`'s prompt kwargs still see it), or persist a copy that omits
  the key. The chat-history save and `settle_claim_and_followup` call keep their current
  order (the monitor's chat must be saved before the child expands its `#fork` prompt).
- After `settle_claim_and_followup` returns: when the follow-up actually launched (the
  meta's `monitor_followup_outcome` is `launched` or the degraded outcome — cleanest is
  to have `settle_claim_and_followup` / the monitor settlement facade surface the
  `FollowupLaunchResult` so the caller gets `artifacts_dir`/`pid` directly), call
  `wait_for_followup_started(...)` before the `monitor_settled` meta write (which now
  also persists `stopped_at`) and the done-marker write.
- All no-follow-up paths (no `monitor_next_action`, `stopped`, `lost`, not-launchable)
  skip the wait and persist immediately — those slots genuinely free up and parked
  agents should be admitted, exactly as today.

`src/sase/monitor/settlement.py::settle_claim_and_followup` currently returns only an
error string; extend it (and, if needed, the shared
`src/sase/shells/settlement.py::settle_shell_claim_and_followup`) so the monitor callers
can see the launch result without re-reading meta. Keep the shared shell helper backward
compatible for gate/plan shells.

### 4. Reorder the proc settlement path

- `src/sase/monitor/proc_adapter.py::settle_monitor_artifacts`: stop persisting
  `stopped_at` in its `write_agent_meta_atomic` call. Keep `stopped_at` in the in-memory
  meta stored on `state["monitor_meta"]` so follow-up prompt composition and the final
  write still carry it.
- `src/sase/monitor/proc_adapter.py::settle_monitor_followup`: after a successful
  launch, call `wait_for_followup_started(...)` before the existing full-meta write (the
  one that sets `monitor_settled`, line ~250 — that write persists `stopped_at`) and the
  done marker. Defensively set `stopped_at` there if it is somehow missing (e.g. a
  crash-resumed settlement whose sidecar state lost `monitor_meta`), so no terminal
  monitor meta can end up without it.
- The proc supervisor runs `settle_proc_shell` itself and its pid is the one on the
  monitor member's meta (`src/sase/monitor/start.py` records `supervisor.pid`), so the
  member stays live/occupying through the wait on this path too. A supervisor crash
  between the `artifacts_settled` and `followup_settled` checkpoints leaves a short
  unavoidable gap until crash-resume; that residual is accepted.

### 5. Explicitly out of scope

- No changes to `src/sase/core/runner_slots/_admission.py` counting semantics.
- No Rust `sase-core` changes: all affected logic (gate polling, counting inputs,
  monitor settlement) is host-side Python.
- Gate-shell and plan-shell follow-ups: a pending gate member deliberately occupies no
  slot, so there is no held slot to hand off there; their (pre-existing) cap-exception
  behavior is untouched. Sibling handoff paths (retry spawn, pipe) are follow-up work,
  not part of this tale.

## Edge Cases To Preserve

- Degraded follow-up launches (fresh claim on the same workspace, workspace #0 fallback)
  still count as launched → still wait.
- The wait must never hang settlement: hard timeout, and early exit when the child dies
  pre-claim.
- Killed/stopped monitors launch no follow-up → no wait (glossary: only `completed`,
  `failed`, `timeout` launch the follow-up).
- TUI effect: a settling monitor shows as live a few seconds longer, until its successor
  claims. That is accurate — the slot really is held — and needs no TUI change.
- The overlap window must still count as one slot, not two (same family, both serial):
  already guaranteed by `running_agent_slot_count`'s per-family boolean, but lock it in
  with a test.

## Tests

1. `tests/monitor/test_monitor_supervise_followup.py` (extend): with a fake follow-up
   launcher, assert the on-disk `agent_meta.json` has **no** `stopped_at` at the moment
   the launcher runs; assert the successor-start wait is invoked only for launched
   results; assert `stopped_at`, `monitor_settled`, and `done.json` are all persisted
   after the wait returns (including the timeout/fail-open path).
2. Proc path tests (wherever `settle_monitor_artifacts` / `settle_monitor_followup` are
   covered today, extending the existing modules): `settle_monitor_artifacts` leaves
   `stopped_at` unpersisted; `settle_monitor_followup` persists it after the wait; a
   settlement resumed from sidecar state still ends with `stopped_at` set in the
   terminal meta.
3. `src/sase/shells/followup.py` unit tests: `wait_for_followup_started` returns `True`
   on `run_started_at`, `False` early on dead pid / `done.json`, `False` on timeout;
   `FollowupLaunchResult` carries the successor `artifacts_dir`/`pid` through
   `launch_shell_followup`.
4. `tests/test_runner_slots.py` (extend) — occupancy invariants across the handoff:
   - live monitor member (pid recorded, no persisted `stopped_at`) + successor record
     without `run_started_at` → family counts 1 (the fixed mid-handoff state);
   - live monitor member + successor with `run_started_at` (overlap) → counts 1, not 2;
   - monitor with `stopped_at` + successor with `run_started_at` → counts 1;
   - regression documentation of the old gap: monitor with `stopped_at` + successor
     without `run_started_at` → counts 0 (this is exactly why the reorder is required).

## Verification

Follow the repo's two-speed rule: run `just install` if the workspace venv is stale,
then `just fmt` and `just check` after the changes (inline; hand long runs to a
monitor). `just check-full` is a landing gate, not an agent-turn default.
