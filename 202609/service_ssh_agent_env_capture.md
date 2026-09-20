---
tier: tale
title: Capture the SSH agent in the service host environment contract
goal:
  Service-driven git work against SSH remotes authenticates, and an empty or unreachable
  agent is reported by `sase service init` instead of surfacing as an opaque
  plan-archive failure.
size: medium
proposed_by: bbugyi200.athena.0nv
create_time: 2026-09-20 07:24:31
status: wip
---

# Give The Service Host A Usable SSH Agent So Telegram Plan Approvals Stop Failing

## Problem

Two approved plan gates failed with an identical terminal error:

- `0nt.f0.f0.1` / gate `94aa2e83` —
  `Tale ready for review: finish_pending_gate_status_refresh.md`
- `0nt.f0.f0.2` / gate `0a04f98b` —
  `Tale ready for review: finish_numbered_notification_tabs.md`

Both error records (`errors/*.json`, `stage: terminal_prepare`,
`code: plan_archive_failed`, `source: telegram`) say:

```
failed to archive approved plan: operational workspace lease failed during preparation:
git@ssh.github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

Both gate shells are still `gate_state: settling`, `status_label: PLAN FAILED`,
`holds_workspace_claim: true`, and both source plan files survive at
`~/.sase/plans/202609/`.

## Root Cause

Confirmed by direct reproduction on athena, not inferred.

1. The approval arrived over Telegram. The Telegram receiver runs as a service proc
   under the systemd user unit `sase.service`, and it detaches the answer as a proc
   (`origin: telegram-gate-detach`, `reserved_by: proc-service:<pid>`) running
   `sase gate answer --id <gate> --kind plan --no-detach --json`.
2. `sase gate answer` reaches `plan_approval_actions._archive_plan_for_approval` →
   `_plan_archive_approval.archive_approved_plan`, which opens an
   `operational_workspace_lease`. `lease.acquire_operational_lease` calls
   `_lease_git.prepare_from_primary_remote`, whose first remote step is
   `git fetch --quiet origin` (`src/sase/workspace_provider/_lease_git.py:20`). `origin`
   is `git@github.com:sase-org/sase.git`, i.e. SSH.
3. That `git fetch` runs with the **service host's** environment. `sase.service`
   inherits `SSH_AUTH_SOCK=/run/user/1000/openssh_agent` from the systemd user manager —
   the socket owned by the systemd `ssh-agent.service` unit. **That agent holds no
   identities** (`ssh-add -l` → `The agent has no identities.`).
4. `~/.ssh/config` gives `Host github.com` only `Hostname ssh.github.com` / `Port 443` /
   `User git` — no `IdentityFile`, and the catch-all `Host *` block explicitly excludes
   `github.com`. So GitHub auth depends entirely on the ambient agent. The on-disk
   default identities (`~/.ssh/id_rsa`, `~/.ssh/id_ed25519`) are both
   passphrase-protected, and `non_interactive_git_env()` sets `SSH_ASKPASS=/bin/false` +
   `SSH_ASKPASS_REQUIRE=force` + `GIT_TERMINAL_PROMPT=0`, so they cannot be unlocked.
5. The key GitHub actually accepts — `id_rsa`,
   `SHA256:1QjOXAh4edlsi2iJ3bnbdTR3d8BY13ZsTU2lg1RmjKA` — is loaded only in the
   **login-session** agent at `/tmp/ssh-*/agent.*`. The interactive shell and the ACE
   TUI process both have that socket; the service host does not.

Reproduction (exact):

```
$ SSH_AUTH_SOCK=/run/user/1000/openssh_agent git ls-remote origin HEAD
git@ssh.github.com: Permission denied (publickey).
$ git ls-remote origin HEAD          # login-session agent
<succeeds>
```

`ssh -v` confirms the asymmetry: with the login agent,
`Server accepts key: ~/.ssh/id_rsa ... agent` → `Authenticated to ssh.github.com`. With
the empty agent, ssh falls back to offering the same key from disk, cannot sign it
non-interactively, and GitHub denies.

So this is **not** a bug in the gate, the plan archive, or the lease. It is a hole in
the service host's captured environment contract:

`src/sase/service/env.py::capture_service_environment` captures a strict allowlist — LLM
provider env vars, `PATH`, `SASE_FEATURE_FLAGS`, the mobile-gateway credential name, and
`SASE_HOME`. It never captures any SSH agent handle. `host_lifecycle.run_host` then
calls `load_service_environment(override_existing=True)`, so whatever the captured file
omits stays at systemd's inherited value — here, the empty agent.

The consequence is systemic, not limited to these two gates: **every** host-owned git
operation the service performs against an SSH remote fails the same way. Gate answers
driven from the ACE TUI work only because the TUI process happens to carry the
login-session socket. Approving from Telegram (or any other service-driven path) is
broken for as long as the service runs with the empty agent, which it still is right
now.

A secondary defect makes this expensive to diagnose: `OperationalLeaseError` surfaces
raw git stderr with no hint about which environment produced it, so
`Permission denied (publickey)` looks like a broken deploy key rather than a missing
agent in one specific process tree.

## Scope And Non-Goals

In scope: the service host environment contract, its readiness reporting, and the
lease's error message.

Explicitly **not** in scope:

- Rewriting `~/.ssh/config` or minting a new GitHub key. That is a host/chezmoi decision
  for the user, and this plan deliberately makes SASE _report_ the problem rather than
  silently pick an identity.
- Re-driving the two stuck gates. `244442ee8` already shipped plan-gate
  `partial_attempt` retry plus resume/restart/cancel actions on `GateExecutionFailed`
  notifications; both plan files survive, so the user retries from ACE after the host is
  fixed. Step 5 only documents that.
- Converting `origin` to HTTPS or routing git through `gh` credentials.

## Implementation

### 1. Capture the SSH agent handle in the service environment contract

File: `src/sase/service/env.py`

In `capture_service_environment`, capture `SSH_AUTH_SOCK` and `SSH_AGENT_PID`, but
**validate before capturing** — a stale socket path is worse than none, because it would
override a possibly-working inherited value with a guaranteed-dead one.

- Read `SSH_AUTH_SOCK` from `environment`.
- Capture it only when the value is non-empty **and** the path exists and is a socket
  (`Path(value).is_socket()`). Guard the stat in `try/except OSError` and treat any
  error as "not capturable".
- Capture `SSH_AGENT_PID` only when `SSH_AUTH_SOCK` was captured and `SSH_AGENT_PID` is
  present (it is absent for systemd- and keyring-provided agents; that is fine and must
  not warn).
- When `SSH_AUTH_SOCK` is set but not a live socket, return a
  `CapturedServiceEnvironment.warnings` entry naming the rejected path and saying the
  service host will fall back to whatever agent the platform manager provides.
- When `SSH_AUTH_SOCK` is unset entirely, return a warning that the service host will
  have no SSH agent and git operations against SSH remotes will fail with
  `Permission denied (publickey)`.

This is the first code to populate `CapturedServiceEnvironment.warnings`. The field
already exists and `service_init_plan` already seeds its warning list from
`capture.warnings` (`src/sase/service/platform.py:163`), so no plumbing is needed — but
confirm that path is live rather than assuming it.

Keep `render_service_environment` / `parse_service_environment_text` untouched: both
names are plain `[A-Za-z_][A-Za-z0-9_]*` and the values are JSON-encoded strings, so the
existing format already handles them.

### 2. Report agent health as a readiness warning

File: `src/sase/service/platform.py`, function `readiness_warnings(env)`

`readiness_warnings` is already called by `service_init_plan` and its output already
reaches the user through `sase service init`, `--check`, and `--diff`
(`src/sase/main/service_handler.py:118,173` and
`src/sase/main/init_service_handler.py:127`). Add a git-remote-auth check there so the
broken state is visible _before_ an approval fails:

- If `env` has no `SSH_AUTH_SOCK`, warn that the captured service environment carries no
  SSH agent, so plan archival, stitch pushes, and any other host-owned git work against
  an SSH remote will fail with `Permission denied (publickey)`; name `sase service init`
  (run from a shell whose agent holds the key) as the remedy.
- If `env` has `SSH_AUTH_SOCK`, probe it: run `ssh-add -l` with that env,
  `capture_output=True`, `stdin=subprocess.DEVNULL`, and a short timeout (2 seconds is
  enough for a unix-socket round trip). Interpret the documented exit codes — `0`
  healthy, `1` reachable but **no identities**, `2` **unreachable**. Warn distinctly for
  the last two, quoting the socket path.
- Wrap the whole probe in `try/except Exception` and degrade to a single "SSH agent
  readiness could not be checked: {exc}" warning, matching how the existing provider-CLI
  block already handles `collect_agent_cli_statuses` failure. A readiness probe must
  never make `sase service init` raise.
- Resolve `ssh-add` through the captured `PATH` with the existing `_resolve_command`
  helper; if it is not resolvable, skip the probe silently rather than warning about a
  missing `ssh-add` (that is noise, not a SASE problem).

Place this after the existing mobile-gateway block and before the `SASE_FEATURE_FLAGS`
drift check, so agent problems read next to the other environment-drift warnings.

### 3. Make the lease's auth failure self-explanatory

File: `src/sase/workspace_provider/_lease_git.py`

`prepare_from_primary_remote` raises `OperationalLeaseError("preparation", detail)` with
raw git stderr for both the fetch and the checkout step. Add a small helper that, when
`detail` matches an SSH public-key denial (case-insensitively:
`permission denied (publickey)`, or `could not read from remote repository` together
with `publickey`), appends one remediation sentence to the preserved original text.

The sentence must state that the failure is usually a missing or empty SSH agent in the
calling process — naming `SSH_AUTH_SOCK`, and `sase service init` when the caller is the
service host — so a future reader does not go hunting for a revoked deploy key. Do not
swallow or reformat the git output; append only.

Keep the helper local to the module and cover both raise sites (fetch and checkout).

### 4. Tests

- `tests/service/test_service_environment.py`
  - `SSH_AUTH_SOCK` pointing at a real socket (bind a `socket.AF_UNIX` socket under
    `tmp_path`) is captured; `SSH_AGENT_PID` rides along when present.
  - `SSH_AUTH_SOCK` pointing at a nonexistent path is **not** captured and yields a
    warning naming the path.
  - `SSH_AUTH_SOCK` absent yields the no-agent warning and no `SSH_AUTH_SOCK` key.
  - `SSH_AGENT_PID` alone, with no usable `SSH_AUTH_SOCK`, is not captured.
  - Check the existing
    `test_capture_service_environment_uses_provider_and_gateway_declared_names` still
    holds; its `environ` has no SSH vars, so its exact-`values` assertion should be
    unaffected — confirm rather than assume, and if the new warning changes what it
    observes, assert the warning explicitly instead of loosening the value assertion.

- `tests/service/test_service_platform.py`
  - `readiness_warnings` warns when `env` has no `SSH_AUTH_SOCK`.
  - With `SSH_AUTH_SOCK` set, stub the `ssh-add` probe and assert distinct warnings for
    exit `1` (no identities) and exit `2` (unreachable), and **no** SSH warning for exit
    `0`.
  - A probe that raises degrades to the "could not be checked" warning and does not
    propagate.

- `tests/workspace_provider/test_workspace_lease.py`
  - A fetch whose stderr is the real
    `git@ssh.github.com: Permission denied (publickey).\nfatal: Could not read from remote repository.`
    text produces an `OperationalLeaseError` whose message still contains the git text
    **and** the new remediation sentence.
  - An unrelated fetch failure (e.g. `fatal: couldn't find remote ref`) does **not**
    gain the remediation sentence.

Follow the existing style in each file (`monkeypatch`, `tmp_path`, no network).

### 5. Document the contract and the host remedy

File: `docs/init.md`

The `sase service init` rows in the Commands table describe captured-environment drift
but never say what the captured environment contains. Add a short paragraph near the
service section (around the `sase service init --check` block at line 110) stating that
the captured environment includes the SSH agent handle, that it is captured from the
shell that runs `sase service init`, and that a login-session agent socket dies with
that session — so `sase service init --check` should be re-run and the service restarted
after a reboot or a new login session. Mention that `--check` now reports an empty or
unreachable agent.

Keep it to a paragraph; this is a behavior note, not a new doc section.

## Applying The Fix On athena (report, do not perform)

The code change alone does not repair the running host, and the coder agent must **not**
mutate the live service. After `just check` passes, report these steps for the user to
run themselves, and state plainly that they were not executed:

1. From an interactive shell whose agent holds the GitHub key (verify with
   `ssh-add -l`), run `sase service init --check`, then `sase service init --diff`, then
   apply.
2. `sase service restart`, then confirm the host picked up the socket:
   `tr '\0' '\n' < /proc/$(pgrep -f 'sase service run')/environ | grep SSH_AUTH_SOCK`
3. Retry the two failed gates from ACE via the `GateExecutionFailed` recovery actions.
4. Note the durability caveat honestly: capturing a `/tmp/ssh-*/agent.*` socket works
   only while that login session lives. The permanent fix is a host-level decision the
   user owns — either load the GitHub key into the systemd `ssh-agent.service` agent at
   `/run/user/1000/openssh_agent`, or give `Host github.com` an `IdentityFile` in
   `~/.ssh/config` (chezmoi-managed via `~/Sync/home/.ssh/config`). Recommend the
   `IdentityFile` route, since it survives reboots and needs no agent at all. Do not
   make that change as part of this tale.

## Verification

```bash
just install    # this workspace may be stale
just fix        # or at minimum: just fmt
sase tool run check
```

`just check` is the whole recipe. Do not run `just check-full`; no TUI rendering changes
here, so no visual snapshot work is needed.

## Acceptance Criteria

1. `capture_service_environment` captures `SSH_AUTH_SOCK` only when it names a live
   socket, captures `SSH_AGENT_PID` only alongside it, and emits a `warnings` entry for
   the missing and stale cases.
2. `readiness_warnings` distinguishes "no agent captured", "agent unreachable", and
   "agent has no identities", and never raises.
3. An SSH public-key denial from the lease's git preparation carries both the original
   git text and a remediation sentence naming `SSH_AUTH_SOCK`.
4. `docs/init.md` records that the captured service environment includes the SSH agent
   handle and is session-scoped.
5. `sase tool run check` passes.
6. The final report states the host steps in "Applying The Fix On athena" were **not**
   performed, and names the two gates the user can retry.
