---
tier: tale
title: Propagate %auto across monitor follow-up launches
goal:
  A monitor's follow-up shell inherits the starter's %auto state, so plans it proposes
  auto-approve exactly as the original launch requested.
size: small
proposed_by: bbugyi200.athena.0lj
create_time: 2026-09-15 16:23:54
status: wip
---

# Propagate `%auto` across monitor follow-up launches

## Problem

An agent launched with `%auto` that starts a sase monitor loses its auto-approval state
when the monitor settles: the mechanically launched follow-up shell proposes plans that
gate for manual review, even though the launch explicitly opted into auto-approval.

Confirmed instance (2026-09-15, agent family `sase-116.5.land`):

- The root shell (artifacts `ace-run/202609/15/20260915135259`) was launched with
  `%auto` and carries `approve: true` in its `agent_meta.json`.
- The monitor member `sase-116.5.land--mon` (`20260915152854`) inherited
  `approve: true`, because `src/sase/monitor/start_lane.py` builds the member meta as a
  full copy of the starter meta (`member_meta = dict(raw_meta)`).
- The follow-up shell the monitor launched at settlement (`20260915155752`) has **no**
  `approve` / `auto_approve_plan_action` / `auto_approve_argument` keys. Its tale plan
  proposal (`plan_action: tale`, submitted 20:04:01Z) therefore created a plan gate
  member (`20260915160424`) that waited for manual approval.
- By contrast, the starter's `%queue` state **did** reach the follow-up shell
  (`queue_weight: 2.0`, `queue_weight_explicit: true`), which isolates the gap to
  `%auto` specifically.

## Root cause

`launch_followup_agent` in `src/sase/monitor/followup.py` composes the successor's
launch prompt in its local `_compose` helper as:

```python
prefix = queue_launch_prefix(meta)
return f"{prefix}{vcs_prefix}{prompt}" if prefix or vcs_prefix else prompt
```

The follow-up prompt machinery re-authors every launch-state directive except `%auto`:
`%queue` via `queue_launch_prefix` (`src/sase/monitor/continuation_delivery.py`),
`%model`/`%effort` via `compose_followup_prompt`, and the VCS workflow prefix via the
frozen continuation intent. The `%auto` state is sitting in the monitor member's `meta`
(as `approve`, `auto_approve_plan_action`, `auto_approve_argument`) but is never
re-authored into the prompt, and no other channel carries it:

- `SASE_AGENT_AUTO_APPROVE` is set by each runner for itself from its **own** parsed
  prompt (`src/sase/axe/run_agent_runner_launch.py`), so the successor never sets it.
- `preserved_agent_metadata` (`src/sase/axe/run_agent_directive_metadata.py`) only
  covers same-shell runner re-execs and deliberately re-derives directives from the
  original prompt; it does not apply to a follow-up spawn.
- `spawn_family_successor` forwards only family-attach and delivery env.

At plan-propose time the successor's `create_plan_gate_shell`
(`src/sase/plan_shell/create.py`) calls `get_auto_plan_approval_action()`
(`src/sase/main/plan_approve_handler.py`), which reads only env vars and the current
shell's `agent_meta.json` — both empty of auto state — so `auto_enabled` is false and
the plan gates.

## Fix

1. **`src/sase/monitor/continuation_delivery.py`** — add an `auto_launch_prefix` helper
   directly below `queue_launch_prefix`, mirroring its shape:

   ```python
   def auto_launch_prefix(meta: Mapping[str, Any]) -> str:
       """Return a live ``%auto`` prefix carrying the starter's auto-approve state."""
   ```

   Resolution order (first match wins):
   - `auto_approve_argument` present as a non-empty string → `f"%auto:{argument}\n"`.
     Re-authoring the raw argument verbatim round-trips `%auto:tale`, `%auto:epic`, and
     the compat aliases (`plan`, `epic_plan`) while preserving the tier-adapter
     validation in `validate_plan_auto_argument` (`src/sase/_plan_gate_metadata.py`).
   - `auto_approve_plan_action` equal to `"tale"` or `"epic"` → `f"%auto:{action}\n"`
     (defensive: normally the argument is present whenever the action is).
   - truthy `approve` → `"%auto\n"` (bare `%auto`, including ACE/CLI launches that set
     `approve` in meta without a prompt directive).
   - otherwise `""`.

   Export it in the module's `__all__`.

2. **`src/sase/monitor/followup.py`** — import `auto_launch_prefix` alongside
   `queue_launch_prefix` and prepend it in `_compose`:

   ```python
   prefix = queue_launch_prefix(meta) + auto_launch_prefix(meta)
   ```

   `_compose` is the single composition point inside `launch_followup_agent`, so every
   settlement caller (supervisor, host completion, resume) gets the prefix, including
   the persisted not-launchable recovery prompt.

The re-authored directive flows through the successor runner's ordinary directive
parsing, so the successor's `agent_meta.json` keys, its own `SASE_AGENT_AUTO_APPROVE`
env, and plan-gate auto-settle behavior all re-derive consistently — the exact mechanism
`%queue`, `%model`, and `%effort` already use. Propagation is unconditional (no "already
consumed" check): within a single shell the auto-approve state stays active for the
whole run today, and a monitor continuation is a mechanical continuation of the same
logical agent (decision `single-turn-agents`), so family-lifetime scope matches existing
semantics.

## Tests

1. **`tests/monitor/test_continuation_delivery.py`** — unit tests beside the existing
   `queue_launch_prefix` tests:
   - `{"approve": True}` → `"%auto\n"`.
   - `{"approve": True, "auto_approve_plan_action": "tale", "auto_approve_argument": "tale"}`
     → `"%auto:tale\n"` (argument wins).
   - `{"auto_approve_plan_action": "epic"}` → `"%auto:epic\n"` (defensive branch).
   - `{}` and `{"approve": False}` → `""`.
   - non-string / whitespace-only `auto_approve_argument` falls through rather than
     emitting a malformed directive.
2. **`tests/monitor/test_monitor_followup.py`** — integration coverage in the style of
   `test_launch_followup_agent_uses_explicit_next_model` (asserting on the captured
   spawn prompt):
   - starter meta with `approve: True` plus `auto_approve_argument: "tale"` → the
     captured follow-up prompt contains the `%auto:tale` line alongside the existing
     fork/model prefixes.
   - starter meta without auto state → `%auto` absent from the captured prompt (guards
     against unconditional emission).

## Verification

- Run `just check` (the agent-default verification lane; `just check-full` remains the
  landing gate per decision `two-speed-verification`).
- Read the `lint_and_test` reference memory before finishing, as required for any
  tracked-file change in this repo.

## Alternatives considered

- **Forward `SASE_AGENT_AUTO_APPROVE*` env through `spawn_family_successor`
  `extra_env`**: rejected — it would bypass directive parsing, leaving the successor's
  `agent_meta.json` inconsistent with its effective state, and diverges from the
  established `%queue`/`%model`/`%effort` prompt re-authoring precedent.
- **Add auto keys to `preserved_agent_metadata`**: rejected — that helper is scoped to
  same-shell runner re-execs, which already re-derive `%auto` from the original prompt;
  follow-up spawns never consult it.
- **Freeze auto state into the monitor continuation intent**: unnecessary — the member
  meta is itself a frozen copy of the starter meta at monitor creation, which is the
  same source `queue_launch_prefix` already reads at settlement.

## Non-goals

- Gate-shell and pipe continuation launches have the same potential gap but their
  follow-up prompts are agent-authored per branch, so intended semantics need a separate
  decision; that audit is filed as task bead `sase-11g` and epic `sase-kp` carries the
  discovered-issue evidence for this bug.
- No feature flag: this restores the documented `%auto` contract ("auto-approve next
  plan") for monitor continuations; the prior gating was the defect.
