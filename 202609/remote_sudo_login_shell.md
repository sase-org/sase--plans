---
tier: tale
title: Resolve remote sudo through the target login environment
goal:
  Make reviewed remote sudo approvals work when the target SASE installation is
  available through login-profile PATH entries such as ~/.local/bin, without weakening
  the SSH, TTY, manifest, or credential boundaries.
size: medium
proposed_by: bbugyi200.athena.0nj
create_time: 2026-09-18 22:07:26
status: wip
---

# Plan: Resolve remote sudo through the target login environment

## Diagnosis

The pending `0ng.f0` gate correctly resolved `apollo` as an enrolled remote target and
sealed the expected three-command manifest, but it has no receipt and no remote handoff
directory. The failure therefore occurred before any reviewed command ran.

On Apollo, `sase` and `sase_sudo_runner` are installed under the user's uv-tool tree,
with `sase` exposed through `~/.local/bin`. Apollo's login shell includes
`~/.local/bin`, while a normal SSH remote-command session (including one with a PTY)
does not. Consequently, the bare `sase sudo exec ...` commands constructed by
`src/sase/sudo/ssh.py` fail to resolve the target CLI. Running the same contract probe
through the target account's login shell succeeds and advertises `detached_execution`.
The remote screenshot transport already uses this login-shell pattern for the same
reason, so remote sudo should share that established behavior.

## Implementation

1. Extract or expose a small shared SSH command encoder in the dispatch layer for
   running an argv through the remote account's configured login shell. Preserve argv
   boundaries with shell quoting, use the remote `SHELL` with a safe `/bin/sh` fallback,
   and keep the helper independent of screenshot- or sudo-specific policy.
2. Migrate the existing remote screenshot command construction to the shared helper
   without changing its behavior. This keeps one canonical implementation of the
   login-environment contract instead of copying a security-sensitive quoting wrapper.
3. Update the remote sudo transport so only target-side SASE invocations use the login
   environment:
   - the `sase sudo exec --contract` capability probe;
   - synchronous `sase sudo exec ...` execution; and
   - detached `sase sudo exec --detach ...` execution. Continue to run manifest staging,
     ledger/log reads, liveness probes, stop signaling, and cleanup as direct minimal
     remote commands. Preserve `ssh -t` for authentication, the existing timeout and
     accepted-exit-code semantics, and the rule that failures leave the gate pending.
4. Improve the pre-execution error returned for a missing or broken target CLI so it
   reports that resolution was attempted in the remote login environment and retains a
   bounded stderr/stdout detail when safe. Do not expose PAM input or any credential
   metadata.
5. Update the remote-sudo documentation to state that target SASE is resolved through
   the account's noninteractive login shell, matching the existing remote screenshot
   contract and normal `~/.local/bin` installations.

## Verification

1. Add focused unit coverage for the shared login-shell encoder, including spaces,
   quotes, dollar signs, and other shell-significant argv content.
2. Extend the OpenSSH-accurate sudo fake so its base SSH `PATH` cannot find `sase` but
   its login profile can. Assert that the contract probe and both synchronous and
   detached target execution succeed through that profile, still allocate a TTY where
   required, and preserve the reviewed manifest/ledger lifecycle.
3. Add failure coverage for a target whose login environment still lacks SASE, verifying
   the actionable diagnostic, bounded output, pending gate, and absence of a staged
   manifest or executed command.
4. Run the focused remote-sudo, detached-sudo, screenshot-transport, and parser tests,
   then run the repository's standard `just check` gate.

## Acceptance criteria

- A target with SASE only in a login-profile path such as `~/.local/bin` passes the
  contract probe and reaches its real remote sudo authentication flow.
- The controller never runs a reviewed command locally for a remote request, never
  captures or proxies credential input, and still validates the returned ledger before
  settling the gate.
- SSH transport failures or a genuinely missing target CLI remain retryable and leave
  the gate pending with an actionable, non-secret diagnostic.
- Existing remote screenshot behavior and direct non-SASE sudo handoff operations do not
  regress.
