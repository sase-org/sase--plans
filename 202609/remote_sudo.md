---
tier: tale
title: Remote sudo over SSH
goal:
  Machine-targeted and remote-raised sudo requests execute only through a reviewed,
  hash-bound SSH TTY handoff while every headless surface remains deny-only.
size: medium
proposed_by: bbugyi200.athena.sase-110.6
bead: sase-110.6
create_time: 2026-09-14 15:37:32
status: wip
---

- **PARENT:**
  [202609/agent_sudo_requests.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_sudo_requests.md)
- **BEAD:** sase-110.6

# Remote sudo over SSH

## Goal

Complete phase `sase-110.6` by extending the reviewed sudo workflow from local-only
execution to machine-targeted and remote-raised requests over a real `ssh -t` terminal
handoff. Preserve the reviewed manifest hash and runner receipt boundary, leave every
fleet/headless approval route deny-only, and add no idle ACE work when no remote sudo
request is pending.

## Implementation

1. Align the Python sudo adapter with the pinned Rust contract before adding transport:
   replace the temporary hashing/badge seam with the required `sase_core_rs` bindings,
   normalize requests into the Rust manifest shape, invoke `sase_sudo_runner` with an
   explicit manifest file and expected digest, and validate its structured ledger. This
   also removes the current circular import and ensures local and remote receipts share
   one contract.
2. Add `ssh_target` to `MachineRecord`, config parsing/serialization, the JSON schema,
   enrollment/add/init plumbing, machine list/show JSON and human presentation, and
   discovery defaults. Use the enrolled record's explicit target when present and the
   alias as the default; keep gateway connection plans unchanged.
3. Add a dedicated sudo SSH transport with explicit argv and injected process seams.
   Probe `sase sudo exec --contract` first, report version skew/missing SASE/unreachable
   targets distinctly, send a bounded sealed manifest through stdin to the internal exec
   endpoint, require the remote TTY, parse only the bounded JSON ledger, and revalidate
   its hash and command coverage before local settlement. Support enrolled aliases via
   `ssh_target` and direct unenrolled SSH destinations with an explicit review badge.
4. Extend `sase sudo` with the internal `exec` command and route machine-targeted
   `answer --run` through SSH while preserving local execution, auth lease behavior,
   partial-attempt handling, and pending-gate behavior on transport/auth failures. Keep
   internal options exempt from public short-alias requirements and document the
   user-visible remote behavior in CLI help.
5. Teach ACE's local sudo review/handoff to label remote targets and use the same
   terminal-attached command path. Detect remote-raised sudo attention entries from
   their projected `requires_tty` option metadata: expose denial through the existing
   durable fleet action, but route Authenticate over SSH through a synchronous suspend,
   then re-poll the inventory and refresh notifications. Generic remote gates retain
   their existing modal and durable execution path.
6. Add fixture-first coverage for machine-record round trips and CLI output, alias and
   direct-target resolution, contract probing, manifest transfer, hash/TTY/version and
   connectivity failures, ledger round-trip, local-vs-remote answer routing, and ACE
   remote-raised authenticate/deny behavior. Assert mobile/fleet headless approval
   remains refused through `requires_tty` rather than introducing surface allowlists.

## Verification

- Run focused sudo, dispatch machine/config, parser/completion, and ACE remote-attention
  tests, including offline fake-SSH/fake-target cases.
- Regenerate or update CLI completion snapshots only through the repository's normal
  workflow if parser changes require them.
- Run formatting and the canonical `just check` suite required by project memory.
- Before closing `sase-110.6`, run `sase bead epic-symbols sase-110.6` and resolve or
  re-key every remaining symbol, then close only this phase with a note summarizing the
  verified local/remote contracts.

## Constraints

- Never collect, forward, derive, log, or fixture authentication material; PAM remains
  between the human terminal, SSH, and `/usr/bin/sudo`.
- Never execute remote sudo through the HTTPS fleet bridge or any detached/headless
  proc. Denial may continue over the existing fleet attention transport.
- Do not change fleet gateway semantics, create task beads, close ancestor beads, or
  perform the live apollo proof reserved for the later acceptance phase.
- Keep all slow reads, SSH work, and subprocess waits outside Textual's event loop and
  serial message-pump callbacks.
