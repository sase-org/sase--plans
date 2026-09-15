---
tier: tale
title: Make the ACE sudo terminal handoff honest and unblock stale-flag-pinned answers
goal:
  Authenticate on a pending sudo gate reaches a real /usr/bin/sudo prompt on the
  terminal, and every handoff failure surfaces its actual reason instead of the generic
  toast.
size: medium
proposed_by: bbugyi200.athena.0l9
create_time: 2026-09-15 11:43:52
status: wip
---

# Plan: Make the ACE sudo terminal handoff honest and unblock stale-flag-pinned answers

## Problem

Bryan pressed **Authenticate** on a pending sudo gate
(`sudo-5617dc91-e3a2-4561-ae42-d985f8e8135e`, raised by epic sase-110's phase sase-110.7
worker to tighten athena's sudo policy). Instead of a sudo/PAM password prompt on the
real terminal, ACE restored with the generic toast **"Sudo handoff failed; gate remains
pending"**. `/usr/bin/sudo` was never invoked. The gate is still pending and still
answerable (it has no expiry).

## Root cause (diagnosed and verified, 2026-09-15)

Failure chain, confirmed on the live host:

1. The ACE TUI process started 2026-09-13 16:57. At startup
   `src/sase/main/ace_handler.py:159-166` calls `install_process_feature_flags()`, which
   resolves feature flags and **pins the full resolved set into the process
   environment** as `SASE_FEATURE_FLAGS` (`apply_feature_flags_env`,
   `src/sase/feature_flags/snapshot.py:219-242`,
   `src/sase/feature_flags/env.py:96-101`). At that moment `agent_sudo_requests` (beta,
   default off, flag bead sase-111) resolved to **false**.
2. On 2026-09-15 08:02 the flag was enabled in saved host state
   (`~/.sase/feature_flags.json` now has `"agent_sudo_requests": true`). But in
   `resolve_feature_flags` (`src/sase/feature_flags/resolver.py:117+`) the precedence is
   defaults → config layers → saved state → overrides → **env** → CLI, so the stale env
   pin outranks the new saved state in every child ACE spawns.
3. On Authenticate, ACE ran `sase sudo answer <id> --run --json` via
   `_run_sudo_terminal_handoff`
   (`src/sase/ace/tui/actions/agents/_notification_sudo.py:149-195`), inheriting the
   stale pin. `handle_sudo_command` calls `require_sudo_requests_enabled("sudo")` at its
   very top (`src/sase/sudo/cli.py:41`), **outside** `_answer`'s GateError→`emit_json`
   handling (`src/sase/sudo/cli.py:139-143`), so the CLI exited 1 with a plain-text
   stderr line and **no JSON on stdout**. Reproduced verbatim in a flag-off environment
   with the read-only `sase sudo list --json`:
   `sase sudo: failed [feature_disabled] sudo: sudo requests are behind the agent_sudo_requests beta flag. ...`
   (exit 1, empty stdout).
4. With exit ≠ 0 and no JSON payload, ACE's `_sudo_cli_message`
   (`src/sase/ace/tui/actions/agents/_notification_sudo.py:213-254`) fell through to the
   generic fallback toast at line 254 — hiding the actionable `feature_disabled` reason.
   The epic plan (`plan:202609/agent_sudo_requests.md`, `ace-review` phase) requires an
   "honest toast" here.

**Latent second defect (not reached this time, would block the retry):**
`run_sudo_runner_file` invokes the runner as bare `"sase_sudo_runner"` via PATH lookup
(`src/sase/sudo/runner.py:59-66`). The binary is installed only inside the uv tool venv
(`~/.local/share/uv/tools/sase/bin/sase_sudo_runner`, a console script of the
`sase_core_rs` dependency wheel); uv links only the primary package's scripts into
`~/.local/bin`, so the runner is on **no** PATH entry of the ACE process or a login
shell. The epic plan's `sudo-gate` contract says the runner is "resolved from the
installed venv bin like the federation worker" — the existing precedent is
`Path(sys.executable).parent` in `src/sase/dispatch/federation/_supervisor.py:56`. Once
the flag issue is fixed, an unfixed lookup would fail every local authentication with
`runner_unavailable`.

## Scope

Three targeted fixes in this repo (no sase-core changes; the Rust runner itself is
fine). Do **not** weaken the `agent_sudo_requests` fail-closed gate — the flag is epic
scaffolding that phase sase-110.8 removes; both flag states must stay tested and the Off
branch must stay explicit.

### 1. Resolve `sase_sudo_runner` from the installed venv bin

In `src/sase/sudo/runner.py`:

- Resolve the runner executable by checking
  `Path(sys.executable).parent / "sase_sudo_runner"` first (the installed-venv console
  script beside the running interpreter, mirroring
  `src/sase/dispatch/federation/_supervisor.py:56`), falling back to
  `shutil.which("sase_sudo_runner")`.
- If neither resolves, raise the existing `GateError("runner_unavailable", ...)` with a
  message that names both looked-up locations so the toast/CLI output is actionable.
- This function also backs the target side of remote handoffs (`sase sudo exec` →
  `run_sudo_runner_file`), so the fix covers local and `ssh -t` paths at once.

Tests: venv-bin resolution wins when both exist; PATH fallback works; both-missing
raises `runner_unavailable` naming both locations. Use tmp dirs and monkeypatching;
never invoke real sudo.

### 2. Honor the `--json` error contract for every `sase sudo answer` failure

In `src/sase/sudo/cli.py`:

- When `--json` is passed to `answer`, **every** `GateError` must produce the
  `_error_payload` JSON envelope on stdout plus the documented nonzero exit code —
  including the three currently-leaking pre-`try` sites: the top-level
  `require_sudo_requests_enabled` gate, `_resolve_sudo_gate_id` (use the raw ref as the
  payload's `request_id` when resolution itself fails), and `_decision`'s
  `tty_required`.
- Non-`--json` (human) invocations keep today's loud plain-text failure behavior.
- Extend the same top-level JSON envelope to `list`/`show` with `--json` only if it
  falls out trivially from the same restructuring; `answer` is the required surface (it
  is the one ACE consumes).
- `feature_disabled` maps through `_error_outcome` to `runner_error` today; that is
  acceptable — ACE's `_sudo_cli_message` already renders `code`/`message` suffixes
  ("Sudo handoff failed: sudo requests are behind the agent_sudo_requests beta
  flag..."). Do not invent new outcome vocabulary.

Tests: flag-off + `--json` → exit ≠ 0 with a JSON payload whose `code` is
`feature_disabled` and whose `message` contains the enable hint; unknown gate ref +
`--json` → JSON `not_found` envelope; flag-off human invocation unchanged. Assert the
ACE toast mapping renders the feature-disabled message via `_sudo_cli_message` (pure
function, direct unit test).

### 3. Let the terminal handoff see current host flag state

In `src/sase/ace/tui/actions/agents/_notification_sudo.py`
(`_run_sudo_terminal_handoff`):

- Spawn the `sase sudo answer` child with an environment copy that **drops
  `SASE_FEATURE_FLAGS`** (import the `SASE_FEATURE_FLAGS_ENV` constant from
  `sase.feature_flags`; build
  `env = {k: v for k, v in os.environ.items() if k != SASE_FEATURE_FLAGS_ENV}` for the
  `subprocess.run` call).
- Rationale: the env pin exists so an agent run sees consistent flags for its whole
  turn; a human-initiated terminal handoff is a fresh host action and must honor the
  current saved flag state. A days-old ACE must not silently veto a flag Bryan enabled
  after ACE started. This makes `sase flag enable agent_sudo_requests` effective without
  restarting ACE.
- Keep everything else about the spawn unchanged (stdout pipe, stderr/stdin on the TTY,
  suspend flow). This is not a TUI perf surface: no refresh, render, or
  zero-pending-path work changes.

Tests: monkeypatch `subprocess.run` and assert the child env omits `SASE_FEATURE_FLAGS`
while preserving other variables; existing handoff toast tests stay green.

## Out of scope

- The systemic issue that a long-lived ACE's env pin propagates into _agent runs_ and
  durable procs generally (this run's own agent environment inherited the stale `false`
  pin the same way) — tracked separately as its own task bead; do not expand this tale
  into flag-pinning redesign.
- Any change to `sase_sudo_runner` itself, sudoers policy, or the sase-110.7/110.8 phase
  work. The pending gate stays pending; answering it is Bryan's action after this lands.
- Removing or relaxing the `agent_sudo_requests` flag (phase sase-110.8 owns removal).

## Verification

- Unit/behavior tests above; both flag states covered for the CLI contract.
- `just check` green (read the lint/test memory before finishing; use `/sase_monitor` if
  slow).
- Read the CLI-rules memory before finishing (CLI behavior change, no new
  options/subcommands are added — keep it that way).
- Safe end-to-end repro check without touching the live gate: in a shell with
  `SASE_FEATURE_FLAGS` forced to `{"agent_sudo_requests": false}`,
  `sase sudo list --json` must now emit the JSON error envelope (exit ≠ 0); with the pin
  absent it must list gates normally (saved state is enabled on this host).
- Credential hygiene is absolute (epic constraint): no password/PAM material — nor
  lengths, digests, or attempt counts — in any test fixture, log, or assertion.
- After landing: Bryan retries **Authenticate** on the pending
  `sudo-5617dc91-e3a2-4561-ae42-d985f8e8135e` gate from the still-running ACE; the child
  now resolves the flag from saved state, resolves the runner from the venv bin, and the
  real `/usr/bin/sudo` prompt appears on the terminal. (Manual, human step — do not
  script it and do not answer the live gate from tests.)
