---
tier: epic
title: Global agent identities and reconstructable family sidecars
goal:
  Local machines persist identity-relative agent names while agents sidecars and
  SASE_AGENT commit provenance use globally unique username.machine.agent names; one
  agent commit publishes its reconstructable top-level hood and families automatically;
  remote integration, the incoming-update badge, comprehensive updates, and explicit
  sync are reliable, fast, and truthful.
phases:
  - id: identity-core
    title: Rust global identity and portable relationship contract
    depends_on: []
    size: medium
    description:
      "'Rust global identity and portable relationship contract' section: define
      canonical owner/global/local name transforms, hood and family classification, and
      validated batch relationship rewrites in sase-core and pyo3."
  - id: identity-config
    title: Nested user/machine configuration and initializer migration
    depends_on:
      - identity-core
    size: medium
    description:
      "'Nested user/machine configuration and initializer migration' section: migrate
      machine overlays to id.username/id.machine_name, prompt and diagnose required
      identity, and update config, init, doctor, and docs."
  - id: local-storage
    title: Identity-relative local persistence and lookup
    depends_on:
      - identity-core
      - identity-config
    size: large
    description:
      "'Identity-relative local persistence and lookup' section: remove hood prefixes
      from new local writes, localize imported identities, and preserve unambiguous
      lookup/display compatibility for legacy and foreign names."
  - id: sidecar-v2
    title: Sidecar v2 hood snapshots, family import, and Markdown
    depends_on:
      - local-storage
    size: large
    description:
      "'Sidecar v2 hood snapshots, family import, and Markdown' section: export complete
      top-level hood/family snapshots, transactionally reconstruct their relationships,
      migrate v1 safely, and generate deterministic overview pages."
  - id: commit-publish
    title: Linked SASE_AGENT provenance and automatic commit publication
    depends_on:
      - sidecar-v2
    size: medium
    description:
      "'Linked SASE_AGENT provenance and automatic commit publication' section: remove
      new SASE_MACHINE tags, link the full global SASE_AGENT identity to sidecar
      Markdown, and checkpoint targeted post-commit publication."
  - id: incoming-status
    title: Foreign-update detection snapshots and cached integration
    depends_on:
      - sidecar-v2
      - commit-publish
    size: medium
    description:
      "'Foreign-update detection snapshots and cached integration' section: distinguish
      foreign manifest changes during periodic fetches and add exact-SHA, no-network
      integration separate from full synchronization."
  - id: admin-sync
    title: Incoming-only badge, cached comprehensive update, and Updates-tab sync
    depends_on:
      - incoming-status
      - commit-publish
    size: medium
    description:
      "'Incoming-only badge, cached `,U`, and Updates-tab `a`' section: show only
      foreign incoming work, apply captured fetched changes without network access, and
      add a tracked full-sync action to the Admin Center."
  - id: chezmoi-migration
    title: Athena chezmoi identity migration
    depends_on:
      - identity-config
    size: small
    description:
      "'Athena chezmoi identity migration' section: migrate sase_athena.yml to
      id.username bbugyi200 and id.machine_name athena in the linked chezmoi repository
      and apply it through the repository's required workflow."
  - id: verification
    title: Cross-machine end-to-end verification and rollout docs
    depends_on:
      - commit-publish
      - incoming-status
      - admin-sync
      - chezmoi-migration
    size: medium
    description:
      "'Cross-machine end-to-end verification and rollout docs' section: exercise
      multi-user/multi-machine identity, hood publication, linked provenance, atomic
      family revival, cached and explicit sync, migrations, and recovery."
create_time: 2026-09-09 19:52:58
status: wip
---

- **PROMPT:**
  [prompts/202607/agents_sidecar_identity_and_families.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/agents_sidecar_identity_and_families.md)

# Plan: Global agent identities and reconstructable family sidecars

## Goal

Correct the hidden `--agents` sidecar so local machines keep identity-relative agent
names while sidecars and primary repository commit provenance use globally unique
`<username>.<machine_name>.<agent>` names; publish the committing agent's complete
top-level hood and family data automatically from `sase commit`; reconstruct those
relationships transactionally on other machines; generate polished, linkable agent and
family Markdown; and make background, comprehensive-update, and explicit-sync behavior
fast and truthful.

## Context

The completed `sase-8k` epic established the agents sidecar, `machine_name`,
machine-qualified local names, per-agent transport bundles, a full-duplex sync engine,
and an ACE status badge/comprehensive-update leg. Its implementation is a useful
foundation, but several of its design premises need to change:

- It embeds `<machine_name>.` in every locally persisted name (`agent_meta.json`, chats,
  the name registry, launch metadata, and relationship fields). Machine and username
  hoods should instead be a transport/provenance namespace, not part of locally owned
  agent state.
- It exports only completed agents that themselves have a primary-repository commit. A
  commit by one member should publish every locally owned agent in that member's
  top-level hood, recursively, including all members and structural metadata for any
  family in that closure.
- Its v1 bundle cannot faithfully reconstruct a family as a unit: imports allocate
  timestamps independently, omit relationship/launch artifacts needed by the loader and
  dismissal archive, and have no atomic source-run-ID to destination-run-ID rewrite.
- It emits both `SASE_AGENT` and `SASE_MACHINE`; `SASE_AGENT` is plain text and cannot
  take readers to the agent history.
- Its badge treats local ahead/unexported/error state as a remote update, and `,U`
  performs a network sync across every enabled project instead of applying only updates
  already discovered by the periodic checker.
- Explicit network synchronization exists as an app action/CLI but has no dedicated `a`
  action in the Admin Center's Updates tab.

The implementation spans the primary Python/TUI repository, the linked `sase-core` Rust
repository, and the linked `chezmoi` repository. Open linked repositories through
`/sase_repo`; do not assume a sibling path.

## Settled identity contract

### Three representations

Treat identity conversion as an explicit boundary operation. Do not infer ownership from
an arbitrary dotted string.

1. **Global/sidecar form:** `<username>.<machine_name>.<local-name>`, for example
   `bbugyi200.athena.foo.bar.baz--code`. This is the only form written into new v2
   sidecar keys, sidecar-generated Markdown, and new `SASE_AGENT` primary-repository
   tags.
2. **Local durable form:** the shortest unambiguous spelling relative to the configured
   identity:
   - source username and machine both match local config → `foo.bar.baz--code`;
   - username matches but machine differs → `athena.foo.bar.baz--code`;
   - username differs → `alice.athena.foo.bar.baz--code`.
3. **Legacy form:** existing local `<machine>.<agent>` records and v1 sidecar entries
   remain readable. Do not mass-rename historical artifact/chat directories. Normalize
   them at lookup/export boundaries, and ensure every newly written or refreshed local
   record uses the local durable form.

Every imported record must retain explicit `source_username`, `source_machine`,
`global_name`, source run ID, and source digest provenance. Name parsing alone is not a
substitute for those fields. Dismissal-date prefixes stay outside identity qualification
exactly as they do today.

### Config form and initialization

The selected machine overlay owns:

```yaml
id:
  username: bbugyi200
  machine_name: athena
```

`id.username` is mandatory for launches, commit provenance, and agents-sidecar
operations. The initializer explains that it **must be globally unique among SASE
users**, **should be identical on all of one user's machines**, and that a GitHub
username is the recommended value. It cannot prove global uniqueness, so it validates a
path/hood-safe, dot-free segment and obtains explicit user input.

Keep the bounded machine-local `~/.sase/machine_name` selector: it selects one machine
overlay but is not an agent-name prefix. Overlay discovery recognizes `id.machine_name`;
top-level `machine_name` remains read-only migration input. `sase config init` and bare
`sase init` migrate the selected overlay in place by writing both nested fields and
removing the legacy top-level key while preserving unrelated YAML/comments. A configured
nested identity is idempotent.

### Hood/family export scope

For a committing local agent, remove any family `--<role>` suffix only for structural
classification, then take the first dotted local hood segment. If `foo.bar.baz--code`
commits, the export scope is every locally owned project agent whose semantic name is
`foo`, starts with `foo.`, or is a family member rooted anywhere in that closure. This
includes active artifacts, terminal artifacts, and dismissed/archive bundles; excludes
imported foreign agents; and includes agents that never committed themselves.

Family containers are structural records even when no bare agent named for the family
exists. A closure export is one consistent snapshot: source run IDs, parent IDs, wait
edges, family names, clan names, and every other name-bearing relationship are
globalized together.

### Import/revival contract

Import one changed hood snapshot transactionally under the existing project sync lock:

1. Validate every bundle and relationship before writing.
2. Allocate destination artifact timestamps/run IDs as a batch, preserving source IDs
   when free and deterministically probing collisions.
3. Build a complete source-run-ID → local-run-ID map and rewrite `parent_timestamp`,
   workflow-child references, retry references, wait dependencies, and group records
   through it.
4. Localize all names according to the three-representation contract and claim both
   local spellings and global provenance without overwriting an existing unrelated
   claim.
5. Stage portable artifact markers, raw prompts, transcripts when available, and group
   records; atomically publish the staged state, then update the artifact/name indexes.
   Roll back the whole hood on failure.

Imported snapshots are historical/inactive, never represented as live remote processes.
After loading and dismissal, the existing `R` flow must be able to revive a whole
imported family with its membership, parent/child structure, prompts, and display names
intact.

## Sidecar v2 layout and generated Markdown

Retain the machine-level hidden clone and privacy/visibility policy. Upgrade the
transport to a strict, dual-read v2 schema:

```text
README.md
manifest.json
agents/
  bbugyi200.athena.foo.bar.baz--plan/
    README.md
    meta.json
    state.json
    prompt.md
    chat.md                 # optional while a source agent is still active
    commits.json
  bbugyi200.athena.foo.bar.baz--code/
    ...
families/
  bbugyi200.athena.foo.bar.baz.md
hoods/
  bbugyi200.athena.foo.md
```

- `manifest.json` records schema version, explicit username/machine ownership, per-agent
  digest, per-family digest, source run IDs, and hood snapshot generations. It never
  relies on splitting a directory name to establish ownership.
- `meta.json` is a portable allowlist of identity, model/provider, project/change,
  family/clan/tribe, and source timing data. `state.json` contains the portable
  marker/relationship fields required to materialize loader-compatible historical
  artifacts. `prompt.md` stores the exact persisted `raw_xprompt.md`; `chat.md` is
  optional until available and refreshes the digest later. Absolute paths, PIDs,
  workspace numbers, credentials, and machine-local paths remain forbidden.
- `agents/<global-name>/README.md` is a deterministic, attractive overview with identity
  breadcrumbs, source state, model/provider, timing, project/change association,
  commits, family/hood links, and links to prompt/transcript.
- `families/<global-family>.md` is generated whenever a family first appears and
  regenerated when it changes. It includes a compact lineage/tree, an ordered
  role/member table, status/model/timing summaries, commits, and direct links to every
  member page and transcript. Rootless family containers are presented honestly.
- `hoods/<username>.<machine>.<top-level>.md` provides a recursive tree and links to all
  agents/families in the snapshot. The root README links username → machine → hood
  indexes, making the repository navigable without reading JSON.
- Markdown is deterministic for a given manifest: stable ordering, no generated-at
  timestamp in page bodies, no local paths, and byte-identical regeneration. Digests
  include the canonical source records, not volatile rendering.

Read v1 manifests/bundles for migration. Entries owned by the current machine can be
re-exported under the configured v2 global identity and retired only after their
replacement validates. Foreign v1 entries remain supported as username-unknown legacy
provenance; do not guess a username or silently merge ambiguous records. Document and
test the mixed v1/v2 period.

## Commit provenance and automatic publication

New primary commits contain no `SASE_MACHINE` line. Runtime updates remove an
inherited/existing `SASE_MACHINE`, and PR tag inheritance continues filtering it so it
cannot reappear. Historical `SASE_MACHINE` remains read-only input for v1 backfill.

`SASE_AGENT` uses `LinkedCommitTagValue` and the shared Rust footer formatter, following
`SASE_PLAN`:

```text
SASE_AGENT=[bbugyi200.athena.foo.bar.baz--code][1]

[1]: https://github.com/<owner>/<project>--agents/blob/<branch>/families/bbugyi200.athena.foo.bar.baz.md
```

A family member links to its family overview; a non-family agent links to
`agents/<global-name>/README.md`. Generate the URL from the configured agents-sidecar
remote and its resolved branch using the existing hosted-remote URL helpers. For a
non-hosted/disabled/not-created sidecar, retain the full global label without inventing
a GitHub URL and report the publication limitation clearly.

After a successful `create_commit` or `create_pull_request` dispatch and after
`write_result_marker` has made the new association visible, `sase commit` attempts a
sync for the current project only. Proposals do not publish. Make this an idempotent
checkpointed tracking step so `sase commit --resume` retries publication without
creating another primary commit. A primary commit remains successful if sidecar network
publication fails: leave the local sidecar/outbox and status visibly pending, print an
actionable warning, and let the next commit, `sase agent sync`, or Admin Center `a`
retry it. Never claim that the sidecar push succeeded when it did not.

The exporter must support the current agent before `done.json` exists. A commit-time
snapshot may omit a transcript but must still publish prompt, state, family
relationships, and commit attribution; later terminal/full sync refreshes the same
global entry rather than creating a duplicate.

## Sync and ACE behavior

### Periodic detection and badge

The periodic long-cadence worker remains the only background network fetch. After
fetching, compare the validated local and fetched-upstream v2 manifests (using
`git show <upstream>:manifest.json`, without checking out or integrating) and record:

- fetched upstream commit SHA;
- identities and agent/hood digests changed since local HEAD;
- count of changes owned by identities other than the exact local username+machine;
- detection/fetch time and any validation error.

The badge is visible only when an enabled project has a validated incoming change from
another machine (including another machine belonging to the same username). Local ahead
state, pending exports, missing sidecars, and ordinary errors remain available in
CLI/task diagnostics but do not light the incoming-update badge. Unknown-owner v1
incoming data is conservatively considered foreign. Clicking the badge applies the
already-fetched snapshot; it does not fetch all projects.

### `,U` / comprehensive update

The preview captures immutable agents status from the last successful periodic
detection. The agents leg is runnable only for project/SHA pairs with detected foreign
changes. Execution:

- performs no fetch and no `git pull`;
- locks each selected sidecar;
- rebases/fast-forwards local sidecar commits against the exact already-fetched commit
  object captured by the preview;
- imports/localizes only the validated data through that captured manifest;
- does not export or push unrelated local state;
- skips truthfully if the captured object is unavailable or the project was already
  integrated.

Thus `,U` scales with detected work, not with the total number of enabled projects. A
newer periodic fetch does not silently widen an already-confirmed preview.

### Explicit Admin Center sync

Add `a` → **Sync agents** to the Admin Center Updates pane. It delegates to one tracked,
deduplicated background task that runs full-duplex `sync_agents()` for every enabled
project: fetch/pull, import, export, commit, and push. It is the explicit "check
everything now" path, refreshes local agent/index state, rewrites the status snapshot,
updates the badge, and reports a per-project summary. Add the binding to pane hints/help
and guard it while an agents sync or comprehensive update owns the same task scope.

Keep `sase agent sync` as the CLI equivalent. Its `--check` remains no-network by
default and `--refresh` remains the explicit network status refresh.

## Scope boundaries and safety

- Do not edit `sase/memory/*.md`, `AGENTS.md`, or provider instruction shims. The user
  requested product design and a chezmoi machine-config migration, not a SASE memory
  update.
- Do not mass-rename existing local artifact/chat directories or rewrite historical
  primary commits.
- Do not republish imported foreign bundles as locally owned data.
- Do not let malformed/untrusted sidecar data write outside the machine state root,
  overwrite unrelated name claims, or partially materialize a hood.
- Keep the hidden agents sidecar out of launched workspaces and generated memory exactly
  as today.
- Preserve privacy consent, `disabled`, and `visibility: private` behavior. Update
  consent/docs to explain that a commit publishes its full top-level local hood, not
  just the committing agent.
- Shared identity conversion, hood membership, family-base parsing, and portable
  relationship validation belong in `sase-core`; Python/TUI code calls bindings/facades
  rather than reimplementing them.
- Every TUI network/disk operation stays off the event loop and Textual message pump.
  Periodic callbacks remain thin; user mutations use tracked tasks; render paths consume
  immutable snapshots only.

## Phase 1: Rust global identity and portable relationship contract

**ID:** `identity-core`  
**Depends on:** none  
**Size:** medium

In the linked `sase-core` repository, replace the machine-prefix-only domain API with an
explicit source/local identity model while retaining compatibility bindings for old
callers during the Python migration.

- Add validated `AgentOwnerIdentity { username, machine_name }` and global agent
  identity operations: globalize a locally owned/legacy-current name for sidecar use;
  localize a global name for a target identity; classify exact-local,
  same-user/other-machine, other-user, and username-unknown legacy ownership; preserve
  dismissal prefixes.
- Define canonical helpers for agent family base, family-member detection, top-level
  hood extraction, and recursive hood membership that correctly handle dotted hoods and
  `--<role>` family suffixes.
- Define/validate the portable v2 relationship wire used by import: global name-bearing
  fields, source run IDs, parent/retry/wait edges, family/clan records, and batch
  rewrite against a destination run-ID map. Reject missing targets, cycles where
  forbidden, duplicate global identities/run IDs, path-unsafe components, and
  cross-owner relationships.
- Expose typed pyo3 bindings and a Python facade contract. Keep old machine-hood
  functions only as deprecated migration shims until all primary-repo callers move.
- Cover the identity matrix, idempotence, legacy machine-only names, family examples,
  hood boundary false positives, relationship rewrites, timestamp collisions, and
  malformed payloads in Rust/binding tests. Do not edit release-plz version fields.

Run the linked core repository's formatting/lint/test suite and land/rebuild the binding
before dependent Python work.

## Phase 2: Nested user/machine configuration and initializer migration

**ID:** `identity-config`  
**Depends on:** `identity-core`  
**Size:** medium

In the primary repository, migrate config, initialization, doctor, and docs from
top-level `machine_name` to the nested identity object.

- Update `src/sase/config/sase.schema.json` with optional-per-document `id`, whose
  selected machine overlay must resolve valid `username` and `machine_name` fields; no
  bundled default. Add `get_agent_identity()` / `require_agent_identity()` and
  compatibility `get_machine_name()` accessors backed by the nested object.
- Update overlay selection/discovery in `src/sase/config/core.py` so `id.machine_name`
  marks machine overlays and the existing selector chooses one. Recognize top-level
  `machine_name` only as migration input. Include all relevant selector/config stats in
  the existing cache token without adding YAML parsing to render paths.
- Rework `src/sase/main/config_init_handler.py`: migrate a selected legacy overlay;
  prompt for username whenever absent; explain global uniqueness/same-user reuse/GitHub
  recommendation before accepting it; reuse the one existing username as a convenience
  when unambiguous; detect conflicting usernames across known machine overlays;
  minimally set nested keys and unset the legacy key through the source-preserving YAML
  helpers and chezmoi write/deploy path.
- Make `sase config init`, `sase init config`, bare `sase init`, `--check`, and
  `sase doctor` report missing/legacy/ conflicting identity states accurately. Agent
  launch, commit provenance, and sidecar operations use the required accessor; unrelated
  read-only CLI/config repair remains usable before initialization.
- Update parser/help text and `docs/configuration.md` / `docs/agents_sidecar.md`. Follow
  CLI sorting/short-option rules.
- Test clean creation, machine-only migration, missing username, reuse across two
  machine overlays, conflicting users, non-TTY failure, selector/cache behavior,
  comment-preserving nested edits, chezmoi direct/deferred deployment, init-order,
  doctor, and schema inventory.

Run `just install` before `just check`.

## Phase 3: Identity-relative local persistence and lookup

**ID:** `local-storage`  
**Depends on:** `identity-core`, `identity-config`  
**Size:** large

Replace the current "qualify on every local write, strip on display" policy with
explicit local-storage and transport boundaries.

- Replace `src/sase/core/machine_hood_facade.py` with an identity facade whose APIs make
  intent obvious: `to_local_storage_name`, `to_global_sidecar_name`,
  ownership/provenance classification, and lookup candidates.
- Audit every existing `qualify_local_agent_name` call. New locally launched agents
  persist bare local names in name registry claims, planned-name/env metadata,
  `agent_meta.json`, `done.json`, chats, workflow/family/clan fields, waits, output
  variables, retries, and attachment resolution. No machine or username hood is added on
  these writes.
- Localize imported global records by the identity matrix. Preserve explicit provenance
  in artifacts and registry records so global references resolve without guessing.
  Reject/diagnose collisions rather than auto-renaming.
- Keep read compatibility for existing local `athena.*` records on athena and for any
  early `bbugyi200.athena.*` records: present and resolve them as bare, but do not
  rewrite directories wholesale. Same-user foreign-machine and other-user names remain
  visibly qualified at the required level.
- Update agent hood/family/clan grouping, completion, chats, name
  allocation/reservation, family attachment/promotion, prompt references,
  archive/dismissed bundles, and the artifact index so equivalent local/legacy/global
  references converge on one identity without collapsing unrelated users who share a
  machine name.
- Add focused audit tests over local persistence choke points, plus
  launch/family/clan/lookup/chat/TUI cases for all three identity classes and legacy
  data.

Run `just install` before `just check`, including visual tests if displayed tree/name
snapshots change.

## Phase 4: Sidecar v2 hood snapshots, family import, and Markdown

**ID:** `sidecar-v2`  
**Depends on:** `local-storage`  
**Size:** large

Evolve `src/sase/agents_sync/` from completed per-agent bundles to transactional hood
snapshots.

- Add v2 typed models/strict I/O and the layout above. Build candidate inventory from
  active/terminal artifacts and dismissed/archive bundles via existing indexed APIs;
  select the committing agent's whole top-level locally owned hood; include family
  containers and every required prompt/state/relationship record even when members have
  no commits or are not terminal.
- Support targeted closure export for commit-time publication and full reconciliation
  export for `sase agent sync`. Hash stable portable records, allow an initially absent
  chat, refresh in place when state/chat/commits change, and never export imported
  provenance as local ownership.
- Generate deterministic per-agent, family, hood, and root Markdown with stable
  links/order and human-readable empty/ rootless states. Stage `manifest.json`,
  `agents`, `families`, and `hoods` together.
- Implement dual-read v1/v2 validation and the conservative migration policy. Upgrade
  current-owned v1 entries only after validating their v2 replacements; retain foreign
  username-unknown entries without guessing.
- Import a hood as one staged transaction using the Rust relationship rewrite. Allocate
  timestamps as a batch, localize names and group fields, reconstruct loader-compatible
  historical markers/raw prompts/chats, claim provenance, and update indexes only after
  atomic publication.
- Extend status/outcome counts from "agents" to snapshots/agents/families without
  breaking JSON consumers silently; bump every affected schema version.
- Test the exact `foo.bar.baz--code` closure example; active/terminal/dismissed members;
  non-committing members; rootless families; optional-chat refresh; two users with the
  same machine name; two machines for one user; v1 mixed migration;
  corrupt/cyclic/path-traversal input; timestamp collision rewrite; rollback;
  idempotence; and import → load → dismiss family → `R` revive-family behavior.
- Update sidecar seed README/consent text and `docs/agents_sidecar.md` for the broader
  publication scope.

Run `just install` before `just check`.

## Phase 5: Linked `SASE_AGENT` provenance and automatic commit publication

**ID:** `commit-publish`  
**Depends on:** `sidecar-v2`  
**Size:** medium

Connect primary commits to the new pages and make commit-triggered publication
automatic.

- Refactor `src/sase/workflows/commit/runtime_tags.py`: write only runtime `AGENT`;
  remove stale/inherited `MACHINE`; retain legacy `MACHINE` parsing solely in historical
  backfill; return `LinkedCommitTagValue` for hosted agents sidecars with the full
  global label and deterministic agent/family URL.
- Add an agent-link helper parallel to `plan_paths.py`, resolving configured agents
  sidecar remote, branch, family base, and safe GitHub blob URL. Update Rust
  footer/parser parity only if the existing generic linked value contract exposes a gap.
- Add an idempotent `publish_agents_snapshot` tracking step after the first result
  marker in `CommitWorkflow._run_tracking_steps` for successful commit/PR methods.
  Resolve and sync only the current enabled project, and checkpoint success so resume
  cannot duplicate work.
- Make automatic publication report exported families/agents and push state. Treat
  sidecar failure as a visible, retryable auxiliary failure without invalidating the
  already-created primary commit; preserve local pending evidence and ensure later full
  sync retries it.
- Update PR-body agent display to use the global linked identity consistently where
  Markdown is supported.
- Test linked standalone/family footers and reference numbering with `SASE_PLAN`; no
  `SASE_MACHINE` on new or rewritten commits/PR inheritance; plain fallback for
  disabled/non-hosted sidecars; pre-terminal commit export; current-project targeting;
  checkpoint/resume idempotence; push failure then retry; and historical v1 footer
  backfill.

Run `just install` before `just check`.

## Phase 6: Foreign-update detection snapshots and cached integration

**ID:** `incoming-status`  
**Depends on:** `sidecar-v2`, `commit-publish`  
**Size:** medium

Separate network detection, cached inbound integration, and full synchronization in
`src/sase/agents_sync/`.

- Bump the atomic status snapshot schema to record fetched SHA and changed source
  identities/digests. After periodic fetch, validate the upstream manifest through
  `git show` and compare it with local HEAD; count only exact-identity foreign changes
  as incoming. Treat username-unknown v1 data conservatively.
- Keep no-network revalidation cheap and preserve detected SHA evidence until integrated
  or superseded. Ahead, unexported, missing, and error details remain observable but are
  not `incoming_foreign_changes`.
- Add a cached-integration API accepting captured project/SHA pairs. It performs no
  network operation, updates against the exact fetched object under lock, imports
  validated changes, never exports/pushes, and returns typed stale/already-
  applied/object-missing outcomes.
- Keep full `sync_agents()` semantics for explicit CLI/Admin Center actions and
  automatic targeted publication. Ensure all modes serialize on the same per-project
  lock and rewrite status truthfully.
- Tests assert network call counts: periodic recompute fetches; periodic revalidate does
  not; cached integration never fetches/pulls; full sync does. Cover same-machine remote
  changes ignored, same-user other-machine changes detected, same-machine-name
  other-user changes detected, mixed identities, deletions, v1 unknown ownership,
  captured-SHA immutability, stale object, and concurrent auto/full/cached operations.

Run `just install` before `just check`.

## Phase 7: Incoming-only badge, cached `,U`, and Updates-tab `a`

**ID:** `admin-sync`  
**Depends on:** `incoming-status`, `commit-publish`  
**Size:** medium

Wire the new mode split into ACE without adding work to render or keypress paths.

- Change `AgentsSyncIndicator` and formatting so only projects with
  `incoming_foreign_changes > 0` render the badge. Tooltips name the source machine(s)
  and explain that click/`,U` applies already detected data. Badge click submits the
  cached-integration tracked task, not full sync.
- Capture the prior periodic agents snapshot in `ComprehensiveUpdatePreview`; make the
  agents leg runnable only for its detected project/SHA pairs; render exactly those
  projects in confirmation; execute the no-network cached-integration API. Never widen
  the captured preview after confirmation.
- Add `a` / `sync_agents` to `PluginsBrowserPane.BINDINGS`, guarded availability, hints,
  help, and command wording. It delegates to the shared tracked full-sync action, uses
  the existing `agents-sync` exclusive scope/dedup key, refreshes agents/index/status on
  completion, and keeps the Admin Center responsive.
- Preserve the pump-safe timer/worker/coalescing shape in `actions/agents_sync.py`;
  perform UI mutations only on the UI thread and all git/manifest/artifact work in
  workers/tracked tasks.
- Test badge filtering/click behavior, tooltip/source rendering, missing/error states
  staying hidden, preview capture, exact target execution, zero network calls for `,U`,
  `a` binding and dedup/task reporting, concurrent task guards, indicator refresh, and
  full comprehensive summaries.
- Update PNG visual snapshots for the incoming-only badge and measure that Agents-tab
  j/k p95 remains unaffected.

Run `just install` before `just check` and run `just test-visual` for intentional
snapshot changes.

## Phase 8: Athena chezmoi identity migration

**ID:** `chezmoi-migration`  
**Depends on:** `identity-config`  
**Size:** small

In the linked `chezmoi` repository, edit `home/dot_config/sase/sase_athena.yml` from:

```yaml
machine_name: athena
```

to:

```yaml
id:
  username: bbugyi200
  machine_name: athena
```

Preserve surrounding configuration and ordering as much as the source-preserving
migration permits. Validate the YAML and confirm the primary repository's new config
loader selects it through the existing local selector. Do not edit memory files. If this
phase commits through the normal finalizer, follow the linked repository instruction to
run `chezmoi update -a --force` after that commit so the migrated config is applied.

## Phase 9: Cross-machine end-to-end verification and rollout docs

**ID:** `verification`  
**Depends on:** `commit-publish`, `incoming-status`, `admin-sync`, `chezmoi-migration`  
**Size:** medium

Exercise the full design with temporary SASE homes and local bare remotes before
declaring the epic done.

- Simulate `bbugyi200.athena`, `bbugyi200.zeus`, and `alice.athena` on one project.
  Assert local writes are respectively bare, sibling-machine-qualified, or fully
  qualified while the shared sidecar always uses complete global names.
- Build a `foo` hood containing a multi-member `foo.bar.baz` family plus
  sibling/descendant agents, make only `foo.bar.baz--code` commit, and prove automatic
  publication includes the entire expected closure and beautiful pages; the linked
  primary footer resolves to the family page.
- On each remote identity, perform periodic detection, assert the badge excludes
  exact-local changes and includes the other two identity classes, run `,U` with all
  network runners set to fail if called, and verify only the captured project/SHA is
  integrated.
- Dismiss the imported family and use the real `R` revival flow to restore the complete
  group and relationships. Cover timestamp/name collisions and a mid-import failure with
  no partial state.
- Press Updates-tab `a` to prove all enabled projects perform full duplex sync and
  badge/status/index state refreshes.
- Exercise automatic-push failure, later retry, disabled/private/not-created/non-hosted
  sidecars, v1 mixed data, corrupt manifests, and two users whose machines have the same
  name.
- Finish CLI help and docs, including migration guidance: run `sase config init` on
  every machine, the nested identity requirement, broader privacy implications, linked
  commit examples, badge/`,U` cached semantics, and explicit `a` / `sase agent sync`
  recovery.
- Run `just install`, `just check`, focused local-bare-remote integration tests, and
  `just test-visual`. Record any intentionally deferred cleanup as a bead rather than
  weakening verification.

## Acceptance criteria

- No newly written locally owned agent record embeds the configured username or machine
  hood; sidecar and new `SASE_AGENT` values always carry both.
- Same-machine, same-user/different-machine, different-user/same-machine-name, and
  legacy v1 identities remain distinct and resolve to the specified local forms.
- One family-member commit automatically attempts to publish the entire locally owned
  top-level hood, including non-committing and dismissed members, with deterministic
  agent/family/hood pages.
- A remote sync reconstructs relationship-complete historical artifacts atomically, and
  a dismissed imported family is revivable through `R`.
- New primary commits and PR bodies never add `SASE_MACHINE`; `SASE_AGENT` is the full
  global identity and links to the correct agent/family page whenever the hosted sidecar
  URL is available.
- The agents badge means exactly "another machine has fetched changes waiting"; local
  pending/error state does not light it.
- `,U` performs no agents-sidecar network fetch and integrates only the immutable work
  detected earlier; Updates-tab `a` is the explicit full-network sync.
- The requested athena config is migrated to `id.username: bbugyi200` and
  `id.machine_name: athena`.
- All changed repositories pass their required checks, and no
  memory/generated-instruction file is edited.
