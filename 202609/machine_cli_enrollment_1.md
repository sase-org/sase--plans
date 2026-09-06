---
tier: tale
title: Complete the machine CLI and enrollment phase
goal:
  Operators can explicitly enroll, inspect, maintain, and diagnose remote SASE machines
  through the CLI and onboarding flow without implicit trust, ambient network work, or
  secret exposure.
size: medium
proposed_by: bbugyi200.athena.sase-xe.8
bead: sase-xe.8
create_time: 2026-09-06 19:22:05
status: wip
---

- **BEAD:**
  [sase-xe.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.8.md)

# Plan: Complete the machine CLI and enrollment phase

## Goal

Complete bead `sase-xe.8` by adding a production-quality `sase machine` command group,
an optional remote-machine enrollment step in `sase init`, and dispatch-specific doctor
checks. The implementation must consume the provider/config/credential contracts from
`sase-xe.7` and the fleet gateway authentication contract from `sase-xe.4`; it must not
reimplement either prerequisite. Machine aliases are viewer-local labels, while the
gateway installation ID remains authoritative identity and the configured provider is
routing metadata.

The CLI must preserve the settled safety properties: no discovery or network access on
ordinary config loading or `machine list`; no implicit trust of discovered devices; no
secret in YAML, JSON output, logs, errors, or doctor reports; no fallback to another
endpoint/provider; and an installation mismatch remains quarantined until the operator
deliberately repairs it. Keep the whole user-reaching surface behind the existing
`remote_dispatch` beta flag while retaining read-only config diagnostics with the flag
off.

## Implementation

1. Reconcile the dependency baseline before coding. Confirm the checkout contains the
   completed `sase.dispatch` provider execution, typed layered config, credential-store,
   and flag APIs from `sase-xe.7`, and that its pinned `sase_core_rs`/linked core
   exposes the committed fleet gateway v1 contract from `sase-xe.4`. If the closed
   provider phase's stitch has not reached the checkout, refresh through the normal
   host-owned update path rather than recreating that phase or editing a sibling
   workspace. Bind the implementation to those public APIs and the contract snapshot's
   exact enrollment, hello, rotate/revoke, error, schema-version, protocol-header, and
   identity fields.

2. Add a cohesive machine service layer under `src/sase/dispatch/` and keep the CLI
   handler thin. Define immutable result models and explicit errors for local identity,
   enrolled records, discovery candidates, enrollment/repair outcomes, and live status.
   Implement a bounded fleet HTTP client (timeouts, response/body limits, HTTPS
   validation, version headers, typed response validation, and stable mapping of gateway
   errors) for `/api/fleet/v1/enroll` and `/api/fleet/v1/hello`; support dependency
   injection so tests never need ambient network access. Treat the target-generated
   bootstrap material as a pasteable, validated enrollment bundle containing the
   bootstrap ID, one-time secret, installation pin, and supported versions. Read that
   material only from the injected `_init_input_func`/TTY prompt path, never a public
   secret-valued command-line option.

3. Implement machine-registry mutations over the active local machine overlay with the
   repository's source-preserving YAML helpers and chezmoi write-target policy. `add`
   resolves an explicit discovery candidate or the supplied endpoint/provider, performs
   enrollment, verifies the returned authoritative installation ID against the bootstrap
   pin, stores the bearer token under a non-secret `credential_ref`, then records the
   alias/provider/endpoint/pin/reference with rollback or actionable recovery on a
   partial write. `remove` removes the enrolled record and its local stored credential
   without deleting follow history. `rename` moves only the alias key and preserves the
   installation pin and credential reference. `repair` requires an explicit new target
   bootstrap bundle, re-enrolls the quarantined/reinstalled endpoint, rotates the local
   secret reference, and changes the identity pin only after a successful authoritative
   response. Clear config caches after writes and preserve unrelated YAML/comments.

4. Implement read and provider operations. `discover` invokes only explicitly enabled
   discovery providers through the bounded provider runner, merges deterministic
   candidates without enrolling them, and renders provider and reachability hints plus
   provider-local diagnostics. `list` reads no provider and performs no network I/O; it
   renders the configured local `id.machine_name` row first, followed by aliases sorted
   deterministically with provider, endpoint, redacted/present pin/reference metadata,
   and any bounded cached status available from prior explicit probes. `status` resolves
   the requested aliases (or all enrolled aliases), resolves credentials without
   exposing tokens, performs independent authenticated hello calls with per-host
   deadlines, validates negotiated protocol/identity/capabilities, quarantines
   mismatched pins in the result, and reports every host even when some fail. Preserve
   optional authoritative running-count fields when the available gateway contract
   supplies them; otherwise render them as unavailable rather than inventing zero.

5. Add `src/sase/main/parser_machine.py` and `machine_handler.py`. Register
   `register_machine_parser` at every required integration point: the string-lazy
   `_COMMAND_REGISTRARS` map, static `parser_full_registrars.py` import/catalog, sorted
   `entry.py` handler branch, and root help inventory where applicable. Use
   `RawDescriptionHelpFormatter`, complete descriptions/examples, alphabetically sorted
   `add`, `discover`, `list`, `remove`, `rename`, `repair`, and `status` children,
   positional required values, and short aliases for every public long option. Name the
   default child exactly `list` so the central bare-group delegation and runtime notice
   apply without custom logic. Provide stable schema-versioned JSON for scriptable
   read/result commands and colored, scannable human output with secrets redacted.

6. Integrate enrollment with `sase init` using the existing plan-then-apply framework.
   Add a `machine` `InitCommandSpec` immediately after `config` in `init_registry.py`,
   plus the explicit `sase init machine` parser/entry alias needed by coordinator
   prompts. Its planner is read-only and represents optional machine setup as an
   `InitPlan` / `InitAction`; it must not discover providers or contact hosts during
   `--check`, `--diff`, JSON planning, or noninteractive startup. The apply path is
   TTY-gated, explicitly runs discovery, presents provider/reachability-labelled
   candidates, allows the operator to select zero or more machines, and calls the same
   enrollment service as `machine add`. Existing coordinator confirmation authorizes the
   selected actions, but never authorizes silent service exposure, tailnet policy
   changes, or a guessed machine choice. Ensure config has established `id.machine_name`
   before this spec runs and preserve the established `_init_input_func`/`_init_stdin`
   injection seams.

7. Add `src/sase/doctor/checks_dispatch.py` and lazily register
   `dispatch_check_specs(context)` in stable order in `doctor/runner.py`. Default checks
   stay offline and validate alias/local-name conflicts, parsed machine records,
   provider/endpoint/reference shape, installation pins, credential
   presence/readability, and config-to-credential coherence without serializing secret
   values. Mark live reachability/hello checks `deep=True`, give them bounded
   per-machine failures and precise remediation, and skip network activation cleanly
   when the beta flag is off. Keep config/coherence diagnostics useful in both flag
   states so disabled features do not conceal malformed durable state.

## Verification

Add focused tests under `tests/dispatch/`, `tests/main/`, and `tests/doctor/` for:

- exact parser registration parity, sorted help and options, short aliases, bare
  `machine -> machine list` delegation notice, handler routing, and stable JSON;
- local-row-first offline listing with an assertion that providers/network are never
  touched, deterministic discovery aggregation, partial per-provider failures, and both
  beta-flag states;
- successful manual and discovered enrollment, ambiguous/missing candidates, invalid or
  replayed bootstrap bundles, incompatible protocol, bounded responses/timeouts,
  identity quarantine, secret-redaction, config/credential rollback, remove, rename, and
  deliberate repair while preserving identity/follow semantics;
- init registry order (`config`, then `machine`), check/diff/JSON no-network behavior,
  TTY gating, injected selections and bootstrap prompts, decline/zero-selection
  behavior, and reuse of the exact machine enrollment path;
- default doctor coherence findings, missing/orphan credentials, malformed pins and
  records, deep-only reachability, partial host failure, quarantine, redaction, and
  flag-off behavior.

Run focused machine, init, parser/help/default-list, dispatch, doctor, config-schema,
required-plugin, feature-flag, and fleet-contract tests during development. Because this
changes the SASE repo, read the mandatory lint/test memory before final verification,
run its required install/check sequence, and fix only failures caused by this phase.
Before closing, run `sase bead epic-symbols sase-xe.8`; resolve every remaining symbol
or re-key its Justfile entry to `sase-xe` or a still-open later phase. Close only
`sase-xe.8` with a note naming the checks that passed. Do not close the parent epic or
any ancestor/flag bead, and record any genuinely out-of-scope discovery only as a
`PROPOSED FOLLOW-UP:` note on this phase.
