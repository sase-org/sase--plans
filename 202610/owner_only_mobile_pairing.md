---
tier: tale
title: Make mobile pairing issuance owner-only
goal:
  Prevent other local accounts from minting mobile bearer tokens while preserving
  owner-issued device pairing and existing sessions.
size: medium
proposed_by: bbugyi200.athena.sase-1mc
proposed_with: codex/gpt-6-astra
bead: sase-1mc
create_time: 2026-10-17 10:34:19
status: wip
---

- **BEAD:** sase-1mc

# Make mobile pairing issuance owner-only

Complete task bead `sase-1mc`: an unrelated local account must be unable to obtain a
mobile bearer token simply by contacting the gateway. Implement this as one coordinated
change across the Rust backend, Python CLI, and the Android contract tests. The domain
and compatibility boundaries are known, so one coding agent can implement and verify the
whole change without separately landed phases.

## Confirmed scope

The bead was `in_progress` when read. Source inspection confirms the reported flaw; this
planning turn did not probe the running gateway or launch an agent through it.

- In `sase-core`, `crates/sase_gateway/src/routes/mobile_handlers.rs::pair_start`
  accepts unauthenticated JSON, inserts an in-memory challenge, and returns both its ID
  and code. `pair_finish` consumes the challenge and issues a bearer token.
- `routes/tests/session_handlers.rs::pair_start_returns_short_lived_code_without_token`
  and `server.rs::listener_smoke_exercises_pairing_auth_and_session` explicitly expect
  this unauthenticated issuance to succeed. There is no already-landed owner check in
  the inspected implementation.
- In `sase`, `src/sase/integrations/mobile_gateway.py::_request_pairing` posts without
  credentials. Foreground `start` and service-managed `pair` both use it. The service
  host starts the binary without issuing a challenge.
- In `sase-android`, `SessionRepository.pairManually` and QR pairing already use only
  `finishPairing` with an owner-supplied ID and code. The generic API client and
  contract tests additionally expose `startPairing`, but the pairing UI does not call
  it.

Before implementation, reread `sase-1mc` with a task-specific audit reason. If it is now
closed, inspect the landed changes before doing anything; reopen only if this scope is
genuinely unfinished. Open `sase-core` with `/sase_repo` and `sase repo open sase-core`,
and Android with `sase repo open gh:sase-org/sase-android`. Work only at their printed
paths and read each repository's applicable instructions.

## Chosen security boundary

Issue challenges through the owner's local CLI and a Rust-owned protected challenge
store. Remove HTTP challenge issuance entirely. Keep HTTP redemption and existing
paired-device tokens compatible.

This chooses the bead's local-code-delivery option. A persistent administration secret
sent over loopback would also require protecting against accidental proxying, redirects,
and a different user's counterfeit listener. A local issuer requires no such secret or
new privileged network transport. A Unix socket would preserve the in-memory store but
require another listener and its lifecycle; a small cross-process challenge store fits
the existing local Rust-binding workflow.

The trusted principal is the gateway's OS account, including its already-authorized
same-user processes. Root and a compromised owner account are outside this boundary.
Code and pairing ID remain a short-lived enrollment capability intentionally shown to
the owner and entered on the device. Neither can be obtained through a read/list HTTP
API. Loopback addresses, forwarded headers, an existing mobile token, and knowledge of
the state path must never authorize challenge issuance.

There is no flag or fallback that restores unauthenticated issuance. This is a security
fix, not a staged migration that keeps the vulnerable behavior callable.

## Implementation

### 1. Add the protected local issuer in `sase_core`

Create a focused `mobile_pairing` domain module in `crates/sase_core/src/`, with
schema-versioned request/response records, typed errors, and functions to issue and
consume challenges. Export it from the core module tree and add/register a thin PyO3
issuer binding, such as `mobile_pairing_issue`, in `crates/sase_core_py`. Do not put
domain logic or filesystem authorization in Python.

- Store challenges under `<sase_home>/mobile_gateway/pairing/`. The default home must
  match the gateway's existing `SASE_HOME`, then `HOME/.sase`, resolution. The caller
  may supply the effective explicit state root.
- Keep the current CLI response fields (`schema_version`, `pairing_id`, six-digit
  `code`, `expires_at`, `host_label`, optional fingerprint) and default five-minute
  lifetime. Use a cryptographically secure random generator for the ID and code; give
  IDs at least 128 bits of entropy. Inject time into core operations for tests.
- Persist only a domain-separated hash of the ID/code combination, its expiration, and
  necessary metadata. Return the raw code once to the local caller. Never expose codes
  or hashes through public health, structured errors, Debug output, or audit logs.
- Make issue/prune/consume transactions safe across independent CLI and gateway
  processes: use a securely opened, stable interprocess lock and atomic replacement of a
  small versioned store. An in-process mutex alone is insufficient.
- Preserve the current one-attempt semantics: a matching ID is consumed even if its code
  is wrong or expired. Commit consumption before issuing a device token. Two
  simultaneous valid redemptions can issue at most one token. A persistence failure must
  not mint a token or restore an already-consumed challenge.
- Remove expired entries during normal store operations; unknown IDs and malformed input
  cannot create entries. IDs are lookup keys, never untrusted filesystem paths. Pending
  challenges survive a gateway restart only until their original expiration.

Filesystem protection is part of the fix, not just an umask assumption:

- On Unix, create the dedicated directory as `0700` and data/lock/temporary files as
  `0600` from their first creation, including under a permissive umask. Verify current
  effective-user ownership, object type, and restrictive modes before trusting existing
  entries. Reject symlink, hard-link, wrong-owner, or insecure preexisting protected
  objects instead of following or silently repairing them.
- Validate the containing state directories against substitution by another account.
  Reject attacker-writable containing directories and unsafe custom roots; use
  descriptor-relative/no-follow operations where necessary to avoid check/open races.
  Ordinary owner-controlled existing `0755` state directories can remain; the new
  protected subtree supplies confidentiality. Canonicalize ancestors consistently so
  macOS `/tmp` aliases do not spuriously fail. Root-owned sticky temporary ancestors
  used by private test directories need deliberate handling, not a blanket world-write
  exception.
- Keep all targets compiling. On platforms without implemented owner-isolation
  enforcement, fail new challenge issuance and redemption closed with an actionable
  unsupported-platform error. Do not copy the existing fleet store's non-Unix no-op
  permission branch. This initial implementation guarantees pairing on Linux/macOS; new
  pairing on other platforms requires equivalent owner ACL enforcement before it may be
  enabled. Existing bearer sessions remain usable there.
- Missing store means no pending challenge to redeem. Corrupt data, invalid permissions,
  and failed locking/writes are explicit failures, never an empty permissive store or an
  invitation to fall back to HTTP issuance.

Test the Rust API itself and the registered Python binding, including schema errors and
secret-free failure messages. Binding tests must demonstrate that a separately
constructed store can redeem a challenge issued through the binding.

### 2. Make the gateway redeem only owner-issued challenges

Update `crates/sase_gateway/src/routes/router.rs`, `mobile_handlers.rs`, `state.rs`, and
supporting errors/wires to use the core store.

- Remove the `POST /api/v1/session/pair/start` route. Requests now receive the existing
  structured `404` response, including when a mobile bearer token is supplied. Remove
  the in-memory challenge map and its issuance handler; do not leave a second issuer.
- Have `pair_finish` validate the existing request wire and atomically consume a core
  challenge, then use the existing `DeviceTokenStore::pair_device`. Preserve the current
  `pairing_rejected` and `pairing_expired` responses, token format, device persistence,
  and audit behavior. Storage failures return a redacted server error.
- Add an additive, non-secret health capability identifying local issuance support, for
  example `pairing: { issuance: "owner_local_v1" }`. Use a representation whose absence
  old clients tolerate; existing Android JSON parsing must accept the extra field. The
  marker supports upgrade diagnostics, not authorization.
- Ensure every real server entry point and router construction uses the same
  protected-store redemption path. There must be no test-only permissive constructor
  available to production and no dependence on which network address was bound.
- Update `contract.rs`, the committed mobile API snapshot, gateway README, and health
  fixtures: only health and code redemption are unauthenticated mobile operations; start
  is no longer an available route. Describe local issuance separately from the HTTP
  route table. Retain required redemption wire shapes and schema version 1. Generate the
  snapshot with the documented
  `UPDATE_MOBILE_CONTRACT=1 just test -p sase_gateway committed_` recipe, not by editing
  generated JSON independently.
- Adapt pairing test helpers to issue through the real core API. Keep explicit negative
  HTTP tests so changing the helper cannot accidentally conceal this regression.

### 3. Route Python pairing through the local binding

Update `src/sase/integrations/mobile_gateway.py` and its tests. Keep Python responsible
for argument/config resolution, readiness checks, presentation, and binding delegation.

- Replace `_request_pairing`'s HTTP POST with the Rust issuer. Pass the effective state
  root consistently in foreground startup and the separate `pair` command: CLI override,
  configured `mobile_gateway.state_dir`, then the gateway default.
- Add `pair -H/--state-dir`, matching `start`, so a gateway started with a nondefault
  root remains pairable. Read `cli_rules.md`, give clear local-owner wording to help,
  and regenerate CLI documentation/completions through the repository's normal fix
  workflow. No administrator secret CLI option or config key is needed.
- Keep HTTP health/readiness checks, but require the new local-issuance capability
  before issuing or printing a challenge. Give a clear upgrade/restart error for an old
  gateway. Health requests send no challenge, code, or credential; an unreachable
  gateway or missing Rust binding must not trigger an unauthenticated POST fallback.
- Preserve foreground process shutdown/error behavior and the existing service-host
  delegation: service startup still prints guidance to run `sase mobile gateway pair`
  and does not put pairing codes in long-lived service logs.
- Retain the existing owner-facing code/ID/expiration output. A configured bind address
  or port chooses the readiness target, never a remote authorization mechanism; the
  issuer must run as the owner on the machine holding the gateway's state root.

### 4. Align the Android consumer and documentation

The Android pairing UI should continue to work without a new flow. In the opened Android
repo, update the relevant contract snapshot sections, API tests, and fixtures so none
claim that remote challenge creation succeeds. Remove the unused
`GatewayApiClient.startPairing` method and its imports/test success path. Keep DTOs only
if they still model locally supplied pairing/QR data. Preserve manual/QR redemption,
token storage, and existing sessions. Avoid unrelated API renames or refreshing
unrelated stale contract sections as part of this fix.

In `sase`, read `docs_writing.md` and the docs-writing contract before updating:

- `docs/reference/integrations/mobile-gateway.md`: replace the unauthenticated
  curl-start example with local owner CLI issuance, explain the shared state root,
  supported owner-permission boundary, short TTL and one-attempt redemption, and correct
  the statement that loopback alone is safe on a multi-user host.
- `docs/contribute/mobile-mvp-runbook.md` and `mobile-gateway-api.md`: fix affected
  pairing, restart, and smoke-test instructions. Keep code/capability details in the
  appropriate reference/contributor pages.
- Explain upgrade ordering: install the matching core and CLI, then restart an old
  gateway before relying on the fix. Existing paired device tokens remain valid; this
  change does not retroactively revoke tokens obtained before the upgrade. Do not
  restart real services, clear real devices, or reinstall globally during tests.

No memory edit is required. No unrelated fleet authorization or device revocation UI
work belongs in this task.

## Verification and acceptance

Use isolated temporary state roots and ephemeral listener ports. Do not probe the real
owner's running gateway or launch real agents to demonstrate the vulnerability.

1. Rust security regressions prove anonymous and bearer-authenticated HTTP start both
   fail without returning a code or creating a challenge/device; unauthenticated
   privileged routes remain denied. Repeat the assertion over an actual temporary TCP
   listener so route middleware/construction is covered.
2. Local issuance followed by HTTP finish succeeds, and the returned token authenticates
   a harmless session read. Unknown, expired, incorrect, and replayed codes fail. A
   wrong attempt burns the ID. Concurrent valid finishes mint exactly one token, even
   through independently created store handles/processes; gateway restart does not
   extend expiry. Existing stored devices still authenticate and revoke as before.
3. Core tests cover restrictive creation under permissive umask, file/directory and lock
   validation, symlink/hard-link refusal, hostile containing paths, corruption,
   atomic-write failure, and cross-process serialization. Test ownership decisions with
   metadata/policy seams where another UID cannot be created without privileges; do not
   require sudo. Include macOS path alias coverage and non-Unix fail-closed compile/test
   coverage. Assert raw code/token absence in durable data and logs.
4. Python tests cover foreground and service-managed entry points, default/env/config/
   explicit state roots, `pair -H`, health capability mismatch, absent binding/store
   errors, and unchanged printed fields. Assert there is no HTTP start request or
   insecure retry. Run the new binding integration against the locally built core.
5. Android targeted `GatewayApiClientTest`, `MobileApiContractTest`,
   `GatewayDtoFixtureTest`, and `SessionRepositoryTest` prove the contract changes and
   unchanged manual redemption. Follow its README/CI with
   `./gradlew testDebugUnitTest lintDebug assembleDebug --no-daemon --stacktrace` when
   the Android toolchain is available. Provision the normal workspace test prerequisites
   or use a SASE monitor; do not claim an unrun Android check passed.
6. Run targeted Rust tests with `just test -p sase_core mobile_pairing`, the relevant
   `sase_core_py` binding tests, and gateway session/listener tests; never bare cargo.
   Then format and run `sase tool run check` from both the core and primary checkouts.
   In `sase`, run `just fix` before the final gate. Read `lint_and_test.md` and use
   `/sase_monitor` for long verification. Do not run `just check-full` unless explicitly
   requested in a later instruction.

Investigate failed checks and identify genuine out-of-scope discoveries with
`/sase_new_task`, including current bead `sase-1mc` and independent evidence. Do not
silently weaken checks or call unresolved failures unrelated.

## Completion

After implementation and verification, run the user-requested command:

```bash
sase bead close sase-1mc --note "<actual checks and results proving HTTP cannot issue challenges, owner CLI pairing works, permissions and replay are enforced, and clients remain compatible>"
```

Do not close the bead merely for authoring or approving this plan. Keep it open through
unfinished implementation or verification. Submit the final declaration with commit
decisions for every changed repo, including the external Android repo. Follow
`docs/contribute/rust-backend.md`'s CI-pin workflow: the host commits/pushes the changed
linked core first and writes its SHA to `sase-core-revision.txt` before committing
`sase`. Do not invent a future SHA, commit manually, or edit the published dependency
window. Confirm the finalization context covers both core and primary changes so the new
binding cannot land with the old source pin.
