---
tier: epic
title: Gateway checkpoint fix, notification hygiene, and retention sweep (items 11-13)
goal:
  A busy SQLite WAL checkpoint reports busy instead of erroring, read paths checkpoint
  passively, and the gateway serves a cached fleet snapshot at once while one rebuild
  runs in the background. remote-attention keeps one row per request, wait_checks raises
  one notice per failed root blocker with a one-key expire action, and task-bead triage
  keeps one notification per bead. Settled triage gate bundles are reaped. Notification
  payloads over 8 KB move to sidecar files, and broken file references are pruned.
  file-hooks successes are silent, workspace-rescue dedups, prune nudges have a size
  floor, agent_state_refresh logs each unchanged verdict once, and no-op claim-ledger
  rows stop. Tool-run logs, tui.log, proc log locks, registry temp files, and index
  backups follow stated horizons.
parent_bead: sase-1tz.4
decisions:
  index_backups:
    ask: Should an age-based reaper delete stale agent-index .bak backups?
    choices:
      report:
        Backups get a human-action disk row and a gated one-time delete; nothing deletes
        them automatically
      reap: An age-based reaper deletes agent-index backups older than 30 days
    default: report
    why:
      A sase-zw gate already declined deleting backups, and no SASE code writes these
      files
    answer: report
  quiet_triage_rule:
    ask:
      Remove the quiet-task-beads rule from the chezmoi config once the triage-volume
      fix lands?
    choices:
      remove: Beads-tab toasts and sounds return once the human runs chezmoi apply
      keep: The rule stays, and the phase records per-bead volume for a later call
    default: remove
    why: The rule's own text says to remove it once the triage-volume fix is in
    answer: remove
phases:
  - id: wal_busy
    title:
      Treat SQLite's busy -1 checkpoint result as busy and checkpoint passively on read
      paths
    description:
      "wal_busy: in sase-core, a busy wal_checkpoint that returns -1 counts reports
      checkpoint_busy instead of failing u64 conversion. The gateway's post-build hook
      uses PASSIVE and runs after waiters are woken. Maintenance keeps TRUNCATE. Move
      the pin."
    size: small
    depends_on: []
  - id: gateway_swr
    title: Serve cached fleet snapshots stale-while-revalidate in the gateway
    description:
      "gateway_swr: an expired cached snapshot is served at once, marked stale, while
      one background rebuild runs. Index-permit waits are bounded. A cold cache keeps
      today's bounded wait. Record the relationship on sase-1s5 and move the pin."
    size: medium
    depends_on:
      - wal_busy
  - id: attention_key
    title: Key remote-attention rows by stable request identity
    description:
      "attention_key: remote-attention rows key on (origin, request_id). A new revision
      updates and resurfaces the one row. An identical re-emission writes nothing. A
      local dismissal still sticks for its revision. Legacy per-revision rows migrate
      once."
    size: medium
    depends_on: []
  - id: blob_store
    title: Add opt-in sidecar storage for oversize notification payloads in sase-core
    description:
      "blob_store: sase-core stores action_data values and notes over a caller-supplied
      byte limit as content-addressed sidecar files, with a read binding. Compaction
      garbage-collects unreferenced sidecars and prunes broken files references. Default
      behavior is unchanged. Move the pin."
    size: medium
    depends_on:
      - attention_key
  - id: blob_adopt
    title: Resolve sidecar payloads in every reader, then externalize payloads over 8 KB
    description:
      "blob_adopt: the TUI, Telegram, and mobile readers that need full action_data or
      note text resolve sidecar references. Then the Python store wrapper externalizes
      payloads over 8 KB by default, and live oversize rows migrate once."
    size: medium
    depends_on:
      - blob_store
  - id: wait_blocker
    title: Raise one wait_checks notice per failed root blocker and freeze its +1s
    description:
      "wait_blocker: terminally blocked waiters group under one notice per root blocker
      that names every direct and transitive waiter. The notice changes only when its
      waiter set changes and never +1s otherwise. Legacy per-waiter rows settle once."
    size: medium
    depends_on: []
  - id: waiter_expire
    title: Offer a one-key action that expires a failed blocker's waiters
    description:
      "waiter_expire: under blocked_waiters = offer, the root-blocker notice gets a
      one-key TUI action. It expires every waiter the notice names through the existing
      user cancel path for a waiting agent, records the blocker as the reason, and
      settles the notice. Nothing expires waiters automatically."
    size: medium
    depends_on:
      - wait_blocker
  - id: triage_volume
    title: Keep one task-triage notification per bead across presentation changes
    description:
      "triage_volume: measure which fingerprint inputs churn triage gates. Stop
      canceling and recreating a pending gate for presentation drift the gate does not
      render. Recreated gates reuse one notification row per bead. Remove or keep the
      quiet-task-beads rule per the decision."
    size: medium
    depends_on: []
  - id: gate_reconcile
    title: Settle orphaned triage gates and reap settled gate bundles
    description:
      "gate_reconcile: a pending automation-owned gate whose notification was dismissed,
      or whose bead is resolved, is cancelled. A dismissal is not undone by the next
      pass. Settled task-triage and snooze bundles older than a horizon are reaped.
      Other gate kinds are only counted."
    size: medium
    depends_on:
      - triage_volume
  - id: notify_quieting
    title:
      Silence success toasts, dedup workspace-rescue, floor prune nudges, and skip no-op
      claim writes
    description:
      "notify_quieting: file-hooks successes become one silent row per hook, and
      workspace-rescue gets a stable dedup key. Artifact-run prune nudges fire only
      above a size floor or under disk pressure. hold_workspace_claim writes no ledger
      row when nothing changed."
    size: medium
    depends_on: []
  - id: state_refresh_memo
    title: Log each unchanged agent_state_refresh verdict once
    description:
      "state_refresh_memo: agent_state_refresh persists each run's unresolved verdict
      with an evidence fingerprint. A pass that sees the same fingerprint skips the
      warning and the dismissed-bundle lookup. A changed fingerprint re-evaluates and
      logs once. Stays clear of sase-1s5.3's lookup changes."
    size: medium
    depends_on: []
  - id: log_horizons
    title:
      Apply the tool-run log horizon on a schedule, age-rotate tui.log, and verify
      digest and run pruning
    description:
      "log_horizons: an hourly housekeeping chop applies the existing tool-run retention
      owner, not only under disk pressure, and removes emptied run dirs. tui.log rotates
      by age as well as size. Verify error-digest pruning and that artifact_run_prune is
      unblocked on both hosts."
    size: medium
    depends_on: []
  - id: clutter_reapers
    title: Fix the proc log-lock and registry temp-file leaks and reap their backlog
    description:
      "clutter_reapers: deleting proc logs also removes their lock files. The
      name-registry writer sweeps its own stale temp files under its lock. Owner steps
      reap both backlogs from the log_horizons chop. Index backups follow the
      index_backups decision."
    size: medium
    depends_on:
      - log_horizons
proposed_by: bbugyi200.athena.sase-1tz.4
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-20 14:00:17
status: wip
---

- **PROMPT:**
  [prompts/202611/hygiene_gateway_notifications_retention.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/hygiene_gateway_notifications_retention.md)
- **PARENT:**
  [202611/notification_log_audit_fixes.md](https://github.com/sase-org/sase--plans/blob/main/202611/notification_log_audit_fixes.md)

# Plan: Gateway checkpoint fix, notification hygiene, and retention sweep (items 11–13)

This child epic implements phase `hygiene` (**sase-1tz.4**) of parent epic **sase-1tz**,
`plan:202611/notification_log_audit_fixes.md`. It covers items 11–13, which are audit
findings F11 (gateway WAL checkpoint and snapshot waits), F12 (notification hygiene),
and F13 (log and state retention).

## Read first (every phase)

- **Audit.** The source of truth is
  `research:202611/apollo_athena_sase_notification_and_log_audit/apollo_athena_sase_notification_and_log_audit.md`
  (sections F11–F13 and ranked fixes 11–13). Read it with
  `sase artifact read <ref> "<reason>"`. If the reference resolves as missing, first
  open the research sidecar with `sase repo open sase--research -r "<why>"`. The
  `…__cdx.md`, `…__cld.md`, `…__grk.md`, `…__mus.md`, and `…__gem.md` swarm reports in
  the same directory hold the raw counts, log excerpts, and probes.
- **Final parent decisions.** The parent epic's DECISIONS are final; do not re-ask them.
  `blocked_waiters = offer` selects `waiter_expire`'s design: a human expires waiters
  with one key, and nothing expires them automatically. `held_heartbeat` and
  `codex_bundle` select nothing in this epic; phases `outage` and `defects` already
  implemented them.
- **Rust core boundary.** Shared backend behavior lives in `sase_core`. Here that means
  the agent-index WAL checkpoint and the gateway (`wal_busy`, `gateway_swr`), the
  notification store's reconcile and payload format (`attention_key`, `blob_store`), and
  the ToolRun retention policy, which `log_horizons` calls but does not change. Open
  sase-core with `sase repo open sase-core -r "<why>"` and work only in the printed
  path. A phase that changes sase-core code sase builds or calls moves
  `sase-core-revision.txt` (see `docs/rust_backend.md`). Pin a sase-core commit that
  contains every sase-core change already landed by earlier phases; never move the pin
  backwards. The wait_checks chop, gate bundles, triage, file hooks, rescue,
  agent_state_refresh, the claim ledger, and the disk owner steps stay in Python, where
  they live today.
- **Other repos.** `blob_adopt` changes sase-telegram's `action_data` readers. Open it
  with `sase repo open sase-telegram -r "<why>"`, and run its guarded check as
  `sase tool run check` from that checkout (see its `AGENTS.md`). `triage_volume` edits
  the chezmoi source only under `quiet_triage_rule = remove`. Open it with
  `sase repo open chezmoi -r "<why>"`. Never reach these repos by another path.
- **Verification.** `just check` is the agent verification recipe. Read the
  `lint_and_test.md` reference memory before finishing. Run `just check-full` only when
  explicitly instructed. Every behavior change gets focused tests, including the
  regression scenarios named in its phase. sase-core changes run sase-core's own check
  from its checkout (see its `AGENTS.md`).
- **Live hosts are read-only.** Global install and `chezmoi apply` are human-only, so
  live acceptance happens after the user installs. Phases may take read-only
  measurements on apollo and athena. The `tailnet.md` reference memory covers SSH
  access. Each measuring phase records two things as a note on its own phase bead: the
  exact before-numbers, and the command to repeat after install. It also adds a one-line
  summary note on **sase-1tz.4**. Never dismiss notifications, cancel gates, repair
  state, or delete files on either host. The reapers in this epic do their one-time
  cleanup on their first run after install. Propose any other one-time live action
  through `/sase_gate`.
- **Discovered work.** Phase workers are epic phase workers. Record follow-up work as
  `PROPOSED FOLLOW-UP:` notes on your own phase bead, never as new task beads.
- **Docs, config, and memory.** User-visible behavior changes update the matching page
  under `docs/`. Read the `docs_writing.md` reference memory first. New config keys get
  defaults in `src/sase/default_config.yml` and the config schema. Read `sase_flags.md`
  before changing user-reaching behavior whose old branch must stay reachable, and
  `cli_rules.md` before adding a CLI subcommand or option. Read `tui.md` before changing
  the TUI, and `ace.md` before changing anything under `src/sase/ace/`. Any phase that
  touches `tools/` reads `tools.md` first. A survey of the always-loaded notes and the
  `dispatch.md`, `ace.md`, and `tui.md` reference notes found no memory note that this
  epic makes false, so this plan authorizes no memory edits. If a phase finds such a
  note, it records a `PROPOSED FOLLOW-UP:` and leaves memory alone.
- **Already landed: build on it, don't rebuild it.**
  - Phase `outage`'s store guard (sase-core `d8681dbb`, pinned).
    `append_notification_plus_one` and `upsert_notification` coalesce an identical +1
    within 600 s (`NOTIFICATION_PLUS_ONE_COALESCE_WINDOW_SECONDS`). They skip a +1 on a
    dismissed row with no store write unless the caller passes
    `plus_one_dismissed=True`. Python passes `plus_one_dismissed=` and
    `coalesce_window_seconds=` (`src/sase/notifications/store.py:170-244`).
  - Phase `outage` gave `disk_pressure` episode-scoped dedup keys
    (`src/sase/core/disk_pressure_recovery.py:83-88`). This epic only handles
    `workspace-rescue`.
  - Phase `noise` added error-digest pruning in `96b3716cb3`. It prunes after 30 days,
    keeps the newest 50, and runs at most daily (`src/sase/axe/error_digest.py:96-171`,
    called from `sase_chop_error_digest.py:38-42`). `log_horizons` only verifies it.
  - Phase `defects` (`b6f6c63e3e`) made bead_task_triage cancel the stale gate and
    advance the generation on `request_id_conflict`
    (`src/sase/scripts/sase_chop_bead_task_triage.py:519-573`). `triage_volume` and
    `gate_reconcile` keep that branch intact.
  - `b0de76422d` skips projects that SASE does not manage when collecting artifact-run
    protections.

## Baseline (from the audit and read-only code reading, 2026-11-19/20)

- **F11.** sase-core's `checkpoint_agent_artifact_index_wal_if_oversized`
  (`crates/sase_core/src/agent_scan/index/checkpoint.rs:53-88`) runs
  `PRAGMA wal_checkpoint(TRUNCATE)` and passes the -1 counts from a busy checkpoint to
  `u64::try_from` (`:83-86`). It errors instead of reporting busy, and the existing busy
  test (`tests/checkpoint.rs:148`) only covers non-negative counts. Apollo's gateway
  logged presentation snapshot builds of 158 s and about 30 min, and waiters on an empty
  cache got 504s.
- **F12.**
  - Athena has 1,615 remote-attention rows all-time, including duplicate pairs with
    identical revisions.
  - Athena shows 4,590 wait_checks +1s across its last 50 rows, and 412 "can never
    self-resolve" rows.
  - `interaction_requests/` holds about 2,500 dirs on apollo and 5,030 on athena. A
    read-only census on athena on 2026-11-20 found 5,104 `task_triage` dirs: 4,461
    cancelled, 217 responded, and 426 pending. All 426 pending dirs are tracked, with
    live undismissed notifications. The top cancel reasons were
    `task_triage_presentation_changed` (2,139), `task_bead_no_longer_ready` (1,586), and
    `bead_closed` (621). So the backlog is mostly settled bundles that nothing deletes,
    plus gate churn.
  - Notification payload size:
    - `action_data` is over half the active store. Athena's `JumpToAgent` `prompt`
      values total 1.64 MB live and 64 MB in the 111 MB archive.
    - Notes hold up to 170–218 KB.
    - 543 of 3,441 `files` references on athena are broken.
  - 210 of 212 apollo `file-hooks` rows are ✅ successes, and 117 of 135
    `workspace-rescue` rows have no dedup key.
  - Prune nudges fire for 873 KiB on a host with 283 GiB free.
  - agent_state_refresh logs about 1,000 redundant warning lines a day per host, up to
    246 per agent.
  - Thousands of "already pinned (no-op)" claim-ledger rows are written each night.
- **F13.**
  - Athena's `~/.sase/tools/logs` is 1.3 GiB. 4,476 of its 8,068 run dirs are older than
    30 days (511 MiB), and 1,356 are empty.
  - Proc log locks: 6,716 `.log.lock` files on apollo; 13,702 on athena against 589
    logs.
  - Orphaned `.agent_name_registry.json.*.tmp` files: 3 (22.9 MB) on apollo, 9 (85.6 MB)
    on athena.
  - A 222 MiB `agent_artifact_index.sqlite.pre-matched-deploy-20260927T064040Z.bak` on
    athena.
  - `tui.log`'s rotated segments date from Oct 10–11.
  - Athena's latest `artifact_run_prune` run showed `unavailable=0`, `status=ok`, and
    every candidate protected.

## Coordination with open work

- **sase-1s5** (open epic; phase .3 is open) speeds up presentation builds and owns
  agent_state_refresh's dismissed-bundle lookup (`_locate_dismissed_bundles_for_run`,
  `src/sase/agents_sync/_publication_state_repair_observations_runs.py:214-253`). It
  also owns `filesystem_observations_for_run` (`:308-370`). It changes none of
  `refresh_scope`, `wait_for_build`, `timeout_response`, or `checkpoint.rs`.
  `gateway_swr` complements it: builds get faster under sase-1s5, and waiters stop
  waiting on builds here. `gateway_swr` records that relationship as a note on sase-1s5.
  `state_refresh_memo` leaves those two functions alone and notes its overlap on
  sase-1s5.3.
- **sase-zw / sase-zw.8** (in progress) own disk retention. sase-zw.8 note #3 rules out
  a second tool-run reaper and asks that the `tool_run_retention` owner be kept, so
  `log_horizons` schedules that owner and adds no policy. sase-zw note #2 records that
  backups were _not_ approved for deletion, which is why `index_backups` defaults to
  `report`. `log_horizons` and `clutter_reapers` note their overlap on sase-zw.8 and
  leave managed-temp safety and inventory discovery to sase-zw.8.7.8.
- **sase-168.1** (closed) guarantees that a local remote-attention dismissal sticks for
  its revision. `attention_key` keeps that guarantee and its tests.
- **sase-ir** (closed) added `_reconcile`'s inactive-project sweep in
  `sase_chop_bead_task_triage.py`. `gate_reconcile` builds on `_reconcile`; it does not
  add a second triage loop.
- **sase-115** (rowless proc logs) was canceled. `clutter_reapers` handles only lock
  files and does not reopen it.

## Phase `wal_busy`: busy checkpoints report busy (item 11, F11)

**Today.**

- `AgentArtifactIndexWalCheckpointWire` (`checkpoint.rs:17-33`) has `checkpoint_busy`
  and two `u64` frame counts.
- The gateway calls the checkpoint with TRUNCATE after every successful presentation
  build, in `run_scoped_build` (`crates/sase_gateway/src/fleet_reads/service.rs:1309`,
  `:1334`). That call sits inside `spawn_blocking`, holds an index permit, and runs
  before waiters are woken.
- An `Err` logs "fleet index WAL checkpoint failed" (`:1347-1352`).
- Maintenance calls it from `terminalize_stale_active_agent_artifact_index_rows_with`
  (`crates/sase_core/src/agent_scan/index/maintenance.rs:217`), which discards the
  result. Python reaches that path through the `terminalize_stale_active_*` binding.

**Required changes.**

1. **Busy result.** When SQLite returns -1 for the log or checkpointed frame count,
   report `checkpoint_busy = true` and treat the counts as unknown, for example with
   `Option<u64>`, or by clamping to 0 with an explicit unknown flag. Never return `Err`.
   Find every consumer of the wire, in Rust and through any binding, and keep the change
   backward-compatible for each one. Prefer an additive field over a type change if
   anything serializes the wire.
2. **Passive on read paths.** Add a mode parameter. The gateway's post-build hook uses
   `PASSIVE`, which never blocks writers, and runs after the build result has woken its
   waiters and released the index permit. The maintenance write path keeps `TRUNCATE`
   and logs, rather than discards, a busy or failed result once per call.
3. **Logging.** The gateway logs a busy checkpoint at debug or info with the WAL size,
   not as a warning.
4. **Tests.** Force a real busy `TRUNCATE` that yields -1, for example with a second
   connection holding a write transaction, and assert `checkpoint_busy` with no error.
   Assert that the gateway hook runs after waiters are woken, and that PASSIVE is used
   on the read path. Keep `checkpoint_hook_runs_after_presentation_success_only`
   (`fleet_reads/tests/refresh.rs:472`) passing.
5. **Pin and docs.** Move the pin. Update the "Fleet Snapshot Logging" section of
   `crates/sase_gateway/README.md` (`:289-300`).
6. **Measure.** Before: count "out of range integral type conversion attempted" and
   "fleet index WAL checkpoint failed" lines in each host's gateway log over 7 days,
   read-only, and record the `agent_artifact_index.sqlite-wal` size. Record the
   after-install command.

## Phase `gateway_swr`: never block waiters on a slow rebuild (item 11, F11)

**Today.**

- Each scope has a `ScopeState` with a cache, a single in-flight build, and a back-off
  (`service.rs:128-136`).
- After `FLEET_SNAPSHOT_STALE_SECONDS = 60` (`resolution.rs:43`), the cache counts as a
  miss (`unexpired_scoped_snapshot`, `:1152-1165`).
- Every caller then joins the build and waits up to `SNAPSHOT_REFRESH_TIMEOUT = 4s`
  (`:65`, `wait_for_build` `:1019`) before `timeout_response` returns the cached copy
  marked stale (`:1117-1132`).
- With no cache, the response is `FleetReadError::Timeout`, which becomes a 504
  (`routes/errors.rs:453-458,495`).
- `run_scoped_build` waits for one of two index permits with no timeout (`:1315`).

**Required changes.**

1. **Stale-while-revalidate.** For an ordinary read, a cached snapshot older than the
   freshness window is returned at once, marked stale with its age and a `revalidating`
   reason. The existing single-flight build starts in the background if none is running.
   A finished build fills the cache as it does today. Choose and document an upper bound
   on how stale a served snapshot may be. Past it, a reader waits the bounded refresh
   timeout as today, and still gets the stale copy rather than an error if the build
   does not finish.
2. **Forced refresh unchanged.** Explicit forced refresh (`reconcile`, `:411-415`) keeps
   its join-and-wait contract, because its caller asked for fresh data.
3. **Cold cache.** With no cache at all, keep today's bounded wait and timeout. If the
   measurement shows cold-cache 504s matter, for example after gateway restarts, record
   a `PROPOSED FOLLOW-UP:` to persist the last snapshot instead of building that here.
4. **Bounded permit waits.** A build waiting on an index permit gives up within a
   bounded time and records a back-off reason. A queued build must not hold the
   in-flight slot forever.
5. **Clients.** Check how the Python client
   (`src/sase/ace/tui/models/_fleet_agents_payload.py` and
   `tests/ace/tui/test_fleet_agents_feed_honesty.py`) and the mobile client show
   staleness. The new reason must render as stale data, not as an error.
6. **Tests** in `crates/sase_gateway/src/fleet_reads/tests/refresh.rs`:
   - an expired cache is served at once while exactly one rebuild runs;
   - a slow build never delays an ordinary reader past a small bound;
   - a forced refresh still waits;
   - a cold cache still times out with a bound;
   - the stale bound is honored;
   - a permit wait gives up.

   Keep `slow_build_stampede_starts_exactly_one_build` and its siblings green.

7. **Pin, docs, coordination.** Move the pin. Update
   `docs/reference/integrations/mobile-gateway.md`, the stale-cache text in
   `docs/reference/integrations/remote-dispatch.md` (`:246-249,301`), and the gateway
   README. Add a note on **sase-1s5** saying this phase changes how waiters are served,
   not how long builds take.
8. **Measure.** Before, read-only on apollo and athena: the count and maximum of
   presentation builds over 60 s, and the count of 504 responses or `Timeout` errors,
   over 7 days. Record the after-install command.

## Phase `attention_key`: one remote-attention row per request (item 12.1, F12)

**Today.**

- `_dedup_key` (`src/sase/dispatch/attention_inbox.py:555-561`) returns
  `remote-attention:{origin}:{request_id}:{revision}`, and the row id is a uuid5 of that
  key (`:387`).
- `reconcile_remote_attention_inbox` (`:138-237`) calls `reconcile_notification_rows`
  with the reversible marker `remote_attention_auto_dismissed`. It auto-dismisses older
  revisions of the same request (`:197-207`) and rows a complete host no longer lists
  (`:208-217`).
- sase-core's `reconcile_notification_rows`
  (`crates/sase_core/src/notifications/store.rs:1221-1338`) keeps an on-disk dismissal
  that has no marker (`:1290-1295`).
- The docs promise that a new revision makes a new unread row
  (`docs/reference/automation/notifications.md:23-39`).

**Required changes.**

1. **Stable key.** Key rows on `remote-attention:{origin}:{request_id}` and derive the
   row id from it. Keep the revision in `action_data["remote_attention_revision"]`.
2. **Resurface on a new revision.** Add a reconcile option in sase-core that names an
   `action_data` key, for example a resurface-on-change key. When the incoming value
   differs from the stored value, the row takes the incoming read and dismissed state,
   so it becomes unread and undismissed. When the values are equal, a user dismissal
   sticks as it does today. This keeps sase-168.1's guarantee: a dismissal sticks for
   its revision. Update the binding and the Python wire.
3. **No-op re-emission.** Reconciling a row identical to the stored row writes nothing
   and does not bump the store generation. Add the check in sase-core if reconcile does
   not already skip the write.
4. **Legacy migration.** On the first pass, per-revision rows for a request fold into
   the new row:
   - They are auto-dismissed with the existing reversible marker.
   - The new row inherits a user dismissal from the legacy row with the same revision,
     so nothing the user dismissed resurfaces.
   - The supersede-old-revision loop goes away once no legacy rows remain. Keep it only
     as long as the migration needs it.
5. **Tests.** Update `tests/test_dispatch_attention_inbox.py`, including the cases at
   :162, :198, :224, :256, and :345. They must show:
   - a re-emission with the same revision writes nothing;
   - a new revision updates the one row and resurfaces it;
   - a dismissal sticks for its revision;
   - the migration from legacy rows keeps user dismissals.

   Add a sase-core parity test beside
   `tests/integration/notification_store_parity/reconcile_upsert.rs`.

6. **Pin and docs.** Move the pin. Rewrite `notifications.md:23-39` for one row per
   request.
7. **Measure.** Before, read-only: remote-attention rows on both hosts, all-time and
   undismissed, plus duplicate (origin, request_id) groups. Record the after-install
   command.

## Phase `blob_store`: opt-in sidecar payloads in sase-core (item 12.4, F12)

**Today.**

- `NotificationWire` (`crates/sase_core/src/notifications/wire.rs:38-79`) stores
  `notes: Vec<String>`, `files: Vec<String>`, and `action_data: BTreeMap<String,String>`
  inline.
- The only size limit is the 2,000-character cap on +1 notes (`wire.rs:8`).
- Compaction archives dismissed rows after 3 days (`store.rs:31-34,1162-1220`) into
  `notifications-archive.jsonl`, which has no retention.
- Reconcile always keeps the on-disk `files` (`store.rs` around `:1232-1247`), and
  nothing checks that those paths exist.

**Required changes.** Default behavior must be byte-identical until `blob_adopt` opts
in, so no reader ever sees a reference it cannot resolve.

1. **Sidecar format.** Write requests (append, upsert, and reconcile rows) gain an
   optional externalize limit in bytes. When it is set, any `action_data` value or note
   longer than the limit is written once to a content-addressed file under the
   notifications directory, for example `blobs/<sha256>`, through an atomic write. The
   value is replaced by a versioned reference string that records the hash and byte
   length. A note keeps a short inline preview in front of its reference, so lists still
   render. Define the reference grammar once in Rust.
2. **Read binding.** Expose a binding that resolves a reference to its full text and
   returns a clear "missing" result for an absent sidecar. Add Python helpers in
   `src/sase/notifications/` that pass ordinary values through unchanged, for example a
   resolved `action_data` value and a resolved note.
3. **Garbage collection.** Compaction deletes sidecars that no live or archived row
   references and that are older than a short grace period, so an in-flight write is
   never collected. Add a compaction option, off by default, that externalizes oversize
   values in existing live rows.
4. **Broken `files`.** Compaction drops `files` entries whose paths no longer exist on
   rows it rewrites. Report the number dropped in the compaction result.
5. **Tests.** Cover:
   - the round trip;
   - the limit boundary;
   - dedup of identical payloads;
   - the preview;
   - a reference to a missing sidecar;
   - GC that keeps referenced and fresh sidecars;
   - pruning of broken `files` entries;
   - a write with no limit that is byte-identical to today.
6. **Pin and docs.** Move the pin. Describe the sidecar layout in
   `docs/reference/automation/notification-storage.md`.

## Phase `blob_adopt`: resolve, then externalize payloads over 8 KB (item 12.4, F12)

**Today.** These readers use `action_data` or notes:

- About 35 TUI files under `src/sase/ace/tui/`.
- sase-telegram's `inbound_handlers/project_context.py:189-191`, which reads the nested
  `prompt`.
- `formatting.py`, which reads several keys.
- sase-core's `mobile.rs:598-642`, which reads only short keys.

The largest payload sources are the full `prompt` and other fields that
`src/sase/axe/run_agent_runner_finalize.py:459-511` puts in `JumpToAgent` `action_data`,
and the unbounded `error_summary` notes (`:421-422`).

**Required changes.**

1. **Readers first.** Enumerate every reader of a key or note that can exceed 8 KB. The
   full sweep covers sase, sase-telegram, and sase-core's mobile projection. Each reader
   that needs the full value calls the resolver. A reader that only shows text may use
   the inline preview.
2. **Turn it on.** Then make the Python store wrappers pass an 8 KB limit by default.
   Run the one-time live-row migration through compaction. Archived rows stay as they
   are.
3. **Shrink at the source.** Where a sender embeds data that already lives on disk, such
   as a prompt in the agent's artifact dir, store a path instead if the readers allow
   it. Cap `error_summary` notes with a pointer to the full report.
4. **Tests.** A `JumpToAgent` row with a 200 KB prompt stores under 8 KB inline, and the
   TUI and Telegram readers get the full prompt back. Run sase-telegram's
   `sase tool run check`.
5. **Docs.** Update `notification-storage.md`.
6. **Measure.** Before, read-only: the live store size, the bytes taken by `action_data`
   and notes, rows over 8 KB, and broken `files` counts on both hosts. Record the
   after-install command.

## Phase `wait_blocker`: one notice per failed root blocker (item 12.2, F12)

**Today.**

- `upsert_terminal_blocked_wait_notification`
  (`src/sase/scripts/_chop_wait_checks_terminal.py:136-192`) keys one row per waiter
  (`wait_checks:terminal-blocked:{waiter_dir}`, `:183`). It adds a "Still blocked on…"
  +1 on every pass. The chop runs every 2 s.
- The loop in `_chop_wait_checks_run_waiters.py:445-487` handles each waiter separately.
- `TerminalBlocker` (`_chop_wait_checks_common.py:22-26`) names only the direct
  dependency.
- Stranded-bead waits already share one row per bead (`:195-299`), which is the model to
  follow.

**Required changes.**

1. **Root blocker.** Walk each terminally blocked waiter's dependency chain to its root
   terminal blocker. A waiter blocked by another waiter that is itself terminally
   blocked counts under the same root. Bound the walk and guard against cycles.
2. **One notice per root.** Key the notice on the root blocker's identity, for example
   `wait_checks:terminal-blocker:{root artifact_dir}`. The notice names the blocker, its
   outcome, and every direct and transitive waiter, with a count. Store the waiter list
   in `action_data` for `waiter_expire`.
3. **Frozen +1s.** A pass with the same waiter set writes nothing. A changed set updates
   the row in place, without a +1, and marks it unread only when a waiter joins. Rely on
   the outage store guard for any +1 that remains; never pass `plus_one_dismissed=True`.
4. **Settle.** When every waiter is gone (expired, released, or cancelled), the notice
   is dismissed with a reversible marker. Legacy per-waiter rows whose waiter is now
   covered by a root notice are auto-dismissed once, with a marker.
5. **Tests.** Extend `tests/test_axe_chop_wait_checks_terminal_outcomes.py` (keys at
   :251 and :462). Required cases:
   - N waiters on one failed blocker give one row;
   - a transitive waiter is grouped under the root;
   - repeated passes write nothing;
   - a new waiter updates the row;
   - legacy rows settle;
   - cycles terminate.
6. **Docs.** Update `docs/reference/automation/notification-types.md` (`:33`,
   `:192-212`).
7. **Measure.** Before, read-only: wait_checks rows, +1 totals, and "can never
   self-resolve" rows on both hosts. Record the after-install command.

## Phase `waiter_expire`: one key expires the blocked waiters (item 12.2, F12)

> [!note] Parent decision `blocked_waiters = offer` is final. The notice offers the
> action; nothing expires waiters automatically.

**Today.**

- There is no expire operation for waiters.
- Notification actions are dispatched by `notification.action` in
  `src/sase/ace/tui/actions/agents/_notification_dispatch.py:69-106`.
- Badges and icons live in `src/sase/ace/tui/modals/notification_modal_constants.py`.
- The existing waiting-agent controls are the TUI `unwait` (`_wait_apply.py:257-266`)
  and `cancel_waiting_marker_for_hijack` (`src/sase/axe/run_agent_wait_markers.py:188`).

**Required changes.**

1. **Operation.** Add a non-TUI Python operation that expires the waiters a root-blocker
   notice names. For each waiter it uses the same path and terminal state that a user
   cancel or kill of a waiting agent uses today, recording a reason that names the
   blocker. It never launches the waiter. It skips waiters that already started or
   settled, and reports per-waiter results. Add a terminal state only if none exists,
   and justify that in a bead note.
2. **One-key action.** Root-blocker notices declare a new action, with a TUI key, a
   badge, and a confirmation that names the number of waiters. Dispatch runs the
   operation and then settles the notice. Read `tui.md` and `ace.md` first. Update the
   keymap defaults in `src/sase/default_config.yml` if the key is configurable.
3. **Tests.** Cover:
   - expire settles every named waiter and the notice;
   - an already-started waiter is skipped;
   - a waiter that resolved between render and confirm is skipped;
   - cancelling the confirmation does nothing;
   - nothing expires without the action.
4. **Docs.** Document the action next to the wait_checks notice docs and in the TUI
   notification docs.

## Phase `triage_volume`: one triage notification per bead (item 12.3, F12)

**Today.**

- `presentation_fingerprint` (`src/sase/scripts/_bead_task_triage_gates.py:49`) hashes
  every persisted field that changes a gate.
- When a pending gate's fingerprint changes, `_reconcile` cancels it with
  `task_triage_presentation_changed` (`sase_chop_bead_task_triage.py:415-470`, reason at
  `:194`). The next pass creates a new generation with a new request id, a new bundle
  dir, and a new notification row (`_bead_task_triage_gates.py:41-46`).
- Athena counted 2,139 such cancellations. The user's chezmoi config silences Beads-tab
  toasts with `quiet-task-beads` until "the triage-volume fix is in"
  (`home/dot_config/sase/sase.yml:259-267`).

**Required changes.**

1. **Measure first.** Read-only on athena: per-bead generation counts, and which
   fingerprint inputs changed between consecutive generations. Diff the persisted
   `request.json` presentations. Record the top churn sources on this bead.
2. **Fingerprint only what renders.** Drop fingerprint inputs that do not change what
   the gate shows or how it behaves. Keep the snooze, flag, and task-type-display rules
   documented in `presentation_fingerprint`'s docstring.
3. **One row per bead.** A required re-presentation either updates the pending gate in
   place, if `notification_gates` supports a safe presentation update, or recreates it
   under one stable notification dedup key per (project, bead, gate kind). Either way
   the user sees one row, refreshed in place, and it becomes unread again only when the
   gate kind changes or the bead re-enters ready. Keep the `request_id_conflict` branch
   from `b6f6c63e3e` working.
4. **Tests.** Extend `tests/test_axe_chop_bead_task_triage_*.py`:
   - N edits to a ready bead's non-rendered fields cause no cancellation;
   - a rendered change refreshes one row;
   - a kind change resurfaces it;
   - conflict recovery still works.
5. **The quiet-task-beads rule.**

   > [!decision] quiet_triage_rule = remove After the tests show one row per bead,
   > delete the `quiet-task-beads` rule (and only that rule) from the chezmoi source
   > `home/dot_config/sase/sase.yml`. Leave athena's `quiet-task-beads-telegram` rule
   > alone. Validate the edited config without applying it. Note on this bead that it
   > takes effect when the human runs `chezmoi apply` after installing.

   > [!decision] quiet_triage_rule = keep Leave the chezmoi config untouched. Record the
   > before and after per-bead row rates on this bead so the user can remove the rule
   > later.

6. **Docs.** Update `docs/reference/automation/notifications.md` (`:436-437`) and the
   triage docs.
7. **Measure.** Before: triage notification rows created per day and per bead on both
   hosts. Record the after-install command.

## Phase `gate_reconcile`: settle orphans and reap settled bundles (item 12.3, F12)

**Today.**

- Bundles live at `interaction_requests/<kind>/<request_id>/`
  (`src/sase/notification_gates/paths.py:25-29,56-73`). A bundle is settled by
  `response.json` or `cancellation.json`.
- Dismissing a notification only calls `mark_dismissed` and never cancels the gate
  (`notification_modal_basic_actions.py:123,178`).
- `_reconcile` (`sase_chop_bead_task_triage.py:339-640`) cancels tracked gates whose
  bead is no longer gateable. Its two sweeps are at `:589-607`.
- Nothing deletes settled bundles. sase-core reads them for stats
  (`agent_stats/gate_bundles.rs:61-67`, `activity.rs:397`).

**Required changes.**

1. **Orphan reconciliation.** Run this inside `_reconcile`, or in one place it calls,
   for the automation-owned kinds bead_task_triage creates: task triage and bead snooze.
   A pending gate is cancelled when either holds:
   - its notification is dismissed, with reason `notification_dismissed`;
   - its bead is resolved or gone, with the existing reasons.

   A dismissal-cancel is sticky: triage does not recreate the gate for that bead until
   the gate kind changes or the bead re-enters ready after leaving it. Store that in the
   chop's state. For other gate kinds, count dismissed-but-pending gates in the run
   summary and change nothing.

2. **Settled-bundle retention.** Delete settled bundles of those kinds older than a
   configurable horizon (default 30 days, with the key in `default_config.yml`). Use the
   bundle's settlement time. Do the deletion root-confined, without following symlinks,
   under the creation lock, and with a per-pass cap. First check what the sase-core
   stats readers report from old bundles, and document how the horizon affects them.
3. **Tests.** Cover:
   - a dismissed pending triage gate is cancelled and not recreated;
   - re-entering ready recreates it;
   - a resolved bead's gate is cancelled;
   - old settled bundles are reaped, while fresh and pending ones are kept;
   - other kinds are untouched;
   - the cap holds.
4. **Docs.** Document gate-bundle retention and the dismissal semantics.
5. **Measure.** Before, read-only on both hosts: bundle counts per kind and state, and
   pending gates whose notification is dismissed or whose bead is resolved. Record the
   after-install command.

## Phase `notify_quieting`: four small noise fixes (items 12.5 and 12.7, F12)

**Required changes.**

1. **file-hooks.**
   - Today `_notify_run` (`src/sase/file_hooks/runner.py:126-167`) appends a ✅ row on
     every successful hook run.
   - A success now upserts one silent row per hook (`silent=True`, dedup key per hook
     identity), refreshed in place, with no toast and no unread count.
   - Failures keep notifying, deduped per hook and failure signature, so a repeating
     failure +1s through the store guard.
2. **workspace-rescue.**
   - Today `notify_rescue` (`src/sase/workspace_provider/_rescue_store.py:274-312`) and
     `report_sidecar_may_be_lost`
     (`src/sase/axe/runner_workspace_sidecar_rescue.py:153-190`) post through
     `notify_workflow_complete` (`src/sase/notifications/senders.py:21-46`), which has
     no dedup key.
   - Add an optional `dedup_key` to that sender, upserting when it is set.
   - Key rescue notices on the rescued workspace and rescue cause. Pick the cause from
     the measured repeats.
3. **Prune nudge floor.**
   - Today `_should_notify`
     (`src/sase/core/agent_artifact_run_prune_publish.py:185-189`) fires when any run is
     selected. Its rows are priority because sender `axe` is a priority sender.
   - A selection now notifies only when its selected bytes reach a configurable floor
     (default 1 GiB) or the filesystem is under the managed disk-pressure thresholds.
     Reuse the pressure predicate; do not add a second one.
   - Unreadable protection sources and empty shards still notify.
4. **Claim ledger.**
   - Today `hold_workspace_claim` (`src/sase/running_field/_hold.py:70-84`) records
     "already pinned (no-op)" when `plan["changed"]` is false.
   - Skip the ledger write in that case, and return the same result to the caller.
5. **Tests.** One focused test per change:
   - a success writes one silent row across N runs;
   - a failure stays visible;
   - repeated rescues of one workspace give one row;
   - an 873 KiB selection on a roomy disk does not notify, but one under pressure does;
   - a no-op hold writes no ledger row (`tests/test_running_field_hold.py:60`).
6. **Docs.** Update `notification-types.md` (`:31`), `notifications.md` (`:179`),
   `docs/reference/agents/workspaces.md` (`:334-339`), `docs/reference/disk-cleanup.md`,
   and `docs/reference/diagnostics.md` (`:275`).
7. **Measure.** Before, read-only: rows per sender for file-hooks, workspace-rescue, and
   the prune publisher, and the nightly count of "already pinned" ledger rows. Record
   the after-install command.

## Phase `state_refresh_memo`: unchanged verdicts log once (item 12.6, F12)

**Today.**

- The chop (`src/sase/scripts/sase_chop_agent_state_refresh.py:31-32`, every 300 s) logs
  one warning per diagnostic, for example
  `<run>: conflicting_terminal_evidence; preserved completed`. The warnings come from
  `_publication_state_repair_evidence.py:296-300` and `:266-269`.
- State in `agent_state_refresh.json`
  (`src/sase/agents_sync/agent_state_refresh.py:59-69,127-156`) keeps only cursors and
  back-off.
- `schedule.rs:46-50` resets the cursor after a full pass, so every pass re-examines
  every published run.

**Required changes.**

1. **Verdict memo.** Persist, per run identity, the last unresolved verdict and an
   evidence fingerprint. Build the fingerprint from:
   - the run's published state, `started_at`, and `finished_at`;
   - the size and mtime of `agent_meta.json`, `done.json`, `waiting.json`, and
     `pending_question.json`;
   - the identity of the dismissed-bundle lookup result.

   Bound the memo, and prune entries for runs that left the window.

2. **Skip unchanged.** When the fingerprint is unchanged, skip the warning and the
   expensive lookup. Count the skip in the run summary. A changed fingerprint
   re-evaluates and logs once. Emit at most one daily summary line with the number of
   unchanged unresolved runs.
3. **Boundary.** Do not change `_locate_dismissed_bundles_for_run`,
   `filesystem_observations_for_run`, or how verdicts are classified; sase-1s5.3 owns
   them. Add a note on sase-1s5.3 describing the memo.
4. **Tests.** Extend `tests/test_axe_chop_agent_state_refresh.py`:
   - N passes over an unchanged unresolved run log once;
   - an evidence change re-logs;
   - the memo prunes;
   - a corrupt memo is ignored.
5. **Measure.** Before, read-only: agent_state_refresh warning lines per day and the
   maximum repeats per run on both hosts. Record the after-install command.

## Phase `log_horizons`: stated horizons, applied (item 13, F13)

**Today.**

- The 14-day tool-run log horizon (`DEFAULT_TOOL_RUNS_LOG_DAYS`,
  `src/sase/config/tools.py:30`) is enforced by the Rust ToolRun retention
  (`tool_run/store/retention.rs:48-61`) through `tool_run_reap_step`
  (`src/sase/core/disk_footprint_reap_tool_run.py:23`). That only runs from
  `sase disk reap --apply` or under disk pressure (`sase_chop_disk_pressure.py:62,81`).
- `_delete_confined` (`:131-147`) leaves empty run dirs behind.
- `tui.log` uses a size-only `RotatingFileHandler`, 2 MiB × 3
  (`src/sase/ace/tui/log_setup.py:44-49`).

**Required changes.**

1. **Hourly retention chop.** Add one housekeeping chop, for example `state_retention`,
   to the hourly housekeeping routine (`default_config.yml` around `:1721`,
   `@builtin_chop`). `clutter_reapers` adds its classes to the same chop.
   - It applies the existing `tool_run_retention` owner step, unchanged, at most once
     per configured interval, regardless of disk pressure.
   - It then removes run dirs that are now empty, under the same confinement rules.
   - This adds no second tool-run policy (sase-zw.8 note #3).
   - Where old dirs belong to runs that never settled, check whether the ToolRun store
     already marks dead runs. If it does not, record a `PROPOSED FOLLOW-UP:` and do not
     invent a policy.
2. **tui.log by age.** Rotate when the active file's oldest entry is older than a
   horizon (default 14 days), as well as by size. Delete rotated segments older than the
   horizon, at TUI start and at each rotation. Read `tui.md` first.
3. **Verify digests.** Confirm read-only on both hosts that `prune_old_error_digests`
   runs and holds its horizon. Fix it only if it does not.
4. **Verify artifact_run_prune.** Read-only on apollo and athena, confirm the latest run
   reports no unreadable protection sources. If either host is still blocked, fix the
   cause in `collect_ace_run_retention_protections`
   (`src/sase/core/agent_artifact_run_protection.py:42-126`).
5. **Tests.** Cover:
   - the chop applies the tool-run owner without pressure;
   - empty dirs go and live ones stay;
   - the interval gate holds;
   - tui.log rotates by age and prunes old segments.
6. **Docs.** Update `docs/reference/automation/scheduler-housekeeping.md` (`:200-203`
   says tool-run retention runs only under pressure),
   `docs/reference/automation/tools.md` (`:186-192`),
   `docs/reference/config/runtime-and-storage.md` (`:286-300`), and the TUI logging
   docs. Note the overlap on **sase-zw.8**.
7. **Measure.** Before, read-only: the size and old-dir counts of `~/.sase/tools/logs`,
   the `tui.log*` ages, the digest count, and the latest artifact_run_prune summary on
   both hosts. Record the after-install command.

## Phase `clutter_reapers`: fix the leaks, reap the backlog (item 13, F13)

**Today.**

- **Proc log locks.**
  - `log_file_lock` (`src/sase/logs/_bounded.py:77-117`) creates `.{name}.lock` with
    `O_CREAT` and never removes it.
  - `delete_proc_logs` (`src/sase/procs/logs.py:97`, candidates at `:156`) removes only
    `<id>.log` and `<id>.log.1`.
- **Registry temp files.**
  - `write_registry` (`src/sase/agent/names/_registry_store.py:85-128`) writes through
    `mkstemp`, under `_registry_mutation_lock`.
  - A hard kill during the dump of the 44 MB registry leaves the temp file behind.
- **Index backups.**
  - No SASE code writes `agent_artifact_index.sqlite*.bak`; athena's 222 MiB file was a
    manual copy.
  - The `disk_pressure` description says backups are reported, never deleted, but no
    inventory row reports them.

**Required changes.**

1. **Lock leak.**
   - `delete_proc_logs` also removes the proc's lock file, while holding that lock.
   - A reaper class removes `.<id>.log.lock` files that meet all three conditions:
     - no `<id>.log` or `<id>.log.1` exists;
     - no proc row references the id;
     - the file is older than a horizon (default 1 day).
   - It takes the lock non-blocking before unlinking, and skips the file on contention.
2. **Registry temp leak.**
   - Before writing, under `_registry_mutation_lock`, `write_registry` removes
     `.agent_name_registry.json.*.tmp` siblings older than a short age (default 10 min).
   - Confirm first that every registry write holds the lock. If one does not, use only
     the age-gated sweep.
   - A reaper class applies the same sweep from the retention chop.
   - The 44 MB registry size is out of scope; record a `PROPOSED FOLLOW-UP:` if you
     measure it as a cost.
3. **Index backups.**

   > [!decision] index_backups = report Add a disk inventory row for agent-index backups
   > (`*.sqlite*.bak` with `-wal`/`-shm` siblings next to the index), with age and size,
   > marked for human action. Name the row in the disk-pressure notice. Never delete a
   > backup automatically. Propose deleting athena's Sep 26 backup once through
   > `/sase_gate`.

   > [!decision] index_backups = reap Add an age-based reaper class that deletes
   > agent-index backups older than 30 days, together with their `-wal`/`-shm` siblings,
   > root-confined and in preview/apply form. It never touches the live index, its WAL,
   > or the newest `.corrupt-*` quarantine copy.

4. **Wiring.** Each class is an owner step in the `run_disk_reap` pattern
   (`src/sase/core/disk_footprint_reap.py:38-73`), with preview and apply. It is visible
   in `sase disk list` and `sase disk reap`, and the `log_horizons` chop applies it.
   Apply the same safety rules: no following symlinks, root confinement, and a per-pass
   cap.
5. **Tests.** Cover:
   - deleting a proc's logs removes its lock;
   - an orphan lock is reaped;
   - a locked or fresh lock is kept;
   - a stale registry temp file is swept while a fresh one is kept;
   - the backup branch selected by the decision;
   - the caps.
6. **Docs.** Update `docs/reference/disk-cleanup.md` and
   `docs/reference/automation/scheduler-housekeeping.md`. Note the overlap on
   **sase-zw.8**.
7. **Measure.** Before, read-only on both hosts: `.log.lock` and `.log` counts in
   `~/.sase/procs/logs/`, the registry temp file count and bytes, and the backup files
   with their sizes and ages. Record the after-install command.

## Landing (child epic land agent)

1. Confirm that every phase bead is closed and that its notes are accounted for.
2. Confirm that each measuring phase recorded its before-numbers and after-install
   command on its own bead, with a one-line summary on **sase-1tz.4**.
3. Run `just check` on the combined tree. Confirm that the `sase-core-revision.txt` pin
   contains the sase-core commits from `wal_busy`, `gateway_swr`, `attention_key`, and
   `blob_store`. Confirm that sase-telegram's `sase tool run check` passed for
   `blob_adopt`. Under `quiet_triage_rule = remove`, confirm the chezmoi change is
   committed in that repo.
4. Confirm the relationship notes are on sase-1s5, sase-1s5.3, and sase-zw.8.
5. Triage the phases' `PROPOSED FOLLOW-UP:` notes.
6. Close **sase-1tz.4** with a note summarizing the verified result.

Never close the parent epic **sase-1tz** or any other ancestor.
