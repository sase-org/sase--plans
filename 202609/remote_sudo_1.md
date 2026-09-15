---
tier: tale
title: Complete remote sudo over SSH terminal handoff
goal:
  Machine-targeted and remote-raised sudo requests authenticate on the target host over
  SSH while headless paths remain deny-only.
size: medium
proposed_by: bbugyi200.athena.sase-110.6
bead: sase-110.6
status: done
---

- **PARENT:**
  [202609/agent_sudo_requests.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_sudo_requests.md)
- **BEAD:**
  [sase-110.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-110/sase-110.6.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-110.6](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-110.6.md)
- **COMMITS:**
  - [7a1a1ca](https://github.com/sase-org/sase/commit/7a1a1ca3e46554fd99035617484773af8b57cf6e)
    — feat(sudo): support remote sudo handoff over ssh

# Complete machine-targeted and remote-raised sudo over SSH

## Goal

Complete phase `sase-110.6` as one bounded implementation: make enrolled machines carry
an SSH destination, execute locally raised machine-targeted sudo requests on that target
through a real `ssh -t` terminal handoff, and let ACE authenticate remote-raised sudo
requests on their origin host while every headless transport remains deny-only. Preserve
the approved epic's credential boundary: SASE must never collect, relay, record, hash,
or inspect PAM input.

The live apollo proof remains the later acceptance phase's responsibility. This phase
uses offline fake-SSH fixtures and does not change fleet-gateway execution semantics.

## Implementation

1. Align the Python sudo approval boundary with the already-pinned Rust contract before
   adding another transport. Make the documented `/sase_sudo` request fields (`why`,
   per-command timeout, `stop_on_failure`, `output_to_agent`, and `next.prompt`)
   normalize into a canonical `SudoManifestWire`, validate/hash/badge it through
   `sase_core_rs`, and invoke `sase_sudo_runner` with its real `--manifest` and
   `--expected-sha256` interface. Normalize the runner's `SudoLedgerWire` receipt and
   outcomes through the Rust binding so local and remote approvals settle only from a
   hash-matching ledger. Keep review-only reason/snapshot facts separate from the Rust
   execution manifest, and preserve narrowly useful aliases for already-created beta
   requests only when they map unambiguously. Add a regression test that the generated
   skill's example schema is accepted and that the real binding rejects tampered bytes.

2. Extend `MachineRecord` and the dispatch config schema with optional `ssh_target`,
   defaulting at use time to the machine alias. Round-trip it through config and
   connection projections, accept it from direct `sase machine add`, suggest the
   discovered tailnet/MagicDNS selector during guided `sase machine init`, and include
   it in offline `machine list` JSON/human output. Add a sorted
   `sase machine show ALIAS` command for the full local record. Validate SSH
   destinations as data (no leading option, whitespace, control characters, or shell
   fragments), keep all new public options paired with short aliases, and update
   config/parser/completion snapshots and focused machine tests.

3. Add a transport-neutral sudo execution helper plus an internal `sase sudo exec`
   target entrypoint. Local execution and target execution must consume the same sealed
   manifest envelope, revalidate the Rust wire schema and expected digest immediately
   before invoking the installed runner, require a controlling TTY, and return a bounded
   typed ledger without ever accepting credential-shaped fields. Resolve a request's
   target as an enrolled machine's `ssh_target`, otherwise use the request's machine
   value as an explicit unenrolled SSH destination and surface that fact in review.

4. Implement the SSH relay with an injected command runner for fixture testing. First
   probe `sase sudo exec --contract` and fail clearly on an unreachable host, missing
   target CLI, or schema/version mismatch. Because piping a manifest directly to
   `ssh -t` would replace the local terminal and prevent PAM interaction, stage the
   sealed bytes in a mode-0600, opaque, short-lived target-side file over a headless SSH
   call; then run the internal exec entrypoint under `ssh -t` with inherited terminal
   stdio, fetch the ledger after the TTY session, and remove staged manifest/receipt
   state on success, failure, or cancellation. The target re-hashes the staged bytes and
   refuses tampering before sudo runs. Never route approval through the fleet gateway.

5. Route machine-targeted `sase sudo answer` approvals through the SSH helper and local
   requests through the corrected local runner path, sharing lease, receipt validation,
   settlement, retry, and error-outcome behavior. Update the ACE sudo review facts and
   action copy to name the resolved host, show an “unenrolled SSH target” warning when
   applicable, and keep the terminal suspend path synchronous only while ACE is already
   suspended. Refresh the gate after every honest result while auth failure, Ctrl-C,
   timeout, version skew, and connection failure leave it pending.

6. Recognize remote-origin sudo attention conservatively from its typed gate identity
   and TTY-required approve/headless deny option shape. Render approve as **Authenticate
   over SSH** with the target command hint. Deny continues through the existing durable
   `sase machine attention` path; authenticate suspends ACE and invokes
   `ssh -t <origin ssh_target> sase sudo answer <id> --run`, then schedules the existing
   fleet and global-inventory repolls so origin-host settlement, dedup, and successor
   launch remain owner-controlled. Other remote gates/questions retain their current
   modal and durable behavior, and no zero-pending polling or render-path work is added.

7. Add fixture-first coverage for machine config and CLI rendering, enrolled and
   unenrolled resolution, contract probe/version skew, staged-manifest cleanup, manifest
   tamper refusal, target TTY refusal, ledger round-trip, unreachable/missing target
   errors, ACE machine-targeted handoff, remote-origin authenticate versus durable deny,
   and the invariant that mobile/fleet/detached approval remains refused while deny is
   accepted. Update only relevant sudo/remote-dispatch documentation and CLI epilogs.

## Verification and completion

Run the focused sudo, machine, dispatch-attention, ACE action/modal, parser,
config-schema, and completion tests while iterating. Before completion, read the
project's lint/test memory, run `git diff --check`, the binding validators affected by
the manifest change, and `just check`; use the SASE monitor workflow for any
long-running verification. Recheck `git status` so only intentional files remain.

Finally run `sase bead epic-symbols sase-110.6`. Resolve every listed symbol or re-key
its Justfile entry to `sase-110` or a still-open later phase, rerun the required checks
after that edit, and close only `sase-110.6` with a note naming the focused and
repository-wide verification that passed. Record any genuinely out-of-scope discovery as
a `PROPOSED FOLLOW-UP:` note on this phase rather than creating or closing another bead.
