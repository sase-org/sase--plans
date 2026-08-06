---
tier: tale
title: Stop dismissed agents from turning owned legacy-v1 hoods into unknown-user imports
goal:
  Legacy-v1 hoods this machine published stay owner-observed after their local runs are dismissed, ACE stops advertising
  the owner's own hoods as `unknown-user.<machine>.<hood>`, and the v2 owner manifest can no longer silently collapse to
  a single publication's hoods.
proposed_by: bbugyi200.athena.u1
create_time: 2026-08-06 09:45:34
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.u1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.u1.md)
- **COMMITS:**
  - [8b8acb4](https://github.com/sase-org/sase/commit/8b8acb4335881db4d490650461652237a405dd60) — fix(agents-sync): stop
    dismissed agents' owned legacy-v1 hoods from importing as unknown-user

# Plan: Stop dismissed agents from turning owned legacy-v1 hoods into `unknown-user` imports

## Problem

The ACE top bar showed a green `⇅ 5` badge whose tooltip read:

```
Cached incoming agent hoods from other owners:
sase:
  unknown-user.athena.ig      — 1 run, 0 families
  unknown-user.athena.il      — 1 run, 0 families
  unknown-user.athena.sase-8w — 9 runs, 0 families
  unknown-user.athena.toobig-l — 8 runs, 0 families
  unknown-user.athena.toobig-n — 5 runs, 0 families
```

Every one of those hoods is this machine's own work. `athena` is the owner's machine, and the 24 runs behind the five
hoods are ordinary local `sase` agent runs from 2026-07-22/23. They are being offered for import as if they belonged to
a stranger.

This is not only a mislabel. Accepting the badge (click, or `,U`) routes those hoods through
`bundles.integrate_foreign_bundles`, which would call `_create_imported_artifact` for all 24 entries: 24 brand new
artifact directories stamped `imported_from_machine: athena`, plus 24 `athena.<hood>...` claims in the agent name
registry. The existing safety valve, `_guard_owner_observed_legacy_import`, only fires when the group is already
classified `OWNER_OBSERVED`, which is exactly the classification that fails here, so it does not protect this case. The
user's own agent history would be duplicated back into ACE as foreign data.

## Verified root cause

Ownership of a legacy-v1 hood is decided by `classify_legacy_v1_group_ownership`
(`sase_core/src/agent_identity/identity.rs`, reached through `sase.core.agent_identity_facade`):

```rust
let owner_observed = group_machine_name == target.machine_name
    && (evidence.v2_hood_published || evidence.proven_entry_count > 0);
```

Both evidence signals were destroyed at once for exactly these five hoods.

**1. `v2_hood_published` was wiped by an owner-manifest collapse.** `users/bbugyi200/machines/athena/manifest.json` in
the agents sidecar (`~/.sase/projects/gh_sase-org__sase/repos/agents`) grew monotonically for days, then lost 1388
entries in a single publication:

| agents-repo commit | time                | hoods in owner manifest            |
| ------------------ | ------------------- | ---------------------------------- |
| `2a8dbe6bd`        | 2026-08-05 18:51    | 1376                               |
| …                  | …                   | steadily 1376 → 1389               |
| `5f244a229`        | 2026-08-06 09:11:53 | **1389** (includes all five hoods) |
| `6ad0b6e2a`        | 2026-08-06 09:16:12 | **1** (`tv` only)                  |
| `28cce4210`        | 2026-08-06 09:30:15 | 3                                  |

`6ad0b6e2a` is the exact `fetched_sha` recorded in all five cached items. The hood _directories_ survived (1391 of them
are still in the tree); only the manifest that references them was truncated. The commit's diff is 9 files: the manifest
plus the browsing README losing 1390 lines.

`publication_planning.plan_hoods` seeds its entry map from `read_owner_manifest(repo_root, owner, project)`, and
`v2_manifest_io.read_owner_manifest` returns an **empty** `V2OwnerManifest` when the file is not present:

```python
path = repo_root / owner_manifest_path(owner)
if not path.is_file():
    return V2OwnerManifest(owner, project)
```

A targeted publication of one hood (`tv`) starting from an empty manifest produces exactly the observed 1-hood manifest.
The agents-repo reflog shows a whole-tree `reset: moving to HEAD` and `checkout: moving from main to main` one second
before that commit — the signature of `vcs_clean_workspace` (`git reset --hard HEAD` + `git clean -fd`) followed by a
branch checkout, i.e. a workspace-preparation pass running against the agents sidecar clone, which all 24 workspaces
share. The precise interleaving that made the file unreadable is **not** fully diagnosed; this plan hardens the silent
fallback rather than claiming to have fixed the race.

**2. `proven_entry_count` was zeroed by agent dismissal.** The fallback proof is
`incoming_detection._legacy_proven_entry_count` (and its integration-side twin
`bundles._find_proven_current_v1_artifact`), which both draw their candidate rows from `bundles._v1_artifact_rows`:

```python
meta = _read_json_object(artifact / "agent_meta.json")
done = _read_json_object(artifact / "done.json")
if meta is not None and done is not None:
    rows.append((artifact, meta, done))
```

Dismissing or cleaning up an agent in ACE calls `delete_agent_artifacts`
(`src/sase/ace/tui/actions/agents/_killing_utils.py`), which reaches
`sase_core::agent_cleanup::execution::delete_agent_artifact_markers` and unlinks `done.json`, `workflow_state.json`, and
`prompt_step_*.json` while leaving `agent_meta.json`, `commit_results.json`, and the rest of the artifact intact. An
artifact that loses `done.json` therefore disappears from the v1 evidence set entirely.

Measured on the live store for `gh_sase-org__sase`:

- All 24 entries behind the five hoods have a local artifact directory that still exists, still has `agent_meta.json`
  with the right `name`, and still has commit markers.
- **24 of 24** have a commit SHA that matches the published `commits.json` — every one would be proven.
- **0 of 24** have `done.json`.
- Scanning all 257 legacy-v1 hoods in the remote v1 `manifest.json` against local artifacts, exactly **5** have zero
  `done.json`-bearing rows — precisely `ig`, `il`, `sase-8w`, `toobig-l`, `toobig-n`, with run counts 1, 1, 9, 8, 5,
  matching the tooltip exactly.
- Repo-wide, 5930 artifacts have `agent_meta.json` but only 1555 still have `done.json`.

With both signals gone the group classifies `FOREIGN`, `incoming_detection.legacy_captured_item` caches it with
`source_owner_kind="username_unknown_v1"` and `source_username=None`, and
`ace/tui/agents_sync_format.captured_agent_hood_label` renders the `None` username as `unknown-user`.

The same subsystem already treats `done.json` as optional: `inventory_sources.run_from_artifact` reads it with
`required=False` and relies on `agent_meta.json`. The v1 evidence path is the inconsistent one.

## Scope

Three defects, all in `src/sase/agents_sync/`, plus tests. Fix 1 alone clears the badge; the cached items are
self-healing, because once a group classifies `OWNER_OBSERVED` the capture pass pushes its key into
`discarded_hood_keys` and `incoming_cache_storage.prune_project_cache` deletes the cache objects. No manual cleanup of
`~/.sase/agents_sync/cache/objects` is needed or wanted.

### Fix 1 — v1 ownership evidence must survive dismissal

`src/sase/agents_sync/bundles.py`

- `_v1_artifact_rows`: keep a row whenever `agent_meta.json` parses, regardless of `done.json`. Substitute an empty dict
  for a missing/unparseable `done.json` so the row tuple type `tuple[Path, dict[str, Any], dict[str, Any]]` and all
  three consumers' `done.get(...)` calls keep working unchanged.
- Update the `v1_artifact_rows` docstring to state that `done.json` is optional and that `agent_meta.json` is the
  authoritative marker.
- `_find_proven_current_v1_artifact`: replace the `meta.get("imported_from_machine") is not None` guard with the shared
  `sase.agents_sync.inventory_io.is_imported(meta, done)` helper.

`src/sase/agents_sync/incoming_detection.py`

- `_legacy_proven_entry_count`: make the same `is_imported(meta, done)` substitution.

`is_imported` is strictly safer than the single-key check: it also matches `imported_digest`, `imported_source_owner`,
`imported_snapshot_digest`, `imported_transaction_key`, and `source_owner`. Verified against the live store — of the
~5.9k local `agent_meta.json` files, 3 carry v1 import markers and 79 carry v2 import markers (`imported_source_owner` /
`imported_snapshot_digest` / `imported_transaction_key`, which the current check misses), and **no** genuinely local
artifact carries any of these keys. `_imported_markers` writes every import key into `meta` as well as `done`, so a
dismissed imported artifact still cannot pose as local evidence once `done.json` is gone.

Proof strength is unchanged: an entry is still only proven when the artifact timestamp matches, the artifact is not
imported, the machine-qualified name matches, and a published commit SHA matches a local commit marker.

### Fix 2 — never self-import a run this machine already has

`src/sase/agents_sync/bundles.py::integrate_foreign_bundles`

Defence in depth for any future evidence gap. When `group_machine == owner.machine_name` and a local **non-imported**
artifact exists whose directory name equals `entry.artifact_timestamp` and whose machine-qualified name equals
`entry.name`, skip that entry instead of creating or refreshing an imported artifact. Count it as `unchanged` and append
a diagnostic to `IntegrationCounts.diagnostics` naming the hood, the entry, and the local artifact directory.

Note this deliberately does **not** require a commit-SHA match — that is what makes it a backstop for the proof path
rather than a copy of it. Reuse the `rows_by_timestamp` map the function already builds, so no extra filesystem scan is
introduced.

### Fix 3 — make the owner-manifest collapse impossible to do silently

`src/sase/agents_sync/publication_planning.py::plan_hoods`

- Before seeding `entries`, count this owner's on-disk hood directories, i.e. the children of
  `repo_root / f"users/{owner.username}/machines/{owner.machine_name}/hoods"` that contain a `snapshot.json`. A single
  `iterdir()` and `is_file()` per child; do not parse snapshots on this path.
- If `repo_root / owner_manifest_path(owner)` does **not** exist while one or more such hood directories do, raise
  `AgentsSyncFormatError` naming the manifest path and the hood count. This is the exact silent data-loss path and must
  become a visible, quarantined failure instead of a truncating republication.
- If the manifest exists but omits hoods that have on-disk snapshots, append a diagnostic to
  `V2PublicationCounts.diagnostics` reporting how many hoods are unreferenced and pointing at the repair command from
  Fix 4. Do **not** fail here: the sase agents repo is currently in exactly this degraded state (1391 hood directories,
  3 manifest entries), and hard-failing would block all publication until repair runs.

### Fix 4 — repair the manifest that already lost 1388 entries

`src/sase/agents_sync/publication_repair.py`

Add a manifest-reconstruction repair alongside the existing `repair_owner_hood_digests`, following that function's shape
exactly: it already reads only `users/{username}/machines/{machine}/...`, builds a payload dict, and returns
`(payload, report)` for the shared locked-sync plumbing to commit.

- For each on-disk hood directory of the local owner that is missing from the manifest, read `snapshot.json`, and
  rebuild
  `V2OwnerHoodEntry(content_digest(snapshot_bytes), hood_file_set(snapshot), len(snapshot.runs), <count of containers with kind == "family">)`.
- **Recovery must be tolerant.** `publication_validation.load_validated_publication` re-verifies every hood in the
  manifest and raises on the first mismatch, so a hood whose referenced run files have been pruned or have drifted would
  brick publication for the whole owner. Validate each candidate hood in isolation first — snapshot parses,
  owner/project/hood identity match, and every `V2FileReference` it names exists with the recorded size and digest — and
  skip candidates that fail, recording each skip in the returned report rather than raising.
- Merge recovered entries with the existing ones (existing entries win) and emit the re-signed owner manifest into the
  payload.

`src/sase/main/parser_agent.py` and `src/sase/agents/cli_sync.py`

Expose it as a new `sase agent sync --repair-manifest` flag, mirroring the existing `--repair-digests` wiring (flag →
handler branch → `_emit_sync_outcomes(..., mode="repair-manifest", ...)`) and adding a `REPAIR_MANIFEST_COMMAND`
constant next to `REPAIR_DIGESTS_COMMAND`.

**Before touching the CLI, the implementing agent MUST read `sase/memory/cli_rules.md` through the `/sase_memory_read`
skill** — the repo requires it for any new CLI subcommand or option. Also refresh the `sase ace` help popup only if a
keybinding or ACE option changes; this flag does not, so it should not.

## Tests

`tests/agents_sync/` already has the fixtures needed: `bundle_fixtures.write_bundle`, and `incoming_cache_fixtures`
(`LOCAL_OWNER`, `PROJECT`, `write_legacy_group`, `publish_owner`, `refresh`, `setup_v2_remote`, `seed_target_for`). Note
that `tests/agents_sync/test_incoming_cache_v1.py` and `test_bundles.py` monkeypatch `bundles._v1_artifact_rows`, so
those patches must stay type-compatible with the relaxed row builder.

Add, in `tests/agents_sync/test_incoming_cache_v1.py` (detection) and `test_bundles.py` (integration):

1. **Regression, detection.** A legacy-v1 group on the owner's machine whose local artifacts have `agent_meta.json` and
   matching commit markers but **no** `done.json`, with the hood absent from the v2 owner manifest, classifies
   `OWNER_OBSERVED`, produces no pending cache item, and prunes any previously cached item for that hood key. This test
   must fail before Fix 1.
2. **Name fallback still works.** An artifact whose `agent_meta.json` has no `name` but whose `done.json` does is still
   proven, guarding the `meta.get("name") or done.get("name")` fallback.
3. **Imported artifacts are still excluded.** An artifact carrying only `imported_source_owner` (v2 import marker, no
   `done.json`) does not count as owner evidence — this fails before the `is_imported` swap.
4. **Genuinely foreign stays foreign.** A group whose machine is not the owner's machine still classifies `FOREIGN` and
   still caches, with `source_username=None`.
5. **Fix 2.** With evidence deliberately withheld so the group classifies `FOREIGN`, `integrate_foreign_bundles` creates
   no imported artifact for an entry whose timestamp and name match a local non-imported artifact, returns it as
   `unchanged`, and reports a diagnostic.
6. **Fix 3.** `plan_hoods` raises when the owner manifest file is absent but hood snapshot directories exist; it emits a
   diagnostic (and still publishes) when the manifest merely omits on-disk hoods; and it is unchanged on a clean
   repository with no hood directories.
7. **Fix 4.** Manifest repair restores entries for hoods whose snapshots are intact, skips and reports a hood whose
   referenced run file was deleted or whose bytes drifted, leaves an already-complete manifest untouched (empty
   payload), and never reads another owner's path family.

## Verification

```bash
just install
just check
```

`just check` runs the whole-repo lint gates plus the diff-scoped test lane. Because this change touches
`src/sase/agents_sync/` and `src/sase/main/parser_agent.py`, also run the subsystem suite directly and, if scoped
selection escalates or looks unusual, the full lane:

```bash
python -m pytest tests/agents_sync -x -q
just check-full   # before landing
```

Do **not** run the repair command against the real `~/.sase` agents sidecar as part of implementation. Proposing that
repair run is the project owner's call; the plan's deliverable is the working, tested command.

## Out of scope / follow-ups

- **The race that emptied the owner manifest is not diagnosed.** Fix 3 converts the silent truncation into a hard error
  and Fix 4 repairs the damage, but the interleaving between agents-sync publication and the whole-tree
  `git reset --hard HEAD` / `git clean -fd` / `git checkout main` that workspace preparation runs against the _shared_
  agents sidecar clone is still unexplained. The implementing agent should file a task bead for it through the
  `/sase_new_task` skill, referencing agents-repo commits `5f244a229` → `6ad0b6e2a` and reflog entries at 2026-08-06
  09:16:11.
- **Orphaned incoming cache.** `~/.sase/agents_sync/cache/objects` holds 243 cached hoods; 238 belong to project key
  `gh_sase-org__sase-2`, which is no longer a registered project, so `prune_project_cache` never visits them. Worth a
  separate task bead for orphan-project cache pruning; not required to clear the badge.
- Changing the `unknown-user` label itself. It is a truthful rendering of the v1 wire format, which carries no username.
  Once Fix 1 lands it should only appear for hoods that really are another owner's.
- Changing `delete_agent_artifact_markers` in `sase-core` to preserve `done.json`. Dismissal removing loader markers is
  deliberate — the agent must not reload — so the evidence reader is the correct thing to fix, and the fix stays in this
  repo with no `sase-core` change required.
