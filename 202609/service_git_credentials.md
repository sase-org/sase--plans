---
tier: tale
title: Give the service host a durable git credential and a truthful readiness check
goal:
  The service host authenticates to GitHub unattended, and SASE decides SSH readiness
  from real git-remote authentication instead of agent identity counts, so plan-approval
  gates stop failing with `Permission denied (publickey)`.
size: medium
proposed_by: bbugyi200.athena.0nv.f0
create_time: 2026-09-20 09:52:02
status: wip
---

# Give the service host a durable git credential and a truthful readiness check

## Problem

Approving a plan gate fails at `terminal_prepare` with `Permission denied (publickey)`.
Four gates are now stuck this way (`sase-11y.7.f1`, `sase-133.5.2.f0`, `0nt.f0.f0.1`,
`0nt.f0.f0.2`).

Commit `6087c0a8e` added a remediation hint and an effective-environment warning for
exactly this failure. Those diagnostics work -- `sase service status` prints the correct
cause right now -- but nothing about the host changed, so the failure repeats unchanged.

## Root cause (reproduced)

The service host authenticates to GitHub with nothing.

1. `~/.sase/service/env` contains only `GEMINI_API_KEY` and `PATH`. It has no
   `SSH_AUTH_SOCK`, and it is dated before `6087c0a8e`, so `sase service init` has not
   been re-run since the fix landed.
2. `sase.service` therefore inherits `SSH_AUTH_SOCK` from the systemd user manager,
   which is `/run/user/1000/openssh_agent`.
3. That agent is live but holds **no identities**:
   `SSH_AUTH_SOCK=/run/user/1000/openssh_agent ssh-add -l` ->
   `The agent has no identities.` (exit 1).
4. Every host-owned git operation over SSH then fails:
   `SSH_AUTH_SOCK=/run/user/1000/openssh_agent ssh -T git@github.com` ->
   `git@ssh.github.com: Permission denied (publickey).` -- byte-for-byte the gate's
   error.

The interactive shell's agent (`/tmp/ssh-bd1YBG80VDga/agent.13572`) does hold the key
GitHub accepts (`id_rsa`, `SHA256:1QjOXAh4...`), which is why the same git commands
succeed by hand and fail in the service.

### Why the previous fix could not hold

Re-running `sase service init` would capture the interactive shell's
`/tmp/ssh-*/agent.*` socket. That socket belongs to a login session and dies with it,
after which the host silently falls back to the empty systemd agent and this failure
returns. The previous plan's suggested alternative -- an `IdentityFile` for
`Host github.com` -- was sound in shape but not as stated: `~/.ssh/id_rsa` and
`~/.ssh/id_ed25519` are both passphrase-protected, so neither can be used unattended.

### The second, quieter bug

`ssh_agent.py` decides readiness from `ssh-add -l`, i.e. from how many identities an
agent holds. That is not what failed. Authentication to the git remote is what failed,
and the two disagree in both directions:

- Verified: with `IdentitiesOnly`/`IdentityFile` and the **empty** systemd agent,
  `ssh -T git@github.com` authenticates successfully while `ssh-add -l` still reports
  `The agent has no identities` (exit 1).

So once the host is fixed the way this plan fixes it, today's code would both refuse to
capture the environment (`_capture_ssh_agent` returns `{}` on state `empty`) and warn
forever in `sase service status` about a host whose git access is perfectly healthy. The
check must measure the thing that breaks.

## Approach

Fix the host so the service has an unattended credential, then make SASE's readiness
check assert real remote authentication instead of agent contents.

Host remediation is **user-owned** and written out in "Host steps" below: it creates a
key and registers it on the user's GitHub account, which an agent must not do on its
own. The implementing agent does steps 1-4 only.

## Implementation steps

### 1. Probe git remote authentication, not agent contents

In `src/sase/service/ssh_agent.py`:

- Add a probe that asks whether the environment can authenticate to the git remote, by
  running `ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new -T git@github.com`
  and classifying the result. GitHub exits `1` with `successfully authenticated` on
  success and prints `Permission denied (publickey)` on failure; treat any other outcome
  (timeout, DNS, network) as `unknown`, never as failure.
- Keep the existing `ssh-add -l` probe, but demote it to explanatory detail used only to
  word the warning, never to decide readiness.
- Drive readiness from the remote probe: readiness is `ready` when the remote accepts,
  `denied` when it answers `Permission denied (publickey)`, and `unknown` otherwise.
  Only `denied` warns. `unknown` stays silent, because an offline host must not be
  reported as a credential failure.
- Give the probe a short timeout (the existing 2s is too tight for a network round trip;
  use ~5s) and never let it raise into a caller.

Keep both `scope="captured"` and `scope="effective"` wordings.

### 2. Stop refusing to capture a working environment

In `src/sase/service/env.py`, `_capture_ssh_agent` must no longer treat an empty or
unreachable agent as a reason to capture nothing when the environment can still
authenticate. Capture `SSH_AUTH_SOCK` when it names a live socket; base any warning on
the step-1 remote probe. A host that authenticates by `IdentityFile` with no agent at
all must capture cleanly and warn about nothing.

### 3. Preflight the credential before accepting a plan approval

In the plan-approval gate path (`src/sase/plan_approval_actions.py`, around the two
`plan_archive_failed` sites at lines ~519 and ~694), run the step-1 remote probe before
the decision is accepted. On `denied`, refuse the answer with the remediation text
instead of accepting and then failing in `terminal_prepare`. `unknown` must not block.
This turns "approved, then failed after acceptance" into "refused before acceptance,
with the reason".

### 4. Tests and docs

- `tests/service/`: remote probe returns `ready` / `denied` / `unknown` for
  authenticated, `Permission denied (publickey)`, and timeout/network results; `unknown`
  and `ready` emit no warning; a no-agent `IdentityFile`-style environment captures
  cleanly and warns about nothing (the regression this plan exists to prevent).
- Plan-approval tests: a `denied` preflight refuses the answer and does not reach
  archival; an `unknown` preflight proceeds.
- `docs/init.md`: replace the empty-agent framing with the two supported ways to give
  the host an unattended credential (dedicated passphrase-less key via `IdentityFile`,
  or a key loaded into the systemd user agent), and note that an `/tmp/ssh-*/agent.*`
  socket captured from a login shell is not durable.

## Host steps (user-owned, not for the implementing agent)

**Immediate unblock** (survives logout -- `Linger=yes` -- but not reboot). Load the
GitHub key into the _systemd_ agent, which is the one the service actually inherits;
this prompts for the passphrase:

```
SSH_AUTH_SOCK=/run/user/1000/openssh_agent ssh-add ~/.ssh/id_rsa
```

**Durable fix.** Give the host a dedicated passphrase-less key, the pattern already
proven on this machine by `id_bob_vault`/`bob-vault-sync`:

```
ssh-keygen -t ed25519 -N "" -C "sase-service@athena" -f ~/.ssh/id_sase_service
```

Register `~/.ssh/id_sase_service.pub` on the GitHub account (this is the step an agent
must not take unilaterally), then add to `Host github.com` in `~/Sync/home/.ssh/config`
-- a plain symlinked file, **not** chezmoi-managed (only `.ssh/tailnet.conf` is):

```
IdentityFile ~/.ssh/id_sase_service
IdentitiesOnly yes
```

Verify against the empty agent, which is what the service sees:

```
SSH_AUTH_SOCK=/run/user/1000/openssh_agent ssh -T git@github.com
```

Then `sase service init`, `sase service restart`, and confirm `sase service status`
reports no SSH warning.

**Recover the four stuck gates** with their `resume` recovery action, e.g.:

```
sase gate answer --kind plan --id e0d3c083-d47f-4ce9-a801-df73113651d3 \
  --option approve --option commit --resume
```

## Acceptance criteria

1. Readiness is decided by git-remote authentication; `ssh-add -l` no longer decides it.
2. An environment that authenticates by `IdentityFile` with an empty agent captures
   cleanly and produces no warning.
3. A host that cannot reach the network is reported as `unknown`, never as a credential
   failure.
4. A `denied` credential refuses a plan-approval answer before acceptance, naming the
   remediation.
5. New tests cover `ready`/`denied`/`unknown` and the no-agent capture case.
6. `docs/init.md` documents both durable credential options and the non-durability of a
   captured login-shell agent socket.
7. `sase tool run check` passes, or every failure is reproduced on master with this
   change stashed and filed.

## Out of scope

- The stitch/push credential path beyond the shared lease seam.
- The PR mirror, which `6087c0a8e` already established does not share this seam.
- Rotating or removing the existing passphrase-protected keys.
