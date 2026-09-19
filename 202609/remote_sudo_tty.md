---
tier: tale
title: Show the remote sudo PAM prompt on the controlling TTY
goal:
  Approving a machine-targeted sudo request from ACE shows the target host's PAM prompt
  on the reviewer's terminal instead of a blank hung session.
size: medium
proposed_by: bbugyi200.athena.0nn
create_time: 2026-09-19 07:56:06
status: wip
---

# Show the remote sudo PAM prompt on the controlling TTY

## Goal

When Bryan approves a machine-targeted sudo request from ACE, the suspended terminal
must show the target host's real `/usr/bin/sudo` PAM conversation. Today that session
looks hung and never displays a password prompt, because `ssh -t` inherits ACE's piped
`--json` stdout.

## Incident

`0ng.f1` created remote sudo gate `sudo-2290d0b8-6797-4eab-8b6a-5a303e8bc108` to inspect
and remove `/root/foobar` on enrolled machine `apollo`. Bryan approved from ACE. The
handoff terminal stayed blank with no PAM prompt.

The durable gate later settled as approved via `sudo_finalize` in about one second of
execution: all three reviewed commands ran, `inspect-root-foobar` printed
`-rw-r--r-- 1 root root 7 Sep 18 22:51 /root/foobar`, and `/root/foobar` was removed.
That silent success is consistent with apollo allowing `sudo -v` without a password
after the runner's mandatory `sudo -k`. It is not success of the review UX: the same
stdio wiring would deadlock until timeout on any host that actually prompts.

Do not re-run the foobar deletion as proof; the file is already gone. Proof is a
password-required remote target, or a fixture that shows PAM bytes on the controlling
TTY while `sase sudo answer --json` stdout remains a pipe.

## Confirmed diagnosis

Layered contracts that already exist:

- Local `sase_sudo_runner` prompts through `/dev/tty` after `sudo -k` then `sudo -v`,
  then runs the reviewed commands with `sudo -n`. Piping the runner's stdout is safe
  locally because PAM does not use that pipe.
- Remote approval stages the sealed manifest over headless SSH, then runs
  `ssh -t <target>` with `sase sudo exec --detach ...` through the account login shell
  (`src/sase/sudo/ssh.py`). The remote runner's PAM conversation is on the SSH PTY.
  OpenSSH relays that PTY over the **SSH client's stdout/stdin**, not `/dev/tty`.
- ACE approval (`src/sase/ace/tui/actions/agents/_notification_sudo.py`) suspends the
  TUI and runs `sase sudo answer --run --detach --json` with `stdout=subprocess.PIPE` so
  it can parse the JSON envelope. stderr/stdin stay on the real terminal. That pipe was
  introduced for honest `--json` toasts in
  [202609/sudo_handoff_stale_flag_pin.md](202609/sudo_handoff_stale_flag_pin.md) and is
  correct for local sudo.
- [202609/remote_sudo_1.md](202609/remote_sudo_1.md) already forbade piping the manifest
  into `ssh -t` because that "would replace the local terminal and prevent PAM
  interaction", and required `ssh -t` with inherited terminal stdio. The later `--json`
  pipe on the parent `sase sudo answer` process reintroduced the same failure one layer
  up: `_run_target_exec` / `_run_target_exec_detached` pass `stdout=None, stderr=None`,
  so `ssh -t` inherits ACE's pipe. The password prompt is written into an unread pipe.
  The reviewer sees a blank hung terminal. If PAM needs a password, the session waits
  forever on stdin the reviewer has no reason to type into.

Contract probe and manifest staging stay headless (no `-t`). They must keep capturing
stdout. Only the authentication `ssh -t` hop is wrong.

## Fix

Python-only, in this repo. No `sase-core` runner change, no sudoers change, no new CLI
options, no fleet-gateway path, and no capturing of PAM input.

1. In `src/sase/sudo/ssh.py`, attach the authentication SSH child to the controlling
   terminal instead of inheriting possibly-piped stdio:
   - Add a small helper that opens `/dev/tty` read/write and returns subprocess stdio
     kwargs (`stdin`/`stdout`/`stderr` all that fd). Raise the existing `tty_required`
     GateError if `/dev/tty` cannot be opened. Close the fd after the child exits.
   - Use that helper in `_run_target_exec` and `_run_target_exec_detached` only. Leave
     contract probe, staging, ledger/log reads, liveness, stop, and cleanup on captured
     or discarded stdio as they are today.
   - Keep `ssh -t`, login-shell wrapping for target `sase sudo exec`, accepted runner
     exit codes, and timeouts unchanged.
   - Do not pass the sealed manifest on SSH stdin. Staging stays a separate headless
     copy.

2. Before the `ssh -t` hop, write one non-secret status line to the controlling TTY (not
   to piped stdout), naming the target host, for example
   `sase sudo: authenticate on apollo`. This is how a NOPASSWD target stays honest: the
   reviewer sees the handoff instead of a blank suspend, even when PAM issues no prompt.

3. In `_terminal_banner` in `src/sase/ace/tui/actions/agents/_notification_sudo.py`,
   include the resolved machine/host from the already-loaded modal data. Keep
   `stdout=subprocess.PIPE` for `sase sudo answer --json`; that pipe is the toast
   contract and must keep working once `ssh -t` no longer inherits it.

4. Update `docs/sudo.md` Remote Flow: authentication `ssh -t` is attached to the
   reviewer's `/dev/tty` so ACE/`--json` stdout pipes cannot swallow the PAM
   conversation. Do not document passwords, prompt text, or attempt counts.

## Tests

Extend `tests/test_sudo_ssh.py` with injected stdio, not live `sudo` and not live
apollo:

- Default the new `/dev/tty` helper behind a monkeypatchable factory so existing
  FakeOpenSSH tests stay offline and do not require a controlling TTY in CI.
- Assert both synchronous and detached `ssh -t` exec calls receive the factory's
  stdin/stdout/stderr objects, while `--contract`, `mkdir` staging, `cat` ledger, and
  cleanup calls do not.
- Regression: a parent with `stdout=subprocess.PIPE` still settles from the handshake or
  ledger **file** after `ssh -t`; the fake PAM-like bytes must not need to be read from
  that pipe for the helper to return.
- Missing `/dev/tty` on the authentication hop raises `tty_required` and leaves no
  settled receipt (gate remains pending). Use a factory that raises OSError.
- Keep the login-environment assertions from
  [202609/remote_sudo_login_shell.md](202609/remote_sudo_login_shell.md): target `sase`
  still runs through the login shell; staging/cleanup stay direct.

Extend `tests/ace/tui/test_notification_sudo.py` so the suspend banner names the machine
when modal data has one, and so `subprocess.run` is still called with
`stdout=subprocess.PIPE` (ACE JSON contract unchanged).

Do not add visual PNG snapshots unless the modal layout changes; this tale should not
change the review modal layout.

## Verification

- Read `cli_rules.md` and `lint_and_test.md` before finishing. No new public CLI
  options.
- Run focused `tests/test_sudo_ssh.py`, `tests/test_sudo_detach.py` if the detached
  remote path is touched, and `tests/ace/tui/test_notification_sudo.py`.
- Run `just fix` then `just check`. Use `/sase_monitor` if `just check` runs long.
- Never invoke real `sudo`, never put credential material in tests or logs, and never
  route this through the fleet gateway.

## Acceptance

- ACE Authenticate on a machine-targeted sudo request shows the target PAM prompt on the
  suspended terminal when the target requires a password.
- `sase sudo answer --json` with stdout piped still returns a parseable envelope; ACE
  toasts stay honest.
- A target that does not prompt (NOPASSWD) still shows the TTY status line and returns
  instead of looking hung.
- Auth cancel, TTY missing, SSH failure, and contract mismatch leave the gate pending.
- Local sudo approval keeps working with ACE's stdout pipe, because the local runner
  still talks to `/dev/tty` directly.

## Out of scope

- Changing apollo sudoers or forcing a password on NOPASSWD hosts.
- Replacing ACE's `--json` stdout pipe with a sidecar file.
- sase-core runner stdio, fleet-gateway sudo, or a live `/root/foobar` retest.
