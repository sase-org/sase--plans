---
tier: epic
title: sase monitor hardening — a supervisor that cannot silently orphan, wedge, or
  lie
goal: 'The monitor supervisor survives every command shape a real build throws at
  it — continuous output, partial lines, binary bytes, backgrounded grandchildren,
  TERM-ignoring children — enforces its deadline on every tick, never leaves a live
  process group behind when it dies, reconciles wedged lanes without a human, and
  only reports a monitor terminal once the claim, the log, and the follow-up have
  actually been settled.

  '
phases:
- id: wire
  title: Monitor supervision fields on the agent scan wire
  depends_on: []
  size: small
  description: 'wire: add the process-identity, settlement, idle-timeout, and follow-up-output
    marker fields to the Rust `AgentMetaWire` and its Python mirrors so the hardening
    phases have somewhere durable to record them.

    '
- id: stream
  title: Rebuild the supervisor's stream and wait loop
  depends_on: []
  size: medium
  description: 'stream: replace the blocking line-oriented `readline()` loop with
    a pipe-backed bounded writer plus a `child.poll()` tick loop, guard the whole
    supervisor lifecycle in `try`/`finally`, make every reader rotation-aware, and
    pass the monitor workflow to `release_workspace()`.

    '
- id: identity
  title: Durable process identity for the supervisor and its child
  depends_on:
  - wire
  - stream
  size: small
  description: 'identity: persist the monitored command''s pgid and a boot-id/start-ticks
    identity for the supervisor pid before the command can outlive its recorder, scrub
    agent identity from the command''s environment, and validate identity before signalling.

    '
- id: transaction
  title: Transactional monitor start and settlement
  depends_on:
  - identity
  size: medium
  description: 'transaction: take a per-lane lock inside `start_monitor()`, hold the
    command behind a go barrier until the workspace claim is secured, fingerprint
    the request for honest idempotency, and write the terminal marker only after settlement.

    '
- id: reconcile
  title: Active, complete reconciliation of dead supervisors
  depends_on:
  - transaction
  size: medium
  description: 'reconcile: make dead-supervisor reconciliation kill the surviving
    tree, release the claim, and dispose of the dropped follow-up; run it from `list`,
    the TUI, and the axe scheduler rather than only from `stop`; mark pre-reboot monitors
    `lost`.

    '
- id: idle
  title: --idle-timeout for commands that hang without exiting
  depends_on:
  - wire
  - stream
  size: small
  description: 'idle: add an opt-in idle timeout that fires when a monitored command
    has produced no output for N seconds, so `--timeout` can be generous without being
    useless.

    '
- id: followup
  title: Follow-up prompt trust boundary and inherited routing
  depends_on:
  - wire
  - stream
  size: medium
  description: 'followup: treat retained command output as untrusted data in the composed
    prompt, add `--next-output none|tail|file`, fence the command and cwd cells, and
    carry the starter''s model and reasoning effort to the follow-up agent.

    '
- id: fidelity
  title: Close the monitor fidelity gaps
  depends_on:
  - transaction
  size: small
  description: 'fidelity: print the start summary before the handoff kill, write `monitor_output_path`
    and `run_started_at`, and read the member''s own marker files instead of re-querying
    the artifact index in every polling loop.

    '
- id: docs
  title: Monitor documentation and skill hazards
  depends_on:
  - reconcile
  - idle
  - followup
  - fidelity
  size: small
  description: 'docs: document the new supervision guarantees, states, and flags in
    `docs/monitors.md`, and tighten the `/sase_monitor` skill with the hazard list
    and the flags it never mentioned.

    '
- id: exercises
  title: End-to-end hardening exercises
  depends_on:
  - docs
  size: xsmall
  description: 'exercises: run real monitors and agents against the regression matrix
    rows that automated tests cannot express, and report what the CLI, TUI, and follow-up
    agent actually did.'
proposed_by: bbugyi200.athena.sase-kp.land.w1
create_time: 2026-08-13 09:02:19
status: wip
bead_id: sase-ku
---

- **PROMPT:** [prompts/202608/monitor_hardening.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/monitor_hardening.md)
- **BEAD:** [sase-ku](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ku/README.md)

# Plan: `sase monitor` hardening

## Problem

The `sase-kp` epic shipped `sase monitor` and got the hard part right. A monitor is an
ordinary agent-family member — an artifacts directory with `agent_meta.json`,
`live_reply.md`, and `done.json` — so the artifact scanner, family roster, runtime
aggregation, chat history, workspace claims, `%wait`, and the `#fork` handoff all work
on monitors with almost no monitor-specific code. `store.py` is 289 lines and there is
no new store. That decision should stand.

The value leaks one layer down, in the **command supervisor**.
`supervise._stream_output()` (`src/sase/monitor/supervise.py:159-188`) drives output
streaming _and_ timeout enforcement from a single blocking, line-oriented,
strictly-decoding `readline()` loop:

```python
ready, _, _ = select.select([stdout], [], [], _POLL_SECONDS)   # supervise.py:171
if ready:
    line = stdout.readline()      # blocks on a partial line; decodes strictly
    if line:
        capture.append(line)
        continue                  # ← deadline and kill escalation never evaluated
    break                         # ← pipe EOF, not process exit
if child.poll() is not None:
    break
if deadline is not None and not timed_out and time.monotonic() >= deadline:
    ...                           # only reachable when select() times out
termination.maybe_escalate()      # ← same starved branch
...
for line in stdout:               # unbounded drain
    capture.append(line)
child.wait()                      # ← no timeout
```

That single choice produces a family of failures, every one of them reproduced against
real subprocesses by the research investigation
(`202608/monitor_command_substrate/monitor_command_substrate.md` in the `research`
sidecar) and re-verified in the shipped code at `HEAD`:

| Command shape                                          | Result                                                                                |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| continuous output (`while true; do echo chatty; done`) | **timeout never fires** — the `continue` starves the deadline branch                  |
| partial line (`printf` with no newline, then sleep)    | **blocks inside `readline()`** — `select()` guarantees a readable byte, not a newline |
| backgrounded grandchild holding stdout                 | **pipe EOF never arrives** — completion is EOF, not process exit                      |
| stdout and stderr both closed, process alive           | blocks in an unbounded `child.wait()`                                                 |
| **one invalid UTF-8 byte**                             | `UnicodeDecodeError` **escapes the supervisor at t≈0**                                |
| TERM-ignoring **and** chatty                           | SIGTERM sent, **SIGKILL escalation starved**                                          |
| quiet `sleep 8` (control)                              | correct — which is why every shipped test passes                                      |

Every supervisor test today uses a short, quiet, newline-terminated command
(`echo hello world`, `sh -c 'echo boom >&2; exit 3'`, `true`, `sleep 30` in
`tests/monitor/test_monitor_supervise.py`). The gaps survived review because the
fixtures cannot express them.

**The invalid-byte case is the one to fix first**, and it is the only failure that is
both instant and permanent:

```text
one invalid byte
  → UnicodeDecodeError escapes _stream_output()  (text=True ⇒ errors='strict')
  → run_supervisor() guards only the Popen call (supervise.py:105-131); the
    _stream_output() call at supervise.py:134 has no guard at all
  → supervisor dies; start.py:161-163 sends its stdout/stderr to DEVNULL, so the
    traceback is invisible
  → the child's process GROUP is still alive, and no pgid was ever persisted
    (start.py:174 records only the supervisor's own pid) → nothing in SASE can kill it
  → agent_meta.monitor_state stays "running" and no done.json is written
  → the lane is wedged, the workspace claim is held forever, and the --next
    follow-up never launches
```

Triggers are ordinary: a compiler emitting latin-1 diagnostics, a test printing binary,
a `grep` that hits a binary file, a tool running under a non-UTF-8 locale.

The timing raises the stakes. `sase-kp.11` landed a `sase/memory/build_and_run.md`
instruction telling every agent to run `just check-full` through `/sase_monitor` and
never inline — and a continuously-chatty command like `just check-full` is exactly the
shape whose timeout cannot fire. The documentation that makes the feature pay off also
maximizes the blast radius of the bug underneath it.

Around the supervisor sit four more confirmed defects:

- **Reconciliation does not reconcile.** `store._reconcile_dead_supervisor()`
  (`store.py:135-163`) marks the record `failed` but kills nothing, releases no claim,
  and silently drops the `--next` follow-up. It is only reachable through
  `sase monitor stop`, so a wedged lane needs a human. And `start_monitor()`'s
  same-command early return (`start.py:75-83`) has no liveness check, so a dead monitor
  is reported to its caller as live — the agent is then killed for nothing.
- **The command can run without the claim.** `start.py` spawns the supervisor at line
  151 and only transfers or acquires the claim at lines 177-195. The supervisor reads
  `monitor_state="running"` and may `exec` immediately. On claim failure the best-effort
  SIGTERM at line 197 may itself be starved by the streaming bug.
- **Terminal state precedes settlement.** `_finish_monitor()` writes `done.json` before
  launching the follow-up or releasing the claim, so observers see `completed` while
  settlement is still running. Phase `kp.6` already recorded the resulting flaky test.
- **`kp.9` promoted a fidelity gap into a routine one.** Every approved epic launch now
  runs `start_monitor(..., inherit_lane_workspace_claim=False)`
  (`src/sase/bead/epic_launch.py:164`), which claims **shared workspace 0**.
  `_release_claim_and_notify()` calls `release_workspace(..., cl_name=cl_name)` with no
  `workflow` argument (`supervise.py:301-305`); `workflow` is an optional matcher
  (`running_field/_operations.py:160-164`), so the release matches on
  `(workspace_num, cl_name)` alone and can remove a claim the monitor does not own — on
  the slot shared with `%wait` agents.

Finally, the monitored command **inherits the dead starter's agent identity**:
`start.py:166` passes `env=os.environ.copy()` and `supervise.py:106` passes no `env=` at
all, so the command runs with `SASE_AGENT=1`, `SASE_AGENT_NAME`, and
`SASE_ARTIFACTS_DIR` pointing at an agent that no longer exists. Every comparable
boundary in this repo scrubs these (`scrub_agent_identity_env()` in
`src/sase/agent/env_hygiene.py`). Inside a monitored command, `sase artifact create` and
`sase var` would pass their `SASE_AGENT == "1"` guards and write into the dead starter's
directory, and a nested `sase monitor start` would call `kill_agent_runner_group()` on a
stale pid. With `kp.9` landed, `sase bead work` runs on this path routinely.

## Solution overview

Stop hand-rolling a supervisor. SASE already has a working one: `sase.tasks`.

`tasks.logs.open_task_log()` returns `_BoundedTaskLogPipe`
(`src/sase/tasks/logs.py:103-168`), whose `fileno()` is the **write end of an
`os.pipe()`**, drained by a daemon thread that reads 64 KiB chunks and appends through
`append_bytes_locked(..., truncate_oversized=True)`
(`src/sase/logs/_bounded.py:94-142`). `tasks/supervisor.py:42-55` then waits on
`child.poll()` at a 50 ms tick and escalates TERM→KILL from that same loop. Handing the
child a real file descriptor and polling for process exit is what makes partial lines,
decoding, and grandchild-held pipes stop mattering:

| Failure                               | Why the pipe-and-poll shape fixes it                                                |
| ------------------------------------- | ----------------------------------------------------------------------------------- |
| timeout starved by chatty output      | the deadline lives in the poll loop, not behind a `continue`                        |
| blocked on a partial line             | the supervisor never calls `readline()`; the drain thread takes `os.read()` chunks  |
| invalid UTF-8 byte                    | the data path is `bytes`; decoding happens at the boundary, with `errors="replace"` |
| grandchild holds stdout               | completion is `child.poll()`, not pipe EOF                                          |
| stdout and stderr closed              | the poll loop does not depend on the pipe at all                                    |
| SIGKILL escalation starved            | escalation is evaluated on the same 50 ms tick                                      |
| 19.7 MB of unbuffered per-line writes | the drain thread batches into 64 KiB appends                                        |

Two things are **not** adopted from `tasks`:

1. **Its retention.** `append_bytes_locked` keeps only `path` plus one rotated `path.1`,
   so a long run loses the head. `OutputCapture`'s head + tail elision
   (`src/sase/monitor/output.py:38-79`) is better for a follow-up LLM — you want the
   start of the build _and_ the failure at the end. Keep that accumulator and feed it
   `bytes` chunks from the drain thread.
2. **Its store.** Monitors keep using agent artifacts. Nothing here adds a monitor
   database.

Around that corrected substrate, this epic restores three invariants the current code
does not hold:

- **Nothing outlives its recorder.** The child's pgid and a boot-id/start-ticks identity
  for the supervisor are persisted before the command can escape, so `stop`, escalation,
  and reconciliation can always find the tree and can never signal a recycled pid.
- **A monitor is terminal only once it is settled.** The child has exited, the log is
  finalized, the claim is disposed, and the follow-up is launched or explicitly recorded
  as failed — _then_ `done.json` is written.
- **A wedged lane heals itself.** Reconciliation kills the surviving tree, releases the
  claim, and surfaces the dropped follow-up, and it runs from `list`, the TUI, and the
  axe scheduler rather than waiting for a human to type `sase monitor stop`.

### Vocabulary

| Term           | Meaning                                                                    |
| -------------- | -------------------------------------------------------------------------- |
| starter        | The agent that ran `sase monitor start` (absent for host-started monitors) |
| monitor member | The `--mon` family member representing the supervised command              |
| supervisor     | The detached `sase monitor _supervise` process that runs the command       |
| follow-up      | The agent launched into the lane after the command finishes                |
| settlement     | Log finalized, claim disposed, follow-up launched or recorded as failed    |
| drain thread   | The daemon thread that reads the child's output pipe in chunks             |

## Constraints every phase must respect

- **Do not touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims**
  (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`). `sase/memory/build_and_run.md`
  describes monitors and looks tempting; editing it requires the user's explicit
  permission, which this plan does not grant. If a memory update looks warranted, record
  a `PROPOSED FOLLOW-UP:` note on your phase bead instead.
- **`docs/monitors.md` and `src/sase/xprompts/skills/sase_monitor.md` are not memory
  files** and are in scope for the `docs` phase.
- **Cross-repo work goes through `/sase_repo`.** The `wire` phase must
  `sase repo open sase-core` before reading or writing anything under it.
- **Keep every module under the `_lint-toobig` cap.** `supervise.py` is already 371
  lines and the `stream` phase adds to it; split early rather than late.
- **Run `just check` before you finish.** Hand `just check-full` to `/sase_monitor` —
  and if the monitor you are testing is broken, say so plainly rather than working
  around it.

## Non-goals

Deliberately excluded, with the reasoning kept here so a later agent does not
re-litigate them:

- **No secrets redaction pipeline.** The exact shell command is persisted in
  `agent_meta.json`, saved to chat history, and echoed in notifications. A shared
  redaction pipeline over task and monitor logs, plus an argv form
  (`sase monitor start -- cmd arg...`) alongside an explicitly-named shell mode, is the
  right long-term shape — and it is its own epic, with its own chunk-boundary test
  matrix. Not bundled with a supervision fix.
- **No monitor telemetry or lifecycle journal.** Counters are worth adding _after_ the
  supervision and settlement invariants are correct, not while they are being changed.
- **No shared supervisor kernel extraction between `tasks` and `monitor`.** This epic
  extracts exactly one shared primitive (the pipe-backed bounded writer). Consolidating
  the rest is consolidation of proven code and belongs after these phases settle the
  correct shape.
- **No admission control, no bounded retries, no `--next-on`.** Useful, but they are
  policy on top of a substrate that must first be correct. Replacement/queueing policy
  in particular must never be silently defaulted, because monitored commands mutate
  workspaces and deployment targets.
- **No systemd/cgroup backend.** Transient user services with `KillMode=control-group`
  are well-evidenced on this host, but SASE also supports macOS. This is a P2 hardening
  backend _behind_ a corrected portable supervisor, not a replacement for one.
- **No cron or recurring monitors, no automatic restart.** AXE already owns recurring
  execution, and most shell commands are not proven idempotent.
- **No standalone/host lanes.** `resolve_lane()` still requires existing agent
  artifacts, so `sase monitor start --lane whatever` from a plain shell still fails.
  Worth fixing; it is a new capability rather than a correctness fix, and it would touch
  the same `start.py` paths four phases here already rewrite.
- **No move of the monitor lifecycle into Rust.** Relocating only the pure projection
  layer (`MonitorRecord.from_record()`, `monitor_state_bucket()`, the `done.json`
  precedence rules) is a legitimate reading of the core-backend boundary rule and worth
  a deliberate decision later. It is not urgent and must not be bundled with this work.

---

## Phase `wire`: Monitor supervision fields on the agent scan wire

The agent artifact scanner is Rust and its wires are **explicit field lists** — unknown
keys in `agent_meta.json` are dropped, not round-tripped. Every field the later phases
need must be declared on both sides first, exactly as `sase-kp.1` did for the original
monitor fields.

```bash
sase repo open sase-core -r "Add monitor supervision fields to the agent scan wire"
```

Work items:

1. In `crates/sase_core/src/agent_scan/wire.rs`, add to `AgentMetaWire` (all
   `#[serde(default)]`, alongside the existing `monitor_*` block at lines 339-369):

   | Field                          | Type             | Meaning                                                    |
   | ------------------------------ | ---------------- | ---------------------------------------------------------- |
   | `monitor_pgid`                 | `Option<i64>`    | Process-group id of the monitored command                  |
   | `monitor_supervisor_identity`  | `Option<String>` | `"<boot_id>:<start_ticks>"` for the supervisor pid         |
   | `monitor_settled`              | `bool`           | Whether log, claim, and follow-up disposition are complete |
   | `monitor_idle_timeout_seconds` | `Option<f64>`    | Idle budget; absent means no idle timeout                  |
   | `monitor_next_output`          | `Option<String>` | `none` \| `tail` \| `file`                                 |
   | `monitor_request_fingerprint`  | `Option<String>` | Idempotency key for the start request                      |

2. Bump `AGENT_SCAN_WIRE_SCHEMA_VERSION` (`wire.rs:24`, currently `5`) and mirror the
   bump in `src/sase/core/agent_scan_wire_records.py`.
3. Mirror every field in `src/sase/core/agent_scan_wire_markers.py` (`AgentMetaWire`,
   after `monitor_tail_lines` at line 210), and in `agent_scan_wire_conversion.py` if
   the conversion helpers enumerate fields.
4. Rust tests: extend `agent_meta_wire_round_trips_every_monitor_field` (`wire.rs:674`)
   with the new fields, and keep `agent_meta_wire_without_monitor_fields_still_parses`
   (`wire.rs:702`) green. Python tests: round-trip parity in `tests/core/`.

Land the core change as its own commit in `sase-core`, then the Python mirror. The
Python side must tolerate a core binary that predates the fields — they simply arrive
absent, which `known_field_kwargs` already handles.

This phase ships no behavior. Do not add reader or writer logic for the new fields; the
phases that own them do that.

## Phase `stream`: Rebuild the supervisor's stream and wait loop

The core fix. This phase _removes_ net code.

### Extract the shared pipe-backed bounded writer

`_BoundedTaskLogPipe` (`src/sase/tasks/logs.py:103-168`) is exactly the right primitive
and is private to `sase.tasks`. Promote it to `src/sase/logs/pipe.py` as
`BoundedLogPipe`, with one addition: an optional `on_chunk: Callable[[bytes], None]`
invoked by the drain thread for every chunk it appends.

- `sase.tasks.logs.open_task_log()` becomes a thin call into it with no `on_chunk`, and
  its observable behavior must not change — the existing task-log tests are the
  regression guard.
- An exception raised by `on_chunk` must not kill the drain thread or lose the on-disk
  append; record it the way `_write_error` already records an `OSError` and re-raise on
  `close()`.
- Keep the 64 KiB read chunk and the `log_file_lock` +
  `append_bytes_locked(..., truncate_oversized=True)` append path unchanged.

### Rework `OutputCapture` to consume bytes, not own the file

`src/sase/monitor/output.py` currently opens `live_reply.md` with `buffering=0` and
writes once per line — a chatty command burns a core on ~2.8M unbuffered write syscalls.
Change it to a pure in-memory accumulator:

- `append_bytes(chunk: bytes) -> None` replaces `append(text: str)`; the head + tail
  elision logic and `retained_text()` / `truncated` / `total_bytes` semantics are
  unchanged and their existing tests must stay green.
- It no longer opens or closes a file. Delete `close()` and the `_file` attribute.
- The supervisor wires it up as `BoundedLogPipe(..., on_chunk=capture.append_bytes)`.

### Rewrite the supervisor lifecycle

In `src/sase/monitor/supervise.py`:

1. Spawn the child with `stdout=<the BoundedLogPipe>`, `stderr=subprocess.STDOUT`,
   `stdin=DEVNULL`, `start_new_session=True`, `close_fds=True`, and **no** `text=True` —
   the data path is bytes now.
2. Replace `_stream_output()` with a wait loop modeled on `tasks/supervisor.py:42-55`,
   ticking at `_POLL_SECONDS = 0.05` (down from `0.5`):
   - completion is `child.poll() is not None` — **never** pipe EOF;
   - the total-runtime deadline is evaluated on **every** tick;
   - TERM→KILL escalation is evaluated on **every** tick;
   - there is no `readline()`, no `select()`, and no unbounded `for line in stdout`.
3. Close the pipe (which joins the drain thread and flushes the final chunks) **after**
   the child exits, then take `capture.retained_text()`. The drain thread must be joined
   before the retained view is read, or the follow-up prompt races the tail.
4. Wrap the **whole** lifecycle in `try` / `except BaseException` / `finally`, the way
   `tasks/supervisor.py:137-152` does. On any internal failure: TERM→KILL the child
   tree, finalize the log, and still write a terminal marker recording the supervisor
   error. A supervisor must never exit without either a live child _or_ a terminal
   marker.
5. Give the supervisor somewhere to say what went wrong. `start.py:161-163` sends its
   stdout and stderr to `DEVNULL`, so today a traceback is invisible; write the
   `finally` block's error into the monitor's own `done.json` (`error`) and append a
   single explanatory line to the retained output.

### Bound the log on disk, and make every reader rotation-aware

The bounded writer rotates `live_reply.md` to `live_reply.md.1` at the byte cap, which
`OutputCapture` never did. That bound is wanted — the probe wrote 19.7 MB in 8 seconds —
but every reader must cope:

- Add a monitor log cap constant and an `SASE_MONITOR_LOG_MAX_BYTES` env override,
  mirroring `ENV_MAX_BYTES` in `tasks/logs.py`.
- `_handle_monitor_show`'s `_output_text()` (`src/sase/main/monitor_handler.py:382`)
  must stitch `live_reply.md.1` then `live_reply.md`, the way
  `tasks.logs.read_task_log_tail()` does.
- `_read_new_text()` (`monitor_handler.py:388`) must detect a shrink —
  `st_size < offset` means rotation — and restart from offset 0 instead of returning
  nothing forever.
- The TUI's `TailCache` (`src/sase/agent/artifact_files_cache.py:41-75`) already resets
  on shrink, so it is correct as-is. Add a test that pins that, because the monitor now
  depends on it.

### Also in this phase (same blast radius)

Pass the monitor's own workflow to the release so it cannot remove a claim it does not
own. `_release_claim_and_notify()` (`supervise.py:301-305`) must call:

```python
release_workspace(
    get_project_file_path(project_name),
    int(workspace_num),
    MONITOR_WORKSPACE_CLAIM_WORKFLOW,
    cl_name=cl_name,
)
```

This is one line, and it is urgent rather than cosmetic: since `kp.9`, every approved
epic launch claims **shared workspace 0** (`epic_launch.py:164` →
`inherit_lane_workspace_claim=False` → `claim_workspace(project_file, 0, ...)`), the
same slot `%wait` agents use.

### Tests

Every row is a currently-failing boundary. Drive the real `run_supervisor()` against
real subprocesses; a fixture that cannot express the failure is not a test of it.

| Exercise                                                                 | Required invariant                                       |
| ------------------------------------------------------------------------ | -------------------------------------------------------- |
| continuous output (`while true; do echo chatty; done`) past the deadline | timeout wins; state is `timeout`; the tree is dead       |
| `printf` with no trailing newline, then a long sleep                     | output is visible on disk; the deadline still fires      |
| a backgrounded grandchild inherits and holds stdout                      | completion is process exit, not pipe EOF                 |
| child closes both stdout and stderr and keeps running                    | the supervisor stays responsive and the deadline fires   |
| child emits `\xff` then sleeps                                           | the supervisor survives; the byte is replaced, not fatal |
| child traps TERM and prints continuously                                 | SIGKILL lands after the grace period                     |
| output beyond the cap while `show --follow` is running                   | disk stays bounded; the follower does not stall          |
| the existing quiet-command tests                                         | byte-identical behavior                                  |

## Phase `identity`: Durable process identity for the supervisor and its child

Nothing may outlive its recorder.

1. **Persist the child's pgid.** `sase.tasks` records `pgid=child.pid`
   (`tasks/supervisor.py:119`, `tasks/models.py:45`) precisely so a dead supervisor
   cannot strand a live tree. The monitor supervisor must write `monitor_pgid` to the
   member's `agent_meta.json` immediately after `Popen` returns and **before** it enters
   the wait loop.
2. **Close the pid-recording window.** `start.py:174` writes the supervisor's `pid`
   _after_ `Popen`, leaving an interval in which a crash leaves no pid at all. With the
   `transaction` phase's go barrier the supervisor cannot have started the command yet,
   but the pid must still be written before the barrier is lifted; order these two
   explicitly and say so in a comment.
3. **Record a stronger identity than a pid.** `store.stop_monitor()` checks only that
   the numeric pid is alive (`store.py:112`), so it can signal a recycled pid. Reuse the
   pattern in `src/sase/axe/maintenance.py:140-182`: `/proc/<pid>/stat` field 19
   (`start_ticks`) plus `/proc/sys/kernel/random/boot_id`. Write it as
   `monitor_supervisor_identity` and add a `supervisor_is_alive(record)` helper that
   returns `False` when the pid is alive but its identity does not match. On platforms
   without `/proc`, record an empty identity and fall back to the current liveness check
   — this must not break macOS.
4. **Use it everywhere a signal is sent.** `stop_monitor()` and the escalation paths
   must consult `supervisor_is_alive()` / the recorded pgid rather than a bare pid.
5. **Scrub agent identity from the monitored command.** Apply
   `scrub_agent_identity_env()` (`src/sase/agent/env_hygiene.py:6`) to the child
   environment in `supervise.py`'s `Popen` — which today passes no `env=` at all and so
   inherits `SASE_AGENT`, `SASE_AGENT_NAME`, and `SASE_ARTIFACTS_DIR` from the dead
   starter — and drop `SASE_ARTIFACTS_DIR` explicitly, since it does not carry the
   `SASE_AGENT_` prefix the scrubber matches on. Mirror the same scrub on the supervisor
   spawn in `start.py:166`.

Tests: a monitored command sees no `SASE_AGENT*` and no `SASE_ARTIFACTS_DIR`;
`monitor_pgid` is present in `agent_meta.json` while the command runs; a record whose
pid has been recycled (simulate by rewriting the recorded identity) is treated as dead
and never signalled.

## Phase `transaction`: Transactional monitor start and settlement

`start.py:75` → `130` is a check-then-create with no lock. The one caller that gets this
right today does it externally — `epic_launch.py:146` wraps the whole resolve-and-start
in `log_file_lock(tasks_dir() / "epic-launch-submit")`. The generic primitive must
enforce its own invariant instead of depending on every future caller's discipline.

1. **Per-lane lock.** Take `log_file_lock()` on a per-(project, lane) path around the
   whole duplicate check → lane resolve → member create → spawn → claim sequence in
   `start_monitor()`. Two simultaneous starts in one lane must produce exactly one
   running command; the loser gets `MonitorAlreadyRunningError` naming the winner.
2. **Launch barrier.** The supervisor must not `exec` the command before the workspace
   claim is held. Create the member, spawn the supervisor, record its pid, transfer or
   acquire the claim, and only then write a `.monitor_go` file into the member's
   artifacts directory. `run_supervisor()` polls for that file with a bounded budget
   (~30s) before spawning the child; if it never appears it writes a terminal `failed`
   marker with a clear error and runs nothing. On claim failure `start_monitor()` tears
   the member down as it does today — but now with the certainty that no mutating
   command ever ran.
3. **Honest idempotency.** The same-command early return (`start.py:78-79`) currently
   treats command-string equality as identity, so a request with a different `cwd`,
   timeout, `--next`, statuses, or reason silently returns the old monitor. Compute a
   `monitor_request_fingerprint` over the whole request; return the existing record only
   on a fingerprint match, and raise `MonitorAlreadyRunningError` describing the
   difference otherwise. The epic-launch dedupe (which passes an identical request each
   time) must keep working — cover it.
4. **Settle before going terminal.** Reorder `_finish_monitor()` so `done.json` is
   written **last**:
   - child exits → close the pipe and join the drain thread → write the retained output
     and `stopped_at`
   - dispose the claim: launch the follow-up (which transfers the claim to it) or
     release the claim
   - set `monitor_settled = true` on `agent_meta.json`
   - write `done.json` and notify Then make the projection honest: `MonitorRecord` gains
     a `settled` field, and `is_terminal` requires _both_ a terminal `monitor_state` and
     `settled`. A monitor whose child has exited but whose follow-up has not launched is
     not yet terminal, so `active_monitor_for_lane()`, `--follow`, and `%wait` all keep
     waiting. This is the invariant behind the flaky test `kp.6` recorded.

Tests: two concurrent `start_monitor()` calls in one lane; a forced claim-transfer
failure leaves no executed command (assert on a sentinel file the command would have
created); a delayed or failing follow-up launch produces no premature terminal state and
no leaked claim; a changed request against a running same-command monitor raises rather
than silently returning.

## Phase `reconcile`: Active, complete reconciliation of dead supervisors

Today `_reconcile_dead_supervisor()` (`store.py:135-163`) marks the record `failed` and
stops. For the headline use case — `just check-full --next "fix what failed"` — a dead
supervisor therefore means the tree keeps running, the claim is held, the follow-up
never launches, and the lane just goes quiet. And nothing calls it except `stop`.

1. **Make it complete.** Reconciliation must, in order: kill the surviving tree via the
   recorded `monitor_pgid` (TERM, then KILL after the grace period); finalize the log;
   release the workspace claim with `MONITOR_WORKSPACE_CLAIM_WORKFLOW`; and dispose of
   the follow-up. Disposition means: if a `--next` action was recorded and never
   launched, either launch it with an outcome line that says the supervisor died, or
   record `monitor_followup_error` and notify — pick one, implement it, and state which
   in `docs/monitors.md`. Silently dropping it is the current behavior and is what makes
   the failure invisible.
2. **Make it reachable.** Run it from `sase monitor list`, from the TUI's monitor
   refresh path, and from the axe scheduler beside the existing stale-claim sweep. It
   must be cheap when there is nothing to reconcile (only monitors whose
   `monitor_state == "running"` and whose supervisor fails `supervisor_is_alive()`), and
   it must be safe to run concurrently from all three — take the same per-lane lock the
   `transaction` phase introduced.
3. **Fix the liveness lie at start.** `start_monitor()`'s early return must check
   `supervisor_is_alive()` before returning an existing record. A dead monitor must be
   reconciled and then replaced, not reported as live to a caller that is about to kill
   itself.
4. **Add the `lost` state.** A running monitor whose `monitor_supervisor_identity`
   carries a different boot id than the current boot cannot be reasoned about: its pid
   space is gone. Add `"lost"` to `MONITOR_STATES`, `TERMINAL_MONITOR_STATES`,
   `MONITOR_STATE_CHOICES` (`src/sase/main/parser_monitor.py:9`), the state glyph table
   (`src/sase/ace/tui/widgets/prompt_panel/_agent_monitor_section.py:23-29`), and
   `monitor_state_bucket()` in `src/sase/monitor_state.py` with bucket `Failed`. A
   `lost` monitor is **never** implicitly re-run, and its `--next` follow-up is not
   launched — the command's effect on the workspace is unknown.

Tests: SIGKILL a real supervisor with a live grandchild, then reconcile — the tree is
dead, the claim is released, the follow-up is dispositioned, and a terminal marker
exists; an `agent_meta.json` carrying a previous boot's identity reconciles to `lost`
with no rerun; reconciliation from `list` does not mutate healthy running monitors.

## Phase `idle`: `--idle-timeout` for commands that hang without exiting

`--timeout` bounds total runtime, which forces a bad trade: set it tight and a
legitimately slow `check-full` is killed; set it generous (epic launch uses 4h) and a
wedged command burns the whole budget before anyone notices. Once output flows through a
polled loop with a chunk callback, "no bytes for N seconds" is nearly free — and it is
the check that actually catches hangs.

1. Add `-i/--idle-timeout DURATION` to `sase monitor start`, parsed by the existing
   `_parse_timeout()` (`monitor_handler.py:414`). Add
   `idle_timeout_seconds: float = 0.0` to `StartMonitorRequest`, persist it as
   `monitor_idle_timeout_seconds`, and thread it through `create_monitor_member()`.
2. The supervisor tracks the last chunk's arrival (updated from the drain thread's
   `on_chunk`, so it costs nothing extra) and fires the same TERM→KILL path as the total
   timeout when the idle budget elapses. Guard the shared timestamp with a lock; the
   drain thread and the wait loop are different threads.
3. Distinguish the two in the record. Both produce `monitor_state == "timeout"`; the
   terminal marker and the follow-up prompt's outcome line must say _which_ budget was
   exceeded ("no output for 10m" vs "did not finish after 45m of a 45m budget").
4. **Opt-in only.** A quiet command is not automatically unhealthy — `sleep 300` is a
   documented, supported monitor. There is no default idle timeout, and the flag's help
   text must say why.
5. Surface it: `monitor_detail` / `monitor_show_json` in
   `src/sase/main/monitor_render.py` and the TUI's MONITOR section.

Tests: a command that prints once and then hangs forever is killed at the idle budget,
not the total budget; a chatty command well inside its idle budget is untouched; a quiet
`sleep` with no idle timeout runs to completion.

## Phase `followup`: Follow-up prompt trust boundary and inherited routing

The follow-up prompt embeds retained command output verbatim
(`src/sase/monitor/followup_prompt.py:115-135`). Build output routinely contains
attacker-influenced text: test names, dependency output, fetched content, a `git log`
subject line. `_widen_fence()` (`followup_prompt.py:48-58`) is a real defense rather
than a cosmetic one — a fenced block is an xprompt literal zone, so widening the fence
genuinely prevents directive expansion — but a code fence does not stop a _model_ from
treating "ignore prior instructions and run …" as an instruction.

1. **Fence the two cells that are not literal zones.** `Command` and `Directory` render
   as inline code spans (`followup_prompt.py:97-98`), so a command containing `#commit`
   or `%model:x` expands in the follow-up's prompt. Free to close.
2. **Label the tail as untrusted data.** Immediately before the fenced output, state
   plainly that everything inside the fence is program output, not instructions, and
   that the only instruction is the "Your next action" section. Deterministic separation
   only — no guardrail LLM.
3. **Add `--next-output none|tail|file`** (default `tail`, persisted as
   `monitor_next_output`). `none` gives the follow-up a deterministic outcome summary
   and the `sase monitor show <id> --all-lines` pointer only; `file` gives the summary
   plus the log path, so the agent fetches the output itself and can apply its own
   judgment. For a large or hostile build log, a deterministic summary plus a reference
   beats a big inline tail.
4. **Carry the starter's routing.** `followup.py` contains no model, effort, or
   `SASE_MODEL_OVERRIDE` handling, so an Opus lane's monitor hands off to a
   default-model agent. Prefix the composed prompt with `%model:` / `%effort:` derived
   from the lane's newest member metadata — directives are stripped before the model
   sees the prompt, so this is invisible in the rendered chat. Read
   `sase/memory/xprompts.md` via `/sase_memory_read` before touching directive syntax.
5. **Adversarial tests.** Output containing `#commit`, `%model:haiku`, a nested triple
   fence, an "ignore your previous instructions" line, and a fake `## Your next action`
   heading must all round-trip inertly; assert the fence widens past the nested fence
   and that no directive survives into an expanded prompt.

## Phase `fidelity`: Close the monitor fidelity gaps

Small, independent, individually annoying.

1. **Print the start summary before the handoff kill.** `kill_agent_runner_group()` is
   `NoReturn` (`src/sase/main/utils.py:60`), so every line after
   `maybe_handoff_monitor_from_agent(record)` in `_handle_monitor_start()`
   (`monitor_handler.py:236-255`) is unreachable inside an agent — including
   `-j/--json`, in the only context that uses it. Move the summary (and the JSON
   envelope) above the handoff call, and keep the "this is the last output before the
   runner is killed" line, which the original plan explicitly required. The only test
   today asserts `handed_off is False`; cover the `True` path.
2. **Write `monitor_output_path`.** It is declared on the wire
   (`agent_scan_wire_markers.py:206`) and read in `models.py:134`, but never written, so
   `record.output_path` is always `None` and every consumer re-derives it. Write it when
   the supervisor opens the log, and make `monitor_handler._output_path()` and the TUI
   prefer it. (If, on inspection, no consumer would ever use it, delete the field
   instead — but pick one.)
3. **Write `run_started_at`.** `create_monitor_member()` does not set it
   (`src/sase/monitor/member.py:57-74`), so displayed runtime includes member creation
   and supervisor spawn latency. `compute_row_runtime()` already prefers
   `run_start_time` (`src/sase/ace/tui/models/agent_time.py:450`) — the supervisor just
   has to record `run_started_at` at the moment it actually spawns the child.
4. **Stop re-querying the artifact index in tight loops.** `--follow` (0.5s),
   `stop_monitor()`'s wait loop, and `_wait_for_monitor()` each call `get_monitor()`,
   which runs a full-history, unlimited, hidden-inclusive index query
   (`store.py:248-275`). They already know the member's `artifacts_dir`; add a
   `read_monitor_marker(artifacts_dir)` that reads that member's `agent_meta.json` and
   `done.json` directly and returns the same `MonitorRecord`, and use it in all three
   loops. `resolve_lane()`'s one-shot query at start is fine and stays.

## Phase `docs`: Monitor documentation and skill hazards

`docs/monitors.md` (162 lines) and `src/sase/xprompts/skills/sase_monitor.md` (80 lines)
both predate everything in this epic.

1. Update `docs/monitors.md` with: the supervision guarantees (process-exit completion,
   deadline on every tick, bounded rotating log, TERM→KILL escalation), the settlement
   invariant and what "terminal" now means, the `lost` state and the no-implicit-rerun
   rule, reconciliation and where it runs from, `--idle-timeout`, `--next-output`, and
   the follow-up's untrusted-output boundary.
2. Tighten `/sase_monitor`. Read `sase/memory/generated_skills.md` with
   `/sase_memory_read` first — the skill is generated from its source template into
   managed locations, and editing the deployed copy is wrong. Add the hazard list the
   skill never states: the command runs under `sh -c` (quoting is the caller's problem);
   one active monitor per lane, and starting a second is an error; no interactive or
   TTY-requiring commands; output is bounded and rotates. Document the flags it never
   mentions — `--cwd`, `--lane`, `--label`, `--tail-lines` — plus the new
   `--idle-timeout` and `--next-output`.
3. Do **not** edit `sase/memory/build_and_run.md` or any other memory file. If the
   `just check-full` guidance there needs to change in light of this work, record a
   `PROPOSED FOLLOW-UP:` note on your phase bead.

## Phase `exercises`: End-to-end hardening exercises

Automated tests cover the deterministic rows. This phase covers what only a real run can
show. Launch real monitors (and real follow-up agents) and report what the CLI, the TUI,
and the follow-up agent actually displayed — not what the code should have done.

1. `just check-full` through `/sase_monitor` with a `--next` action, from inside a real
   agent: the summary prints before the kill, the monitor row shows live runtime, the
   follow-up launches into the same lane and workspace with the starter's model, and its
   prompt shows the fenced untrusted-output notice.
2. A chatty command past its deadline: it is killed, the row shows `timeout`, the tree
   is gone (`pgrep` the command), and the log on disk is bounded.
3. `sase monitor show --follow` across a log rotation: the follower keeps printing.
4. A monitor started with `--idle-timeout` against a command that stalls after first
   output: killed on the idle budget, with the outcome line naming that budget.
5. `kill -9` a live supervisor, then `sase monitor list`: the lane reconciles itself
   without a human, the claim is released, and the follow-up disposition is visible.
6. An approved epic launch end to end, since `kp.9` routes it through this path: the
   `EPIC APPROVED` → `EPIC CREATED` transition, on shared workspace 0, releasing only
   its own claim.

Report each exercise's observed output. A row that could not be exercised is a finding,
not a gap to paper over.
