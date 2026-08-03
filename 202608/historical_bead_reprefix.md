---
tier: epic
title: Safely re-prefix historical bead identities
goal: Historical beads whose IDs leaked a ProjectSpec key can be migrated to the project's
  display-name prefix through a dry-run-first, restartable workflow that preserves
  lineage, rewrites owned references, and keeps old IDs and hosted URLs working as
  compatibility aliases without changing immutable commit history.
phases:
- id: core-reprefix
  title: Rust bead identity and alias primitive
  depends_on: []
  size: large
  description: 'core-reprefix: add the Rust-backed collision-safe full-store bead
    ID mapping, canonical event/projection rewrite, persistent old-ID aliases, exact
    token rewriting, PyO3 bindings, and mixed-prefix lineage tests.'
- id: reference-rewriters
  title: Plan, ChangeSpec, and compatibility-page rewriters
  depends_on:
  - core-reprefix
  size: medium
  description: 'reference-rewriters: add codec-driven plan and ChangeSpec rewrite
    planners plus canonical and old-ID compatibility bead pages, with exact-match
    audit and malformed-input coverage.'
- id: agent-history
  title: Historical agent identity and chat migration
  depends_on:
  - core-reprefix
  size: large
  description: 'agent-history: migrate derived bead-named agents, structured run artifacts,
    chats, registries, and agents-sidecar bundles while preserving old hosted agent
    links through explicit compatibility aliases.'
- id: migration-cli
  title: Migration CLI and multi-store transaction
  depends_on:
  - core-reprefix
  - reference-rewriters
  - agent-history
  size: large
  description: 'migration-cli: expose the default-dry-run migrate-prefix command and
    compose deterministic preflight, locking, receipts, recovery refs, atomic apply,
    scoped commits, rollback, and resumable publication across every target.'
- id: integration-docs
  title: End-to-end verification and documentation
  depends_on:
  - migration-cli
  size: medium
  description: 'integration-docs: prove mixed-prefix migration, aliases, immutable
    commit history, idempotency, and injected recovery paths in a multi-repository
    fixture, then document and run the complete Rust/Python validation suite.'
proposed_by: bbugyi200.athena.sase-eh
create_time: 2026-08-03 04:47:52
status: wip
bead_id: sase-ei
---

- **PROMPT:** [prompts/202608/historical_bead_reprefix.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/historical_bead_reprefix.md)
- **BEAD:** [sase-ei](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ei/README.md)

# Plan: Safely re-prefix historical bead identities

## Goal

Add an explicit, restartable `sase bead migrate-prefix` workflow that converts every historical bead whose prefix is the
leaked ProjectSpec key to the project's display-name prefix, rewrites the durable references SASE owns, preserves old
IDs and URLs as aliases for immutable history, and proves that a mixed-prefix store can migrate without losing event
lineage or leaving a partially rewritten project.

## Why this is an epic

This is not one file-format migration. The canonical bead graph belongs to `sase-core`; plans and ChangeSpecs have their
own structured writers and locks; historical agent identity spans local run artifacts, chat history, registries, and the
agents sidecar; and the final command must coordinate several Git repositories plus non-Git local state. The Rust
identity contract must land first, but the plan/ChangeSpec and agent-history adapters can then be implemented and tested
independently before the transaction layer composes them.

## Observed starting point

The forward mint guard repairs only `config.json.issue_prefix` before a new top-level bead is created. It deliberately
leaves old IDs in place. A live pre-fix `bob-cli` store demonstrates the migration shape:

- `config.json` has `issue_prefix: gh_bobs-org__bob-cli` and `next_counter: 6`;
- five canonical event streams are named `gh_bobs-org__bob-cli-1.jsonl` through `gh_bobs-org__bob-cli-5.jsonl`, with
  phase IDs, dependency edges, references, and event IDs embedding the same prefix;
- the plans sidecar has `bead_id` frontmatter and generated `BEAD` links using those IDs;
- historical agent bundles use the IDs as local/global names and in `bead_id`, `epic_bead_id`, `phase_bead_id`, clan,
  family, prompt, and transcript fields;
- primary-repository commit footers contain old bead and agent links.

The last category is immutable published history and must not be rewritten. The migration instead keeps explicit old ID
aliases and compatibility pages so historical commit messages and hosted URLs remain useful while every current
canonical identity uses the corrected prefix.

## Product contract

### Command and defaults

Add the alphabetically registered command:

```text
sase bead migrate-prefix [-f OLD] [-t NEW] [-j] [-w] [-y]
```

- With no prefix options, infer `OLD` as the current project's ProjectSpec key and `NEW` as its configured
  `PROJECT_NAME`. Reject inference when the names are equal, unavailable, or the destination is not a safe bead prefix.
- `-f/--from-prefix` and `-t/--to-prefix` must be supplied together and exist for recovery or deliberate expert use;
  validate them with the same prefix policy used for minting.
- The default is a read-only, colored dry run. `-j/--json` emits the same deterministic audit as structured data.
- `-w/--write` applies the exact audited plan after revalidation. Interactive writes require a final confirmation;
  `-y/--yes` answers it non-interactively. Every public long option has a short alias and help contains concrete
  dry-run, apply, and recovery examples.

The audit groups changes by `beads`, `plans`, `changespecs`, `agents`, `chats`, and `immutable_history`. For every group
it reports files, canonical identities, exact-reference counts, skips, blockers, and preimage digests. It prints the
complete old-to-new bead mapping and derived old-to-new agent-name mapping. A dry run changes no bytes, creates no
commits, and does not acquire mutating locks.

### Mapping and collision rules

Build one closed, injective map before any write. A source ID must match `OLD-<base36>(.<decimal>)*`; preserve the full
counter/child suffix verbatim, so `gh_bobs-org__bob-cli-2.4` becomes `bob-cli-2.4`. Include descendants, dependency
targets, bead artifact references, and every canonical issue in the map. Reject the whole migration when:

- any destination ID already belongs to an issue outside the mapping;
- two sources map to one destination, a parent would be missing, or an event stream is invalid;
- an affected bead is `claimed` or `in_progress`, or an affected agent is live;
- a required repository is dirty, detached, mid-operation, missing its configured upstream, or cannot be locked;
- an owned structured reference cannot be classified safely.

Mixed stores are supported: already-correct IDs such as `bob-cli-6` stay byte-for-byte identical, `next_counter` never
moves backward, and a collision such as both old and new forms of counter 3 aborts before any mutation.

### Canonical rename and aliases

The Rust primitive owns bead-domain semantics. It rewrites stream filenames and every ID-bearing field in event records
and payloads, including creation snapshots, parents, dependencies, removal lists, bead refs, and exact bead-ID tokens in
stored prose. Recompute content-derived event IDs deterministically and return the event-ID mapping in the audit
receipt; preserve stream/event order, operation count, timestamps, actors except when an actor is one of the derived
renamed agents, notes, status, resolution, and all nonidentity data. Regenerate `issues.jsonl` from the renamed events
and validate that its reduction is isomorphic to the preimage under the ID map.

Extend bead config with a backward-compatible, default-empty `id_aliases` mapping. The migration records every old ID as
an alias of its new canonical ID. Exact old IDs resolve through this map before shorthand resolution on every read and
mutation path; alias chains and cycles are rejected, and new aliases cannot shadow canonical IDs. Machine output may
report the canonical ID, but ordinary commands given an old ID continue to work. Bead-page refresh emits small
compatibility pages at the old hosted paths that identify and link to the new canonical page. These aliases are the
contract for unrewritable commit history; no branch or commit hash is changed.

### Owned reference rewrites

Use one longest-match, boundary-aware bead-ID token rewriter exposed by the Rust core and consume it from the host.
Never use raw prefix replacement. Structured owners are rewritten through their codecs:

- canonical and legacy plan frontmatter: `bead_id`, `bead`, and `parent_bead`; then rebuild the generated `BEAD`
  header/link through the existing plan-header helpers;
- bead `bead:` refs and exact tokens in canonical issue fields/events through the core primitive;
- active and archived ChangeSpec `BUG:` and `REFS:` values only when their parsed value contains an exact mapped bead
  ID, under `changespec_lock` with `write_changespec_atomic`; do not touch commit/timestamp drawers or unrelated
  ProjectSpec-key fields;
- affected local agent artifacts and dismissed bundles: structured bead fields plus derived clan/family/workflow/local
  and global names, relationship targets, prompt directives, transcript headings/body references, notifications, and
  prompt-history references;
- agents-sidecar bundles, manifests, hood/family snapshots, indexes, digests, and hosted compatibility aliases,
  regenerated from migrated local sources through the normal agent-sync model rather than ad hoc sidecar text edits;
- local chat-history files whose cataloged run is affected, including a safe filename rename when the filename embeds
  the old derived agent name and the destination is free.

Free-form files that merely quote an old ID but are not owned references are reported as audit-only findings. Primary
Git commit subjects, bodies, trailers, trees, refs, and SHAs are always in this category. After apply, an audit may find
old IDs only in the explicit alias/receipt records, compatibility pages, and immutable-history findings.

### Transaction, recovery, and idempotency

Applying a migration is a two-stage local transaction followed by restartable publication:

1. Recompute the dry-run plan while holding every required lock in deterministic path order. Verify all preimage digests
   and blockers again.
2. Create a durable migration receipt under SASE state with a stable migration ID, mapping, target paths, pre-heads,
   file preimages for non-Git state, planned commit subjects, and per-target state. Create
   `refs/sase/bead-reprefix/<id>` recovery refs for every Git target before writing.
3. Write each file atomically, run format/codec validation, reduce the renamed bead store, rebuild indexes/pages, and
   commit one scoped migration commit in each changed sidecar. Do not push during this stage.
4. If any write, validation, or local commit fails, restore non-Git preimages and reset only the migration's clean,
   unpushed sidecar commits to their recorded recovery refs. Preserve the receipt and print the failure and exact retry
   command. Never discard unrelated changes.
5. Push the locally durable commits synchronously in deterministic order. A partial push is not history-rewritten or
   force-rolled back: mark successful targets in the receipt, retain every local commit/recovery ref, and tell the user
   to rerun the same command. A rerun verifies commit subjects/tree digests and resumes only incomplete targets.
6. After all targets publish, mark the receipt complete. Further identical runs are no-ops. If all canonical data was
   already migrated but publication was interrupted, the command resumes instead of reporting “nothing to do.”

This recovery model makes a pre-publication failure exactly reversible and a post-publication failure safely
forward-recoverable without rewriting shared history.

## Phases

### 1. Rust bead identity and alias primitive

Implement this phase in the linked `sase-core` repository, opened through `sase repo open sase-core` before any read or
edit.

- Add backward-compatible `id_aliases` config wire support, validation, exact-alias resolution ahead of shorthand, and
  coverage across read, history, dependency, close/open/update/remove, and work/claim operations.
- Add serializable preview/apply request and outcome wires for a full-store prefix migration. Centralize safe-prefix,
  source-ID parsing, injective mapping, collision, parent, and alias validation.
- Add the boundary-aware exact-ID text/reference rewriter and apply it to every bead event/payload identity surface.
  Stage a complete renamed store, deterministically recompute event IDs, regenerate the projection, and refuse to
  install it unless pre/post reductions are isomorphic under the returned maps.
- Preserve `next_counter` and unrelated already-correct streams. Make rejected requests and previews byte-identical.
  Expose preview/apply and token rewriting through `sase_core`, `sase_core_py`, binding documentation, and Python
  facade/model adapters.
- Test legacy config decoding, alias chains/cycles/shadowing, old-ID compatibility, mixed-prefix stores, cross-stream
  dependencies, refs and prose, event history, collisions, invalid stores, deterministic output, and injected write
  failure.

### 2. Plan, ChangeSpec, and compatibility-page rewriters

Depends on phase 1.

- Build pure planners that consume the core ID map and return preimage digest, rewritten bytes, structured action, skip,
  or blocker for each plan and ChangeSpec target. Plans must use the frontmatter and header codecs; ChangeSpecs must use
  the parser and atomic writer contract for both active and archive files.
- Distinguish ProjectSpec keys, ChangeSpec names, and actual mapped bead IDs so values such as the `bob-cli` ChangeSpec
  name are never altered just because they contain the old prefix text.
- Extend bead-page generation with canonical pages plus deterministic old-ID compatibility pages backed by `id_aliases`;
  update known-ID/link validation so both historical and new hosted links remain valid while navigation presents the
  canonical ID.
- Add focused tests for `bead_id`/legacy `bead`/`parent_bead`, generated headers, Markdown links, `BUG:` URL spellings,
  `REFS:` entries, active/archive files, malformed documents, ambiguous ownership, alias page cleanup, and exact no-op
  bytes.

### 3. Historical agent identity and chat migration

Depends on phase 1.

- Generalize the existing historical auto-name migration machinery into a reusable explicit mapping engine. Discover
  affected runs from structured bead metadata, derive complete local/global/clan/family/workflow agent mappings, and
  reject collisions or live affected agents before mutation.
- Plan and atomically rewrite the affected timestamp-keyed run artifacts, dismissed bundles, relationship records,
  prompt directives/history, notifications, registries/indexes, and cataloged chat files. Rewrite exact mapped bead or
  agent identities only; retain prose and commit subjects that are historical evidence.
- Extend the agents-sync schema/model to publish renamed bundles and deterministic old-name compatibility entries so old
  `SASE_AGENT` links in immutable commit messages still reach the canonical agent. Recompute all bundle, manifest, hood,
  and family digests from source instead of patching generated JSON/Markdown blindly.
- Add dry-run/apply report models and tests covering phases, land-agent families, clans, wait relationships, dismissed
  agents, chat filename collisions, stale indexes, old hosted links, malformed artifacts, and idempotent regeneration.

### 4. Migration CLI and multi-store transaction

Depends on phases 1, 2, and 3.

- Register and dispatch `sase bead migrate-prefix` with the command/help/option contract above. Resolve the current
  project's primary, beads, plans, and agents locations through repository/project inventory rather than guessed paths;
  include both active and archived ChangeSpec files and project-local chat/artifact state.
- Compose the core preview with both reference planners into one stable human/JSON audit. Add immutable Git-history
  scanning for exact old bead/agent footers and links, but never schedule those objects for mutation.
- Implement deterministic lock acquisition, active-work and dirty-repository preflight, digest revalidation, durable
  receipts/backups/recovery refs, atomic local apply, structured validation, scoped local commits, rollback before any
  push, and synchronous resumable publication after all commits exist.
- Make restart states explicit in both rendering and JSON (`planned`, `applying`, `committed`, `partially_published`,
  `complete`, `rolled_back`, `failed`). Verify that SIGTERM/crash-style interruption after every stage can be resumed or
  restored without guessing.
- Integrate the completed migration with `sase bead doctor`: report an available historical migration when key-prefixed
  canonical IDs remain, report incomplete receipts with the exact resume command, and report alias/compatibility drift.

### 5. End-to-end verification and documentation

Depends on phase 4.

- Build a realistic multi-repository fixture with old IDs 1-5, an already-correct ID 6, phase lineage, cross-root
  dependencies, refs, plans, active/archive ChangeSpecs, completed phase/land agent families, chats, agent-sidecar
  bundles, and primary commits containing old footers.
- Prove the dry run is byte-identical; apply the migration; run bead doctor, plan-link validation, agent-sync
  validation, page validation, and all relevant Rust/Python tests; then prove old CLI IDs and old hosted paths resolve
  while every current surface renders the canonical prefix.
- Inject collision, dirty-tree, malformed-record, failed-write, failed-local-commit, first-push, and later-push
  failures. Verify pre-push rollback and post-push forward recovery, then rerun a completed migration as a no-op.
- Assert the primary repository's refs, commit graph, commit IDs, and commit object bytes are unchanged. Restrict
  residual old-prefix audit findings to alias/receipt/compatibility records and immutable commit history.
- Document the command, safety gates, audit JSON, alias behavior, commit-history limitation, and exact recovery
  procedure in the bead and CLI docs. Run `just install` before final validation and finish with `just check`, including
  the linked Rust workspace checks.

## Phase graph

```text
core-reprefix
├── reference-rewriters ─┐
└── agent-history ───────┼── migration-cli ─── integration-docs
                         ┘
```
