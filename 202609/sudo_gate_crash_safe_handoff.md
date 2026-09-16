---
tier: epic
title: Crash-safe sudo/gate handoff and codex turn-integrity detection
goal: 'An in-agent gate creation that dies mid-flight can never end as a silent SUCCESS
  again: the host fails the run loudly, the codex provider detects aborted turns,
  and the sudo skill prevents the yield-and-abandon pattern.

  '
phases:
- id: gate-intent-adjudication
  title: Gate-creation intent marker and host adjudication
  depends_on: []
  size: large
  description: 'gate-intent-adjudication: write a durable intent marker at the top
    of the shared agent-side gate creation path and make the host runner lifecycle
    fail the run with a needs-attention notification when an intent is left unconsumed.'
- id: codex-turn-integrity
  title: Codex provider turn-integrity detection
  depends_on: []
  size: medium
  description: 'codex-turn-integrity: treat a codex turn that ends with an empty final
    answer plus a command killed at teardown as a provider failure that raises for
    retry instead of returning success.'
- id: sudo-skill-foreground-guidance
  title: Sudo skill foreground-execution guidance
  depends_on: []
  size: small
  description: 'sudo-skill-foreground-guidance: update the generated sase_sudo skill
    template to require foreground execution of sase sudo request and to state that
    any returned tool result means the turn did not end.'
- id: sudo-e2e-verification
  title: End-to-end verification of the sudo handoff
  depends_on:
  - gate-intent-adjudication
  - codex-turn-integrity
  - sudo-skill-foreground-guidance
  size: small
  description: 'sudo-e2e-verification: re-run the /root/foobar sudo request to confirm
    gate, bundle, and notification creation, and simulate a killed CLI to confirm
    the run now fails loudly.'
proposed_by: bbugyi200.athena.0lw.r0.f0
create_time: 2026-09-16 10:41:57
status: wip
bead_id: sase-11t
---

- **PROMPT:** [prompts/202609/sudo_gate_crash_safe_handoff.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/sudo_gate_crash_safe_handoff.md)
- **BEAD:** [sase-11t](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11t/README.md)

# Plan: Crash-Safe Sudo/Gate Handoff and Codex Turn-Integrity Detection

## Problem

On 2026-09-16 agent `0lw.r0` (chat `gh_sase_org__sase-ace_run-0lw_r0-260916_100310`,
model codex/gpt-5.6-sol) ran `sase sudo request` to test `/sase_sudo` (removing
`/root/foobar`). The run finished with `outcome: completed` / SUCCESS, but **no sudo
gate, no bundle, and no notification were ever created**. The user waited for a sudo
gate notification that never arrived, and nothing anywhere reported a failure.

## Verified root cause

Evidence from the codex rollout
(`~/.codex/sessions/2026/09/16/rollout-2026-09-16T10-04-07-01a0aa88-*.jsonl`), the run's
`tool_calls.jsonl`, and the workflow output log
(`~/.sase/workflows/202609/gh_sase-org__sase_ace-run-260916_100310.txt`):

1. The codex model invoked `sase sudo request` through codex's unified exec tool with
   `yield_time_ms: 10000`.
2. After 10.2s the exec tool yielded back `"Script completed / Output:"` with **empty
   output** while the `sase` CLI was still running (cold start plus gate-shell creation
   takes longer than 10s).
3. The model — primed by the skill text "the command creates a reviewed sudo gate and
   intentionally ends your current turn" — concluded the handoff was underway and
   emitted an **empty final answer** (`last_agent_message: null`).
4. Codex task teardown then killed the still-running exec session: the rollout records
   the `sase sudo request` CommandExecution item with
   `status: failed, exit_code: -1, duration: 17.0s`. The CLI died **before its first
   durable write**: no gate-shell member directory exists under the project's
   `artifacts/ace-run/202609/16/` for that window, and
   `~/.sase/interaction_requests/sudo/` was last modified 2026-09-15 (no bundle, no
   store record, no notification).
5. `codex exec` exited 0. The codex provider treats a zero exit plus captured text as
   success (`invoke()` in `src/sase/llm_provider/codex.py` only fails on
   `return_code != 0`), so the host published the streamed commentary as a normal reply,
   ran the finalizers, and wrote `done.json` with SUCCESS.

Ruled out during diagnosis:

- The Sep 11 gate-handoff ImportError (commit `71717d95a0`) was already fixed by
  `b9c28f25e` and live via the editable install when the retry ran (the installed
  checkout fast-forwarded at 10:02; the retry ran at 10:04). That ImportError is what
  broke the _first_ test run (`0lw`), not this one.
- The `agent_sudo_requests` feature flag is enabled.
- The restored `gate_shell/agent_handoff.py` path works: agent `0lx--plan` created an
  `epic_plan` gate through `create_gate_shell` successfully at 10:05:39, one minute
  after this failure.

Incidental (not causal, filed separately as a task bead): SASE's shadow `CODEX_HOME`
symlinks the shared `~/.codex/state_5.sqlite` and `sessions/` into every disposable
per-run home, so the shared state db accumulates rollout paths under deleted shadow-home
prefixes ("stale rollout path" ERROR spam at startup, "failed to record rollout items:
thread not found" at teardown).

## Why this needs fixing beyond this one run

Any in-agent gate creation (`sase sudo request`, `sase gate`, LaunchApproval, workflow
HITL) has zero durable footprint until `create_gate_shell` performs its first write. If
the runner or its harness kills the CLI before that point, the turn ends silently: the
host has no signal that a gate was intended, and the run is published as a success. This
will recur for any gate-creating command on any provider whose exec tool can return
before the command exits.

## Design

Two independent defenses plus a guidance fix, then end-to-end verification. All changes
are Python host/orchestration behavior in this repo; nothing crosses the Rust core
backend boundary (no other frontend needs to share it).

## Gate-creation intent marker and host adjudication

Make in-agent gate creation crash-safe:

- Write the intent marker at the top of the shared agent-side creation path so every
  gate kind is covered once: `_request` in `src/sase/sudo/cli.py` enters through
  `create_gate_shell` (`src/sase/gate_shell/transaction.py`) and hands off via
  `src/sase/gate_shell/agent_handoff.py`. The marker (kind, request id, timestamp)
  belongs in the creator's `$SASE_ARTIFACTS_DIR`, written where creator resolution
  already happens; clear or supersede it on all clean exits (successful
  `.sase_gate_pending` handoff marker write, auto-resolved gates, clean `GateError`
  reported to the caller, and the existing `_compensate_failed_handoff` path).
- Host side: where the runner lifecycle classifies the finished turn and checks
  `has_pending_handoff` (`src/sase/agent/pending_handoff.py`,
  `src/sase/axe/run_agent_runner_signals.py`,
  `src/sase/axe/run_agent_runner_lifecycle.py`, `src/sase/axe/run_agent_exec_gate.py`),
  detect an unconsumed intent marker: fail the run (not SUCCESS), surface the reason in
  the workflow output and `done.json`, and emit a needs-attention notification that
  names the lost gate kind and request id.
- Regression tests for: killed-CLI-before-first-write (intent present, nothing else),
  normal handoff (intent superseded), clean CLI error (intent cleared, run continues
  normally), and auto-resolved gates.

## Codex provider turn-integrity detection

In `src/sase/llm_provider/codex.py` (and its `--json` stream parser,
`stream_and_parse_codex_json_output`): when the stream shows the task completed with an
empty/absent final agent message and a command item that was killed at teardown
(`status: failed`, `exit_code: -1` recorded after `task_complete`, or equivalently a
recorded `ToolUse` with no `ToolResult`), raise instead of returning success, so the
workflow retry subsystem treats the invocation as failed. Preserve current behavior for:
nonzero exit (already raises), legitimate handoffs (the runner process group is
SIGTERMed by `kill_agent_runner_group`, so `invoke()` never returns through the success
path), and ordinary turns with a non-empty final message. Add tests with captured stream
fixtures matching the aborted-turn shape from run `ace-run/202609/16/20260916100310`.

## Sudo skill foreground-execution guidance

Update the sase_sudo skill source template under `src/sase/xprompts/skills/` (read
`sase/memory/generated_skills.md` first; the deployed copies are generated and managed):
instruct agents to run `sase sudo request` in the foreground and wait for the process to
exit; never run it with a yield/background window or early-return timeout; and state
explicitly that if the tool call returns any result at all (JSON descriptor or error),
the turn has NOT ended and the agent must report that result instead of stopping. Apply
the same wording to any other skill/xprompt that documents a turn-ending gate command if
the pattern exists there. Regenerate and deploy the managed skill outputs per the
generated-skills procedure.

## End-to-end verification of the sudo handoff

With the other three phases landed and the editable install updated:

- Re-run the original test: launch an agent that submits the `/root/foobar` removal
  through `/sase_sudo`, and confirm a sudo gate shell record, a bundle under
  `~/.sase/interaction_requests/sudo/`, and a notification are created (approval/denial
  stays with Bryan; creation is the success signal).
- Simulate the failure mode: kill the `sase sudo request` process mid-request and
  confirm the run is now reported failed with a needs-attention notification instead of
  silent SUCCESS.
- Confirm a normal non-gate agent run still completes SUCCESS with no spurious
  intent-marker failures.
