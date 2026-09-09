---
tier: epic
title: "`sase pipe`: a first-class hand-off to the next agent family member"
goal: "An agent can end its own turn and hand the work to its next family member with
  one command, `sase pipe '<prompt>'`, exposed to agents as the `/sase_pipe` skill; the
  `sleep 1` monitor hack is no longer needed, and every in-process family-successor
  hand-off in the runner (plan approval, questions, pipe) plus every out-of-process
  family spawn (monitor follow-up) runs through one shared engine.

  "
phases:
  - id: successor
    title: Shared in-process family-successor engine
    depends_on: []
    size: medium
    description:
      "successor: extract the runner's repeated become-the-next-family-member dance into
      one engine and migrate the plan-approval coder hand-off and the questions
      follow-up onto it, behavior unchanged."
  - id: handoff
    title: Shared pending-handoff marker protocol
    depends_on: []
    size: small
    description:
      "handoff: give the four marker-writing CLI hand-offs one guard/write/kill helper
      and one marker registry, and register the pipe marker there."
  - id: spawn
    title: Shared out-of-process family spawn
    depends_on: []
    size: medium
    description:
      "spawn: put the detached child spawn and the family-attach env encoding behind one
      primitive that monitor follow-up and retry spawn both call."
  - id: pipe
    title: The `sase pipe` command
    depends_on:
      - successor
      - handoff
    size: medium
    description:
      "pipe: add the CLI, its guards and chain bound, the pending marker it writes, and
      the runner adoption that continues as the piped successor."
  - id: skill
    title: The `/sase_pipe` skill and user documentation
    depends_on:
      - pipe
    size: small
    description:
      "skill: author the bundled skill source, wire it into the skill enumeration tests,
      and document the command across the CLI, monitor, and xprompt docs."
  - id: smoke
    title: End-to-end pipe exercises
    depends_on:
      - pipe
      - skill
      - spawn
    size: xsmall
    description:
      "smoke: run real agents that pipe, confirm the family chain and ACE rows, and land
      the epic on a full verification run."
proposed_by: bbugyi200.athena.05f
status: done
bead_id: sase-p8
create_time: 2026-09-09 19:49:47
---

- **PROMPT:**
  [prompts/202608/agent_pipe.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/agent_pipe.md)
- **BEAD:**
  [sase-p8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-p8/README.md)

# Plan: `sase pipe`: a first-class hand-off to the next agent family member

## Why

An agent that wants its successor to pick up the work has no direct way to ask for one.
Agent-initiated launches go through `LaunchApproval` (`/sase_run`), which is correct for
fan-out but wrong for a hand-off: it needs a human answer, and the agent is asking to be
_replaced_, not to gain a helper. So agents have learned a workaround:

```bash
sase monitor start --command 'sleep 1' --reason '...' --next '<the real prompt>'
```

That works only because `sase monitor start` already kills the calling agent and the
monitor supervisor already launches a family child when its command settles. Everything
else about it is noise: a supervisor process, a proc row, a monitor member agent, a
captured log, a timeout, a `MONITORING`/`MONITORED` status, and a one-second sleep whose
only job is to finish.

The machinery for doing this honestly is already in the runner. When a plan is approved,
`handle_accepted_plan` does not spawn anything — it saves the planner's chat, allocates
the `--code` suffix, creates follow-up artifacts, sets `state.current_prompt`, and lets
the execution loop run the coder as the next family member in the same process, same
workspace, same claim. `handle_questions_marker` does the identical dance for the
question follow-up. `sase pipe` is that same hand-off, with the prompt supplied by the
agent instead of by a plan or an answered question.

That also answers the second half of this epic. Four call sites now build a family
successor, and they share almost nothing:

| Site                            | Mechanism               | What it duplicates                                   |
| ------------------------------- | ----------------------- | ---------------------------------------------------- |
| `run_agent_exec_plan_accept.py` | in-process continuation | step bump, promote, suffix, artifacts, model, prompt |
| `run_agent_exec_questions.py`   | in-process continuation | the same six steps, written separately               |
| `monitor/followup.py`           | detached spawn          | family-attach plan → env → `spawn_agent_subprocess`  |
| `axe/run_agent_retry_spawn.py`  | detached spawn          | timestamp, workflow name, claim transfer             |

Each of the first two is ~60 lines of near-identical sequencing whose ordering
constraints (promote at step 2, allocate before create, write model meta after create)
are load-bearing and undocumented. Adding pipe as a third copy would be the wrong move;
extracting the engine first and adding pipe as its third caller is the right one.

## What `sase pipe` is, and is not

`sase pipe '<prompt>'` ends the calling agent's turn and continues the run as that
agent's next family member, in the same workspace, with the prompt the agent wrote.

- It is **not** a launch. No new workspace, no new claim, no `LaunchApproval`, no
  approval gate. The agent is handing over its own turn, exactly as an approved plan
  hands the planner's turn to the coder. A user who does not want this can kill the
  family; nothing new is reachable that a monitor `--next` did not already reach.
- It is **not** a monitor. Nothing is executed, captured, timed out, or supervised.
  `/sase_monitor` remains the answer for "run this long command" and for "wait".
- It is **not** fan-out. One successor, serially. `/sase_run` remains the answer for
  helpers, reviewers running alongside, and clans.

## Design constraints this epic must hold

- **In-process continuation, not a spawn.** The successor runs in the same runner
  process on the next loop iteration, like the plan and question hand-offs. This is what
  makes pipe cheap and race-free: the workspace claim never moves, the parent's chat is
  saved by the same process that starts the successor, and there is no window in which
  two agents are live in one lane. The detached-spawn path is unified separately (phase
  `spawn`) because monitor follow-up genuinely needs it — the starter is already dead by
  the time its command settles.
- **Behavior-preserving extraction.** Phase `successor` must not change a single
  observable output of plan approval or question follow-up: same suffixes, same names,
  same `agent_meta.json` fields, same relationships, same prompt artifacts. Its evidence
  is the existing test suite passing untouched, plus new tests pinning the engine.
- **The piped prompt is a full sase prompt.** The execution loop wraps
  `state.current_prompt` in `create_anonymous_workflow()` on every iteration, so `%`
  directives and `#` references in a piped prompt are live for the successor. This is a
  feature (a `%model` prefix is exactly how `handle_accepted_plan` routes the coder) and
  a hazard the skill must state.
- **Chains must be bounded.** Nothing stops a piped successor from piping again. A
  successor whose prompt tells it to pipe would otherwise loop until the workspace or
  the user's patience runs out. Bound the chain with a configuration field, not a
  feature flag: users are meant to choose this number forever.
- **No feature flag.** `sase pipe` is a new command that nothing invokes until an agent
  or a user types it, so there is no default-path behavior to shield; the shared-engine
  phases are behavior-preserving refactors covered by the existing suites. Per
  `sase/memory/sase_flags.md`, neither is a flag candidate, and the chain bound is a
  config field rather than a flag.

## Boundary note

Under `sase/memory/rust_core_backend_boundary`, shared backend behavior belongs in
`../sase-core/crates/sase_core`. This epic stays in Python. Every mechanism it touches —
the runner execution loop, SIGTERM hand-off markers, artifacts-directory creation, agent
name allocation, and detached process spawning — is Python host machinery today, and the
Rust core is already consulted underneath it through the existing facades
(`prepare_agent_launch`, `reserve_launch_timestamp_batch`,
`canonical_agent_artifact_path`). No new wire contract, storage format, or domain rule
is introduced: pipe composes host-side process lifecycle. A second frontend cannot
"pipe" anything, because piping is a property of a live runner process.

## Shared in-process family-successor engine

Add `src/sase/axe/run_agent_successor.py`. It owns the sequence both existing in-process
hand-offs perform today, in the order they perform it, with the ordering constraints
documented in the module docstring.

Define a frozen request record:

- `SuccessorRequest` — `base_meta` (the metadata the successor inherits), `suffix` (an
  explicit canonical suffix such as `--code`) _or_ `suffix_template` (an `@`-bearing
  template such as `--@` or `--plan-@`) with exactly one of the two set,
  `extra_reserved_suffixes`, `agent_family_role`, `prompt`, `relationships`,
  `prompt_artifact_label`, and an optional `model` carrying the already-resolved
  follow-up model record.

Expose one function, `continue_as_successor(ctx, state, request) -> str`, returning the
successor's agent name and performing, in order:

1. `state.agent_step += 1`, and
   `promote_to_workflow(ctx.artifacts_dir, ctx.agent_name, role_suffix=...)` when the
   step reaches 2 and the agent is named. This is the step that turns a lone agent into
   a family root; it must stay before artifact creation.
2. Resolve the successor suffix: use `request.suffix` verbatim when given, otherwise
   `allocate_agent_family_child_suffix(ctx.agent_name, request.suffix_template, extra_reserved_suffixes=...)`,
   with the existing `_fallback_question_suffix`-style template render as the
   unnamed-agent fallback.
3. Set `state.current_role_suffix`, then
   `state.current_artifacts_dir = create_followup_artifacts(...)` with
   `agent_name_override=plan_chain_agent_name(ctx.agent_name, suffix)`,
   `workflow_name=ctx.agent_name`, `agent_family_role`, and `relationships`.
4. Write the follow-up model metadata when `request.model` is set, reusing the existing
   `_write_followup_model_meta` / `_write_followup_model_alias_meta` logic moved here
   from `run_agent_exec_plan_accept.py` (the `_FollowupModel` record and its writers
   move with it; `run_agent_exec_plan_accept_models.py` keeps the _resolution_ helpers).
5. Set `state.current_prompt` to the request's prompt.
6. `store_followup_prompt_artifact(state.current_artifacts_dir, state.current_prompt, label=request.prompt_artifact_label)`.

Then migrate both callers:

- `run_agent_exec_plan_accept.py` calls it with `suffix=PLAN_CHAIN_CODER_SUFFIX`, the
  plan relationships block, the resolved `followup_model`, and the assembled coder
  prompt (model prefix, optional `#fork:<base>--plan` under the
  `coder_inherits_planner_chat` flag, VCS prefix, plan body, coder extras, embedded
  refs). Prompt _assembly_ stays in the plan module; only the sequencing moves.
- `run_agent_exec_questions.py` calls it with the question suffix template, the question
  relationships block, `agent_family_role=followup_role`, and the assembled follow-up
  prompt. Its `interrupted_suffix` bookkeeping, `promote_to_workflow` role-suffix
  argument, and Q&A round merging stay where they are.

Test `tests/axe/test_run_agent_successor.py` for the engine directly: explicit suffix,
allocated suffix, unnamed-agent fallback, step-2 promotion firing exactly once, model
meta written only when a model is supplied, relationships and prompt artifact recorded.
The existing plan-accept and questions suites must pass unmodified — that is the
behavior-preservation evidence, so do not edit them to fit the refactor.

## Shared pending-handoff marker protocol

Four commands already end an agent turn by writing a JSON marker into
`SASE_ARTIFACTS_DIR` and sending SIGTERM to the runner group: `sase questions`
(`.sase_questions_pending`), `sase plan propose` (`.sase_plan_pending`),
`sase monitor start` (`.sase_monitor_pending`), and now `sase pipe`
(`.sase_pipe_pending`). Each re-implements the guard/write/fsync/kill sequence, and
`PENDING_HANDOFF_MARKERS` in `src/sase/agent/pending_handoff.py` must be kept in sync by
hand.

In `src/sase/agent/pending_handoff.py`, make the marker set the single registry: name
each marker as a constant, keep `PENDING_HANDOFF_MARKERS` derived from those constants,
and add the pipe marker. `src/sase/monitor/handoff.py` keeps `MONITOR_PENDING_MARKER` as
a re-export so its importers do not churn.

Add `write_pending_handoff_marker(marker, payload, *, artifacts_dir=None)` and
`handoff_guard()` to a new `src/sase/agent/pending_handoff_write.py` (kept out of the
read-only classification module so `run_agent_runner_signals.py` keeps its cheap
import). `handoff_guard()` returns the resolved artifacts dir or raises a typed error
when `SASE_AGENT` or `SASE_ARTIFACTS_DIR` is unset, and refuses when a pending handoff
marker already exists — a second hand-off from one turn is a bug, and today it silently
races. `write_pending_handoff_marker` stamps `timestamp` (the runner discards markers
that postdate the kill), writes atomically, and fsyncs.

Migrate `questions_command_handler.py` and `plan_propose_handler.py` onto the helper;
`monitor/handoff.py::write_monitor_pending_marker` keeps its own record-shaped payload
but writes through it. `run_agent_runner_signals.py` derives its
`_NON_MONITOR_HANDOFF_MARKERS` from the registry, so the pipe marker automatically joins
the set that suppresses the SIGTERM workspace release — the successor needs the claim
the parent is holding.

Test that a second marker write from one turn raises, that the guard's messages name the
missing environment variable, and that `_NON_MONITOR_HANDOFF_MARKERS` contains the pipe
marker.

## Shared out-of-process family spawn

`monitor/followup.py::_spawn_followup` and `axe/run_agent_retry_spawn.py` both spawn a
detached child agent from inside a live process and move the workspace claim to it, and
both hand-roll it. Put the common primitive in a new `src/sase/agent/detached_child.py`:

- `spawn_detached_child(*, meta_source, project_name, prompt, workspace_dir, workspace_num, cl_name, extra_env, transfer_from_pid, vcs_ref=None, update_target=..., is_home_mode=...) -> AgentLaunchResult`
  — reserves the timestamp via `reserve_launch_timestamp_batch(1)`, derives the
  `ace(run)-<ts>` workflow name, and calls `spawn_agent_subprocess` with
  `retry_transfer_from_pid`. This is the claim- continuity primitive, and it is all
  `run_agent_retry_spawn.py` needs; that module keeps owning its handoff record, retry
  env, fallback-model selection, and metrics.
- `family_attach_env(plan) -> dict[str, str]` — the `INTERNAL_AGENT_NAME_BYPASS_ENV`
  plus `FAMILY_ATTACH_ENV` JSON encoding currently duplicated between
  `monitor/followup.py` and `agent/_family_attach_launch.py`. Both call it.
- `spawn_family_successor(...)` — resolves the family-attach plan for a
  `FamilyAttachDirective`, applies the `parent_is_running=False` / role / parent-
  workspace overrides, layers `family_attach_env`, and delegates to
  `spawn_detached_child`.

`monitor/followup.py` keeps everything genuinely monitor-shaped: the starter settle
poll, the prompt composition, the three-rung workspace degradation ladder (transfer →
fresh claim → workspace 0), and the `monitor_followup_*` metadata it records. Only its
spawn body is replaced.

Explicitly **not** merged here, with the reason recorded in the new module's docstring:
`bead/work.py` builds `%id(!name, bead=...)` prompt directives and launches through
`launch_agents_from_cwd`, which is the user-equivalent launch path with its own
validation, gating, and fan-out. It is already the shared path for what it does;
rerouting it through a family-spawn primitive would bypass launch validation, not share
it.

Test the primitive with a faked `spawn_agent_subprocess`: timestamp reservation and
workflow-name derivation, the transfer pid being forwarded, the family env encoding
round-tripping through `load_family_attach_plan_from_env`. The existing monitor
follow-up suites must pass unmodified.

## The `sase pipe` command

### CLI

Register `pipe` in `src/sase/main/parser.py`'s lazy registry and add
`src/sase/main/parser_pipe.py` and `src/sase/main/pipe_handler.py`, routed from
`entry.py` next to the other single-purpose agent commands.

```
sase pipe PROMPT [-f] [-j] [-m MODEL] [-n TOKEN] [-r TEXT]
```

`PROMPT` is positional because the command cannot run without it
(`sase/memory/cli_rules` forbids required options). Options, alphabetical in help, each
with a short alias:

| Option         | Effect                                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `-f, --fresh`  | Successor starts with a clean context window. Default is to prefix `#fork:<current agent name>` so it inherits this chat. |
| `-j, --json`   | Print a machine-readable hand-off summary instead of the rich one. Printed _before_ the kill, never after.                |
| `-m, --model`  | Model or alias for the successor (`opus`, `opus@high`, `sonnet`, `codex/gpt-5`). Default: inherit this agent's model.     |
| `-n, --name`   | Successor role token: `review` yields `<family>--review`. Default: the next free numbered member (`--@`).                 |
| `-r, --reason` | One-line reason, recorded on the successor and shown in agent lists.                                                      |

Help text carries one canonical example and the "this ends your turn" warning; output is
colored through the existing `rich` console helpers, matching `monitor_render.py`.

### Guards and the chain bound

The handler, before writing anything:

- Calls `handoff_guard()` from phase `handoff`; outside an agent it exits non-zero with
  "`sase pipe` is only available inside a sase agent".
- Rejects an empty or whitespace-only prompt.
- Validates `-n/--name` through `canonical_plan_chain_suffix`, rejecting a token that
  would collide with a reserved suffix (`plan`, `q`, `code`, `epic`, `commit`, `mon`).
- Reads the calling agent's own `agent_meta.json` for its `pipe_depth` relationship
  (absent means 0) and refuses when the next link would exceed `max_agent_pipe_chain`.
  The error names the limit, the config key, and the chain length reached.

Add `max_agent_pipe_chain: 8` to `src/sase/default_config.yml` beside
`max_running_agents`, with a comment explaining that it bounds runaway self-piping, and
read it through the existing config accessor used for `max_running_agents`.

### Marker and runner adoption

The handler prints its summary, writes `.sase_pipe_pending` through the phase-`handoff`
writer with `{prompt, reason, model, name_token, fresh, pipe_depth, timestamp}`, and
calls `kill_agent_runner_group(artifacts_dir)`. Printing must precede the kill; the
process does not return from it.

Add `src/sase/axe/run_agent_exec_pipe.py` with
`handle_pipe_marker(pipe_data, ctx, state) -> str | None`, and dispatch it from
`_handle_killed_iteration` in `run_agent_exec.py` alongside the plan, questions, and
monitor markers (including the user-kill drain list, so a user kill discards a pending
pipe). It:

1. `normalize_handoff_interruption_state` + `finalize_handoff_artifacts_as_completed` on
   the current artifacts dir, mirroring `handle_monitor_marker`.
2. Saves the current member's chat with a synthesized "# Pipe hand-off" response naming
   the successor, the reason, and the piped prompt; records `chat_path` on the member's
   meta and step marker. The chat must be saved before the successor is created, because
   a default (non-`--fresh`) pipe forks it by name.
3. Resolves the successor model from `-m/--model` through the existing model-resolution
   helpers, producing the `_FollowupModel` record the shared engine writes.
4. Builds the successor prompt: optional model directive prefix, optional
   `#fork:<current agent name>` (omitted under `--fresh`), then the piped prompt
   verbatim. The piped text is never rewritten or escaped — it is the agent's prompt.
5. Calls `continue_as_successor` with `suffix_template="--@"` (or the explicit
   `--<token>` suffix), `agent_family_role="feedback"` (or the token), relationships
   `{piped_from, pipe_reason, pipe_depth: n+1, patch_name, changespec_name}`, and
   `prompt_artifact_label="Piped prompt"`.
6. `reset_killed()` and returns `None`, so the execution loop runs the successor.

Tests: `tests/main/test_pipe_handler.py` for guards, the depth bound, name validation,
`--json` shape, and that the summary is printed before the kill;
`tests/axe/test_run_agent_exec_pipe.py` for marker adoption — successor name and suffix,
fork prefix present by default and absent under `--fresh`, model prefix and meta,
`pipe_depth` incrementing, the parent's chat saved first, and a marker postdating the
kill being ignored. Extend `tests/test_axe_run_agent_exec_killed_iteration.py` and
`tests/test_llm_provider_postprocessing.py` for the new marker.

## The `/sase_pipe` skill and user documentation

Add `src/sase/xprompts/skills/sase_pipe.md`. Frontmatter `name: sase_pipe`,
`skill: true`, `log_skill_use: false`, and this description, which both restricts the
skill to explicit user requests and says what it is for:

> Hand this agent's turn to a fresh successor in the same agent family: this turn ends
> and the next family member starts immediately with the prompt you write, in the same
> workspace, optionally on a different model or with a clean context window. Use ONLY
> when the user explicitly asks you to pipe or hand off work to another agent. Not for
> running or waiting on a command (`/sase_monitor`), and not for launching helper agents
> (`/sase_run`).

The body stays under ~70 lines — every token is loaded into some agent's context — with
these sections and nothing else:

- **Core Rule**: only on explicit user request; the command kills this agent, so the
  turn will not return normally; say everything inside the piped prompt, not after the
  command.
- **Canonical Invocation**: one `sase pipe '<prompt>' --reason '...' --model opus`
  example.
- **When To Pipe**: remaining work needs a different model or effort; this context
  window is spent and the rest is self-contained; the next step is a distinct role that
  deserves its own agent row and reply. Then the three non-uses: a long command
  (`/sase_monitor`), parallel helpers (`/sase_run`), and work the agent can just do now.
- **Options**: the five flags, one line each, each saying what the default is.
- **Hazards**: the piped prompt is re-parsed, so `%` directives and `#` references in it
  are live — fence literal syntax; the successor runs in the same workspace with the
  same uncommitted changes and Patch, so it takes no workspace reference; a non-zero
  exit means nothing was handed off and the agent is still running; chains are bounded
  by `max_agent_pipe_chain`; do not keep working after the command.

Then wire it up:

- Add the `sase_pipe` case to `tests/main/test_init_skills_sources.py`'s skill
  enumeration with the phrases the body must keep.
- Add the row to the bundled-skills table in `docs/xprompt.md`.
- Document the command in `docs/cli.md` next to `sase monitor`, and add the distinction
  to `docs/monitors.md`: monitors are for running and waiting on commands, pipe is for
  handing off the turn — and `sleep 1 --next` is no longer the way to get a successor.
- Note the new hand-off in whichever agent-family documentation already lists the plan
  and question hand-offs, so pipe appears where a reader looks for family members.

Do not run `sase skill init` from this phase's tree. `sase/memory/generated_skills.md`
requires the source change to be committed and landed on the canonical branch first;
deployment is the owner's step after this epic lands.

## End-to-end pipe exercises

Verification that unit tests cannot give, run against real agents in a scratch project:

1. A pipe with defaults: confirm the parent's row shows the synthesized hand-off reply,
   the successor appears as the next family member, it inherits the chat, and both share
   one workspace and one claim.
2. A pipe with `--fresh --model` and an explicit `--name`: confirm the successor's name,
   its recorded model in ACE, and the absent fork.
3. A two-link chain, then a chain driven past `max_agent_pipe_chain`: confirm the bound
   fails the command with the configured limit named and leaves the agent running.
4. `sase monitor start --command 'sleep 1' --next '...'` still behaves exactly as before
   — the phase `spawn` refactor must not have changed it.

Land the epic on `just check-full` through `/sase_monitor`, never inline, per
`sase/memory/build_and_run`. Record any flake that is not caused by this epic as a task
bead through `/sase_new_task` rather than absorbing it here.
