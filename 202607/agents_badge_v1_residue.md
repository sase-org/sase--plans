---
tier: epic
title: Truthful agents-repo update badge and ignore-proof hood publication
goal: 'The ACE agents-sync badge counts only genuinely incoming foreign work, this
  machine''s own legacy-v1 sidecar residue can never be re-imported as duplicate agents,
  hood publication cannot be silenced by user gitignore rules, and the retired v1
  payload is removed under explicit evidence-gated authorization.

  '
phases:
- id: core_evidence
  title: Core commit-SHA equivalence and evidence-aware v1 ownership
  depends_on: []
  size: medium
  description: '"''Phase 1: Core commit-SHA equivalence and evidence-aware v1 ownership''
    section: add pure abbreviated/full commit-SHA equivalence and an evidence-taking
    v1 group ownership classification to sase-core, expose both through the sase_core_rs
    binding and Python facades, and cover the full matrix in Rust and Python tests."

    '
- id: detection
  title: Owner-observed v1 never counts as an incoming update
  depends_on:
  - core_evidence
  size: medium
  description: '"''Phase 2: Owner-observed v1 never counts as an incoming update''
    section: apply the shared ownership rule in both agents-sync detection paths so
    this machine''s own v1 residue is counted as owner-observed instead of pending,
    prune its cache objects, and keep the cached no-network reconcile path from resurrecting
    a stale pending set."

    '
- id: import_guard
  title: Ignore-proof sidecar payload staging and stranded-hood repair
  depends_on: []
  size: medium
  description: '"''Phase 3: Ignore-proof sidecar payload staging and stranded-hood
    repair'' section: make the shared agents-sidecar commit choke point immune to
    user gitignore rules so hood directories like `bbugyi200.athena.gz` are staged
    and committed, republish the already-stranded hoods, and add a diagnostic for
    owner manifests that reference files missing from the commit."

    '
- id: import_safety
  title: Legacy v1 import can never fabricate owner duplicates
  depends_on:
  - core_evidence
  size: medium
  description: '"''Phase 4: Legacy v1 import can never fabricate owner duplicates''
    section: route the v1 import proof through the shared evidence rule so proven-owned
    entries are recorded unchanged instead of imported, refuse to mint legacy machine-qualified
    imported names for the current owner, and report honest dispositions."

    '
- id: retire_v1
  title: Evidence-gated v1 payload retirement and dead-code removal
  depends_on:
  - detection
  - import_safety
  size: medium
  description: '"''Phase 5: Evidence-gated v1 payload retirement and dead-code removal''
    section: add an explicit dry-run-by-default command that retires this machine''s
    v1 sidecar payload only when the current owner''s v2 manifest fully covers it,
    delete the dead v1 export code, and stop reporting unexported counts derived from
    the retired manifest."

    '
- id: verify
  title: End-to-end verification, surfaces, and documentation
  depends_on:
  - detection
  - import_guard
  - import_safety
  - retire_v1
  size: small
  description: '"''Phase 6: End-to-end verification, surfaces, and documentation''
    section: verify on athena that the badge drains to zero without creating imported
    artifacts, that the stranded hoods publish, and refresh badge tooltip, help, and
    CLI language to match the corrected semantics."

    '
create_time: 2026-07-25 07:05:28
status: wip
---

# Plan: Truthful agents-repo update badge and ignore-proof hood publication

## Problem

The ACE top bar permanently shows `⇅ 238`. The badge is documented as "cached foreign agent updates ready to import",
and its tooltip invites the user to click it or press `,U`. Every one of those 238 items is this machine's own history,
and clicking would corrupt the local agent list.

### What the 238 actually are

`~/.sase/agents_sync/status_snapshot.json` for `gh_sase-org__sase` holds 238 `pending_updates`, every one of them:

```
source_owner_kind = "username_unknown_v1"
source_username   = null
source_machine    = "athena"
```

They come from the root `manifest.json` of the agents sidecar
(`~/.sase/projects/gh_sase-org__sase/repos/agents/manifest.json`). That file is a frozen legacy-v1 manifest holding 338
pre-v2 runs, all with `machine: athena`, which group into 238 top hoods. It was last written on 2026-07-24 by an old
`chore(agents): sync from athena` commit; the current publisher writes `chore(agents): sync from bbugyi200.athena` and
never touches it. `write_manifest` is still imported by `src/sase/agents_sync/bundles.py` but is never called anywhere
in `src/` — the v1 exporter is gone. The file is pure residue.

All 238 v1 top hoods are already republished under the owner's v2 shard: the committed manifest at
`users/bbugyi200/machines/athena/manifest.json` lists 1060 hoods and is a strict superset of the 238. The badge is
therefore advertising, as "incoming work to pull down", 238 hoods this machine published itself and has already
re-published in the current format.

### Why detection calls them foreign

`capture_fetched_agent_updates()` in `src/sase/agents_sync/incoming_detection.py` handles the two layouts
asymmetrically. For a v2 owner manifest it calls `classify_agent_ownership(...)` and, on `EXACT_OWNER`, counts the hoods
as `exact_owner` and never makes them pending (`incoming_detection.py:149`). For the v1 root manifest it runs no
ownership check at all: every group produced by `legacy_manifest_groups()` is counted into `validated_foreign` and
published into the pending cache (`incoming_detection.py:106-126`).

That is a gap against the sase-8v design, which states in `sase/repos/plans/202607/global_agent_hoods.md` ("Legacy v1 is
read-only and never guessed"):

> A v1 entry matching the current machine may be promoted to the configured v2 identity only when a matching local owned
> artifact/commit association proves ownership.

The promotion rule was implemented, but only in the _import_ path (`bundles._find_proven_current_v1_artifact`).
Detection — the path that feeds the badge — never consults it. `sase-8v.8` ("Incoming-only badge") made the badge
foreign-only in the v2 sense while leaving v1 unconditionally foreign.

The Rust classifier is identity-only by design and cannot close this gap on its own: `classify_agent_ownership` in
`sase-core/crates/sase_core/src/agent_identity/identity.rs:202` returns `UsernameUnknownV1` for any v1 source, with no
evidence input.

### Why the promotion rule could not have saved it either

Even where it runs, the promotion rule is structurally dead. `_find_proven_current_v1_artifact` (`bundles.py:125`)
intersects the bundle's commit SHAs with the local artifact's commit markers:

```python
source_shas = {commit.sha.lower() for commit in bundle.commits}
...
local_shas = {sha.lower() for marker in commit_markers(artifact_dir) ...}
if source_shas & local_shas:
    return artifact_dir
```

The sidecar bundle stores **full 40-hex** SHAs; the local `commit_result.json` / `commit_results.json` markers store
**abbreviated** SHAs. A concrete pair from run `20260706234243` / `athena.06--code`:

```
bundle  commits.json  : "d7e06b77b42d89ecf4bb1538c6f89c6fe700124e"
local   commit_result : "result": "d7e06b77b", "commit_result": "d7e06b77b"
```

Exact set intersection can never match. Measured over the full pending set: **0 of 338 entries** are provable today.
With prefix-aware equivalence, 311 of 338 entries and 232 of 238 hoods become provable; the remaining 24 entries have no
surviving local artifact at their timestamp and 3 have a name match with no SHA overlap.

### The trap this leaves behind

Because detection marks all 238 foreign and the promotion proof always fails, clicking the badge or pressing `,U` runs
`integrate_cached_agent_updates` → `integrate_foreign_bundles`, which for each of the 338 entries falls through to
`_create_imported_artifact` and `claim_imported_registered_name("athena.06--code", "athena", ...)`. That would:

- create 338 duplicate artifacts of this machine's own runs, and
- re-mint legacy machine-qualified names of exactly the class `sase-91.5` was written to stop.

The local artifact store currently contains 4634 artifacts and **zero** with `imported_from_machine` set, so nothing has
been imported yet and no receipts file exists (`~/.sase/agents_sync/receipts/` is absent). The badge has never drained
because a receipt is only written by a completed import, and the only available import is the destructive one.

The user's premise is confirmed on the other side too: `,U` (`update_sase` in leader mode) reaches
`_execute_agents_leg`, which calls `integrate_cached_agent_updates` only — it never pushes. Pushing lives in
`sync_agents`, reachable from the Updates-tab `a` action and `sase agents sync`.

### A separate, real publication defect

The snapshot also carries two quarantine diagnostics:

```
bbugyi200.athena.gz: quarantined exact-owner v2 hood: could not read fetched object
  'agents/bbugyi200.athena.gz/README.md': fatal: path ... exists on disk, but not in '4a804c5f...'
bbugyi200.athena.o:  (same, for agents/bbugyi200.athena.o/README.md)
```

The cause is the user's global excludes file:

```
$ git check-ignore -v agents/bbugyi200.athena.gz/README.md
/home/bryan/.gitignore_global:11:*.gz    agents/bbugyi200.athena.gz/README.md
```

Hood directories are named `<username>.<machine>.<hood>`, so a hood named `gz` produces a directory ending in `.gz` and
a hood named `o` produces one ending in `.o`. Both match the user's global ignore patterns, so the whole directory is
invisible to git.

`commit_agents_payload_if_dirty` (`src/sase/agents_sync/git_sync_ops.py:24`) — the single choke point used by both
`sase commit` publication (`commit_publication.py:215,286`) and full sync (`git_sync_transaction.py:52,176`) — is
defeated twice over:

- its dirty probe is `git status --porcelain --untracked-files=all -- <paths>`, which honours excludes and therefore
  reports the sidecar clean, and
- its stage step is `git add -- <paths>` with no `--force`, which would skip the files anyway.

The owner manifest under `users/` is _not_ ignored, so it is committed listing files the commit does not contain.
Result: the two hoods are permanently unpublished (no cross-machine recovery, dead `SASE_AGENT` links) and every status
refresh emits quarantine noise. Patterns in this user's global ignore that can strike a hood name:
`*.aux *.bak *.dump *.gz *.log *.o *.out *.pyc *.svg *.swp *.tar *~`. Two of 1060 current hoods collide today; the
collision recurs by chance whenever the name allocator emits one of those short tokens.

## Design

### The ownership rule for a v1 group

A legacy-v1 manifest group is **owner-observed** — this machine's own history, never an incoming update — when
`group.machine == owner.machine_name` **and** at least one of:

1. **v2 coverage.** The current owner's v2 manifest (`users/<username>/machines/<machine>/manifest.json`) already
   publishes hood `group.hood`. This is first-party evidence written by this owner in the current format, strictly
   stronger than a local artifact, and it covers 238 of 238 groups here.
2. **Artifact/commit proof.** Some entry in the group matches a local, non-imported artifact by timestamp and
   machine-qualified name, and shares a commit SHA under prefix-aware equivalence.

Everything else stays foreign, including any v1 group whose machine token differs from ours and any same-machine group
with no evidence. The machine token alone never promotes anything, which keeps the sase-8v rule intact: two users with a
machine both named `athena` still cannot be confused, because promotion requires our own v2 manifest or our own local
artifacts to vouch for the hood.

### Where the rule lives

`classify_agent_ownership` already lives in `sase-core`. The new pieces are pure functions of their inputs and belong
beside it per the Rust core backend boundary: a CLI, web frontend, or editor integration must classify these hoods
identically to the TUI. Evidence _gathering_ (walking artifact dirs, reading git markers) stays in Python; only the
decision crosses into Rust.

### Ordering

Phase 2 alone stops the false badge. Phase 4 removes the duplicate-import trap. Phase 3 is independent of both and can
run in parallel. Phase 5 is the durable cleanup that sase-8v explicitly deferred to "a separately authorized cleanup"
and must land after the detection and import phases so retirement is never load-bearing for correctness.

## Phase 1: Core commit-SHA equivalence and evidence-aware v1 ownership

Work in `sase-core`, opened with `/sase_repo` (`sase repo open sase-core`). Use the path the skill prints; do not touch
the repo any other way.

Add to `crates/sase_core/src/`:

- **Commit-SHA equivalence.** A pure predicate that treats two hex SHAs as the same commit when the shorter is a
  case-insensitive prefix of the longer and is at least 7 hex digits. Reject non-hex input, empty input, and prefixes
  shorter than 7 digits rather than guessing. Place it beside the existing git/commit helpers (`git_query/` or
  `commit_footer.rs` neighbourhood — pick whichever module the existing code makes natural) and export it from `lib.rs`.
- **Evidence-aware v1 group ownership.** A function taking the group's machine token, the target `AgentOwnerIdentity`,
  and a small evidence record (`v2_hood_published: bool`, `proven_entry_count: usize`, `total_entry_count: usize`) and
  returning a typed classification of owner-observed vs foreign, implementing the rule in the Design section. Keep
  `classify_agent_ownership` unchanged — this is an additional, evidence-taking entry point, not a redefinition of the
  identity-only one.

Expose both through the `sase_core_rs` PyO3 binding, following the wire and error conventions the neighbouring bindings
already use.

In the sase repo, add thin facades: extend `src/sase/core/agent_identity_facade.py` with the ownership entry point and
put the SHA predicate wherever the existing commit facades live. Match the `require_rust_binding` pattern already used
by `classify_agent_ownership`.

Tests: full Rust unit matrix (equal full SHAs, full vs abbreviated in both argument orders, 7-digit boundary, 6-digit
rejection, non-hex rejection, case mixing; and every ownership combination of machine match/mismatch × v2-coverage ×
proven-count zero/partial/all). Python tests asserting the facades round-trip the binding.

## Phase 2: Owner-observed v1 never counts as an incoming update

Apply the Phase 1 rule in `src/sase/agents_sync/`.

**Detection** (`incoming_detection.py`). In the `relative == "manifest.json"` branch of `capture_fetched_agent_updates`,
classify each group from `legacy_manifest_groups()` before capturing it:

- Gather evidence once per refresh, not per group: read the current owner's v2 manifest hood set from the fetched commit
  (the loop already reads owner manifests — reuse that, do not re-shell git), and build the local artifact index once
  via the existing `bundles._v1_artifact_rows` seam.
- Owner-observed groups increment `exact_owner`, are removed from `pending`, and are never passed to
  `publish_cache_object`. They must not appear in `quarantine_diagnostics` — they are normal, not exceptional.
- Foreign groups keep today's behaviour exactly.

**Cached reconcile** (`incoming_cache.reconcile_pending_items`, reached from `status._reconcile_project_status`). This
path is no-network and no-git by contract, and today it simply carries the persisted 238 forward. It must drop persisted
items that the owner-observed rule excludes, using only evidence available without git or network — the persisted item's
`source_machine` plus the owner identity plus locally cached v2 coverage. Persist whatever the detection phase needs so
the cached path can decide without re-reading the sidecar; if that requires a small addition to the status snapshot
schema, bump `STATUS_SCHEMA_VERSION` and let an unknown version fall back to a clean re-fetch
(`_read_agents_sync_status_snapshot` already returns `None` on version mismatch). Do not weaken the no-network
guarantee.

**Cache pruning.** `prune_project_cache` must delete the cache objects that stop being pending. The 238 residue objects
under `~/.sase/agents_sync/cache/objects/` should disappear after one refresh.

Tests: a v1 group on the owner's machine with v2 coverage is owner-observed; the same group with no coverage but a
proven artifact is owner-observed; the same group with neither stays foreign; a v1 group on a different machine stays
foreign regardless of coverage; a same-machine group is not promoted when the v2 manifest belongs to a different
username; the cached reconcile path drops owner-observed persisted items and touches neither git nor the network; a
stale snapshot at the old schema version is discarded rather than misread. Update any existing expectations in
`tests/agents_sync/test_incoming_cache.py` and `tests/agents_sync/test_status.py` that assert the current
unconditional-foreign behaviour, and adjust the assertion rather than the rule.

## Phase 3: Ignore-proof sidecar payload staging and stranded-hood repair

Fix `commit_agents_payload_if_dirty` in `src/sase/agents_sync/git_sync_ops.py`. Both callers (`commit_publication.py`,
`git_sync_transaction.py`) go through it, so one fix covers `sase commit` publication and full sync.

The payload pathspecs (`README.md`, `schema.json`, `users`, `agents`, `families`) are entirely SASE-generated, so
forcing them past every ignore source is correct and precise:

- Stage with `git add --force -- <paths>`. `--force` defeats the global excludes file, any in-tree `.gitignore`, and
  `.git/info/exclude` alike; scoping to the known pathspecs keeps it from pulling in anything else.
- Decide dirtiness from staged state, not from an ignore-sensitive probe. Replacing the
  `status --porcelain --untracked-files=all` precheck with a stage-then-`git diff --cached --quiet` decision is the
  simplest form that cannot be silenced; keep a cheap precheck only if it is proven ignore-independent. Preserve the
  existing `bool | str` contract and the `agents_git_error` wrapping.

**Repair the stranded hoods.** With staging fixed, the next publication naturally picks up the untracked
`agents/bbugyi200.athena.gz/` and `agents/bbugyi200.athena.o/` trees. Verify that happens rather than assuming it; if
the publication path short-circuits on an unchanged manifest digest, add whatever makes the repair reachable without a
manual push.

**Diagnose the divergence class.** Add a check that flags an owner manifest listing files absent from the commit that
carries it — a strictly-better message than today's `could not read fetched object ...`, which reads like git corruption
when the real cause is a local ignore rule. Where the divergence is detected against the local checkout, name the ignore
rule using `git check-ignore -v` so the next occurrence is self-explaining.

Do not restrict the hood-name alphabet. Hood names like `gz` and `o` are valid SASE names; the defect is in the git
layer.

Tests: a repo whose `core.excludesFile` ignores `*.gz` still commits a `<user>.<machine>.gz` hood directory; the dirty
decision is `True` when the only change is an ignored-by-user path; an unchanged payload still returns `False` and
creates no empty commit; the manifest/commit divergence diagnostic fires and names the offending pattern.

## Phase 4: Legacy v1 import can never fabricate owner duplicates

Fix the import side in `src/sase/agents_sync/bundles.py` and `incoming_integration.py`.

- **Use prefix-aware SHA equivalence** in `_find_proven_current_v1_artifact`, via the Phase 1 facade. This alone lifts
  provability from 0/338 to 311/338 on this machine.
- **Route the group decision through the shared rule.** `integrate_foreign_bundles` currently decides per entry. Give it
  the group-level owner-observed verdict from Phase 1 so that a group proven to be ours is recorded as `unchanged` in
  its entirety and never reaches `_create_imported_artifact`. This is what closes the gap for the 27 entries that have
  no surviving local artifact but sit inside a hood that is demonstrably ours.
- **Refuse to mint owner-duplicating legacy names.** Make `_create_imported_artifact` / `claim_imported_registered_name`
  reject a claim whose machine token equals the current owner's machine when the group classified as owner-observed.
  This is a hard guard, not a warning: it is the backstop that keeps a future regression in detection from silently
  duplicating the user's history, and it complements the `sase-91.5` rule against non-terminal family-role names.
- **Honest dispositions.** A group skipped as owner-observed must not report `applied` with fabricated counts. Give
  `CachedIntegrationResult` a truthful outcome for it and make `cached_agents_result_line` /
  `summarize_cached_agents_results` render it plainly, so the `,U` task log says what happened. If this needs a new
  value in `CachedIntegrationDisposition`, add it and keep `ok` true for it.

Note that `integrate_agent_imports_with_receipts` (the full-sync path used by the Updates-tab `a` action and
`sase agents sync`) shares `integrate_foreign_bundles` and must inherit the same protection; it currently writes a
receipt per legacy group regardless of what the import did.

Tests: a proven-owned v1 group imports nothing and creates no artifact; an unproven foreign v1 group still imports
exactly as it does today; the registry guard rejects an owner-machine legacy claim; the full-sync path and the cached
path agree on the same fixture; abbreviated-vs-full SHA proof matches using a realistic marker payload
(`"result": "<9 hex>"` against a 40-hex `commits.json`).

## Phase 5: Evidence-gated v1 payload retirement and dead-code removal

The sase-8v design deferred this deliberately: "Do not mass-delete or rename the existing sidecar payload. V2 owner
manifests coexist until a separately authorized cleanup can safely retire v1." Phases 2 and 4 make the residue harmless;
this phase removes it.

- **Add an explicit retirement command** under `sase agents` (follow `sase/memory/cli_rules.md`). Dry-run by default,
  mutating only under an explicit flag. It must:
  - refuse unless every v1 group for the current machine is covered by the current owner's v2 manifest, printing the
    uncovered groups instead of proceeding;
  - only ever remove this machine's own v1 payload — the root `manifest.json` and the `agents/<machine>.*` directories
    for `owner.machine_name`, leaving any other machine's v1 data untouched;
  - commit and push through the existing publication choke point so locking, rebase, and outbox behaviour are unchanged;
  - print exactly what it removed. Git history keeps the payload recoverable.
- **Delete the dead v1 exporter.** `write_manifest` is imported in `bundles.py` and called nowhere in `src/`. Remove the
  dead import and, if nothing else uses it, the function and its export from `io.py`, keeping the v1 _readers_ intact.
- **Stop reporting a count derived from retired data.** `unexported_agents` is computed by
  `count_unexported_local_agents` against the legacy manifest and surfaced by `src/sase/agents/cli_sync.py:122`. It
  currently reads 40 and means nothing. Either derive it from v2 publication state or drop the column; do not leave a
  number whose only input is about to be deleted.

Check `sase/memory/symvision.md` before removing symbols.

Tests: retirement refuses when a v1 group has no v2 coverage; retirement with `--dry-run` mutates nothing; retirement
removes only the current machine's payload and leaves another machine's v1 entries in place; the resulting sidecar still
passes detection with zero pending items; the CLI surface matches the conventions in `cli_rules.md`.

## Phase 6: End-to-end verification, surfaces, and documentation

Verify on athena against the real state, and report measured numbers rather than assertions.

1. `sase agents status --refresh` (or the equivalent) and confirm `pending_updates` for `gh_sase-org__sase` drops from
   238 to 0, `exact_owner` accounts for the v1 groups, and the two `bbugyi200.athena.gz` / `bbugyi200.athena.o`
   quarantine diagnostics are gone.
2. Confirm the badge disappears from the ACE top bar and that `,U` reports no cached agent work.
3. Confirm the artifact store still holds zero artifacts with `imported_from_machine` set — the fix must drain the badge
   without importing anything.
4. Confirm `~/.sase/agents_sync/cache/objects/` is pruned and no receipts were fabricated.
5. Confirm `git ls-tree` in the sidecar now contains `agents/bbugyi200.athena.gz/` and `agents/bbugyi200.athena.o/`, and
   that their `SASE_AGENT` links resolve.

Then correct the user-facing language, which currently describes v1 residue as importable foreign work:

- `AgentsSyncIndicator._build_tooltip` in `src/sase/ace/tui/widgets/agents_sync_indicator.py` ("Cached foreign agent
  updates ready to import", "Click to import this captured cache without fetching. Press ,U for the comprehensive cached
  update").
- The `?` help popup and any Updates-tab text describing the badge, per `src/sase/ace/CLAUDE.md`'s help-popup
  maintenance rule.
- `sase agents status` column headers and help text touched by Phase 5.

Add a visual snapshot only if the badge's rendered form changes; the expected outcome here is that it stops rendering.

## Constraints and conventions

- Run `just install` before `just check` in a workspace directory, and run `just check` before reporting completion.
  Bead-only and `sdd/research/` changes are exempt.
- `sase-core` changes must be made through `/sase_repo`, with the Rust wire, binding, and tests updated there and the
  Python callers or adapters updated here.
- Do not modify `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims. The user has not granted permission
  for memory edits in this conversation.
- Read `sase/memory/cli_rules.md` before adding the Phase 5 subcommand and `sase/memory/symvision.md` before removing
  symbols.

## Reference measurements

Taken on athena, 2026-07-25, before any change. A phase agent should be able to reproduce these.

| Observation                                                 | Value                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------- |
| Badge / `pending_updates` for `gh_sase-org__sase`           | 238                                                           |
| Source classification of all 238                            | `username_unknown_v1`, username `null`, machine `athena`      |
| Root `manifest.json` entries / distinct top hoods           | 338 / 238                                                     |
| Machines represented in root `manifest.json`                | `athena` only                                                 |
| v2 hoods in `users/bbugyi200/machines/athena/manifest.json` | 1060                                                          |
| v1 top hoods absent from the v2 hood set                    | 0                                                             |
| Local artifacts with `imported_from_machine` set            | 0                                                             |
| Import receipts on disk                                     | none (`~/.sase/agents_sync/receipts/` absent)                 |
| Cache objects under `~/.sase/agents_sync/cache/objects/`    | 258                                                           |
| v1 entries provable today (exact SHA intersection)          | 0 / 338                                                       |
| v1 entries provable with prefix-aware SHA equivalence       | 311 / 338 (232 / 238 hoods fully proven)                      |
| Remaining unprovable entries                                | 24 with no local artifact at timestamp, 3 with no SHA overlap |
| Hoods ignored by `~/.gitignore_global`                      | `bbugyi200.athena.gz` (`*.gz`), `bbugyi200.athena.o` (`*.o`)  |
| Sidecar `git status --porcelain` with those hoods untracked | empty                                                         |
