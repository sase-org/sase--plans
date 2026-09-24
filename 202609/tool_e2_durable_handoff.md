---
tier: epic
title: 'E2: durable ToolRun hand-off and lifecycle control'
goal: 'A handed-off ToolRun has a durable identity before its caller lets go, stays
  discoverable, followable, waitable, and stoppable through that identity, and always
  settles to either an authoritative outcome or an explicit, typed uncertainty — built
  on the existing monitor and proc executors, with no new supervisor.

  '
phases:
- id: core-contract
  title: Extend the sase-core ToolRun contract for reservation, adoption, and owner-aware
    settlement
  depends_on: []
  size: large
  description: 'core-contract: add reservation with a private launch envelope, an
    atomic claim, a durable stop request, typed terminal causes, persisted finish
    diagnostics, the owner log locator, and owner-aware reconcile to the sase-core
    ToolRun store and bindings, all additive at wire schema 1, then move sase''s core
    revision pin.'
- id: standalone-handoff
  title: Hand a ToolRun off to a plain durable proc with sase tool run -H
  depends_on:
  - core-contract
  size: large
  description: 'standalone-handoff: split the executor so one body serves foreground
    and adopted runs, add the hidden claim-then-run worker and the shared launch module,
    and ship fail-closed sase tool run -H outside agents over a plain proc, behind
    the tool_handoff beta flag.'
- id: monitor-handoff
  title: Reserve the ToolRun when a monitor start hands off a tool run
  depends_on:
  - standalone-handoff
  size: medium
  description: 'monitor-handoff: when a monitor''s proc will run a ToolRun, reserve
    and bind it to the monitor before hand-off and run the adopting worker, leaving
    monitor_command, execution_argv, and -f bindings untouched, fail-open to E1.5
    wrapping, behind the same flag.'
- id: lifecycle-controls
  title: Stop, follow, and wait on a ToolRun by id
  depends_on:
  - standalone-handoff
  size: medium
  description: 'lifecycle-controls: add sase tool stop, sase tool show -F/--follow,
    and sase tool wait as ownership-aware facades over the run''s execution owner,
    with stop-requested versus stopped reporting and viewer-only interruption.'
- id: settlement
  title: Settle hand-off runs truthfully after crashes and deliver once
  depends_on:
  - monitor-handoff
  - lifecycle-controls
  size: large
  description: 'settlement: feed owner facts into reconcile, record stop and timeout
    causes from the owner''s termination intent, settle from proc and monitor settlement,
    publish exactly one notification for proc-owned hand-offs, and report expired
    owner logs explicitly.'
- id: acceptance-and-adoption
  title: Prove the hand-off contract end to end and remove the beta flag
  depends_on:
  - settlement
  size: medium
  description: 'acceptance-and-adoption: extend the ToolRun smoke harness with the
    hand-off fault matrix, update docs, the sase_monitor skill source, and the named
    memory, and remove the tool_handoff flag.'
proposed_by: bbugyi200.athena.0qj
create_time: 2026-09-24 08:40:17
status: wip
bead_id: sase-17p
---

- **PROMPT:** [prompts/202609/tool_e2_durable_handoff.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_e2_durable_handoff.md)
- **BEAD:** [sase-17p](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17p/README.md)

# Plan: E2 — durable ToolRun hand-off and lifecycle control

## Outcome and scope

E1 gave SASE named tools and a machine-local ToolRun ledger for foreground runs. E1.5
made the wrapper faithful (ownership roots, owner process groups, merged streams,
identity-matched reaping) and made `verify` monitors wrap what they run. What is still
missing is exactly the roadmap's E2 promise, restated as a contract:

> A handed-off tool has a durable identity before the caller relinquishes control. It
> remains discoverable, observable, and stoppable through that identity, and eventually
> reports either an authoritative outcome or an explicit uncertainty.

A normal user must be able to verify it without reading internals:

- `sase tool run -H check` from a terminal returns at once with a run id; closing the
  shell does not affect the run; `sase tool show RUN -F` streams it live;
  `sase tool stop RUN` stops it; `sase tool wait RUN` returns its exit code; exactly one
  ToolRun is linked to exactly one proc; one notification reports the result.
- `sase monitor start -p verify ... -- just check` inside an agent prints the reserved
  ToolRun id before the agent's turn ends, and the same run id works with `show -F`,
  `stop`, and `wait`.
- Killing any process at any point of a hand-off leaves either a recoverable run or a
  run that says, in a typed field, why its outcome is unknown or why its command never
  ran. Nothing is replayed and no success is fabricated.

Design inputs, read through `sase artifact read`:

- `research:202609/sase_tool_epic_roadmap/sase_tool_epic_roadmap.md` (§1, §3.2, §4 E2)
  and, only if a phase needs the detail, its `__a` and `__b` siblings.
- `research:202609/sase_tool_guarded_recipes_and_monitor_wrapping/sase_tool_guarded_recipes_and_monitor_wrapping.md`
  (§7) and `plan:202609/tool_e15_enforced_adoption.md` — the wrapping, ownership, and
  process-group contracts this epic extends and must not regress.
- `research:202609/muse_harness_waits_vs_sase_monitors__critique.md` — the source of
  `sase tool wait` and of the deferred detach/join work (`sase-17g`).
- A 2026-09-24 external review of E2 (not in the artifact store; its conclusions are
  folded into "Decisions this plan settles" below, and this plan is the authority where
  they differ).
- `decisions:record-before-admit`, `decisions:guarded-recipes`,
  `decisions:single-turn-agents`, `decisions:host-owned-completion`,
  `decisions:rust-core-required`, plus `sase/memory/lint_and_test.md`,
  `sase/memory/sase_flags.md`, `sase/memory/cli_rules.md`,
  `sase/memory/generated_skills.md`, and `sase/memory/symvision.md` through
  `/sase_memory_read`.

Planning baseline: sase master `b85538009`; core pin
`ae9dbf6e0719761d825478021aa8ff8d7b27aae8` (11 commits past `v0.34.73`; the linked
checkout's `master` equals the pin and has no ToolRun commits after it). E1 (`sase-135`)
and E1.5 (`sase-16h`) are closed. `sase-145` (persist ToolRun finish diagnostics) is
open and `ready` and is absorbed by phase `core-contract`. `sase-17g` (detach / wait /
join), `sase-17e` (mechanical inline-vs-monitor routing), and `sase-12r` (monitor start
lost mid-start) are open and stay out of scope (see "Relationship to open work").
Recheck each fact when implementing; do not redo landed work and do not change unrelated
task lifecycles.

Every phase that touches a repository other than the sase checkout opens it with
`/sase_repo`, reads that repo's own `AGENTS.md`, and uses only the printed path. Paths
below are repository-relative and never name a numbered checkout. Commits and release
sequencing stay with host-owned finalizers. No implementation change precedes approval.

## Decisions this plan settles

1. **Keep E2's scope and the executor stack; redesign the launch and settlement
   contract.** A ToolRun stays a semantic record over the existing executors: the inline
   wrapper, the monitor (itself a proc-shell row run by the shared proc supervisor), and
   the plain durable proc. No new supervisor, no service-host oneshot procs for tool
   runs (they carry the hidden `service` marker and slot limits; tool procs must stay
   visible), no workflow engine. The MCP Tasks extension is a possible future adapter
   over ToolRuns, not E2's state machine.

2. **Launch identity is a protocol: reserve → bind → adopt → acknowledge.** The launcher
   pre-allocates the owner id (a proc id, or the monitor id), then writes one `created`
   ToolRun that already names that owner and carries a private launch envelope. Only
   then does it start the owner, whose command is a hidden worker that atomically claims
   that exact run (`created → running`) and executes the frozen invocation. The launcher
   acknowledges only after the reservation is committed and the proc barrier released.
   Binding at reservation time needs no separate bind call and closes the window the
   external review worried about, because no user command can run before the reservation
   names its owner. `SASE_TOOL_RUN_ID` keeps its E1 meaning (enclosing parent run) and
   never means "adopt this run".

3. **The printed handle must be durable.** Explicit `sase tool run -H` is
   **fail-closed**: if the reservation cannot be committed, nothing is started and the
   command says so (the foreground form is the fail-open alternative). Foreground
   `sase tool run` stays **fail-open**, unchanged. A monitor start's reservation is
   **fail-open**: the monitor id is the handle an agent receives and it is already
   durable, so a failed reservation falls back to today's E1.5 wrapping with one reason
   line in the monitor log. Once a hand-off is accepted, later recording failures
   (observe, sample, finish) stay fail-open and are reported as incomplete evidence,
   never as a reason to replay.

4. **Agents hand off through `sase monitor start`; `sase tool run -H` refuses inside an
   agent or a live owner.** A hand-off inside an agent ends the turn, so it needs an
   explicit continuation policy (profile, policy, `--next`, prepared completion,
   checkpoint, model). `sase monitor start` already owns and validates that surface.
   Duplicating it on `sase tool run` would fork one policy grammar into two parsers, and
   defaulting it (for example, silently applying `verify`) would let the execution mode
   decide what the next agent does. This deliberately narrows the roadmap's wording
   ("`-H`: monitor leg inside agents"): the monitor leg exists and is reached through
   `sase monitor start ... -- sase tool run TOOL` or E1.5's automatic wrap. Inside an
   agent, `-H` exits 2 and prints that exact monitor form. Inside a live monitor or proc
   owner, `-H` also exits 2: a detached run would escape that owner's lifetime and
   workspace claim. A TUI `!` command is already a detached proc, so the message there
   says to drop `-H`.

5. **Frozen invocation.** The resolved invocation (the fields of `ResolvedToolArgv`) is
   stored privately in the ToolRun store at reservation and returned only by a
   successful claim. It never appears in list/show/JSON output or in the proc row, which
   carries only the worker argv (`<sase> tool _adopt RUN`). A catalog edit after
   acceptance cannot change what runs under that id.

6. **Additive wire, schema 1, no new enum values.** Older cores reject unknown
   `executor`, `state`, or `source` strings when loading rows, and several sase venvs on
   one machine can share `~/.sase/tools/runs.sqlite`. So E2 adds only nullable columns
   and optional fields, reads its new text fields leniently, and keeps
   `executor: inline` (the worker-to-child relation). How a run was accepted lives in a
   new `launch_mode` (`foreground` | `handoff`; NULL means foreground). Its execution
   owner stays in the existing `owner_kind`/`owner_id`, whose E1.5 meaning ("the owner
   this run executes under") is unchanged.

7. **Typed terminal causes; stop and timeout are not verification failures.** Every
   settlement written by this epic records `terminal_cause`, one of: `exited`, `signal`,
   `interrupt`, `stop_requested`, `timeout`, `launch_failed`, `owner_lost`, or
   `wrapper_lost`. A requested stop settles `signaled` with `stop_requested` in every
   phase, including before the command started. A launch that never ran the command
   settles `failed` with `launch_failed` and no exit code. A diagnostic states "command
   was not run" wherever that is known.

8. **One process-control path and one log owner.** Stopping a hand-off goes through its
   owner's existing control API. The ToolRun reaper never signals a run that has an
   owner. The owner's log is the output of record: its locator is recorded on the run,
   `show -l` and `show -F` read it, and nothing copies it. A missing or expired owner
   log is reported, never reconstructed.

9. **One delivery per hand-off, through the existing path.** A proc-owned hand-off
   publishes exactly one settlement notification: a deterministic id, a durability check
   under a lock, and retries from every settling process. A monitor-owned run delivers
   only through the monitor's own continuation and never also notifies.

10. **No replay, no idempotency keys, no single-flight.** Each `-H` invocation is a new
    request, and a run id executes at most once, which the claim guarantees. A lost
    acknowledgement leaves the first run discoverable in `sase tool runs` with its owner
    and does not dedupe a retry. That follows the roadmap's deferral of single-flight
    joining (zero repeated request fingerprints measured) instead of inventing a key no
    caller can supply today. One ToolRun is not an exactly-once guarantee for external
    side effects, and the docs say so.

11. **`sase tool wait` ships in E2; detach/join does not.** A bounded, exit-mirroring
    wait costs little beside `show -F` (the same observation loop), is useful to scripts
    after `-H`, and is the first slice `sase-17g` needs. In-agent `--detach` and
    `sase monitor start --join` need a turn-end and workspace-claim design of their own,
    so they stay in `sase-17g`.

12. **One beta flag, `tool_handoff`.** It gates `sase tool run -H` (phase
    `standalone-handoff`) and monitor-start reservation (phase `monitor-handoff`, which
    touches the hot verify-monitor path). Both would otherwise expose a hand-off whose
    stop, wait, and crash recovery are not landed yet. `stop`, `show -F`, and `wait` are
    complete for existing runs in their own phase and are not gated. Phase
    `acceptance-and-adoption` removes the flag.

13. **Absorb `sase-145`.** Settled hand-offs must explain themselves ("command was not
    run", "recovered from owner result", "log expired"), which needs the same durable
    finish-diagnostics channel `sase-145` asks for. Phase `core-contract` builds it and
    closes `sase-145` once that bead's acceptance holds.

## Binding contracts

### Launch protocol

```text
launcher (sase tool run -H, or sase monitor start)
  1. resolve the invocation (catalog or ad-hoc; cwd explicit)       -> exit 2 on usage
  2. allocate run id and owner id (proc id | monitor id)
  3. tool_run_begin(commit_running=false, launch_mode=handoff,
                    owner_kind, owner_id, launch envelope,
                    launcher pid/boot/identity, events path)         -> -H: refuse on failure
  4. submit the owner (proc or monitor proc) with argv
     [<sase>, tool, _adopt, RUN], tags [tool-run, tool-run:RUN],
     request_fingerprint tool-run:RUN                                -> on failure: finish
                                                                        failed/launch_failed
  5. after the proc barrier is released: acknowledge (run id, owner id, hints)
worker (sase tool _adopt RUN, inside the owner, env carries SASE_PROC_ID / SASE_MONITOR_ID)
  6. install signal handlers, then tool_run_claim(run, owner from env, worker identity,
                                                  owner log locator)
       claimed  -> run the frozen invocation exactly as a foreground run under an owner
       refused  -> print the typed reason to the owner log, run nothing, exit 2
       stopped  -> the claim settled it signaled/stop_requested; run nothing
```

- `<sase>` is `sys.executable -m sase`, never the first `sase` on `PATH`, exactly as
  E1.5's monitor wrapping does.
- The worker ignores inherited `SASE_TOOL_RUN_ID`/`SASE_TOOL_RUN_EVENTS`; the launcher
  also overlays them empty in the owner's env. A hand-off run records no
  `parent_run_id`: like an agent, a hand-off is a new ownership root.
- After a successful claim the worker is byte-for-byte the E1.5 owner-mode executor:
  child in the worker's process group, no independent escalation, merged pipe into the
  owner log, stage ingestion, load samples, and before/after fingerprints. Its stderr
  lines (`sase tool run RUN`, stage lines, footer) are unchanged, so monitor evidence
  extraction (`--next-output auto`) keeps working.
- Environment markers (`SASE_TOOL_NAME`, `SASE_TOOL_PROJECT_ROOT`) are derived from the
  envelope, so the recipe guard sees a hand-off exactly as it sees a foreground run.

### Durable record additions (sase-core, wire schema stays 1)

| Addition                | Where                              | Meaning                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ----------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `launch_mode`           | begin request, run                 | `foreground` (default, NULL on old rows) or `handoff`                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `launch` envelope       | begin request only; private column | the frozen invocation, returned only by a successful claim                                                                                                                                                                                                                                                                                                                                                                                                             |
| `tool_run_claim`        | new request/result + binding       | atomic `created → running` for a `handoff` run whose owner matches; sets the worker's pid/boot/identity, the running time, and the owner log locator; returns the envelope. Expected refusals (`not_created`, `owner_mismatch`, `already_claimed`, `not_handoff`) are typed result values, not exceptions. A pending stop request makes the claim settle the run `signaled`/`stop_requested` and return `stopped`. Replay by the same claimant identity is idempotent. |
| `tool_run_request_stop` | new request/result + binding       | durable, idempotent stop request (`requested_by`, `reason`, timestamp; first request wins); an already-settled run returns `already_settled` without error                                                                                                                                                                                                                                                                                                             |
| `terminal_cause`        | finish request, reconcile, run     | the typed cause from decision 7; unknown stored values read back verbatim                                                                                                                                                                                                                                                                                                                                                                                              |
| `settled_by`            | run                                | `wrapper` (wrapper or worker finish), `reconcile` (lost), or `owner` (outcome recovered from the owner's result)                                                                                                                                                                                                                                                                                                                                                       |
| `stop_request`          | run                                | the recorded stop request, if any                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `logs.owner_log_path`   | begin request, claim request, run  | the owner's log file (the proc log, or a monitor's `live_reply.md`), for owner-bound runs                                                                                                                                                                                                                                                                                                                                                                              |
| persisted `diagnostics` | finish, reconcile, run             | durable settlement diagnostics (`sase-145`)                                                                                                                                                                                                                                                                                                                                                                                                                            |
| transitions             | store                              | add `created → failed` (only with `launch_failed`) and `created → signaled` (only with `stop_requested`)                                                                                                                                                                                                                                                                                                                                                               |
| duplicate `run_id`      | begin                              | a typed duplicate-run error instead of a raw SQLite error                                                                                                                                                                                                                                                                                                                                                                                                              |
| reap candidates         | reconcile                          | emitted only for runs with no owner                                                                                                                                                                                                                                                                                                                                                                                                                                    |

New columns follow the existing `ensure_child_observation_columns` precedent:
`ALTER TABLE ... ADD COLUMN` on write-open, a has-column probe with a NULL placeholder
in `load_run`, and schema version unchanged, so older cores can still read the store.
The retention and stats SQL that hard-codes the unsettled states needs no change,
because no state is added.

### Reconciliation (Rust decides; Python only observes)

Python supplies, per unsettled run, the existing wrapper liveness fact plus, for
owner-bound runs, an owner fact:
`{kind, id, state: active | terminal | missing | unknown, exit_code?, termination_reason?, stop_requested?}`.
The owner fact is read from the proc store, since a monitor's proc row has
`proc_id == monitor_id`. Owner facts are collected for hand-off runs only. A foreground
run nested in an enclosing owner (for example
`sase proc run -- sh -c 'sase tool run check; deploy'`) shares its owner with other
work, so the owner's exit code says nothing about the run. Rust applies this table and
nothing else. Rows are evaluated in order, and the first match wins:

| Run                   | Facts                                                                                    | Result                                                                                                                                                                     |
| --------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `created`, handoff    | owner active, or launcher alive                                                          | none (starting, or the launcher will settle it)                                                                                                                            |
| `created`, handoff    | owner terminal, stop requested on the run or the owner                                   | `signaled`, `stop_requested`, "command was not run"                                                                                                                        |
| `created`, handoff    | owner terminal otherwise                                                                 | `failed`, `launch_failed`, "command was not run"                                                                                                                           |
| `created`, handoff    | owner missing and launcher dead                                                          | `failed`, `launch_failed`, "command was not run"                                                                                                                           |
| `created`, handoff    | anything unknown                                                                         | none; a dead launcher alone is never proof                                                                                                                                 |
| `running`, handoff    | worker alive, or worker dead and owner still active                                      | none (the owner is settling)                                                                                                                                               |
| `running`, handoff    | worker dead, owner terminal with a non-negative exit code, termination `success`/`error` | recovered: `succeeded` if 0, else `failed` with that code; `exited`; `settled_by: owner`; diagnostic "recovered from owner result; fingerprints and stages may be missing" |
| `running`, handoff    | worker dead, owner terminal via stop or timeout                                          | `signaled`, `stop_requested` or `timeout`; the child's own exit is not invented                                                                                            |
| `running`, handoff    | worker dead, and the owner terminal via supervisor loss or reboot, or missing            | `lost`, `owner_lost`                                                                                                                                                       |
| `running`, foreground | existing wrapper rules, with or without an enclosing owner                               | `lost`, `wrapper_lost`                                                                                                                                                     |

Every transition is idempotent: re-applying the same facts to a settled run is a no-op,
and never produces another transition, notification, or continuation. A launcher or
worker whose own finish loses the race to reconcile treats "already settled" as success,
not as an error. Nothing is ever re-executed.

### Stop, follow, and wait

- **`sase tool stop RUN`** records the durable stop request first, then routes by owner:
  - a hand-off run owned by a proc → the proc stop path (`kill_proc` /
    `stop_proc_shell`);
  - a hand-off run owned by a monitor → `stop_monitor`, whose existing semantics
    suppress the follow-up;
  - an inline foreground run with no owner → SIGTERM to the recorded wrapper pid, only
    while its process-start identity still matches (the wrapper forwards to its child
    and escalates as today);
  - a foreground run nested inside an enclosing owner → exit 2, naming that owner and
    the command that stops it. Signaling the nested wrapper could hit sibling processes
    in the owner's shared process group, and stopping the owner could stop more than the
    run.

  It reports `stop requested` separately from `stopped`. It tolerates completion racing
  the stop, including the proc store's "cannot request stop for terminal status" race.
  Stopping an already-settled run prints "already <state>; nothing to do" and exits 0.
  When the owner is terminal but the run is still `created`, stop settles the run itself
  as `signaled`/`stop_requested` with "command was not run". Exit codes: 0 for stop
  requested, stopped, or already settled; 2 for an unknown run or a refused nested run;
  1 when owner control fails.

- **`sase tool show RUN -F/--follow`** prints what is retained so far, then streams the
  run's output of record (the owner log for owner-bound runs, the retained logs for
  inline runs) together with stage lines, until the run settles, then prints the
  terminal summary. It polls the ledger with backoff and runs a read-only reconcile (no
  reaping), so a dead owner settles while someone follows. Ctrl-C detaches the viewer
  and exits 130; the run continues. `-F -j` waits, then prints the final JSON envelope.
  `-F` with `-l` is a usage error.
- **`sase tool wait RUN [-t DURATION] [-T N] [-j]`** blocks until the run settles or the
  deadline passes. Exit codes:
  - a settled run with an exit code returns that code;
  - a settled run without one (`lost`, `launch_failed`, stopped before start) returns 1
    and prints the terminal cause;
  - a passed deadline returns 124 and prints "still running";
  - an unknown run returns 2;
  - Ctrl-C returns 130.

  The run is never affected. `-T N` appends the last N lines of the output of record.

### Output, logs, and retention

- An owner-bound run's output of record is the owner's log, including any owner or
  worker metadata lines. `show -l` replays it through the owner adapters' existing
  readers, including the rotated `.log.1` segment. Inline runs keep E1's retained
  stdout/stderr files.
- Proc rows are pruned after `procs.history_limit` newer terminal procs, and their
  store-owned logs go with them. The ToolRun summary survives (`summary_days`). `show`
  on such a run reports the owner and the log as no longer retained, and names which
  one.
- `sase disk` keeps a single owner per byte: the ToolRun owner never counts or reaps
  proc or monitor logs.

### CLI surface

Per `sase/memory/cli_rules.md`: subcommands stay sorted
(`list, run, runs, show, stop, wait`), every public long option gets a short alias, and
help text states the exit-code contract. `sase tool run` gains `-H/--hand-off`, which,
like the other run options, must precede `TOOL` or `--` because the remainder is
captured verbatim. With `-H`, `-q` prints only the full run id (as `sase proc run -q`
prints only the proc id), and `-v` or an explicit `-T` is a usage error. `show` gains
`-F/--follow`, matching `sase monitor show` and `sase proc show`. `stop` takes
`-j/--json`. `wait` takes `-j/--json`, `-T/--tail-lines`, and `-t/--timeout`, whose
duration grammar is the one `sase monitor start` already parses. `_adopt` is an internal
subprocess entry point, hidden like `sase monitor _supervise`, so the short-alias rule
does not apply. This epic adds no config fields; if a phase adds one, it also goes in
`src/sase/default_config.yml`.

### Memory and governance

Every memory or skill-source change is named here so plan approval carries its
authorization. Each is routed through `/sase_memory_write` and republished with
`sase memory init`:

- `sase/memory/lint_and_test.md` (phase `acceptance-and-adoption`) — the roadmap's E2
  guidance change. Keep `check-full` routed through `/sase_monitor`, and add that the
  monitor prints the reserved ToolRun id, that `sase tool show RUN -F`,
  `sase tool stop RUN`, and `sase tool wait RUN` work on it, and that `sase tool run -H`
  is the non-agent hand-off and refuses inside agents.
- the `glossary:tool-run` strand (phase `acceptance-and-adoption`) — it currently
  defines a Tool Run as "one foreground execution", which becomes false. Redefine it as
  one execution, run in the foreground or handed off to a monitor or proc that owns it.
- skill source `src/sase/xprompts/skills/sase_monitor.md` (phase
  `acceptance-and-adoption`, not memory) — the start output now names the ToolRun id,
  and `sase tool stop`/`show -F` are equivalent handles. Preview with
  `sase skill init --diff` only, per `sase/memory/generated_skills.md`.

**Deliberately not in this plan: a decisions strand.** The fail-closed `-H` rule
(decision 3) narrows one clause of `decisions:record-before-admit` ("recording stays
fail-open") for explicit hand-offs only. The durable home for that change of course is a
new decisions strand plus a `superseded-in-part` mark on `record-before-admit`. It was
not requested, so this plan does not write it. The reviewer can add it in plan feedback.
Otherwise the final phase records it as a `PROPOSED FOLLOW-UP:` note, and `docs/tool.md`
documents the rule in the meantime.

The ongoing agent-session rename (`sase-17m`) changes shared monitor/session field
spellings. Any new monitor meta field in this epic uses the flat `monitor_*` convention
the adapter table already maps, reads through the shared accessors
(`agent_session_shell_from_mapping` and friends), and never embeds `agent_family` or
`family_shell` spellings in a new durable contract.

### Acceptance cannot depend on a green master

`check` fails most of the time on the ledger's real history (red master, `sase-j0`).
Every exit criterion below is observable behavior — a run id, a state, a cause, an
owner, an argv, a surviving or absent process, a notification count — never
"`just check` passes". Phases still run `sase tool run check` as `lint_and_test.md`
requires (and `sase tool run check` inside the sase-core checkout for core work), but
fixture commands stand in for real verification in acceptance tests.

### Relationship to open work (out of scope)

- `sase-17g` (in-agent `--detach`, ceiling-bounded wait, `sase monitor start --join`)
  builds on this epic's reservation, owner binding, and `wait`. Its turn-end and
  workspace-claim design stays with it.
- `sase-17e` (declared duration classes and mechanical routing) and E6 automatic
  routing: E2 routes only on the explicit `-H` and on monitor start.
- `sase-12r` (a monitor start killed before acknowledgement): with phase
  `monitor-handoff`, such a start leaves a `created` run bound to the monitor id, which
  reconcile settles `launch_failed`. That is durable evidence of the lost start, but the
  runner-side fix stays in `sase-12r`.
- TUI surfaces (E5), receipts (E4), triage (E3), admission (E7), fleet (E8), and any
  change to the tool catalog schema (`ToolDefinitionWire` is `deny_unknown_fields` and
  changing it moves every definition digest).

## 1. core-contract

Open `sase-core` with `/sase_repo`, read its `AGENTS.md`, and work only in the printed
path. Run its verification as `sase tool run check` from inside that checkout (about
five minutes; give it a generous explicit timeout). Never run bare `cargo`.

Under `crates/sase_core/src/tool_run/` implement every row of "Durable record additions"
and the reconcile table in "Reconciliation" exactly as specified:

- `wire.rs`:
  - new optional request fields with `#[serde(default, skip_serializing_if = ...)]`;
  - the new claim and stop request/result wires, which stay `deny_unknown_fields` like
    their siblings;
  - result fields on `ToolRunWire`;
  - a typed launch envelope carrying the `ResolvedToolArgv` fields: `argv`, `cwd`,
    `tool_name`, `extra_args`, `display_argv`, `private_argv`, `definition`, `digest`,
    and `adhoc`.

  Validate that the envelope is consistent with the reserved identity: its argv must
  equal the stored protected argv (private argv when present, else display argv), and a
  named definition must re-digest to `definition_digest`. Keep `canonical_event`
  replay-stable: a new event field must not serialize when absent, or old replays
  conflict.

- `store/connection.rs`:
  - add the new columns through a generalized ensure/`ALTER` helper on write-open,
    covering `launch_mode`, `launch_envelope_json`, `terminal_cause`, `settled_by`,
    `stop_request_json`, `owner_log_path`, and `diagnostics` persistence;
  - keep `meta.schema_version` at 1.
- `store/lifecycle.rs`:
  - begin with `commit_running: false` plus `launch_mode: handoff` (this path has no
    tests today; add them);
  - add a typed duplicate-run error and the claim, including its atomic stop-request
    check and replay-by-claimant;
  - add the stop request;
  - add the two new `created` transitions, each restricted to its cause;
  - make finish accept `terminal_cause` and persist diagnostics (closing the gap where
    `canonical_event` strips them);
  - apply the owner-aware reconcile table;
  - restrict reap candidates to runs with no owner.
- `store/query.rs`: has-column probes and lenient reads for every new column. List and
  show never serialize the envelope, just as they never serialize `private_argv`.

Add store tests for:

- every reconcile table row;
- the claim: success, each refusal, and replay;
- stop requests: before claim, during running, and on a settled run;
- both new transitions and their cause restrictions;
- a duplicate run id;
- persisted finish diagnostics;
- an old-shape store (without the new columns) read by the new core;
- a new-shape store still loadable by the pre-change query shape, which is what an older
  core on the same machine runs;
- query output never containing the envelope.

Add golden fixtures in `tool_run/fixtures/` for the claim request/result, the stop
request/result, and an owner fact.

Expose `tool_run_claim` and `tool_run_request_stop` through `crates/sase_core_py` beside
the other `tool_run_*` bindings, extend the binding round-trip test, and keep
`ToolRunError` text stable for existing callers.

On the sase side:

- add thin adapters to `src/sase/core/tool_run.py`;
- extend `tools/check_sase_core_rs_bindings`, `tools/validate_sase_core_rs`, and
  `tools/smoke_sase_core_rs_tool_runs` deliberately, never by pasting whatever a run
  prints;
- extend `tests/core/test_tool_run_store.py` with a real-binding round trip of reserve,
  claim, stop, finish, and reconcile.

Nothing in `src/sase/` calls the new adapters yet. Read `sase/memory/symvision.md` and
whitelist them with `--epic-symbol <this epic's bead id>(<symbol>)` in the Justfile's
symvision command. Phases `standalone-handoff` and `lifecycle-controls` remove those
entries when their callers land. Once the sase-core commit is pushed and reachable, move
`sase-core-revision.txt` past it so CI builds a core exposing the bindings before phase
`standalone-handoff` calls them. Do not touch the published `sase-core-rs` window in
`pyproject.toml`; the release job owns it.

Close `sase-145` in this phase only if its three cases are covered: the typed spawn
failure reason behind 127/126, stage-ingest diagnostics, and log-write facts. Each must
be sent at finish (the executor already passes some of them) and visible in
`sase tool show RUN -j`. If the Python rendering is not reached here, leave the bead
open, add a note, and let phase `settlement` finish and close it.

## 2. standalone-handoff

Create the `tool_handoff` beta flag with `sase flag new tool_handoff`. Its three
sentences are:

- enabled: `sase tool run -H` hands a run off, and monitor starts reserve their ToolRun;
- disabled: `-H` is refused with a one-line message naming the flag, and monitor starts
  keep E1.5 wrapping;
- remove when: the E2 epic lands with the smoke fault matrix green.

Paste the printed registry entry. Every flagged path gets tests in both states.

Split `src/sase/tool/executor.py` so there is one post-begin body. The foreground path
keeps begin-then-run with E1's fail-open semantics. The adopted path claims then runs,
and never runs anything without a claim. Spawning, pumps, stage ingestion, samples,
fingerprints, observe, finish, the footer, and signal handling must not fork into two
copies. The shared body records `terminal_cause` on every finish it writes: `exited` for
a normal exit, `interrupt` for a forwarded SIGINT, `signal` for any other signal, and
`launch_failed` for a spawn failure (127/126). Phase `lifecycle-controls` refines
`signal` into `stop_requested`, and phase `settlement` refines it into `timeout`. Make
argv resolution cwd-explicit: `resolve_run_argv` resolves the catalog and project root
from a given directory instead of the process cwd, because a monitor start resolves on
behalf of `--cwd`. The foreground caller passes the process cwd, so its behavior is
unchanged.

Add `src/sase/tool/handoff.py` as the single launch module both legs use:

- invocation → envelope;
- reservation, returning a typed result that says whether the reservation failed;
- the worker argv and env overlay (empty `SASE_TOOL_RUN_ID`/`SASE_TOOL_RUN_EVENTS`);
- the owner tags and request fingerprint;
- launch-failure settlement (`failed`/`launch_failed`, "command was not run", the owner
  error as a diagnostic).

Add the hidden `sase tool _adopt RUN` worker (parser plus handler). It derives its owner
from `SASE_MONITOR_ID` (monitor) or else `SASE_PROC_ID` (proc), installs signal handlers
before claiming, records the owner log locator from `SASE_PROC_LOG_PATH`, and then runs
the shared body in owner mode. A refused claim prints its typed reason and exits 2
without spawning. A signal that arrives before spawn settles the run as a foreground run
does today, with `stop_requested` when the run or its owner recorded a stop.

Add `-H/--hand-off` to `sase tool run` (see "CLI surface"). Outside an agent and outside
any live owner:

- resolve;
- pre-allocate the proc id;
- reserve fail-closed; on failure exit 1, say nothing was started, and name the
  foreground form;
- submit through `submit_proc_request` with `origin="tool-run"`, the worker argv, the
  redacted `sase tool run …` form as the logical `command`, a `tool:<name>` label
  (`tool:ad-hoc` for ad-hoc), the invocation's project/workspace/session inference as
  `sase proc run` does it, the tags and fingerprint, and a follow-up block
  `{"kind": "tool-run", "run_id": RUN}`;
- on a submit failure, settle the run `launch_failed` and exit 1;
- otherwise print the acknowledgement: run id, tool, proc id, and `show -F` / `wait` /
  `stop` hints, stating that the run may still be starting.

Inside an agent (`SASE_AGENT`), or when `resolve_ownership` finds a live monitor or proc
owner, exit 2 before reserving anything and print the exact
`sase monitor start -p verify --reason '<why>' -- sase tool run <words>` form. The proc
leg rides the proc runner's existing `detach_scope` handling; add no tool-local cgroup
code.

Make the Python liveness pass stop reporting a dead launcher as proof for a `created`
hand-off run. The core rule already refuses it; do not rely on that alone.

Tests: the worker's claimed, refused, and stopped paths with a fake owner env; an
unchanged foreground path (existing executor tests stay green untouched); `-H` from a
temp project with an isolated `SASE_HOME` returns before a slow fixture command
finishes, and the run then settles with the child's exit code under `owner_kind=proc`;
exactly one ToolRun and one proc with tag `tool-run:RUN`; a secret-bearing ad-hoc argv
never reaches the proc row (`sase proc show -j` shows only the worker argv and the
redacted logical command); an unwritable store refuses and spawns nothing, while
foreground `sase tool run` in the same state still runs fail-open; a catalog edit
between reservation and claim still executes the frozen argv; refusals inside an agent
and inside an owner; both flag states. Remove this phase's callers' symvision whitelist
entries.

## 3. monitor-handoff

In `src/sase/monitor/start.py`, after replay detection, monitor-id allocation, and
member creation, and only while `tool_handoff` is enabled, reserve the ToolRun whenever
`resolve_monitor_tool_wrap` produces a tool run for the proc:

- rule 2, an agent-written `sase tool run …`: parse its words with the real `tool run`
  parser; if it carries output-mode options or does not parse, keep the E1.5 argv
  unchanged;
- rule 7, a named upgrade;
- rule 8, an ad-hoc `verify` wrap.

Resolve against `request.cwd`. Reserve with `owner_kind=monitor` and
`owner_id=monitor_id`; attribute it to the starter agent (the start runs in the agent's
own shell, so `SASE_AGENT_NAME` attribution is direct). The proc argv is then the worker
argv plus the owner tags. Never reserve for rule 1 (a host-owned `execution_argv`), and
never on the replay path.

Leave `monitor_command` exactly as written and leave `monitor_execution_argv` unset for
ordinary monitors, because `host_completion_state._command_argv` reads them and
prepared-completion `-f` bindings must keep matching. Persist the run id as the flat
`monitor_tool_run_id` meta field. Show it in the start output (`tool run: RUN`), the
`--json` envelope, and `sase monitor show`. Where the follow-up's command-run breakdown
already lists command facts, add the run id so the follow-up can `sase tool show RUN`.

If the reservation fails, keep the E1.5 argv (fail-open) and pre-write one reason line
to the monitor log, as E1.5 does for unwrapped runs. If `start_monitor` fails after the
reservation (for example `MonitorAlreadyRunningError` or a claim or submit error),
settle the run `launch_failed` with the monitor error as its diagnostic, alongside the
existing claim undo, completion rollback, and member teardown.

Tests:

- `sase monitor start -p verify -- just check` in a fixture project creates exactly one
  named run, with `owner_kind=monitor`, the starter agent, and the id printed before the
  hand-off;
- an explicit `-- sase tool run check` is reserved, not double-recorded;
- an `-f` monitor's host completion still succeeds, and its recorded command is
  unchanged;
- an epic launch creates no ToolRun;
- a failed reservation falls back with a reason line;
- a start failure after reservation leaves the run `failed`/`launch_failed`;
- `--next-output auto` extraction tolerates the worker's lines;
- both flag states.

## 4. lifecycle-controls

Implement `sase tool stop`, `sase tool show -F/--follow`, and `sase tool wait` per
"Stop, follow, and wait" and "CLI surface", in `src/sase/main/parser_tool.py`,
`src/sase/main/tool_handler.py`, and `src/sase/tool/` (a new `control.py` for stop and
wait, and `query.py` for follow). Owner routing goes through small owner adapters: proc
via `sase.procs`, monitor via `sase.monitor`. Do not add a second TERM/KILL loop for
owner-bound runs.

For the inline wrapper, signal only after the recorded wrapper identity matches. Have
the foreground executor record `stop_requested` as its terminal cause when a durable
stop request exists at the time the forwarded SIGTERM settles it. Also have a foreground
run under an enclosing owner record `logs.owner_log_path` at begin (from
`SASE_PROC_LOG_PATH`), so `show -l` and `show -F` can find E1.5 monitor-wrapped output
as well; runs recorded before this change simply lack the locator, and `show` says so.
Share one observation loop between follow and wait: ledger polling with backoff,
read-only reconcile, owner log tailing through the owner adapters' readers, and stage
lines from the events file.

Render `launch_mode`, the owner, `stop_request`, `terminal_cause`, `settled_by`, and
persisted diagnostics in `show` (human and `-j`). Mark hand-off runs and a `created` run
("starting") in `sase tool runs` without breaking existing JSON consumers.

Tests:

- stop of a proc-owned hand-off while it starts, while it runs, and racing completion;
- stop of a monitor-owned run (the follow-up is suppressed);
- stop of an inline run in another process (identity mismatch signals nothing and says
  why);
- refusal of a nested foreground run with the owner pointer;
- stop of a settled run;
- `show -F` streaming, then settling; Ctrl-C leaves the run running;
- `wait` exit mirroring, 124 on deadline, 1 for a run with no exit code, and `-T` tails;
- JSON shapes for `stop`, `wait`, and `show -F -j`.

Remove this phase's callers' symvision whitelist entries. Keep the `sase tool --help`
subcommand list sorted.

## 5. settlement

Feed owner facts into reconcile. In `src/sase/tool/liveness.py`, for each unsettled
hand-off run (and only those, per "Reconciliation"), read the owner's proc row: status,
exit code, the result envelope's `termination_reason`, and `stop_requested_at`. Send it
as the owner fact beside the existing wrapper fact. A missing row is `missing`; an
unreadable store is `unknown`. The executor path keeps `reap_orphans=True`, but core no
longer authorizes candidates for owner-bound runs, and Python must still refuse to
signal any group whose owner is active.

Make the stop and timeout causes knowable at the moment of termination:

- `procs/supervisor.py` persists a termination intent (`stop`, `total-timeout`, or
  `idle-timeout`, plus a timestamp) into the proc runtime directory **before** it
  signals the command group;
- `sase.procs` exposes a small reader for it;
- when the worker is terminated, it settles with `stop_requested` or `timeout` from that
  intent (or from a durable ToolRun stop request), and otherwise with `signal`.

Settle from the owner's side:

- register the `tool-run` follow-up kind in `settle_proc_shell`'s follow-up step (beside
  the monitor and xprompt kinds), so a proc-owned hand-off is reconciled the moment its
  proc settles;
- in monitor settlement, reconcile the bound run (from `monitor_tool_run_id`) before the
  follow-up decision;
- both steps are idempotent, checkpoint-safe under resumed settlement, best-effort (a
  ToolRun or notification error is recorded and never wedges proc settlement), and never
  launch anything.

Deliver once. For proc-owned hand-offs only:

- publish one notification per run with `sender="tool-run"`, a deterministic id (`uuid5`
  over `tool-run-settled:<run_id>`), and the exactly-once pattern already used by
  `bead/epic_launch_handoff_notifications.py` (a durability check under a file lock);
- the content is the tool, state, cause, duration, exit code, and a `sase tool show RUN`
  action;
- attempt it from the worker after its own finish, from the proc settlement step, and
  from any reconcile pass that settles or observes a settled, undelivered hand-off. The
  one id collapses them.

Monitor-owned runs never notify. Report expired or pruned owner logs explicitly in
`show`/`show -l`/`show -F`. If `sase-145` is still open, finish its Python rendering and
close it here.

Tests:

- kill the launcher after reservation and before submit → `launch_failed`;
- kill the launcher after submit → the run completes normally (caller death is not run
  loss);
- SIGKILL the worker while the owner is alive → settled from the owner result with
  `settled_by: owner` and missing evidence stated;
- SIGKILL the supervisor → `lost`/`owner_lost`, no rerun;
- a total timeout and an idle timeout → `signaled`/`timeout`;
- a stop before the barrier → `signaled`/`stop_requested`, "command was not run";
- a finish that fails on a busy store → recovered from the owner;
- repeated reconcile passes and resumed settlements produce exactly one notification;
- a pruned owner row → the summary survives and the log is reported missing;
- reboot simulated through a boot-id mismatch → truthful `lost`, nothing re-executed.

## 6. acceptance-and-adoption

Extend `tools/smoke_sase_tool_runs` and its pytest twin
(`tests/test_sase_tool_runs_smoke.py`) with a hand-off case group. Keep the harness's
rules: isolated `SASE_HOME` and private `HOME`, a temporary git project, fixture
commands only, fault injection only at the filesystem and process boundary, and live
cases labeled `not-run` without `--live`. Cover this matrix and map each row to a report
case:

| Scenario                                                 | Required result                                                                                                                |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Caller exits right after an accepted `-H`                | the same run stays observable with exactly one owner and settles normally                                                      |
| Agent-style monitor hand-off (live)                      | the run id is printed before the hand-off; correct attribution and workspace claim; only the selected continuation branch runs |
| Acknowledgement lost, request retried                    | the first run stays discoverable; the retry is a separate run; no run id executes twice                                        |
| Crash at reserve, submit, barrier, claim, and settlement | a recoverable state or a typed uncertainty; no fabricated success; no replay                                                   |
| Stop while starting, running, and completing             | the owner receives it; races are handled; unrelated processes are never signaled                                               |
| Viewer Ctrl-C (`show -F`, `wait`)                        | the viewer exits; the run continues                                                                                            |
| Store cannot record the launch                           | `-H` refuses and starts nothing; foreground stays fail-open                                                                    |
| Catalog edited after acceptance                          | the frozen argv runs under the original id                                                                                     |
| Owner row and logs pruned                                | the summary stays useful; the missing output is named                                                                          |
| Service restart / reboot                                 | restart survival where `detach_scope` escapes to a scope; a reboot yields truthful reconciliation and no rerun                 |
| Settlement delivery                                      | exactly one notification per proc-owned hand-off; none for monitor-owned runs                                                  |

Remove the `tool_handoff` flag: delete every Off branch, make the On branches
unconditional, drop the registry entry and both-state test scaffolding, and close the
flag bead in the same change. Then:

- document in `docs/tool.md`: hand-off, lifecycle controls, failure semantics including
  fail-closed `-H` versus fail-open foreground and monitor reservation, terminal causes,
  no replay and no exactly-once external side effects, and platform notes (a
  terminal-launched proc stays in the terminal's cgroup unless `detach_scope` escapes);
- update the "Tool-run wrapping" section of `docs/monitors.md` for reservation and
  `monitor_tool_run_id`;
- make the named memory and skill-source edits from "Memory and governance" through
  `/sase_memory_write`, then `sase memory init`, then `sase skill init --diff` as a
  preview only.

Record in the phase bead:

- a snapshot of `sase tool runs -a -j` owner-kind and `launch_mode` counts, as the
  adoption baseline for later epics;
- a `PROPOSED FOLLOW-UP:` note for the decisions strand described in "Memory and
  governance", unless plan feedback already added it to this phase;
- anything else discovered as `PROPOSED FOLLOW-UP:` notes, not new beads.
