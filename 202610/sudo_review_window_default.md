---
tier: tale
title: Give sudo gates a one-day review window by default
goal:
  An unanswered `sase sudo request` stays approvable for a day instead of killing its
  agent session after five minutes.
size: small
decisions:
  review_window:
    ask: How long should an unanswered sudo request stay open by default?
    choices:
      day: One day (86400s), matching every other gate-turn gate and the sudo cap
      four_hours:
        Four hours (14400s); stale root commands expire sooner, but overnight phases
        still die
    default: day
    why:
      Approval already needs Bryan at a PAM terminal, so the window must outlast an
      absence
    answer: day
proposed_by: bbugyi200.athena.112
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-17 13:25:13
status: wip
---

# Give sudo gates a one-day review window by default

## Why

The `bob-cli-8v.1` and `bob-cli-8v.2` epic phases (bob-cli project, `%auto`) both died
on 2026-10-17 even though nothing was wrong with their work:

- Each agent did its non-root prep, then submitted one `sase sudo request` (Gate 1b for
  8v.1: enable Alertmanager, reload Prometheus, fire a test alert; an install batch for
  8v.2). Neither request set `gate_timeout_seconds`. The `sase_sudo` skill never
  mentions that field.
- `normalize_sudo_request` in `src/sase/sudo/models.py` defaults the review window to
  **300 seconds**. Every other gate-turn gate defaults to
  `GATE_TURN_DEFAULT_TIMEOUT_SECONDS` (24h, in
  `src/sase/notification_gates/model_turn.py`). Plain `sase gate create` gates with no
  turn block wait until answered.
- Bryan was away from a terminal. `gate_turn_reclaim` cancelled both gates with
  `reason: timeout`, and their gate turns settled as `timeout`. The unanswered axis
  launches no follow-up by design, so both agent sessions ended as failed. The epic's
  `bob-cli-8v.land` agent is now stuck on a "wait dependency can never self-resolve"
  notification.
- This keeps happening: 9 of the 22 sudo requests on athena were cancelled as timed out.
  The approved ones took a median of about two minutes, but three took 12–40 minutes and
  got through only because they were accepted after their window had already passed.

A second problem makes this worse. The review window also falls back to the top-level
`timeout_seconds` / `timeout` keys, which are documented as the **default per-command
timeout**. So an agent that writes `"timeout_seconds": 30` to bound each command also
gets a 30-second approval window. The 60s and 180s windows on past athena requests look
like this mistake.

## Changes

### 1. Normalization (`src/sase/sudo/models.py`)

- In `normalize_sudo_request`, read the review window only from `gate_timeout_seconds`.
  When it is missing, default it to the one-day window:
  `int(GATE_TURN_DEFAULT_TIMEOUT_SECONDS)`, imported from
  `sase.notification_gates.model_turn` (watch for import cycles; a module-level
  `SUDO_DEFAULT_GATE_TIMEOUT_SECONDS = 24 * 60 * 60` constant is fine if the import
  would cycle). Under `review_window = four_hours`, use `4 * 60 * 60` instead and keep a
  local named constant.
- Top-level `timeout_seconds` / `timeout` stay accepted, but they now only set the
  default per-command timeout (`default_timeout`). They no longer feed
  `gate_timeout_seconds`.
- Keep the existing `_timeout` validation: a positive integer of at most one day.
- No `sase-core` change is needed. The review window is not part of the sudo manifest or
  its hash, and normalization lives entirely in this Python module.
- No feature flag is needed. This is a changed default, not a deprecation. The old
  behavior is still available by passing `gate_timeout_seconds` explicitly.
- Stored pending gates are unaffected. `SudoRequest.to_dict()` always writes an explicit
  `gate_timeout_seconds`, so when `kind_validation/sudo.py` re-normalizes a stored
  payload it gets the same value back.

### 2. Tests (`tests/test_sudo_gate_request.py`)

Add focused coverage through `normalize_sudo_request` and `build_sudo_gate_request`:

- Omitting `gate_timeout_seconds` gives the default window, both on
  `SudoRequest.gate_timeout_seconds` and on the envelope's top-level
  `gate_timeout_seconds`.
- A top-level `timeout_seconds: 30` (and separately `timeout: 30`) makes commands
  without their own timeout get `timeout_seconds == 30.0`, while the envelope window
  stays at the default.
- An explicit `gate_timeout_seconds: 600` is honored.
- `gate_timeout_seconds` of `0`, `86401`, or a float/bool is still rejected with
  `invalid_timeout`.

The shared helpers in `tests/_sudo_gate_helpers.py` and
`tests/_sudo_acceptance_helpers.py` set a top-level `timeout_seconds: 30`. Run the whole
`tests/test_sudo_*.py` set plus `tests/ace/tui/test_notification_sudo.py`. Fix any
assertion that silently relied on the old coupling by setting `gate_timeout_seconds`
explicitly in that test. Do not change the helpers' meaning.

### 3. Reference docs (`docs/reference/automation/sudo.md`)

This changes the behavior of an existing feature, so update its Reference page only:

- Agent Contract table: the `gate_timeout_seconds` default becomes `86400` (one day) (or
  `14400` under `four_hours`). The `timeout_seconds` row becomes just "Default
  per-command timeout" and loses the "also the review window…" clause.
- Replace the "The review window is short by default…" paragraph. The new text says the
  window defaults to one day, that an unanswered request then settles as timed out and
  launches no follow-up (the agent session ends), and that agents should set a shorter
  `gate_timeout_seconds` only when the commands become unsafe to run later.

### 4. Agent skill source (`src/sase/macros/skills/sase_sudo.md`)

Add one bullet under **Rules** so agents know the field exists and what timing out
costs:

- The request stays reviewable for `gate_timeout_seconds` (default one day). If nobody
  approves in that window, the gate times out, no successor runs, and the session ends.
  Set a shorter window only when the batch would be unsafe to run later. Top-level
  `timeout_seconds` is only the default per-command timeout; it never shortens the
  review window.

Do not hand-edit generated provider `SKILL.md` copies. Per the generated-skills
contract, Bryan redeploys with `sase skill init --force` from the landed tree.

## Verification

- Read the `lint_and_test` reference memory and run the recipe it prescribes
  (`just check` through `sase tool run`).
- Targeted run: `tests/test_sudo_gate_request.py`, the rest of `tests/test_sudo_*.py`,
  and `tests/ace/tui/test_notification_sudo.py`.

## Out of scope

- Launching a follow-up turn when a sudo gate times out. The "unanswered gates never
  silently spawn agents" policy in `gate_turn/followup_policy.py` stays as it is. With a
  one-day window, timeouts should be rare.
- Recovering the two dead `bob-cli-8v` phases and the blocked land agent. That is an
  operator action and is reported separately.
