---
tier: tale
title: Machine agent hoods end to end
goal: Persist every newly allocated local agent under the configured machine hood
  while preserving legacy compatibility, bare local references and presentation, foreign-machine
  visibility, and machine-aware commit provenance.
bead: sase-8k.3
parent: sase/repos/plans/202607/agents_sidecar_repo.md
create_time: 2026-07-22 12:24:52
status: wip
---

- **PROMPT:** [202607/prompts/machine_agent_hoods.md](prompts/machine_agent_hoods.md)

# Plan: Machine agent hoods end to end

## Context and invariants

The prerequisite phases have added optional `machine_name` configuration and the authoritative Rust/PyO3 machine-hood
helpers. This phase consumes those helpers throughout the existing Python naming pipeline. When `machine_name` is
absent, every path must retain today's unqualified behavior. When it is present, the durable identity is
`<machine_name>.<local-name>`, while the local user-facing identity remains `<local-name>`; a local bare name and its
qualified spelling are equivalent for lookup and collision checks. Existing unqualified artifacts are legacy local
identities and are not renamed. A name qualified by another discovered machine is visible and resolvable in full but
cannot be launched as a new local agent.

Durable qualification must happen exactly once at identity boundaries. Derived family, clan, fork, wait, retry,
template, and auto names continue to allocate their local shape first, then qualify the root identity before reservation
or persistence. Raw artifact and registry data remain qualified, while stripping happens only in
prompt/read/presentation adapters.

## Implementation

1. **Introduce the Python machine-hood facade and local-name contract.**
   - Add `src/sase/core/machine_hood_facade.py` as the single Python entry point to the completed `sase_core_rs`
     validation, qualify, strip, and classification bindings, following the strict `require_rust_binding` facade
     pattern.
   - Layer small configuration-aware helpers over those primitives for no-op behavior without `machine_name`, durable
     local qualification, local display stripping, local/legacy equivalent spellings, canonical comparison keys, and
     known foreign-hood classification using the discovered machine overlays.
   - Keep the helpers idempotent for already-local-qualified names and family/clan descendants. Do not migrate
     historical files or infer arbitrary first dotted segments as foreign machines.

2. **Make allocation, validation, and reservation machine-aware at their narrow choke points.**
   - Update explicit launch preflight to reject known foreign-machine prefixes, compare requests through local canonical
     keys, and detect collisions against both qualified registry entries and legacy unqualified entries in both
     directions. Preserve force-reuse, clan-container, family-container, and suggestion diagnostics in local display
     form.
   - Update the registry mutation and lookup helpers plus `AgentNameNamespaceReservationIndex` so new local agent, clan,
     family, planned, and template reservations use qualified durable keys while exact-name and namespace checks treat
     legacy and qualified local spellings as equivalent. Registry rebuilds continue to retain the spelling found in
     historical artifacts.
   - Adapt auto/template/resume/wait/retry allocation and parent-side `PlannedNameAllocator` flows so token selection is
     performed against the canonicalized reservation space, prompts and template matching can use bare local forms, and
     `SASE_AGENT_PLANNED_NAME` carries the qualified durable result owned by the future artifact directory.
   - Ensure failed-launch cleanup, forced reuse, planned-reservation release, and suggestion generation resolve the same
     equivalent key rather than leaking or orphaning qualified reservations.

3. **Persist the qualified identity through every launch composition path.**
   - In child identity resolution, canonicalize explicit, planned, repeated, resumed, waited-on, and auto-allocated
     names before metadata construction, registry claim, and `SASE_AGENT_NAME` publication; retain exact legacy behavior
     when configuration is absent.
   - Qualify clan containers and members as one rooted hood (`athena.clan.member`) and family roots before adding
     `--<suffix>` members (`athena.family--code`). Update parent-side multi-prompt/chop planning and dynamic family
     attachment so scans, availability checks, promotion, metadata (`name`, `workflow_name`, `agent_family`,
     `agent_clan`), and registry conversions all agree on the durable form without double-prefixing.
   - Verify follow-up/plan-chain names, launch results, `agent_meta.json`, done-marker names, chat metadata/filenames
     where an agent identity is present, and inherited environment values all receive the same qualified identity.

4. **Resolve local references against both modern and legacy storage.**
   - Update exact agent, family, clan, template, resume, wait, chat, kill/show, and related name lookups to expand a
     local reference into deterministic qualified-then-legacy candidates. Explicit qualified local references must also
     fall back to legacy storage; foreign qualified references remain exact.
   - Preserve raw stored names in returned artifact/domain records where downstream persistence needs them, while
     returning or rendering bare local references at user-facing boundaries. Audit `@name`, `%wait`, `#fork`/`#resume`,
     template references, family attachment, and editor/mobile catalog paths so each accepts the local bare spelling.

5. **Strip only the local machine hood from every presentation surface.**
   - Compute `Agent.presented_agent_name` from the family reference or raw agent name and strip the configured local
     hood. Keep `agent_name` raw. Apply the same final-layer transformation to clan labels, completion
     names/labels/member lists, running-agent/editor/mobile catalogs, chat catalog rows, CLI output, notifications, and
     any other direct agent-name render sites found by the final audit.
   - Build Agents-tab hood/ancestor/descendant keys from the locally presented spelling so the machine hood never
     becomes a kinship group for local agents. A foreign name such as `zeus.bar` remains fully qualified and groups
     under `zeus`.
   - Keep stored chat headers and filenames qualified when sourced from a durable agent identity, but strip the local
     hood in chat listings and prompt-facing resolution. Do not rewrite existing transcript files.

6. **Record machine-aware commit provenance.**
   - Update `src/sase/workflows/commit/runtime_tags.py` so `SASE_AGENT` is locally qualified even when its fallback
     source is legacy/unqualified metadata, and `SASE_MACHINE` prefers configured `machine_name` with the existing
     hostname/environment fallback only when configuration is absent.
   - Preserve the existing sanitization, tag ownership/replacement, legacy footer parsing, and non-agent commit
     behavior.

## Tests and verification

1. Add focused facade and naming tests for configured/unconfigured behavior, idempotence, known foreign classification,
   qualification of nested/family shapes, canonical legacy equivalence, both collision directions, namespace conflicts,
   auto/template allocation order, planned reservation lifecycle, and foreign launch rejection.

2. Extend launch tests across single and multi-prompt planning, explicit/template/auto identities, resume/wait/retry
   derivation, clan creation/joining, chop plans, dynamic family promotion/attachment, environment publication, and
   persisted metadata. Assert the machine hood occurs once at the root and that missing `machine_name` leaves prior
   expectations unchanged.

3. Extend lookup and presentation tests for bare and explicitly qualified local references against both qualified and
   legacy fixtures, exact foreign lookup behavior, Agents-tab presented names and hood grouping, prompt/editor
   completion output, running/mobile catalogs, chat list/show behavior, and qualified durable chat/meta data. Include a
   `zeus.*` fixture that remains fully qualified on an `athena` machine.

4. Update commit-runtime-tag tests to cover configured machine identity, qualification of environment and metadata agent
   sources, hostname fallback without configuration, and preservation of existing footer semantics.

5. Run `just install` before checks so the linked `sase-core` binding containing the new APIs is rebuilt. Run focused
   naming, launch, lookup, TUI/completion, chat, and commit-tag suites while iterating; then grep for remaining raw
   user-facing `agent_name`/`workflow_name` render sites and unqualified reservation paths. Finish with the mandatory
   full `just check` and resolve every failure before closing `sase-8k.3` only.
