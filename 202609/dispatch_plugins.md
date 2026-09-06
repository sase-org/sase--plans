---
tier: tale
title: Dispatch provider plugins, configuration, and credentials
goal:
  Dispatch providers are lazy, isolated, typed, securely configured, and beta-gated for
  later remote-fleet phases.
size: medium
proposed_by: bbugyi200.athena.sase-xe.7
bead: sase-xe.7
create_time: 2026-09-06 16:29:07
status: wip
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.7.md)

# Dispatch provider plugins, configuration, and credentials

## Goal

Complete phase `sase-xe.7` by adding the provider-neutral Python dispatch layer
described by `plan:202609/remote_dispatch_fleet.md` and the consolidated remote-dispatch
research. The result must inventory providers without importing them, execute only the
selected provider in a cancellable bounded subprocess, ship usable Tailnet and HTTPS
built-ins, parse enrolled-machine configuration with source provenance, keep credentials
out of YAML in a private store, and leave every runtime activation surface behind the
temporary `remote_dispatch` beta flag.

This phase does not add the `sase machine` CLI, enrollment handshake, federation worker,
or `%dispatch` directive; later phases consume the contracts built here. It does not
change the Rust protocol: the connection-plan contract and its validation binding
already exist in `sase-core` from phase `sase-xe.2`.

## Implementation

1. Scaffold the planned beta flag through the supported feature-flag workflow, never by
   hand-editing the registry. Use `sase flag new remote_dispatch` with authored behavior
   stating that enabling the flag activates configured dispatch providers, disabling it
   keeps dispatch provider execution inert while local behavior is unchanged, and
   removal waits for the epic's fleet-wide acceptance phase. Paste the generated
   registry entry, keep the dedicated removal bead open for phase `sase-xe.15`, and add
   explicit On/Off call sites around provider activation rather than around
   schema/config inspection.

2. Add a `sase.dispatch` package with immutable, schema-versioned request/result models
   and a Pluggy hookspec for exactly these keyword-argument hooks:
   `dispatch_provider_spec()`, `dispatch_discover(request)`, and
   `dispatch_connection_plan(request)`. Pin the argument name `request`, validate and
   serialize every cross-process envelope, use the singular Pluggy project name while
   the entry-point group remains `sase_dispatch`, and expose only serializable provider
   metadata, discovery candidates/diagnostics, and Rust-compatible connection plans.
   Provider code must never own enrollment, credentials, retries, caching, agent
   listing, launch, or lifecycle actions.

3. Extend plugin inventory and packaging for `sase_dispatch`. Treat it as a provider
   group so `collect_plugin_inventory()` reads entry-point metadata without calling
   `load()`, `SASE_DISABLE_PLUGIN_DISPATCH` follows the existing convention, and plugin
   catalog/version surfaces recognize it. Register the built-in Tailnet and HTTPS entry
   points in `pyproject.toml`, canonicalizing the distribution-owned entries to
   `builtin@tailnet` and `builtin@https` for configuration while preserving package and
   entry-point provenance.

4. Implement metadata lookup plus selected-provider execution using the finalizer trust
   model. The parent process resolves exactly one canonical provider reference from
   entry-point metadata, enforces global/group disable variables and duplicate/mismatch
   diagnostics, checks the beta flag, and sends a bounded JSON request to a dedicated
   worker module. The worker alone loads and registers that one entry point, invokes the
   matching Pluggy hook with keyword arguments, validates the typed result, and emits a
   bounded JSON response. Reuse or safely generalize the existing bounded subprocess
   primitive to support an explicit cancellation signal as well as timeout,
   process-group termination, output caps, a minimal environment, and provider-local
   failure results; one provider failure must not import or disable unrelated providers.

5. Implement the two built-in providers against the shared contracts. Tailnet performs
   discovery only when explicitly invoked, runs `tailscale status --json` under the host
   deadline, and defensively accepts missing/extra fields and alternate peer shapes
   while preserving device/DNS labels and treating visibility/online values only as
   hints. A missing or failing binary yields a provider-scoped unavailable diagnostic.
   HTTPS has no ambient discovery and builds a static, validated TLS connection plan.
   Both connection-plan hooks produce only serializable routing metadata and pass their
   final mapping through `sase_core_rs.fleet_validate_connection_plan`; URLs must be
   absolute HTTPS without embedded credentials, identity pins and credential references
   remain opaque, and provider choice remains routing metadata.

6. Add strict dispatch configuration and secret storage. Extend
   `src/sase/config/sase.schema.json` plus the comments-as-documentation defaults in
   `src/sase/default_config.yml` with `dispatch.discovery` entries using `use:` and a
   `dispatch.machines` mapping whose records contain `use`, HTTPS endpoint, pinned
   `installation_id`, and `credential_ref` plus bounded provider/TLS options needed by
   the connection contract. Default discovery includes `builtin@tailnet`, while an empty
   machine map performs no imports, subprocess launches, network work, or timers during
   ordinary config loading. Build frozen dataclasses and replay `load_config_layers()`
   like finalizer config: validate allowlists and IDs per layer, retain field
   provenance, merge machines by alias and higher-layer field overrides, replace
   explicitly supplied discovery lists, reject plugin-config activation, collect
   actionable unknown/type diagnostics, and preserve parsing/diagnostics even while the
   beta is Off. The generic `use:` walk must continue to require third-party prefixes in
   `plugins.required`.

7. Store raw bearer credentials only under the SASE home in a dispatch-specific JSON
   store with a `0700` parent directory and `0600` file, atomic replacement, bounded and
   schema-checked reads, stable named `credential_ref` keys, and explicit get/set/delete
   APIs for the enrollment phase. Support environment-variable indirection as a
   reference source without copying those secrets into YAML or logs. Reuse the existing
   sensitive-key redaction convention for diagnostic/display projections, and ensure
   config, provider requests, exceptions, and serialized connection plans contain only
   credential references, never token values or authorization headers.

## Verification

Add focused tests for the exact hook signatures and schema versions, metadata-only
inventory (including an asserted zero-load fake provider), selected-provider isolation,
timeout/cancellation/output bounds, malformed worker/provider responses, disable
environment variables, and both `remote_dispatch` flag states. Add fixture-driven
Tailnet parsing tests for absent binaries and missing/extra/status-shape fields,
HTTPS/TLS validation tests through the Rust binding, layered config/provenance and
unknown-key diagnostics, schema/default parity, third-party `use:` requirements,
credential file and directory modes, atomic CRUD, env indirection, and
redaction/no-secret envelopes.

Run the focused dispatch, inventory, config-schema, required-plugin, feature-flag, and
fleet-contract tests first. Because this changes the SASE repository, read the mandatory
lint/test memory before final verification, run its required `just install` prerequisite
if applicable, then run the agent-default `just check` lane. Before closing the phase,
run `sase bead epic-symbols sase-xe.7` and resolve every remaining symbol or re-key its
Justfile entry to `sase-xe` or a still-open later phase. Finally close only `sase-xe.7`
with a note naming the passing checks; do not close the parent epic or the flag-removal
bead.
