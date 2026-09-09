---
tier: epic
title: Eliminate artifact-link merge conflicts with immutable link events
goal: "Automatic artifact-link writes never produce a merge conflict an agent must
  hand-resolve: link mutations become immutable, content-addressed events reduced
  deterministically in Rust core and published through the host-owned machine lane with
  durable retry, while a conservative semantic resolver auto-repairs legacy links/*.json
  conflicts during rollout.

  "
phases:
  - id: links-resolver
    title: Semantic resolver for link-index conflicts
    depends_on: []
    size: large
    description:
      "links-resolver: add a Rust three-way links/*.json index merge and wire a second
      semantic resolver into both rebase-repair paths so distinct-reader conflicts stop
      pausing agents."
  - id: event-contract
    title: Immutable link-event contract and reducer in Rust core
    depends_on: []
    size: large
    description:
      "event-contract: define content-addressed link-event files, canonical
      serialization, operation identity, and a deterministic event-set reducer in
      sase-core with Python bindings."
  - id: outbox-identity
    title: Durable operation identity in the link outbox
    depends_on:
      - event-contract
    size: medium
    description:
      "outbox-identity: give every queued link mutation a full-strength operation id,
      stop count-collapsing distinct operations, and retire entries only after verified
      publication."
  - id: event-publisher
    title: Automatic link writes publish as events through the machine lane
    depends_on:
      - outbox-identity
    size: large
    description:
      "event-publisher: behind a new beta flag, automatic writers enqueue events instead
      of dirtying agent sidecar clones, and a serialized per-project publisher batches
      them into the hidden host-owned clones with bounded integrate-push retry."
  - id: event-readers
    title: Readers, projections, and maintenance consume reduced events
    depends_on:
      - event-contract
      - outbox-identity
    size: large
    description:
      "event-readers: aggregate rebuilds, link list/doctor, managed Markdown blocks, and
      rename handling read reduced events overlaid with pending outbox operations, with
      alias events replacing event-path rewrites."
  - id: cutover-import
    title: Fence, import legacy indexes, and cut over
    depends_on:
      - links-resolver
      - event-publisher
      - event-readers
    size: large
    description:
      "cutover-import: one-shot deterministic import of frozen legacy indexes into
      baseline events, flag enablement and removal, read-only legacy indexes, and doctor
      guardrails against post-import legacy writes."
  - id: acceptance
    title: Multi-clone acceptance suite and conflict-free guarantee
    depends_on:
      - cutover-import
    size: medium
    description:
      "acceptance: end-to-end suite with independent clones and two simulated machines
      proving zero link-metadata merge pauses, no lost or double-counted operations, and
      clean crash recovery at every publication boundary."
proposed_by: bbugyi200.athena.08o
bead_id: sase-yi
create_time: 2026-09-09 19:52:12
status: wip
---

- **PROMPT:**
  [prompts/202609/artifact_link_event_store.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/artifact_link_event_store.md)
- **BEAD:**
  [sase-yi](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yi/README.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                    | Why                                                                    |
| ------------ | --------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| derives-from | [research:202609/artifact_link_event_store/artifact_link_event_store.md][1] | Implements the consolidated research recommendation for link events    |
| related      | [bead:sase-ye][2]                                                           | The event-publisher phase adds the durable retry/aging this bug tracks |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202609/artifact_link_event_store/artifact_link_event_store.md
[2]: https://github.com/sase-org/sase--beads/blob/main/pages/sase-ye/README.md

<!-- sase:links:end -->

# Plan: Eliminate artifact-link merge conflicts with immutable link events

## Problem and evidence

Every document artifact has exactly one tracked sidecar index,
`links/<document-relative-path>.json`, holding its entire link neighborhood. Reading a
popular artifact mutates that same file in every agent's independent sidecar clone, so
concurrent agents produce add/add and modify/modify git conflicts that a human or agent
must hand-resolve:

- Agent `research.1n.cdx` (2026-09-08) hit a both-added conflict on
  `links/202609/remote_machine_management_enablement.md.json` in the research sidecar,
  repaired it by hand, and hit a second conflict in the same file on
  `sase stitch create --resume` because a third reader had pushed meanwhile.
- Roughly 126 of 451 research-sidecar commits (~28%) are link-only commits, so the
  contention window is constantly refreshed.
- The consolidated research report
  `research:202609/artifact_link_event_store/artifact_link_event_store.md` analyzes the
  failure and recommends the architecture this epic implements. Read it (audited, via
  `sase artifact read`) before implementing any phase.

The conflict chain today:

1. `sase artifact read` inside an agent run records a `read` row:
   `src/sase/artifact_cli/read.py` `_record_read_link` calls
   `ArtifactLinkStore.upsert_row` immediately and appends an outbox entry.
2. `upsert_row` (`src/sase/sdd/_artifact_link_store_rows.py`) writes each document
   endpoint's index via `_upsert_sidecar`
   (`src/sase/sdd/_artifact_link_store_sidecar.py`), which read-modify-replaces the
   whole JSON file under a per-clone flock. The lock cannot help across clones.
3. The agent finalizer (`src/sase/finalizers/reconciliation.py`,
   `_auto_commit_artifact_link_indexes_if_possible`) commits the dirty `links/*.json`
   from the agent's own sidecar clone; publication then rebases onto origin.
4. Both integration paths repair only bead conflicts.
   `src/sase/sdd/_repository_integration.py` `_repair_or_abort_rebase` aborts on any
   non-bead unmerged path; `src/sase/vcs_provider/plugins/_git_commit_dispatch.py`
   `_continue_rebase_resolving_beads` requires every unmerged path to start with the
   bead-store prefix and otherwise leaves the rebase paused for the agent.

Two failure modes result: the SDD path (sidecar publication) aborts and retries forever
against a moving remote, and the primary-commit path pauses and burns an agent turn on
hand-repair.

## Current architecture facts phase workers need

- **Row model (schema v2).** Rust owns it: `crates/sase_core/src/artifact_link/wire.rs`
  in the linked `sase-core` repo (`ArtifactLinkRowWire`, `ArtifactLinkIndexWire`,
  `upsert_artifact_link_row`, `artifact_link_dedup_key`). The dedup key is (source_ref,
  relation, target_ref) with undirected relations sorted. A rewrite of an existing edge
  with origin `read`/`prompt_ref` increments `uses` by exactly +1 and replaces the
  description; `created_at`/`created_by` stay fixed. This upsert is NOT idempotent:
  replaying the identical read row three times yields uses 1, 2, 3.
- **Storage split.** Document-kind endpoints go to sidecar `links/*.json`; `bead:`
  endpoints go to the bead event store as `LinkAdded`/`LinkRemoved` events
  (`src/sase/sdd/_artifact_link_store_bead_rows.py`,
  `src/sase/sdd/artifact_link_beads.py`); `agent:`/`stitch:`-only pairs live only in the
  rebuildable machine-local aggregate (`~/.sase/projects/<key>/artifact-links.json`).
  `NON_SIDECAR_KINDS` and path helpers live in
  `src/sase/sdd/_artifact_link_store_support.py`.
- **Outbox.** `src/sase/sdd/artifact_link_outbox.py` keeps a machine-local JSONL queue
  (`artifact-link-outbox.jsonl`, schema v1, 12-hex-char uuid ids). `_converged_rows`
  collapses all queued entries sharing (source, relation, target) into ONE row keeping
  the max `uses` — distinct repeated reads are lost as separate operations. Entries
  retire only after a successful upsert AND commit; commit failure retains everything.
- **Machine write lane (epic sase-y3, landed).** Background link maintenance never
  writes into a primary checkout's nested sidecar clones:
  `src/sase/sdd/_artifact_link_machine_store.py` resolves hidden host-owned clones at
  `~/.sase/projects/<key>/repos/<role>`, freshly integrated from the remote per use;
  every background writer must pass
  `authorize_store_mutation(root, mutation_origin="machine")`
  (`src/sase/sdd/_artifact_link_authorize.py`). Callers today: the hourly
  `sase_chop_artifact_link_backfill` chop and the Referenced-By publication drain.
  Audited reads still write the agent's own workspace sidecar clone — that is the lane
  this epic closes.
- **Commit choke point.** `commit_artifact_link_indexes`
  (`src/sase/sdd/_artifact_link_commit.py`) stages only paths passing the canonical
  checks in `src/sase/sdd/_artifact_link_files.py` and pushes through the same
  `run_managed_sync_worker` (`src/sase/bead/sync_worker.py`) machinery beads use, with
  one push attempt from `commit_artifact_link_indexes` callers and no retry/aging (open
  bug `bead:sase-ye`).
- **Bead precedent to mirror.** Bead conflicts resolve semantically:
  `src/sase/bead/conflict_resolver.py` reads the three git stages (note
  `_upstream_and_local_stages` returns (2,3) during rebase and (3,2) otherwise), calls
  Rust `merge_bead_event_streams_with_relocation`
  (`crates/sase_core/src/bead/events.rs`), writes byte-identical output
  (`conflict_resolver_store_writer.py`: `separators=(",", ":")`, `ensure_ascii=False`),
  stages, and lets the caller `rebase --continue`. Locks: worker lock (outer,
  `<git-dir>/sase-bead-sync.lock`) then store write lock (inner,
  `src/sase/sdd/_git_contention.py`), never reversed; the store lock is non-reentrant
  (hand off with `handoff_store_git_write_lock`).
- **Serialization gotcha.** `atomic_write_json` (`src/sase/agents_sync/io.py`) writes
  `indent=2, sort_keys=True` plus trailing newline. Any resolver or writer must
  reproduce the exact bytes the normal writer would produce, or every merge creates
  spurious diffs.

## Target design

Adopt the research recommendation: immutable, content-addressed link events with
deterministic Rust reduction, published through the existing host-owned machine lane,
with a conservative legacy resolver as rollout protection.

Non-negotiable design rules (justified in the research report):

1. **One immutable file per operation.** Durable automatic mutations become event
   objects at `link-events/v1/<first-2-sha-hex>/<sha256-of-canonical-bytes>.json` in the
   owning document sidecar — a namespace independent of document paths. Ordinary
   commands only create objects; nothing rewrites, renames, compacts, or deletes a
   published object. Different operations add different paths; a retry re-adds identical
   bytes. Git then has nothing to conflict on.
2. **Full-strength operation identity.** Every event carries a schema version, project
   identity, a 128-bit random operation id minted once at enqueue time, the canonical
   relation-aware edge identity, operation type, provenance (`created_by`, origin,
   timestamp), and any observed-predecessor ids. The payload is frozen before
   enqueueing. Same path with different bytes is corruption: reject it.
3. **Deterministic reduction in Rust.** `sase-core` owns event validation,
   canonicalization, and reduction to the existing `ArtifactLinkRowWire` projection.
   `uses` = count of distinct observation operation ids (plus imported baseline counts),
   so retry never double-counts and a genuine repeated read counts once each. Manual
   puts record superseded versions; concurrent descriptions resolve by a documented
   event-id tie-break with both versions retained for inspection. Removal is an add-wins
   tombstone naming the versions the remover observed; a concurrent unseen addition
   survives. The reducer must yield the same logical result for any file order, arrival
   order, or duplicate delivery — test those properties directly.
4. **Host-owned publication.** Automatic writers durably enqueue first and update the
   local pending view; one publisher per project/machine batches operations into the
   hidden host-owned clones, holds the publication lock across integration and writes,
   and retries rejected pushes bounded-ly. Operation payloads are retained until
   publication to every required owner is verified. This extends, not replaces, the
   sase-y3 lane and closes the `bead:sase-ye` retry gap.
5. **Bead endpoints keep bead-store ownership.** Document-side events carry the same
   operation id through the bead adapter so repair never mints a second mutation.
6. **Legacy protection stays.** The `links-resolver` phase's semantic merge remains
   wired while any history or straggler binary can still produce v2 index conflicts.

Boundary rule: all shared semantics (event model, canonical bytes, reduction, index
merge) land in the linked `sase-core` repo (`sase repo open sase-core`) with tests
there, exposed through `sase_core_rs` bindings (`crates/sase_core_py/src/lib.rs`,
contract list in `tools/validate_sase_core_rs`), then consumed by thin Python adapters
here — per the rust_core_backend_boundary memory. Phases that add bindings must ratchet
the binding floor the way prior epics did and run both repos' verification (`just check`
here; the sase-core repo's own gate there).

## Semantic resolver for link-index conflicts

Contain the bleeding first; this phase is independently shippable and immediately stops
the observed failure class.

- **Rust.** Add
  `merge_artifact_link_indexes(base, ours, theirs) -> ArtifactLinkIndexWire` to
  `crates/sase_core/src/artifact_link/` in sase-core:
  - Validate all three documents (schema v2, `artifact_ref` agreement).
  - Union rows by relation-aware dedup key. Rows whose key appears on only one side of
    the base merge cleanly (add/add of distinct readers — the dominant case).
  - Same-key rows: if exactly one side differs from base, take that side. If both differ
    identically, take either. Otherwise fail closed with a structured error naming the
    ambiguous keys — do NOT guess counters (`max(uses)` and `ours+theirs-base` are both
    wrong for some histories) or "later" descriptions (`created_at` never changes on
    rewrite).
  - Missing base (both-added file) is the empty index.
  - Deterministic output order: keep base/ours row order, then new rows sorted by
    (created_at, dedup key) so both sides of a symmetric merge produce identical bytes.
  - Expose as a `sase_core_rs` binding; property-test symmetry, idempotence, and the
    ambiguity refusals.
- **Python.** Add a links conflict resolver module beside
  `src/sase/bead/conflict_resolver.py` reusing `conflict_resolver_git.py` primitives
  (stage reads, rebase-aware ours/theirs ordering, `git_add`) and writing through the
  same `atomic_write_json`-equivalent bytes the normal writer produces. Handle:
  - both-modified and both-added `links/**/*.json` via the Rust merge;
  - modify/delete conservatively: fail closed (deletion may be a legacy rename repair);
  - anything else: not claimed.
- **Wire both paths.**
  - `_repair_or_abort_rebase` (`src/sase/sdd/_repository_integration.py`): replace the
    single `resolve_bead_conflicts` call with a resolver chain (bead resolver, links
    resolver); each round every unmerged path must be claimed by some resolver or the
    existing abort behavior stands. Extend `SddIntegrationStatus` handling
    (`_repository_types.py`) if a new repaired status is clearer than reusing
    `REPAIRED_BEAD_CONFLICTS`; update `succeeded()` and the machine-recovery whitelist
    in `_repository_transaction.py` accordingly.
  - `_continue_rebase_resolving_beads` (`_git_commit_dispatch.py`): generalize
    `_all_bead_conflicts` to "every conflicted path is claimed by a registered
    resolver", dispatch each resolver, then `rebase --continue`. Keep the current
    agent-facing paused-conflict message for unclaimed paths.
- **Out of scope here:** managed `## Links`/`## Referenced By` Markdown block conflicts
  (event-readers phase) and any change to how links are written.
- **Tests.** Rust unit/property tests; Python resolver tests mirroring
  `tests/sdd/test_artifact_link_*`; an integration test that constructs two clones,
  commits divergent reader rows to the same index, and proves both repair paths complete
  without pausing; a test proving an ambiguous same-key conflict still fails closed with
  the informative message.

## Immutable link-event contract and reducer in Rust core

All in sase-core, additive, inert until later phases call it.

- **Event model** (`crates/sase_core/src/artifact_link/`, new module): serde wire
  structs for a versioned event envelope with `schema_version`, `project_key`,
  `operation_id` (32 lowercase hex chars = 128 bits), `kind`, canonical edge identity
  (reuse `artifact_link_dedup_key` canonicalization), provenance (`created_by`,
  `origin`, `created_at`), and kind-specific payload:
  - `observation` — one actual read or prompt-ref occurrence; retry reuses the id;
    prompt batches carry an explicit occurrence count with stable identity.
  - `edge-put` — manual/migrated/derived assertion with description and the list of
    superseded operation ids it observed.
  - `edge-remove` — tombstone naming observed operation ids; add-wins semantics;
    tombstones are retained even when their referenced events have not arrived.
  - `alias` — artifact identity/rename mapping (old ref -> new ref) applied during
    reduction; handles chains deterministically; a reused filename must not inherit an
    unrelated artifact's identity.
  - `baseline-import` — one-shot legacy snapshot carrier (per-edge rows with preserved
    uses/provenance, source heads, import identity) used by cutover-import.
- **Canonical bytes:** a specified canonical JSON serialization (sorted keys, fixed
  separators, no floats, trailing-newline decision documented); the content address is
  the sha256 of those bytes; validation rejects a path/digest mismatch or same-path
  different-bytes.
- **Reducer:** `reduce_link_events(events, aliases) -> rows` producing
  `ArtifactLinkRowWire` rows: `uses` = distinct observation ids + baseline count;
  descriptions from the causally latest `edge-put`/observation with the documented
  event-id tie-break; tombstone add-wins; global dedup by operation id across owner
  copies (the same event may exist in two sidecars when both endpoints are documents).
  Deterministic under permutation and duplication — property-test exactly that.
- **Bindings:** expose validate/canonicalize/reduce/merge-path helpers through
  `crates/sase_core_py/src/lib.rs`, extend `tools/validate_sase_core_rs`'s contract
  list, and ratchet the Python-side binding floor.
- **No behavior change in this repo** beyond the floor ratchet; Python adapters come in
  later phases.

## Durable operation identity in the link outbox

Rework `src/sase/sdd/artifact_link_outbox.py` so the queue is an operation journal, not
a row cache:

- Outbox schema v2: each entry is a frozen canonical event payload (from the
  event-contract bindings) plus queue metadata; ids become the event's 128-bit operation
  id. Read/keep compatibility with existing v1 entries (drain them under the old
  semantics or migrate them to synthetic events at first drain — choose one, document
  it, and test it).
- Delete the count-collapsing behavior of `_converged_rows` for v2 entries: distinct
  operation ids are distinct events; batching happens at publication, not by discarding
  operations. Preserve the v1 convergence only for leftover v1 entries.
- Retirement: an entry leaves the queue only after its event file's publication is
  verified (commit present in the hidden clone's integrated history or pushed head), or
  via the existing stale-terminal drop lane (which must keep its audit trail). Survive
  interruption after file creation, after commit, after push, and before acknowledgement
  — idempotent replay by content address makes re-publication safe.
- Local pending view: queued-but-unpublished operations must be visible to the aggregate
  rebuild (event-readers phase consumes this; this phase provides an API to enumerate
  pending events).
- The eager `store.upsert_row` call in `_record_read_link` stays in this phase (behavior
  unchanged); event-publisher removes it behind the flag.

## Automatic link writes publish as events through the machine lane

- **Flag.** Create the beta feature flag with `sase flag new link_events` (kind `beta`,
  default off) as epic scaffolding; registry entry + both-states tests per the flags
  memory note. The cutover-import phase removes it.
- **Enqueue-only writers (flag on).** Audited reads (`_record_read_link`), prompt-ref
  citation recording, derivation sweeps, and backfill stop calling `store.upsert_row`
  for document endpoints and stop dirtying any agent-checkout `links/**` path. They mint
  the event, enqueue it (outbox-identity), update the local aggregate pending view, and
  report honestly (a read records "queued", not "published"). Bead endpoints keep their
  existing bead-store write, now carrying the operation id through the adapter. Flag off
  preserves today's path untouched.
- **Publisher.** A per-project/machine publisher (mirror `run_managed_sync_worker`
  structure and its worker-lock/store-lock ordering) drains pending events in batches:
  writes event objects into the hidden host-owned clones from
  `resolve_machine_artifact_link_store`, one commit per batch per repo, integrates and
  pushes with bounded retry on non-fast-forward rejection, and only then acknowledges
  retirement. Extend `src/sase/sdd/_artifact_link_files.py` so canonical
  `link-events/v1/**` paths are committable, and `_artifact_link_commit.py` to stage
  them; publication failures leave a retryable queue — never an agent merge task. This
  closes `bead:sase-ye` (single push attempt, no retry/aging) for link publication; note
  the fix on that bead rather than duplicating it.
- **Finalizer.** With the flag on, the finalizer's
  `_auto_commit_artifact_link_indexes_if_possible` no longer commits agent-checkout link
  dirt for event-lane origins (there is none); it may still trigger a publisher kick so
  an agent's reads publish promptly.
- **Human commands** (`sase artifact link add`/`rm`) switch to the same operation API
  but keep synchronous acknowledgement: they publish through the machine lane before
  reporting success, preserving the current published-on-success contract.
- **Tests:** flag-off parity (existing suites pass unchanged), flag-on enqueue-only
  behavior, publisher batch/retry/ack, crash-injection between create/commit/push/ack,
  and the hidden-clone e2e pattern from
  `tests/sdd/test_artifact_link_hidden_clone_e2e.py`.

## Readers, projections, and maintenance consume reduced events

- **Aggregate rebuild** (`_artifact_link_store_aggregate.py`, `reconcile`): rows =
  reduce(durable events across owning sidecars, alias set) merged with legacy v2 index
  rows (pre-import) plus the pending-outbox overlay, then existing projected rows.
  Pre-cutover, legacy indexes and events coexist; reduction must not double-count an
  edge that exists in both (baseline-import defines the final dedup, but until then
  event-lane edges are disjoint from legacy edges by construction — assert that).
- **Neighborhood and list reads** (`load_artifact_rows`, `artifact_link_neighborhood`,
  `link list --source store`) read reduced truth; `--source index` keeps including
  machine-local pending and projected rows.
- **Doctor** (`src/sase/artifact_cli/link_health.py`, `doctor.py`): event-store health
  (digest mismatches, orphaned tombstones, alias cycles, pending-queue age with the p95
  age surfaced, unpublished batches) alongside existing checks.
- **Renames:** with the flag on, `_artifact_link_renames.py` emits `alias` events
  through the machine lane instead of rewriting event files (legacy index rewriting
  stays for legacy files until cutover). Late events naming an old ref reduce correctly
  through the alias chain.
- **Managed Markdown blocks:** regeneration stays restricted to the machine lane
  (`_artifact_link_refresh.py` already runs there post-sase-y3). Extend the
  links-resolver chain to regenerate managed-block-only conflicts in artifact Markdown
  from reduced truth, preserving the authored body; an authored-body conflict remains a
  normal document conflict.
- **Tests:** reducer-adapter parity (same rows before/after enabling events for the same
  logical history), pending-overlay visibility, doctor findings, alias reduction,
  managed-block-only conflict regeneration.

## Fence, import legacy indexes, and cut over

Ordered, one-shot, user-authorized:

- **Fence.** A capability marker in the sidecar (e.g. a versioned
  `link-events/STORE.json`) plus a version floor check: capable binaries refuse legacy
  index writes once the marker is present; the import command refuses to run until the
  operator confirms every machine that writes this project's links (per the tailnet
  fleet) runs a capable release. Document plainly that a marker cannot stop an old
  binary — the links-resolver merge stays wired as the safety net for stragglers, and a
  doctor check flags any post-import legacy index modification loudly.
- **Import.** New CLI (follow cli_rules: sorted help, short aliases, no required
  options): freeze the canonical sidecar heads, produce `baseline-import` events from
  the authoritative legacy state (dedup endpoint copies by edge identity, preserve
  uses/provenance, never invent historical operation ids), record source heads and
  import identity, validate (reduced rows == legacy rows exactly), publish through the
  machine lane, and only then resume event publication. Preview by default; mutate only
  with `--apply` (matching artifact retention conventions). After import, baseline
  events replace raw legacy indexes as reducer input — indefinite raw-plus-event union
  would resurrect removed links and double-count.
- **Legacy read-only.** Legacy `links/**/*.json` become read-only history: writers
  refuse, doctor guards, aggregate ignores them post-import (baseline carries their
  content). Do not delete them in this epic.
- **Flag removal.** Enable `link_events` by default, delete the Off branch (the legacy
  document-endpoint write path in `upsert_row`/`_upsert_sidecar` writers, the
  finalizer's link-dirt auto-commit, v1 outbox handling), make the On branch
  unconditional, remove the registry entry, and close the flag bead in the same change
  per the flags memory note.
- **Tests:** import determinism (same input heads -> byte-identical baseline events),
  refusal paths, post-import straggler simulation (legacy write appears -> doctor flags
  it, resolver still merges it, nothing is silently lost).

## Multi-clone acceptance suite and conflict-free guarantee

An end-to-end suite (marker-gated if slow) with independent clones and two simulated
machines exercising, at minimum:

- add/add on a brand-new report; many concurrent readers of one hot report;
- exact retry (same operation id) versus genuine repeated read (distinct ids);
- concurrent descriptions; remove/add and remove/update races; stale clones publishing
  late; duplicate endpoint delivery of the same event; bead+document mixed links;
- rename plus a late read that still names the old ref;
- crash/kill at every publication boundary (after event write, after commit, after push,
  before acknowledgement) followed by clean recovery;
- import replay and managed-block regeneration.

Required outcomes: zero user-visible link-metadata merge pauses across the suite; no
lost or double-counted operations; verified content hashes; equal logical projections
from equal event sets regardless of order. Record (as test-emitted diagnostics or doctor
counters) pending-queue p95 age and link-only commit counts so batching savings can be
measured instead of assumed — the research explicitly declined to project them.

## Epic-wide working agreements

- Phase workers record discovered work as `PROPOSED FOLLOW-UP:` notes on their own phase
  bead — never new beads. Suggested land-agent follow-up: file a memory-type task bead
  proposing a decisions-web strand for "artifact links are immutable events" (the
  sase-y3 lane record task `bead:sase-yd` is adjacent precedent).
- Rust-first ordering per the core-boundary memory: land wire/API + tests in the linked
  `sase-core` repo, ratchet the binding floor, then adapt Python callers here. Open
  sase-core only via `sase repo open sase-core`.
- Verification is two-speed: `just check` per phase (after `just install` in a fresh
  workspace); the land agent runs `just check-full` through a monitor before landing the
  combined tree. sase-core changes additionally run that repo's own gates.
- Never hand-edit generated pages, bead statuses, or memory files; use the governing
  skills/commands.
