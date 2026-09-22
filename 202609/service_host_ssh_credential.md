---
tier: tale
title:
  Give the service host a reboot-proof GitHub credential and stop stale-agent
  recurrences
goal:
  The athena service host authenticates to GitHub with no SSH agent, so the
  external_issue_mirror auth_error and toobig_split lease failures stop. The host also
  ignores a captured SSH agent socket that died, and `sase service init`/`status` warn
  before a reboot when the only working credential is a login-session agent.
size: medium
proposed_by: bbugyi200.athena.0p3
create_time: 2026-09-22 07:57:55
status: wip
---

# Plan: Reboot-proof service-host GitHub credential

## Problem

Since 06:31 today every axe error digest has reported, for `actstat` and `bob-cli`:

```
external_issue_mirror[...] degraded (auth_error): operational workspace lease failed during
preparation: git@ssh.github.com: Permission denied (publickey).
```

At 06:32 `toobig_split[sase]` also failed, with the same lease error repeated for all
~19 typed-admission units, and the SDD `plans` clone fetch fails for the same reason.

What was verified on athena:

- **Reboot.** athena rebooted at 06:31:25. The last clean mirror tick was 06:19.
- **Pinned dead agent.** `~/.sase/service/env` pins
  `SSH_AUTH_SOCK=/tmp/ssh-bd1YBG80VDga/agent.13572` and `SSH_AGENT_PID=13573`. That is a
  login-shell agent captured by an earlier `sase service init`, and its directory no
  longer exists.
  - The host start path calls `load_service_environment(override_existing=True)` in
    `src/sase/service/env.py`, which applies that dead path over the systemd socket
    (`/run/user/1000/openssh_agent`).
  - Every routine child, chop script, and lease `git fetch` inherits the dead path.
  - `docs/init.md` says the host "falls back to whichever agent your platform manager
    gives every unit". The code does not do that; liveness is checked only at capture
    time, in `_capture_ssh_agent`.
- **No agentless fallback.**
  - The systemd agent is live but empty.
  - `~/.ssh/id_rsa` and `~/.ssh/id_ed25519` are passphrase-protected.
  - So nothing can authenticate unattended. `sase service status` already reports this.
- **Not a new failure.**
  - Commit `624a29ea7` (Sep 20) fixed the reporting side and explicitly left the host
    credential to the user.
  - The later `sase service init` from a login shell worked only until this reboot.
  - Nothing warned that the capture would not survive a reboot.
- **Backoff and digest.** The mirror's exponential backoff works: state is in
  `~/.sase/external_mirror/issues/<project>.json`, currently `failures=6`. It only
  spaces out the errors; they recur in every hourly digest until the credential is
  fixed.

The code cannot mint a credential, so the fix has two parts:

1. **Host remediation on athena** (Part A): the only thing that stops today's errors.
2. **Hardening** (Part B), so a future reboot or logout cannot quietly recreate this
   state.

Do Part A first so the errors stop right away. If Part A is blocked, still do Part B and
report exactly what the user must run.

## Part A: Host remediation on athena (outside the repo)

This follows option 1 in `docs/init.md`: a dedicated passphrase-less key through
`IdentityFile`. **Approving this plan authorizes all three of these actions:**

- Creating a passphrase-less private key at `~/.ssh/id_sase_service` on athena.
- Registering its public key as an authentication key on the `bbugyi200` GitHub account.
- Editing the SSH config file behind the `~/.ssh/config` symlink.

Do not print or copy the private key anywhere.

1. **Key.** If `~/.ssh/id_sase_service` already exists, reuse it and never overwrite it.
   Otherwise run
   `ssh-keygen -t ed25519 -N "" -C "sase-service@athena" -f ~/.ssh/id_sase_service`,
   then confirm the private key is mode `0600`.
2. **GitHub registration.**
   - Check `gh ssh-key list` for the public key, and skip this step if it is already
     there.
   - Otherwise run
     `gh ssh-key add ~/.ssh/id_sase_service.pub --title "sase-service@athena" --type authentication`.
     The gh token has `admin:public_key`.
   - If this fails, stop Part A. Report the exact command, or the web path (GitHub →
     Settings → SSH keys), for the user to run, then continue with Part B.
3. **SSH config.**
   - `~/.ssh/config` is a symlink to a Syncthing-shared file, currently
     `~/Sync/home/.ssh/config`; resolve it with `readlink -f`.
   - Edit the **target file** in place. Keep the symlink and the file's `0600` mode. Do
     not use `sed -i` on the symlink path, because that replaces the link with a regular
     file.
   - The file is shared across machines, so guard the new identity to apply only where
     the key exists. Insert this directly after the existing `Host github.com` block,
     and leave every other block unchanged:

     ```
     # SASE service host unattended GitHub credential (sase docs/init.md).
     # No-op on machines without the key.
     Match originalhost github.com exec "test -r %d/.ssh/id_sase_service"
       IdentityFile ~/.ssh/id_sase_service
     ```

   - Do not add `IdentitiesOnly yes`. Interactive shells keep trying their agent keys
     first, and the service falls through to this file when its agent is dead or empty.
   - `tailnet.conf` is chezmoi-managed and is not touched.

4. **Verify the credential without any agent.**
   - `ssh -G github.com | grep -i identityfile` lists `~/.ssh/id_sase_service`, and
     `ssh -G github-bob` still lists only `~/.ssh/id_bob_vault`.
   - `env -u SSH_AUTH_SOCK -u SSH_AGENT_PID ssh -o BatchMode=yes -T git@github.com`
     prints `successfully authenticated`.
   - `SSH_AUTH_SOCK=/nonexistent/agent.sock ssh -o BatchMode=yes -T git@github.com` also
     succeeds. This mimics the dead socket the host currently has.
   - `sase service status` no longer prints a `refused by the git remote` warning.
5. **Clear the mirror backoff through the real lease path, agentless.**
   - Run
     `env -u SSH_AUTH_SOCK -u SSH_AGENT_PID sase bead sync-external --full -p actstat`,
     then the same command with `-p bob-cli`.
   - Both must finish without `auth_error`. A clean pass resets the persisted `failures`
     count. This is the same reconciliation the scheduled job runs.
   - Do not hand-edit `~/.sase/external_mirror/issues/*.json`.
6. **Leave the service environment alone.**
   - Do **not** run `sase service init --yes` from the agent. It would recapture the
     agent's own `PATH`, provider variables, and agent socket.
   - Do not restart the service. SSH config and key files are read on every connection,
     so no restart is needed.
   - The stale `SSH_AUTH_SOCK` in `~/.sase/service/env` becomes harmless once the key
     works. After Part B ships, the host ignores it anyway.
7. **Optional confirmation.** The next scheduled `external_issue_mirror[actstat]` and
   `[bob-cli]` runs should show no `auth_error`. Check the newest run log under
   `~/.sase/axe/lumberjacks/external_mirror/chops/external_issue_mirror[<name>]/runs/`,
   polling in the foreground for up to 20 minutes. Record whatever you observe, but a
   pending tick is not a failure.

**Rollback** (report these in the summary): `gh ssh-key delete <id>`, remove the
three-line `Match` block, and delete `~/.ssh/id_sase_service*`.

## Part B: Hardening in the sase repo

This is service-host environment glue and stays in Python (`src/sase/service/`). No Rust
core change is needed.

### B1. Ignore a dead captured agent at host start

- **`src/sase/service/env.py`, `load_service_environment`:**
  - If the captured `SSH_AUTH_SOCK` is not a live socket, apply neither it nor the
    captured `SSH_AGENT_PID`, in both override modes. The host then keeps whatever agent
    the platform manager provided.
  - Use one liveness predicate shared with `_capture_ssh_agent`, so capture and load
    agree on what "live" means. Keep today's `Path.is_socket()` semantics.
  - Other captured names are applied exactly as before.
  - The returned applied-names tuple must not include the skipped names.
- **`src/sase/service/effective_env.py`, `_effective_service_environment`:** apply the
  same rule. `sase service status` must then describe the agent the host would really
  use.
- **Effective-scope warning wording.**
  - When the persisted socket is stale and the fallback is refused, the warning must
    name the ignored stale path and the fallback agent (or state that no agent is
    provided). Example: "the captured SSH agent at X no longer exists, so the host uses
    the platform manager's agent at Y, which is reachable but holds no identities …".
  - Extend `_agent_detail` in `src/sase/service/ssh_agent.py` or its caller for this.
    Warnings are still emitted only when the remote answers `denied`.

### B2. Warn when the only accepted credential is a login-session agent

This is the gap that let the Sep 20 re-init look healthy.

- **Where:** add a durability check to `service_init_plan`
  (`src/sase/service/platform.py`), so it covers `sase service init`, `--check`,
  `sase service status`, and the doctor check.
- **What it checks:**
  1. Always, the environment init is about to capture (`desired_env`).
  2. When the unit is installed and its effective `SSH_AUTH_SOCK` differs from
     `desired_env`'s, the effective environment too. This is the same dedupe rule
     `effective_ssh_agent_warnings` uses.
- **It warns only when all of these hold:**
  - the checked environment has an `SSH_AUTH_SOCK`;
  - that socket is not the platform manager's inherited `SSH_AUTH_SOCK` (from
    `_inherited_environment`);
  - the git remote answers `ready` for the checked environment;
  - the git remote answers `denied` for the fallback environment. The fallback is the
    same environment with `SSH_AUTH_SOCK`/`SSH_AGENT_PID` replaced by the manager's
    inherited `SSH_AUTH_SOCK`, or removed when the manager provides none. With B1, this
    is exactly what the host would see after a reboot or logout.
- **It stays silent when:**
  - either answer is `unknown`;
  - the agent _is_ the manager's. That covers option 2 in `docs/init.md` and the macOS
    launchd agent.
  - the host authenticates through an `IdentityFile`, because the fallback then answers
    `ready`.
- **Wording:** say that the service host's only accepted credential is the SSH agent at
  `<sock>`. That agent belongs to a login session, not the platform manager. After a
  reboot or logout the host falls back to `<manager agent | no agent>`, which the git
  remote refuses. Point to docs/init.md for giving the host an unattended credential.
  Distinguish the captured and effective scopes the way the existing readiness warnings
  do. The warning makes the plan `needs_attention`.
- **Cost:** within one `service_init_plan` call, never issue a second
  `probe_git_remote_auth` for an environment already probed. That covers the capture's
  existing readiness probe and the effective probe. One extra probe per checked
  environment, for the fallback, is the budget.
  - Reuse the existing answers, either by memoizing per (`SSH_AUTH_SOCK`,
    `SSH_AGENT_PID`, `PATH`) for the duration of the plan or by threading the answer
    through.
  - Keep `probe_git_remote_auth` and `ssh_agent_readiness_warnings` never-raising, and
    keep the capture API backward compatible for existing callers.

### B3. Docs (`docs/init.md`, "Service host" section)

- Replace the paragraph that claims the host "silently falls back". Now it is accurate:
  at host start, a captured agent socket that no longer exists is ignored and the
  manager's agent applies. A captured login-shell socket is still not a durable
  credential.
- Document the new durability warning and when it fires.
- In option 1, add two notes:
  - The agentless verification command:
    `env -u SSH_AUTH_SOCK -u SSH_AGENT_PID ssh -o BatchMode=yes -T git@github.com`.
  - A tip for an `~/.ssh/config` shared across machines: guard the identity with
    `Match originalhost github.com exec "test -r %d/.ssh/id_sase_service"` instead of
    editing `Host github.com`, so machines without the key are unaffected.
- In option 2, note that after a reboot you only need to `ssh-add` into the manager's
  agent again. A stale capture no longer masks it.
- Run `just fmt` so Markdown formatting matches.

### B4. Tests

- **`tests/service/test_service_environment.py`:**
  - `load_service_environment(override_existing=True)` with a persisted stale
    `SSH_AUTH_SOCK` and `SSH_AGENT_PID` leaves the pre-set inherited values in
    `os.environ` and omits both from the applied names.
  - A live persisted socket is still applied.
  - Non-SSH names are unaffected.
- **`tests/service/test_service_effective_env.py`:**
  - The effective environment keeps the manager's socket when the persisted one is
    stale.
  - Update `test_plan_reports_stale_persisted_agent_despite_healthy_caller_shell`: the
    effective warning now names the manager socket as the agent in use and mentions the
    ignored stale path.
  - Durability matrix:
    - it warns (captured scope) for a session agent answering `ready` with a
      manager-fallback answering `denied`;
    - it is silent when the fallback answers `ready` (an IdentityFile host), when the
      agent equals the manager's, when the primary answer is `denied` (the readiness
      warning already covers that), and when either answer is `unknown`;
    - effective-scope reporting follows the same dedupe;
    - count probe calls to prove no environment is probed twice in one plan.
- Use the existing `_live_socket`, `_fake_agents`, `_fake_remote`, and `_Manager`
  helpers. The suite-wide fixture already forbids real `ssh -T git@github.com`, so no
  test may touch the network or the real `~/.ssh`.

### Verification

- Run `sase tool run check`, or `just check` if `sase tool` is unavailable, until it
  passes.
- Follow the `lint_and_test` and `symvision` reference memories through
  `/sase_memory_read` if lint flags new or private symbols.
- Do not run `just check-full`.

## Out of scope

- **Mirror backoff and digest alerting stay as they are.** With the credential fixed
  they go quiet, and persistent failures should keep being reported.
- **Typed-admission error aggregation**
  (`src/sase/axe/chop_runner_script_result_launch.py` and `_chop_lifecycle_runner.py`):
  - It repeats an identical message once per unit, which made the 06:37 digest 25 KB.
  - Separately, the partial-launch branch drops the joined error before the digest.
  - This is a separate subsystem. File one task bead for it through `/sase_new_task`
    (evidence: `~/.sase/axe/error_digests/digest_20260922_063743.txt`), and do not fix
    it here.
- Heuristic discovery of other live agent sockets (`/tmp/ssh-*/agent.*`) is rejected. It
  would adopt session or forwarded agents that die the same way.
- Running `sase service init --yes` or restarting the service from the agent.
