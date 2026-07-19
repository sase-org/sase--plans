---
tier: tale
title: Stop stale SASE_AGENT_* env leaking into new agent launches
goal: 'Newly launched agents, chop script runs, and the axe daemon never inherit another
  agent''s SASE_AGENT_* identity/behavior environment, so a Telegram (or any chop)
  launch can no longer claim a colliding agent name, silently auto-approve plans,
  or attribute chats/phases to the wrong run.

  '
create_time: 2026-07-18 22:26:20
status: wip
prompt: 202607/prompts/agent_env_identity_leak.md
---

# Plan: Stop stale SASE*AGENT*\* env leaking into new agent launches

## Incident / Motivation

A Telegram launch notification showed `🚀 CLAUDE(claude-fable-5) Launched @sase-6v.9 workspace #12` for a brand-new
prompt that contained no `%name:` directive. `sase-6v.9` was NOT a fresh name: it already belonged to a running codex
epic phase-worker (launched with `%name:sase-6v.9 %clan(sase-6v, tribe=epic)`). The Telegram plugin displayed the name
faithfully — the launched agent really did claim `sase-6v.9`, and `~/.sase/agent_name_registry.json` now records the
second agent under `collision_owners` for that entry. The correct behavior would have been a fresh auto name
(`e8`-style).

Root cause chain (verified live via `/proc/<pid>/environ` of the affected processes):

1. `src/sase/axe/_process_start.py` (`start_axe_daemon_result`) spawns the axe daemon with `env = os.environ.copy()`.
   The daemon gets auto-(re)started from arbitrary process contexts — including from inside running agents and chop runs
   (the daemon running at the time of this investigation itself carried `SASE_CHOP_LUMBERJACK=hooks` /
   `SASE_CHOP_NAME=workflow_checks` vars from the chop context that restarted it). At some point the daemon was
   restarted from inside the codex worker `sase-6v.9`'s process tree and inherited its full agent identity env.
2. `src/sase/axe/chop_script_runner.py` (`run_chop_script`, `stream_chop_script`) builds the chop subprocess env as
   `dict(os.environ)` + per-run extras (and passes `env=None`, i.e. full inherit, when there are no extras). The
   `tg_inbound` chop run therefore executed with the codex worker's `SASE_AGENT_*` vars.
3. `src/sase/agent/launch_spawn.py` env shaping (the `env_shape` stage) starts from `dict(os.environ)` and scrubs
   several inherited categories (`_remove_inherited_sase_codex_home`, `_remove_inherited_workspace_preallocation_env`,
   `_remove_inherited_deferred_workspace_env`, `_remove_inherited_agent_var_context_env`,
   `_remove_inherited_multi_agent_prompt_env`, `_remove_inherited_model_alias_overrides`,
   `_remove_inherited_linked_repo_env`) — but has no scrub for the agent identity/behavior family. The spawned agent
   inherited, among others:
   - `SASE_AGENT_PLANNED_NAME=sase-6v.9` and `SASE_AGENT_NAME=sase-6v.9`
   - `SASE_AGENT_CHAT_PATH=<the codex worker's chat file>`
   - `SASE_AGENT_ROOT_TIMESTAMP=<the codex worker's artifacts timestamp>`
   - `SASE_AGENT_AUTO_APPROVE=1`
4. The child name-resolution ladder in `src/sase/axe/run_agent_directives.py` reads
   `planned_name = os.environ.get("SASE_AGENT_PLANNED_NAME")` and, for a prompt with no user-explicit name, claims that
   planned name. It trusted the leaked value and claimed `sase-6v.9`, producing the registry collision.

The blast radius is bigger than the display glitch:

- **Name collision**: two live agents share one name; `sase agent show`, kill/fork/wait targeting, and the registry all
  become ambiguous.
- **`SASE_AGENT_AUTO_APPROVE=1` leak**: `src/sase/main/plan_approve_handler.py` consults this env var directly, so a
  user launch that expected a manual plan-approval gate can silently auto-approve.
- **`SASE_AGENT_CHAT_PATH` / `SASE_AGENT_ROOT_TIMESTAMP` leaks**: commit tracking
  (`src/sase/workflows/commit/commit_tracking.py`), plan gates (`src/sase/plan_gate.py`), and exec markers
  (`src/sase/axe/run_agent_exec_markers.py`) attribute work to the wrong agent's chat/run.

No changes are needed in the sase-telegram repo: it correctly reports the name the agent actually claimed (from
`AgentLaunchResult.agent_name` / polled `agent_meta.json`).

## Design

Principle: **agent identity env is per-launch data, never ambient inheritance.** Every variable in the
`SASE_AGENT`/`SASE_AGENT_*` family that a new process legitimately needs is already set deliberately at its spawn
boundary (via `prepared.env_delta`, which carries the launch request's `extra_env` — retry handoff, clan membership,
family attach, etc.). Anything arriving only via ambient `os.environ` at these boundaries is stale by definition and
must be dropped.

### 1. Scrub agent identity env when spawning a new agent

In `src/sase/agent/launch_spawn.py`, add a scrub helper alongside the existing `_remove_inherited_*` family, e.g.
`_remove_inherited_agent_identity_env(env)`, that removes `SASE_AGENT` and every key starting with `SASE_AGENT_` from
the copied environ. Call it in the `env_shape` stage with the other scrubbers, **before**
`subprocess_env.update(prepared.env_delta)` so every value the current launch deliberately sets survives. A prefix-based
scrub (rather than an allowlist of known names) also covers future `SASE_AGENT_*` vars automatically.

Verify during implementation that `prepared.env_delta` (produced by the Rust-side `prepare_agent_launch` in
`sase_core_rs`) is the sole channel for deliberate `SASE_AGENT_*` values at this boundary, including the request's
`extra_env` (`AgentLaunchRequestWire.extra_env`). If any launch flow sets a needed `SASE_AGENT_*` value on
`subprocess_env` outside `env_delta`, order the scrub before that write too. The existing narrower helper
`_remove_inherited_agent_var_context_env` (drops `SASE_AGENT_VAR_UPSTREAMS_ENV`) becomes redundant once the prefix scrub
exists; fold it in or delete it, keeping its intent documented.

Deliberately do NOT scrub `SASE_CHOP_*` here: chop context vars are how `record_chop_agent_launch_from_env` and
`agent_meta.json` record which chop launched the agent (e.g. `chop_lumberjack: telegram`), and each chop run sets its
own values.

### 2. Scrub agent identity env when running chop scripts

In `src/sase/axe/chop_script_runner.py`, both `run_chop_script` and `stream_chop_script` currently build
`subprocess_env` only when extras are provided (otherwise the child fully inherits). Change both to always compose the
env: copy `os.environ`, strip `SASE_AGENT`/`SASE_AGENT_*`, then apply the per-run `env` extras. Chop scripts are system
jobs and must never run with an agent's identity — this stops a poisoned daemon (or a chop triggered from inside an
agent) from propagating agent identity into anything a chop spawns.

Extract the composition into a small pure helper (environ mapping in, dict out) so it can be unit-tested without
spawning processes, and reuse the same underlying scrub logic as change 1 (a shared helper, e.g. in
`src/sase/agent/identity.py` or a small env-hygiene module, imported by both call sites).

### 3. Sanitize the axe daemon's own environment at startup

In `src/sase/axe/_process_start.py` (`start_axe_daemon_result`), the daemon env is currently `os.environ.copy()` plus
the lifecycle-lock fd var. Strip both `SASE_AGENT`/`SASE_AGENT_*` and `SASE_CHOP_*` prefixed vars from that copy before
spawning. The daemon is a long-lived system service; whatever context restarts it (an agent's shell, a chop run, the
TUI), it must not carry that context's identity. (`SASE_CHOP_*` is safe to strip here — unlike change 1 — because the
daemon is not a chop child; each chop run it later dispatches gets fresh `SASE_CHOP_*` values from the runner.)

### 4. Defense-in-depth: stop trusting ambient planned names in the child

In `src/sase/axe/run_agent_directives.py`, the ladder honors `SASE_AGENT_PLANNED_NAME` whenever the prompt has no
user-explicit name. Even with scrubs 1–3, keep a guard so a stale planned name can never steal a live agent's name:
before using `planned_name`, verify the durable name registry's reservation for that name points at THIS run's artifacts
directory (the parent reserves planned names with the child's future artifacts dir via
`PlannedNameAllocator._reserve_planned_name` / `reserve_registered_name`). On mismatch, log a warning and fall through
to the normal allocation ladder (auto name) instead of claiming. Reuse the registry read helpers in
`src/sase/agent/names/` rather than re-parsing the JSON.

## Testing

- Unit tests for the shared scrub helper: `SASE_AGENT`, `SASE_AGENT_NAME`, `SASE_AGENT_PLANNED_NAME`,
  `SASE_AGENT_AUTO_APPROVE` removed; non-matching keys (`SASE_CHOP_NAME`, unrelated vars) preserved where the boundary
  keeps them.
- `launch_spawn` env-shaping test: with ambient poisoned vars set (planned name, auto approve, chat path) and an
  `env_delta` that sets this launch's own values, the composed env contains only the deliberate values.
- `chop_script_runner` composition-helper tests: full-inherit case (no extras) still strips agent identity; extras
  override correctly.
- `_process_start` test for the daemon env sanitizer (pure helper on a fake environ).
- `run_agent_directives` regression test: ambient `SASE_AGENT_PLANNED_NAME` naming a registry entry reserved for a
  different artifacts dir → child ignores it and allocates an auto name (this is the exact incident reproduction).
- Run `just check`.

## Risks & Notes

- **Rust core boundary**: env composition stays in Python (`launch_spawn.py` already hosts the `_remove_inherited_*`
  family); `prepare_agent_launch`/`spawn_prepared_agent_process` contracts in `sase_core_rs` are unchanged. If
  implementation reveals the Rust side reads ambient `SASE_AGENT_*` expecting inheritance, stop and re-evaluate before
  scrubbing that variable.
- **Follow-up/step agents**: coder/epic follow-ups and retry handoffs receive their context via `extra_env` →
  `env_delta`, so the scrub must not break them; the tests above should include one `extra_env`-carrying case (e.g.
  `SASE_AGENT_RETRY_HANDOFF`).
- The currently-running collision (`sase-6v.9` claimed by both the epic worker and the Telegram-launched agent, plus
  that agent's leaked `SASE_AGENT_AUTO_APPROVE=1`) is existing state this plan does not remediate; it is surfaced to the
  user separately.
