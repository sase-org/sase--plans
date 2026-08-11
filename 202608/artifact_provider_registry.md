---
tier: tale
title: Provider registry, plugin hooks, and config
goal:
  Artifact ref and file-hook providers can be discovered, validated, merged into config,
  diagnosed, and initialized without breaking launch paths.
size: medium
proposed_by: bbugyi200.athena.sase-js.3
bead: sase-js.3
create_time: 2026-08-11 15:43:29
status: done
---

- **PARENT:**
  [202608/artifact_ref_contract.md](https://github.com/sase-org/sase--plans/blob/main/202608/artifact_ref_contract.md)
- **BEAD:**
  [sase-js.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-js/sase-js.3.md)

# Provider registry, plugin hooks, and config

Implement phase `sase-js.3` of the approved Artifact Reference Contract design. This
phase establishes provider discovery and configuration only; prompt-context resolution,
use recording, file capture, publication, and ACE presentation remain in later phases.

## Implementation

1. Add a `sase_artifact` pluggy project with ref-provider and file-hook-provider
   hookspecs. Discover the `sase_artifact_refs` and `sase_file_hooks` entry-point groups
   once per config-token registry assembly, honor their independent disable variables,
   retain distribution/version provenance, and register SASE's builtin providers through
   the same host path.
2. Build a deterministic artifact-provider registry. Validate document specs and compute
   stable digests through `sase_core_rs`, reject duplicate provider ids/kinds and
   reserved third-party kinds, and expose structured diagnostics while omitting invalid
   providers from the effective registry. Include the builtin `plan` document provider
   plus the builtin entry-kind descriptors needed by downstream phases.
3. Replace the retired sidecar renderer policy with provider-backed spec normalization.
   Resolve `ref.use`, deep-merge inline field overrides with list replacement, support
   fully inline document specs, keep `filters.path_globs` as a deprecated alias for
   `inventory.globs`, preserve sidecar role/source provenance, and fail soft per invalid
   entry. Prove provider-backed and equivalent inline config normalize identically.
4. Extend file-hook configuration with provider templates. Resolve `file_hooks[].use`,
   deep-merge user overrides with list replacement, allow providers to require local
   fields such as `command`, retain local-project auto-scoping, and skip unresolved or
   invalid entries with actionable warnings rather than breaking launch paths.
5. Update the JSON schema and plugin inventory for both provider groups. Cover the
   declared ref-spec fields and the provider-backed file-hook form while retaining
   inline compatibility.
6. Make repository initialization add `ref: {use: plan}` to the builtin plans sidecar
   idempotently. Do not overwrite an existing provider-backed or inline `ref` mapping,
   including when the plans role was already configured.
7. Extend doctor diagnostics to report missing providers, invalid/duplicate specs,
   unsupported schema versions, invalid overrides, and the clone-versus-installed-plugin
   distinction with config paths and install guidance. Runtime loading must remain
   fail-soft even for every doctor-visible failure.

## Verification

- Add focused unit tests for hookspec discovery, deterministic ordering/cache tokens,
  duplicate and invalid provider rejection, builtin registration, merge/list-replacement
  semantics, inline-versus-`use` parity, file-hook required overrides, fail-soft
  logging, schema acceptance/rejection, init idempotency/non-clobbering, and doctor
  findings.
- Run focused affected tests while iterating.
- Run `just check` after the implementation and inspect the final diff/status before
  closing `sase-js.3` with a note naming the verification performed.
