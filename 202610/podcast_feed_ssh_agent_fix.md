---
tier: tale
title: Fix research audio episodes never reaching the podcast feed
goal:
  Queued research audio episodes publish to the apollo feed and appear in AntennaPod,
  and the shell ssh-agent bootstrap can no longer silently strand Codex-run agents on an
  empty agent.
size: medium
proposed_by: bbugyi200.athena.0xj
create_time: 2026-10-06 17:01:39
status: wip
---

# Plan: Fix research audio episodes never reaching the podcast feed

## Problem

AntennaPod shows none of the episodes rendered by recent `research.*.audio` agents. The
episodes render fine but never reach the feed on `apollo`; they pile up in the local
sase-listen outbox.

## Diagnosis (verified 2026-10-06)

1. **Symptom.** Every Codex-run audio render since 2026-10-06 (`research.3r.audio`,
   `research.3s.audio`, `research.3t.audio`) returned `published: false`,
   `publish_queued: true` and the warning
   `Auto-publish to apollo failed (feed host unreachable (tried apollo, apollo-do): bryan@159.223.165.54: Permission denied (publickey).)`.
   `~/.local/state/sase-listen/outbox/` holds three queued episodes:
   `highlights-and-listening-one-intake-gesture-6d5453`,
   `bob-ref-a-reference-library-for-reading-agents-0f5129`, and
   `twelve-more-readings-to-inspire-sase-goals-round-two-64b1ee`. The feed host itself
   is healthy (`sase-listen doctor` reports `apollo via apollo · 25 episodes`), and the
   2026-10-05 Codex audio runs (`research.3o.audio`, `research.3p.audio`) published
   normally.
2. **apollo accepts only a passphrase-protected key.** Of the keys in `~/.ssh`, apollo
   accepts only `id_rsa` (fingerprint
   `SHA256:1QjOXAh4edlsi2iJ3bnbdTR3d8BY13ZsTU2lg1RmjKA`), and that key has a passphrase.
   sase-listen connects with `ssh -o BatchMode=yes`, so publishing works only through an
   ssh-agent that already has `id_rsa` loaded. At diagnosis time that was the agent
   started 2026-10-01 (pid 18266, socket `/tmp/ssh-siD4gxJUzAdJ/agent.18263`). The TUI,
   tmux, and every agent runner inherit that agent.
3. **The `~/.zshrc` ssh-agent bootstrap misfired.** The chezmoi-managed bootstrap
   (`home/dot_zshrc`, "Starts ssh-agent automatically") compares
   `pgrep -u "${USER}" ssh-agent` against `~/.ssh-agent-thing`. When the two don't
   match, it starts a fresh agent and overwrites that file, and from then on every new
   interactive zsh uses the new agent. The old agent keeps running with the keys still
   loaded. This misfired at 2026-10-05 16:05:40 and again at 16:07:38, during a muse
   agent's probe runs that launched `claude` from `/tmp/sase-probe-1`. One way to get a
   mismatch: if the scrubbed environment had `USER` unset, `pgrep -u ""` prints nothing.
   The misfires left two orphaned empty agents (pids 2145128 and 2164915), and
   `~/.ssh-agent-thing` now names `/tmp/ssh-1ZBc9FkKOLZs/agent.2164914`, which "has no
   identities".
4. **Codex snapshots carry the bad socket into every command.** Codex CLI runs an
   interactive zsh once per session and saves its environment
   (`~/.codex/shell_snapshots/*.sh`). It sources that snapshot before every command, and
   the snapshot's `export SSH_AUTH_SOCK=...` overrides the good socket inherited from
   the runner. Every snapshot up to 2026-10-05 names `agent.18263`. Every snapshot since
   2026-10-06 06:26 names the empty `agent.2164914`. The `@audio` alias runs on Codex,
   so `sase-listen render`'s auto-publish had no key to offer and was denied. Claude
   Code snapshots do not export `SSH_AUTH_SOCK`, so Claude-run agents and manual renders
   from older panes kept publishing.
5. **Reproduced.** `sase-listen doctor --json` reports `feed:host ok` with the keyed
   agent. With `SSH_AUTH_SOCK` set to the empty agent's socket, it fails with exactly
   the agents' error.
6. **Why nothing recovered.** Only two things retry the outbox: a manual
   `sase-listen publish --pending`, or the best-effort flush at the start of the next
   publish. Every later publish ran in the same broken Codex environment and failed too.

The root cause is the non-idempotent bootstrap in the shared dotfiles. The fix repairs
the live state on athena right away, makes the bootstrap idempotent, makes sase-listen's
error name the agent state, and publishes the queued episodes.

## Changes

### 1. Repair the live ssh-agent state on athena (no passphrase needed)

Codex sessions started before this step still hold the broken snapshot, and this agent
may be one of them. Run every SSH-dependent command below with an explicit
`SSH_AUTH_SOCK=...` prefix, not the inherited value.

1. Find the live agent that holds the `id_rsa` fingerprint above. Probe each
   `/tmp/ssh-*/agent.*` socket and `$XDG_RUNTIME_DIR/openssh_agent` with
   `SSH_AUTH_SOCK=<sock> ssh-add -l`. If no agent holds the key (for example, athena
   rebooted), skip the rest of steps 1 and 4. Report that the user must run
   `ssh-add ~/.ssh/id_rsa` once (it needs the passphrase), then
   `sase-listen publish --pending`. Continue with steps 2 and 3.
2. Make `~/.ssh/agent.sock` a symlink to that socket. This is the stable path the new
   bootstrap in step 2 uses. If `~/.ssh/agent.sock` already exists, inspect it first.
   Replace it only if it is dead or points at an agent without the key.
3. Back up `~/.ssh-agent-thing` (for example, to `~/.ssh-agent-thing.bak-20261006`),
   then rewrite it so the old bootstrap, which stays deployed until the user's next
   `chezmoi update`, converges on the same agent:
   - Set `SSH_AUTH_SOCK=<$HOME>/.ssh/agent.sock; export SSH_AUTH_SOCK;`.
   - Set `SSH_AGENT_PID=<keyed agent pid>; export SSH_AGENT_PID;`.
   - Keep the line `echo Agent pid <keyed agent pid>;`, because the old `grep -q`
     matches on it.
4. Stop the orphaned empty agents. These are `ssh-agent` processes owned by the user
   whose socket reports `The agent has no identities`. Confirm each pid-to-socket
   mapping with `ss -xlp` before killing anything. Never touch the keyed agent or the
   systemd-managed `/usr/bin/ssh-agent -D` (`ssh-agent.service`). At diagnosis time the
   orphans were pids 2145128 and 2164915. Remove their leftover `/tmp/ssh-*` socket
   directories.
5. Verify that a fresh interactive shell lands on the keyed agent:
   `env -u SSH_AUTH_SOCK -u SSH_AGENT_PID zsh -i -c 'echo $SSH_AUTH_SOCK; ssh -o BatchMode=yes apollo true'`
   must print the `~/.ssh/agent.sock` path and exit 0. The `ssh-agent` process count
   must not grow.

### 2. chezmoi: make the ssh-agent bootstrap idempotent (`home/dot_zshrc`)

Open the linked repo with `sase repo open chezmoi` and read its `AGENTS.md`. Replace the
`pgrep` / `~/.ssh-agent-thing` block with a small function, called once, that does the
following:

- Uses one stable per-user socket, `${HOME}/.ssh/agent.sock`. It creates `~/.ssh`
  (mode 0700) if needed.
- Probes with `SSH_AUTH_SOCK="${sock}" ssh-add -l`:
  - Exit 0 (has identities) or 1 (reachable but empty): reuse the agent as is.
  - Exit 2 (unreachable): take a lock with zsh's `zsh/system` `zsystem flock` on
    `${sock}.lock`, re-probe, remove the stale path, and start
    `ssh-agent -a "${sock}" >/dev/null`. Release the lock afterwards. The lock keeps
    concurrent shell startups, such as a swarm's parallel Codex snapshots, from
    orphaning an agent.
- Always exports `SSH_AUTH_SOCK="${sock}"` and unsets `SSH_AGENT_PID`. This keeps
  today's behavior: interactive shells always use the local agent, never a forwarded one
  that disappears when the Mac's SSH session drops.
- Never uses `pgrep`, `$USER`, `kill`, or `~/.ssh-agent-thing`. It never stops or
  replaces a live agent.
- Prints nothing, because tools snapshot interactive shells. It quietly does nothing
  when `HOME` is unset or `ssh-agent`/`ssh-add` are not installed. It must stay
  portable: the same `dot_zshrc` deploys to athena, apollo, the Mac, and the work
  machine.

Tests (bashunit, `tests/bash/`):

- Extend `zshrc_portability_test.sh` with static assertions. The file must no longer
  contain `pgrep -u`, `.ssh-agent-thing`, or `kill "${ssh_agent_pid}"`. It must start
  the agent with `ssh-agent -a` on the `${HOME}/.ssh/agent.sock` path.
- Add a behavioral test, `zshrc_ssh_agent_test.sh`. Extract the function from
  `home/dot_zshrc` the way `sase_completion_test.sh` extracts zshrc fragments, and skip
  when zsh is missing (`require_zsh` pattern). Use a temp `HOME` and stub
  `ssh-add`/`ssh-agent` on `PATH` that log their argv, so no real agents start. Cover
  these cases:
  - Live socket: nothing is spawned.
  - Empty agent (exit 1): nothing is spawned.
  - Dead socket (exit 2): exactly one `ssh-agent -a $HOME/.ssh/agent.sock` call, and the
    stale path is removed.
  - `USER` unset: same result as the live-socket case.
  - A foreign inherited `SSH_AUTH_SOCK`: replaced by the stable path.
  - Sourcing twice: idempotent, with no extra spawn.

Run the repo's checks (`sase tool run check` if the repo guards `check`; otherwise
`just check`) and fix anything that fails.

Rollout note for the final report: the user applies the change with `chezmoi update` on
each machine. Step 1 already fixes athena before then. On other machines, the first new
shell starts an empty agent at the stable path, and the first interactive `ssh` re-adds
the key through the existing `AddKeysToAgent yes`. The stable path also never goes stale
for long-lived processes or captured service environments.

### 3. sase-listen: name the agent state on publickey denials

Open the linked repo with `sase repo open sase-listen` and read its `AGENTS.md`. Keep
its rules: no `sase` imports, and run checks through `sase tool run check`.

- In `src/sase_listen/feedhost.py`, add a small helper that describes the SSH agent in
  the current environment:
  - `SSH_AUTH_SOCK` unset: say so.
  - `ssh-add -l` exits 1: `the SSH agent at <sock> holds no identities`.
  - `ssh-add -l` exits 2: `the SSH agent at <sock> is unreachable`.
  - `ssh-add -l` exits 0:
    `the SSH agent at <sock> holds N identities, but the host accepted none`.
  - `ssh-add` missing, a timeout, or an `OSError`: return nothing. The helper must never
    raise.
- When `run_remote` exhausts its destinations and the last stderr contains
  `Permission denied (publickey`, append that description to the `FeedHostUnreachable`
  message. The message already flows into render warnings, the outbox `last_error`, and
  doctor's `feed:host` detail. Also set a specific hint: SSH reached the host but no key
  was accepted; load the key the host accepts into that agent (`ssh-add`), confirm
  `ssh -o BatchMode=yes <dest> true` from the same environment, then run
  `sase-listen publish --pending`. Other transport failures keep today's message and
  hint.
- Tests in `tests/test_feedhost.py`: reuse `_install_fake_ssh` with a destination that
  exits 255 with `Permission denied (publickey).`, and put a stub `ssh-add` on `PATH`.
  Assert the message names the socket for each case (no identities, unreachable, unset).
  Assert that a non-publickey failure such as `ssh: connect failed` never probes the
  agent and keeps its message.
- In `docs/troubleshooting.md`, under "Feed host unreachable", add a short
  `Permission denied (publickey)` paragraph. It should say that a passphrase-protected
  key needs a loaded agent in the environment that renders. It should note that tools
  which snapshot an interactive shell (Codex) use whatever `SSH_AUTH_SOCK` the shell rc
  exports. It should end with how to check (`ssh-add -l`) and retry
  (`sase-listen publish --pending`).

### 4. Publish the queued episodes and verify

1. Run `SSH_AUTH_SOCK="$HOME/.ssh/agent.sock" sase-listen publish --pending --json`.
   Every queued episode must publish, and `~/.local/state/sase-listen/outbox/` must be
   empty. The queue may also hold `research.3u.audio` or `research.3v.audio` episodes if
   those agents queued before step 1; they flush in the same run.
2. Run `SSH_AUTH_SOCK="$HOME/.ssh/agent.sock" sase-listen doctor --json`. `feed:host`
   must be ok, and `feed:outbox` must report nothing queued.
3. Confirm each flushed episode id appears in the host feed, for example with
   `SSH_AUTH_SOCK=... ssh apollo "grep -c '<episode_id>' ~/.local/share/sase-listen/feed/feed.xml"`.
4. Check `sase agent list -a` for `research.3u.audio` and `research.3v.audio`. If either
   has not rendered yet, note that its Codex session will snapshot the repaired agent
   and publish on its own. Any later queued leftovers flush on the next successful
   publish.
5. In the final report, tell the user to refresh the feed in AntennaPod. Also list the
   `chezmoi update` rollout step from change 2.

## Not chosen

- **Platform agents (systemd `ssh-agent.socket` on Linux, launchd on macOS).** This is
  cleaner on athena alone. But the shared `dot_zshrc` must behave the same on apollo,
  the Mac, and the work machine. A zshrc-owned stable socket is portable and keeps
  today's always-local-agent behavior. Revisit if publish-capable agents start running
  under systemd services, which see `/run/user/<uid>/openssh_agent`.
- **Reusing any reachable inherited `SSH_AUTH_SOCK` in zshrc.** tmux panes and the TUI
  would bind to the agent forwarded from the Mac session, which vanishes on disconnect
  and would break unattended agents.
- **Working around it in SASE's Codex provider** (disabling shell snapshots or pinning
  `SSH_AUTH_SOCK`). The snapshot's export overrides inherited values, and the defect is
  in the dotfile. A SASE workaround would hide the next rc regression and lose snapshot
  PATH and aliases.
- **A dedicated passphrase-less feed key on apollo.** It would remove the "load `id_rsa`
  once per boot" dependency, but it widens the credentials apollo accepts. It is not
  needed to fix this regression, because publishing worked for days on the agent model.
  It is a possible follow-up if reboots keep stalling unattended publishing.

## Verification summary

- `env -u SSH_AUTH_SOCK zsh -i -c 'ssh -o BatchMode=yes apollo true'` exits 0 on athena,
  and no new `ssh-agent` process appears.
- The chezmoi and sase-listen checks pass, including the new bashunit and pytest cases.
- The outbox is empty, `sase-listen doctor` is green, and the three episode ids (plus
  any flushed `3u`/`3v` episodes) are in apollo's `feed.xml`.
