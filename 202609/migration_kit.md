---
tier: epic
status: done
title: Temporary offline migration kit for the canonical-only cutover
goal:
  Deliver a bounded, explicitly invoked migration kit that can back up, dry-run, apply,
  resume, verify, and restore the fleet's canonical-only state conversions, prove it on
  protected copies of real Linux and macOS data, and hand every later phase of the
  canonical-only cutover a per-host operation manifest -- without mutating production
  data or adding any automatic runtime path.
phases:
  - id: kit-contract
    title: Land the migration wire contract and bindings in the Rust core
    depends_on: []
    description:
      "kit-contract: Add a temporary `migration` module to sase_core (manifest and
      journal wire types, tree digests, semantic fingerprints, residue classification,
      procs reconciliation), expose it through sase_core_py, land it, publish a core
      release exposing the new bindings, then bump the host's sase-core-rs floor and
      ratchet the pinned revision."
    size: medium
  - id: kit-backup
    title: Build the backup and restore engine and the host drain inventory
    depends_on:
      - kit-contract
    description:
      "kit-backup: Introduce the `sase migrate` command group with `backup` and
      `restore`, implement quiescent, checksummed, SQLite-consistent backups written
      outside every runtime root, implement staged restore with verification, and
      produce the missing fleet inventory of installed distributions, entry points,
      shell completions, timers, and scheduled updaters (census gap G3)."
    size: medium
  - id: kit-driver
    title: Build the dry-run, apply, journal, and operation catalog
    depends_on:
      - kit-backup
    description:
      "kit-driver: Add `sase migrate list|plan|resume|run|status|verify`, the operation
      catalog with the four shipped operations, digest-gated apply, idempotent re-apply,
      durable journal-based resume, conflict refusals, bounded locks, atomic
      same-filesystem writes, symlink containment, and the startup-isolation guard."
    size: medium
  - id: kit-rehearsal
    title: Rehearse the kit on real data across Linux and macOS
    depends_on:
      - kit-driver
    description:
      "kit-rehearsal: Run the synthetic edge-case matrix and protected real-data
      rehearsals on athena and mac from isolated checkouts, prove restoration and
      interrupted-run resume, and publish the per-host operation manifests plus the
      acceptance receipt that later cutover phases consume."
    size: medium
proposed_by: bbugyi200.athena.sase-x7.2
parent_bead: sase-x7.2
bead_id: sase-x7.2.1
create_time: 2026-09-09 19:52:29
---

- **PROMPT:**
  [prompts/202609/migration_kit.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/migration_kit.md)
- **PARENT:**
  [202609/canonical_only_fleet_cutover.md](https://github.com/sase-org/sase--plans/blob/main/202609/canonical_only_fleet_cutover.md)
- **BEAD:**
  [sase-x7.2.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-x7/sase-x7.2.1.md)

# Temporary offline migration kit

This is the focused plan for phase `migration-kit` of the canonical-only fleet cutover.
The parent epic requires "a small, temporary offline migration driver, not a permanent
migration framework", and four downstream phases (`canonical-producers`,
`telegram-bridge`, `shared-format-bridge`, `local-state-cutover`) depend on it.

## Why this is an epic rather than a tale

Three hard barriers make this multi-agent work, not one direct implementation:

1. **Two repositories and two landings.** The shared conversion and validation semantics
   belong in `sase-core`; the driver and backup orchestration belong in the host repo.
   The host cannot call a new `sase_core_rs` binding until a core release exposing it is
   published and `pyproject.toml`'s `sase-core-rs` floor is raised (today
   `>=0.32.19,<0.33.0`) and `sase-core-revision.txt` is ratcheted.
   `tools/probe_core_floor` is advisory in `just check`, but
   `tools/check_sase_core_rs_bindings` fails CI's floor-smoke venv. The parent epic
   forbids collapsing two landings into one.
2. **An apply path must never exist before a proven backup path.** Sequencing the backup
   engine ahead of the driver is a safety property, not scheduling convenience.
3. **The macOS leg is independently risky.** Rehearsing on `mac` means an isolated
   clone, an isolated `uv` venv, and a from-source Rust build on Darwin, run over
   best-effort SSH to a laptop that is offline unless it is open. It cannot be
   guaranteed to fit inside the turn that writes the code.

The phases below are all `medium` and are deliberately serial. Nothing here widens the
parent epic's scope: the kit's surface is fixed by the "What the kit is not" section and
is deleted in its entirety by `enforce-and-verify`.

## Context and evidence

Read through the audited artifact interface this turn:

- `file:explicit:5d205f8bf16e8de09e033937` -- the `fleet-census` ledger (Tier A-F
  contracts, dispositions, per-host data locations, findings F1-F6, gaps G1-G5).
- `file:explicit:50875b31c45c9d504c5dce72` -- the `fleet-census` narrative report and
  its machine-by-machine maintenance and restart schedule.

Facts this plan depends on, all from that census plus read-only probes taken while
planning:

| Fact                     | athena                                                                                                                       | mac                                                       | apollo                                             |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------- |
| OS / Python              | Linux / 3.14.7                                                                                                               | Darwin 26.5 / 3.12.5                                      | Linux / 3.12.3                                     |
| sase install             | editable                                                                                                                     | editable from a git checkout, launched via a uv tool venv | editable                                           |
| Toolchain                | cargo, rustc, uv, git, rsync                                                                                                 | cargo, rustc, uv, git, rsync                              | cargo, rustc, uv, git, rsync                       |
| `~/.sase` size           | 17G (largest children: projects 7.5G, cache 3.7G, artifacts 1.6G, procs 1.4G)                                                | 2.2G                                                      | 2.7G                                               |
| Free space on `$HOME`    | 360G                                                                                                                         | 62G                                                       | 53G                                                |
| Tier B residue           | `~/.sase/tasks/`, `agent_tags.json` (245K), `locks/code-swap.lock` + `code-swap-v2.lock`, `user_question/`, `plan_approval/` | none                                                      | `~/.xprompts/{pick_plan.md,sshot.yml}` (confirmed) |
| Import-leg purge preview | 79 artifacts / 809 dismissed bundles                                                                                         | 9,894 artifacts / 6,501 chats / 9,924 bundles             | 10,076 artifacts / 6,562 chats / 10,076 bundles    |

Live SQLite stores exist under `~/.sase` with active WAL sidecars
(`agent_artifact_index.sqlite` plus `-wal`/`-shm`, `chats_catalog.sqlite`,
`dismissed_bundles/index.sqlite`, `telemetry/metrics.sqlite`). Any backup that copies
those files byte-wise while a writer is live is unusable; the engine must use SQLite's
own backup facility.

athena has no systemd user timers and no cron entries that run SASE. That is a
point-in-time observation on one host only, not the G3 inventory, which `kit-backup`
still owes for all three hosts.

## Scope: what the kit is

A single, explicitly invoked command group, `sase migrate`, with:

- an **operation catalog** of exactly four shipped operations (below), each declaring
  its roots, preconditions, backup requirement, apply action, verification query, and
  rollback unit;
- a **dry run** that mutates nothing and emits a manifest carrying host identity, root
  and repo revisions, source paths, destinations, source digests, schema versions,
  record counts, semantic fingerprints, detected conflicts, estimated space, backup
  location, and the intended action;
- an **apply** path gated on those source digests still matching, on prerequisites being
  complete, and on a verified backup existing;
- a **durable journal** that makes a failed or partial run resumable without skipping
  unconverted records, and makes a repeated apply a no-op;
- a **backup and restore engine** covering config and state roots and repo working
  trees, including dirty and untracked content, ignored local stores, import registries,
  chats, artifacts, dismissed bundles, pending actions, gates, prompt history, procs and
  logs, and managed provenance;
- a **rehearsal corpus** proving the above on synthetic edge cases and on protected
  copies of representative real data on Linux and macOS.

### The four shipped operations

The census dispositions decide this set. The kit adds only what has no supported
migrator today.

1. `import-purge` -- wraps the supported `sase agent names purge-local-state` (dry run
   by default, `--apply` to execute) behind a mandatory verified backup of
   `~/.sase/{chats,artifacts,dismissed_bundles,agents_sync,projects}`, and re-runs the
   preview plus the local-import doctor check as its verification query. Census row
   `tier-c-import-leg-residue`. Owner: `local-state-cutover`.
2. `procs-residue` -- the one genuinely missing conversion.
   `src/sase/procs/_migration.py` can mark itself complete when the new store already
   exists, leaving legacy rows and log conflicts unresolved; a marker is therefore not
   proof. This operation parses the residual `~/.sase/tasks/tasks.jsonl` rows, matches
   each to a canonical proc record, classifies the `logs/` symlink state, and
   **refuses** if any legacy row has no canonical counterpart or disagrees with one.
   Only a fully matched residue may be archived. Census row
   `tier-b-finished-migrations`.
3. `state-residue` -- a table-driven archive of declared inert residue:
   `~/.sase/agent_tags.json`, `~/.sase/user_question/`, `~/.sase/plan_approval/`, and
   apollo's `~/.xprompts/`. Every entry names its canonical counterpart and a
   precondition query; the operation refuses when the counterpart is absent or when any
   live agent, notification, or gate record still references the residue. Census row
   `tier-b-finished-migrations`.
4. `lock-residue` -- read-only classification of `~/.sase/locks/code-swap.lock` and the
   uncatalogued `~/.sase/locks/code-swap-v2.lock` (census finding F3): holder, mtime,
   and the code path that writes each. It **refuses to archive any lock the current code
   still writes** and records its classification into the manifest so
   `local-state-cutover` inherits a decided answer rather than a name-similarity guess.

Everything else the census lists is deliberately excluded:

- The `*_worker` model-alias migration is one chezmoi edit plus `chezmoi apply` x3
  (census report, maintenance schedule item 3). A parser-aware conversion engine for a
  single hand edit would be exactly the permanent framework the parent epic forbids.
  `canonical-producers` owns it.
- The `type: short|long` memory note on mac is one file, and memory files are edited
  through the memory workflow, not by a migration driver. `canonical-producers` owns it.
- Patch `COMMITS:`/`STITCHES:` headings, plan path prefixes, gate schema, plan chain
  suffixes, and chat-link timestamps are Tier E. `shared-format-bridge` owns those
  conversions and registers them into this catalog; the kit ships the registration
  point, not the conversions.
- Telegram pending actions are Tier D. `telegram-bridge` and `telegram-cutover` own the
  conversion; the kit owes them only backup coverage of `~/.sase/telegram/` and
  `~/.sase/pending_actions/`.

## Scope: what the kit is not

- **Not automatic.** Nothing in `sase.migration_kit` may be imported or executed during
  interpreter startup, plugin discovery, completion, agent launch, import, finalization,
  or any ordinary read. `kit-driver` lands a subprocess import guard modelled on
  `tests/test_chop_import_budget.py` asserting `sase.migration_kit` is absent from
  `sys.modules` after `import sase` and after `sase --help`.
- **Not extensible by third parties.** The operation catalog is a module-level tuple in
  `src/sase/migration_kit/operations/`. No entry points, no plugin SPI, no dynamic
  discovery.
- **Not a retention system.** The kit never expires, prunes, or deletes a backup, and
  `restore` never consumes one.
- **Not a scheduler.** No timer, no daemon, no axe integration, no `sase update` hook.
- **Not permanent.** `enforce-and-verify` (`sase-x7.14`) archives and then deletes the
  entire `sase migrate` group, `src/sase/migration_kit/`, the core `migration` module,
  and its bindings. Every new module carries a header comment naming that owning bead so
  the deletion sweep cannot miss it.

### Feature flag decision

**No feature flag.** A `sunset` flag keeps an old branch reachable while callers
migrate; a `beta` flag hides an unfinished feature that a landed phase would otherwise
expose. The kit is neither: it is additive, has no automatic route into existing
behavior, and changes no existing code path. Containment comes from the command living
only in `sase --full-help` (do not add it to `_COMPACT_ROOT_COMMANDS` in
`src/sase/main/parser.py`) and from being invoked only by the cutover phases. Removal is
already owned by a phase bead, so a flag bead would duplicate it. Do not create one.

## Execution rules for every phase

These inherit from the parent epic and are not negotiable here.

- Open `sase-core` only with `sase repo open sase-core -r "<why>"` and use the printed
  path. Read census artifacts only with `sase artifact read <ref> "<why>"`.
- Shared parsing, migration semantics, storage contracts, and validation go in
  `sase-core/crates/sase_core`; bindings and parity tests in `crates/sase_core_py`.
  Python is thin orchestration. Do not add a Python fallback when the binding is absent;
  call through `require_rust_binding("<literal>")` so
  `tools/check_sase_core_rs_bindings` can see the name statically.
- **No production data may be mutated by any phase of this child epic.** The only
  permitted real-host writes are (a) backups, which are read-only with respect to their
  sources and are written outside every runtime root, and (b) writes inside scratch
  copies the phase created. Every catalog operation is exercised in `--apply` mode only
  against scratch copies.
- Changed host source: `just install` when the workspace environment is stale, then
  `just check`. Run `just check-full` only through `/sase_monitor` with the
  `TESTING`/`TESTED` status pair. Changed core: `./scripts/check.sh all` (or
  `just check`) in the core checkout, which includes the PyO3 crate; core-only
  `cargo test` is not sufficient. Update `CHANGELOG.md`; `just check` lints it.
- Use `/sase_monitor` for long builds, cross-machine rehearsals, and observation waits.
  Do not promise a later continuation.
- Symbols a later phase will consume may use
  `--epic-symbol <this epic's bead id>(<symbol>)` in the `Justfile` symvision
  invocation. Each phase **must** run `sase bead epic-symbols <its own phase bead>`
  before closing and clear or re-key every leftover entry; stale entries turn unrelated
  agents' `just check` red.
- Phase workers create no beads. Record discovered work with
  `sase bead note <phase bead> 'PROPOSED FOLLOW-UP: <summary -- detail>'`.
- Completion is host-owned: submit a declaration, never create commits, branches, or PRs
  directly.

## Locked design decisions

These are decided here so no phase re-litigates them.

### Command surface

`sase migrate` subcommands, alphabetical, every long option with a short alias, no
required options (a required value is a positional):

| Subcommand            | Behavior                                                                                     | Phase      |
| --------------------- | -------------------------------------------------------------------------------------------- | ---------- |
| `backup <root>`       | Capture a verified backup of one declared root. Dry run unless `-a/--apply`.                 | kit-backup |
| `list`                | Print the operation catalog with per-host applicability. Bare `sase migrate` delegates here. | kit-driver |
| `plan <operation>`    | Dry run. Emit the manifest, mutate nothing, print the manifest path.                         | kit-driver |
| `restore <backup-id>` | Verify checksums, restore into a staging path; swap into place only with `-a/--apply`.       | kit-backup |
| `resume <run-id>`     | Continue an interrupted run from its journal. Dry run unless `-a/--apply`.                   | kit-driver |
| `run <manifest>`      | Execute a manifest. Dry run unless `-a/--apply`.                                             | kit-driver |
| `status`              | Show runs, journal state, and resumability.                                                  | kit-driver |
| `verify <run-id>`     | Re-check post-conditions and semantic fingerprints against the manifest.                     | kit-driver |

`kit-backup` introduces the group with `backup` and `restore` only; bare `sase migrate`
prints group help until `kit-driver` adds the `list` child, at which point the central
`_default_list_subcommands()` convention in `src/sase/main/parser.py` takes over. Follow
`sase/memory/cli_rules.md`: sorted subcommands and options, `-a/--apply` matching the
existing `sase agent names purge-local-state` idiom (dry run is always the default),
`-j/--json` on every reporting subcommand, and colored human output.

Register through `src/sase/main/parser_migrate.py` and
`src/sase/main/migrate_handler.py` added to `src/sase/main/parser_full_registrars.py`,
with the handler importing `sase.migration_kit` lazily inside the dispatch function.

### Where the kit writes

Everything the kit produces lives **outside** `sase_home()` and outside any repo:

```
$HOME/cutover-backups/            # override with SASE_CUTOVER_BACKUP_DIR, mode 0700
  backups/<backup-id>/            # MANIFEST.json, SHA256SUMS, provenance.json, payload
  runs/<run-id>/                  # manifest.json, journal.jsonl, receipt.json
```

The default is deliberately not under `~/.sase`, `~/.local/state/sase`, or `~/sase`, and
shares no string prefix with them, so no discovery glob and no purge operation can reach
it. `kit-backup` must assert this with a test and record the check in the manifest, and
must confirm on mac that the chosen root is not inside an iCloud-synced directory. The
kit writes nothing under `~/.sase` except while applying an operation; `kit-driver`
lands a test for that invariant.

### Rust core contract (`kit-contract`)

New module `crates/sase_core/src/migration/` (`mod.rs`, `manifest.rs`, `journal.rs`,
`digest.rs`, `residue.rs`), registered in `lib.rs`, with a header comment naming
`sase-x7.14` as its deletion owner:

- `MIGRATION_WIRE_SCHEMA_VERSION: u32 = 1`.
- `MigrationManifest` / `MigrationOperationEntry` / `MigrationBackupRecord` serde types
  carrying every dry-run field listed above, each with a `#[serde(flatten)]`
  `BTreeMap<String, Value>` tail so unknown extension fields survive a read/write cycle.
- `MigrationJournalRecord` plus the state machine
  `planned -> backed_up -> applying -> applied -> verified`, with `failed` and `refused`
  terminals, and `plan_next_step()` replaying an append-only journal to decide the
  resume point. Replay recomputes source digests and returns a refusal when any digest
  moved since the manifest was written.
- `tree_digest()` over `(relative path, mode, symlink target, sha256 of content)` sorted
  by path, using the existing `sha2` workspace dependency, and `fingerprint()` over a
  normalized record stream so host and later phases agree on semantic equality.
- `residue::classify()` returning
  `Archive | AlreadyDone | RefuseMissingCounterpart | RefuseLiveReference` from declared
  residue entries plus observed filesystem facts.
- `procs::reconcile_plan()` returning matched, unmatched, and conflicting sets from
  legacy rows and canonical proc ids.

Bindings in `crates/sase_core_py/src/lib.rs` named `migration_*` to match the existing
`telemetry_*` convention, with binding-parity tests. Reuse `store_lock.rs` for bounded
locking rather than adding a second lock implementation.

### Backup engine (`kit-backup`)

- Refuse unless measured source size x 1.15 fits in the destination filesystem.
- For every `*.sqlite`/`*.db` under a root, copy via Python's
  `sqlite3.Connection.backup()` and record `PRAGMA integrity_check`; store that copy
  instead of the live file and its `-wal`/`-shm` sidecars. Files that are not valid
  SQLite are copied verbatim. Quiesce first where a writer can be stopped; where it
  cannot, record that the copy was taken hot.
- Preserve modes, mtimes, and symlinks without dereferencing; record uid/gid and report
  ownership deltas at restore rather than requiring root.
- Include dirty and untracked working-tree content and ignored local stores; never rely
  on git to cover ignored data.
- Write `SHA256SUMS` for every stored member plus `provenance.json` recording host
  identity, `sase version` output for host/core/plugins, root revisions, the kit
  revision and checksum, and the invoking run id.
- `-s/--secondary DIR` writes a second durable copy; when omitted the manifest records
  that no secondary exists, and the per-host cutover manifests produced by
  `kit-rehearsal` must name one.
- `restore` verifies `SHA256SUMS` before touching anything, restores to a staging path
  by default, and reports the diff against the live root; `-a/--apply` performs the swap
  and never deletes the backup.

### Conflict, atomicity, and idempotence rules (`kit-driver`)

- Apply requires: every source digest in the manifest still matches, every declared
  prerequisite is complete, and a verified backup record exists for each affected root.
  Any mismatch is a refusal with a precise reason, not a warning.
- A destination that exists and differs is a conflict: refuse and print the difference.
  Merge only when semantic equivalence is proved by `fingerprint()`.
- Resolve every path with `realpath` and refuse when it escapes its declared root. Never
  follow a symlink out of the inventoried roots.
- Take bounded locks via the core helper; on timeout refuse. Never break a lock. Refuse
  when a live writer holds the root.
- Writes are same-filesystem temp plus `os.replace`, then `fsync` on the file and its
  parent directory. A cross-device destination is a refusal, not a copy-then-delete.
- Residue is archived into the backup tree and only then removed, with a receipt; the
  delete happens only after the copy is verified by checksum.
- Re-running an applied manifest exits 0 as a no-op, matching on run id, digests, and
  post-state fingerprint.
- Never use a global find-and-replace for `task`, `COMMITS`, `changespec`, paths, or
  prompt syntax. Every conversion is structural and revalidated by re-parse.

## Phases

### 1. kit-contract

Land the core module, bindings, and tests described above; run the core checkout's
`./scripts/check.sh all`. Then close the deployment barrier the host repo depends on:
land core, obtain a published `sase-core-rs` version that exposes the new `migration_*`
bindings, raise the floor in `pyproject.toml`, and ratchet `sase-core-revision.txt` with
`just ratchet-core-revision`, then `just install` and `just check` in the host repo.

If the release cannot be published within this phase, **stop and hand the barrier to the
epic runner**. Do not import an unpublished binding, do not vendor a Python
reimplementation, and do not weaken `tools/check_sase_core_rs_bindings`.

Acceptance: core gates pass; the new bindings are importable from a published wheel at
the declared floor; the pin is ratcheted; `just check` passes in the host repo with no
new advisory from `tools/probe_core_floor`; a note on the phase bead records the exact
core version and revision every later phase must match.

### 2. kit-backup

Deliver `src/sase/migration_kit/backup.py` and `restore.py`, the `sase migrate` group
with `backup` and `restore`, and their tests. Then produce the fleet inventory the
census left open as gap **G3**, on all three hosts, read-only: installed distributions
and their exact versions and code directories, console-script entry points, shell
completion registrations, systemd user units and timers, launchd agents on mac, cron
entries, and every scheduled or automatic updater that could deploy code during a
maintenance window. athena's timer and cron surface was probed clean while planning;
re-derive it properly and cover mac and apollo. Toolchain presence is already
established for all three hosts in the fact table above, so the gap is the scheduler,
entry-point, and completion surface.

Publish the inventory with
`sase artifact create -p <file> -l "..." --bead <phase bead>`. It defines the drain list
`local-state-cutover` executes and the "prevent scheduled updaters from deploying
removal code" rule the parent epic requires.

Acceptance: `backup` and `restore` unit and integration tests pass, including SQLite WAL
consistency, symlink and mode preservation, checksum verification, free-space refusal,
and a staged restore that reports ownership deltas; a real backup of a scratch copy of
`~/.sase` on athena completes with a verified checksum manifest; the G3 inventory
artifact is published and attached; `just check` passes.

### 3. kit-driver

Deliver `src/sase/migration_kit/` (catalog, manifest, journal, locks, atomic writes,
render) and `operations/` with the four shipped operations, the remaining `sase migrate`
subcommands, the startup-isolation guard, and the write-containment invariant test.
Implement every conflict, atomicity, and idempotence rule above. Every operation
implements dry run, apply, and verify; `lock-residue` implements dry run and verify
only, because it is classification and refuses to archive.

Acceptance: `sase migrate plan` on canonical fixtures produces a no-op manifest;
`sase migrate run --apply` on old fixtures converts exactly and matches the expected
fingerprint; a run killed mid-apply loses no records and `sase migrate resume` finishes
it without skipping any; conflicting fixtures are refused with actionable text; a second
apply of the same manifest is a no-op; `sase.migration_kit` is absent from `sys.modules`
after `import sase` and `sase --help`; the kit writes nothing under `~/.sase` outside an
apply; `just check` passes and `just check-full` passes through `/sase_monitor`.

### 4. kit-rehearsal

Run the full acceptance matrix and publish the evidence later phases consume.

Synthetic matrix, as tests in the repo: canonical no-op, exact conversion, mixed
canonical/old sections, corrupted source, symlink escaping the declared root,
destination conflict, concurrent lock holder, interrupted write, and disk-full. Simulate
`ENOSPC` through an injected writer seam, and additionally against a small real bounded
filesystem on Linux where one can be created, skipping that variant when it cannot.

Real-data rehearsals, on protected copies only:

- **athena (Linux):** snapshot the residue and import-leg roots into a scratch root,
  then run plan, run, verify, backup, and restore end to end. Include `procs-residue`
  against a copy of the real `~/.sase/tasks/` tree, which is the case most likely to
  expose the marker-is-not-proof defect.
- **mac (macOS):** clone the kit revision into a scratch directory that is **not** a
  worktree of mac's live editable checkout, build `sase_core_rs` from source into a
  throwaway `uv` venv, and run the same matrix against copies of mac's `~/.sase`
  subsets. Never run `sase update`, never touch the live install, and never let an
  editable refresh pick the kit up. mac is best-effort over SSH; if it is unreachable,
  wait with `/sase_monitor` rather than substituting a Linux run, and if it stays
  unreachable stop and report rather than closing the phase on Linux-only evidence.
- **apollo:** no rehearsal run. apollo is unattended with a live Telegram writer.
  Produce its manifest from read-only probes and mark its operations as rehearsed on
  athena, which shares its platform.

Deliverables, each published as an artifact attached to the phase bead:

1. Per-host operation manifests for athena, mac, and apollo: the concrete ordered
   operation list, roots, expected counts, backup and secondary-copy locations, free
   space required, estimated duration measured during rehearsal, drain list from the G3
   inventory, and the verification query for each.
2. A rehearsal receipt recording the kit revision and checksum, baseline package and
   config revisions per host, every matrix case with its outcome, the measured backup
   and restore durations and sizes, and the `lock-residue` classification of both athena
   locks (census F3).

Backups taken during rehearsal prove the engine and measure cost; they are **not** the
cutover backups. `local-state-cutover` retakes them inside its own quiesced window. Say
so explicitly in the manifests.

Acceptance is the parent epic's stated bar, in full: no-op dry runs on canonical
fixtures, exact conversion on old fixtures, no data loss on interruption, conflict
refusals, a successful restoration rehearsal, a concrete operation manifest for each
fleet host, and no production data mutated. `just check-full` passes through
`/sase_monitor`; `sase bead epic-symbols` is clean.

## Refusals and stop conditions

Any phase that hits one of these stops and reports rather than working around it:

- A core release exposing the new bindings cannot be published (`kit-contract`).
- A host is unreachable or its toolchain is missing, so its leg cannot be rehearsed.
- An operation's precondition cannot be proved -- in particular a legacy proc row with
  no canonical counterpart, or a code-swap lock the current code still writes.
- Free space, checksum verification, or SQLite integrity checks fail on any host.
- A conversion cannot be shown lossless by re-parse and fingerprint comparison.

A refusal recorded in a manifest is a successful outcome for this epic. Silently
downgrading a refusal to a warning is not.

## Handoff to the parent phase

`sase-x7.2` is satisfied when `kit-rehearsal` closes with its two artifacts published
and attached. Its land agent decides the close; no phase worker here closes `sase-x7.2`,
the parent epic `sase-x7`, or any other ancestor.
