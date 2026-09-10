---
tier: tale
title: Reduce bas disconnect-detection latency
goal:
  Established bas sessions detect an unresponsive SSH connection in roughly ten seconds
  so autossh can restart promptly.
size: small
proposed_by: bbugyi200.athena.0df
create_time: 2026-09-09 19:59:42
status: wip
---

# Reduce `bas` disconnect-detection latency

## Objective

Make an established `bas` session notice an unresponsive SSH connection and hand control
back to autossh in roughly 10 seconds instead of waiting around a minute, while
preserving the existing interactive remote-shell, tmux-environment, X11, TTY, and
reconnect behavior.

## Current behavior and rationale

`home/bin/executable_bas` currently invokes `autossh -M 0`. With monitoring port 0,
autossh does not probe the connection itself; it restarts only after its child SSH
process exits. An idle SSH TCP connection can therefore remain apparently alive after
the Mac loses WiFi until the operating system reports a socket failure. OpenSSH's
encrypted server-alive probes are the mechanism recommended by the autossh manual for
this mode.

Configure the SSH child used by `bas` with:

- `ServerAliveInterval=5`, so SSH probes after five seconds without server traffic.
- `ServerAliveCountMax=2`, so two unanswered probes terminate the SSH process after
  approximately ten seconds and allow autossh to restart it.

Keep these settings on the `bas` command line so they apply consistently to this
auto-reconnecting workflow and do not make every ordinary SSH session equally
aggressive. Retain `-M 0`: enabling autossh's forwarding-based monitor would introduce
port allocation and remote forwarding behavior that server-alive probes do not need. Do
not change `ConnectTimeout`, `AUTOSSH_GATETIME`, or `AUTOSSH_POLL`; they govern
connection establishment, the initial starting gate, or retry/backoff rather than
detection of a dead established session.

## Implementation

1. Update `home/bin/executable_bas` in the linked `chezmoi` repository to pass
   `-o ServerAliveInterval=5` and `-o ServerAliveCountMax=2` to the SSH child through
   autossh, before the destination host. Add a concise comment explaining that these
   options make SSH exit promptly when the connection stops responding so the existing
   autossh wrapper can restart it. Preserve all current flags, command construction,
   quoting, and shell/tmux setup.
2. Extend `tests/bash/bas_test.sh` with a focused regression test that runs `bas`
   against the stubbed autossh and asserts that both server-alive options, the TTY flag,
   and the destination host are forwarded in the expected pre-host argument sequence.
   Keep the existing host/TTY and remote-command assertions intact.

## Validation

1. Run `bashunit ./tests/bash/bas_test.sh` (or the repository's equivalent
   `just test-bash`) and confirm the new keepalive assertion and all existing `bas`
   regression tests pass.
2. Run the repository-wide `just check` to catch formatting, lint, and unrelated test
   regressions.
3. After applying the chezmoi change on the MacBook, manually start `bas athena`, let
   the session become idle, disable WiFi, and confirm SSH exits and autossh begins its
   restart path in approximately ten seconds. Restore WiFi and confirm the session
   reconnects and still reaches the expected `tm sase` environment.

## Acceptance criteria

- `bas` always supplies `ServerAliveInterval=5` and `ServerAliveCountMax=2` before its
  SSH destination.
- An established but unresponsive `bas` connection is detected in approximately ten
  seconds, subject to normal timer scheduling, and autossh retains responsibility for
  restarting the exited SSH process.
- The behavior is local to `bas`; global SSH configuration is unchanged.
- Existing remote command quoting, login-shell selection, tmux environment refresh, X11
  forwarding, TTY allocation, and host forwarding continue to work.
