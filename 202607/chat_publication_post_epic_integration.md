---
tier: tale
title: Finish Chats and agents-publication integration
goal: Chats reports committed sidecar provenance and queued, quarantined, or mixed
  publication state correctly.
create_time: 2026-07-24 23:42:28
status: done
---

- **PROMPT:** [202607/prompts/chat_publication_post_epic_integration.md](prompts/chat_publication_post_epic_integration.md)

# Plan: Finish Chats and agents-publication integration

## Goal

Make the Chats provenance catalog describe the final `sase-91` publication system truthfully and transactionally:
`shared` must come only from one committed agents-sidecar tree, publication state must come through the outbox-owned
typed decoder, multiple requests for one agent must aggregate deterministically, and queued/quarantined work must remain
visible even when an older or locally committed chat already classifies as shared.

## Verified current state

Both source epics and all of their phases are closed. The final branch already contains the important integration work:

- the `sase-91` identity facade reads the historical names `4x--epic.f-0`, `fi--code.f0`, `fi--code.f0--plan`, and
  `fi--code.f0--code` without raising;
- `sase-90` looks up both the owner-qualified v2 spelling and the machine-qualified v1 spelling, and a live
  `fi--code.f0--plan` transcript currently classifies as shared;
- `ChatCatalogEntry`, CLI JSON, and the TUI distinguish active publication from a fully quarantined request;
- the recovered sidecar contains committed v1 and v2 agent entries, and the live publication outbox is currently empty.

The two pre-completion review plans remain useful but should not be implemented wholesale:

- `~/.sase/plans/202607/chats_publication_epic_integration.md` correctly found the v1/v2 spelling mismatch and legacy
  identity failures as they existed at that time. Those failures are now handled by the landed identity work and the
  dual-candidate lookup, so a new shared `published_index.py`, cross-epic bead dependencies, and edits to the completed
  epic plans are no longer warranted.
- `~/.sase/plans/202607/chat_publication_integration.md` correctly identified the remaining committed-tree race, outbox
  schema ownership, shared-plus-outstanding state, and missing transition coverage. Its proposed outbox fingerprint is
  unnecessary because the assembled catalog does not cache publication state; it reloads the outbox for every catalog
  snapshot.

Do not rewrite the completed `sase-90` or `sase-91` epic plans, their closed beads, historical chats/artifacts, or
existing sidecar payload. This follow-up plan is the durable correction record.

## Problems to solve

### 1. A dirty sidecar worktree can be cached as published

`src/sase/history/chat_catalog_provenance/sidecars.py` obtains the sidecar `HEAD`, scans the mutable `agents/*/chat.md`
worktree, and caches the result under that SHA. Publication prepares files before committing them while holding the
agents-sync lock, but the off-thread Chats catalog does not take that lock. A concurrent catalog load can therefore call
an uncommitted or partially prepared chat `shared`, then preserve that false result after the worktree is rolled back
because `HEAD` did not change.

### 2. Chats owns a second, lossy outbox decoder

`load_publication_backlog()` opens every `agents-publication-outbox.json` directly, duplicates schema-v1/v2 parsing, and
silently treats malformed or unsupported state as an empty backlog. This bypasses the typed compatibility contract in
`sase.agents_sync.publication_outbox` and can drift again on a later schema change.

The map also overwrites rows by agent name. Multiple primary revisions for one agent are legal because the logical key
is `(global_agent, primary_revision)`. Whichever row happens to be last currently wins, so a mixture of active and
quarantined requests can be reported as either wholly queued or wholly quarantined.

### 3. Outstanding publication disappears for shared chats

The catalog attaches outbox fields to both local and shared entries, but the detail renderer only shows them in the
`local` branch. A sidecar commit whose push has not completed is therefore labeled `shared` with no indication that
publication is still queued or quarantined. The same omission affects an already-shared agent with a later queued
revision.

### 4. CLI JSON drops attempt counts

`ChatCatalogEntry.publication_attempts` is rendered in the TUI, but `chat_info_to_json()` omits it. Machine consumers
cannot reproduce the diagnostic shown by the UI, despite the stable JSON contract documenting the publication fields.

## Implementation

### Read one immutable committed sidecar tree

Refactor the Git-backed path in `src/sase/history/chat_catalog_provenance/sidecars.py`:

- Resolve one commit SHA with the existing non-interactive agents-sync Git runner.
- Enumerate `agents/<name>/chat.md` from that exact commit using one bounded `git ls-tree` invocation against the
  resolved SHA. Parse only exact, safe paths beneath `agents/`; do not inspect `chat.md` contents.
- Build and cache the published-name index under that SHA. Because `ls-tree <sha>` addresses an immutable object, a
  concurrent worktree write or later `HEAD` move cannot mix generations.
- Preserve the non-Git directory-scan fallback for isolated tests and explicitly non-repository fixtures, but never use
  it for a Git checkout whose committed tree could not be read; return an unavailable index plus a diagnostic instead.
- Change the generated-index namespace/token version (or otherwise invalidate the existing sidecar cache) so a payload
  produced by the old worktree scan cannot survive under an unchanged `HEAD`.
- Keep all Git/filesystem work inside the existing catalog worker. Warm loads may perform the cheap `HEAD` lookup but
  must reuse the cached decoded index and must not run another tree enumeration. Render and navigation paths remain free
  of subprocesses, stats, and globs.

Retain the existing v2-first, v1-fallback candidate lookup. Add explicit coverage for a v1-only chat, a v2-only chat,
coexisting spellings, and the landed historical names; do not introduce a second published-index abstraction merely to
rename the current focused sidecar module.

### Put the publication subsystem back in charge of its schema

In `src/sase/agents_sync/publication_outbox.py`, expose the smallest read-only typed snapshot function needed by catalog
consumers. It should return immutable `AgentPublicationOutboxItem` values through the existing `_read_outbox()`
compatibility decoder without acquiring or creating the mutation lock. This is a consistent point-in-time read because
every writer replaces the complete JSON file atomically with `os.replace`; document that contract and retain the
existing lock-taking `list_agent_publications()` behavior for publication workflows.

Make the typed decoder validate the actual JSON types consumed by Chats, especially `attempts` and `quarantined`.
Schema-v1 rows with no quarantine fields remain active. Malformed state raises the existing stable `RuntimeError`
boundary rather than being partially coerced.

Update `load_publication_backlog()` to use that public typed snapshot:

- inspect only project directories that actually contain an outbox, avoiding lock files or writes during catalog reads;
- collect a diagnostic and continue when one project's snapshot is malformed or unreadable;
- group all matching requests for each `(project_key, global_agent)` and `(project_key, local_agent)`;
- introduce a typed publication disposition: `queued`, `quarantined`, or `mixed`;
- choose `queued` when all matching requests are active, `quarantined` when all are quarantined, and `mixed` when both
  kinds exist;
- expose the maximum attempt count and the last error from the deterministically most recently updated matching request,
  with stable tie-breakers.

Carry backlog diagnostics into `ChatCatalogSnapshot.diagnostics` without changing chat provenance merely because
publication metadata could not be read. Suppress local publication metadata for `remote` entries so remote provenance
remains authoritative even if fixture or compatibility names collide.

For compatibility, keep `publication_pending` and `publication_quarantined`:

- `publication_pending` is true for `queued` and `mixed` because retryable work exists;
- `publication_quarantined` is true only for `quarantined`, meaning no matching request will retry automatically;
- add a derived `publication_disposition` surface so `mixed` is not hidden from new consumers.

### Render provenance and publication as separate dimensions

Update `src/sase/ace/tui/widgets/artifacts/chats_detail.py` so publication diagnostics are rendered after the base
provenance facts for both `local` and `shared` chats:

- local + queued: keep `Queued to publish`;
- local + quarantined: keep `Publication quarantined` and the manual retry command;
- shared + queued: say that committed sidecar content exists while publication completion remains queued;
- shared + quarantined: show both the committed chat and the stopped retry state;
- mixed: state that retryable and quarantined requests coexist, include the aggregate attempts/error, and retain the
  manual retry hint without implying that all work has stopped.

Never collapse a shared entry back to local because an outbox row exists, and never describe a fully quarantined item as
queued. Remote entries show neither local publication copy nor local publication JSON state.

Append `publication_attempts` and `publication_disposition` to the existing stable key order in
`src/sase/history/chat_catalog.py`; preserve every existing key and its order. Update
`src/sase/xprompts/skills/sase_chats.md` to document the new fields, the compatibility booleans, mixed state, and the
fact that provenance and publication are independent.

Because that skill source is generated, run `sase skill init --force`, use the `sase_repo` skill before touching the
chezmoi linked repo, and run `chezmoi apply`; do not edit a deployed provider `SKILL.md` directly.

Add a concise section to `docs/agents_sidecar.md` explaining that Chats calls content `shared` only when `chat.md`
exists in the local sidecar's committed tree. An outstanding outbox row separately means the related publication has not
completed, including the committed-but-not-pushed case.

## Tests and transition proof

Extend the focused tests rather than building a second catalog implementation:

- In `tests/history/test_chat_catalog_provenance.py`, initialize a real temporary Git sidecar. Prove that an uncommitted
  `agents/<name>/chat.md` stays local even with `force=True`, committing it changes the chat to shared, and a warm load
  at unchanged `HEAD` does not enumerate the tree again. Include v1-only, v2-only, coexisting, and historical-name
  cases.
- Exercise typed schema-v1 and schema-v2 snapshots, malformed field diagnostics, two active revisions, two quarantined
  revisions, and a mixed active/quarantined set. Assert deterministic attempts/error selection and remote precedence.
- Add a cross-epic transition test that keeps the catalog cache between steps: local + queued; local + quarantined;
  dirty prepared sidecar files with unchanged `HEAD`; committed shared + queued; shared + quarantined; mixed; and
  acknowledged shared. Check the catalog model at every step.
- In `tests/ace/tui/test_artifacts_chats_detail.py`, cover local and shared queued/quarantined/mixed copy and assert the
  mutually exclusive wording guarantees.
- In `tests/main/test_chat_handler.py`, assert the exact appended JSON key order and values for attempts/disposition,
  including `mixed` and `null`.
- Add or extend publication-outbox tests for the lock-free typed snapshot, schema-v1 compatibility, strict field types,
  and malformed input. Confirm the snapshot read creates no lock file and performs no write.

## Validation

1. Run `just install` before repository checks.
2. Run the focused publication-outbox, provenance catalog, CLI JSON, and Chats detail tests.
3. Run `sase skill init --force` and `chezmoi apply`, then verify the generated `sase_chats` skill matches its source.
4. Run `just test-visual` because Chats detail rendering changes; update no golden unless a fixture intentionally
   exercises the new publication copy, and rerun without update mode if a golden changes.
5. Run `just check`.
6. Re-read the primary and chezmoi diffs. Confirm the completed epic plans/beads, memory files, generated instruction
   shims, historical artifacts/chats, and agents-sidecar payload are untouched.

## Acceptance criteria

- A dirty or partially prepared Git sidecar worktree can never make a chat shared or poison the cache under an old
  `HEAD`; the published index corresponds to one immutable commit.
- V1-only and v2-only publications, including the known historical names, classify without raising.
- Chats consumes the publication subsystem's typed schema decoder and degrades malformed outbox state to a diagnostic.
- Multiple requests for one agent aggregate deterministically as queued, quarantined, or mixed.
- Shared chats continue to show committed sidecar provenance while also showing unfinished or quarantined publication.
- Remote provenance wins and carries no local publication state.
- CLI JSON includes attempt counts and an explicit disposition without reordering or removing existing keys.
- Warm catalog loads and TUI navigation preserve the established performance constraints.
- Focused tests, visual tests, generated-skill deployment, and `just check` pass.
