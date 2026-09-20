---
tier: tale
title: Stop the recurring service-host SSH credential axe error flood
goal:
  "A dead SSH agent in the service host no longer crashes bead-writing chops on every
  tick: the failure is classified as a credential failure, degrades into the mirror's
  existing backoff, and is reported by `sase service status` for the environment the
  service actually runs with."
size: medium
proposed_by: bbugyi200.athena.0ny
create_time: 2026-09-20 08:30:57
status: wip
---

# Stop the recurring `Permission denied (publickey)` axe error flood

## Problem

`~/.sase/axe/error_digests/` has carried the same two errors every hour since
2026-09-19: `external_issue_mirror[actstat]` and `external_issue_mirror[bob-cli]` both
die with

```
sase.workspace_provider._lease_model.OperationalLeaseError: operational workspace
lease failed during preparation: git@ssh.github.com: Permission denied (publickey).
```

`toobig_split[sase]` fails the same way through typed admission (`condition_error`),
intermittently.

## Root cause (verified on athena, 2026-09-20)

Five links, each confirmed against the live machine:

1. Every host chop that writes beads takes an operational workspace lease.
   `acquire_operational_lease` (`src/sase/workspace_provider/lease.py:102`) calls
   `prepare_from_primary_remote`, which runs `git fetch origin` against
   `git@github.com:sase-org/sase.git` over SSH.

2. Chops inherit the service host's environment. The whole tree (`sase service run` →
   `sase scheduler run` → `sase axe routine run <routine>` → chop) runs with
   `SSH_AUTH_SOCK=/run/user/1000/openssh_agent`, inherited from the systemd **user
   manager** environment. Verified by reading `/proc/<pid>/environ` for every process in
   the `sase.service` cgroup.

3. That agent is a **live socket holding no identities**:
   `SSH_AUTH_SOCK=/run/user/1000/openssh_agent ssh-add -l` →
   `The agent has no identities.` The user's real keys live in a login-session agent at
   `/tmp/ssh-*/agent.<pid>`, which every interactive process and SASE agent has.

4. The captured service environment `~/.sase/service/env` (written 2026-09-19 10:41,
   before `bbca06d9e` added SSH-agent capture) holds only `GEMINI_API_KEY` and `PATH`.
   It therefore does not override the inherited empty agent.

5. `~/.ssh/config`'s `Host github.com` block declares no `IdentityFile`, and
   `~/.ssh/id_rsa` — the key GitHub accepts, confirmed via `ssh -v`
   (`Server accepts key: ... RSA SHA256:1QjOXAh4...`) — is passphrase-protected. With an
   empty agent there is no usable credential at all.

Reproduced directly:

```bash
SSH_AUTH_SOCK=/run/user/1000/openssh_agent GIT_TERMINAL_PROMPT=0 \
  git -C <primary-checkout> ls-remote origin -h
# git@ssh.github.com: Permission denied (publickey).
```

### Why it recurs forever instead of backing off

`run_issue_mirror_for_project` already has exponential backoff: a provider listing
failure calls `next_backoff`, persists `state.failures` / `state.next_attempt_at`, and
later ticks short-circuit with `degraded="backoff"`
(`src/sase/external_mirror/issues.py:129-150`).

The lease failure happens **after** that, in the write stage
(`_apply_issue_mirror_for_source`, `src/sase/external_mirror/issues.py:318`), and
`OperationalLeaseError` escapes the whole function as an unhandled exception. The mirror
state file is never updated, `failures` stays `0`, backoff never engages, and the chop
crashes identically on every scheduled tick.

### Why nothing warned

`sase service status` does print `service_init_plan(force=True).warnings`, and
`ssh_agent_readiness_warnings` (`src/sase/service/ssh_agent.py`) already detects an
agent that is reachable but empty. But `readiness_warnings` is handed `desired_env` —
the env **freshly captured from the calling shell**
(`src/sase/service/platform.py:158,201`). Run from a healthy interactive shell it
reports healthy, no matter what the running service is actually using. Confirmed live:
`sase service status` printed zero warnings while the service host was failing every git
fetch.

`_capture_ssh_agent` (`src/sase/service/env.py`) also only checks
`Path(sock).is_socket()`. An agent with no identities passes that check, and capturing
nothing silently means "inherit whatever the platform manager has" — which on this
machine is exactly the empty agent.

## Operator remediation (human action — cannot be done by an agent)

The code changes below stop the flood and make the condition visible. They cannot
restore the credential: `~/.ssh/id_rsa` is passphrase-protected, so loading it requires
a human. Recommend to the user, in order of durability:

1. **Preferred — dedicated passphrase-free key.** Generate an ed25519 key used only by
   the service host, add it to the GitHub account, and point `Host github.com` at it in
   the chezmoi-managed `~/.ssh/config` with `IdentitiesOnly yes`. Fully unattended,
   survives reboots, no agent needed.
2. **Interim — load the key into the persistent agent.**
   `SSH_AUTH_SOCK=/run/user/1000/openssh_agent ssh-add ~/.ssh/id_rsa`, then
   `systemctl --user restart sase.service`. Survives until the agent restarts; needs the
   passphrase once per boot.
3. **Avoid — `sase service init` from the current login shell.** It captures
   `/tmp/ssh-*/agent.<pid>`, which dies with that login session and silently re-breaks
   the service later.

Do not pick for the user; surface the choice.

## Implementation

Repo-relative paths only. All work is Python-side: the shared classifier this plan
reuses already exists in the Rust core, so no `sase-core` change is needed.

### 1. Classify lease git failures instead of pattern-matching them

`src/sase/workspace_provider/_lease_git.py` hand-rolls a substring test in
`_with_ssh_agent_hint`. The Rust core already classifies exactly this string:
`crates/sase_core/src/retryability/mod.rs` lists `"permission denied (publickey)"` in
`AUTH_MARKERS` and returns a `permanent` verdict with reason `authentication`. It is
reachable from Python today via
`sase.core.retryability_facade.classify_failure_retryability` (already used by
`src/sase/github_cli.py` and `src/sase/sdd/_git.py`).

- Drive the remediation sentence off a `RETRY_OPERATION_GIT` classification rather than
  the ad-hoc `lowered` check. Keep the raw git text verbatim and keep the existing
  wording (tests in `tests/workspace_provider/test_workspace_lease.py` assert
  `SSH_AUTH_SOCK` appears in the message).
- Carry the verdict onto the raised error so callers can branch on it. Add a
  credential/auth marker to `OperationalLeaseError`
  (`src/sase/workspace_provider/_lease_model.py`) — e.g. an optional
  `retryability: RetryabilityVerdictWire | None` field plus an `is_credential_failure`
  property. Keep the rendered message byte-identical;
  `operational_lease_failure_message` shape is a core contract.

### 2. Stop the mirror from crashing, and let backoff engage

In `src/sase/external_mirror/issues.py`, wrap the `source == "chop"` write stage so an
`OperationalLeaseError` becomes a degraded report instead of an escaping exception:

- On a credential-class failure use `degraded="auth_error"`. That reason is already in
  `_CHECK_ERROR_REASONS` (`src/sase/scripts/sase_chop_external_issue_mirror.py:21`), so
  the chop reports `check_error` with a summary rather than a traceback and exit 1.
- On any other lease failure use a new `degraded="lease_unavailable"` reason.
- In both cases feed `next_backoff` and persist `state.failures` /
  `state.next_attempt_at` exactly as the provider-error path does, guarded by
  `not dry_run`. This converts "identical traceback every tick, forever" into a couple
  of attempts followed by exponential backoff capped at `ISSUE_MAX_BACKOFF_SECONDS`.
- Do **not** call `record_tracker_probe` for a lease failure. A dead SSH agent is not a
  tracker outcome, and recording it as one would corrupt the auth-probe record that
  drives tracker health. Check `TrackerProbeOutcome`'s vocabulary in
  `src/sase/external_mirror/auth.py` before deciding where, if anywhere, this belongs.

Apply the same treatment to the PR mirror
(`src/sase/scripts/sase_chop_external_pr_mirror.py` and its `src/sase/external_mirror/`
entry point) if it shares the write-stage lease path — verify before changing; do not
widen beyond what actually shares the seam.

### 3. Report SSH-agent readiness for the service's real environment

The blind spot is that readiness is computed from the caller's environment.

- In `src/sase/service/platform.py`, evaluate SSH-agent readiness against the
  **effective service environment** — the persisted `~/.sase/service/env` overlaid on
  what the unit actually inherits — not `desired_env` captured from the invoking shell.
  `sase service status` must report the failure while the service is failing, even when
  the operator's own shell has a healthy agent.
- Keep `sase service init` reporting readiness for what it is _about to_ capture; those
  are two distinct questions and both are wanted.
- In `src/sase/service/env.py`, make `_capture_ssh_agent` treat "live socket, no
  identities" as unusable, reusing the `ssh-add -l` probe rather than duplicating it
  (factor the probe out of `src/sase/service/ssh_agent.py` if that is cleaner). Phrase
  the warning in terms of the _effective_ agent: capturing nothing means inheriting the
  platform manager's agent, which is the actual failure mode here.

### 4. Tests

- `tests/workspace_provider/test_workspace_lease.py` — credential verdict is attached to
  the raised `OperationalLeaseError`; the existing message assertions still pass; a
  transport-class git failure is **not** marked credential.
- `tests/external_mirror/` — a lease failure in the chop write stage yields a degraded
  report (`auth_error` / `lease_unavailable`), persists an advanced `state.failures` and
  a future `next_attempt_at`, and does not raise; the next tick short-circuits to
  `degraded="backoff"`; `dry_run` persists nothing.
- `tests/service/test_service_environment.py` — an agent that is a live socket with no
  identities is not captured and produces the warning.
- `tests/service/test_service_platform.py` — readiness reflects the effective service
  environment, so a healthy caller env does not mask an unhealthy service env.

### 5. Docs

`docs/init.md` already describes SSH-agent capture (`bbca06d9e`). Extend it with the
empty-agent failure mode and the three remediation options above.

## Verification

```bash
just install    # workspace venvs drift; pinned deps may have moved
just fix
sase tool run check
```

Do not run `just check-full` — not explicitly requested here.

## Out of scope (recorded so it is not re-investigated)

- **`epic_launch_flush` exit `-7`.** Present in the same digest, but it is a one-off: a
  scan of every run record under `~/.sase/axe/lumberjacks/` finds exactly one negative
  exit code in all history, at 2026-09-20T08:14:22. Signal 7 is `SIGBUS`, and
  `~/.local/share/uv/tools/sase/bin` and `uv-receipt.toml` both have mtimes inside the
  08:00–08:30 window — a `uv tool install` replaced the interpreter and venv files while
  that child was starting, which is the textbook mmap-backed `SIGBUS`. The chop ran
  clean at 08:17:15, 08:17:54, and 08:18:33. Transient, not a defect; no change.

- **Whether a bead-store write should need a network fetch at all.**
  `prepare_from_primary_remote` fetches `origin` on _every_ operational lease, so every
  host chop that writes a bead depends on GitHub reachability and SSH credentials.
  Preparing from the local primary checkout — which other machinery already
  fast-forwards roughly every 40 minutes — would decouple bead writes from the network
  entirely and make this whole class of failure impossible. That is a real architectural
  question and probably warrants a decision record, but it is a much larger change than
  this fix and must not be smuggled into it.
