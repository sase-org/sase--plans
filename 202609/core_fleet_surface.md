---
tier: tale
title: Package the gateway and expose the fleet setup surface
goal: A normal sase-core-rs wheel runs the gateway, issues policy-preserving local
  fleet bootstraps, and advertises its canonical fleet protocol through public health.
size: medium
proposed_by: bbugyi200.athena.sase-xe.16.1
bead: sase-xe.16.1
status: done
---

- **PARENT:** [202609/remote_dispatch_completion.md](remote_dispatch_completion.md)
- **BEAD:**
  [sase-xe.16.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.16.1.md)

# Plan: Package the gateway and expose the fleet setup surface

## Goal

Complete `sase-xe.16.1` in the linked `sase-core` repository so a normal `sase-core-rs`
wheel provides a runnable `sase_gateway`, Python can issue a local single-use fleet
bootstrap through the existing Rust store, and unauthenticated health advertises the
gateway's supported fleet protocol versions from the canonical constant. Preserve the
existing gateway arguments, loopback-default policy, contract-output modes, store
permissions, bootstrap policy, and secret handling. Do not modify release versions or
path-dependency version pins, and do not change the primary `sase` repo unless an
epic-symbol entry must be re-keyed as part of phase closure.

## Current state and binding contracts

- `crates/sase_gateway/src/main.rs` currently owns all gateway parsing and async
  startup, while `run_federation_worker_cli` already demonstrates the reusable
  library-function/thin-bin pattern required by PyO3.
- `FleetCredentialStore::issue_bootstrap` already enforces schema validation, the
  default `FLEET_BOOTSTRAP_TTL_SECONDS` (600 seconds), a strict future expiry,
  installation-pin equality, scope normalization/defaults, supported-protocol
  negotiation, single-use secret persistence by hash, and private 0700/0600 storage. The
  binding must delegate to it and must not add an HTTP route, logging, or echoing.
- `GET /api/v1/health` returns `HealthResponseWire`. It must gain exactly an additive
  `fleet` object with `supported_protocol_versions` derived from
  `FLEET_PROTOCOL_VERSION`; the public response does not expose installation identity or
  credentials. An absent object on older gateways remains valid for consumers.
- The committed mobile and fleet snapshots are generated from `api_v1_contract_snapshot`
  and `fleet_api_v1_contract_snapshot`. The fleet snapshot's
  `protocol_negotiation.supported_versions` is currently a hard-coded `[1]` and must use
  the same constant as health.
- Wheel packaging currently exposes only `sase_federation_worker` through a Python shim
  and registered `py.allow_threads` binding. The new gateway entry point must follow
  that pattern and be smoked from an installed wheel on the CI/release platform jobs.

## Implementation

1. Extract gateway CLI startup into the `sase_gateway` library.
   - Move the argument model/parser and startup orchestration out of `main.rs` into a
     library module and export `run_gateway_cli(args)` from `lib.rs`.
   - Make `run_gateway_cli` create and own the Tokio runtime, handle both
     contract-output modes before serving, and block on the existing
     `serve(GatewayConfig)` path. Keep every current long/short option, validation/error
     string, default bind/home/push behavior, and server lifetime intact.
   - Reduce the Rust binary to the same thin error-printing/exit-status shim used by
     `sase_federation_worker`. Move and retain the existing parser tests with the
     library implementation, adding focused coverage for reusable contract-output and
     unknown-argument behavior where useful.

2. Add the packaged Python gateway and bootstrap binding.
   - Add `sase_core_rs.gateway` beside `federation_worker.py`; its `main()` passes
     `sys.argv[1:]` to a registered `gateway_main(args)` PyO3 function and returns zero
     on success. Add `sase_gateway = "sase_core_rs.gateway:main"` to
     `crates/sase_core_py/pyproject.toml` without touching package/crate versions.
   - Register `gateway_main` in `crates/sase_core_py/src/lib.rs` and invoke
     `sase_gateway::run_gateway_cli(args)` under `py.allow_threads`, mapping failures to
     `PyRuntimeError`, exactly like the federation-worker binding.
   - Register a dict-in/dict-out `fleet_issue_bootstrap(sase_home, request)` binding.
     Deserialize `FleetBootstrapIssueRequestWire`, construct `FleetCredentialStore` at
     the requested home, call `issue_bootstrap` with the Rust current-time helper under
     `py.allow_threads`, and serialize `FleetBootstrapIssueResponseWire`. Map malformed
     request/policy errors to useful Python value errors and storage/runtime failures to
     runtime errors without including the bootstrap secret in any diagnostic.
   - Document the two new public binding functions in the module API list. Add PyO3
     tests that exercise module registration and issuance through a temporary SASE home:
     dict output, canonical protocol/pin/scope fields, approximately 600-second default
     expiry, optional pin rejection, and persistence that does not contain the returned
     plaintext secret. Retain the store's existing permission and replay tests as the
     authority for 0700/0600 and single-use semantics.

3. Advertise the fleet protocol through health and contract generation.
   - Add a serializable fleet-health wire record and a `fleet` field to
     `HealthResponseWire`; construct its supported-version list from
     `FLEET_PROTOCOL_VERSION` (or the existing helper that itself derives from that
     constant) in the route.
   - Export the new wire record as appropriate and update every exact/full health-body
     assertion in `wire.rs` and `routes.rs`. Strengthen the listener smoke assertion in
     `server.rs` so the public HTTP path proves the fleet advertisement is present.
   - Teach the mobile contract schema about the new nested health record, and replace
     the fleet contract's hard-coded supported version with `FLEET_PROTOCOL_VERSION`.
     Regenerate both committed JSON snapshots through the gateway's contract-output
     commands rather than editing JSON by hand, then let the committed-snapshot tests
     prove they match generation.
   - Update the gateway README health-route description to identify the non-secret fleet
     protocol advertisement and its compatibility purpose.

4. Extend installed-wheel coverage for both console scripts.
   - In the normal CI wheel smoke, assert both executables exist, both Python shim
     `main` functions are callable, and both installed commands answer `--help`.
   - Mirror the gateway executable/module/help checks in the Linux, macOS, and Windows
     release-wheel smoke blocks alongside the existing federation-worker checks, using
     the platform's existing venv path conventions.

## Verification and closure

1. Format during implementation with the repository script, regenerate the two contract
   snapshots, and run focused tests for `sase_gateway` plus the `sase_core_py` binding
   while iterating.
2. Run `./scripts/check.sh` (the repository's full fmt-check, clippy `-D warnings`, and
   workspace-test gate) from the opened `sase-core` root. Do not substitute a crate-only
   test lane.
3. Build the abi3 wheel with maturin in a temporary output directory, install it into a
   fresh temporary virtualenv, import `sase_core_rs`, verify both shim modules and
   executables, and run both `sase_gateway --help` and `sase_federation_worker --help`.
   Confirm no generated wheel/build artifacts are left as tracked changes.
4. Review the diff for accidental secret values, release-version edits, generated-file
   drift, and unrelated changes; run `git diff --check` and confirm the linked repo is
   otherwise clean apart from this phase's work.
5. From the primary `sase` workspace, run `sase bead epic-symbols sase-xe.16.1`. Resolve
   every remaining symbol or re-key its Justfile entry to `sase-xe.16` or a still-open
   later phase before closing. Do not create task beads; record any out-of-scope issue
   as a `PROPOSED FOLLOW-UP:` note on `sase-xe.16.1`.
6. Close only the assigned phase with
   `sase bead close sase-xe.16.1 --note "<concise verification evidence>"`. Do not close
   `sase-xe.16`, `sase-xe`, or any ancestor.
