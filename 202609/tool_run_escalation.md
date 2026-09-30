---
tier: epic
title: Inline-then-escalate ToolRuns (sase-17g)
goal: 'An agent''s `sase tool run` never loses a run to its provider''s synchronous
  ceiling: every run starts inline from the agent''s point of view, and a run still
  going near the ceiling moves into a monitor under the same ToolRun id without being
  cancelled or rerun, via `sase tool run --detach`, ceiling-bounded `sase tool wait`,
  and `sase monitor start -J/--join`.

  '
phases:
- id: core-detach-join
  title: sase-core starter scope, monitor join, and sync wait budget
  depends_on: []
  size: large
  description: 'core-detach-join: in the linked sase-core checkout, add the ToolRun
    `starter` record for detached hand-off runs, the atomic `tool_run_join` / `tool_run_release_join`
    APIs, `continuation_mode` in the launch envelope, and the `tool_run_sync_wait_budget`
    policy, with migrations, bindings, fixtures, and tests.'
- id: soft-ceiling
  title: Configurable per-provider soft ceiling export
  depends_on: []
  size: medium
  description: 'soft-ceiling: add the `tool_runs.soft_ceiling` config and export `SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS`
    around each provider invocation, scrubbed at every agent, monitor, and proc boundary.
    Nothing consumes it yet.'
- id: detach-run
  title: Starter-scoped detached runs and sase tool run --detach
  depends_on:
  - core-detach-join
  size: large
  description: 'detach-run: pin the new core, create the `tool_run_escalation` beta
    flag, and share one hand-off launcher between `-H` and the new agent-only `-d/--detach`.
    Scope detached runs to their starter runner with a worker watchdog and an end-of-invocation
    cleanup. Suppress their settlement notification and render `starter`/`join` in
    `sase tool show`.'
- id: bounded-wait
  title: Ceiling-bounded wait, follow, and the escalation block
  depends_on:
  - detach-run
  - soft-ceiling
  size: medium
  description: 'bounded-wait: extract a shared `follow_run` helper from `show -F`
    without changing its output. Bound agent `sase tool wait` and `show -F` by the
    core-computed sync wait budget, and print one shared escalation block (the `-J/--join`
    form) when a joinable run is still going.'
- id: monitor-join
  title: sase monitor start -J/--join and the joiner worker
  depends_on:
  - detach-run
  - bounded-wait
  size: large
  description: 'monitor-join: add `-J/--join RUN` to `sase monitor start`. It records
    the join atomically before the proc starts and releases it if the start fails.
    A hidden `sase tool _join` worker streams the run into the monitor log and mirrors
    its exit. Monitor stop and timeout, `sase tool stop`, and settlement all route
    through the join. `-f` is refused with `--join` in v1.'
- id: inline-escalation
  title: Agent sase tool run escalates instead of being killed
  depends_on:
  - soft-ceiling
  - detach-run
  - bounded-wait
  size: large
  description: 'inline-escalation: when the flag is on and a budget exists, an agent''s
    plain `sase tool run` starts detached and follows the run with inline-identical
    output. On settlement it returns the run''s exit. At the budget or on a signal
    it prints the escalation block and exits without stopping the run. It falls back
    to today''s inline run whenever detaching is unavailable.'
- id: guidance-and-flag-removal
  title: Agent guidance, docs, live harness case, and flag removal
  depends_on:
  - monitor-join
  - inline-escalation
  size: medium
  description: 'guidance-and-flag-removal: delete the flag''s Off branches and close
    its bead. Rewrite the Muse single-turn directive and the `sase_monitor`/`sase_final`
    skill sources for inline-then-escalate, finish the cross-cutting docs, and add
    the live escalate-then-join harness case.'
proposed_by: bbugyi200.athena.0u3
create_time: 2026-09-29 20:32:10
status: wip
bead_id: sase-1cx
---

- **PROMPT:** [prompts/202609/tool_run_escalation.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/tool_run_escalation.md)
- **BEAD:** [sase-1cx](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1cx/README.md)

# Plan: Inline-then-escalate ToolRuns (sase-17g)

## Why, and what the research decided

`sase-17g` asks for three things, so that an agent never has to guess up front whether a
command runs inline or in a monitor:

- `sase tool run --detach` starts a ToolRun without blocking;
- `sase tool wait <id>` waits no longer than the caller's synchronous ceiling;
- `sase monitor start --join <id>` adopts the already-running ToolRun into a monitor.

Read the bead with `sase bead read sase-17g -r '<why>'`. It also carries a note from
`sase-1cp.land`:

- `SASE_PROVIDER_SYNC_CEILING_SECONDS` already exists: Muse 600, Claude
  `BASH_MAX_TIMEOUT_MS // 1000` (14400 by default), unset otherwise.
- Configurable soft ceilings for providers without a hard ceiling stay in this bead.

The controlling research is
`research:202609/sase_tool_e6_e8_go_no_go/sase_tool_e6_e8_go_no_go.md`; read it with
`sase artifact read`. `plan:202609/tool_inline_routing.md` (`sase-17e`, landed) shipped
the ceiling export and the duration-class refusal that this epic builds on. The findings
that shape this design:

- **The largest measured loss.** Inline `check` runs are killed at the ceiling: 160 on
  athena, and 57 of them were rerun by the same agent within 30 minutes.
  - The kills cluster at 530–550 s. That is the Muse directive's `timeout 540` wrapper
    habit. Muse itself kills at 600 s and discards _all_ output.
  - Every kill came from Muse or was unattributed.
- **Hand-offs that were not needed.** 38% of handed-off `check` runs finished in under 5
  minutes, and each one paid a turn boundary.
- **No forecast will fix this.** A p10–p90 band is 18.6× wide, and predictive routing
  cannot help a run that has already started.
- **Recommendation.** Every run starts inline; a run still going at the caller's ceiling
  minus a margin moves into a monitor without restarting. Acceptance, measured over one
  athena week:
  - zero `check` runs signaled at 530–550 s;
  - no same-agent kill→rerun pairs;
  - fewer than 5% of monitor-owned runs finish in under 2 minutes;
  - every escalated run keeps **one** ToolRun id, so `show -F`, `wait`, and `stop` work
    across the move.

### Design consequence

Prompt rules plateau. The Muse directive already says "never cancel or rerun an
in-flight command", and the kills kept growing. So escalation must be **mechanical**:
when a budget exists, an agent's plain `sase tool run` starts the run detached and
follows it inline. At the budget it returns control with the exact join command while
the run keeps going. The three bead primitives are both the building blocks and the
explicit escape hatches.

A run cannot be handed off after it starts as an inline child: the wrapper owns its
pipes and it dies with the wrapper. So a run that might escalate must start under a
detached owner from the beginning.

## Contract (applies to every phase)

### Terms

- **Detached run.** A normal hand-off ToolRun that carries a `starter` record.
  - It keeps `launch_mode: handoff`, an owning proc, and the `_adopt` claim worker.
  - Every existing hand-off path therefore applies unchanged: claim, reconcile,
    `show -F`, `wait`, `stop`, receipts, and triage.
  - `starter` names the agent and the agent-runner process that started it: `agent`,
    `pid`, `boot_id`, `process_start_identity`.
- **Join.** A durable relation `{kind: monitor, id, joined_ts, requested_by}` from a
  detached run to the monitor that took responsibility for it.
  - The run's **owner stays the proc that executes it.** Ownership transfer was
    rejected. With transfer, reconcile would ask the monitor, not the dead worker, and a
    dead worker under a live monitor could never settle.
  - Stop and settlement route _through_ the join (phase `monitor-join`).
- **Wait budget.** How long an agent may block on one run. Rust computes it from the
  hard ceiling (`SASE_PROVIDER_SYNC_CEILING_SECONDS`) and the soft ceiling
  (`SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS`):
  - **Hard margin** = clamp(15% of the ceiling, 90 s, 300 s).
  - **Hard budget** = max(ceiling − margin, ceiling / 2).
  - **Soft budget** = the soft ceiling itself. A soft ceiling kills nothing, so it needs
    no margin.
  - **Budget** = the smaller of the budgets that are present. `source` is `hard` or
    `soft`; a tie reports `hard`. With neither present there is no budget.
  - **Worked values:** Muse 600 → **510 s** (under both the 540 s `timeout` habit and
    the 600 s kill); Claude 14400 → 14100 s; 60 → 30 s.
- **Joinable.** The run is unsettled, has a `starter`, has no stop request, and is
  unjoined or joined by the same monitor, and the caller's `SASE_AGENT_NAME` equals
  `starter.agent`.

### Lifecycle rules

1. **One id end to end.** A detached run keeps its id through detach, every bounded
   wait, escalation, join, and settlement. Nothing in this epic ever reruns a command.
2. **An unjoined detached run never outlives its starter.** Two mechanisms stop it, with
   cause `stop_requested` and a reason naming the starter:
   - **eagerly**, at the end of the provider invocation that started it;
   - **as a backstop**, by the worker's watchdog once the starter runner process is
     gone.
3. **A joined run lives only as long as its joining monitor.** If the monitor ends first
   (stop, timeout, or lost), the run is stopped.
4. **Recording legs**, following `decisions:explicit-handoff-fails-closed`:
   - Explicit `--detach` is **fail-closed**: the printed id is the only handle. If the
     reservation, proc launch, or starter identity fails, exit `1` and nothing is
     started.
   - The automatic path (phase `inline-escalation`) is **fail-open**. On any failure it
     runs today's inline foreground run with one `sase:` warning line.
   - `-J/--join` is **fail-closed**. If the join cannot be recorded, no monitor starts
     and the agent's turn continues.
5. **Hand-off notifications.** Detached runs never raise the settlement notification:
   either the starter reads the result inline or the joined monitor's follow-up does.
6. **Ceiling semantics are unchanged.** `SASE_PROVIDER_SYNC_CEILING_SECONDS` still means
   the hard kill ceiling, and the `sase-17e` duration-class refusal still reads only it.
   The refusal applies to the plain form. `--detach` of a `long` tool is allowed,
   because a join can finish it.
7. **Humans, CI, monitors, procs, and nested runs are untouched.** Without `SASE_AGENT`
   there is no detach, no budget, and no automatic path. `--detach` is refused outside
   agents in favour of `-H`, and inside a live owner or a parent run.
8. **Boundary.** See `docs/rust_backend.md` and the core-boundary memory.
   - **Rust `sase_core::tool_run` owns** the `starter` and `join` records, the join and
     release compare-and-swap, the budget policy and its constants, and the envelope's
     `continuation_mode`.
   - **Python owns** the CLI, proc launches, the starter resolver, the watchdog,
     cleanup, rendering, docs, and skills, as thin adapters.
   - Open the linked sase-core checkout only with `sase repo open sase-core -r '<why>'`,
     work in the printed path, and read its `AGENTS.md` first.

### The escalation block

There is one shared builder, added in phase `bounded-wait` beside `monitor_start_form`
in `src/sase/tool/routing.py`. `wait`, `show -F`, and the automatic path all print it on
stderr. Tune the wording, but keep every element:

- tool name and run id;
- elapsed time;
- "still running" and "was not stopped";
- the budget source, naming its variable;
- the join form, shell-quoted with `shlex` like `monitor_start_form`;
- the bounded re-wait form;
- the turn-end stop rule.

```text
sase tool run: check (run 0f1a2b3c...) is still running after 8m30s; it was not stopped.
This agent's provider kills synchronous commands at 10m (SASE_PROVIDER_SYNC_CEILING_SECONDS=600).
Hand the same run to a monitor and end this turn (nothing reruns):
  sase monitor start -J 0f1a2b3c... -p verify -r 'finish check (joined run)' -n '<what the follow-up should do with the result>'
Or wait again inline, bounded by the same ceiling:
  sase tool wait 0f1a2b3c... -T 200
Unless a monitor joins it, this run is stopped when this agent's turn ends.
```

The source line depends on the budget:

- **Soft budget:** say "This agent's soft ceiling is 20m
  (SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS=1200)."
- **Run that isn't joinable** (not detached, another agent's run, or already joined
  elsewhere): print only the existing "still running" line and the re-wait form. Never
  print a join command that would be refused.

### Exit codes

| Command                                            | Outcome                                                                  | Exit                                                                                               |
| -------------------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| agent `sase tool run` (automatic path)             | settled                                                                  | the run's exit code; `128+signal`; `143` for a stop with no code; `1` for `lost`                   |
| agent `sase tool run` (automatic path)             | budget reached                                                           | `124`, escalation block, run continues                                                             |
| agent `sase tool run` (automatic path)             | SIGTERM / SIGINT / SIGHUP received                                       | `143` / `130` / `129`, escalation block, run continues                                             |
| `sase tool run --detach`                           | accepted                                                                 | `0`                                                                                                |
| `sase tool run --detach`                           | usage or refusal                                                         | `2`                                                                                                |
| `sase tool run --detach`                           | nothing started                                                          | `1`                                                                                                |
| `sase tool wait` / `sase tool show -F` in an agent | budget reached                                                           | `124`; `wait` keeps its existing codes otherwise                                                   |
| `sase monitor start -J`                            | not an agent, not detached, another agent's run, or a conflicting option | `2`                                                                                                |
| `sase monitor start -J`                            | settled, stop requested, or joined elsewhere                             | `1`, with the run's state and a `sase tool show` pointer; no monitor starts and the turn continues |

### The feature flag

Phase `detach-run` creates the beta flag `tool_run_escalation` (default off) with
`sase flag new` (see `sase_flags.md`). It is epic scaffolding:

- It gates `--detach`, `-J/--join`, the `wait`/`show -F` budget, and the automatic path.
- With the flag off, `--detach` and `-J` are refused as not enabled, with exit `2`.
- The watchdog, the eager cleanup, and the notification suppression act only on detached
  runs, so they are not flag branches and stay ungated.
- Every gated behaviour is tested in both states.
- Phase `guidance-and-flag-removal` deletes the Off branches and closes the flag bead.
  The feature therefore goes live exactly when the epic lands.

## Phase `core-detach-join` (sase-core)

Work in the linked sase-core checkout. Follow its `AGENTS.md` recipe "Add a core
function and expose it to Python", and never run bare `cargo`. Everything here is
additive, so keep `TOOL_RUN_WIRE_SCHEMA_VERSION` at 1.

1. **Starter record.**
   - Add `ToolRunStarterWire { agent, pid, boot_id?, process_start_identity? }` in
     `tool_run/handoff_wire.rs`.
   - Add `starter: Option<...>` to `ToolRunBeginRequestWire` and `ToolRunWire`, with
     `#[serde(default, skip_serializing_if = "Option::is_none")]`.
   - `validate_handoff_begin` (`store/lifecycle.rs`): `starter` requires
     `launch_mode: handoff`, a non-empty `agent`, and `pid > 0`; otherwise it is a
     validation error.
   - Store it in a new `runs.starter_json` column.
2. **Join record and APIs.**
   - Add `ToolRunJoinRecordWire { kind, id, joined_ts, requested_by? }` on `ToolRunWire`
     as `join`, stored in `runs.join_json`.
   - Add these to `store/handoff.rs` beside `claim` and `request_stop`, each in one
     IMMEDIATE transaction:
     - `join(request) -> result`. The request (`deny_unknown_fields`) is
       `{schema_version, run_id, joiner_kind, joiner_id, agent?, requested_by?, now_ts?}`.
       It returns `outcome: joined | refused`, `refusal?`, `replayed`, `run`, and
       `diagnostics`. The refusal is one of:
       - `not_detached`: no starter;
       - `settled`;
       - `stop_requested`;
       - `joined_elsewhere`;
       - `agent_mismatch`: `agent` given and not equal to `starter.agent`.

       The same joiner is a replay (`joined`, `replayed = true`). An unknown run is the
       same not-found error `request_stop` uses.

     - `release_join(request) -> result`. Clear the join only when the kind and id
       match. It returns `outcome: released | not_joined | joined_elsewhere` and is
       allowed in any state, so a failed monitor start can always roll back.

   - Neither API changes `state`. Follow how `stop_request_json` is written.

3. **Envelope continuation.**
   - Add `continuation_mode: Option<String>` to `ToolRunLaunchEnvelopeWire`, validated
     as `always | never | known`, and serialized only when present. That keeps existing
     stored envelopes and fixtures byte-identical.
   - `claim` returns it inside `launch`, as today.
4. **Budget policy.**
   - In `tool_run/duration.rs`, add named constants for the margin (15%, 90 s floor, 300
     s cap).
   - Add `sync_wait_budget(request) -> response`. The request is
     `{ceiling_seconds?, soft_ceiling_seconds?}`; the response is
     `{budget_seconds?, source?, margin_seconds?, ceiling_seconds?, soft_ceiling_seconds?}`,
     following the contract formula.
   - A zero value is an error; Python passes only validated positive integers.
5. **Store.**
   - Add both columns to `SCHEMA_SQL` and to the `ensure_child_observation_columns` list
     in `store/connection.rs`.
   - Make `load_run` read them with the existing missing-column `NULL` fallback.
   - Projections stay unchanged; the TUI is a non-goal.
   - Extend `store/tests/compat.rs` so an old-shape store stays readable (starter and
     join are `None`) and gains the columns on first write.
6. **Tests and fixtures.**
   - Begin: with a starter, without handoff (error), and with an empty agent (error).
   - `show` and `claim` round-trip the starter and envelope continuation.
   - The full join and release matrix, including the replay.
   - Budget: 600→510/hard, 14400→14100, 60→30, soft only, both (smaller wins), tie→hard,
     and zero→error.
   - Add golden fixtures beside `fixtures/claim_request.json` for the join request and
     result, the release request and result, and a starter-bearing begin request, pinned
     like the existing hand-off fixtures (`store/tests/handoff.rs`).
7. **Bindings.**
   - Add `tool_run_join`, `tool_run_release_join`, and `tool_run_sync_wait_budget` in
     `crates/sase_core_py/src/telemetry/`, modelled on `tool_run_claim` and
     `tool_run_duration_fit`.
   - Register them in `register_telemetry`.
   - Extend `tool_run_bindings_round_trip_python_dicts`.
8. **Verify.** Run `sase tool run check` inside the checkout; it takes about 5 minutes.
   Use a Conventional Commit subject such as
   `feat(tool-run): add detached starter scope, monitor join, and sync wait budget`.
   This is not `feat!`: released sase never sends the new fields, and result wires are
   lenient.

## Phase `soft-ceiling` (sase)

This phase is independent of the core and runs in parallel with it. Model it on the
`provider-ceiling` phase of `plan:202609/tool_inline_routing.md`.

1. **Config.** Add to `src/sase/default_config.yml` under `tool_runs:`:

   ```yaml
   soft_ceiling:
     default: ""
     providers: {}
   ```

   - Values are durations such as `90s`, `20m`, or `1h`, keyed by registered provider
     name. Empty means none.
   - Comment it: it is the most time an agent should block on one `sase tool run` before
     escalating to a monitor; it kills nothing; the effective budget is the smaller of
     this and the hard ceiling's budget.
   - Add a settings getter in the style of the existing `tool_runs` getters.
   - A malformed or non-positive value is ignored with one warning. It never fails a
     launch.

2. **Export.**
   - Add `SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS_ENV` to `src/sase/env_contracts.py`.
   - In `invoke_agent` (`src/sase/llm_provider/_invoke.py`), set the variable to the
     execution provider's resolved soft ceiling in whole seconds, or pop it. Do this
     beside the hard ceiling: resolve the provider entry first, then `default`.
   - Restore the previous value in the same `finally`, exactly like the hard ceiling.
3. **Scrub.** Add an exact-key pop to `scrub_agent_identity_env`
   (`src/sase/agent/env_hygiene.py`). Never scrub by prefix.
4. **Tests.**
   - Resolution order: the provider entry, then default, then none.
   - Malformed values are ignored.
   - Set during `provider.invoke`; restored or popped afterwards, including on error; an
     inherited value is popped when none is configured.
   - Scrub coverage in `tests/test_agent_env_hygiene.py`.
   - The monitor child env lacks the variable
     (`tests/monitor/test_monitor_start_supervisor.py`).
5. **Docs.**
   - `docs/configuration.md` (`tool_runs`).
   - A new row in `docs/tool.md`'s environment-contract table.
   - A sentence in `docs/llms.md`'s provider-ceiling text.

Nothing consumes the value yet, so this phase changes no behaviour and needs no flag.

## Phase `detach-run` (sase)

1. **Pin.**
   - Confirm the `core-detach-join` commit is on sase-core's remote master.
   - Run `just ratchet-core-revision`, then `just install`.
   - Confirm the "Check pinned core bindings" lint (`tools/check_sase_core_rs_bindings`)
     passes.
   - If the core commit is not pushed yet, stop and record why on this phase's bead.
     Never bypass binding validation.
2. **Flag.** Create `tool_run_escalation` with
   `sase flag new tool_run_escalation -k beta` and the three sentences below, then paste
   the printed registry entry.
   - `--when-enabled`: "An agent's `sase tool run` under a provider ceiling starts the
     run detached, waits within the ceiling budget, and escalates to
     `sase monitor start -J` instead of being killed; `--detach`, `-J/--join`, and
     ceiling-bounded `sase tool wait`/`show -F` are available."
   - `--when-disabled`: "Agent `sase tool run` stays a plain inline run, `--detach` and
     `-J/--join` are refused as not enabled, and `sase tool wait`/`show -F` ignore the
     provider ceiling."
   - `--remove-when`: "The sase-17g epic's final phase lands; it deletes the Off branch
     and closes this bead."
3. **Facades.**
   - Add `tool_run_join`, `tool_run_release_join`, and `tool_run_sync_wait_budget` to
     `src/sase/core/tool_run.py` through `require_rust_binding`, and export them in
     `__all__`.
   - Do not restate the budget formula in Python.
4. **Starter resolver.** Add a new module, for example `src/sase/tool/starter.py`.
   - `resolve_starter(env)` returns `{agent, pid, boot_id, process_start_identity}` or a
     typed reason. It uses `SASE_AGENT`, `SASE_AGENT_NAME`, and the runner pid in
     `$SASE_ARTIFACTS_DIR/agent_meta.json` (the pid `kill_agent_runner_group` reads),
     with the identity from `process_identity_token`.
   - `starter_alive(starter)` checks the same boot, a live pid, and a matching start
     identity. PID reuse is never proof of life.
   - Confirm the agent_meta pid is the process that runs `invoke_agent`. If it is not,
     key step 8's cleanup on the agent_meta identity, never on `os.getpid()`.
5. **One launcher.**
   - Extract the proc-submission half of `execute_handoff`
     (`src/sase/tool/handoff_launch.py`) into a shared function. `-H`, `--detach`, and
     later the automatic path all call it.
   - `-H` output and behaviour stay **byte-identical**; `tests/tool/test_handoff.py`
     pins them.
   - `reserve_handoff_run` (`src/sase/tool/handoff.py`) gains `starter=` and
     `continuation_mode=` (written into the envelope). Detached procs keep
     `origin: tool-run` and add a `tool-run-detached` tag.
6. **`-d/--detach`.** Add it in `src/sase/main/parser_tool.py`, mutually exclusive with
   `-H`.
   - It is agent-only: outside an agent it is refused with the `-H` form, and inside a
     live owner or a parent run it is refused like `-H`.
   - `-v` and `-T` are usage errors. `-k` and `-x` are allowed and travel as
     `continuation_mode`.
   - It skips the duration-class refusal.
   - It is fail-closed (contract rule 4). If the starter cannot be resolved, exit `1`
     with "cannot identify the starting agent runner; nothing was started".
   - Output mirrors `-H` and adds two lines:
     - `detached: stopped when this agent's turn ends unless a monitor joins it`
     - `join with: sase monitor start -J <id> -p verify -n '<...>'`
   - `-q` prints only the id.
   - Update the `run` help text, and run `just sync-completion-spec` if
     `tests/completion/test_snapshot.py` drifts.
7. **Worker.** In `src/sase/tool/adopt.py`, a claimed run whose `run` carries `starter`
   starts a daemon watchdog thread that polls about every 5 s.
   - While `starter_alive` holds, it keeps going.
   - Once the starter is gone, the run stays alive only if it has a join whose monitor
     is still active. Classify the monitor the way `sase.tool.owner.observe_owner_fact`
     does: monitor rows share the proc id and are read from the proc store. Factor a
     small `(kind, id)` helper out of it rather than restating the rules.
   - Otherwise it records a stop request with `requested_by: sase` and a reason naming
     the case: "starter agent X ended without joining" or "joining monitor Y ended". It
     then stops its own proc through the same owner path `sase tool stop` uses.
   - Test that a worker which stops itself settles `signaled` / `stop_requested` with
     the reason recorded.
   - Also: when the envelope carries `continuation_mode`, it overrides
     `agent_default_continuation_mode`.
8. **Eager cleanup.**
   - In `invoke_agent`'s `finally`, next to the ceiling restores, call a best-effort
     `stop_unjoined_detached_runs(...)` that never raises. Guard it with an agent
     context (`artifacts_dir` and an agent name).
   - Find candidates with `tool_run_briefs(states=[created, running], agents=[name])`,
     then read each with `tool_run_show`. Candidates are runs whose `starter` matches
     this runner and whose join is absent or names a monitor that is no longer active.
   - Stop each one through a public helper extracted from
     `src/sase/tool/control_stop.py` (request stop plus owner routing), which
     `handle_stop` also uses.
   - A monitor handoff records its join before it kills the runner (phase
     `monitor-join`), so a joined run is never touched here.
9. **Notifications.** In `src/sase/tool/notify.py`, `_eligible` excludes any run with a
   `starter`.
10. **`sase tool show`.**
    - Human output adds "detached by agent X (stopped when that agent's turn ends unless
      joined)" and "joined by monitor Y at T".
    - JSON already carries `starter` and `join` through the core envelope.
11. **Tests** go in a new `tests/tool/test_detach.py` plus extensions of
    `test_handoff.py` and `test_lifecycle_controls.py`:
    - flag off refuses;
    - the non-agent, live-owner, parent-run, `-H` + `-d`, `-v`, and `-T` refusals;
    - an unresolvable starter exits 1 with no row written;
    - a `long` tool is accepted detached;
    - the envelope carries `-k`/`-x`;
    - the watchdog stops an unjoined run when a fake starter process exits, and keeps a
      run whose join names an active monitor;
    - the cleanup stops only matching unjoined runs;
    - no notification for detached runs;
    - `-H` output is unchanged.
12. **Harness.** Add hermetic `tools/smoke_sase_tool_runs` cases for detach plus
    starter-death, with the fake starter as a sacrificial `sleep` process named in a
    temporary `agent_meta.json`. Keep `tests/test_sase_tool_runs_smoke.py` in step.
13. **Docs.** Add a `docs/tool.md` subsection, "Detached runs", under "Hand-off and
    lifecycle control". Cover the starter scope, the watchdog and cleanup, the recording
    leg, and the flag.

## Phase `bounded-wait` (sase)

1. **`follow_run`.**
   - Extract the poll, stream, and stage-line loop of `handle_follow`
     (`src/sase/tool/query_follow.py`) into a reusable helper, for example
     `src/sase/tool/follow_run.py`. It takes:
     - a deadline;
     - an output-streaming switch;
     - a stage-line sink;
     - a stop event, so a caller's signal handler can end the loop.
   - It returns a typed outcome: settled with the final envelope, deadline, or stopped.
   - `show -F` moves onto it with output **byte-identical**; the existing query tests
     pin it. `wait_for_settlement` stays the single poll-and-reconcile primitive
     underneath.
2. **Budget adapter.**
   - `sync_wait_budget(env)` in `src/sase/tool/routing.py` reads both ceiling variables,
     using the existing `read_sync_ceiling` rule for each.
   - It calls `tool_run_sync_wait_budget`, and returns `None` without `SASE_AGENT`, with
     the flag off, or when the core reports no budget.
   - A missing or raising binding returns `None` with one warning, so it fails open to
     today's unbounded behaviour.
3. **`sase tool wait`** (`src/sase/tool/control_wait.py`):
   - In an agent with a budget, the effective deadline is the smaller of `-t` and the
     budget. When `-t` is clamped, one stderr line says so.
   - At the deadline it prints the escalation block when the run is joinable (see
     Terms), else the existing "still running" line.
   - It exits `124`, as today.
   - `-j` adds an `escalation` object (`budget_seconds`, `source`, `joinable`,
     `join_command`) at the deadline. `schema_version` stays 1.
4. **`sase tool show -F`.** In an agent with a budget, it stops following at the budget,
   prints the same block, and exits `124`. Document that exit in its help.
5. **Escalation block builder.** Build it per the contract, including elapsed time
   measured from the run's `running_ts`, or `created_ts` while it is still `created`.
6. **Tests.**
   - Clamping, with and without `-t`.
   - Soft versus hard wording.
   - Joinable versus non-joinable blocks.
   - The JSON escalation object.
   - Flag off keeps the old unbounded behaviour.
   - Humans are never bounded.
   - `show -F` parity without a budget.
7. **Docs.** Cover the budget formula, `124`, and the block in `docs/tool.md`.

## Phase `monitor-join` (sase)

1. **CLI.**
   - Add `-J/--join RUN` to `sase monitor start` (`src/sase/main/parser_monitor.py`,
     handler `src/sase/main/monitor/start.py`).
   - It conflicts with a command remainder, `-c`, `-f`, and `-a`; each conflict is a
     usage error, exit `2`. For `-f`, name the follow-up and suggest `-n`.
   - It requires an agent caller.
   - Before any slow work or in-flight marker, pre-validate joinability with
     `tool_run_show`, using the exit codes in the contract.
   - Keep options sorted. Add an example to the epilog, and run
     `just sync-completion-spec` if the snapshot drifts.
2. **Start flow.** Add a `join_run_id` field to `StartMonitorRequest`
   (`src/sase/monitor/request.py`) and handle it in `start_monitor`
   (`src/sase/monitor/start.py`):
   - **Command.** `command` is synthesized as the run's canonical `sase tool run ...`
     words, the way `routing._run_words` does it (ad-hoc runs use `-- <argv>`). The
     label defaults to `tool:<name> (joined)`, and the reason to "finish <tool> (joined
     run)".
   - **Skipped steps.** Skip `resolve_monitor_tool_wrap` and
     `maybe_reserve_monitor_tool_run`.
   - **Join.** After member creation and before `submit_proc_request`, call
     `tool_run_join` with kind `monitor`, the monitor id, and the caller's agent.
     - A refusal tears the member down the way other pre-submit failures do.
     - A `ProcSubmitError` calls `tool_run_release_join`, beside the existing rollbacks.
   - **Meta.** Set `monitor_tool_run_id` and a new `monitor_tool_run_joined: true` meta
     field. The proc argv is the joiner worker
     (`[sys.executable, -m, sase, tool, _join, RUN]`).
   - **Everything else** is the normal path: lane, workspace claim transfer, pending
     marker, and runner kill.
   - The `--json` envelope gains `tool_run_joined`.
3. **Joiner worker.** Add a hidden `sase tool _join RUN` subparser, internal like
   `_adopt`, and a module such as `src/sase/tool/join_worker.py`.
   - It re-asserts the join as an idempotent replay with its own `SASE_MONITOR_ID`. A
     refusal exits `2`.
   - It calls `follow_run` with no deadline, streaming the output of record **from
     offset 0** plus stage lines into the monitor log.
   - On settlement it prints a one-line summary and the triage footer lines
     (`footer_triage_lines`) so the follow-up's tail carries the verdict. It exits with
     the run's code, mapped as in the contract.
   - On SIGTERM or SIGINT it records a stop request. Use reason "joining monitor Y
     stopped", or "timed out" when the proc's termination intent is a timeout. It then
     stops the run through its owner, waits at most 15 s for settlement, and exits `143`
     or `130`.
4. **Stop routing.**
   - In `_route_owner_stop` (`src/sase/tool/control_stop.py`), a run joined by an active
     monitor stops **that monitor** (`stop_monitor`, which suppresses the follow-up).
     The joiner then stops the run.
   - If the joining monitor is already terminal, fall back to the proc-owner path.
5. **Settlement.** In `src/sase/monitor/proc_adapter.py`, skip `settle_monitor_tool_run`
   for a joined run: its owner fact names the proc, not the monitor. The follow-up
   prompt already carries `tool_run_id` (`followup.py`). `sase monitor show` and its
   JSON label the run as joined.
6. **Tests.**
   - Every refusal and exit code.
   - Join recorded before submit; release on a submit failure.
   - A settled-between-validate-and-join race exits `1` with no monitor.
   - The joiner:
     - mirrors success and failure codes;
     - streams pre-join output;
     - on stop and on timeout, stops the run, which settles `stop_requested` with the
       reason.
   - `sase tool stop` on a joined run stops the monitor with no follow-up.
   - A lost joiner lets the watchdog stop the run.
   - Flag off refuses `-J`.
   - Unit-test the start flow with the existing monitor-start fixtures; the runner-kill
     handoff stays mocked as in today's tests.
7. **Harness and docs.**
   - Add a hermetic smoke case: detach, join as a non-handoff test caller, settle, and
     check the mirrored exit code.
   - Document `-J` in `docs/monitors.md` (tool-run section) and `docs/tool.md`.
   - Add a `PROPOSED FOLLOW-UP:` note for `-f` with `--join`. It needs the joiner to
     mirror stage diagnostics into `SASE_MONITOR_DIAGNOSTICS_DIR`, because the prepared
     `pass` gate checks `required_stages` (`continuation/completion_eval.rs`). It also
     needs a guard that the workspace fingerprint at join time equals the run's
     `fingerprint_before`.

## Phase `inline-escalation` (sase)

1. **Engagement.** In `execute_tool_run` (`src/sase/tool/executor.py`), after the
   duration-class refusal and before today's inline path, take the escalating path only
   when all of these hold:
   - the flag is on;
   - not `-H` or `--detach`;
   - `SASE_AGENT` is set;
   - `resolve_ownership()` reports no owner and no parent run;
   - `sync_wait_budget()` returns a budget.

   Otherwise nothing changes. That keeps humans, CI, monitors, procs, nested runs, and
   providers with neither ceiling (Codex, Grok, and others unless a soft ceiling is
   configured) inline.

2. **Start.**
   - Reconcile with `reap_orphans=True` as today.
   - Resolve the starter, then reserve and launch through the shared launcher, carrying
     `continuation_mode` from `-k`/`-x` or the agent default.
   - Any failure warns once
     (`sase: inline escalation unavailable (<why>); running inline`) and runs today's
     inline body. A reservation whose launch failed is settled `launch_failed` first, as
     `-H` does.
3. **Follow.** Print the same first line, `sase tool run <id>`; LLM Calls rows and
   agents key on it. Then call `follow_run` with the budget as its deadline.
   - Stage lines match inline formatting.
   - `-v` streams the output of record.
   - Compact mode (the agent default, or `-q`) streams nothing.
   - Install SIGTERM, SIGINT, and SIGHUP handlers that end the follow without stopping
     the run.
4. **Settled.**
   - Wait, bounded to about 10 s, for the owner proc to reach a terminal status, so the
     worker's triage and receipt have settled.
   - Render a footer from the ledger that matches `write_run_footer`'s compact and
     streaming forms: state and exit, duration, the unattributed line, the `-T` tail of
     the output of record on failure, truncation, the triage lines and verdict, and the
     `sase tool show <id> -l` pointer.
   - Share that renderer in `src/sase/tool/executor_display.py`. Do not duplicate triage
     formatting.
   - Return the mapped exit code.
5. **Budget or signal.** Print the escalation block and exit `124`, `143`, `130`, or
   `129`. The run continues; the watchdog or cleanup stops it at turn end unless it is
   joined.
6. **Environment parity.** The detached child's environment differs from an inline
   child's only by the scrubbed agent identity and owner markers. Confirm that no
   catalog recipe behaves differently: guarded recipes, visual-snapshot update gating
   keyed on `SASE_AGENT`/`SASE_MONITOR_ID`, and `CI`. Record the finding in a bead note.
   stdin is no longer inherited; say so in help and docs.
7. **Help.** Update the `sase tool run -h` description: one paragraph on
   inline-then-escalate, plus the new exit codes.
8. **Tests.** Put them in a new `tests/tool/test_inline_escalation.py`. Use the
   fixture-catalog pattern from `tests/tool/test_executor.py` and a soft ceiling of a
   couple of seconds, so the budget is tiny and real:
   - Fast success and failure: the exit codes and compact footer match an inline run of
     the same fixture, including the tail and the show pointer.
   - `-v` streams output.
   - A slow tool exits `124` with the block while the run keeps going under the same id,
     and `sase tool wait` later returns its code with no second row.
   - SIGTERM to the follower leaves the run running.
   - `-k`/`-x` reach the worker.
   - Launch failure falls back inline with one warning.
   - Flag off, no budget, no agent, and nested runs stay inline.
   - A `long` tool under a 600 s ceiling is still refused before anything starts.
9. **Harness.** Add a hermetic smoke case: an escalating run, then a later `wait`
   returns the same run.

## Phase `guidance-and-flag-removal` (sase)

1. **Flag removal.**
   - Delete every `tool_run_escalation` Off branch and its tests, and make the On branch
     unconditional.
   - Remove the registry entry and close the flag bead in the same change (see
     `sase_flags.md`).
   - `tools/check_feature_flags` must pass.
2. **Muse directive** (`_muse_single_turn_directive` in
   `src/sase/llm_provider/muse.py`):
   - Keep the synchronous-tool sentence and "never cancel or rerun an in-flight
     command".
   - Replace the known-long routing and `timeout 540` advice for `sase tool run` with
     this: `sase tool run` returns before your ceiling on its own; never wrap it in
     `timeout`. If it prints an escalation block, run the printed
     `sase monitor start -J ...` command next.
   - Keep the `timeout` wrapper advice only for commands that are not `sase tool run`.
   - Update the tests that pin the directive text.
3. **Skill sources** (do not deploy from this phase):
   - `src/sase/xprompts/skills/sase_monitor.md`: in "Decide Before You Start", say that
     `sase tool run` handles the ceiling itself (start inline, and join the printed run
     with `-J` when it escalates), and add a `-J` example to "Canonical Invocation".
   - `src/sase/xprompts/skills/sase_final.md`: add a note that an escalated check is
     finished with `-J ... -n`, not by rerunning it under a prepared `-f` monitor.
   - Keep provider numbers out of skills.
4. **Docs.**
   - `docs/tool.md`: rework "Inline routing" and "Hand-off and lifecycle control" into
     one "Inline-then-escalate" story. The refusal stays for `long`/`unbounded`.
   - `docs/monitors.md`: the join.
   - `docs/llms.md`: the Muse section and the soft ceiling.
   - `docs/configuration.md`: remove any "beta flag" wording.
5. **Live harness case.** Add a `tools/smoke_sase_tool_runs --live` case: a detached run
   from a simulated agent, a real `sase monitor start -J` whose runner kill targets a
   sacrificial process group, the mirrored exit code, and the one run id. Run `--live`
   once and record the report path in a bead note.
6. **Tests.** Everything touched stays green. In particular, the completion snapshot,
   the flag lint, and the directive-text tests.

## Landing

1. Confirm each phase's evidence, including the environment-parity note and the `--live`
   report. Run `just fix`, then `sase tool run check`. The lander's own check now runs
   through the automatic path whenever its provider has a budget.
2. **Cheap live smoke.** Use a temporary git project with a fixture catalog whose `slow`
   tool is `[sh, -c, "sleep 20; echo done"]`, from a shell that sets `SASE_AGENT`,
   `SASE_AGENT_NAME`, and `SASE_ARTIFACTS_DIR` to a temp dir. Its `agent_meta.json`
   names a sacrificial `sleep 600` pid. Never use this repo's catalog.
   - With `SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS=5`, `sase tool run slow` exits `124`
     at about 5 s and prints the block.
   - `sase tool show <id>` shows it running, detached by that agent.
   - `sase tool wait <id>` returns `0`, and `sase tool runs -a -t slow` shows exactly
     one run.
   - Repeat, then kill the sacrificial pid: the run settles `signaled` /
     `stop_requested` within about 10 s, with the starter reason.
   - Record
     `echo "$SASE_PROVIDER_SYNC_CEILING_SECONDS" "$SASE_PROVIDER_SYNC_SOFT_CEILING_SECONDS"`
     from the lander's own shell.
3. **Close the bead.** Run
   `sase bead close sase-17g --note "<phases landed; smoke evidence; what stays out of scope>"`.
4. **Deploy the skills.** From the clean landed tree, run `sase skill init --force`,
   then `chezmoi apply` if it was skipped (see `generated_skills.md`).
5. **Follow-ups.** File the phases' `PROPOSED FOLLOW-UP:` notes through `/sase_new_task`
   where they are warranted. Expected ones:
   - `-f` with `--join` (stage-diagnostics mirroring and the fingerprint guard).
   - TUI surfaces: a `⚒` chip on the joining monitor's row, and detached/joined badges
     in the Runs card and the Tools pane.
   - A `memory` task bead for `lint_and_test.md` and `glossary:tool-run`. Both still say
     agents hand off only through `sase monitor start`, and describe check routing
     without `-J` escalation. The bead should name the exact replacement sentences.
   - Whether to ship default soft ceilings for Codex and Grok, decided from `tool_runs`
     evidence.

## How to tell it worked (post-landing observation, not a landing gate)

These are the research's acceptance criteria, measured over one athena week with
read-only `runs.sqlite` queries (see the research's §8):

- No `check` run is signaled at 530–550 s or 600 s.
- There are no same-agent kill→rerun pairs.
- Fewer than 5% of monitor-owned runs finish in under 2 minutes.
- Every escalated run shows one run id from start to join to settlement.
- Runs stopped with the starter-ended reason stay rare. Many would mean agents escalate
  but don't join, which is a guidance gap.

## Non-goals

- Forecasts, `run -E`, `sase tool stats`, ETAs, and `timeout: auto`: the parked
  remainder of E6.
- E7 and E8.
- Receipt reuse (E4b).
- TUI changes.
- `-f` prepared completion with `--join`.
- Joining `-H` or monitor-owned runs.
- Changing the hard-ceiling contract or the `sase-17e` refusal.
- Default soft-ceiling values.
- **No memory edits.** Phases record stale memory as `PROPOSED FOLLOW-UP:` notes, and
  landing files the `memory` task bead.

## Verification (every phase)

- **sase phases:** run `just fix`, then `sase tool run check`. Run `just install` first
  if the workspace venv is stale.
- **The sase-core phase:** run `sase tool run check` in that checkout.
- Do not run `check-full` (`decisions:check-full-is-explicit`).
- Phase workers never create beads, except the flag bead that `sase flag new` creates in
  `detach-run`. They append `PROPOSED FOLLOW-UP:` notes to their own phase bead.
