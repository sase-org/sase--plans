---
tier: epic
title: Global agent hoods, reconstructable sidecars, and cached remote sync
goal: Keep locally owned agent names identity-relative on disk while publishing globally
  unique username.machine.agent identities; make one agent commit publish its complete,
  revivable top-level hood with deterministic agent/family overviews; replace SASE_MACHINE
  with a linked SASE_AGENT; and make remote detection, cached integration, and explicit
  full synchronization truthful, fast, and recoverable.
phases:
- id: identity-domain
  title: Rust owner identity and relationship domain
  depends_on: []
  size: large
  description: Define explicit username/machine ownership, global/local name transforms,
    hood/family classification, and portable relationship validation/rewrite in sase-core
    and its Python bindings.
- id: identity-config
  title: Nested identity config and initializer migration
  depends_on:
  - identity-domain
  size: medium
  description: Move machine_name under id, require id.username in the selected machine
    overlay, migrate legacy overlays through config/init/doctor, and document the
    stable per-user identity contract.
- id: local-persistence
  title: Identity-relative local persistence and registry compatibility
  depends_on:
  - identity-domain
  - identity-config
  size: large
  description: Stop adding the current machine hood to locally owned records, preserve
    explicit imported provenance, and keep legacy qualified records resolvable without
    mass-renaming history.
- id: sidecar-publish
  title: Owner-sharded v2 hood snapshots and beautiful overviews
  depends_on:
  - identity-domain
  - local-persistence
  size: large
  description: Publish complete project-scoped top-level hood snapshots, active/terminal/dismissed
    family data, deterministic owner indexes, and linkable agent/family Markdown through
    a conflict-resistant v2 sidecar layout.
- id: sidecar-revival
  title: Transactional import, v1 migration, and family revival
  depends_on:
  - sidecar-publish
  size: large
  description: Validate and import hood snapshots as recoverable batches, rewrite
    all run relationships, preserve conditional local naming, support legacy v1 safely,
    and expose synced families directly to the R revival flow.
- id: commit-publication
  title: Linked SASE_AGENT and automatic commit publication
  depends_on:
  - sidecar-publish
  size: medium
  description: Stop producing SASE_MACHINE, render the full global SASE_AGENT as an
    agent/family link, and add an idempotent post-commit targeted publish with a durable
    retry outbox.
- id: incoming-cache
  title: Foreign-only detection cache and no-network integration
  depends_on:
  - sidecar-revival
  size: large
  description: Materialize validated immutable snapshots during periodic fetches,
    track per-hood import receipts, and provide a no-network cached integration API
    distinct from full-duplex synchronization.
- id: updates-experience
  title: Incoming-only badge, cached comprehensive update, and Updates-tab sync
  depends_on:
  - incoming-cache
  - commit-publication
  size: medium
  description: Make the badge and indicator click foreign-update-only, make ,U consume
    its captured cache without fetching, and add a tracked Updates-tab a action for
    full synchronization of all enabled projects.
- id: athena-migration
  title: Athena chezmoi identity migration
  depends_on:
  - identity-config
  size: small
  description: Migrate sase_athena.yml to id.username bbugyi200 and id.machine_name
    athena in the linked chezmoi repository and verify selection/application through
    the repository's required workflow.
- id: end-to-end
  title: Cross-machine verification, rollout, and documentation
  depends_on:
  - local-persistence
  - sidecar-revival
  - commit-publication
  - updates-experience
  - athena-migration
  size: medium
  description: Exercise three identities, exact hood closure, linked publication,
    direct family revival, cached and explicit sync, legacy migration, collision/recovery
    cases, visual behavior, and final user documentation.
create_time: 2026-07-23 12:58:55
status: wip
---

# Plan: Global agent hoods, reconstructable sidecars, and cached remote sync

## Goal

Correct the hidden `<project>--agents` sidecar so a machine's own agent data is stored with local names, while sidecar
records and primary-repository provenance use globally unique `<username>.<machine_name>.<local-agent-name>` identities.
A commit by any member must publish the complete project-scoped top-level hood, including every family and
non-committing member in that closure, with enough portable state to reconstruct and revive a family on another machine.
The published repository must be pleasant to browse, and the TUI must distinguish lightweight, already-fetched incoming
integration from an explicit full network sync.

This is an epic because the shared Rust domain, config migration, broad local-persistence reversal, sidecar wire,
transactional importer, commit workflow, update engine, TUI, and linked chezmoi repository require independently
landable phases with explicit compatibility seams.

## Current implementation and corrections

The existing implementation provides a useful base:

- Top-level `machine_name`, selected by the bounded machine-local `~/.sase/machine_name` file.
- Rust/Python machine-hood qualification helpers.
- Machine-qualified names throughout locally owned artifact metadata, chats, registry entries, family/clan metadata, and
  commit footers.
- A machine-level hidden agents clone with `manifest.json` plus
  `agents/<machine-qualified-name>/{meta.json,chat.md,commits.json}`.
- A full-duplex `sync_agents()` transaction, status snapshot, periodic fetch, top-bar indicator, `sase agent sync`,
  indicator-click action, and comprehensive-update leg.

The corrective design must account for these concrete limitations:

- Local qualification is spread across launch, allocation, family/clan, registry, rename, revival, chat, and metadata
  paths. Reversing only the launch writer would leave mixed new state.
- The v1 exporter scans only completed artifacts in the target project, requires a transcript and at least one commit
  per exported agent, and therefore cannot publish the active committer or commit-less family/hood members.
- The v1 importer creates/refreshed agents independently. It does not batch-remap timestamps, parent/retry/wait edges,
  raw prompts, embedded workflows, or family archive records, so it cannot promise a coherent `R` revival.
- One top-level manifest is rewritten by every machine. That will become a merge-conflict hotspot once multiple users
  and machines publish concurrently.
- `sync_agents(projects=...)` still runs `git pull --rebase` and network pushes; filtering projects does not make it a
  cached/no-network integration API.
- The existing indicator predicate lights for ahead, unexported, missing, and error states, not specifically for foreign
  data waiting to be imported.
- The existing commit workflow writes `SASE_AGENT` and `SASE_MACHINE` before dispatch, but the artifact commit marker
  needed for targeted publication is written only after dispatch.

Implementation spans the primary SASE repository, the linked `sase-core` repository, and the linked `chezmoi`
repository. Each linked repository must be opened through the repository-access workflow and landed according to its own
instructions.

## Settled design contracts

### Identity is explicit data, not a dotted-string guess

The selected machine overlay defines:

```yaml
id:
  username: bbugyi200
  machine_name: athena
```

`id.username` is a path-safe, dot-free SASE identity segment. It must be unique among SASE users, should be identical on
every machine owned by the same user, and should normally be the user's GitHub username. SASE can validate syntax and
require explicit input, but it cannot prove global uniqueness.

`id.machine_name` is unique among that user's machines. The existing machine selector remains `~/.sase/machine_name`; it
selects an overlay but is not an agent-name prefix.

Do not derive source ownership by splitting an arbitrary local name. Every v2 record carries an explicit source
`{username, machine_name}`, and every imported local record retains that owner plus its canonical global identity and
content digest.

### Three deliberate name forms

1. **Global transport/provenance form** `<username>.<machine_name>.<local-name>`, for example
   `bbugyi200.athena.foo.bar.baz--code`. New sidecar manifests, records, pages, and `SASE_AGENT` values always use this
   form.
2. **Locally owned durable form** `foo.bar.baz--code`. New records created by the current machine never embed its
   username or machine name.
3. **Imported durable form**, derived from an explicit source owner:
   - matching username and matching machine: treat as the current machine's own round-trip and do not duplicate it;
   - matching username, different machine: `zeus.foo.bar.baz--code`;
   - different username: `alice.athena.foo.bar.baz--code`.

Dismissal-date prefixes are archive presentation state, not global identity. Strip them before globalizing a semantic
agent name and carry dismissal time/state as fields. Legacy dismissed spellings remain readable locally.

The name registry gains explicit origin/global-name metadata for imports. It also reserves every configured sibling
machine hood and every observed foreign username hood as a container namespace. If a pre-existing local name conflicts
with an incoming qualified spelling, quarantine that hood import and report the exact collision; never overwrite,
silently rename, or partially materialize it.

### Identity config has one authoritative source

The selected machine overlay is the sole owner of `id.username` and `id.machine_name`. Defaults, plugins, ordinary
overlays, and project-local config must not be able to change commit provenance per project. Identity accessors resolve
the selector and selected raw overlay as one cached snapshot, while the ordinary merged config can continue to expose
the selected `id` object for display.

Top-level `machine_name` is read-only migration input. New writers only emit the nested object. A legacy or partially
configured installation may inspect config, run init/doctor/help, and view local history, but agent launches, new commit
provenance, and agents-sidecar mutations fail with an actionable `sase config init` diagnostic until both fields
resolve.

### Export scope is exact and privacy-bounded

For the committing agent:

1. Remove a terminal family role suffix such as `--code` for classification.
2. Find the first dotted local hood segment (`foo` for `foo.bar.baz--code`).
3. Within the target project's artifact, dismissed, and archive inventory, include every locally owned agent whose
   semantic name is exactly `foo`, begins with `foo.`, or is a family member rooted anywhere in that closure.
4. Include every sequential-family and clan/container record needed to preserve those memberships, even when a bare
   family container is structural and has no corresponding agent process.

For the example `foo.bar.baz--code`, this covers `foo.bar.baz--plan`, the `foo.bar.baz` family, existing ancestors such
as `foo.bar` and `foo`, and siblings/descendants such as `foo.boom`, `foo.boom.bam`, and `foo.bar.kazam`.

The closure includes active, waiting, terminal, failed, dismissed, and archived agents, including agents with no commits
and an active committer whose `done.json` or final transcript does not exist yet. Imported foreign records are never
re-exported as locally owned. The project boundary is intentional: publishing a commit must not leak an unrelated
private project's same-named hood into this project's sidecar. Update consent text to make the broader within-project
hood/transcript publication explicit.

Targeted commit publication exports that one hood. A full explicit sync reconciles every locally owned hood in the
project that has at least one primary-repository commit association.

### Portable snapshots are reconstructable, not machine clones

Every exported run has a stable source run ID and a strictly allowlisted record containing:

- explicit global/local name and source owner;
- source project/change identity and source artifact layout/run ID;
- model, provider, reasoning effort, role, tribe, family, clan, and recorded status/timing;
- parent, workflow-child, retry, wait, family, and clan relationships expressed through stable source run IDs or global
  names;
- exact raw xprompt, portable embedded-workflow metadata, and prompt-step data needed to restart;
- commit metadata for the sidecar's primary project;
- an optional transcript/response when one exists.

Never transport PIDs, credentials, workspace numbers, absolute paths, host paths, or other machine-local execution
state. Imported snapshots are historical snapshots, never live remote processes.

Import validates a whole hood before mutation, allocates all destination artifact IDs as a batch, rewrites every
relationship through one source-to-destination map, and uses a transaction journal so crashes can finish or roll back
without leaving a half-family. Derived indexes and registry entries are rebuilt/updated only after the artifact batch is
committed.

Each imported family also receives an idempotent saved-group/archive record keyed by global family plus snapshot digest.
It appears in the existing `R` modal with an "Agents sidecar" source label, so a user can revive the complete family
directly without first dismissing individually imported rows. The ordinary Agents loader may still show imported
historical rows.

### V2 is owner-sharded and Git-friendly

New repositories use the following conceptual layout:

```text
README.md
schema.json
users/
  bbugyi200/
    README.md
    machines/
      athena/
        README.md
        manifest.json
        hoods/
          foo/
            snapshot.json
            README.md
agents/
  bbugyi200.athena.foo.bar.baz--code/
    README.md
    meta.json
    state.json
    prompt.md
    chat.md                  # optional until available
    commits.json
families/
  bbugyi200.athena.foo.bar.baz.md
```

The exact local/global identity remains explicit inside every JSON record. Directory names are validated path
components, never used as the sole ownership proof.

Each machine writes only its own owner manifest and globally unique agent/family paths. Username, machine, hood, and
root Markdown indexes are deterministic derived views. Sharding removes the single manifest as a routine cross-machine
conflict; full sync regenerates derived indexes after merging owner manifests.

An owner manifest maps top-level hoods to immutable snapshot digests and referenced files. A hood snapshot is
append/refresh-oriented: ordinary export never deletes previously published runs merely because local cleanup made an
artifact temporarily unavailable. Explicit future tombstones require their own validated wire and are not invented in
this epic.

Pages contain no volatile "generated at" value and are byte-identical for identical snapshot content. A family page uses
breadcrumbs, a compact summary, an accessible lineage tree (optionally enhanced by GitHub Mermaid), an ordered
role/member table, source states, model/provider/timing, commit links, and links to each member's prompt/transcript.
Agent pages provide the corresponding solo detail. Root, username, machine, and hood pages make the repository navigable
without opening JSON.

### Legacy v1 is read-only and never guessed

Existing v1 `manifest.json` and `agents/<machine-qualified-name>` data remain readable during migration. New code does
not write v1.

- A v1 entry matching the current machine may be promoted to the configured v2 identity only when a matching local owned
  artifact/commit association proves ownership.
- A foreign v1 machine name has unknown username provenance. Preserve that fact, treat it as foreign for update
  detection, and never guess that it belongs to the current user merely because the machine token matches.
- Keep foreign v1 data importable through a legacy-owner compatibility path, but never republish it as v2 local data.
- Do not mass-delete or rename the existing sidecar payload. V2 owner manifests coexist until a separately authorized
  cleanup can safely retire v1.

Historical primary commits containing `SASE_MACHINE` remain readable solely for this conservative v1 backfill.

### Commit provenance is linked and eventually consistent across two repositories

New commit messages contain:

```text
SASE_AGENT=[bbugyi200.athena.foo.bar.baz--code][1]

[1]: https://github.com/<owner>/<project>--agents/blob/<branch>/families/bbugyi200.athena.foo.bar.baz.md
```

A family member links to its family page and stable member anchor. A solo agent links to its agent README. Use the
configured agents-sidecar remote, resolved branch, and hosted-remote URL helpers; never synthesize a GitHub URL for a
disabled, missing, or non-hosted sidecar. The fallback is the full global label as plain text.

The primary commit must be dispatched before its canonical commit marker/association exists, so two repositories cannot
be updated atomically. The chosen behavior is:

1. Build the deterministic link and full global label before primary dispatch.
2. After successful commit/PR dispatch and the first durable result marker, run a checkpointed, target-project/hood
   publication attempt.
3. On success, the sidecar page and its commit association are pushed.
4. On sidecar failure, leave the already-created primary commit successful, record a durable outbox item, print an
   actionable warning, and let the next `sase commit`, `sase agent sync`, or Updates-tab `a` retry it.

A stable branch link can briefly be unresolved between the primary push and sidecar push; do not pretend cross-repo
atomicity. Checkpoint and outbox idempotence must prevent a resume from creating a duplicate primary commit or duplicate
sidecar record.

Stop producing `SASE_MACHINE` everywhere, including raw SDD auto-commits. Keep separate sets for tags SASE produces
(`AGENT`) and stale runtime tags it removes (`AGENT`, `MACHINE`) so an inherited old `SASE_MACHINE` cannot survive a
rewrite. The generic footer parser remains tolerant of historical tags.

### Three sync modes have distinct contracts

1. **Periodic detection** is the only background network path. On its longer cadence it fetches each enabled agents
   sidecar, reads owner manifests/snapshots from the fetched commit without checking them out, validates changed foreign
   hoods, and atomically materializes an opaque immutable incoming cache plus a status snapshot.
2. **Cached inbound integration** (indicator click and `,U`) accepts captured cache IDs/SHA/digests, performs no fetch,
   pull, push, export, or sidecar-checkout mutation, imports only those validated snapshots, and advances durable
   per-project/per-owner/per-hood receipts. Receipts prevent the same behind Git history from repeatedly lighting the
   badge.
3. **Full duplex sync** (`sase agent sync`, repo init where applicable, commit-targeted publish, and Updates-tab `a`)
   acquires the shared project lock, fetches/reconciles the sidecar, imports foreign changes, drains publication
   outboxes, exports eligible local hoods, regenerates indexes, commits, and pushes with bounded retry.

The visual indicator means exactly "validated changes from another exact `{username, machine}` identity are cached and
not yet imported." Same-user/other-machine is foreign; exact-current-identity changes are not. Ahead/unexported/error/
missing states remain visible in CLI/task diagnostics but do not light the badge.

## Safety and scope boundaries

- Do not edit SASE memory files, generated agent-instruction files, or provider shims.
- Do not mass-rename historical local artifact/chat directories or rewrite historical primary commits.
- Do not republish imported data under the current identity.
- Keep agents sidecars out of launched workspaces and generated memory, as they are today.
- Preserve `disabled`, visibility/private consent, non-hosted remote, and not-created behavior.
- Treat sidecar input as untrusted: strict schema versions, bounded sizes/counts, path containment, safe UTF-8/JSON,
  digest verification, relationship validation, and no writes before whole-hood validation.
- Serialize periodic cache writes, cached import, full sync, and commit publication with compatible per-project locks.
  Lock contention produces a typed retry/skip, never an unbounded UI wait.
- All TUI disk, Git, JSON, and import work stays off the event loop and Textual message pump. Periodic callbacks remain
  synchronous scheduling stubs; user mutations run as tracked tasks; render paths consume immutable snapshots.
- Shared identity transforms, hood/family membership, and relationship validation/rewrite belong in `sase-core`.
  Python/TUI code calls bindings or thin adapters rather than reimplementing domain rules.
- Do not manually edit release-plz-managed versions in the core repository.

## Phase 1: Rust owner identity and relationship domain

**ID:** `identity-domain`  
**Depends on:** none  
**Size:** large

In the linked `sase-core` repository, replace machine-prefix inference with explicit ownership operations while keeping
temporary compatibility exports for the current Python callers.

- Add a validated `AgentOwnerIdentity { username, machine_name }`. Username validation should accept a conservative,
  lowercase, dot-free SASE/GitHub-friendly segment (including digits and internal `-`/`_`), reject path separators,
  family delimiters, empty segments, and reserved internal namespaces, and provide actionable errors. Reuse existing
  machine validation.
- Add explicit operations:
  - globalize a locally owned semantic name using a supplied owner;
  - verify/globalize legacy current-machine-qualified input;
  - localize a global name using supplied source and target owners;
  - classify exact owner, same user/other machine, other user, and username-unknown v1;
  - remove archive dismissal decoration before global identity construction.
- Add canonical family-member/base parsing, top-level local hood extraction, recursive hood membership, ancestor
  synthesis, and global family/solo link target selection. Handle dotted names and only a terminal `--<role>` suffix;
  reject boundary false positives (`foo` must not match `foobar`).
- Define the portable v2 run/relationship wire or the reusable validation core for it: source run IDs, parent/workflow/
  retry/wait edges, family/clan/container records, and batch destination-ID rewrite. Reject duplicate IDs/global
  identities, dangling required targets, illegal cross-owner references, cycles where the runtime forbids them, and
  unsafe components.
- Expose typed pyo3 bindings and a focused Python identity facade. Retain existing machine-hood symbols as deprecated
  migration shims only until Phase 3 moves every caller.
- Test the full owner matrix, idempotence, legacy-current qualification, family examples, dismissal handling, same
  machine name under two usernames, relationship rewriting/collisions, malformed payloads, and path boundaries.

Run core formatting, clippy, workspace tests, and binding tests. Land/rebuild the core binding before dependent primary
repository phases.

## Phase 2: Nested identity config and initializer migration

**ID:** `identity-config`  
**Depends on:** `identity-domain`  
**Size:** medium

Migrate configuration, onboarding, and diagnostics from top-level `machine_name` to one selected identity object.

- Add an `id` schema object with `username` and `machine_name`; retain top-level `machine_name` only as deprecated
  migration input. Do not add an identity default.
- Refactor selected-overlay discovery to recognize `id.machine_name` as the machine-overlay discriminator, with legacy
  fallback. Create a cached identity snapshot sourced only from the selector plus selected machine overlay. Reject or
  diagnose `id` in ordinary/default/plugin/project fragments so project configuration cannot change global provenance.
- Add `get_agent_owner_identity()` / `require_agent_owner_identity()` and compatibility accessors. Include selector and
  relevant raw overlay signatures in the existing cache token without parsing YAML in render paths.
- Rework `sase config init`:
  - detect selected legacy, partial, complete, missing, and conflicting overlays;
  - explain before prompting that the username **must** be globally unique, **should** be shared by all of this user's
    machines, and should normally be their GitHub username;
  - require explicit username input when no unambiguous existing user identity can be suggested;
  - offer/reuse a single existing username only with clear confirmation, and never guess among conflicts;
  - minimally write `id.username` and `id.machine_name` into the same machine overlay, remove its legacy top-level key,
    preserve unrelated YAML/comments, then write the existing selector;
  - keep TTY injection, chezmoi remapping/deferred deployment, no-commit/no-push/no-apply, and idempotence behavior.
- Make `sase init config`, bare `sase init`, `--check`, and doctor distinguish missing username, legacy migration,
  selector mismatch, and conflicting identities. Keep repair/help/config inspection usable before initialization while
  hard-requiring identity for launches, commit provenance, and agents mutations.
- Update configuration/init/agents-sidecar documentation and schema inventory tests.
- Test clean creation, legacy migration, partial nested config, multiple machines for one username, conflicting
  usernames, invalid/reserved values, non-TTY failure, source-preserving edits, selector/cache invalidation, project
  override rejection, chezmoi direct/deferred deployment, init ordering, and doctor.

Run `just install` before `just check`.

## Phase 3: Identity-relative local persistence and registry compatibility

**ID:** `local-persistence`  
**Depends on:** `identity-domain`, `identity-config`  
**Size:** large

Reverse the existing local-write policy at explicit ownership boundaries.

- Replace ambiguous `qualify_local_agent_name` usage with intent-specific facade APIs:
  `normalize_locally_owned_storage_name`, `globalize_owned_name`, `localize_imported_name`, ownership classification,
  and exact-first lookup candidates.
- Audit every current qualifier in launch planning/identity/env, metadata and done markers, name reservation/allocation,
  template/multi-prompt references, family promotion/attachment, clan membership, waits/output variables/retries,
  rename, revival, directive persistence, chats, and registry rebuild. Every new locally owned write stores a bare
  semantic name and bare family/clan container.
- Extend the name registry schema with explicit origin/global-name/source-owner/digest fields for imports. Same-source
  refresh is idempotent; a different owner collision is a typed error. Reserve sibling-machine and observed-username
  container namespaces so later imports cannot silently alias local descendants.
- Keep exact-first compatibility for existing `athena.*` local records and any early `bbugyi200.athena.*` records:
  display and resolve them as local when explicit current ownership is known, but do not rewrite their directories
  wholesale. Registry rebuild preserves legacy owners and detects ambiguous collisions.
- Localize v2 imports only from explicit source fields. Same-user foreign machines retain the machine hood; other users
  retain username and machine. Update hood/family/clan grouping, completion, prompt references, chat listings, and
  artifact-index projections accordingly.
- Ensure TUI presentation does not show redundant current username/machine hoods, but does show the conditional foreign
  hierarchy. Keep global provenance available in details/tooltips where useful.
- Add focused choke-point tests plus launch/family/clan/registry/lookup/chat/TUI matrices for new local, legacy local,
  same-user foreign, other-user same-machine-name, and collision/quarantine cases.

Run `just install` before `just check`; run visual tests if agent-tree rendering changes.

## Phase 4: Owner-sharded v2 hood snapshots and beautiful overviews

**ID:** `sidecar-publish`  
**Depends on:** `identity-domain`, `local-persistence`  
**Size:** large

Build the new export and browsing contract in `src/sase/agents_sync/`.

- Introduce strict, versioned v2 models/I/O for owner manifests, hood snapshots, run records, relationships, file
  digests, family/container records, and deterministic derived pages. Keep v1 readers separate rather than accepting a
  loose union schema.
- Inventory active/waiting/terminal artifacts plus dismissed bundles/archive records for the target project using
  indexed APIs. Starting from a committing agent, calculate the exact top-level closure and synthesize structural
  ancestors/family containers without treating them as process records.
- Export every locally owned run in the closure, whether or not it committed or has `done.json`/chat. Require raw prompt
  and portable restart metadata when available; an active run may have an absent transcript that is added on a later
  publish under the same source run ID.
- Add two entry points:
  - targeted hood publication used by `sase commit`;
  - full eligible-hood reconciliation used by explicit sync. Both share selection, hashing, and rendering, and neither
    exports imported provenance.
- Write only the current owner manifest and global paths. Stage the owner manifest, snapshot, agent files, family pages,
  and affected indexes together; regenerate all derived views deterministically after a remote merge.
- Generate polished root/username/machine/hood/family/agent Markdown with stable anchors and accessible text fallback.
  Escape all labels/URLs safely; include prompt/chat only through relative links. Add golden tests for representative
  solo, rootless-family, deep-family, active, and mixed-state hoods.
- Preserve prior published run records on temporary local absence; diagnose rather than emit implicit deletions.
- Update seed files and repo-init privacy consent to describe full project-scoped hood publication and optional active
  transcript refresh.
- Extend outcome/status JSON with versioned hood/family/run counts without silently changing v1 consumer meanings.
- Test the exact `foo.bar.baz--code` example, ancestors/siblings/descendants, active committer, non-committing plan
  member, dismissed member, rootless family, optional-chat refresh, idempotent rendering, owner sharding, two concurrent
  machines, paths/digests/size limits, and no foreign re-export.

Run `just install` before `just check`.

## Phase 5: Transactional import, v1 migration, and family revival

**ID:** `sidecar-revival`  
**Depends on:** `sidecar-publish`  
**Size:** large

Turn a validated v2 hood snapshot into coherent local historical/revival state.

- Read and validate every referenced file and relationship before taking the project import lock. Reject unsupported
  schema, oversized data, missing digests, unsafe paths, invalid UTF-8/JSON, bad owner/name agreement, dangling/cyclic
  relationships, and source-run duplication.
- Under the shared lock, allocate destination artifact timestamps/run IDs for the whole hood, preserving source IDs when
  free and deterministically probing collisions. Rewrite parent/workflow/retry/wait/family/clan references through one
  complete map.
- Localize names via the explicit owner matrix and preflight registry/container collisions. Exact-current-owner
  round-trips refresh provenance/receipts without creating duplicate artifacts.
- Stage loader-compatible historical `agent_meta.json`, terminal snapshot marker, raw xprompt, embedded-workflow data,
  prompt-step data, optional transcript, and portable source metadata. Never restore a remote PID/running marker.
- Add an import journal with prepare/apply/finalize/recovery states. Commit staged artifacts and chat files as one
  recoverable batch, then update artifact indexes, registry provenance, and per-family saved-group archive records.
  Tests must kill/restart at phase boundaries and prove either complete recovery or no visible partial family.
- Add idempotent `R` archive integration: one saved group per global family/snapshot digest, correct member order and
  parent mapping, readable source label, prompt preview, and revival/relaunch through the existing UI path. Refresh a
  group when a later digest supplies chat/final state.
- Implement dual v1/v2 reading. Promote current-owned v1 only with matching local evidence; classify all other v1 as
  username-unknown foreign provenance; never guess, merge, or republish it. Keep old sidecar files in place.
- Test import → load → direct `R` revive → relaunch, source/destination timestamp collisions, same-user stripping,
  different-user same-machine-name retention, active-source historical import, family/clan relationships, optional
  files, refresh/idempotence, collision quarantine, v1 ambiguity, and transactional recovery.

Run `just install` before `just check`, including relevant revival visual/integration tests.

## Phase 6: Linked SASE_AGENT and automatic commit publication

**ID:** `commit-publication`  
**Depends on:** `sidecar-publish`  
**Size:** medium

Connect primary commits to the new pages and publish the targeted hood automatically.

- Refactor runtime tags so produced keys contain only `AGENT`, while cleanup/inheritance filtering still removes both
  `AGENT` and legacy `MACHINE`. Remove all `SASE_MACHINE` producers, including raw SDD auto-commits. Keep historical
  parse/backfill tolerance.
- Resolve the current owner once and globalize the agent from env/artifact metadata. Return a `LinkedCommitTagValue`
  using the deterministic family/member or solo-agent target. Reuse hosted-remote and footer formatting utilities so
  `SASE_PLAN`/`SASE_AGENT` reference numbering remains correct.
- Add a sidecar-link resolver that uses the configured target remote and resolved branch, handles private GitHub repos,
  safely encodes paths/anchors, and falls back to a plain global name for disabled/missing/non-GitHub targets.
- After successful `create_commit` or `create_pull_request` dispatch and the first result marker, add a checkpointed
  `publish_agent_hood` tracking step. Proposals do not publish. Resolve the actual primary revision through the VCS
  provider instead of assuming the dispatch result is a SHA.
- Target only the current project and hood. Drain prior outbox items for that project, reconcile/fetch under the shared
  sync lock, export the active committer and closure, commit/push, and mark the tracking step complete.
- On auxiliary failure, persist an outbox record keyed by project/global agent/primary revision/hood digest, leave the
  primary operation successful, and print a concise retry warning. Full sync and later commits drain idempotently.
  Resume after a crash must reuse the marker/checkpoint and never re-dispatch the primary commit.
- Keep PR body/tag rendering consistent where linked values are supported.
- Test family/solo links and anchors, reference numbering with `SASE_PLAN`, stale inherited `SASE_MACHINE` removal,
  auto-commit behavior, hosted/plain fallback, active pre-terminal publication, exact project/hood targeting, commit vs
  proposal, provider revision resolution, crash/resume, push failure/outbox/retry, and legacy footer backfill.

Run `just install` before `just check`.

## Phase 7: Foreign-only detection cache and no-network integration

**ID:** `incoming-cache`  
**Depends on:** `sidecar-revival`  
**Size:** large

Separate background detection and cached import from full Git reconciliation.

- Version the atomic status snapshot to record, per project:
  - fetched upstream SHA/ref and fetch time;
  - validated owner/hood digests different from durable import receipts;
  - source username/machine and whether it is exact-current or foreign;
  - opaque immutable cache IDs, counts, validation/quarantine errors, and cache creation time.
- During the long-cadence periodic fetch, inspect v2 owner manifests and changed snapshots directly from the fetched Git
  object. Materialize only validated pending foreign hoods into an opaque digest-addressed cache via stage + atomic
  rename. Unknown-owner v1 is conservatively foreign.
- Maintain durable per-project/source-owner/top-hood import receipts. Compare fetched hood digests to receipts rather
  than raw Git `behind` count so already imported data does not reappear merely because the sidecar checkout remains
  behind. Exact-current-owner updates can advance observation metadata without lighting the badge.
- Keep short-cadence revalidation network-free: validate cache/receipt existence and local configuration only. Do not
  parse all manifests or scan all artifacts on every tick.
- Add `integrate_cached_agent_updates(captured_items)`:
  - accept immutable project/SHA/cache ID/hood digest records;
  - perform zero network calls and no sidecar checkout mutation;
  - revalidate cached bytes/digests, import through Phase 5, and advance receipts atomically;
  - return typed applied/already-applied/stale/missing/quarantined/failed outcomes.
- Keep `sync_agents()` as explicit full duplex behavior. Refactor shared locks and status rewriting so cached import,
  targeted publish, and full sync cannot overlap unsafely.
- Bound cache retention by preserving pending snapshots and the latest applied receipt evidence; prune old applied
  objects only off the UI thread.
- Test network call counts, cache immutability, exact-current ignored, same-user/other-machine detected, same-machine
  name/different-user detected, v1 unknown detected, multiple owners in one remote commit, corrupt snapshot quarantine,
  captured-SHA stability after a newer fetch, missing/pruned cache, receipt idempotence, and concurrency with full sync.

Run `just install` before `just check`.

## Phase 8: Incoming-only badge, cached comprehensive update, and Updates-tab sync

**ID:** `updates-experience`  
**Depends on:** `incoming-cache`, `commit-publication`  
**Size:** medium

Wire the three-mode sync contract into ACE without adding work to render or keypress paths.

- Replace the indicator predicate with `pending_foreign_hood_count > 0`. Do not show it for local ahead/unexported,
  missing clone/upstream, disabled/not-created, or errors. Tooltip lists projects and source username/machine/hood
  summaries from the immutable status snapshot.
- Make indicator click submit cached inbound integration for the currently displayed captured cache items, not full
  sync. Refresh receipts/status and use the existing fast agent-list background reload on completion.
- Capture immutable agents cache items in `ComprehensiveUpdatePreview`. The agents leg is runnable only when those items
  exist, displays exactly the captured projects/hood counts, and calls the no-network cached integration API. A newer
  periodic fetch must not widen an already-confirmed preview.
- Add `a` / **Sync agents** to the SASE Admin Center Updates pane bindings, availability, footer hints, help, and
  command text. Delegate to one shared tracked full-sync action over all enabled projects, with the existing
  `agents-sync` dedup/exclusive scope.
- The `a` task fetches/reconciles/imports/exports/pushes, drains outboxes, rewrites status/receipts, then schedules the
  existing pump-safe agent refresh and indicator revalidation. Report one truthful outcome line per project and keep
  partial failures non-fatal to other projects.
- Keep periodic callbacks thin/synchronous, Git/import in workers, and user mutations in the tracked task queue. Re-read
  mounted/current UI state after asynchronous completion and avoid full list rebuilds when the existing refresh path
  suffices.
- Keep `sase agent sync` as the CLI equivalent of `a`; `--check` remains local/cached by default and explicit refresh
  remains the network detection path.
- Test badge filtering/count/tooltip/click, cached preview capture, zero network calls from `,U`, stale captured items,
  `a` binding/hints/dedup/task log, overlap guards, post-import agent refresh, and per-project summary. Update PNG
  snapshots and verify Agents-tab navigation performance is unchanged.

Run `just install` before `just check`, and run `just test-visual` for intentional visual changes.

## Phase 9: Athena chezmoi identity migration

**ID:** `athena-migration`  
**Depends on:** `identity-config`  
**Size:** small

In the linked chezmoi repository, migrate `home/dot_config/sase/sase_athena.yml` from:

```yaml
machine_name: athena
```

to:

```yaml
id:
  username: bbugyi200
  machine_name: athena
```

Preserve surrounding ordering/content, validate YAML, and verify the selected-overlay identity accessor resolves
`bbugyi200.athena` with the existing machine selector. Do not edit memory or generated instruction files. If the phase
is committed, follow the linked repository instruction to run `chezmoi update -a --force` after the commit so the
migration is applied.

## Phase 10: Cross-machine verification, rollout, and documentation

**ID:** `end-to-end`  
**Depends on:** `local-persistence`, `sidecar-revival`, `commit-publication`, `updates-experience`, `athena-migration`  
**Size:** medium

Exercise the complete behavior using isolated SASE homes and local bare remotes before declaring the epic complete.

- Simulate `bbugyi200.athena`, `bbugyi200.zeus`, and `alice.athena` on the same enabled project. Prove locally created
  names are bare; imported names become respectively round-trip/no-duplicate, `zeus.<local>`, and
  `alice.athena.<local>`; all sidecar and footer identities stay fully global.
- Construct the exact `foo` hood from the request, including a `foo.bar.baz` plan/code family, ancestors, siblings,
  descendants, active/waiting/terminal/dismissed members, and agents with no commits. Make only `foo.bar.baz--code`
  commit and prove the complete expected closure, owner indexes, and deterministic pages are pushed.
- Follow the primary `SASE_AGENT` link to the family page/member anchor; verify solo links; assert no new `SASE_MACHINE`
  is produced or inherited.
- On another machine, import and load the cached family, open `R`, revive the synced saved group, and relaunch members
  with correct raw prompts, role order, parent/retry/wait relationships, model/provider, and conditional local names.
- Prove periodic detection ignores exact-current-owner changes and caches both foreign identity classes. Run indicator
  click and `,U` with every network runner configured to fail if called; prove only captured cache items are imported
  and receipts clear the badge.
- Press Updates-tab `a` to prove every enabled project performs full network reconciliation, drains a simulated failed
  commit-publication outbox, updates local agent state, and refreshes status/badge/indexes.
- Cover sidecar push failure/retry, disabled/private/not-created/non-hosted targets, concurrent publishers,
  derived-index conflict regeneration, namespace collision quarantine, corrupted/path-traversal/cyclic input, import
  crash recovery, cache pruning/staleness, v1 mixed data, and same machine names under different users.
- Finish CLI help and `docs/configuration.md`, `docs/init.md`, and `docs/agents_sidecar.md`: identity migration,
  publication/privacy scope, v2 browsing layout, linked provenance, active/optional transcript behavior, direct family
  revival, badge semantics, cached indicator/`,U`, explicit `a`, CLI recovery, outbox diagnostics, and v1 limitations.
- Run linked core checks, `just install`, `just check`, focused local-bare-remote integration tests, and
  `just test-visual`. Record any intentionally deferred cleanup as a bead rather than weakening acceptance.

## Acceptance criteria

- New locally owned agent, family, clan, registry, chat, and relationship writes contain neither the current username
  nor current machine hood; legacy qualified records remain readable without mass migration.
- Global sidecar records and every new `SASE_AGENT` label contain exact configured username and machine identity.
  Same-user/different-machine and different-user/same-machine-name imports remain distinct.
- One family-member commit automatically attempts to publish every active/terminal/dismissed, committing/ non-committing
  agent and structural family/container in that project's complete top-level hood.
- The sidecar is browsable through deterministic root/user/machine/hood/family/agent Markdown, and family/solo commit
  links resolve to the correct stable page/anchor after publication.
- New commits, PRs, rewrites, inherited tags, and raw SDD auto-commits do not produce `SASE_MACHINE`; historical parsing
  remains sufficient for conservative v1 backfill.
- A remote hood import is validated and recoverable as a batch, preserves all relevant relationships and restart data,
  and appears as a complete revivable family through `R`.
- A failed automatic sidecar push never causes a duplicate primary commit and is durably retried through the outbox.
- The indicator means only unapplied validated foreign hood changes. Indicator click and `,U` use immutable cached data
  with zero network calls; Updates-tab `a` is the explicit full-network all-enabled-project sync.
- The athena overlay resolves `id.username: bbugyi200` and `id.machine_name: athena`, with the legacy top-level key
  removed.
- Every changed repository passes its required checks, no SASE memory/generated-instruction file is edited, and TUI
  responsiveness does not regress.
