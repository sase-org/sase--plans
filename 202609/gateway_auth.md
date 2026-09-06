---
tier: tale
title: Authenticated fleet gateway enrollment
goal:
  Remote fleet access starts from target authorization and remains bounded, revocable,
  versioned, and identity-pinned.
size: medium
proposed_by: bbugyi200.athena.sase-xe.4
bead: sase-xe.4
status: done
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.4.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-xe.4](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xe.4.md)
- **COMMITS:**
  - [f00ed92](https://github.com/sase-org/sase-core/commit/f00ed92aa41f5bb0a94216b74031b38ac608824f)
    — feat(gateway): add fleet authentication

# Authenticated fleet gateway enrollment

## Goal

Add the first secure fleet-protocol surface to `sase_gateway` without changing the
existing loopback-oriented mobile API. The new surface must support a target-authorized
bootstrap, scoped and revocable controller credentials, negotiated protocol versions,
authoritative installation identity, explicit pin-mismatch quarantine, bounded
unauthenticated traffic, and authentication that does not write durable state on the
request hot path.

## Implementation

1. Add fleet-specific wire records and a committed API contract snapshot for enrollment,
   hello/capabilities, credential rotation, credential revocation, error outcomes, and
   protocol-version negotiation. Keep the fleet schema and route namespace independent
   from mobile v1 while reusing the transport-free installation identity and capability
   contracts from `sase_core`.
2. Add a fleet credential/enrollment store under the SASE home directory. Persist only
   hashes of high-entropy secrets and tokens; use constant-time comparisons, atomic
   mode-0600 writes, an inter-process lock, expirations, single-use bootstrap records,
   revocation metadata, and an in-memory authentication cache that refreshes only when
   durable state changes. Expose a local-only Rust API for an authorized target-side
   caller to issue the short-lived bootstrap secret.
3. Mount `/api/fleet/v1` routes beside `/api/v1`: enrollment, authenticated hello, token
   rotation, and credential revocation. Negotiate the highest mutually supported
   protocol version, return authoritative installation identity and the configured
   machine selector, enforce each handler's declared scope, reject stale/revoked tokens,
   turn a presented installation-pin mismatch into an explicit quarantine response,
   rate-limit enrollment attempts, and apply a small fleet request-body ceiling.
4. Extend public exports and contract generation for the new fleet surface while
   preserving the current mobile contract snapshot byte-for-byte.

## Verification

- Add store tests for hashed-at-rest material, single-use and expiry behavior, rotation,
  revocation, cache invalidation, constant-time lookup paths, mode-0600 files, and
  cross-process-safe updates.
- Add route tests for successful enrollment/hello, replayed and expired bootstrap
  secrets, missing scopes, revoked and rotated tokens, negotiated and incompatible
  versions, installation-pin mismatch quarantine, enrollment rate limiting, and request
  body rejection.
- Verify both committed gateway contract snapshots are current, run focused gateway and
  fleet-contract tests during development, then run the linked core repository's full
  `just check` gate.
