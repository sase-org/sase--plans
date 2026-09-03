---
tier: tale
title: Enable the bob vault-sync launch service on apollo
goal:
  apollo continuously syncs the Bob vault via an enabled-on-launch bob-vault-sync user
  systemd service, mirroring athena.
size: medium
proposed_by: bbugyi200.kellys_mbp.k
---

# Enable the bob vault-sync launch service on apollo

## Goal

Start continuous git-channel syncing of the Bob vault (`~/bob`) on the apollo machine
via `bob vault-sync`, mirroring athena's launch-enabled user systemd service documented
in `docs/vault-git-sync.md` ("Background triggers"). When done, apollo should run
`bob-vault-sync.service` as a user unit that starts on boot (linger is already enabled)
and continuously reconciles the vault against `origin/master`.

## Verified current state (checked 2026-09-03 over `ssh apollo`)

apollo is an Ubuntu 24.x VPS (`ssh apollo`, user `bryan`, home `/home/bryan`):

- `bob` is installed at `~/.cargo/bin/bob` and was built 2026-09-03, so it is current.
  `bob vault-sync --help` works and exposes the `run` and `status` subcommands. Plain
  `bob vault-sync` defaults to `run`, and `run` accepts `-q/--quiet`.
- The service pieces are already deployed (they match the chezmoi sources
  `home/dot_config/systemd/user/bob-vault-sync.service` and
  `home/bin/executable_bob_vault_sync_watch`):
  - `~/.config/systemd/user/bob-vault-sync.service` exists but is **disabled**. It
    hardcodes `HOME=/home/bryan`, `BOB_DIR=/home/bryan/bob`, and
    `ExecStart=/home/bryan/bin/bob_vault_sync_watch`, all of which are correct for
    apollo.
  - `~/bin/bob_vault_sync_watch` exists. It loops: `inotifywait -t 15` on `$BOB_DIR`
    (errors silenced), `sleep 5`, then `bob vault-sync -q`.
- `loginctl show-user bryan -p Linger` reports `Linger=yes`, so the enabled unit will
  start on boot without a login session.
- **Missing pieces** this plan must supply:
  - No `~/bob` directory at all (no vault clone).
  - No `~/.ssh/id_bob_vault` deploy key and no `github-bob` alias in apollo's
    `~/.ssh/config` (only `id_rsa` exists).
  - `inotify-tools` is not installed, and `sudo` on apollo requires a password, so an
    agent cannot install it non-interactively.
- Git identity is configured on apollo (`Bryan Bugyi` / `bryanbugyi34@gmail.com`).
- The vault repository is `git@github.com:bobs-org/bob.git`. athena's pattern (from
  `docs/vault-git-sync.md` and the live GitHub deploy-key list) is a repository-scoped
  **read-write** deploy key titled `bob-vault-sync@athena` used through a `github-bob`
  SSH host alias pointing at `ssh.github.com:443`.
- On the local MacBook, `gh` is authenticated as `bbugyi200` with the `repo` scope and
  can list and add deploy keys on `bobs-org/bob`.
- chezmoi does **not** manage `~/.ssh`, so apollo's SSH config is edited in place
  (athena's was too). No chezmoi repo changes are needed by this plan.

## Non-goals

- Do not set up `bob nightly` / cron maintenance on apollo; athena stays the maintenance
  host.
- Do not change the chezmoi repo (the unit and watch script are already deployed).
- Do not change the highlights xlib rendezvous flow (`bob_xlib_pull`); apollo's
  gitignored `~/bob/xlib/` keeps serving as intake after the clone.
- No new `bob` CLI code, flags, or subcommands.

## Steps

Run remote commands via `ssh apollo '<command>'` unless stated otherwise. Steps 1-8 need
no sudo. Make each step idempotent as described so a rerun after a partial failure is
safe.

### 1. Preflight (read-only)

- Confirm `~/.config/systemd/user/bob-vault-sync.service` and
  `~/bin/bob_vault_sync_watch` still exist on apollo and the unit's `ExecStart` and
  `BOB_DIR` match the values quoted above.
- Confirm `bob vault-sync --help` succeeds and `~/bob` still does not exist. If `~/bob`
  now exists, stop and reassess before cloning over it.

### 2. Generate the deploy key on apollo

```bash
ssh apollo 'test -f ~/.ssh/id_bob_vault || ssh-keygen -t ed25519 -N "" -C bob-vault-sync@apollo -f ~/.ssh/id_bob_vault'
```

### 3. Register the deploy key on GitHub (run from the local machine)

```bash
ssh apollo 'cat ~/.ssh/id_bob_vault.pub' > /tmp/id_bob_vault_apollo.pub
gh repo deploy-key add /tmp/id_bob_vault_apollo.pub --repo bobs-org/bob --allow-write --title bob-vault-sync@apollo
gh repo deploy-key list --repo bobs-org/bob
```

Skip the `add` if `bob-vault-sync@apollo` already appears in the list. The key must be
**read-write** (`--allow-write`), like athena's.

### 4. Add the `github-bob` SSH alias and host key on apollo

Append this block to apollo's `~/.ssh/config` only if `Host github-bob` is not already
present (`grep -q '^Host github-bob' ~/.ssh/config`):

```sshconfig
Host github-bob
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_bob_vault
  IdentitiesOnly yes
  AddKeysToAgent no
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```

Then seed the host key so unattended git never sees a first-connect prompt:

```bash
ssh apollo 'ssh-keygen -F "[ssh.github.com]:443" >/dev/null || ssh-keyscan -p 443 ssh.github.com >> ~/.ssh/known_hosts'
ssh apollo 'ssh -o BatchMode=yes github-bob 2>&1 | head -1'   # expect the "successfully authenticated" banner
```

### 5. Clone the vault

```bash
ssh apollo 'git clone github-bob:bobs-org/bob.git ~/bob'
ssh apollo 'git -C ~/bob remote -v'
```

The clone may take a while (the vault tracks PDFs under `lib/`); raise the command
timeout accordingly or run it under `/sase_monitor`.

### 6. Verify unattended access

Mirror the runbook's cold-environment check:

```bash
ssh apollo 'env -i HOME=/home/bryan PATH=/usr/bin:/bin git -C ~/bob ls-remote origin master'
```

### 7. First manual sync cycle

```bash
ssh apollo 'bob vault-sync'
ssh apollo 'bob vault-sync status --json'
```

Expect a successful cycle with `last_error` empty/null. A fresh clone should mostly be a
no-op (nothing local to commit, nothing to pull).

### 8. Enable the service on launch

```bash
ssh apollo 'systemctl --user enable --now bob-vault-sync.service'
ssh apollo 'systemctl --user status bob-vault-sync.service --no-pager'
ssh apollo 'journalctl --user -u bob-vault-sync.service -n 40 --no-pager'
```

The unit must be `enabled` and `active (running)`. Because `inotify-tools` is missing,
`inotifywait` fails instantly and silently, so the watch loop degrades to roughly a
5-second poll that still runs `bob vault-sync -q` each pass — functional, just busier
than the intended inotify wait. Confirm repeated successful cycles via
`bob vault-sync status --json` (advancing `last_attempt`/`last_success` timestamps).

### 9. Offer the inotify-tools install as a gated command

`sudo` needs a password, so use the `/sase_gate` skill to propose this command for Bryan
to confirm/run:

```bash
ssh -t apollo 'sudo apt-get update && sudo apt-get install -y inotify-tools'
```

Do not block on it: the service is already functional via the poll fallback, and the
watch script picks up `inotifywait` on its next loop iteration after the install — no
restart needed. If the gate is declined or deferred, leave the service running and say
so in the final report.

### 10. Update `docs/vault-git-sync.md` (bob-cli repo)

- **Credentials**: add apollo alongside athena — repository-scoped read-write deploy key
  at `~/.ssh/id_bob_vault` (`bob-vault-sync@apollo`), same `github-bob` alias block, and
  an apollo variant of the unattended-access check
  (`ssh apollo 'env -i HOME=/home/bryan PATH=/usr/bin:/bin git -C ~/bob ls-remote origin master'`).
- **Background triggers**: document that apollo runs the same user systemd unit as
  athena (same `systemctl --user` / `journalctl --user` commands, via `ssh apollo`), and
  note the `inotify-tools` requirement plus the silent ~5s poll fallback when it is
  absent.
- **Highlights bridge**: one sentence noting apollo's `~/bob` is now a full git clone
  and its gitignored `xlib/` continues to serve as the rendezvous intake for
  `bob_xlib_pull`.

## Validation

- `ssh apollo 'systemctl --user is-enabled bob-vault-sync.service'` prints `enabled`;
  `is-active` prints `active`.
- `ssh apollo 'bob vault-sync status --json'` shows a recent successful cycle and no
  `last_error`, and the timestamps advance between two checks ~30s apart.
- `ssh apollo 'git -C ~/bob status --short'` is clean (or shows only expected local
  state), and `journalctl` shows no repeating errors.
- The docs change renders sensibly (`git diff docs/vault-git-sync.md`).

## Rollback

- Stop syncing: `ssh apollo 'systemctl --user disable --now bob-vault-sync.service'`.
- Revoke access: `gh repo deploy-key list --repo bobs-org/bob`, then
  `gh repo deploy-key delete <id> --repo bobs-org/bob` for `bob-vault-sync@apollo`.
- Remove the clone (`rm -rf ~/bob` on apollo) only if explicitly desired — never
  `reset --hard` or force-push the vault (runbook conflict policy).

## Safety notes

- Follow the runbook's conflict policy: no `reset --hard`, no force-push, no
  `-X ours/theirs` in `~/bob`.
- The deploy key is repository-scoped on purpose; do not reuse apollo's `id_rsa` for the
  vault remote.
