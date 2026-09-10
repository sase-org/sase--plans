---
tier: epic
title: Reap leaked agent descendant processes
goal: "Detached processes left behind by a finished agent run are detected, attributed
  to the run that spawned them, surfaced by sase doctor, and reaped instead of taxing
  the shared host indefinitely.

  "
phases:
  - id: core-planner
    title: Leak classifier in sase-core
    depends_on: []
    size: medium
    description: "core-planner: add an agent_leak module to sase-core with wire types
      and a pure classifier that decides which candidate processes are leaked from their
      agent-run stamp, the live-run set, an age grace, and protected pids, then expose
      it through the sase_core_rs binding.

      "
  - id: proc-adapter
    title: /proc inventory adapter and reaper
    depends_on:
      - core-planner
    size: medium
    description: "proc-adapter: add the Python host adapter that walks /proc for
      SASE_AGENT_TIMESTAMP-stamped processes, resolves live runs and protected pids,
      calls the core classifier, and reaps with a SIGTERM/SIGKILL sequence guarded
      against pid recycling.

      "
  - id: doctor-check
    title: sase doctor resource check
    depends_on:
      - proc-adapter
    size: small
    description: "doctor-check: register a resource check that reports leaked processes
      per owning run with age and accumulated CPU, passing clean on a quiet host and
      escalating to a failure above a configurable threshold.

      "
  - id: reap-chop
    title: Observe-only axe chop and reap command
    depends_on:
      - proc-adapter
    size: medium
    description: "reap-chop: add a builtin chop that scans and logs leaks without
      killing, plus the operator-facing sase agent reap command, with automatic killing
      behind a config flag that stays off until a clean soak justifies it.

      "
  - id: session-containment
    title: Contain agent descendants at launch
    depends_on: []
    size: large
    description:
      "session-containment: evaluate systemd user scopes, new sessions with killpg, and
      cgroup delegation against how agents are launched today, then implement the chosen
      containment behind a default-off flag or close the phase with a written
      justification for declining it."
proposed_by: bbugyi200.athena.v3
create_time: 2026-09-09 20:00:12
status: wip
---

- **PROMPT:**
  [prompts/202608/leaked_agent_process_reaper.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/leaked_agent_process_reaper.md)

# Plan: Reap leaked agent descendant processes

## Problem

A SASE agent run can leave behind detached OS processes that outlive the run forever.
Nothing in SASE notices, and nothing ever reaps them. On a shared 64-core dev host that
is not a cosmetic problem: it silently taxes every other agent, every `just check`, and
every interactive TUI session until a human happens to run `uptime`.

This is not hypothetical. On 2026-08-07 a completed agent run left 96 orphaned
`python3 -c "while True: pass"` processes running at nice 5. They burned roughly 64
cores for about 50 minutes and drove the host to a load average of 134 before a human
noticed and asked what was wrong. The processes were reparented to PID 1 and were still
running long after the agent that created them had finished, committed, and exited.

### How it happened

The agent was calibrating the contract-set budget guard and deliberately synthesized CPU
load. Its single shell command was:

```bash
for i in $(seq 1 96); do (python3 -c "
while True: pass" &) ; done
command sleep 2
uptime
.venv/bin/python /tmp/measure_contract.py 2
pkill -f "^python3 -c"
echo "spinners killed"
command sleep 1
uptime
```

Three independent things went wrong, and all three are worth naming because the fix has
to survive any one of them recurring:

1. **The cleanup pattern never matched.** `pkill -f "^python3 -c"` anchors on `python3`,
   but `argv[0]` for a pyenv-shimmed interpreter is an absolute path ending in
   `/bin/python3`. The `^` anchor made the pattern unmatchable. Confirmed after the
   fact: `pgrep -f '^python3 -c'` matched 0 processes while
   `pgrep -f 'while True: pass'` matched 96.

2. **The failure was invisible.** The steps were joined with `;`, not `&&`, so
   `echo "spinners killed"` printed unconditionally after `pkill` exited non-zero. The
   agent read a success message that was simply false.

3. **The evidence was in the output and got skipped.** The command's own trailing
   `uptime` printed `load average: 81.54, 31.62, 17.20` immediately after
   `spinners killed` — load climbing hard, which is the opposite of what a successful
   reap looks like. The agent moved on anyway.

`( ... &)` double-forks, which detaches the children from the shell's job control and
reparents them to PID 1. So when the agent's shell exited, nothing owned them and
nothing cleaned them up.

### Why nothing in SASE caught it

Every mechanism that sounds like it should have caught this operates on bookkeeping
records, not on operating-system processes:

- `cleanup_stale_running_entries` (`src/sase/ace/scheduler/stale_running_cleanup.py`,
  driven by the `stale_running_cleanup` chop) walks RUNNING entries and releases
  _workspace claims_ whose PID is gone. It reads process liveness; it never kills
  anything, and it only looks at PIDs SASE already recorded.
- `cleanup_orphaned_workspace_claims` (`src/sase/ace/scheduler/orphan_cleanup.py`,
  driven by the `orphan_cleanup` chop) releases _workspace claims_ for reverted
  ChangeSpecs. Same story: claims, not processes.
- `sase doctor`'s resource family (`src/sase/doctor/checks_resources*.py`) covers disk
  free, ulimits, inotify watches, and chezmoi. There is no check for host CPU load and
  no check for stray agent descendants.
- The agent teardown path has no process-tree kill. `os.killpg` helpers exist
  (`src/sase/ace/hooks/processes.py`, `src/sase/ace/tui/task_subprocess.py`,
  `src/sase/agent/partial_launch.py`) but they target specific tracked PIDs — hook
  runners, mentor runners, TUI tasks — never the agent's own descendants.
- `sase-core`'s `agent_cleanup` module
  (`../sase-core/crates/sase_core/src/agent_cleanup/`) plans kill/dismiss/skip decisions
  for _tracked agents_ the TUI knows about. An untracked great-grandchild of an agent
  shell is not in its input at all.

### The fingerprint that already exists and nothing consumes

Every process descended from an agent run inherits that run's environment.
`run_agent_exec_markers.py` stamps `SASE_AGENT_TIMESTAMP`, and the launch path stamps
`SASE_AGENT=1`, `SASE_GH_WORKSPACE_NUM`, and the artifact dir. The leaked spinners all
carried:

```
SASE_AGENT=1
SASE_AGENT_TIMESTAMP=20260807161309
SASE_GH_WORKSPACE_NUM=<n>
```

That is an exact, forgery-proof answer to "which agent run owns this process?" —
readable from `/proc/<pid>/environ` — and no code in the repo reads it for this purpose.
Cross-referencing that stamp against the set of live agent runs identifies leaked
descendants with no heuristics and no guessing. The manual cleanup that resolved this
incident used exactly that join and killed 96 processes with zero false positives.

## Approach

Two independent layers, because each covers the other's blind spot:

- **Detection and reaping** (`core-planner` → `proc-adapter` → `doctor-check` /
  `reap-chop`). Catches leaks no matter how the process detached — double-fork, `nohup`,
  `setsid`, `disown`, or a runtime SASE does not control. This is the layer that would
  have caught this incident, and it is low-risk because the ownership join is exact.
- **Containment** (`session-containment`). Closes the hole at the source by making an
  agent run's descendants killable as a unit at teardown. Structurally stronger, but
  riskier and platform-specific, so it lands behind a default-off config flag and does
  not gate the detection work.

Per the Rust core backend boundary: _deciding_ whether a candidate process is leaked is
shared backend behavior — the TUI, `sase doctor`, and an axe chop all need the same
answer — so the classifier lives in `../sase-core`. Reading `/proc` is host I/O and
stays in Python as a thin adapter.

Reaping defaults to report-only. An automatic killer that is wrong even once is worse
than the leak, so the chop ships observe-only and is promoted to killing only after
`reap-chop`'s soak criteria are met.

## Phase details

### `core-planner`

Add an `agent_leak` module to `../sase-core/crates/sase_core/src/` alongside the
existing `agent_cleanup`, following its `wire.rs` / `planner.rs` / `mod.rs` split.

Wire types:

- `LeakCandidateWire` — `pid`, `agent_timestamp`, `ppid`, `start_time_ticks`, `cmdline`,
  `workspace_num`, `cpu_seconds`.
- `LeakScanRequestWire` — `schema_version`, `candidates`, `live_run_timestamps` (the set
  of agent runs currently executing), `min_age_seconds` (grace period), `now_unix`,
  `self_pid` and `protected_pids` (never classify the caller or its ancestors as
  leaked).
- `LeakScanResultWire` — `leaked: Vec<LeakedProcessWire>`, `live: Vec<...>`,
  `skipped: Vec<LeakSkippedItemWire>` with an explicit `reason` on every skip, mirroring
  `AgentCleanupSkippedItemWire`.

Classification rules, all of which must hold for `leaked`:

- The candidate carries a non-empty `agent_timestamp`.
- That timestamp is **not** in `live_run_timestamps`.
- The process is at least `min_age_seconds` old (default 300) — a run that just exited
  may still be tearing down legitimately.
- The pid is not in `protected_pids` and is not `self_pid`.

Anything failing a rule lands in `live` or `skipped` with a reason; nothing is ever
silently dropped. Classification must be a pure function of the request so it is
exhaustively unit-testable with no host access.

Expose the planner through the `sase_core_rs` binding. Add Rust unit tests for each
rule, each skip reason, and the boundary cases (empty candidates, empty live set,
candidate exactly at the age threshold, unparseable timestamp).

Done when Rust tests pass, the binding exports the planner, and a Python caller can
round-trip a request and result.

### `proc-adapter`

Add a Python host adapter — suggested `src/sase/agent/leaked_processes.py` — that:

- Walks `/proc/[0-9]*`, reads `environ`, and keeps processes whose environment contains
  `SASE_AGENT_TIMESTAMP`. Tolerate every per-pid failure (`ProcessLookupError`,
  `PermissionError`, races where the pid vanishes mid-read) by skipping that pid, never
  by aborting the scan.
- Reads `cmdline`, `stat` (ppid, start time, utime/stime), and derives age from
  `/proc/uptime` and the clock tick.
- Resolves `live_run_timestamps` from the existing agent registry / running field,
  reusing whatever `stale_running_cleanup` already uses rather than inventing a second
  source of truth.
- Computes `protected_pids` as the caller plus its full ancestor chain, so the scanning
  process can never target itself or the agent that invoked it.
- Calls the `core-planner` classifier and returns its typed result.
- Provides `reap(leaked, *, dry_run: bool)` that SIGTERMs, waits a short grace, then
  SIGKILLs survivors — and re-verifies each pid's `SASE_AGENT_TIMESTAMP` immediately
  before signalling, so a pid recycled between scan and kill is skipped rather than
  killed.

Non-Linux hosts must degrade to an empty inventory rather than raising.

Tests: a fake `/proc` tree fixture covering a leaked process, a live-run process, a pid
that vanishes mid-scan, an unreadable `environ`, a recycled pid at reap time, and a
non-Linux host.

Done when the adapter returns correct classifications against the fixture tree,
`dry_run=True` signals nothing, and the pid-recycle guard is proven by a test.

### `doctor-check`

Add a resource check — suggested `src/sase/doctor/checks_resources_leaked_processes.py`
— registered in `resource_check_specs` (`src/sase/doctor/checks_resources.py`) next to
the existing disk/ulimit/inotify checks.

The check reports, per owning agent run: leaked process count, the run's display name
(use `sase.project_display_names` conventions — never a ProjectSpec key), oldest process
age, and total accumulated CPU seconds. It passes clean when the inventory is empty,
warns on any leak, and fails when leaked processes exceed a configurable threshold or
their combined CPU time is material relative to `os.cpu_count()`.

The failure message must name the concrete remediation command from `reap-chop` rather
than leaving the user to work it out.

Follow `sase/memory/cli_rules.md` for any new CLI surface, and register the check so
`sase doctor --json` includes it.

Done when `sase doctor` reports leaked processes with a clean pass on a quiet host, and
tests cover the pass, warn, and fail tiers.

### `reap-chop`

Add a builtin chop — suggested `agent_process_reap`, following the
`sase_chop_stale_running_cleanup.py` pattern in `src/sase/scripts/` with its job on the
hook runner in `src/sase/axe/hook_jobs.py`.

Ship it **observe-only**: it scans, logs a structured summary
(`agent_process_reap: candidates=N leaked=M reaped=0 mode=observe`), and records a
metric alongside the existing `stale_running_cleaned` counter. It must not kill anything
on first landing.

Also add the operator-facing command the `doctor-check` message points at, so a human
can reap on demand and see exactly what would die first:

- `sase agent reap --dry-run` — list leaked processes, exit 0.
- `sase agent reap` — reap after showing the list.

Gate automatic killing behind a config flag defaulting to off. Promote it to default-on
only after the observe-only chop has run for a sustained period with zero false
positives — that is, every process it flagged was genuinely leaked. Record the soak
result in the bead so the promotion decision is evidence-based rather than assumed.

Per `sase/memory/cli_rules.md`, `sase agent reap` needs the standard option conventions
and JSON output support.

Done when the chop runs on schedule in observe mode, `sase agent reap --dry-run` lists
this incident's leak shape correctly, and no code path kills without the flag enabled.

### `session-containment`

Investigate and implement launch-side containment so an agent run's descendants are
killable as a unit, closing the leak at its source rather than cleaning up after it.
This phase is independent of the detection phases and can run in parallel with them.

Start with a written evaluation of the options against how agents are actually launched
today (into tmux — the leaked processes inherited the `tmux-spawn-*.scope` cgroup, not a
per-agent one):

- A per-run systemd user scope (`systemd-run --user --scope`), which contains
  double-forked children that a process group cannot.
- `start_new_session=True` plus `killpg` at teardown, which is simpler and already used
  elsewhere in the repo but is defeated by exactly the double-fork that caused this
  incident.
- A plain PID-namespace or cgroup v2 delegation approach.

Implement the option the evaluation selects, behind a config flag defaulting to off,
with teardown wired into the agent completion path. The flag matters: killing an agent's
whole session at teardown could reap something the user deliberately started and wants
to keep, so this needs opt-in soak time before it becomes the default.

If the evaluation concludes containment is not worth its risk on this host setup, say so
explicitly in the bead with the reasoning and close the phase without code. A documented
"no" is a valid outcome here — the detection layer already covers the failure mode.

Done when either containment works behind the flag with teardown proven by a test that
double-forks a child and asserts it dies, or the phase is closed with a written
justification.

## Out of scope

- Changing how agents write shell commands. Prompt-level guidance about verifying
  cleanup (`&&` over `;`, checking `pkill` exit status, trusting `uptime` over an echoed
  success message) may well be worth adding to SASE memory, but memory edits require
  explicit user permission in the moment and are not bundled into this plan.
- Host-wide CPU load alerting unrelated to agent-owned processes.
- Retroactive attribution of past leaks; the artifact transcripts already provide that
  when needed.

## Risks

- **Killing something legitimate.** Mitigated by the exact environment-stamp join, the
  age grace period, protected-pid handling, the pre-signal re-verification against pid
  recycling, and by shipping observe-only with killing behind a default-off flag.
- **`/proc` scan cost.** The scan reads `environ` for every pid on a busy host. Keep it
  off any hot path, run it on the chop's schedule rather than per-refresh, and consult
  `sase/memory/tui_perf.md` before exposing it anywhere the TUI touches.
- **Live-run set accuracy.** If `live_run_timestamps` is incomplete, a running agent's
  own children look leaked. This is the highest-consequence failure mode in the plan:
  reuse the existing registry source of truth rather than a parallel one, and let the
  age grace absorb races around run startup.
