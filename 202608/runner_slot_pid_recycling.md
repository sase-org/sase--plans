---
tier: tale
title: Stop recycled PIDs from permanently leaking runner slots
goal:
  A dead agent whose PID has been reused by an unrelated process is treated as dead, so
  it stops occupying a global runner slot forever and can never be signalled by the kill
  path.
size: medium
proposed_by: bbugyi200.athena.0et
create_time: 2026-08-27 09:57:35
status: wip
---

# Plan

## Problem

`research.1a.cld` (and later `chop.refresh_docs.sase.9_148270.1`) sat in `QUEUED`
forever even though the host had free capacity. The runner-slot admission gate believed
one more agent was running than actually was, so it never admitted the waiter.

The extra "running" agent was a **phantom**: an agent that died over a month earlier,
whose recorded PID had since been recycled by an unrelated process. The liveness probe
could not tell the difference, so the phantom held a global runner slot permanently.

### Confirmed evidence

Agent `toobig-1.split_file.tests.test_axe_chop_result_protocol.dbb69c9d` (artifacts
under `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202607/19/20260719233730`):

- `agent_meta.json` records `pid: 17549` and `run_started_at: 2026-07-20T11:54:27Z`,
  with no `stopped_at` and no done marker.
- The host has rebooted since then (boot time was `2026-08-27T10:47:40Z`), so PID 17549
  cannot possibly still be that agent.
- PID 17549 today is a **thread**, not a process: `/proc/17549/status` reports
  `Tgid: 17441`, `Pid: 17549`, `Name: dconf worker`. TGID 17441 is
  `/usr/bin/python3 /usr/bin/blueman-applet`.
- `sase agent list -j` corroborated this from the product's own diagnostics: the queued
  agent's `runner_slot_holders` listed
  `toobig-1.split_file.tests.test_axe_chop_result_protocol.dbb69c9d` alongside the ten
  genuinely live agents, with `runner_slots_in_use: 11` against
  `max_running_agents: 10`.
- A sweep over all 10,964 scanned artifact records found exactly one record in this
  state, and both independent signals (pre-boot start time, PID-is-a-thread) fire on it.

### Why the probe was fooled

`is_process_alive()` in `src/sase/agent/names/_common.py` decides liveness like this:

1. `is_process_running(pid)` (`src/sase/ace/hooks/processes.py`) — `os.kill(pid, 0)`
   plus a zombie check. **Signal 0 succeeds for a thread ID**, and Linux allocates TIDs
   from the same number space as PIDs, so a recycled PID landing on a thread reads as
   alive.
2. An anti-recycling guard that reads `/proc/<pid>/cmdline` and accepts the process if
   the bytes contain `b"sase"` **or** `b"python"`. blueman-applet's cmdline is
   `/usr/bin/python3 /usr/bin/blueman-applet`, which contains `python`, so the guard
   passed. This guard accepts essentially any Python process on the machine and provides
   almost no real identity evidence.

Neither step consults the kernel boot id or the process start time, which are the only
facts that actually distinguish "the process I recorded" from "some process that later
got the same number".

### How the leak becomes a permanent stall

`is_runner_slot_occupying_record()` in `src/sase/core/runner_slots/_admission.py`
accepts the phantom record (it is an `ace-run` record, has no done marker, has
`run_started_at`, and the probe says live), so `running_agent_slot_count()` counts it.
`_try_claim_runner_slot()` in `src/sase/axe/run_agent_wait_slots.py` computes
`threshold = get_max_running_agents() - 1` (9 for a limit of 10) and `may_start()`
rejects any waiter while `running_count > threshold`. With one phantom, effective
capacity silently drops from 10 to 9 and never recovers, because nothing ever writes
`stopped_at` onto a record whose process vanished without cleanup.

### Latent hazard in the kill path

The same misidentification makes `sase agent kill` dangerous.
`src/sase/agent/user_kill.py` resolves the recorded PID's process group and issues
`killpg(pgid, SIGTERM)`, escalating to `SIGKILL`. For this phantom, `os.getpgid(17549)`
is `17144`, which on the affected host is the entire VNC desktop session process group —
`Xtightvnc`, `xfce4-session`, `xfwm4`, `xfce4-panel`, `Thunar`, and roughly fifteen more
processes. Killing the "stale agent" would have destroyed the user's desktop session.
This must be fixed alongside the capacity leak, not deferred.

## Approach

Give recorded agent PIDs real identity evidence, and make the liveness probe refuse to
trust a PID that cannot prove it is the process that was recorded.

The repository already has exactly this primitive: `src/sase/procs/identity.py` builds a
`"<boot_id>:<start_ticks>"` token from `/proc/sys/kernel/random/boot_id` and field 22 of
`/proc/<pid>/stat`, and `supervisor_is_alive()` / `supervisor_from_previous_boot()`
consume it. `src/sase/monitor/reconcile.py` and `src/sase/sessions/registry.py` use the
same idea. The agent runner-slot path simply never adopted it.

### Boundary note (read before starting)

Do **not** open a `../sase-core` change for this. The agent-artifact scan is
Rust-backed, but `src/sase/core/agent_scan_facade.py` states explicitly that the scanner
"does not check process liveness (that lives in `sase.ace.hooks.processes` and `/proc`
guards)". Process liveness and its identity evidence are deliberately Python-side host
glue, so this work stays entirely in this repository and needs no `AgentMetaWire` field,
no binding change, and no `sase-core-revision.txt` bump.

Concretely, `is_process_alive()` already falls back to reading `running.json` off
`artifact_dir` when the caller's `meta` dict lacks a PID. Reuse that same idiom to read
the identity token off disk when the caller passed a synthesized dict (the runner-slot
gate and `src/sase/agent/running_listing.py` both pass
`{"pid": ..., "stopped_at": ...}`). The extra read is bounded by the number of records
whose PID is currently live — on the affected host that was 25 out of 10,964 records —
because it only happens after the cheap `os.kill` check passes.

## Steps

### 1. Extract a shared boot-aware process-identity helper

Create `src/sase/core/process_identity.py` holding the generic primitives, currently
private inside `src/sase/procs/identity.py`:

- `process_identity_token(pid) -> str` — `"<boot_id>:<start_ticks>"`, or `""` when
  `/proc` is unavailable.
- `process_identity_matches(pid, recorded) -> bool` — `True` when `recorded` is absent
  or unverifiable (unchanged legacy behavior), `False` on a definite mismatch.
- `identity_from_previous_boot(recorded) -> bool`.
- `pid_is_thread(pid) -> bool` — `True` when `/proc/<pid>/status` reports `Tgid != Pid`.
  Treat an unreadable `status` as "not a thread" so non-Linux and permission-denied
  cases keep today's behavior.
- `current_boot_time_utc() -> datetime | None` — derived from `/proc/uptime`, used by
  the legacy backstop in step 3.

`src/sase/core/occupancy_guard.py` is the precedent for OS-level PID helpers living
under `sase.core`; follow its shape. Refactor `src/sase/procs/identity.py` to delegate
to the new module and keep its `supervisor_identity_token` / `supervisor_is_alive` /
`supervisor_from_previous_boot` public API byte-for-byte unchanged, so proc and monitor
reconciliation behavior does not shift. Existing proc/monitor tests must pass untouched.

### 2. Record the identity token wherever an agent PID is recorded

Write `"process_identity": process_identity_token(os.getpid())` next to every
`"pid": os.getpid()` that describes an agent shell:

- `src/sase/axe/run_agent_runner_bootstrap.py` (`_write_bootstrap_agent_meta`)
- `src/sase/axe/run_agent_helpers_artifacts.py` (follow-up/monitor artifact seeding,
  both the `followup_meta` dict and the `workflow_state.json` payload)
- `src/sase/axe/runner_artifacts.py` (axe-spawned runners)
- `src/sase/axe/run_agent_runner_setup.py` (both call sites)
- `src/sase/axe/run_agent_directive_metadata.py`

Where a PID is recorded for a _different_ process rather than `os.getpid()`
(`src/sase/axe/run_agent_retry_spawn.py`, `src/sase/axe/chop_proposal_launch.py`), pass
that PID to `process_identity_token`. Recording is best-effort: an empty token must
never block the write, and an empty token means "no evidence", not "dead".

### 3. Harden `is_process_alive()`

In `src/sase/agent/names/_common.py`, after the existing `pid`/`stopped_at` checks and
after `is_process_running(pid)` returns `True`, apply in order:

1. **Reject thread IDs.** If `pid_is_thread(pid)`, return `False`. A recorded agent
   shell is always a process leader; a PID that is now only a thread of some other
   process is recycled by definition.
2. **Verify the identity token when present.** Read `process_identity` from the passed
   `meta` dict; if absent, fall back to reading `agent_meta.json` from `artifact_dir`
   (mirroring the existing `running.json` fallback, with the same `FileNotFoundError` /
   `JSONDecodeError` / `OSError` tolerance). If a token is recorded and the live PID's
   current token differs, return `False`. If either token is empty or unreadable, fall
   through — never fail closed on missing evidence.
3. **Legacy pre-boot backstop.** For records with no recorded token (every agent that
   started before this change ships), compare the agent's recorded start
   (`run_started_at`, falling back to the artifact directory's timestamp) against
   `current_boot_time_utc()`. If the agent started before the current boot, return
   `False`: no process alive now can predate the current boot. Skip this rule when the
   boot time cannot be determined.
4. **Delete the `b"sase" or b"python"` cmdline guard.** It is superseded by rules 1–3
   and is actively misleading — it is what let blueman-applet impersonate an agent.
   Removing it is the point of the change, not collateral cleanup.

Keep the function's signature and its `meta: dict[str, object]` contract as they are;
callers such as `src/sase/axe/run_agent_wait_slots.py`,
`src/sase/agent/running_listing.py`, and `src/sase/bead/work_liveness.py` must not need
edits.

### 4. Refuse to signal an unverified PID from the kill path

In `src/sase/agent/user_kill.py`, before resolving a process group or sending any
signal, require that the target PID still passes the same identity predicate
(`pid_is_thread` is `False` and `process_identity_matches` holds). On failure, do not
signal anything: record a distinct non-fatal result (a new reason alongside the existing
`permission_denied` shape, e.g. `identity_mismatch`) and report that the agent is
already gone. Killing a recycled PID's process group is unrecoverable and, on the host
that motivated this plan, would have taken down the entire desktop session — this guard
is the reason that scenario stops being reachable.

### 5. Tests

Add unit coverage for the new predicate, driving `/proc` through a temporary directory
or by monkeypatching the readers in `src/sase/core/process_identity.py` rather than by
spawning real processes:

- A PID whose `/proc/<pid>/status` reports `Tgid != Pid` is dead. Use the real observed
  shape as the fixture: `Tgid: 17441`, `Pid: 17549`, cmdline
  `/usr/bin/python3 /usr/bin/blueman-applet`. This is the exact regression.
- A recorded token that differs from the live PID's token is dead.
- A matching token is alive.
- A record with no token whose `run_started_at` predates the current boot is dead.
- A record with no token whose `run_started_at` follows the current boot keeps today's
  behavior and stays alive.
- An unreadable or absent `/proc` yields the pre-change result (no fail-closed
  regression).

Add a runner-slot regression test next to the existing capacity tests
(`tests/test_run_agent_runner_slot_capacity.py`, `tests/test_runner_slots.py`, and the
shared `tests/_runner_slot_fixtures.py`): a snapshot containing one phantom record plus
`max_running_agents - 1` genuinely live agents must yield
`running_agent_slot_count() == max_running_agents - 1`, and `may_start()` must admit the
head of the queue. Assert on the count and the admission decision, not on log text.

Add a kill-path test asserting that an identity mismatch results in no `killpg` call at
all — inject the `killpg` seam the module already parameterizes and assert it was never
invoked.

Check `tests/test_agent_list_runner_slots.py` and `tests/test_agent_list_entries.py` for
fixtures that rely on the removed cmdline guard, and update them to the new evidence
model rather than weakening the new rules to fit them.

## Out of scope

- Any change under `../sase-core`. See the boundary note above.
- Reconciling the three separate occupancy counters (the admission gate, the
  `sase agent list` projection, and the ACE capacity chip). They were verified to agree
  exactly when sampled simultaneously — an apparent 12-vs-11 discrepancy during
  investigation was time skew between two samples taken minutes apart, not a defect.
- A `sase doctor` check for leaked slots. `sase agent list -j` already exposes
  `runner_slot_holders`, which named the phantom correctly; once liveness is fixed there
  is no leak left to report. Worth revisiting only if a future leak class appears that
  the identity predicate cannot see.
- Clearing the specific phantom record on the affected host. Once this ships, rule 3
  reports it dead and the slot is reclaimed the next time occupancy is computed. Note
  that an agent already parked in `wait_for_runner_slot()` polls inside its own
  long-lived process and therefore keeps running the old code: any waiter already stuck
  when this lands must be relaunched after the update to benefit.

## Verification

Run `just check` before returning. If the scoped test lane escalates or reports an
unusual selection, run `just check-full` through `/sase_monitor` instead of inline.

Manual confirmation on a host with a leaked slot, before and after: run a scan that
computes `running_agent_slot_count()` over `scan_agent_artifacts()` with the runner-slot
scan options from `src/sase/axe/run_agent_wait_slots.py`, and confirm the count drops by
the number of phantom records while every genuinely live agent is still counted. Do not
"verify" by killing anything.
