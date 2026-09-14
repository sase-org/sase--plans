---
tier: tale
title:
  Fix the Telegram inbound receiver launch failure and escalate repeated re-arm failures
goal:
  The supervised Telegram long-poll receiver launches reliably from the proc supervisor
  regardless of the host services' PATH, and a receiver that cannot launch surfaces a
  deduped notification instead of silently flooding the proc store while Telegram
  approvals go dead.
size: medium
proposed_by: bbugyi200.athena.0kp
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0kp](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0kp.md)
  - [bbugyi200.athena.0kr](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0kr.md)
  - [bbugyi200.athena.0kv](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0kv.md)
- **COMMITS:**
  - [8586f91](https://github.com/sase-org/sase-telegram/commit/8586f9159ef4577c795e8f3f386879f709e56d11)
    — fix(receiver): anchor telegram proc executables
  - [b547425](https://github.com/sase-org/sase-telegram/commit/b547425f7277957c452cc0f07b9eedbb469c58ff)
    — test(gates): cover sudo generic form
  - [0ec0562](https://github.com/sase-org/sase-telegram/commit/0ec0562cde439741a11c3fcd0099569121579d95)
    — test: isolate sudo gate flag override from env

# Fix The Telegram Inbound Receiver Launch Failure And Escalate Repeated Re-Arm Failures

## Problem

Telegram inbound is completely dead on athena: `~/.sase/telegram/update_offset.txt` has
not advanced since 2026-09-13 ~19:12 EDT, so every button press since then (including
plan-approval presses) is never fetched from Telegram. Outbound delivery still works,
which makes the failure look like "I approved but nothing happened."

Root cause: sase-zr.5 (sase-telegram commit `829e738`) replaced the in-tick polling gap
with one supervised long-poll receiver per bot. `ensure_receiver_running` in
`src/sase_telegram/receiver.py` submits the receiver through
`sase.procs.submit_proc_request` with `argv=["sase_chop_tg_inbound", "--receiver"]` — a
bare console-script name. The proc supervisor spawns argv against the host services'
inherited PATH, which does not include the uv-tool venv bin directory
(`~/.local/share/uv/tools/sase/bin`) — the only place that console script exists. Every
~5-second re-arm therefore dies at spawn:

- Proc rows: `status=error`, `termination_reason=launch-failure`, message
  `could not start command: [Errno 2] No such file or directory: 'sase_chop_tg_inbound'`
  (e.g. proc `fybcqbbeppx1`, log `~/.sase/procs/logs/fybcqbbeppx1.log`).
- Because a failed row is terminal, the per-tick fingerprint replay never matches, so
  each tick submits a fresh doomed row: ~1 failed row per ~7 seconds flooding
  `~/.sase/procs/procs.jsonl`, and no user-visible signal at any point.

Tests cannot catch this by design: `ensure_receiver_running` lets tests substitute a
harmless argv, so the real console-script argv is never exercised anywhere.

A `DISCOVERED ISSUE` note with this evidence is recorded on epic `sase-zr`. No sase-zr
agent is currently running, and sase-zr.6's verification matrix (offset replay,
competing receivers, partial attempts) does not include real-host argv resolution, so
this fix should not wait for the epic.

## Scope

All code changes are in the `sase-telegram` linked repository (open it with
`sase repo open sase-telegram`). This is plugin-internal process orchestration — no
shared backend/domain behavior — so no sase-core work is needed. No new CLI options,
config keys, or memory notes; use module constants for tunables.

### 1. Anchor the receiver argv to the running interpreter

In `src/sase_telegram/receiver.py`, stop submitting the bare console-script name.
Resolve the receiver executable at submission time, in this order:

1. `Path(sys.executable).with_name("sase_chop_tg_inbound")` — the chop tick runs inside
   the installed venv, so the console script is a sibling of the running interpreter in
   every supported install (uv tool and source venv). Use its absolute string when it
   exists and is executable.
2. `shutil.which("sase_chop_tg_inbound")` as a fallback.
3. The bare name only as a last resort (current behavior), so an exotic layout degrades
   to today's behavior instead of a new failure mode.

Keep the `argv` override parameter and its test-substitution behavior. Keep the
fingerprint/concurrency-key semantics unchanged.

Apply the same interpreter-anchored resolution to the gate-answer submission in
`src/sase_telegram/inbound.py` (`submit_gate_response` builds
`argv=["sase", "gate", "answer", ...]`). On athena the bare `sase` happens to resolve
via `~/.local/bin`, but that is luck of the inherited PATH, not a guarantee; anchor it
the same way with the same fallbacks.

### 2. Escalate repeated launch failures and stop the re-arm flood

Give the ensure path (or the tick that calls it) awareness of a receiver that keeps
failing to launch:

- Before re-arming, look up the newest proc row for this receiver's
  `request_fingerprint`. If it is terminal with a launch failure
  (`termination_reason == "launch-failure"`), apply a backoff: skip re-arm unless that
  row is older than a module-constant interval (suggest 300 seconds). This bounds
  proc-store growth to a handful of rows per hour instead of one per tick while still
  self-healing once the cause is fixed.
- On detecting a launch-failed receiver row, create a SASE notification (deduped via a
  stable `dedup_key`, e.g. `telegram-receiver-launch-failure`) stating that the Telegram
  inbound receiver cannot start, including the proc row's message and log path. One
  durable, visible signal — not one per tick. A 17-hour silent outage discovered only
  through a dead approval press is itself part of this bug.

Implementation latitude: use whichever existing `sase.procs` query API and
notification-creation API sase-telegram already has access to; if the proc store offers
no cheap newest-row-by-fingerprint lookup, an on-disk last-failure stamp under
`~/.sase/telegram/` is an acceptable alternative for the backoff, but the notification
must still carry the real proc error message.

### 3. Regression tests that exercise the real argv

- A test that the default (non-substituted) receiver argv resolves to an absolute path
  of an existing executable in the test environment. The package under test installs the
  `sase_chop_tg_inbound` entry point into its own venv, so this test fails on any future
  regression to a bare or missing name.
- Unit tests for the resolver's fallback order (interpreter sibling → `which` → bare
  name) using monkeypatched filesystem/lookup results.
- Tests for the escalation path: a launch-failed newest row triggers exactly one deduped
  notification and suppresses re-arm inside the backoff window; a healthy or absent row
  re-arms exactly as today.
- Keep existing supervision tests (harmless-argv substitution) unchanged.

## Verification

- `just check` in sase-telegram.
- Follow the sase repo's `lint_and_test` memory obligations for any file changed in a
  repo tracked by git this turn.
- Host verification after the fixed plugin deploys (athena's install is an editable
  install of the host checkout, and the still-ticking chop re-arms every ~5 seconds, so
  no service restart is needed):
  1. One proc row labeled "Telegram inbound long-poll receiver" reaches RUNNING and
     stays up; launch-failure rows stop accumulating in `~/.sase/procs/procs.jsonl`.
  2. `~/.sase/telegram/update_offset.txt` advances after sending the bot any message or
     button press.
  3. A pending gate answered from Telegram resolves (its notification dismisses and the
     gate leaves pending state).

## Out Of Scope

- Telegram resume/restart affordances for `partial_attempt` gate errors and the broader
  approval-latency verification — that is sase-zr.6's remit; the epic bead carries the
  incident evidence.
- Recovering the currently wedged plan gate `plan/c7d980c0` (host state, not code): any
  non-identical resubmission — e.g. approval from ACE with real inputs — already
  supersedes the stale empty-input attempt.
- `sase-ve` (source installs missing the Telegram chop console scripts entirely) —
  related packaging defect, different root cause.
- Changing the proc supervisor's argv/PATH resolution policy in the sase repo.
