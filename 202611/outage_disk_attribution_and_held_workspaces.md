---
tier: epic
title: Honest disk attribution and an end to the held-workspace re-settlement loop
goal: Disk inventory reports unmeasured owners as unknown rather than 0 B, and measures
  workspace roots before anything else. Pressure recovery runs whether or not attribution
  is complete, and a missed target produces a distinct escalation. Held workspaces
  settle once per evidence fingerprint. Their notices get a +1 only when evidence
  changes, plus an hourly heartbeat on undismissed rows. The notification store skips
  the rewrite for coalesced and dismissed-row +1s.
parent_bead: sase-1tz.1
phases:
- id: store_guard
  title: Coalesce identical +1s and stop rewriting the store for dismissed rows
  description: 'store_guard: in sase-core, key-based +1s to dismissed rows become
    no-ops unless the caller opts in, and identical +1s within a window coalesce.
    Neither path rewrites the store or bumps its generation. Python adopts the new
    outcomes, and the pin moves.'
  size: medium
  depends_on: []
- id: held_settle
  title: Settle held workspaces once per evidence fingerprint with change-only +1s
  description: 'held_settle: persist an evidence fingerprint and a cheap stat signature
    with the retention record. Later stale-running cleanup passes over unchanged evidence
    then skip git probes, snapshots, decision-sheet publishing, and +1s. Add the hourly
    still-held heartbeat for undismissed rows.'
  size: medium
  depends_on:
  - store_guard
- id: starter_scope
  title: Check that starter-inherited repo lists stay inside the monitor's workspace
  description: 'starter_scope: verify whether closed sase-1rg.3 keeps starter-inherited
    repo lists from probing another workspace. Fix the gap with a regression test
    if it does not.'
  size: small
  depends_on: []
- id: unknown_sizes
  title: Represent unmeasured disk owners as unknown, never 0 B complete
  description: 'unknown_sizes: the tree sizer returns measured, lower-bound, or unmeasured
    results. The sase-core disk-inventory wire moves to v2 with a per-row size state,
    unmeasured-owner reporting, and lower-bound totals. sase disk list renders unknown
    sizes honestly, and the pin moves.'
  size: medium
  depends_on: []
- id: owner_first
  title: Measure workspace and owner roots first under their own budget
  description: 'owner_first: workspace checkouts and owner roots are sized before
    managed-tmp buckets and the stray walk, under a budget separate from the shared
    stray budget. Record the sase-1bf.4 regression finding and the du tolerance measured
    on apollo and athena.'
  size: medium
  depends_on:
  - unknown_sizes
- id: pressure_recovery
  title: Recover first, name unmeasured owners, and escalate missed recovery targets
  description: 'pressure_recovery: the disk_pressure chop reaps before collecting
    a fail-open inventory. Its notice names unmeasured owners and uses an episode-scoped
    dedup key. A missed recovery target produces a distinct escalation with the target,
    reclaimed bytes, and blockers.'
  size: medium
  depends_on:
  - unknown_sizes
  - store_guard
- id: standing_skips
  title: Report repeated managed-tmp reaper skips as one standing state
  description: 'standing_skips: the managed_tmp_reap chop keeps refusing to reap entries
    it cannot fully inspect. It warns once when a skip appears or its reason changes,
    and logs again when the skip clears, instead of warning every pass.'
  size: small
  depends_on: []
proposed_by: bbugyi200.athena.sase-1tz.1
proposed_with: claude/opus
create_time: 2026-11-19 07:28:38
status: wip
bead_id: sase-1tz.1.1
---

- **PROMPT:** [prompts/202611/outage_disk_attribution_and_held_workspaces.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/outage_disk_attribution_and_held_workspaces.md)
- **PARENT:** [202611/notification_log_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202611/notification_log_audit_fixes.md)
- **BEAD:** [sase-1tz.1.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1tz/sase-1tz.1.1.md)

# Plan: Honest disk attribution and the held-workspace re-settlement loop

This child epic implements phase `outage` (**sase-1tz.1**) of parent epic **sase-1tz**,
`plan:202611/notification_log_audit_fixes.md`. It covers items 1 and 2: audit findings
F2 (disk attribution blindness and the Nov 16 apollo ENOSPC outage) and F1 (the
held-workspace +1 loop).

## Read first (every phase)

- **Audit.** The source of truth is
  `research:202611/apollo_athena_sase_notification_and_log_audit/apollo_athena_sase_notification_and_log_audit.md`.
  Read it with `sase artifact read <ref> "<reason>"`; if the reference resolves as
  missing, open the `sase--research` sidecar with `sase repo open` first. The
  `…__cdx.md`, `…__cld.md`, `…__mus.md`, and `…__gem.md` swarm reports in the same
  directory hold the raw F1/F2 counts, log excerpts, and timestamps.
- **Final parent decisions.** The parent epic's DECISIONS are final, so do not re-ask
  them. The one that applies here is **`held_heartbeat = yes`**: each undismissed
  "Workspace held" row gets at most one "still held" +1 per hour, and dismissed rows
  never tick. `codex_bundle` and `blocked_waiters` belong to other parent phases.
- **Rust core boundary.** The notification store, the disk-inventory classifier, and the
  disk-pressure classifier are shared backend behavior and live in `sase_core`. Open
  sase-core with `sase repo open sase-core` and work only in the path it prints. A phase
  that changes a binding sase calls must also move sase's `sase-core-revision.txt` pin
  past that sase-core commit. See `docs/rust_backend.md`, which links to
  `docs/contribute/rust-backend.md` for the pin procedure. Wire schema constants are
  also checked by `tools/validate_sase_core_rs`; read the `tools.md` reference memory
  before touching `tools/`.
- **Verification.** `just check` is the agent verification recipe. Read the
  `lint_and_test.md` reference memory before finishing. Run `just check-full` only when
  explicitly instructed. Every behavior change gets focused tests, including the
  regression scenarios named in its phase.
- **Live hosts are read-only.** Global install is human-only, so live acceptance happens
  after the user installs. Phases may take read-only measurements on apollo and athena;
  the `tailnet.md` reference memory covers SSH access. Each phase that measures records
  the exact before-numbers and the command to repeat after install. It records them as a
  note on its own phase bead and as a one-line summary note on **sase-1tz.1**. Never
  dismiss notifications, repair state, or delete files on either host. Propose any
  one-time cleanup through `/sase_gate`.
- **Discovered work.** Phase workers are epic phase workers. Record follow-up work as
  `PROPOSED FOLLOW-UP:` notes on your own phase bead, never as new task beads.
- **Docs and memory.** User-visible behavior changes update the matching page under
  `docs/` (`docs/reference/cli/disk.md`, `docs/reference/disk-cleanup.md`,
  `docs/reference/cli/notify.md`, `docs/reference/automation/notification-cli.md`,
  `docs/reference/automation/notification-types.md`,
  `docs/reference/automation/scheduler-housekeeping.md`,
  `docs/reference/agents/workspaces.md`). Read the `docs_writing.md` reference memory
  first. A survey found no memory note that states behavior this epic changes. If a
  phase finds one, it records a `PROPOSED FOLLOW-UP:` and does not edit memory itself.
- **Already fixed: verify, don't rebuild.** Commit `c6282f75b2` already removed
  workspace retention from agent bundles. Commit `413694618e` already started reporting
  managed-tmp and disk-pressure outcomes. Build on both.

## Coordination with open work

- **sase-zw → sase-zw.8 → sase-zw.8.7 → sase-zw.8.7.8** is the disk-retention epic
  chain. It is IN_PROGRESS, and its sase-zw.8.7.8 phases have had no commits since
  Sep 15.
  - `sase-zw.8.7.8.4` (`inventory_bounds`) owns bounding _workspace discovery_
    (`collect_workspace_inventory`) and target resolution under the shared deadline.
  - `sase-zw.8.7.8.1` (`scratch_bounds`) owns scratch traversal and launch-liveness
    bounds. Its note #1 documents apollo's unreadable-`environ` liveness stall.

  This epic does not take over that work. It does not bound discovery, and it does not
  change managed-tmp liveness or deletion-safety rules. `owner_first` and
  `pressure_recovery` each add a note on **sase-zw.8** naming what they changed and the
  overlap with `sase-zw.8.7.8.1`/`.4`.

- **sase-11h** ("Retire unsafe disk cleanup paths …") is IN_PROGRESS. Its note #1
  reports that the disk_pressure chop mapped a failing reason to `no_op`.
  `pressure_recovery` checks that this is fixed on master (see `413694618e`). It fixes
  the mapping if not, and records the result as a note on sase-11h.
- **sase-1bf.4** (closed) promised "cover every registered root and every workspace
  checkout before the stray walk." `owner_first` confirms or refutes the audit's
  regression claim and records the finding as a note on sase-1bf.4, without reopening
  it.
  - Planning evidence: the collection order (managed tmp → state → workspaces → rust
    targets → strays), all under one shared 5 s / 50k-node budget, dates from
    `028c5137be` (2026-09-15).
  - `5db68f77f2` (sase-1bf.4) did not change that order. Its docstring ("an exhausted
    shared scan budget can only clip the stray walk") was false when written.
  - So this looks like an unmet requirement rather than a later regression. Verify
    before recording.
- **sase-1rg.3** (closed) scoped monitor retention evidence to the monitor's own
  workspace. `starter_scope` checks its coverage.
- **Parent phase `hygiene` (sase-1tz.4)** relies on `store_guard` for its `wait_checks`
  amplification fix. It also owns `workspace-rescue` dedup and the "already pinned
  (no-op)" claim-ledger writes. `held_settle` reduces those writes as a side effect but
  does not own them. `pressure_recovery` owns the `disk_pressure` dedup key, so
  `hygiene` only handles `workspace-rescue`.

## Phase `store_guard`: sase-core notification store +1 guard

**Today.** `upsert_notification` in `crates/sase_core/src/notifications/store.rs`
(around lines 672–748) works like this:

1. It takes the exclusive lock and reads every row, dismissed ones included.
2. It matches the newest row by `(sender, dedup_key)` through
   `find_newest_matching_index`, with no dismissed filter.
3. It appends the +1 (`apply_prepared_plus_one`).
4. It reads the file a second time, rewrites the whole store
   (`merge_and_rewrite_notifications_unlocked` → `write_notifications_atomic`), and
   bumps `notifications.jsonl.generation`.

The by-id/by-key `append_notification_plus_one` (around lines 633–666) follows the same
pattern. A dismissed row is not revived, but every +1 still rewrites the store. Athena
showed 16 rewrites per minute of a 7.4 MB file.

**Required changes.**

1. Add request fields to `NotificationUpsertRequestWire` and
   `NotificationPlusOneRequestWire` in `crates/sase_core/src/notifications/wire.rs`. Use
   `#[serde(default)]` so older callers keep working:
   - a dismissed-row opt-in, for example `plus_one_dismissed: bool`, defaulting to
     false;
   - a coalesce window, for example `coalesce_window_seconds: Option<u64>`. When it is
     omitted, a documented default applies; choose and state it, around 10 minutes. `0`
     disables coalescing.
2. When the matched row is dismissed and the caller did not opt in, return a new no-op
   action (for example `skipped_dismissed`). Do not append the +1, rewrite the file, or
   bump the generation. The match still counts as a match, so the call must **not**
   create a new visible row.
3. Coalesce when the matched row's last +1 has the same sender and the same normalized
   note, and its timestamp falls within the window. Return a no-op action (for example
   `coalesced`) with no write and no generation bump.
4. Only real appends and creates take the existing rewrite path. Decide whether the
   no-op check can run before the second full read. It must not race a concurrent
   writer: keep the decision under the exclusive lock.
5. Update the PyO3 binding (`crates/sase_core_py/src/notifications/mod.rs`) and its
   tests. Update the Rust parity tests that assert today's behavior, such as
   `notification_plus_one_by_key_picks_newest_including_dismissed` and
   `notification_upsert_matches_dismissed_and_snoozed_rows` in
   `crates/sase_core/tests/integration/notification_store_parity/plus_one_upsert.rs`, so
   they cover both the default and the opt-in. Add tests for coalescing inside and
   outside the window, for no generation bump on a no-op, and for the cap still holding.
6. Python adoption:
   - `src/sase/core/notification_store_wire.py`: mirror the new fields and actions.
   - `src/sase/notifications/store.py` (`upsert_notification`,
     `append_notification_plus_one`): accept the new keyword arguments. On a no-op
     outcome, skip `_invalidate_load_cache()` and pending-action registration.
   - `src/sase/core/agent_artifact_run_prune_publish.py`: it maps every non-`created`
     action to `plus_oned`; handle the new actions.
   - `src/sase/notifications/cli_plus_one.py`: an explicit `sase notify` +1 by id is a
     deliberate user action, so opt in to dismissed rows there.
   - `src/sase/main/notify_handler.py` prints the action as JSON; document the new
     values.
   - Audit every other `upsert_notification` caller. A survey found none that relies on
     +1s landing on dismissed rows. `agent/auto_restart/_notifications.py` deliberately
     rolls over to a new key when the top row is dismissed. Confirm that, and change a
     caller only where a test proves it needs the opt-in.
7. Move the `sase-core-revision.txt` pin. Update `docs/reference/cli/notify.md` and
   `docs/reference/automation/notification-cli.md`.

**Tests.**

- A +1 on a dismissed row is a no-op (no rewrite, generation unchanged) unless the
  caller opts in.
- Identical +1s within the window coalesce; outside the window they append.
- Python store tests cover the new actions and the skipped cache invalidation.

## Phase `held_settle`: settle once per evidence fingerprint, +1 only on change

**Loop today.** `stale_running_sweep` calls `repair_leaked_monitor_claim`
(`src/sase/ace/scheduler/stale_running_leak.py`) for every pinned `ace-monitor` claim
whose pid is dead and whose retention record is an unresolved retain. Every pass then:

1. publishes stray decision sheets, which runs a git status over the plans repo;
2. runs `release_monitor_claim` (`src/sase/monitor/_settlement_retention.py`), which
   calls `settle_stranded_workspace` (`src/sase/turns/_workspace_retention/settle.py`);
3. re-probes `git status --porcelain --untracked-files=all` in every repo and
   re-snapshots worktree diffs (5.6 MB per pass for one apollo workspace);
4. re-pins the claim and rewrites the retention record;
5. calls `notify_retained_workspace` (`src/sase/turns/workspace_retention_surface.py`),
   which upserts on the fixed key
   `workspace-retention:<project>:<artifacts_ts>:<workspace_num>` with an identical
   `plus_one_note`;
6. logs "retained with other dirt."

**Required changes.**

1. **Persist evidence with the retention record.** When `settle_stranded_workspace`
   retains, store an evidence block in the record (`records.py` `persist_record` and the
   reader in `src/sase/workspace_provider/retention_record.py`) with three parts:
   - an evidence fingerprint: a stable hash over the policy reason, the dirt
     classification, affected repo ids, normalized probe errors, and each repo's sorted
     dirty-path set;
   - a cheap stat signature per repo, with no subprocess: for example the mtime and size
     of the resolved gitdir's `index` and `HEAD`, plus the checkout root's mtime;
   - `probed_at`.

   Older records without the block simply settle once more.

2. **Skip unchanged passes.** Before publishing decision sheets or re-settling, the
   leak-repair path reads the record and returns "still held". It must do no git
   subprocess, snapshot, pin call, record write, or notification work when all of these
   hold:
   - the record is still an unresolved retain;
   - the exact claim still holds;
   - every stat signature is unchanged;
   - `now − probed_at` is less than a re-probe interval (a constant; 30 minutes unless
     measurement argues otherwise, stated in the code and docs).

   The `--no-optional-locks` git probe does not touch the index, so probing does not
   perturb the signature.

3. **Re-probe soundly.** When the signature changes or the interval elapses, run the
   normal probe:
   - If the workspace is now clean, or the decision changes, settle normally: release,
     or re-settle with a new snapshot.
   - If the new evidence fingerprint equals the stored one, update only `probed_at` and
     the signature. Skip the snapshot, the pin, and the notification.

   A workspace that becomes clean or changes evidence is released or re-settled within
   one re-probe interval. Custody rules are unchanged, and a failed probe never
   releases.

4. **+1 only on change.** `notify_retained_workspace` receives why it is being called:
   new hold, evidence or reason changed, or heartbeat. It keeps a small notify-state
   marker beside the retention record (suggested; the worker may choose another
   location, but must not churn the custody record hourly). The marker holds the
   notified fingerprint and reason, the last +1 time, and a dismissed latch.
   - A +1 is sent only when the reason or fingerprint changed. The note text names what
     changed.
   - **Heartbeat (`held_heartbeat = yes`).** At most one "Workspace still held
     (<reason>)" +1 per hour per hold. The skip path in step 2 checks the marker's
     timing, which needs no git work.
   - If the store returns the `store_guard` dismissed no-op, set the latch and send no
     further heartbeats or change +1s for that hold. Store compaction archives old
     dismissed rows. A later upsert would otherwise find no match and create a fresh
     visible row; the latch prevents that resurrection. Do not opt in to dismissed-row
     +1s here.
5. One-shot settlement callers keep their behavior but now record the evidence block, so
   later leak-repair passes can skip. These callers are `gate_turn/start_claim.py`,
   `gate_turn/transaction_retention.py`, `axe/_run_agent_runner_shutdown_finalize.py`,
   and `axe/run_agent_exec_staged_retention.py`.
6. Downgrade the per-pass "retained with other dirt" log to one line per change, and log
   a skip summary at most once per pass.

**Tests.**

- N consecutive cleanup passes over an unchanged held workspace cause zero store
  rewrites (generation unchanged) and zero git subprocess calls after the first,
  heartbeat aside. Patch `subprocess.run` and count.
- Touching the gitdir `index`/`HEAD` triggers a re-probe on the next pass.
- A workspace cleaned without touching the index is released once the interval elapses.
- A changed dirty-path set re-snapshots and sends exactly one +1.
- The heartbeat sends one +1 after an hour on an undismissed row, and none on a
  dismissed row.
- The dismissed latch survives the row being archived.
- Existing tests that assert `plus_one_count >= 1` after a second pass
  (`tests/test_stale_running_cleanup_monitor_claims.py`,
  `tests/monitor/test_monitor_retention_lifecycle_baselines.py`) are updated to the new
  semantics.

**Measurement.** Take these read-only on athena before changing anything, and record the
after-install command:

- notification-store generation advance over 60 s (the audit saw about 16 per minute);
- `stale_running_cleanup` runtime from the scheduler logs;
- "retained with other dirt" lines per hour.

The athena target is that store generations no longer advance on every cleanup pass.
Update `docs/reference/agents/workspaces.md`.

## Phase `starter_scope`: starter-inherited repo lists (item 2.4)

One apollo workspace was held for 14 days by a probe error that named a path in a
_different_ workspace. Closed sase-1rg.3 states that starter evidence folds in only for
same-workspace starters, and that foreign starter records outside the monitor checkout
are dropped.

1. Check whether that guarantee also covers starter-inherited repo lists. These are the
   opened-repo markers, `carried_repositories`, and baseline records that
   `collect_monitor_repo_observations` (`src/sase/monitor/settlement_fresh_owner.py`)
   and `discover_workspace_retention_repos`
   (`src/sase/turns/_workspace_retention/discover.py`) take from the starter's evidence
   dir. A linked-repo path under another `sase_<N>` checkout is the suspect case.
2. Read the apollo case read-only (its retention record's `probe_errors`, and the dates
   relative to sase-1rg.3's Nov 6 landing) to tell whether it predates the fix.
3. If a gap exists, drop or re-root foreign paths. Add a regression test with a starter
   whose opened linked repo lives in another workspace.
4. If the guarantee already covers the case, add the regression test that proves it.
5. Record the finding on this phase bead and as a note on sase-1rg.3.

## Phase `unknown_sizes`: unknown is not zero (items 1.1 and 1.4 data)

**Today.**

- `BoundedTreeSizer.size()` in `src/sase/core/disk_footprint_utils.py` returns `0` with
  only a diagnostic once the shared `InventoryScanBudget` is exhausted.
- A `du` timeout falls back to a truncated walk whose partial byte count is reported as
  exact.
- The Rust classifier (`crates/sase_core/src/disk_inventory.rs`, wire v1) has a
  non-optional `size_bytes: u64` and no way to say "unknown".
- Rows therefore read `0 B, coverage: complete`. On apollo, 38 workspace rows read 0 B
  while `du` found ~54 GiB; on athena, 43 of 43 workspace rows read 0 B.

**Required changes.**

1. The sizer returns a measurement with a size state: `measured`, `lower_bound` (du
   timed out and the bounded walk was truncated, so bytes are a floor), or `unmeasured`
   (budget exhausted; bytes unknown). Keep the injectable `tree_size_fn` seam working
   for existing tests, with plain ints treated as `measured`.
2. `DiskFootprintRow` (`src/sase/core/disk_footprint_models.py`) gains the size state.
   Any non-`measured` row reports `coverage: partial` and a diagnostic naming why.
3. **sase-core wire v2.** Add a per-row size state that defaults to `measured`. Make a
   non-measured row force row coverage to `partial`. Report:
   - `unmeasured_owners` (stable labels);
   - whether the totals are lower bounds.

   Keep `size_bytes` numeric (0 for unmeasured) but never counted as an exact zero.
   Update `crates/sase_core/src/disk_pressure.rs` so top-owner rows never present an
   unmeasured row as a small owner. Bump `DISK_INVENTORY_WIRE_SCHEMA_VERSION` to 2 in
   Rust, in `src/sase/core/disk_inventory.py`, and in `tools/validate_sase_core_rs`.

4. **Unattributed math.** `_unattributed_row`
   (`src/sase/core/disk_footprint_inventory_collect.py`) must not present unmeasured
   owners' bytes as unattributed. When owners are unmeasured, label the residual as
   space outside measured rows, say it includes N unmeasured owners, and name them.
   Totals read like "≥X GiB attributed; N owners unmeasured."
5. **Rendering.** `sase disk list` (`src/sase/main/disk_handler.py`
   `_print_disk_report`) shows `unmeasured` or `≥X` instead of `0 B`, and the footer
   states the lower-bound total and the unmeasured-owner count. `--json` carries the new
   fields. Check `largest_unowned_rows` and the doctor next-steps text
   (`src/sase/doctor/checks_resources_disk.py`) so they never report an unmeasured row
   as 0.
6. Move the pin. Update `docs/reference/cli/disk.md` and
   `docs/reference/disk-cleanup.md`.

**Tests.**

- With the real `BoundedTreeSizer` and a tiny budget (for example `max_nodes=1` or
  `max_seconds=0`), rows come back unmeasured/partial, never `0 B` complete.
- A du timeout with a truncated walk yields `lower_bound`.
- The classifier round-trips v2 rows through the real binding.
- The unattributed row's label and amount follow the rules above.

## Phase `owner_first`: measure workspace and owner roots first (item 1.2)

1. Reorder `collect_disk_footprint` (`src/sase/core/disk_footprint_inventory_collect.py`
   with `disk_footprint_inventory_rows.py`):
   - First, size workspace checkouts (including each project's shared primary) and owner
     roots: the fixed `sase_home` rows, per-project `artifacts/ace-run`, and each
     managed-tmp root as a whole. Use an **owner budget** separate from the shared stray
     budget, with one `du -skx` per root under a per-root timeout and an overall
     deadline. Bounded concurrency is allowed. Choose the numbers, justify them, and
     document them.
   - Then size managed-tmp buckets and heavy workspace sub-rows (`.git/objects`,
     `.venv`, `target`, …) with what budget remains.
   - Only then run the stray walk under the shared budget.

   A root that still cannot be measured uses `unknown_sizes`'s unmeasured or lower-bound
   state. Large trees (artifact runs, cargo targets, uv tools) must not starve workspace
   roots. Make the function's docstring true.

2. Do not bound workspace _discovery_ (`collect_workspace_inventory`); that is
   `sase-zw.8.7.8.4`. Size only what discovery returns.
3. Replace the misleading test in `tests/core/test_disk_footprint_inventory.py` ("budget
   exhaustion clips stray walk but keeps workspace rows"). It injects `tree_size_fn` and
   so never exercises `BoundedTreeSizer`. Add a test that uses the real sizer: an
   exhausted stray budget still measures workspace roots.
4. **Measurement and acceptance.** Read-only on apollo and athena:
   - Record `sase disk list --json` workspace rows against `du -skx` of each workspace
     root.
   - State the tolerance the new code meets, accounting for hardlinks and shared Git
     objects, and the command to repeat after install.
   - Acceptance: no unmeasured row shows as `0 B, coverage: complete`, and workspace
     rows track `du` within the stated tolerance.
5. Record the sase-1bf.4 regression finding as a note on sase-1bf.4, and the overlap
   note on sase-zw.8 (see Coordination).

## Phase `pressure_recovery`: recovery independent of attribution, honest notice, escalation (items 1.3–1.4)

**Today** (`src/sase/scripts/sase_chop_disk_pressure.py`):

- The chop calls `collect_disk_footprint()` unguarded _before_ `run_disk_reap`. An
  inventory exception, or a slow inventory, aborts or delays recovery, even though the
  reap never reads the inventory.
- The notice goes through `notify_workflow_complete`, which has no dedup key, so every
  hourly pass under pressure adds a row (119 all-time on apollo).
- The notice leads with "Unattributed N".
- A missed recovery target shows only as `reason=recovery_blocked` in the chop summary.
  On Nov 16 apollo reclaimed 9.7 KiB against a 9.6 GiB target, because the space sat in
  owners that pressure recovery does not reap (workspaces, artifact runs, the
  rust-prebuild cache) and in managed-tmp entries whose inspection was incomplete.

**Required changes.**

1. Run the pressure reap first. Then collect the inventory under a fail-open guard: an
   exception or timeout becomes "attribution unavailable" in the notice and never skips
   or delays recovery. The `managed_tmp_reap` chop and `sase disk reap` must likewise
   never depend on inventory completeness. Add a test proving each.
2. **Notice.** Lead with free space and the recovery result. Name unmeasured owners from
   `unknown_sizes`'s report fields. List only measured top owners, and label the
   residual as `unknown_sizes` defines.
3. **Stable dedup key.** Persist per-filesystem pressure-episode state. An episode
   starts when pressure first becomes active on a filesystem and ends on `space_ok`.
   Upsert the notice with an episode-scoped key such as `disk_pressure:<fs>:<episode>`
   and a +1 per pass. A new episode gets a new visible row.
4. **Escalation.** When available-after is still below the recovery target, upsert a
   _distinct_ escalation row under its own episode-scoped key. It states:
   - the target, the bytes reclaimed, and the space available after;
   - what blocked recovery: blocked or incomplete owners with their reason counts from
     the structured cleanup results;
   - the largest measured owners that pressure recovery does not reap, with their
     reclaim commands.

   Make the chop's structured result `check_error` for a missed target, and check
   sase-11h note #1's reason→`no_op` mapping (see Coordination).

5. Do not change managed-tmp liveness or deletion-safety rules. The escalation reports
   liveness-incomplete counts, and those rules remain `sase-zw.8.7.8.1`'s scope. Note
   the overlap on sase-zw.8.

**Tests.**

- An inventory that raises, or that exhausts its budget, still lets the reap run.
- The notice names unmeasured owners.
- A missed target produces the escalation with target, reclaimed bytes, and blockers.
- A second pass in the same episode adds a +1 rather than a new row.
- `space_ok` ends the episode, and the next episode creates a new row.

Update `docs/reference/disk-cleanup.md`,
`docs/reference/automation/notification-types.md`, and
`docs/reference/automation/scheduler-housekeeping.md`. Record apollo's current
`disk_pressure` row count and the after-install command.

## Phase `standing_skips`: one standing state for un-inspectable reaper entries (item 1.5)

Athena logged ~195 recent "could not fully inspect" warnings that keep preserving one
agent-tmp dir, `agent-tmp/gh_bobs-org__bob-cli-ws0-261007_081241`, whose newest write
time is unobservable. Similar repeats come from unreadable `/proc/<pid>/environ` (for
example `systemd --user`). `src/sase/scripts/sase_chop_managed_tmp_reap.py` logs the
first five `skip_reasons` per root at warning level on every run.

1. Keep refusing to reap these entries; the Rust reaper's safety rules do not change.
2. Persist a small standing-skip state for the chop, keyed by root, entry, and
   normalized reason class.
   - Warn once when a skip first appears or its reason class changes.
   - Log once at info when it clears.
   - Otherwise log a single summary line ("N standing skips unchanged") and keep the
     per-entry detail at debug.
3. Apply the same treatment to repeated unreadable-`environ` skips.

**Tests.** Three passes with the same skip give one warning. A changed reason warns
again. A cleared skip logs once. State-file corruption fails open to today's logging.
Record athena's warning count per day before the change and the after-install command.

## Landing (child epic land agent)

1. Confirm every phase bead is closed and its notes are accounted for.
2. Confirm the coordination notes exist on sase-zw.8, sase-1bf.4, sase-1rg.3, and (if
   applicable) sase-11h, and that each measuring phase recorded its before-numbers and
   after-install command on sase-1tz.1.
3. Run `just check` on the combined tree. Confirm the `sase-core-revision.txt` pin
   includes every sase-core commit from `store_guard` and `unknown_sizes`.
4. Triage the phases' `PROPOSED FOLLOW-UP:` notes.
5. Close **sase-1tz.1** with a note summarizing the verified result.

Never close the parent epic **sase-1tz** or any other ancestor.
