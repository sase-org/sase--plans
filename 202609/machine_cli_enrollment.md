---
tier: tale
title: Machine management CLI, enrollment onboarding, and diagnostics
goal:
  Users can enroll, inspect, repair, rename, and remove remote machines through a safe
  CLI and optional init flow, with local and deep doctor validation.
size: medium
proposed_by: bbugyi200.athena.sase-xe.8
bead: sase-xe.8
create_time: 2026-09-09 20:00:39
status: wip
---

- **PARENT:**
  [202609/remote_dispatch_fleet.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.8.md)

# Machine management CLI, enrollment onboarding, and diagnostics

## Goal

Complete the `machine-cli` epic phase by adding a feature-gated `sase machine` command
group with `add`, `discover`, `list`, `remove`, `rename`, `repair`, and `status`;
connect enrollment and authenticated hello requests to the fleet gateway; offer optional
remote-machine enrollment from `sase init`; and make `sase doctor` validate enrolled
records locally while reserving live probes for deep checks.

The group must preserve one machine vocabulary: the local `id.machine_name` identity is
the first list row and enrolled aliases follow it. Discovery and status are explicit
network operations, listing/rename/removal are local, and enrollment never turns a
discovery candidate into a trusted machine without a target-issued bootstrap secret.
Credentials remain outside YAML, identity pins survive alias/provider changes, and
follow records survive unenrollment.

## Implementation

1. **Consume and verify the prerequisite dispatch contracts before editing.** Confirm
   that the current tree contains the approved `sase.dispatch` provider/config,
   credential-store, and feature-flag APIs from `sase-xe.7`, and that the pinned
   `sase-core` exposes the fleet enrollment and hello contract from `sase-xe.4`. Use
   those APIs directly rather than inventing a second registry, provider loader,
   credential format, or connection-plan model. If the Python dependency is still absent
   after the normal workspace refresh, recover its accepted change before this phase and
   keep this phase's code limited to machine management. Do not modify the Rust gateway
   contract unless a verified incompatibility requires it.

2. **Add a typed machine-management service and bounded gateway client.** Under
   `src/sase/dispatch/`, implement the orchestration shared by CLI, init, and doctor:
   resolve aliases and discovery candidates deterministically; request a selected
   provider's validated connection plan; POST the schema-version-1 enrollment envelope
   to `/api/fleet/v1/enroll`; and GET authenticated `/api/fleet/v1/hello` with the
   protocol-version header. Validate response shapes, negotiated version, target machine
   selector, installation identity, credential record, bearer token, and explicit
   quarantine outcome before any durable write. Keep transport injectable for tests,
   enforce bounded timeouts/body sizes, normalize failures into secret-free typed
   diagnostics, and never include raw bootstrap or bearer material in logs, exceptions,
   JSON output, connection plans, or status caches.

   Commit successful enrollment as one recoverable local operation: store the token in
   the phase-7 mode-restricted credential store, then source-preservingly write the
   machine record (`use`, endpoint, authoritative `installation_id`, and
   `credential_ref`) to the selected writable configuration layer and clear config
   caches. On failure, roll back newly written credential material. Provide local
   rename/remove operations that preserve installation identity and follow records;
   removal deletes the local credential after removing the record. Repair must be an
   explicit re-enrollment/re-pin path using a fresh bootstrap secret and must never
   accept an identity mismatch silently. Maintain only bounded, non-secret status data
   (last seen, negotiated version, capabilities, and authoritative counts when the
   gateway supplies them) for offline list display.

3. **Register and implement the `sase machine` command group.** Add the lazy registrar
   in `parser_registry.py`, the static registrar import/catalog entry in
   `parser_full_registrars.py`, an alphabetical handler branch in `entry.py`, and a
   `parser_machine.py` using `RawDescriptionHelpFormatter`, useful examples, sorted
   subcommands, positional arguments for required values, and short aliases for every
   public long option. Add `machine_handler.py` as a thin shim into the dispatch CLI
   module. Name the default child exactly `list` so the central default-list machinery
   makes bare `sase machine` delegate with the standard notice; include `machine` in
   compact root help only if it improves the curated list without displacing a more
   common command.

   Implement colored human output plus stable JSON for automation. `add` accepts an
   alias, optionally selects `--endpoint` and `--use`, prompts through the injected
   `_init_input_func` convention for bootstrap data, and runs enrollment end to end.
   `discover` invokes only explicitly enabled provider discovery and enrolls nothing.
   `list` performs no network work and renders the local identity first, then enrolled
   records with cached/unknown state. `remove` deletes only local enrollment state and
   preserves follows. `rename` changes only the alias. `repair` clearly displays the
   pinned-versus-presented identity and requires deliberate bootstrap-backed repair.
   `status [NAME ...]` probes selected/all enrolled machines independently and reports
   reachability, protocol, capabilities, identity-pin state, and counts without one hung
   host blocking the rest. With `remote_dispatch` disabled, the command remains
   discoverable but exits with a concise beta-gate explanation and no provider import,
   network, or state mutation.

4. **Integrate optional enrollment into `sase init` without making checks noisy.** Add
   an explicit `machine` init subcommand and an `InitCommandSpec` immediately after
   `config` in `init_registry.py`. Its planner must remain read-only, represent proposed
   config/credential effects through `InitPlan`/`InitAction`, and make
   `--check`/`--diff`/`--json` deterministic and secret-free. Bare onboarding should
   offer machine setup only in an interactive TTY with the feature enabled, after owner
   identity is usable; it may explicitly discover candidates, show provider and
   reachability hints, allow manual HTTPS entry, enroll selected machines through the
   same service as `sase machine add`, and treat the user's existing onboarding choice
   as authorization for those selected actions. Non-interactive checks, already
   enrolled/current configurations, and the flag-Off state must not discover, prompt,
   contact a host, expose a service, or alter Tailnet policy. Preserve the coordinator's
   plan-then-apply semantics and test-injection hooks rather than special-casing writes
   in the parser.

5. **Register dispatch doctor checks.** Add `src/sase/doctor/checks_dispatch.py` and
   lazily include `dispatch_check_specs(context)` in the stable doctor registry. Default
   checks load and validate machine records without provider imports or network:
   aliases, provider/endpoint shape, unique and valid installation pins, credential
   references, credential availability/mode, and quarantine/cache coherence. Emit
   actionable, redacted `DiagnosticCheck` results. Add a separate `deep=True`
   reachability check that uses the shared authenticated hello service with per-host
   bounds and honest partial failure. Empty configuration and flag-Off operation should
   be cheap and explicit, not errors.

## Verification

- Add focused parser/handler tests covering all seven sorted subcommands, excellent
  help, every long-option short alias, narrow/full parser parity, bare-group delegation
  metadata and runtime notice, stable JSON, local-first ordering, and both feature-flag
  states.
- Add service/client tests with fake providers and an injected HTTP transport for
  successful enrollment/hello, incompatible protocols, bad response schemas, bootstrap
  rejection/expiry/replay, timeout, HTTP errors, quarantine, pin mismatch, credential
  rollback, rename/remove/repair, secret redaction, no-network list, and per-host status
  isolation.
- Add init tests proving registry order (`config`, then `machine`), plan serialization
  and diffs, TTY-gated prompting through `_init_input_func`, explicit candidate
  selection/manual HTTPS setup, no enrollment on discovery alone, and zero provider or
  network calls for check/JSON/non-TTY/flag-Off paths.
- Add doctor tests for empty/current, malformed record, missing or insecure credential,
  duplicate identity pin, quarantine, redacted output, lazy registry loading, default
  no-network behavior, and deep reachability outcomes.
- Run the focused machine, dispatch, init, parser, feature-flag, and doctor suites.
  Because this changes the SASE repository, read the required lint/test memory after
  editing, run `just install` if required for the ephemeral checkout, and run the
  mandated `just check` agent lane. Finally run `sase bead epic-symbols sase-xe.8`,
  resolve or re-key every remaining phase symbol, and close only `sase-xe.8` with a note
  naming what passed.
