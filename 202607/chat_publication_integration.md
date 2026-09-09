---
tier: epic
title: Integrate Chats provenance with resilient agents-sidecar publication
goal: "The Chats catalog and UI report committed sidecar provenance and durable queued
  or quarantined publication state correctly across legacy names, retries, recovery, and
  cache invalidation.

  "
phases:
  - id: publication-snapshot
    title: Stable publication observability snapshot
    depends_on: []
    size: small
    description:
      "'Stable publication observability snapshot' section: expose a typed,
      fingerprinted, hood-level read model for queued and quarantined publication
      requests."
  - id: catalog-integration
    title: Transactionally correct Chats classification
    depends_on:
      - publication-snapshot
    size: medium
    description:
      "'Transactionally correct Chats classification' section: classify from one
      committed sidecar tree, consume hood publication state, handle v1 and v2 names,
      and invalidate every relevant cache."
  - id: transition-proof
    title: Honest UX and cross-epic transition proof
    depends_on:
      - catalog-integration
    size: medium
    description:
      "'Honest UX and cross-epic transition proof' section: carry queued versus
      quarantined state through CLI and TUI surfaces and prove real publication
      transitions end to end."
create_time: 2026-09-09 19:53:05
status: wip
---

- **PROMPT:**
  [prompts/202607/chat_publication_integration.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/chat_publication_integration.md)

# Plan: Integrate Chats provenance with resilient agents-sidecar publication

## Goal

Make the `sase-90` Chats catalog and UI consume the durable publication behavior
introduced by `sase-91` without guessing at JSON layout, misreporting quarantined work
as queued, or treating uncommitted sidecar worktree files as shared. The result must
remain correct as a hood moves from local and queued, to quarantined, or to successfully
published, including when the transcript belongs to a legacy agent name.

## Why a follow-up integration epic is required

The two source epics are individually coherent but do not currently define their shared
boundary:

- `sase-90` exposes only `publication_pending` and `publication_last_error`, while its
  detail design also needs an attempt count. `sase-91` adds a visible, manually
  recoverable quarantine state that is not pending and must not be described as "queued
  to publish."
- Agents publication is hood-scoped. A queued commit for one family member republishes
  the complete hood, so matching an outbox row to only the exact transcript agent is
  incorrect.
- `sase-90` proposes scanning `<sidecar>/agents/*` in the mutable worktree and caching
  the result under the repository `HEAD`. During `sase-91` publication, files are
  prepared before the commit and push. A concurrent catalog scan could therefore call an
  uncommitted or partially prepared chat `shared`, then cache that answer under the old
  `HEAD`.
- `sase-91` preserves v1 `machine.agent` data while creating v2 `username.machine.agent`
  data. Exact matching against only the current v2 global name can miss a valid v1
  publication.
- Neither epic owns an end-to-end test of catalog behavior when an outbox item is
  retried, quarantined without a sidecar commit, or successfully drained with a new
  sidecar commit.

## Execution coordination

This is a post-contract integration epic. Do not begin its implementation until
`sase-90.3` (the headless Chats catalog), `sase-91.3` (outbox isolation and quarantine),
and `sase-91.4` (hood deduplication) have landed. When the phases below are materialized
as beads, record those three source phases as external dependencies of the first phase.
The CLI and pane phases in `sase-90` must consume the integrated catalog model rather
than freezing the pre-integration boolean API.

## Design contracts

### Provenance and publication are separate dimensions

Keep the four provenance values from `sase-90`: `local`, `shared`, `remote`, and
`unknown`. Add a separate publication disposition for locally owned chats: `queued`,
`quarantined`, or absent. Remote classification remains authoritative and wins over
local publication metadata.

For all outbox requests matching a `(project_key, local_hood)`:

- disposition is `queued` when any matching request remains retryable;
- disposition is `quarantined` when matching requests exist but all are quarantined;
- attempts is the maximum matching attempt count;
- the displayed error comes from the most recently updated matching request;
- diagnostics retain enough detail to report mixed queued/quarantined rows without
  hiding either.

Do not add quarantine as a fifth provenance value. A chat can still be local while its
publication request is quarantined, and a previously shared hood can have a newer queued
update.

### Read durable state through typed APIs

The Chats catalog must not open `agents-publication-outbox.json` directly. It must
consume a public, read-only snapshot API owned by `sase.agents_sync.publication_outbox`,
so schema migration, locking, quarantine representation, and backward compatibility
remain the publication subsystem's responsibility.

### Shared means committed sidecar content

Build the published-chat index from a specific committed sidecar Git tree, not from
mutable worktree paths. Return the commit SHA with the index and cache only under that
SHA. Prepared-but-uncommitted files must never become `shared`. Outstanding outbox state
is shown separately, including after a local sidecar commit whose push is still queued.

Recognize both supported compatibility spellings by using existing identity helpers and
metadata:

- v2 `username.machine.agent`;
- v1 transport `machine.agent`.

Do not parse or reconstruct owner, hood, family, or legacy-name semantics in the Chats
package. Reuse the total read-side identity helpers delivered by `sase-91.1`.

## Phase 1: Stable publication observability snapshot

**ID:** `publication-snapshot` **Depends on:** none **Size:** small

Define the typed read boundary in `src/sase/agents_sync/publication_outbox.py` after the
final `sase-91.3` schema is present.

- Preserve `list_agent_publications()` for existing callers, and add the smallest
  immutable snapshot/view needed by read-only consumers. It must expose each request's
  project key, local hood, local/global agent, primary revision, queued or quarantined
  disposition, attempts, last error, and update time.
- Return a deterministic generation/fingerprint derived from the durable snapshot so a
  consumer cache invalidates when an item changes disposition, attempts, error,
  membership, or update time even when the agents sidecar `HEAD` does not change.
- Keep reads under the existing outbox lock and retain compatibility with every schema
  version that `sase-91` supports. Malformed state must produce a diagnostic for the
  catalog, not crash the entire Chats listing.
- Add a hood-level aggregation helper implementing the precedence and most-recent-error
  rules above. Multiple primary revisions and multiple agents in the same hood must
  collapse deterministically without losing diagnostic detail.
- Test empty, legacy-schema, queued-only, quarantined-only, mixed, and concurrently
  rewritten snapshots. Assert that changes which leave the sidecar `HEAD` untouched
  still change the publication fingerprint.

Run `just install` before `just check`.

## Phase 2: Transactionally correct Chats classification

**ID:** `catalog-integration` **Depends on:** `publication-snapshot` **Size:** medium

Update the headless catalog delivered by `sase-90.3`.

- Replace direct outbox JSON reads and exact-agent pending matching with the typed
  hood-level publication snapshot from phase 1.
- Replace the boolean publication fields with a backward-compatible model that includes
  disposition, attempts, last error, and relevant diagnostic text. If compatibility
  properties are retained, `publication_pending` is true only for retryable queued work,
  never for quarantine.
- Build the published-chat index from one immutable committed sidecar tree. Prefer a
  focused helper in `sase.agents_sync` that returns
  `(HEAD, published-chat records, diagnostics)` and can enumerate metadata and `chat.md`
  presence without observing dirty worktree state. Detect a `HEAD` change during any
  multi-command fallback and retry once rather than caching a mixed snapshot.
- Index canonical v2 names and validated v1 transport aliases through the existing
  identity facades. Verify the exact historical names from `sase-91` (`4x--epic.f-0`,
  `fi--code.f0`, `fi--code.f0--plan`, and `fi--code.f0--code`) classify without raising.
- Key catalog-derived provenance by transcript stat data, artifact-index generation,
  committed sidecar SHA, and publication fingerprint. A force refresh bypasses all four.
  Render paths continue to perform no filesystem or Git scans.
- Preserve `remote` precedence. A remote imported artifact must remain remote even if a
  compatibility `agents/<global-name>/chat.md` entry or a local outbox row happens to
  match.
- Keep the no-sidecar behavior from `sase-90`: locally created chats are local, not
  unknown. A configured but unreadable sidecar remains unknown and carries a diagnostic.
- Add focused headless tests for v1-only, v2-only, coexisting v1/v2, malformed
  historical names, a dirty worktree with unchanged `HEAD`, an outbox-only fingerprint
  change, and a `HEAD` race.

Run `just install` before `just check`.

## Phase 3: Honest UX and cross-epic transition proof

**ID:** `transition-proof` **Depends on:** `catalog-integration` **Size:** medium

Carry the integrated model through the CLI, Chats pane, and recovery acceptance
coverage.

- Extend `sase chat list` JSON with the publication disposition, attempts, and last
  error while preserving existing key order and compatibility fields. Pretty output may
  stay provenance-focused, but machine-readable output must distinguish queued from
  quarantined.
- Render local queued chats as "Queued to publish" and local quarantined chats as
  "Publication quarantined" with attempts, the attributable error, and the supported
  manual-clear hint supplied by `sase-91`. Never imply that quarantined work will retry
  automatically.
- For a shared chat with a newer queued or quarantined hood request, show both facts:
  the transcript/hood has committed sidecar content and a later publication is
  outstanding. Do not collapse it back to local.
- Add an end-to-end fixture that starts with a legacy-named local chat and coexisting v1
  sidecar data, then exercises these transitions: local + queued; local + quarantined
  with unchanged sidecar `HEAD`; prepared-but-uncommitted sidecar files; successful v2
  commit and outbox acknowledgement; and a later queued update to an already shared
  hood. Assert catalog, CLI JSON, and detail-panel copy after every transition without
  deleting the persistent catalog cache between assertions.
- Include a remote imported chat in the same fixture and prove its provenance never
  changes during local publication transitions.
- Fold this scenario into the final verification of both source epics: it is a required
  acceptance check before `sase-90.8` and `sase-91.6` close. Update
  `docs/agents_sidecar.md` or the Chats-facing documentation once, in the phase that
  lands last, to explain the provenance/publication distinction without duplicating
  recovery instructions.

Run `just install` before `just check`. Run `just test-visual` only if rendered Chats
output changes after its visual goldens exist.

## Acceptance criteria

- Chats never describe a quarantined publication as queued or pending.
- Publication metadata is matched by project and hood, not only by exact agent.
- A dirty or partially prepared sidecar worktree cannot make a chat `shared`; the
  published index corresponds to one committed Git tree and its SHA.
- Both v1 and v2 sidecar spellings are recognized without weakening strict validation
  for newly created names.
- Catalog caches invalidate when either committed sidecar content or durable outbox
  state changes.
- The queued, quarantined, successfully shared, shared-with-newer-pending, and remote
  cases are covered end to end across the headless catalog, CLI JSON, and Chats detail
  rendering.
- Every changed repository passes its required checks. No memory files, generated
  instruction shims, historical artifact directories, chat files, or primary commits are
  rewritten.
