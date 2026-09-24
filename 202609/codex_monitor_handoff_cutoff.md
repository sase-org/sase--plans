---
tier: epic
title: Stop Codex monitor handoffs from being silently cut off
goal: "An agent's in-agent handoff (above all `sase monitor start`) either completes or
  fails loudly. Codex turns can no longer end while a handoff command is still running,
  monitor starts are fast and never silent, and a killed handoff is recorded and
  surfaced instead of leaving an assigned bead stuck in_progress.

  "
phases:
  - id: codex-handoff-guard
    title: Codex adapter conformance for handoff commands
    depends_on: []
    size: medium
    description: "codex-handoff-guard: count only the final answer in the Codex
      turn-integrity signal, classify SASE handoff commands, replace the silent pass
      with a bounded continuation that re-drives a stranded handoff, state Codex's real
      exec-yield ceiling in a single-turn directive, and clarify the monitor and final
      skill wording.

      "
  - id: fast-monitor-start
    title: Make in-agent sase monitor start fast and never silent
    depends_on: []
    size: medium
    description: "fast-monitor-start: resolve a pinned caller without a full project
      scan, do one lane-scoped monitor read per start (adding an index filter in
      sase-core if needed), add start timing, and print a stderr line before any slow
      work.

      "
  - id: aborted-handoff-evidence
    title: Host records and surfaces a killed handoff
    depends_on:
      - fast-monitor-start
    size: medium
    description:
      "aborted-handoff-evidence: write an in-flight marker from in-agent handoff
      commands, have the runner adopt a late handoff or record handoff_aborted, write
      accurate recovery evidence, and notify the user when a handoff was killed."
proposed_by: bbugyi200.athena.0re
create_time: 2026-09-24 16:48:51
status: wip
---

- **PROMPT:**
  [prompts/202609/codex_monitor_handoff_cutoff.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/codex_monitor_handoff_cutoff.md)

# Stop Codex monitor handoffs from being silently cut off

## Incident (why this plan exists)

Phase agent `sase-185.2` (Codex, `gpt-5.6-terra`) implemented its phase, then ran:

```
sase monitor start --profile verify --timeout 45m --next '<run epic-symbols, close sase-185.2, ...>' -- sase tool run check
```

It planned to verify, run `sase bead epic-symbols sase-185.2`, and close the bead in the
monitor's follow-up. The bead was never closed. Sequence, taken from the agent's
artifacts and Codex rollout:

1. Codex ran the command through `exec_command` with `yield_time_ms: 30000`. After 30 s
   it returned `Script completed / Wall time 30.2 seconds / Output: (empty)`. The
   process was still running as a unified-exec session.
2. The `sase_monitor` skill says the start "kills the current agent" and that the turn
   "will not return normally". The model read the empty yield as a finished handoff and
   ended its turn with an empty `final_answer`.
3. At `task_complete`, Codex killed the in-flight `sase monitor start`
   (`CommandExecution status=failed exit_code=-1`, duration 37 s). The kill came after
   the start had already promoted the root agent to `sase-185.2--plan`. It came before
   any monitor member, supervisor, or pending handoff marker existed. No monitor exists
   for the lane.
4. The Codex turn-integrity guard in `src/sase/llm_provider/_subprocess_codex.py` did
   not fire. It only fires when there is no non-empty agent message, and the agent's
   earlier _commentary_ messages counted. The runner treated the turn as a normal
   finish. It ran the finalizer declaration-recovery turn, which committed the work
   (`feat(ace): detach accepted agent launch guards`) with `bead_action: keep`.
5. Result: `sase-185.2` stayed `in_progress` with no live agent. `sase-185.3` and
   `sase-185.land` wait on it with no signal to the user.

This is systematic, not a one-off. Every Codex `sase monitor start` on 2026-09-24
(`0r1--code`, `sase-17d.10.1.2--code`, `0r5--code`, `sase-185.2`) failed the same way,
with yields of 10 s or 30 s and empty output. Earlier Codex handoffs (through
2026-09-19) completed inside the yield window.

The start got slow. Timings measured on the host for the read-only lookups an implicit
in-agent start performs:

| Call                                                                                                                             | Records | Time        |
| -------------------------------------------------------------------------------------------------------------------------------- | ------- | ----------- |
| `monitor.store.project_records(project)` (via `store_lane.resolve_caller_agent`, even when `SASE_ARTIFACTS_DIR` pins the caller) | 12,004  | ~10–11 s    |
| `monitor.store.monitor_records(project)` (called separately by `monitor_blocking_start_for_lane` and `has_any_monitor`)          | 1,332   | ~4–7 s each |

Promotion, workspace claim, member creation, ToolRun reservation, and supervisor ack
come on top of those. So an in-agent start takes 20–40 s and prints nothing until it
finishes. Claude's `Bash` tool blocks for up to 2 minutes, so Claude agents survive
this. Codex's yielding exec does not.

Decision context: `decisions:adapters-normalize-harnesses` says each provider adapter
must tell the model the harness's real synchronous ceiling. Where that is not enough, it
must detect violations with a bounded guard and fail loudly.
`decisions:single-turn-agents` makes `sase monitor` the mechanism that must "exist and
stay reliable".

## Goals

- A Codex agent never ends its turn while a SASE handoff command is still running. If it
  does, the adapter catches it and re-drives the handoff, or fails loudly. It never
  records a silent success.
- An in-agent `sase monitor start` finishes in a few seconds and prints something at
  once, so no harness sees an empty result.
- If a handoff is still killed, for any provider, the host records it and tells the
  user. The run does not fall into a normal-looking declaration recovery.

## Non-goals

- Changing monitor semantics, fingerprints, or replay rules.
- Remediating `sase-185.2` itself. That is operational: relaunch or verify-and-close.
- Changing Claude or Muse wait guards beyond sharing helpers.

## Phases

### Phase `codex-handoff-guard` — Codex adapter conformance for handoff commands

- **Size:** medium
- **Depends on:** none

Make the Codex adapter conform to the single-turn contract for commands that are still
running when a turn ends.

1. **Fix the integrity signal.** In `src/sase/llm_provider/_subprocess_codex.py`, only a
   non-empty _final_ answer may suppress the guard. That means the `final_answer`-phase
   agent message, or `last_agent_message` on `task_complete`. Commentary-phase messages
   must not. Keep today's behavior of treating command items that started without a
   result, or were killed at teardown (`exit_code == -1`), as unfinished.
2. **Classify SASE handoff commands.** Add one shared predicate that says whether a
   command line is a SASE handoff command: one that ends by writing a pending handoff
   marker and killing the runner. Derive the set from the code paths that call
   `kill_agent_runner_group` or `write_pending_handoff_marker`: at least
   `sase monitor start` and `sase plan propose`, plus the pipe, questions, and gate
   handoff entry points. Do not hard-code guesses; find them. A pending or killed
   handoff command at `task_complete` is a stranded handoff, whatever the final message
   says. A pending or killed non-handoff command with an empty final answer keeps
   today's integrity-failure meaning.
3. **Bounded continuation instead of a silent pass or an immediate raise.** Mirror the
   Muse/Claude wait guards. Return the stranded-command facts from
   `stream_and_parse_codex_json_output` to `CodexProvider.invoke` rather than raising
   inside the stream parser. `invoke` then re-invokes Codex with reconstructed context,
   the way the existing interrupt path does (`--- Work So Far ---` plus a
   `--- Required Continuation ---` nudge). The nudge must:
   - name the killed command;
   - say it was killed when the turn ended, and that no monitor or handoff took effect;
   - tell the model to re-run the _identical_ command, so fingerprint replay adopts any
     partially started monitor;
   - tell it to keep waiting on the same exec session until it exits, by re-polling
     (`write_stdin` with empty input) until an exit code appears.

   Budget it with `SASE_CODEX_MAX_WAIT_CONTINUATIONS` (default 2). Log each firing to
   `wait_guard_log.jsonl` with reason `stranded_handoff_command`, sharing the Muse
   helper instead of duplicating it. When the budget is exhausted, raise
   `LLMInvocationError` with the command and the artifacts dir. Never return success.

4. **Tell the model the real ceiling.** Add a Codex single-turn directive, like Muse's
   `_muse_single_turn_directive`. Use a Codex config or developer-instructions channel
   if the pinned Codex CLI supports one; check `codex exec --help` and its config
   reference, and do not assume. Otherwise prefix the prompt the way Muse does. It must
   state:
   - `exec_command` returns control after `yield_time_ms` while the command keeps
     running;
   - anything still running when the turn ends is killed, and its effects never happen;
   - an empty or partial yielded result means "still running", not "done";
   - handoff commands (`sase monitor start`, `sase plan propose`, ...) can take up to a
     minute and must be waited on until they exit.
5. **Skill wording.** In `src/sase/xprompts/skills/sase_monitor.md`, "Core Rule"
   conflates two things. Split them: (a) never wait on the _monitored_ command; (b) the
   `sase monitor start` command itself must run to completion. It can take up to a
   minute and ends by killing your runner, so if your tool returns before it exits
   (yielded or backgrounded session), keep waiting on it and never end your turn while
   it runs. Apply the same clarification anywhere
   `src/sase/xprompts/skills/sase_final.md` ("Prepared Monitor Completion") or the plan,
   pipe, and questions skill sources tell the model the turn "will not return normally".
   Follow the `generated_skills.md` memory: preview with `sase skill init --diff`. Do
   not deploy to chezmoi from an unlanded tree.
6. **Tests.** Build NDJSON fixtures that replay the sase-185.2 shape: commentary
   messages, then an exec of `sase monitor start ...` that starts without a result or
   completes with `exit_code -1` after `task_complete`, then an empty final answer.
   Assert:
   - the guard fires and the continuation prompt names the command;
   - the budget is respected, and exhausting it raises;
   - a clean turn with a non-empty final answer and no stranded command is unaffected;
   - a stranded non-handoff command with a non-empty final answer is unaffected.

### Phase `fast-monitor-start` — make in-agent `sase monitor start` fast and never silent

- **Size:** medium
- **Depends on:** none

Target: an implicit in-agent start (no `-a`, `SASE_ARTIFACTS_DIR` set) completes in
single-digit seconds on a host with ~12k artifact records. It must not load full project
history.

1. **Pinned caller without a full scan.** `store_lane.resolve_caller_agent` currently
   calls `store.project_records(project)` before it checks the pinned
   `SASE_ARTIFACTS_DIR`. Read the pinned artifact dir's record directly first.
   `store.read_monitor_marker` / "read one member's own markers directly, without an
   index query" in `src/sase/monitor/store.py` is the precedent. Accept it under the
   existing ownership rules in `_pinned_caller_record`. Fall back to the full scan only
   when the pinned record is missing or belongs to someone else.
2. **One lane-scoped monitor read per start.** `monitor_blocking_start_for_lane`,
   `has_any_monitor`, and suffix allocation each read `monitor_records(project)`. Read
   once per start and share the snapshot through `_start_monitor_locked`. Prefer a
   lane-filtered read (by `agent_session`) over a project-wide monitor scan. If the
   artifact-index query wire (`AgentArtifactIndexQueryWire` /
   `query_agent_artifact_index`) has no session or lane filter, add one in the linked
   `sase-core` repo. Follow the Rust core boundary: open it with
   `sase repo open sase-core`, update wire, binding, and tests there, and move sase's
   `sase-core-revision.txt` pin (see `docs/rust_backend.md`). Do not add a Python-only
   scanner.
3. **Audit the rest of the start path** for other full-history reads. Candidates:
   `_replayed_lane_monitor`, promotion's `_refresh_artifact_index`,
   `preflight_monitor_workspace_claim`, `create_monitor_member`, ToolRun reservation,
   and the supervisor ack wait. Add cheap phase timing to the start: debug-level, or a
   `monitor_start_timing` entry in the member's artifacts. Future regressions should be
   diagnosable without re-deriving these numbers.
4. **Print immediately.** When `will_handoff_monitor_to_agent_runner()` is true, print
   one line to stderr before any slow work. Example:
   `sase monitor start: starting monitor for lane <lane>; this command hands off your turn when it finishes — wait for it to exit`.
   A yielding harness then shows non-empty, self-explanatory output. Keep the existing
   rule that all stdout (including the `--json` envelope) prints before the runner is
   killed.
5. **Tests.** A unit test proves an implicit pinned start does not call the full
   `project_records` scan (patch and count). Another proves the lane monitor read
   happens once per start. Keep existing monitor start, replay, and lane tests green.
   Report measured before/after start latency on this host in the phase close note.

### Phase `aborted-handoff-evidence` — host records and surfaces a killed handoff

- **Size:** medium
- **Depends on:** `fast-monitor-start`

This safety net works for any provider. A handoff command can still die mid-flight:
Claude `Bash` timeouts, a harness kill, OOM. When that happens, the host must notice
instead of running a normal-looking declaration recovery.

1. **In-flight marker.** When `sase monitor start` runs inside an agent, it writes a
   small in-flight marker into `SASE_ARTIFACTS_DIR` before any slow work: pid, process
   identity, argv, lane, started_at. Reuse `sase.agent.pending_handoff_write` and marker
   conventions. Remove it on every error exit. It is superseded when the pending handoff
   marker is written. Apply the same marker to the other handoff commands identified in
   phase `codex-handoff-guard` if they share the helper cheaply. Otherwise, file the gap
   as a `PROPOSED FOLLOW-UP:` note.
2. **Runner detection.** When the provider turn ends with no pending handoff marker, the
   runner checks for an in-flight marker:
   - If the owning process is still alive, wait a bounded time for it to finish or write
     the pending marker, then adopt the handoff normally. This is the host's own handoff
     protocol, not a harness wait.
   - If the process is dead, record `handoff_aborted` with the command in `done.json`
     and the workflow state. Add a clear section to the declaration-recovery evidence,
     so the recovery commit message is accurate: "this run tried `<cmd>` but it was
     killed before the monitor started; no follow-up agent will run". Send a SASE
     notification naming the agent and its assigned bead, if any. A phase bead left
     `in_progress` then no longer stalls dependents silently.
3. **Tests.** Cover: marker lifecycle on success, error, and kill; runner behavior for a
   live marker, a dead marker, and no marker; recovery evidence text; and the
   notification payload.

## Verification

Each phase runs the project's standard verification (`just check`, via the verify
monitor or prepared monitor completion per `/sase_final`) and the focused tests it adds.
Phase `codex-handoff-guard` also confirms that `sase skill init --diff` shows only the
intended skill wording changes.
