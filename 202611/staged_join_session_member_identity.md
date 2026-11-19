---
tier: tale
title: Staged -J joins accept agent-session members
goal:
  A session member (for example a --code successor) can hand its escalated sase tool run
  to a monitor with sase monitor start -J, so phase agents stop stalling their epics on
  agent_mismatch refusals.
size: small
decisions:
  refusal_recovery:
    ask:
      Should a refused staged join also tell the agent how to keep its verification
      going?
    choices:
      hint:
        Refusals print a re-wait and a fresh-monitor rerun; the monitor skill gets one
        sentence
      skip: Only the identity bug is fixed; refusal text and skill source stay unchanged
    default: hint
    why:
      sase-1tc.4 retried the identical refused join twice, then ended with keep and
      stalled its epic
    answer: hint
proposed_by: bbugyi200.athena.1ec
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 07:20:03
status: wip
---

# Staged `-J` joins refuse agent-session members (`agent_mismatch`)

## Problem

Session members (handoff successors such as every `--code` turn that follows a `--plan`
turn) cannot hand an escalated `sase tool run` to a monitor. `sase monitor start -J RUN`
fails with:

```
sase monitor start: tool run RUN cannot be reserved for staged join (agent_mismatch); no monitor started
```

The run's turn-end cleanup then stops it. If the agent ends its turn with
`bead_action: keep`, nothing resumes the phase and its epic stalls.

### Observed stall (epic `sase-1tc`, phase `sase-1tc.4`)

- The coder turn's `agent_meta.json` has `name: sase-1tc.4--code`,
  `agent_session: sase-1tc.4`, and `role_suffix: --code`. It ran in the same runner
  process as its predecessor `sase-1tc.4--plan`, and its shell had
  `SASE_AGENT_NAME=sase-1tc.4` (the session container).
- `sase tool run check` escalated at 8m29s as run `3ead2af4b580392e4aa10ba628693c74`.
  The run's recorded `starter.agent` is `sase-1tc.4`, because `resolve_starter()` in
  `src/sase/tool/starter.py` reads `SASE_AGENT_NAME`.
- `sase monitor start -J 3ead2af4…` passed the CLI ownership pre-check, because
  `inspect_join_target(run_id, join_caller_agent())` in `src/sase/monitor/join.py` also
  reads `SASE_AGENT_NAME`. The staged branch of `_reserve_join_or_tool_run()` in
  `src/sase/monitor/start_prepare.py` then called
  `_reserve_staged_join(..., agent=lane_start.join_starter_agent)`.
  `resolve_lane_start()` in `src/sase/monitor/start_lane.py` sets `join_starter_agent`
  from the lane artifact's `agent_meta.json` `name`, which here is `sase-1tc.4--code`.
  The Rust function `reserve_staged_join` (sase-core
  `crates/sase_core/src/tool_run/store/handoff.rs`) refuses when
  `starter.agent != agent`, so it returned `AgentMismatch`. Two identical retries failed
  the same way.
- The agent submitted `bead_action: keep`. The end-of-invocation cleanup
  (`stop_unjoined_detached_runs`, which also keys on `SASE_AGENT_NAME`) stopped the run
  with the reason "starter agent sase-1tc.4 ended without joining". The phase bead
  stayed `IN_PROGRESS` with no live agent, so `sase-1tc.5` and `sase-1tc.land` wait
  indefinitely.

### Why this exists

Commit `feae9c8777` (`fix(monitor): preserve staged adoption context`, bead
`sase-1pj.5.1`) introduced `join_starter_agent` as the **pre-promotion** `agent_meta`
name. That fixed a bare starter promoted from `acme` to `acme--0` mid-start. The fix
assumes that, before promotion, the meta name equals `SASE_AGENT_NAME`. That assumption
is false for any agent that is already a session member. `resolve_local_agent_name()` in
`src/sase/agent/identity.py` documents the behavior: session members replace one another
inside one process, which leaves `SASE_AGENT_NAME` set to the session container while
`agent_meta.json` carries the concrete turn.

Every other ToolRun ownership comparison already uses `SASE_AGENT_NAME`:

- the detach starter (`resolve_starter`),
- the CLI join pre-check (`join_caller_agent`),
- the ordinary non-staged join (`start_launch.py`,
  `join_request["agent"] = SASE_AGENT_NAME`),
- turn-end cleanup (`stop_unjoined_detached_runs`).

The staged reservation is the only outlier. The `sase-1pj.5.1` investigation concluded
that the Rust join-ownership check should not be weakened. This plan keeps that check
strict.

## Approach

Make the staged reservation present the same starter identity that stamped the run and
that the CLI pre-check already verified.

### 1. One starter-identity source

In `src/sase/tool/starter.py`, add one small public helper that returns the ToolRun
starter agent name for an environment. It applies exactly the rules `resolve_starter()`
uses today: the agent must be marked by `SASE_AGENT`, and the name is the stripped
`SASE_AGENT_NAME`; otherwise it returns `None`. Route these callers through it so they
cannot drift:

- `resolve_starter()`, keeping its typed failure reasons (`NOT_AGENT`, `NO_AGENT_NAME`)
  and its behavior unchanged,
- `join_caller_agent()` in `src/sase/monitor/join.py`.

Export the helper in `__all__`. Symvision requires a real non-test consumer, and the
callers above provide one.

### 2. Staged reservation uses that identity

In `resolve_lane_start()` (`src/sase/monitor/start_lane.py`), set `join_starter_agent`
from the helper in step 1 when the caller is an agent. Fall back to the current
pre-promotion `raw_meta["name"]` only when the helper returns `None`. This keeps today's
non-agent behavior, and the bare-starter promotion case still matches because its env
name equals its pre-promotion meta name.

Update the `LaneStart` docstring to say that `join_starter_agent` is the ToolRun starter
identity (`SASE_AGENT_NAME`), not the concrete-turn meta name.

Leave `starter_agent` (claim transfer and `SASE_TOOL_RUN_AGENT` attribution) unchanged.

Keep the existing persistence path: `start_stage_persist.py` writes `join_starter_agent`
into the stage claim, and `_start_stage_adopt_context.py` reads it back, so
adoption-time handling sees the same value.

An equivalent plumbing is acceptable if it is simpler: carry the already-verified
`JoinTarget.caller_agent` from `_validate_join_start()`
(`src/sase/main/monitor/start.py`) through `StartMonitorRequest` into the reservation.
Either way, all four comparisons listed under "Why this exists" must end up reading one
source.

Do not change the Rust `reserve_staged_join` ownership check.

### 3. Refusal recovery (decision `refusal_recovery`)

> [!decision] refusal_recovery = hint When `_reserve_staged_join()` in
> `src/sase/monitor/start_prepare.py` refuses, add recovery guidance to the message. Do
> the same in `_join_refusal_message()` in `src/sase/monitor/start_launch.py`, if it
> renders the same agent-facing refusal. The message should say three things:
>
> - The run is stopped when this turn ends unless a monitor joins it.
> - The agent can wait again inline with the bounded re-wait form. Reuse the existing
>   `_rewait_command` text from `src/sase/tool/routing.py`.
> - The agent can start a fresh monitored rerun:
>   `sase monitor start -p verify -n '<…>' -- sase tool run <words>`. Build `<words>`
>   with `_canonical_join_words(run)` from `src/sase/monitor/join.py`, which handles
>   both named and ad-hoc runs.
>
> The message must never suggest repeating the same `-J`.
>
> Before writing the text, confirm that a fresh
> `sase monitor start -- sase tool run check` is admitted (it stages under the monitor
> tool wrap). If a guard refuses it, name the command that guard prints instead.
>
> Add one sentence to `src/sase/macros/skills/sase_monitor.md`, next to the `-J`
> canonical invocation: if `-J` is refused, do not retry the same join; follow the
> recovery the refusal prints. Edit the source template only. Do not run
> `sase skill init --force` or deploy, because generated skills are deployed only from a
> landed revision.

> [!decision] refusal_recovery = skip Leave the refusal text and the skill source
> unchanged.

## Tests

The staged-start builders in `tests/monitor/_start_stage.py` (`make_starter_agent` and
the starter registration helper) already support `agent_session` and custom starter
names.

1. **Regression (write it first and confirm it fails with `agent_mismatch` on current
   code).** Register a session-member starter with meta `name: acme--code`,
   `agent_session: acme`, and `role_suffix: --code`, and set `SASE_AGENT_NAME=acme`.
   Reserve a detached run whose starter agent is `acme`, the same way as the
   `sase-1pj.5.1` repro (`reserve_detached_run(check, agent=acme)`). A staged `-J` start
   must then:
   - succeed,
   - leave the run's `staged_join_reservation.agent == "acme"`,
   - persist `join_starter_agent == "acme"` in the stage claim.

   Adopt it through the existing adoption test path and assert that the join activates.

2. **Bare-starter promotion stays green.** The existing `acme` → `acme--0` case still
   reserves.
3. **Ownership is not weakened.** A run whose starter is `other` is still refused
   (`inspect_join_target` exits 2 with "belongs to agent"), and a direct reservation
   with a mismatched agent still returns `agent_mismatch`.
4. **Identity parity unit test.** Under one environment, including a session-member
   environment whose `agent_meta` name differs from `SASE_AGENT_NAME`, the helper,
   `resolve_starter(env).starter["agent"]`, and `join_caller_agent()` all agree.
5. **Recovery message (if `refusal_recovery = hint`).** Force a refusal and assert that
   the message contains the re-wait form and a fresh-monitor command built from the
   canonical words, and does not contain `-J`.

Run the focused suites (`tests/monitor/`, plus any tool starter or join tests), then the
full verification gate through `sase tool run check`, following the repo's lint-and-test
memory.

## Out of scope

- Changing what `SASE_AGENT_NAME` means for session members (session container versus
  concrete turn). That change would reach far beyond ToolRun ownership.
- Weakening the Rust `reserve_staged_join` or `tool_run_join` ownership checks.
- Repairing the currently stalled `sase-1tc.4` phase. Do not close, reopen, or annotate
  `sase-1tc.4`, `sase-1tc.5`, or the `sase-1tc` epic bead. That recovery is an operator
  action handled separately.
