---
tier: tale
title: Owner-sharded v2 hood snapshots and beautiful overviews
goal: Publish complete project-scoped agent hoods through strict owner-sharded snapshots
  and deterministic browsable Markdown without rewriting legacy v1 data.
bead: sase-8v.4
parent: sase/repos/plans/202607/global_agent_hoods.md
create_time: 2026-07-23 16:02:00
status: done
---

- **PROMPT:** [202607/prompts/owner_sharded_v2_snapshots.md](prompts/owner_sharded_v2_snapshots.md)

# Plan: Owner-sharded v2 hood snapshots and beautiful overviews

## Context and scope

Complete bead `sase-8v.4`, the `sidecar-publish` phase of the global-agent-hoods epic, in the primary SASE repository.
The landed identity phases already provide the authoritative Rust-backed Python facade for owner validation,
local/global name conversion, top-level hood membership, family parsing, structural ancestors, family/solo link targets,
and relationship-batch validation. Reuse those operations instead of reimplementing dotted-name or family semantics in
Python.

This phase owns v2 publication and repository browsing. It does not implement transactional v2 import/family revival,
automatic post-commit invocation/outboxes, cached remote detection, or linked commit footers; those remain in later epic
phases. Keep the existing v1 reader/import path available for legacy sidecars, but stop creating or refreshing v1
transport records. Existing top-level `manifest.json` and `agents/<machine-qualified-name>` data must coexist untouched
with v2 output.

## Strict v2 wire and repository layout

Add a separate v2 model and I/O boundary under `src/sase/agents_sync/` rather than widening the current v1 dataclasses
into a loose union. Define immutable, versioned models for:

- explicit source owner and project identity;
- owner manifests that map each top-level hood to its snapshot digest and complete referenced-file set;
- hood snapshots containing stable run IDs, canonical global/local names, run state/timing, allowlisted restart
  metadata, commit records, optional file references, family/clan containers, and the Rust relationship batch;
- per-run metadata/state/commit payloads and digested prompt/chat files;
- publication counts and diagnostics.

Use canonical JSON and content SHA-256 digests, deterministic ordering, exact-key/strict-schema decoding, conservative
byte/count limits, UTF-8 validation, and validated single path components. Validate every constructed snapshot through
the existing Rust relationship facade before writing. Never serialize PIDs, workspace numbers, credentials, absolute
paths, checkout paths, or other host-local execution state.

Write the v2 layout:

```text
README.md
schema.json
users/<username>/README.md
users/<username>/machines/<machine>/README.md
users/<username>/machines/<machine>/manifest.json
users/<username>/machines/<machine>/hoods/<hood>/README.md
users/<username>/machines/<machine>/hoods/<hood>/snapshot.json
agents/<global-name>/{README.md,meta.json,state.json,prompt.md,commits.json}
agents/<global-name>/chat.md                 # only when readable
families/<global-family>.md
```

The owner manifest is the only mutable authority file shared by one exact `{username, machine_name}` identity. Other
owners' manifests and globally unique paths are read-only to the publisher. Stage each affected hood snapshot, owner
manifest, run files, family pages, and derived indexes as one payload. Use atomic local writes/staging and path
containment checks so a failed build cannot leave a half-rendered publication.

## Project-scoped hood inventory

Build an indexed inventory adapter for one `ProjectTarget`. Query the persistent artifact index/scan facade for active,
waiting, terminal, and failed `ace-run` records, and use the dismissed-bundle summary index to load matching archived
records without walking every project. Read exact selected artifact/bundle files only after indexed selection to obtain
raw prompts, embedded workflow metadata, prompt-step data, commit markers, optional transcripts, and other allowlisted
portable fields.

Normalize current-owned compatibility spellings through one captured `AgentIdentitySnapshot`; reject imported v1/v2
provenance from export. Give every process record a stable, path-safe source run ID derived from durable artifact
identity, and preserve that ID across later refreshes. Classify state from marker presence/outcome while representing
dismissal time/state as fields rather than as part of the canonical global name.

Starting from a requested committing agent, use the Rust facade to find its top-level local hood and include every
locally owned project record whose semantic name is the hood itself or a dotted descendant. Include active, waiting,
terminal, failed, dismissed, and archived runs even when they have no commit, `done.json`, transcript, or final
response. Synthesize the dotted structural ancestors and family/clan containers required to preserve membership, but do
not manufacture process runs for structural containers.

Resolve parent, workflow-parent, retry, and wait references to stable source run IDs when the target is inside the hood;
retain valid optional same-owner global references when a target is outside the captured run set. Construct family and
clan membership from explicit metadata plus canonical family parsing. Validate the complete relationship/container batch
before publication and surface a scoped diagnostic instead of emitting an incoherent snapshot.

## Targeted and full publication

Expose two entry points over one shared inventory, snapshot, hashing, merge, and rendering pipeline:

1. Targeted publication accepts a project target plus a locally owned committing agent selector and refreshes exactly
   that agent's complete top-level hood. This is the API the later commit-publication phase will call.
2. Full reconciliation discovers every locally owned hood in the project with at least one primary-repository commit
   association and publishes each eligible hood. Historical footer backfill may prove local ownership conservatively,
   but must never infer a foreign v1 owner or re-export imported records.

Before replacing a hood snapshot, merge in previously published run records/files that are temporarily absent from the
current local inventory. Refresh a stable run when new terminal state, commits, prompt data, or chat becomes available,
but never emit an implicit deletion or tombstone. Identical inputs must produce byte-identical files and a no-op
publication; no generated-at timestamps or filesystem iteration order may influence content.

Update the full Git sync transaction to retain optional v1 integration while using only the v2 full-reconciliation
writer. After every pull/rebase, rebuild root/user/machine/hood derived views from all validated owner manifests so two
owners publishing concurrently modify disjoint authority files and convergent indexes. Expand the staged/status path set
to the complete v2 payload, preserve the bounded non-fast-forward recompute retry, and identify sync commits with the
full owner identity.

## Deterministic Markdown browsing

Create a pure deterministic renderer for root, username, machine, hood, family, and agent pages. Use the core
`agent_link_target()` contract so family member anchors are exactly `member-<role>` and solo links point to the agent
README expected by later commit footers.

Root/user/machine/hood pages must provide breadcrumbs, owner/hood summaries, stable counts, and relative links. Family
pages must include accessible text lineage, an optional GitHub Mermaid enhancement, an ordered member/role table, source
states, model/provider/timing/commit summaries, stable member anchors, and prompt/chat links. Agent pages must provide
the corresponding solo detail. Escape Markdown labels, table cells, Mermaid labels, and relative URLs; generate links
only from validated components and never embed local absolute paths. Add checked-in Markdown goldens for solo,
rootless-family, deep-family, active/no-chat, and mixed-state hoods.

## Seed, consent, and outcomes

Replace the agents-sidecar initializer's v1-only scaffold with an empty v2 `schema.json` plus deterministic browsing
scaffold, while leaving an existing legacy manifest/payload untouched and ensuring repeated repo initialization never
clobbers populated owner manifests or derived pages. Update the packaged agents README, repo-init previews/tests, and
`docs/agents_sidecar.md` to explain that one publication includes the complete project-scoped top-level hood, can
include active prompts and later transcript refreshes, uses explicit global owner identity, and should be private or
disabled when that scope is not acceptable.

Extend `ExportCounts`/`SyncOutcome` and JSON/CLI rendering with additive, schema-versioned hood/family/run publication
counts. Preserve the meaning and shape compatibility of legacy integration/export fields rather than silently relabeling
a v1 agent-bundle count as a v2 hood count.

## Verification

Add focused tests for strict v2 parsing, canonical digests, unsafe paths, size/count limits, relationship validation,
and atomic/no-partial writes. Exercise the exact `foo.bar.baz--code` case with `foo`, `foo.bar`, the complete
`foo.bar.baz` family, siblings/descendants such as `foo.boom` and `foo.bar.kazam`, an active committer, a commit-less
plan member, failed/waiting agents, a dismissed member, a rootless family, optional-chat refresh, and a foreign import
that must not be re-exported.

Assert targeted scope, full eligible-hood discovery, structural ancestor/container synthesis, stable source IDs,
prior-run preservation on temporary absence, byte-identical repeat publication, owner-sharded manifests, correct
family/solo paths and anchors, escaped Markdown, and no volatile text. Extend local-bare-remote integration coverage to
prove two usernames/machines can publish through a non-fast-forward race without overwriting one another, derived
indexes converge after recomputation, and existing v1 data remains readable and unchanged.

Update initializer, CLI outcome, Git transaction, documentation, and Markdown golden tests in the same change. Run
`just install` first, then focused `tests/agents_sync`, sidecar-init/repo-init, and CLI tests while iterating, followed
by `just check`. Review the final diff to confirm no SASE memory/generated-instruction files changed. Close only
`sase-8v.4` after all checks pass; leave parent epic `sase-8v` open and create no beads.
