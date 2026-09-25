---
tier: epic
title: Make sase bead work resilient to agent-name registry drift
goal: "A `sase bead work` retry never plans to launch an agent name that a live or
  historical owner still holds: registry rebuilds stop dropping in-flight name claims,
  bead-work cleanup selection repairs (or refuses) any remaining registry drift before
  it kills anything, and every launch-name conflict is detected before bead-store
  preclaims, checkpoint commits, or pushes happen.

  "
phases:
  - id: registry-inflight-claims
    title: Registry rebuilds keep in-flight claims
    depends_on: []
    size: medium
    description:
      "registry-inflight-claims: make rebuild_name_registry() (both the optimistic
      _commit_rebuild_locked path and the _rebuild_name_registry_locked fallback) carry
      forward prior local artifact-backed claims whose artifact dir is identity-pending
      (bootstrap agent_meta.json only, no name/clan/session yet) while its runner
      process is alive, re-derive entries for dirs whose named metadata landed after the
      unlocked scan, and keep dropping claims whose dir is gone, dead, or names a
      different identity. Share one per-artifact derivation helper between the full scan
      and the carry-forward. Add regression tests for the incident sequence."
  - id: bead-work-registry-drift
    title: Bead-work selection repairs registry drift
    depends_on: []
    size: small
    description:
      "bead-work-registry-drift: in select_bead_work_launch, detect slots whose registry
      lookup is missing while the ace-run artifact owner view has records for that name,
      force one registry rebuild plus a fresh reservation snapshot, and classify
      normally; if drift survives the rebuild, emit a BLOCKED target so the command
      aborts before any destructive cleanup or bead-store mutation. Zero extra cost when
      there is no drift."
  - id: bead-work-launch-name-preflight
    title: Launch-name preflight before bead-store mutations
    depends_on:
      - registry-inflight-claims
      - bead-work-registry-drift
    size: medium
    description:
      'bead-work-launch-name-preflight: add a plan-only registry reservation API that
      runs the same Rust ownership planner without applying, use it in the epic and task
      bead-work paths right after force-reuse cleanup (before plan snapshot, mark-ready,
      preclaim, checkpoint, push) to fail fast with owner details and the resume
      command, and reuse it to explain launch-time reservation collisions instead of
      surfacing the misleading "try ''X.61''" suggestion.'
proposed_by: bbugyi200.athena.0s9
create_time: 2026-09-25 13:51:07
status: wip
---

- **PROMPT:**
  [prompts/202609/bead_work_registry_drift_resilience.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/bead_work_registry_drift_resilience.md)

# Make `sase bead work` Resilient To Agent-Name Registry Drift

## Goal

A `sase bead work` retry must never plan to launch an agent name that some owner still
holds. Concretely:

1. Registry rebuilds must not drop a name claim that a live runner holds but has not yet
   published in its named `agent_meta.json`.
2. If the registry drifts anyway (any cause), bead-work cleanup selection must notice
   before it kills anything. It either repairs the registry and includes the owner in
   the cleanup preview, or it aborts with a `BLOCKED` explanation.
3. Any remaining launch-name conflict must be caught before bead-store preclaims,
   checkpoint commits, or pushes happen. The error must name the conflicting owner and
   give the resume command, not a meaningless "try 'X.61'" suggestion. Bead-work names
   are deterministic.

## Incident (2026-09-25, local time EDT)

The user ran `sase bead work 19i 17m -Y` to retry epic `sase-19i`. The preview listed
KILL for `sase-19i.3/.4/.5/.land` and REMOVE for `sase-19i.1/.2`, but it did not list
`sase-19i.6`. The cleanup ran and the epic launch checkpoint was committed and pushed.
The launch then failed:

```
Error: agent launch failed for epic sase-19i: registry reservation batch blocked:
bead-work-name-5: agent name 'sase-19i.6' is already taken; try 'sase-19i.61'
```

The rollback restored the 7 preclaims (the bead-store rollback is correct and must be
kept).

Forensics (durable timing log `~/.sase/logs/tui_launch_timing.jsonl*`, artifact dirs,
and the registry at `~/.sase/agent_name_registry.json`):

- The first launch of the epic ran from about 13:06 to 13:11:37. Its runners spawned one
  at a time: `.6` spawned at 13:11:10 and `.land` at 13:11:21. In `.6`'s artifact dir,
  `workflow_state.json` and `raw_xprompt.md` were written at 13:11:11–12.
  `agent_meta.json` was last written at 13:11:32, which is when the named metadata
  landed.
- `.6` was alive and WAITING (`waiting.json`, pid alive) the whole time, up to the
  retry.
- At the retry's `initial_selection` (13:20:51+), the registry was fresh (no rebuild
  happened), but `registry_snapshot.lookup("sase-19i.6")` returned nothing. The summary
  shows `cleanup_owner_count=6`. The artifact owner view (`load_agent_owner_view`) did
  include a live record named `sase-19i.6`.
- The cleanup deleted the six other artifact dirs. That changed the registry's source
  signature, so the wipe's `rebuild_name_registry()` (stage `index_maintenance`)
  re-derived `sase-19i.6` from its now-named `agent_meta.json`. The launch reservation
  `bead-work-name-5` then collided with that entry.
- Right after the failure, the registry held only `sase-19i` (clan container,
  `artifacts_dir` = `.6`'s dir) and `sase-19i.6` (`reservation_kind: claimed`). This
  confirms that a later rebuild restored the entry that had gone missing.

## Root Cause

1. **Rebuilds drop in-flight claims.** `rebuild_name_registry()` in
   `src/sase/agent/names/_registry.py` works in three steps:
   - It scans artifact and bundle sources without holding the lock
     (`_collect_rebuild_source_entries`).
   - Under the allocation lock, it keeps only the planned kinds from the current file
     (`collect_planned_reservation_entries`: `planned`, `planned_clan`,
     `cleanup_in_progress`).
   - It adds the scanned entries and writes the result.

   A runner claims its name inside `resolve_agent_identity`
   (`src/sase/axe/run_agent_directive_identity.py`), where
   `claim_exact_planned_registered_name` flips `planned` → `claimed`. The runner writes
   the named `agent_meta.json` only later: `src/sase/axe/run_agent_directives.py`,
   "Write metadata after the name reservation succeeds", after the clan summary scripts.
   Until then the dir holds only the bootstrap metadata written by
   `_write_bootstrap_agent_meta` in `src/sase/axe/run_agent_runner_bootstrap.py`: `pid`,
   `process_identity`, `output_path`, and `launch_scratch_key`, with no name.
   `_collect_workflow_artifact_entries` derives no entry from such a dir. Any rebuild in
   this window, for example one triggered by a sibling runner's new artifact dir,
   silently drops the live `claimed` entry.

2. **The drop is never noticed.** The freshness proof (`_source_signature` in
   `src/sase/agent/names/_registry_store.py`) fingerprints artifact directory _paths_
   only. This is by design; see the `source_signature_paths` docstring and
   `test_registry_signature_ignores_live_artifact_output`. As a result, the later named
   `agent_meta.json` write never makes the registry stale. The missing claim stays
   missing until an unrelated directory is added or removed.
3. **Bead-work selection trusts the registry alone.** `select_bead_work_launch` in
   `src/sase/bead/cli_work_cleanup_selection.py` does `continue` for any slot whose
   registry lookup is `None`, even when the owner view it just loaded has a record with
   that exact name. The preview under-reports owners, and the name is planned as free.
4. **The conflict surfaces too late.** The first place that notices is the launch
   reservation (`_reserve_planned_bead_work_launch` in
   `src/sase/agent/launch_cwd_bead_work.py`). By then mark-ready, preclaim, the
   checkpoint commit, and the push have all happened. The failure forces a rollback
   commit and push, and the raw planner message suggests a meaningless `X.61` name.

Everything involved (the registry rebuild and scan, and bead-work cleanup selection) is
Python-owned in this repo today. The Rust ownership planner
(`plan_agent_ownership_batch`) is only reused unchanged, so no `sase-core` change or
`sase-core-revision.txt` bump is needed.

## Phase `registry-inflight-claims` — Registry rebuilds keep in-flight claims

Fix the root cause in the registry rebuild. Leave the source-signature design alone:
adding live metadata mtimes to the signature would make the registry permanently stale.

Changes:

- In `src/sase/agent/names/_registry_scan_collectors.py`, factor the per-artifact body
  of `_collect_workflow_artifact_entries` into a helper, for example
  `collect_single_artifact_entries(entries, artifact_dir, *, project_dir, workflow_dir, dismissed_suffixes, identity) -> bool`
  (returns whether a source was parsed). The full scan must call this helper, so the
  scan and the carry-forward can never disagree about which names a dir owns.
- Add `collect_inflight_claim_entries(entries, existing, identity)` next to
  `collect_planned_reservation_entries`. It considers only entries of `existing` (the
  registry file re-read under the lock) whose key is **not** already in `entries` after
  the planned-plus-scanned merge. For each such entry `E` (key `N`):
  - Skip unless `E` is a local artifact-backed claim:
    - `origin` is `None` or `"local"`;
    - `source == "artifact"`;
    - `reservation_kind` is not a planned kind (those are already retained), not
      `auto_prefix`, and not `owner_namespace`. That leaves `claimed`, clan claims, and
      agent-session container claims;
    - `artifacts_dir` is an existing directory.
  - Re-read that one dir's `agent_meta.json` and `done.json` under the lock:
    - **If the payload now yields any identity** (names via `owner_identity_names`, a
      clan via `clan_from_payload`, or an agent session via
      `agent_session_from_payload`), the scan raced the named-meta write. Derive entries
      for this dir with the shared helper into a temporary dict and merge them with
      `setdefault`, so scanned entries win. If `N` is not among them, drop `E`: the dir
      positively names a different identity.
    - **If the payload yields no identity and there is no `done.json`**, the dir is
      identity-pending. Carry `E` forward unchanged (including any `collision_owners`)
      only if `is_process_alive(meta, dir)` is true. Import `is_process_alive` locally
      from `sase.agent.names._common` to avoid cycles. A dead process or a missing pid
      drops `E`, as today. A runner that crashes inside the window therefore still frees
      its name.
  - Candidates are only the keys missing from the merged result, so the happy path does
    no extra directory reads.
- In `src/sase/agent/names/_registry.py`, call the new collector from both
  `_commit_rebuild_locked` and `_rebuild_name_registry_locked`. Call it after
  `entries.update(scanned)`, reusing the same locked `_read_registry(path)` result that
  planned retention already reads. Do not read the file twice. Update the
  `rebuild_name_registry` docstring to state the invariant: _a rebuild never discards a
  live claim it cannot disprove._
- `entry_owner_missing` staleness is unchanged. A carried entry whose dir later
  disappears still makes the registry stale, and the next rebuild drops it.

Tests (extend `tests/test_agent_name_registry_rebuild.py` or add a focused module; reuse
`tests/_agent_names_fixtures.py` and the `patch.object(Path, "home", ...)` pattern):

1. **Incident regression.** Set up an artifact dir with only bootstrap meta
   (`pid=os.getpid()` plus a real `process_identity` token), a planned reservation, and
   `claim_exact_planned_registered_name`. Add a sibling artifact dir so the registry
   goes stale, then run `rebuild_name_registry()`. The claimed name must still resolve.
   Then write the named meta. The name must still resolve and nothing may rebuild
   spuriously.
2. The same retention through the `_rebuild_name_registry_locked` fallback.
3. An identity-pending dir with a dead or missing pid: the entry is dropped (unchanged
   behavior).
4. **Race.** The scan result predates the named meta, but the meta exists by commit.
   Patch `_collect_rebuild_source_entries` to return the pre-meta scan. The entry must
   be present and derived from the dir.
5. The dir now names a different identity: the prior claim is dropped.
6. The dir was removed: the entry is dropped, and the stale detection path is unchanged.
7. A clan-container claim whose only member is identity-pending and live is retained.

## Phase `bead-work-registry-drift` — Bead-work selection repairs registry drift

Defense in depth: even with the rebuild fixed, bead-work cleanup selection must not
trust a registry that disagrees with the artifact owner view it has already loaded.

Changes in `src/sase/bead/cli_work_cleanup_selection.py` (`select_bead_work_launch`):

- After loading `view` and `registry_snapshot`, compute the drifted slots: slots where
  `registry_snapshot.lookup(slot.owner_name) is None` and
  `view.records_for_agent_name(slot.owner_name)` is non-empty. Legacy land-alias slots
  (`allow_populated_clan_skip`) participate the same way.
- If any slots drifted:
  - Print one stderr line naming them, for example
    `agent-name registry is missing 1 owner(s) found in agent artifacts (sase-19i.6); rebuilding the registry before cleanup selection`.
  - Call `rebuild_name_registry()` (public in `sase.agent.names`), then take a fresh
    `registered_name_reservation_snapshot()`. When a timer is present, wrap this in
    `timer.stage("registry_drift_repair", drifted_owner_count=N)`.
  - Then run the existing classification loop.
- In the loop, a slot whose lookup is still `None` but that has artifact records must
  not be treated as free. Emit a `BLOCKED` `CleanupTarget` whose detail names the owner
  dir(s), for example:
  `agent name 'X' has artifact owner(s) at <dirs> that the agent-name registry does not record even after a rebuild; run \`sase
  doctor -v\` and
  retry`. The existing BLOCKED handling in `launch_epic_bead_work`and`launch_task_bead_work`then aborts before any destructive cleanup or bead-store mutation. Dry runs render it through`render_blocked_launch_warning`.
- With no drift there must be no rebuild and no extra snapshot. The happy path costs
  exactly what it does today.
- `_select_preserved_slots_from_registry` already falls through when any owner is
  missing, so it needs no change. `revalidate_bead_work_launch_selection` reuses
  `select_bead_work_launch`, so drift that appears between the preview and the cleanup
  surfaces as the existing "owner changed after cleanup preview; rerun" error. Keep that
  behavior.

Tests (new module under `tests/test_bead/`, following the fixtures in
`test_cli_work_epic_launch_cleanup.py` and `test_cli_work_cleanup_verify.py`):

1. **Incident shape.** The registry has owners for `.1`–`.5` and `.land`, but `.6` is
   missing while a live WAITING `.6` artifact record exists. The selection must rebuild
   exactly once, and the preview must include `.6` as KILL alongside its siblings. `.6`
   must be in `launch_names`.
2. **Unrepairable drift.** Patch the rebuild to a no-op. The result must be a BLOCKED
   target for `.6`, and `launch_epic_bead_work` must raise `BeadWorkError` before
   `preclaim_epic_work` or the checkpoint runs.
3. With no drift, `rebuild_name_registry` is never called.
4. Drift that appears only at revalidation raises the "owner changed after cleanup
   preview" error.

## Phase `bead-work-launch-name-preflight` — Launch-name preflight before bead-store mutations

This is the last line of defense for any conflict that remains, such as a concurrent
launch of the same name after revalidation. Detect it before bead-store preclaims,
checkpoint commits, or pushes happen, and explain it usefully.

Changes:

- **Plan-only registry API.** In `src/sase/agent/names/_registry_batch.py`, add
  `plan_registered_name_reservations(hooks, reservations) -> RegisteredNameReservationBatchResult`.
  It takes one `registered_name_reservation_snapshot`, runs `_plan_reservation_batch`,
  and returns `_result_from_plan(plan)`. It never applies anything and never raises on
  blocked items. Bind it through `src/sase/agent/names/_registry_facade_batch.py` and
  `_registry.py`, then export it from `sase.agent.names` (including `__all__`). It must
  use the same Rust planner as `mutate_registered_name_reservations`, so preflight and
  launch cannot disagree.
- **Bead-work preflight helper.** Add a new module, for example
  `src/sase/bead/cli_work_name_preflight.py`, with
  `preflight_bead_work_launch_names(launch_names, *, resume_command, timer=None) -> None`:
  - For each name, build
    `RegisteredNameReservation(operation="reserve_planned", name=normalize_owned_agent_name(name, identity), artifact_dir=<synthetic never-existing dir under sase_projects_dir()>)`,
    the same shape the real launch reserves.
  - If the plan has blocked items, map each item's `request_id` back to its name. Look
    up the current owner in a snapshot and raise `ForcedReuseCleanupError` with one line
    per name, for example:
    `agent name 'sase-19i.6' is still owned by <artifacts_dir or bundle_path> (state=<state>, kind=<reservation_kind/container_kind>) and was not selected for cleanup`.
    Follow the lines with `No bead state was changed; rerun \`<resume_command>\` to
    review this owner.`
  - Never include the planner's "try 'X.N'" suggestion.
  - Wrap the work in `timer.stage("launch_name_preflight", name_count=N)` when a timer
    is given.
- **Epic path.** In `launch_epic_bead_work` (`src/sase/bead/cli_work_handler.py`), call
  the helper for `selection.launch_names` right after the `force_reuse_cleanup` stage
  and before `plan_snapshot`, `mark_ready`, `preclaim`, and `graph_publication`. Convert
  `ForcedReuseCleanupError` to `BeadWorkError`. The resume command comes from
  `_resume_command(epic_id, capacity=capacity)`.
- **Task path.** Do the same in `launch_task_bead_work`
  (`src/sase/bead/cli_work_task.py`) at the equivalent point: after its forced-reuse
  cleanup and before the task preclaim and checkpoint. Convert the error to
  `TaskBeadWorkError`.
- **Launch-time collisions.** In both paths' `agent_launch` `except` branches, when the
  exception is a `NameCollisionError` (which includes
  `RegisteredNameReservationBatchError`), re-run the preflight helper after the existing
  rollback so the raised message carries the owner details and the resume command. If
  the preflight now finds no blocker (the owner vanished), keep the original message and
  add the rerun hint. The rollback behavior itself is unchanged.
- Dry runs return before cleanup, so the preflight does not apply to them.

Tests:

1. **Parity.** For each registry state below, `plan_registered_name_reservations`
   blocked-ness must match whether `mutate_registered_name_reservations` raises:
   - free name;
   - name claimed by a live artifact;
   - name claimed by a done artifact;
   - stale planned reservation whose dir does not exist;
   - clan container name.
2. **Epic.** A launch name is owned by an artifact that the selection did not include.
   Simulate this by adding the registry entry after revalidation. The launch must raise
   `BeadWorkError` before `preclaim_epic_work` or `checkpoint_epic_work_launch` runs.
   The message must name the owner dir and `sase bead work <epic>`, and must contain no
   `try '`.
3. **Task.** The same check for `launch_task_bead_work`.
4. **Launch-time collision.** The preflight passes, then the reservation is patched to
   raise `RegisteredNameReservationBatchError`. The rollback must still run, and the
   final message must include the owner details and the rerun command.

## Verification

Each phase runs `just check` through `sase tool run`, per the `lint_and_test` reference
memory. Do not run `just check-full`. The epic is done when all phase tests pass. In
particular, the incident regression must pass: a claimed name that is identity-pending
and alive survives a rebuild, a drifted owner shows up in the cleanup preview, and a
launch-name conflict fails before any bead-store mutation.

## Out Of Scope

- **Stale beads git `index.lock`.** In the incident it was already 384s old at first
  sight. `run_with_git_lock_retry` recovered it correctly, but only after the full 6.3s
  backoff ladder. A fast path for locks that are already stale when first seen would be
  a separate, low-value tweak.
- **Slow `owner_discovery` (10–40s) and `index_maintenance` rebuilds (about 50s)** seen
  in the timing log. They are performance issues, not correctness issues.
- Changing the registry source-signature design or the runner's write-meta-after-claim
  ordering. The rebuild-side invariant closes the window no matter when metadata is
  written.
