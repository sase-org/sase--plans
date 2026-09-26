---
tier: tale
title: Gate prepared completion on a covering receipt
goal:
  Explicit no-new completion commits only the freshly verified tree under a covering
  receipt and records its provenance.
size: medium
proposed_by: bbugyi200.athena.sase-1ah.6
bead: sase-1ah.6
create_time: 2026-09-26 10:55:08
status: wip
---

- **PARENT:**
  [202609/tool_e4_verified_completion.md](https://github.com/sase-org/sase--plans/blob/main/202609/tool_e4_verified_completion.md)
- **BEAD:**
  [sase-1ah.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ah/sase-1ah.6.md)

# Gate prepared completion on a covering receipt

## Scope and boundaries

Implement only phase `sase-1ah.6` of `plan:202609/tool_e4_verified_completion.md`. The
existing default prepared-completion policy remains `pass`; `no-new` is an explicit,
sealed opt-in. Do not change child exit codes, skip verification runs, close the parent
epic, remove the `tool_receipts` flag, or take over phase 7 documentation and rollout
work. The Rust core owns policy decisions and typed receipt eligibility; Python observes
host state and calls the Rust bindings.

## Implementation

1. In the linked `sase-core` checkout, extend conditional-completion
   request/intent/evaluation wires with an `accept` policy that defaults to `pass` for
   older intents. Include the value and policy version in the seal and validate it on
   load and bind. Permit `no-new` only for a bound, named `sase tool run` verify monitor
   with its actual settled ToolRun identity; keep the pass policy and its preview/output
   byte-identical. Update frozen verify-policy resolution so the failed branch may enter
   host completion only for an explicit prepared `no-new` intent, while timeout,
   stopped, lost, raw, and unrelated failures route to recovery. Add Rust binding round
   trips and focused compatibility tests; coordinate the `sase-core-revision.txt` pin
   before Python requires new bindings.

2. In sase's `final prepare` adapter, accept `accept: pass|no-new`, pass it to Rust, and
   reject malformed or unbound no-new requests. At monitor settlement, retrieve the
   settled ToolRun from the monitor/owner link, not by a latest-run search. Confirm its
   command, source run, verdict, policy, and complete fingerprint match the sealed
   verification and required repository obligations. Call Rust receipt lookup with
   `no_new_failures` acceptance, retain its typed refusal, and require a covering
   receipt from that exact run. A NEW, UNKNOWN, missing, expired, invalidated,
   changed-definition, changed-policy, or incomplete result goes through ordinary
   recovery.

3. Add one host-owned precommit gate that re-observes **all** obligated repositories
   immediately before the first mutating commit action, compares them with the verified
   fingerprint, and repeats the Rust lookup for the same receipt/source run and policy.
   Refuse the whole no-new completion before any commit if any repository is absent,
   dirty in a new way, or cannot be checked. Re-run this gate on resumed host completion
   rather than trusting a persisted prior success; do not allow the
   `can_finish_without_rerun` shortcut to bypass it. Preserve the existing pass-only
   path. Thread the eligibility evidence into the commit and bead-close host actions as
   receipt id, run id, verdict, and KNOWN signature list. On ordinary
   `sase final submit` without a covering receipt, record `unverified` as nonblocking
   provenance.

4. Add focused Rust and Python fixtures for default pass goldens; explicit no-new
   KNOWN/FLAKY nonzero completion; NEW/UNKNOWN and unrelated or lost monitors; wrong
   source run; policy change; expiry, invalidation, missing receipt, and fingerprint
   drift; multi-repository all-or-nothing refusal; resume after drift; and provenance on
   commit/bead-close versus ordinary unverified finalization. Verify the monitored child
   exit remains nonzero. Run relevant focused tests and each changed repository's
   required `sase tool run check` gate (not `check-full`). If a check fails identically
   on clean base, note it as `PROPOSED FOLLOW-UP:` on `sase-1ah.6` and proceed with the
   phase close.

5. Before closing, run `sase bead epic-symbols sase-1ah.6`, resolve each remaining
   symbol or re-key its Justfile line to an open bead, and close **only** `sase-1ah.6`
   with `sase bead close sase-1ah.6 --note "<verified evidence>"`. Record other
   discovered work only through a `PROPOSED FOLLOW-UP:` note on this phase bead. Submit
   repository changes through the SASE final declaration.

## Acceptance

An explicit no-new prepared completion commits only when its failed verification has a
current `no_new_failures` receipt from the same settled run covering every obligated
repository and the tree still matches at precommit. Any mismatch has one typed refusal,
no commit, and an ordinary recovery follow-up. The default pass path produces its
existing golden output and behavior. Commit and bead-close actions have auditable
receipt provenance, while ordinary unverified finalization remains usable.
