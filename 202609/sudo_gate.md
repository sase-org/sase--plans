---
tier: tale
title: Typed sudo gate and CLI front doors
goal:
  Agents can request a reviewed privileged command batch and terminal users can safely
  approve or deny it without SASE ever collecting a credential.
size: medium
proposed_by: bbugyi200.athena.sase-110.2
bead: sase-110.2
status: done
---

- **PARENT:**
  [202609/agent_sudo_requests.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_sudo_requests.md)
- **BEAD:**
  [sase-110.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-110/sase-110.2.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-110.2](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-110.2.md)
- **COMMITS:**
  - [7b85eb6](https://github.com/sase-org/sase/commit/7b85eb6c11ed1b098c06f372d62609ca55fae3e4)
    — feat(sudo): add typed sudo gate workflow

# Typed sudo gate and CLI front doors

## Goal

Complete phase `sase-110.2` by adding the feature-flagged `sudo` notification-gate kind
and the `sase sudo request|answer|list|show` terminal workflow. The implementation must
preserve the approved design's privilege boundary: SASE never accepts or stores a
credential, approval can execute only from a controlling TTY through the external
`sase_sudo_runner`, reviewed commands remain exact/hash-bound, and every headless path
can display or deny but cannot approve.

This tale intentionally excludes the Rust runner implementation and published binding
floor (`sase-110.1`/`sase-110.3`), the bespoke ACE review/suspend modal (`sase-110.4`),
the generated skill and provider guard (`sase-110.5`), and remote SSH relay support
(`sase-110.6`).

## Implementation

1. Create the required `agent_sudo_requests` beta flag with `sase flag new`, add its
   generated registry entry, and centralize the enabled-state check used by every new
   user-facing sudo entry point. Keep the disabled branch explicit: `sase sudo` and
   direct creation of a new sudo gate fail closed with a concise opt-in hint, while
   already-persisted data remains safely inspectable where required. Add both-state
   tests and keep the flag registry/bead integrity checks green.

2. Add a focused sudo domain package that validates and normalizes the version-one
   request contract (`reason`, non-empty uniquely identified argv commands, `run_as`,
   `cwd`, reviewed environment, timeout, stop policy, output policy, optional machine,
   and next prompt). Reject nested privilege tools, interactive programs, forbidden
   environment keys, malformed shell requests, and unsnapshotshotted agent-writable
   executables before a gate is created. Route canonical manifest validation/hash and
   risk-badge derivation through a thin injected core-binding seam so unit tests can run
   before the core pin lands, while an integration test requires the real binding.

3. Register the bespoke `sudo` gate adapter (`SudoRequest`, sender `sudo`, `SUDO` /
   `SUDOED`, lock icon, priority alongside questions), its kind validator, bundle-owned
   approve/deny command resources, and its dynamic follow-up hook. Require approve to
   carry `requires_tty: true`, require a runner-produced receipt before settlement, and
   render the bounded, fenced, untrusted ledger with authentication outcomes distinct
   from command failures. Update all hand-maintained action/kind/icon/tag/debug/toast
   tables that generic notification projection depends on, without adding a
   sudo-specific ACE modal in this phase.

4. Extend the gate option model and verified executor with optional `requires_tty`
   metadata. Check for a controlling TTY before accepting a decision or starting an
   option command, return a structured `tty_required` error, and leave the gate
   pending/answerable. Surface the disabled approve branch and `sase sudo answer <id>`
   hint in generic CLI show/answer projections. Prove that the direct headless gate
   answer, detached answer proc, and mobile bridge cannot approve, while deny remains
   executable on those same transports.

5. Add the sorted `sase sudo` parser and dispatcher. Make the bare group delegate to
   `list`; give every public long option a short alias and complete help/examples.
   `request` reads one JSON object from stdin, validates loudly before side effects,
   builds the gate shell and notification, prints its descriptor, writes the generic
   pending-handoff marker, and terminates the agent runner exactly once. `list` and
   `show` reuse verified gate lookup/projection and add sudo facts, risk badges, and any
   ledger. Repair the stale gate-shell lazy handoff exports by routing them to a small
   creator-handoff compatibility module backed by `sase.shells.handoff`, with regression
   tests for the existing generic gate-create callers.

6. Implement `sase sudo answer` as the only approval front door. Render the exact
   commands, verified execution facts, visibly untrusted reason, and risk badges; accept
   approve/deny/select non-interactively through flags or interactively on a real TTY.
   Deny uses the normal verified executor and optional feedback. Approve seals the
   reviewed subset into a hash-bound manifest, invokes the installed `sase_sudo_runner`
   attached to the controlling terminal (never pipes a password, never sets
   `SUDO_ASKPASS`, never invokes a shell), validates the returned ledger, and only then
   records a receipt and settles through the existing executor journal. Preserve
   partial-attempt state so a renewed review can resume at the first command not
   recorded `ran`; authentication failure, cancellation, runner failure, or missing TTY
   leaves the gate answerable.

7. Add focused unit and integration coverage for schema rejection, risk badges, resource
   hashing, flag-off/flag-on behavior, CLI parser/help/default-list behavior, loud
   request validation, pending-marker adoption, fixed lazy exports, receipt-required
   approval, TTY refusal on every headless surface, deny portability, resume-at-N,
   bounded ledger prompts, and absence of credential-shaped fields/values in bundle,
   journal, result, argv, and environment fixtures.

## Verification

- Run focused tests for the new sudo package, gate models/executor, CLI parser and
  handlers, notification/mobile projections, gate-shell handoff, and feature flags.
- Run the repository's required generated/static checks, including feature-flag
  integrity, CLI/help invariants, symbol lint, and core-binding validation.
- Run `just check` after consulting the project lint/test memory, fixing every failure
  attributable to this phase and re-running focused tests after any repair.
- Before closing `sase-110.2`, run `sase bead epic-symbols sase-110.2`; resolve or
  re-key every remaining symbol to a still-open epic/phase, then close only `sase-110.2`
  with a note naming the verified focused tests and `just check` result.
