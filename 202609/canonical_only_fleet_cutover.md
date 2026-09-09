---
tier: epic
title: Canonical-only SASE across athena, mac, and apollo
goal:
  Migrate every fleet producer and live data store to canonical contracts, remove
  obsolete compatibility behavior from SASE and its coupled plugins, and verify the
  final installation on every machine while preserving historical records and a tested
  rollback.
phases:
  - id: fleet-census
    title: Establish the complete compatibility and fleet inventory
    depends_on: []
    description:
      "fleet-census: Build a versioned contract-by-host census, inspect all projects and
      runtime producers, and establish migration, backup, and acceptance evidence for
      every compatibility removal without destructive operations."
    size: medium
  - id: migration-kit
    title: Build and rehearse the temporary migration tooling
    depends_on:
      - fleet-census
    description:
      "migration-kit: Plan and implement a bounded offline migration kit with dry-run
      manifests, conflict detection, semantic verification, crash recovery, and restore
      rehearsals. Keep shared conversion logic in Rust core and keep the kit out of
      normal runtime startup."
    size: large
  - id: canonical-producers
    title: Migrate configuration, prompts, editor integration, and automation
    depends_on:
      - fleet-census
      - migration-kit
    description:
      "canonical-producers: Plan and update chezmoi sources, project and home prompts,
      generated skills and memory guidance, Neovim producers, scripts, and plugin
      callers to emit canonical forms already supported by the bridge runtime; deploy
      from landed sources on all three machines."
    size: large
  - id: telegram-bridge
    title: Move Telegram to the shared pending-action API
    depends_on:
      - fleet-census
      - migration-kit
      - canonical-producers
    description:
      "telegram-bridge: Implement and test the Telegram adapter to the canonical shared
      pending-action store, preserving existing callback identities, locking, terminal
      states, and transport metadata. Stage the required host/core API and publish a
      wheel usable by the remote machines while old host readers remain available."
    size: medium
  - id: shared-format-bridge
    title: Prepare canonical shared formats and coordinated wire contracts
    depends_on:
      - migration-kit
      - telegram-bridge
    description:
      "shared-format-bridge: Plan and implement parser-aware conversions for Patch,
      plan, gate, prompt, and catalog records, plus the necessary Rust/PyO3/host/plugin
      contract changes. Stage a tested bridge release and temporary sunset flags only
      where a real mixed-format interval exists."
    size: large
  - id: local-state-cutover
    title: Back up and migrate local state on all three machines
    depends_on:
      - canonical-producers
      - migration-kit
    description:
      "local-state-cutover: Execute the rehearsed maintenance runbook on athena, mac,
      and apollo; back up quiescent state, apply the supported import-state purge,
      migrate remaining local stores and roots, remove verified residue without
      following symlinks, and produce per-host receipts before resuming compatible
      writers."
    size: medium
  - id: telegram-cutover
    title: Deploy Telegram and retire the second store
    depends_on:
      - telegram-bridge
      - local-state-cutover
    description:
      "telegram-cutover: Deploy the verified host/core/plugin cohort in mac, athena,
      apollo order, reconcile existing pending actions without losing approvals, restart
      every Telegram writer, and observe at least the 24-hour stale-action interval with
      no retired-store writes before archiving the old store."
    size: medium
  - id: shared-data-cutover
    title: Convert shared records and prove fleet convergence
    depends_on:
      - shared-format-bridge
      - local-state-cutover
      - telegram-cutover
    description:
      "shared-data-cutover: Freeze affected writers across all hosts, deploy and verify
      the bridge cohort, migrate every authoritative mutable store and reachable sidecar
      head, publish and synchronize canonical records, verify semantic equality and
      replayability, and issue the all-host certificate that permits reader deletion."
    size: medium
  - id: canonical-contracts
    title: Remove legacy shared-format and cross-repo API branches
    depends_on:
      - shared-data-cutover
    description:
      "canonical-contracts: Plan and remove the old Patch, plan, gate, catalog, plugin,
      and Rust/Python wire contracts against the certified data set. Remove Telegram
      merging and old bundle resolution only after its cutover receipt; delete temporary
      sunset Off branches and make canonical behavior unconditional."
    size: large
  - id: remove-facades
    title: Delete renamed APIs, command aliases, and TUI test shims
    depends_on:
      - canonical-contracts
    description:
      "remove-facades: Replace all actual facade consumers and monkeypatch targets with
      canonical imports, delete obsolete packages and CLI/UI aliases, remove production
      mock detection, and preserve canonical coverage while renaming stale test
      filenames."
    size: medium
  - id: remove-layout-migrations
    title: Delete completed local migrations and fallback storage roots
    depends_on:
      - remove-facades
    description:
      "remove-layout-migrations: Remove completed migrators, migration commands,
      alternate roots, unsharded fallback readers, duplicate stores, old supervisors,
      and migration-only locks using the fleet receipts. Preserve current cache
      rebuilding, recovery, project identity, and storage invariants."
    size: medium
  - id: remove-config-prompt-compat
    title: Make configuration and prompt parsing canonical-only
    depends_on:
      - remove-layout-migrations
    description:
      "remove-config-prompt-compat: Plan and remove all census-listed legacy
      configuration folding, prompt/workflow/directive syntax, query normalizers,
      obsolete schema entries, and facade re-exports across Rust, Python, and editor
      consumers. Enforce current contracts and update shipped examples, defaults, and
      generated source templates together."
    size: large
  - id: historical-codec
    title: Isolate immutable bead history decoding
    depends_on:
      - remove-config-prompt-compat
    description:
      "historical-codec: Keep the minimal writer-free decoder required by existing
      append-only bead note events inside Rust historical replay, remove the Python
      implementation and live-input normalization, and verify identical full replay,
      note identities, current projections, and history queries. Do not rewrite
      committed event history."
    size: medium
  - id: enforce-and-verify
    title: Close the compatibility inventory and add regression enforcement
    depends_on:
      - historical-codec
    description:
      "enforce-and-verify: Audit every original contract and newly discovered fallback,
      enforce canonical exports and rejected retired inputs in CI, document the precise
      historical exception and rollback runbook, remove temporary migration
      implementation from shipping repositories after archiving it, and pass the
      combined host/core/plugin verification gates."
    size: medium
  - id: final-fleet-rollout
    title: Deploy and validate the canonical-only fleet
    depends_on:
      - enforce-and-verify
    description:
      "final-fleet-rollout: Deploy the exact tested final package and configuration
      cohort to all hosts, regenerate provider skills and completions from landed
      sources, restart every consumer, run canonical and negative smoke tests plus the
      observation window, and publish three passing receipts with retained backups."
    size: medium
proposed_by: bbugyi200.athena.0gk
bead_id: sase-x7
create_time: 2026-09-09 19:52:16
status: wip
---

- **PROMPT:**
  [prompts/202609/canonical_only_fleet_cutover.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/canonical_only_fleet_cutover.md)
- **BEAD:**
  [sase-x7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-x7/README.md)

# Canonical-only fleet cutover

This is an epic because migration tooling, three live machines, multiple repositories,
immutable history, and several deployment barriers cannot safely be handled by one
coding agent. Its authoring size is xlarge; each phase above has its own work size.
Large phases receive a focused planning handoff. Their plans must preserve this epic's
data-safety rules and deployment barriers, rather than defer the central decisions.

The requested deliverable for this turn is this proposal. No source, configuration,
runtime data, or memory content is to be changed before it is proposed and approved. The
scratch plan is the only authored file during planning.

## Context and refreshed evidence

Required context was read through the audited artifact interface:
`research:202609/canonical_only_cutover/canonical_only_cutover.md`. It supplies the
contract inventory and explains why state conversion precedes code deletion. This plan
adopts that ordering and adds executable migration/rollback requirements, explicit
deployment prerequisites, broader prompt/API coverage, and smaller ownership boundaries.

Read-only probes on 2026-09-05, against host checkout `ee358364a`, establish:

| Observation                         | athena, local                                                                                                   | mac, SSH alias `mac`                                                                     | apollo, SSH alias `apollo`                                                            |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| OS / Python                         | Linux / 3.14.7                                                                                                  | Darwin / 3.12.5                                                                          | Linux / 3.12.3                                                                        |
| Host / Rust core                    | 0.17.1+104.gee358364a / 0.32.23                                                                                 | same                                                                                     | same                                                                                  |
| GitHub / research plugins           | 0.2.9 / 0.2.0+10.g6ed87637a                                                                                     | same                                                                                     | same                                                                                  |
| Telegram                            | editable 0.4.9+2.g4b4fa4a92                                                                                     | wheel 0.4.9                                                                              | wheel 0.4.9                                                                           |
| Active services                     | axe and lumberjacks                                                                                             | axe, lumberjacks, ACE                                                                    | axe and lumberjacks                                                                   |
| Project lifecycle list              | actstat, bob-cli, sase                                                                                          | bob-cli, sase                                                                            | bob-cli, sase                                                                         |
| Legacy import purge preview         | 79 artifacts, 809 dismissed bundles/identities, journals/caches/receipts                                        | 9,894 artifacts, 6,501 chats, 9,924 dismissed bundles plus journals/caches/receipts      | 10,076 artifacts, 6,562 chats, 10,076 dismissed bundles plus journals/caches/receipts |
| Telegram old / shared store entries | 16 / 3,090                                                                                                      | absent / 98                                                                              | 8 / 97                                                                                |
| Distinct local residue              | tasks tree: 14,886 files; agent_tags.json: 245,212 bytes; old lock; 39 question files and 7 plan-approval files | research reports one stale home memory note with `type: long`; revalidate before editing | `.xprompts/sshot.yml` and `.xprompts/pick_plan.md`                                    |
| Deprecated model aliases            | 3 doctor findings                                                                                               | 3 doctor findings                                                                        | 3 doctor findings                                                                     |

Athena's archived SASE ProjectSpec has 57 `COMMITS:` and 11 `STITCHES:` sections; its
`sase-core` ProjectSpec has 5 `COMMITS:` sections. The sampled project roots have no
`.gp` files. These are point-in-time measurements, not future apply manifests. Counts of
top-level metadata files in artifacts/dismissed bundles do not prove an unsharded
payload exists; inspect their schema and role before classification.

Two updates to the research matter:

1. The opened chezmoi source now contains `home/dot_config/sase/sase_apollo.yml`.
   Reconcile its contents and deployment; do not create another overlay based on the
   report's earlier absence claim.
2. `main/update_handler.py` distinguishes editable fast-forward/reconciliation from
   managed uv upgrades. Package version equality and one `sase update` invocation are
   not evidence of an atomic fleet transaction or restarted in-memory code. Record exact
   installed code/build identities and verify the complete cohort after update.

Broader doctor probes also report pre-existing warnings for model presets, local xprompt
definitions and SDD content, plus an apollo timezone warning. The remote home-scope init
blocker specifically means no VCS checkout was found at the probe's working directory;
it is not evidence of corrupt project config. Both remotes also report 112 provider
skill files differing from rendered sources and prettier missing from the noninteractive
PATH. Re-run diagnostics per scope, make the render toolchain available, and reconcile
generated drift from landed sources. Capture the baseline rather than claiming the fleet
is already healthy. Resolve findings caused by legacy state in the owning migration
phase; record unrelated findings without changing intended timezone or provider choices
to obtain an artificial green.

The research reports 27,881 version-1 bead events, including 2,829 nonempty string-note
payloads. This planning turn inspected the Rust/Python decoding paths, but did not
recount those sidecar events. The census must refresh that measurement across all
stores. It also must refresh the research's facade/test and retired-skill counts.

## Scope and the historical exception

Remove every operative backward-compatibility writer, reader, normalizer, alias,
adapter, discovery fallback, automatic upgrade, compatibility-only API and test seam
identified by the census, including unmarked cases discovered through callers and
fixtures. Include SASE Python/TUI, the Rust core and bindings, relevant gateway/LSP
contracts, coupled installed plugins, chezmoi, Neovim, generated skills, shell
completion, and runnable project/home content on all three machines.

There is one deliberate exception to literal deletion of all old-format decoding:
preserve the minimal codec needed to interpret existing append-only bead note events.
Put it behind the historical event reader in Rust, with no legacy writer and no route
from current mutation input. Remove the duplicate Python implementation. Replaying
unaltered audit history is a supported operation, not permission for new legacy input.
This exception is part of the approval decision, not a claim that literally every
historical decoder will disappear.

Preserve canonical `issues.jsonl` projections and current cache rebuilds, current
task-bead behavior, intentional shorthand/options, platform portability, optional
capability handling, retries, and error recovery. A variable named `legacy_path`, a
module called a facade, a schema version number, or an `except ImportError` is only a
search lead. For example, `issues.jsonl` is still maintained and checked against events;
plan provenance `COMMITS` and Patch section `COMMITS:` are different contracts.

Do not rewrite append-only events, old git commits, raw chat transcripts, original
archived prompts, or arbitrary prose to achieve a zero-match grep. Migrate mutable
metadata and replay inputs explicitly, preserving original text and provenance. If the
census proves that a second historical codec is necessary, report that concrete conflict
for plan review before broadening this exception or deleting its data.

## Execution rules shared by every phase

- Open every other repo with `/sase_repo`, including the registered projects and
  `gh:sase-org/sase-core`; use only the returned checkout. Do not assume a sibling
  directory exists. Read sidecar artifacts with `sase artifact read`, including when
  inspecting a document to decide its migration. Use the same audited access workflow on
  remote machines. Bulk scans must inventory through the audited repository/store
  interfaces and record the inspected corpus and revision.
- Shared parsing, migration semantics, storage contracts, and validation belong in
  `sase-core/crates/sase_core`; update `crates/sase_core_py` and parity tests. Python
  provides thin orchestration/adapters and TUI presentation. Do not add a Python
  fallback when the new binding is absent.
- Each phase's dependency is satisfied by its stated evidence, including required
  deployment and observation, not merely by merged code. Use host-owned finalizers for
  commits, branches, PRs, sidecar publication, and staged releases. If the epic runner
  cannot stage a required bridge landing, arrange that host-owned checkpoint before
  launching dependent removals; do not collapse two landings into one.
- Retain current runtime readers until their migration certificate is complete for every
  host/project. Prepare removal code in isolated checkouts. Prevent scheduled updaters
  or editable source refreshes from deploying it before the release barrier.
- Producer and bridge implementation phases are ordered because they touch shared core
  exports and plugin callers. Local-state conversion may overlap preparation of
  shared-format tooling only while that tooling remains undeployed; one maintenance
  controller owns production writes and package updates on each host at a time.
- Use `/sase_monitor` for observation waits, long builds, and `just check-full`;
  continuation is mechanical. Do not keep an agent asleep for 24 hours, promise a later
  continuation, or stop the maintenance controller with the services it manages.
- Changed SASE source requires `just install` when the workspace environment is stale,
  then `just check`. The combined tree requires monitor-only `just check-full` with
  TESTING/TESTED. Core requires its root `just check` or `scripts/check.sh`, including
  PyO3; core-only cargo tests are insufficient. Run each plugin's prescribed checks.
- Source templates own generated skills. Deploy only from clean, landed sources using
  canonical `sase skill init` and the prescribed chezmoi workflow. Never patch provider
  SKILL.md copies directly or override source-provenance guards for convenience.
- Use `/sase_memory_write` before actual memory edits and audited reads first. The named
  memory edits below only migrate legacy formats and remove obsolete guidance within the
  requested cleanup. Do not add unrelated governance memory or rewrite accepted decision
  bodies. Document the cutover policy in ordinary project docs.

## 1. fleet-census

Create a versioned machine-readable ledger plus a concise human report. Each row must
identify a specific old contract, canonical replacement, defining source symbols,
writers, readers, data locations, affected hosts/projects/plugins, disposition, migrator
or purge operation, verification query, rollback unit, and owning phase. Disposition is
exactly remove, migrate-then-remove, current behavior, or the named historical
exception. Unexplained entries block closure.

Inventory all project lifecycle states, registered aliases, linked and sidecar repos,
workspaces, home scope, installed distributions/entry points, shell/editor helpers,
provider skill locations, service managers, timers, scheduled updates, and executable
paths. Cross-check registry output against residual on-disk project records; the athena
`sase-core` record already demonstrates that the enabled-project list alone is
insufficient. Cover disabled and dormant projects that can later become active.

Resolve configured `SASE_HOME`, XDG roots, machine selectors, overlays, repo providers,
and custom storage roots. Inspect effective config layers per project, then compare
managed source and rendered output. Avoid dumping credentials or prompt contents into
the report. `sase config layers` has no JSON flag at this baseline; use its real CLI or
thin read-only adapter, and `sase doctor -C config -j` for diagnostic JSON.

Search Python/Rust/Lua/Vim/YAML/JSON source, imports, exports, plugin hooks, serde
aliases/default adapters, alternate filenames, CLI parser registrations, schema
versions, dynamic getattr/signature adaptation, and fixture expectations. Seed with the
category table below; follow actual call sites even when no legacy marker exists. Audit
every source family, rather than treating the report's eight formats as complete.

Inventory remaining import origins using `sase agent names purge-local-state` dry runs
and the local-import doctor check. Separately classify live native records and imported
copies. Hash/count bead event streams through the opened stores and compare full
canonical replay. Do not treat schema version 1 as itself legacy.

Acceptance: all three hosts have fresh census results; inaccessible hosts, repositories,
unknown records, and unclassified active writers are explicit blockers. The report
contains exact read-only reproduction commands and a machine-by-machine maintenance and
restart schedule. No migration or purge has yet run.

## 2. migration-kit

Implement a small, temporary offline migration driver, not a permanent migration
framework. It wraps supported migrators/purge commands and adds only missing
conversions. A shared Rust conversion API may temporarily support the kit; it must not
run automatically during imports, startup, completion, or ordinary reads.

For each operation, a dry run emits host identity, root/repo revision, source path,
destination, source digest, schema, record counts, semantic fingerprints, conflicts,
estimated space, backup location, and intended action. Apply requires matching source
digests and completed prerequisites. Repeated apply is a no-op. A failed or partial run
must be resumable from a durable journal without skipping unconverted records.

Back up each affected config/state root and repo working tree, including dirty and
untracked content, ignored local stores, import registries, chats, artifacts, dismissed
bundles, pending actions, gates, prompt history, procs/logs, and managed provenance.
Preserve modes, symlinks, ownership where applicable, and database consistency. Use
SQLite's backup facilities or quiescent copies including WAL state. Keep backups outside
runtime discovery and sync roots, access-restricted, with checksums and enough free
space. Keep a second durable copy where practical; do not silently rely on git to cover
ignored data. Do not automatically expire or purge these backups.

Conversions must be parser-aware and lossless: preserve canonical precedence,
identities, timestamps, ordering, references, comments where meaningful, and unknown
extension fields. Mixed canonical/old sections merge only when semantic equivalence is
proved; conflicting values require explicit resolution. Never use global replacements
for `task`, `COMMITS`, `changespec`, paths, or prompt syntax. Use atomic same-filesystem
writes, bounded locks, and compare-before-replace checks; do not follow symlinks outside
the inventoried roots. An existing destination or migration marker is not proof of
completion: the current proc migrator can mark complete when the new store already
exists, leaving old records/log conflicts unresolved.

Rehearse using protected copies of representative real data on Linux and macOS plus
synthetic mixed-format, corruption, symlink, conflict, concurrency, disk-full, and
interrupted-write cases. Compare semantic results before/after and test restoration.
Record migration-kit revision/checksum and baseline package/config revisions.

Acceptance: no-op dry runs on canonical fixtures, exact conversion on old fixtures, no
data loss on interruption, conflict refusals, a successful restoration rehearsal, and a
concrete operation manifest for each fleet host. No production data mutated.

## 3. canonical-producers

Update authoritative content before rejecting what it currently generates:

- Chezmoi `home/dot_config/sase/sase.yml`: migrate retired builtin aliases
  `medium_worker`, `small_worker`, `xsmall_worker` to `medium`, `small`, `xsmall`.
  Compare against existing canonical values; preserve intended provider/effort routing
  instead of overwriting a conflict. Inspect every overlay, including the now-managed
  apollo overlay. Migrate any census findings in keymaps, model config, chops, repos,
  memory templates, identity, and external-mirror settings with effective-value checks.
- Project/home config, xprompts, workflows, snippets, query profiles, launch scripts,
  scheduled commands, shell completion, editor mappings, and reusable stashed prompts:
  remove old command/field/syntax producers using the canonical contract table. Confirm
  scope precedence and rendered prompt equality. Leave raw archived user text intact;
  create canonical replay inputs when needed.
- `sase-nvim`: replace old catalog consumers/producers, `.gp` filetype aliases, obsolete
  schema globs, old path/ref syntax, and old completion protocol handling. Preserve
  intentional current browse/fallback UI capabilities unless they depend solely on a
  retired protocol. Coordinate any hook/filetype name changes with chezmoi Lua config
  and its corresponding snippet definitions.
- All installed plugin sources, including `sase-github`, `sase-telegram`, and
  `sase-research-artifacts`: update legacy imports, command examples, hook arguments,
  and generated content. API changes unavailable in the current host belong to the
  shared-format bridge cohort, not an prematurely deployed consumer.
- Source templates under `src/sase/xprompts/skills/`: remove the `sase_changespecs`
  template when its callers have moved to `sase_patches`; synchronize affected skills
  with CLI removals. Preview pruning before deploying, including retired provider
  namespaces identified in the report. Remove only generator-owned retired entries.
- Through the memory workflow, canonicalize any `type: short|long` frontmatter in
  managed notes, including mac's reported `~/sase/memory/sase_beads.md` if still
  present. Reconcile it with its authoritative source rather than deleting unique prose.
  Update affected legacy-only statements in existing `sase/memory/xprompts.md`,
  `generated_skills.md`, and `sase_beads.md` when their corresponding behavior is
  removed; preserve unrelated guidance. Regenerate instructions with `sase memory init`.
  Edit generator templates instead of generated notes where required.

Land source changes through the host, deploy chezmoi using its prescribed
`chezmoi update -a --force` after source publication, and use a reviewed diff to avoid
overwriting unrelated local drift. Verify the actual installed Neovim plugin, shell
completion, and provider skills on every machine, including long-lived editor sessions.

Acceptance: producer inventory contains no active legacy emissions; old data roots
awaiting the explicit migration phases are recorded separately. Model-alias doctor
warnings are gone on all hosts, effective settings and rendered prompts match approved
intent, and updated producers work on the selected bridge cohort.

## 4. telegram-bridge and 7. telegram-cutover

The live blocker is `sase-telegram/src/sase_telegram/pending_actions.py`, whose primary
path is `~/.sase/telegram/pending_actions.json`. The host currently merges it in
`src/sase/notifications/pending_actions.py::_merge_legacy_telegram` as transport
`telegram_legacy`. A path replacement is insufficient: these stores have different
schemas and lifecycle rules.

Move shared lifecycle/storage semantics into Rust where changes are required, expose a
supported thin host API, and port Telegram add/get/remove/list/expiry behavior to it.
Preserve action IDs/prefix collision handling, notification IDs, chat/message IDs,
creation and expiry times, files, cancellation, duplicate callback suppression, and
terminal/handled state. Serialize read-modify-write across host and plugin writers;
never let importing the old store resurrect a completed action or extend its deadline.

Provide a one-shot conversion of outstanding actions after stopping the old writer. Use
stable original callback identifiers so existing Telegram messages remain usable.
Resolve any terminal-state conflicts from authoritative gate/response records. Leave
unknown or irreconcilable records as blockers with a report rather than dropping them.

Stage the compatible host/core API before the dependent plugin. Publish a real wheel for
the remote installs and test that wheel in an isolated runtime, not only the editable
checkout. Deploy mac first, then attended athena, then apollo. On each host verify
loaded package provenance and restart the actual inbound/outbound workers.

Observe no legacy writes for at least 24 hours after the last old writer exits, and
until every migrated old callback is resolved, explicitly canceled, or expired by its
original lifecycle. Use a monitor continuation; freeze unexplained new writes as a
cutover failure. Run transport tests with fixtures and a local fake API; this plan does
not authorize sending unsolicited real Telegram messages or executing real approvals as
smoke tests. Archive the old stores after reconciliation and observation.

Acceptance: all hosts use the shared API, all old actions are accounted for, duplicate
callbacks cannot repeat an operation, and old store mtimes/content remain absent or
unchanged throughout observation. Only then may canonical-contracts delete the merge,
transport tag, `_LEGACY_AWAITING_KEY`, `_LEGACY_EQUAL_TIMESTAMP_ID`, `bundle.legacy`,
and `_resolve_legacy_bundle` branches whose complete caller sets have migrated.

## 5. shared-format-bridge and 8. shared-data-cutover

Use a two-landing rule for real persisted/wire transitions: stage canonical writers and
conversion with old readers still available; migrate and converge all consumers; then
land reader deletion. Stage a coherent tested host/core/plugin cohort. Add temporary
sunset flags with `sase flag new` only for real transition intervals, with enabled
canonical behavior, explicit disabled old behavior, both-state tests, and a per-host
zero-use removal condition. Never hand-add registry entries. Do not change the five
unrelated existing flags merely because they are flags.

The following seeds are mandatory and the census may expand them:

| Contract family                  | Starting points                                                                                        | Required conversion/proof                                                                                                                                                                                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Patch records                    | `ace/patch/storage.py`, parser/serializer, `project_spec_path.py`, Rust `project_spec`                 | `.gp` to `.sase`; `## ChangeSpec`, `COMMITS:`, `CL:` to canonical equivalents; lossless active/archive parsing, mixed-section ordering, STITCH identities and timestamps                                                                                           |
| Plans and artifact references    | `sdd/associations`, `plan_header_*`, `plan_chain.py`, Rust `plan/refs`, `sdd/plan_validate.py`         | Legacy roots/prefixes, links in headers/frontmatter, `@plans`, parent metadata, size defaults and chain suffixes; preserve valid relative/file arguments and git provenance; eliminate Python patch-to-changespec wire translation by changing the actual Rust API |
| Gates and pending approvals      | `notification_gates/model_validation.py`, `paths.py`, runtime schemas                                  | Distinguish gate request v2/v3 from the current pending-action store schema v2; migrate complete bundles and indexes without changing IDs or losing responses/cancellations/approval state                                                                         |
| History and identities           | `history/chat_links.py`, prompt archive/storage, `agent/names`, dismissed bundles                      | Timestamps, names/refs, flat/sharded lookup and metadata, compatibility metadata keys; canonical lookup yields the same referenced record, with original transcript/prompt bytes retained                                                                          |
| Config, content, workflow syntax | `config`, `content_layout`, `memory/notes.py`, `xprompt` and Rust directive parsers                    | Canonical roots, memory types, model/chop/repo/identity shapes, prompt step fields, positional template placeholders, standalone workflow markers, retired directive forms; preserve expansion and source precedence                                               |
| Beads and task metadata          | Rust bead wire/events/read, Python bead adapters, task types                                           | Current mutable task/plan records have explicit valid sizes/types; correct them via supported mutations and appended events, never event-file rewrites; distinguish current `## Types` uses and canonical projections from obsolete parsing aliases                |
| Plugin/gateway/editor wire       | Rust PyO3, LSP/gateway schemas, `core/wire_conversion.py`, `vcs_project_completion.py`, provider hooks | One canonical Patch/proc/stitch field and discriminator set; update producers, consumers, schemas, bindings, and golden/parity tests together; remove stale signature/field negotiation after rollout                                                              |

Before applying, synchronize and record every authoritative sidecar head and dirty
worktree through its owner. Stop all affected producers across the fleet for the
conversion window. Migrate mutable authoritative records once, publish the resulting
revisions through the host, and synchronize every replica. Convert uncommitted local
variants separately; do not replace them with upstream content. Old dormant clones must
either receive canonical content or be barred from launching/writing until their
recorded update step completes. Do not force-rewrite remote history.

For plan sizes, do not silently choose a worker model for ambiguous historical work.
Derive the intended size from existing explicit metadata and approved descriptions;
surface ambiguity for resolution before removing launch defaults. For template changes,
preserve positional indexing/default semantics and verify compiled expansion, including
literal braces, fences, multiline inputs, and nested workflow steps.

Acceptance: canonical parses and before/after semantic fingerprints agree for all
mutable records; all refs and representative historical views still resolve; immutable
corpus digests are unchanged; fresh processes on every host load the bridge cohort; zero
live old-format consumers/producers remain. Publish the migration certificate with
per-host/project results and exact corpus/kit/release revisions. No unreachable host or
unexamined replica may be marked clean.

## 6. local-state-cutover

For each bounded maintenance window, stop new launches and scheduled updates, drain
agents/procs and pending operations that hold affected files, then stop axe/lumberjacks,
Telegram workers, editor/TUI writers, and any relevant systemd/launchd/cron restart
source. Record PIDs, start times, executable paths, and a restart recipe. Check open
handles and path writes; stopping axe alone does not stop already launched workers. Keep
the controlling SSH/maintenance process outside that stop set. Take the final consistent
backup only after quiescence, then revalidate the apply manifest.

Use `sase agent names purge-local-state --apply` as the supported operation for old
import materialization, after comparing the fresh preview to the backup and preserving
referenced originals. Do not manually delete its registries or reproduce its semantics
in a script. Re-run the preview and local-import doctor check to verify it is empty;
verify native records and unrelated data are unchanged. The approved scope is the
previewed imported copies, not arbitrary artifact retention/trash purging.

Athena needs record-by-record inspection of the old tasks/log tree, tribe/tag store,
PR-mirror `checks` state, obsolete code-swap lock, and question/plan-approval
directories. Check canonical counterparts and active references before archiving
leftovers. Remove a transition symlink itself, never the canonical directory it targets.
An old lock inode must have no holder before unlinking. Preserve unique logs and
historical results through canonical references or the explicit backup; a filename count
is not proof that they are duplicates.

Apollo's old xprompt files require semantic comparison with canonical home files and
source ownership. Reconcile unique content, then archive the obsolete root outside
discovery. Mac's memory drift uses the audited memory/source regeneration process. All
hosts need generated retired-skill pruning and scans for alternate workspace,
home/project config, memory, prompt, notification, and artifact paths.

Restart the compatible cohort after each successful local conversion; do not keep the
whole fleet down during code development or the Telegram observation window. On failed
verification, keep affected writers stopped, restore that host's exact state/config and
package cohort, verify restoration, then resume. Shared data rollback instead uses the
coordinated procedure below.

Acceptance: per-host backup/restore evidence and operation receipts; zero applicable
local migration findings; clean supported purge previews; no open handles or new writes
to archived roots after restart. Unrelated records, unique history, identity, and
canonical settings remain intact.

## 9–12. Remove compatibility code against certified state

The deletion phases are intentionally ordered to avoid concurrent edits to shared
registries, parsers, and core bindings. Each removal updates its ledger row with source
diff, cited migration receipt, canonical positive tests, and retired-input tests.

**canonical-contracts** removes certified old format readers, normalizers, alternate
wire fields, plugin signature adaptation, and the Telegram second-store reader. Make the
canonical Rust wire contract authoritative; update and pin the corresponding binding,
host, plugin, LSP/gateway schema and fixture versions together. Fail clearly on an
incompatible installed core rather than recovering with getattr/signature introspection.
Remove each temporary sunset flag's Off branch and registry entry and close its flag
bead in the same host-owned change. Retain no switch to legacy behavior.

**remove-facades** deletes the obsolete `ace/changespec` and `core/changespec` packages,
TUI changespec mirrors, `main/parser_changespec.py`, changespec/task/vcs handlers and
render/parser aliases, and all other ledger-proven compatibility facades. Replace actual
mobile/TUI/import consumers first. Delete top-level aliases `changespec`, `vcs`, `task`,
`artifact-file`, nested `prompt sdd`, legacy Patch long options, duplicate namespace
fields, and retired init/config/workspace options discovered by the census. Keep
required canonical short options while deleting obsolete long spellings. Update help,
dispatch, docs, default_config.yml, schemas, completions and skill sources.

Delete `_compat_loader` and `_is_mock` from the Patch load path and other production
monkeypatch detection; tests must patch the canonical dependency. Remove legacy TUI
tab/subtab/key aliases only after saved state and keymaps are canonical. Preserve
functional coverage in the stale-named tests: separate mechanical file/docstring renames
from behavior deletion in reviewable host-owned changes, and delete only assertions
whose purpose was the removed compatibility contract.

**remove-layout-migrations** removes the completed ProjectSpec extension, prompt
history, proc/task, memory-root, agent-name, dismissed bundle, workspace, PR-mirror,
tribe/tag and lock migrators, their CLI entry points, startup calls, markers and tests
that exist solely to trigger them. Remove legacy flat-history searches and
`include_legacy` parameters/defaults, alternative content roots, old supervisors, and
registry/SQLite compatibility stores after their census disposition is proved. Retain
authoritative projections and current-schema cache rebuild paths. Remove Python mirrors
of core migration behavior rather than merely leaving dead adapters.

**remove-config-prompt-compat** eliminates deprecated top-level folds and old schema
alternatives in both Python and Rust, including repos, identity, memory templates, model
aliases, external mirror, chops lists, old keybindings, and source discovery. Remove
legacy positional `{N}`/`{N:default}` template substitution, workflow `prompt` as an
alias for `agent`, `wraps_all`, implicit standalone workflow dispatch, old fanout forms,
obsolete auto modes, query/catalog aliases, and other census findings after their
producer and saved-input migrations. Preserve intentional current shorthand. Delete
compatibility-only private exports and monkeypatch forwarding in xprompt facades while
retaining legitimate canonical public module boundaries.

Unknown/deleted CLI options, config values, typed references, wire fields, and parsed
directive forms should fail current-contract validation instead of silently folding or
being dropped. Use current grammar/schema validation rather than preserving a large
retired-token-to-replacement map. Arbitrary prompt prose and nonexistent old fallback
directories must not acquire new surprising parsing behavior.

Acceptance for each phase: its applicable ledger rows are closed, canonical CLI/UI
behavior still works, targeted old inputs are rejected or old roots are not searched,
and `just check` plus core/plugin checks pass. Changes affecting TUI presentation run
the dedicated PNG suite and inspect diffs; removal of mock seams must not regress
asynchronous loading, refresh coalescing, or first paint.

## 13. historical-codec

The research exception currently spans Rust `bead/wire.rs::parse_legacy_note_blob`,
`LEGACY_NOTE_ID_PREFIX`, rekeying helpers, and Python `bead/note_codec.py`/DB decoding.
Current `IssueWire` deserialization accepts string notes generally, so moving a function
to a file named archive is not enough to establish the boundary.

Separate historical event payload decoding from current mutation/request validation.
Move only the necessary blob parsing and stable event-derived note-ID handling into a
Rust historical replay module. Current mutation APIs and new projection writes accept
and emit canonical structured notes only. A normal free-text argument to the current
`sase bead note` command remains supported and creates a canonical note record.

Remove the Python blob parser and old SQLite/JSONL fallback once those materialized
stores have been converted/rebuilt from authoritative records. Retain thin bindings only
if a real historical caller needs them. Preserve version-1 event semantics and optional
fields belonging to that schema; do not invent a signed checkpoint system or rewrite
thousands of events solely to remove this codec.

Acceptance: full replay before/after agrees across every project's events, including
note IDs, attribution, timestamps, edited/removed notes, lost-note history, task
lifecycle, dependencies, refs, and projection drift checks. Original event bytes and git
history are unchanged. Current APIs reject old note-blob wire input. The sole retained
old-format behavior is reachable only through the recorded historical reader.

## 14. enforce-and-verify

Audit the complete original inventory again, including newly exposed dependencies. No
row remains migrate-then-remove, no temporary sunset Off branch survives, and no old
reader is kept merely because a test still patches it. Classify remaining search matches
explicitly as current behavior or the narrow historical decoder.

Add `tools/check_backcompat` to the normal verification lane with focused structural
rules: retired imports/exports, CLI/parser aliases, known root fallbacks, schema/wire
aliases, and production test-mock dispatch must not return. Supplement with marker
discovery for review and exact symbol-level registrations for legitimate exceptions or
future temporary sunset flags. Do not use a repo-wide word ban, a directory-wide
allowlist, or rename old helpers to hide them. Test that the lint detects a reintroduced
branch and permits the documented archive codec/current resilience paths.

Document the fleet certificate schema, retirement matrix, historical boundary, canonical
authoring rules, rollout receipts and restore instructions in ordinary project
documentation such as `docs/canonical_only_cutover.md`. Do not create new memory
decision records as incidental governance work. Keep relevant existing memory and skill
examples synchronized with the actual removals as specified above.

Archive the verified migration kit and its exact baseline dependency lock/build
artifacts through SASE's artifact workflow outside shipping repositories. Remove
temporary converters, bridge-only APIs, commands, migration-only tests/fixtures, and
special-case detectors from the runtime and repository tools once all receipts pass.
Retain canonical validation, compact negative regressions, replay fixtures, the
enforcement lint, and externally archived recovery tooling. A future ordinary startup
must not perform a historical upgrade or search a retired root.

Run the host combined `just check-full` via monitor, core root `just check` including
binding/contract parity, and affected plugin/editor suites. Cover Python 3.12 and the
athena 3.14 runtime, macOS and Linux paths/symlinks/locking, current command defaults,
config precedence, fresh-home initialization, artifact resolution, workflow expansion,
gate duplicate handling, proc lifecycle, current cache rebuilding, and bead replay. Run
PNG visual checks for intentional UI changes; do not discard unrelated goldens.

Acceptance: exact final build identities and passing reports are recorded for all
affected repos, the retirement matrix is closed, the shipping trees contain no temporary
compatibility implementation, and final deployment can use a tested cohort.

## 15. final-fleet-rollout and rollback

Deploy the tested cohort in mac, attended athena, apollo order. Canary means a
controlled stop/update/verify/restart window, not an assumption that mac is idle. Freeze
background source updates while staging. Record exact Git SHAs, wheel/build hashes,
loaded module locations, config/template revision, editor plugin revision, and
source-manifest provenance. `sase version` equality alone is insufficient.

Deploy generated skills and shell completion from the clean landed source; apply chezmoi
on all hosts and verify every provider directory. Restart axe/lumberjacks, Telegram,
ACE, editors/LSP, and relevant service managers so no old code remains in memory.
Re-enable only the producers whose cohort and census pass. Retain an SSH recovery path
even if SASE cannot start.

On each host run canonical CLI and isolated workflow/agent smoke tests, the relevant
config doctor checks with strict warning handling, artifact and history resolution,
project/proc queries, editor completion, gate lifecycle fixtures, and representative
retired-input probes. Require no new diagnostics and no unresolved compatibility
findings; retain an explicit disposition for unrelated baseline warnings. Use temporary
test state to avoid creating real approvals or side effects. Compare native data
counts/fingerprints and event hashes to the migration receipts.

Observe for at least 24 hours and at least one full cycle of every inventoried periodic
producer, whichever is longer; exercise a safe dry run for rarely scheduled producers.
Use a monitor continuation and collect absent/unchanged retired-path hashes, writer
process/build evidence, errors, and per-host health. Any new legacy write reopens its
ledger row and blocks completion; do not solve it by restoring a silent fallback.

Rollback before reader deletion uses the retained bridge cohort. After final cutover,
stop affected writers first. Restore the exact prior code/config/data as a coordinated
unit, with host and shared-sidecar revisions treated separately. For shared stores,
coordinate every replica and preserve new post-cutover records in a recovery copy; never
blindly roll a single host backward against newer shared data or force-reset published
history. Use the rehearsed reverse conversion/forward-fix route where live data has
advanced, then verify refs, pending actions and event/projection equality before
restarting. Keep backup locations and commands usable outside SASE itself.

The epic is complete only when all three final receipts pass, no known legacy producer
or live parser path remains, all referenced data is accounted for, the historical
exception is enforced, combined checks and the observation window pass, and recovery
artifacts remain available. A host being offline, a plugin wheel not being published, or
an unfinished observation window is outstanding work, not a successful cutover.
