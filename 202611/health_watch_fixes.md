---
tier: epic
title: 'Health-watch fixes: wait notices name their waiter, failed staged-monitor
  sessions never park waiters silently, fleet snapshots build within their refresh
  timeout, and dismissed runs converge in the published agents sidecar'
goal: Every "Wait dependency can never self-resolve" notice names the blocked agent
  (never a project key). An agent waiting on a session whose root turn and monitor
  both failed is flagged within one wait_checks pass instead of waiting silently.
  apollo's presentation fleet snapshot builds in well under the gateway's 4 s refresh
  timeout, so remote fleet readers stop receiving stale "timeout" snapshots every
  minute. agent_state_refresh finds each run's dismissed bundle by exact identity,
  so dismissed runs stop being published as active or waiting forever, and an incomplete
  dismissal lookup is never reported as live.
decisions:
  covering_index:
    ask: Also add a covering lineage index so single-agent related-artifact lookups
      get faster?
    default: false
    why: batching alone removes the measured build cost without an index schema migration
    answer: false
phases:
- id: wait-notice-fixes
  title: Terminal-blocked wait notices name their waiter, and the silent-waiter case
    is reproduced and closed
  depends_on: []
  size: medium
  description: 'wait-notice-fixes: resolve the Waiter label from the waiter''s agent_meta.json
    instead of waiting.json''s cl_name (F8), then reproduce the athena sase-1ra.7
    session shape (failed root turn plus failed staged monitor) in a wait_checks test
    and fix wherever its terminal blocker is dropped (F9).'
- id: fleet-lineage-batch
  title: Batched dismissal-lineage resolution makes fleet snapshot builds fast
  depends_on: []
  size: medium
  description: 'fleet-lineage-batch: in sase-core, replace the per-candidate SQL lineage
    walk in resolve_agent_session_dismissal_lineage with one lineage-column load per
    (project, workflow dir) pair and an in-memory walk with identical results, prove
    equivalence and O(pairs) query count in tests, move sase''s sase-core pin, and
    verify on apollo (F10).'
- id: dismissal-evidence-lookup
  title: State repair finds dismissed bundles by exact run identity and never reads
    an incomplete lookup as live
  depends_on: []
  size: medium
  description: 'dismissal-evidence-lookup: add exact agent-name/source-run-id filters
    (with supporting indexes) to sase-core''s dismissed-bundle summary query, make
    sase''s _locate_dismissed_bundles_for_run use them, stop filesystem_observations_for_run
    from silently dropping a failed dismissal lookup, move sase''s sase-core pin,
    and verify that apollo''s published active/waiting rows for dismissed runs converge
    (F11).'
proposed_by: bbugyi200.apollo.cy--6
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 06:51:21
status: wip
bead_id: sase-1s5
---

- **PROMPT:** [prompts/202611/health_watch_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/health_watch_fixes.md)
- **BEAD:** [sase-1s5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1s5/README.md)

# Health-watch fixes: wait notices, silent waiters, slow fleet snapshot builds, and stuck dismissed runs

## Context

A six-check health watch of the SASE agents on apollo started on 2026-11-08. Each check
waits two hours, then inspects notifications, running agents, axe chop results, the
service host, monitors, gates, workspace claims, and logs.

- **Check 1/6 (00:40 UTC).** apollo's services were healthy. The scheduler and gateway
  had 0 restarts, and every axe chop run in the previous three hours ended `ok` or
  `no_op`. No error digest had been written since the `artifact_run_prune`
  sidecar-protection errors cleared at 2026-11-07 21:40 UTC. The check found two
  wait-system defects (F8, F9).
- **Check 2/6 (02:55 UTC).** apollo was still healthy. The service host had been up
  since 2026-11-07 21:59 UTC with `NRestarts=0`, there were no new error digests and no
  failed agents, and all 350 chop runs since check 1 ended `success`, `no_op`, or
  `skipped`. The TUI logged no hitch after 2026-11-07 21:59 UTC. The check did find one
  persistent performance defect in the gateway (F10).
- **Check 3/6 (05:10 UTC).** apollo was still healthy. The service host was unchanged
  (`NRestarts=0`, gateway and scheduler at 0 restarts), there were no new error digests,
  and all 609 chop runs since check 2 ended `success`, `no_op`, `skipped`, or `ok`. F10
  persisted unchanged (builds of 6.9–8.2 s, one timed-out waiter per minute). Reading
  the `checks` lumberjack log found a convergence regression in `agent_state_refresh`
  (F11).
- **Check 4/6 (07:25 UTC).** apollo was still healthy, with no new findings. The service
  host was unchanged (`NRestarts=0`), there were no new error digests or notifications,
  and all 642 chop runs since check 3 ended `success`, `no_op`, `skipped`, or `ok`. F10
  reproduced exactly: 128 presentation builds of 6.7–8.0 s (mean 7.2 s) left 121
  timed-out waiters in two hours. F11 reproduced too: an `agent_state_refresh` tick that
  examined 100 runs still counted 79 as live and 13 as unresolved, while apollo ran one
  agent. The `agent_waits` chop has reported `dead_waiting=7` without change since at
  least 2026-11-07 14:48 UTC; these are the stale waiters whose notices F8 relabels.
- **Check 5/6 (09:35 UTC).** apollo was still healthy, with no new findings. The service
  host was unchanged (`NRestarts=0`, gateway and scheduler at 0 restarts), there were no
  new error digests, notifications, or failed agents, and all 631 chop runs since check
  4 ended `success`, `no_op`, `skipped`, or `ok`. F10 reproduced again: 124 presentation
  builds of 6.8–8.2 s (mean 7.3 s) left 122 timed-out waiters. F11 reproduced too: the
  latest full `agent_state_refresh` tick examined 100 runs and still counted 73 as live
  and 18 as unresolved, with `changed=0`. `dead_waiting=7` was still unchanged.
- **Check 6/6 (11:45 UTC).** apollo was still healthy, with no new untracked defects.
  There were no new error digests or failed agents, and all 643 chop runs since check 5
  ended `success`, `no_op`, `skipped`, or `ok` (four were still running and making
  progress). At 11:40 UTC `sase_core_rs` 0.44.0 (sase-core `origin/master`) was
  reinstalled as an editable build and the service host restarted cleanly
  (`NRestarts=0`). That changed neither finding. F10 continued across the restart: 120
  presentation builds of 6.1–10.9 s (mean 7.3 s) left 116 timed-out waiters, and the
  first request after the restart got a `504 Gateway Timeout` (see F10's cold-cache
  bullet). F11 reproduced too: the three full `agent_state_refresh` ticks since check 5
  each examined 100 runs with `changed=0`, counting 38, 92, and 2 as live and 14, 0, and
  26 as unresolved. `dead_waiting=7` was still unchanged.

The three phases are independent and touch different code paths, so they can run in
parallel. `fleet-lineage-batch` and `dismissal-evidence-lookup` both land a sase-core
change and move sase's `sase-core-revision.txt`. Whichever lands second rebases its pin
past both sase-core commits.

## Phase `wait-notice-fixes` (sase)

### F8: terminal-blocked wait notices name a project key, not the waiter

`upsert_terminal_blocked_wait_notification` in
`src/sase/scripts/_chop_wait_checks_terminal.py` builds its `Waiter:` line from
`_waiting_agent_label`. That helper returns `waiting.json`'s `cl_name`. For ordinary
`ace(run)` agents, `cl_name` is the project directory key (`gh_sase-org__sase`), not the
agent. `waiting.json` carries no agent name at all; its keys are `cl_name`,
`patch_name`, the queue fields, `slot_requested_at`, `timestamp`, and the wait-priority
fields. On athena, all 20 retained "Wait dependency can never self-resolve"
notifications read `Waiter: gh_<org>__<repo>`. The user cannot tell which agent to kill
and relaunch, which is the only instruction the notice gives. It also breaks the
accepted decision that project surfaces never show a directory key
(`decisions:project-labels-never-keys`).

### F9: a failed staged-monitor session parked its waiters with no notice

On athena, epic `sase-1ra` was created on 2026-11-04. Phases `sase-1ra.8` and
`sase-1ra.9` were launched as waiters with `%w(sase-1ra.7, for_epic=false)` and
`%w(bead=sase-1ra.7)`. Their artifacts are
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202611/04/20261104180116` and
`.../20261104180117`.

Agent session `sase-1ra.7` ended like this:

- **Root turn `sase-1ra.7--plan`** (`.../20261104180113`). `done.json` has
  `name: sase-1ra.7` and `outcome: failed`, with error
  `MonitorError: tool run dd4cfab4… already has a stop request; nothing to join`. This
  is the F2b staged-adoption bug that `sase-1rg` has since fixed. `agent_meta.json` has
  `agent_session_role: root` and `monitor_id: fwh140mxsd39`.
- **Monitor `sase-1ra.7--mon`** (`.../20261104185106`). `done.json` has
  `outcome: monitored`, `monitor_state: failed`, and `status_bucket: Failed`, with the
  same error. No follow-up was launched. `staged_monitor_launch.json` is present, and
  `parent_timestamp` is `20261104180113`.

Bead `sase-1ra.7` closed at 2026-11-05T00:02:23Z. The waiters never started. They also
never received a terminal-blocked notice: athena's notification history has the
`sase-1ra.7` failure notice and nothing for its waiters. They sat in `WAITING` until
athena's 2026-11-07 06:02 reboot SIGTERMed them ("Agent killed while waiting"). The epic
then stranded with phases 8–12 `in_progress` and no live agent on either machine. Phase
`sase-1ra.8` is the planned fix for apollo's zombie staged monitor `5kej4t`.

Athena retains only the newest 24 `wait_checks` runs, so the logs from that window are
gone. Reading current master, the waiter should produce a blocker:

- `_is_superseded_failed_handoff_turn_member` keeps the failed root, because the
  monitor's follow-up outcome is not `launched`.
- `_terminal_blocker_members` then picks the newest failed member, which is the monitor.
- `recovery_in_flight` is false, because the root's `done.json` has no `recovery` key.

So either the code installed on athena then behaved differently, or some path drops this
blocker before notification. Candidates are `blocked_on` label matching, name resolution
of `sase-1ra.7` against members named `sase-1ra.7--plan`/`--mon`, and the mixed
agent+bead wait whose bead half resolved. Today, nothing in the tree covers this session
shape.

### Change 1 — name the waiter (F8)

- Replace `_waiting_agent_label` with a resolver that reads the waiter's agent name from
  `<waiter_dir>/agent_meta.json` (`name`). It must tolerate a missing or malformed file,
  then fall back to the artifacts directory name. It must never return `cl_name` or any
  other project directory key.
- If the notice also names the project, use its display name through the existing
  project display snapshot helper, never the key.
- Name the blocker by its agent name too, when the candidate carries one. Keep the
  dependency label and artifact path, because recovery uses them.
- Keep the dedup key unchanged (`wait_checks:terminal-blocked:<waiter_dir>`), so
  existing rows update in place rather than duplicating.
- Tests: extend the terminal-blocked notification coverage in the
  `tests/test_axe_chop_wait_checks*.py` family, using
  `tests/_axe_chop_wait_checks_helpers.py`. Assert that the `Waiter:` note shows the
  agent name for an `ace(run)` waiter whose `cl_name` is a project key, and falls back
  to the directory name when `agent_meta.json` is absent.

### Change 2 — reproduce and close the silent-waiter case (F9)

1. Build a fixture that mirrors the athena session shape above, with only the marker
   fields the wait index reads:
   - The failed root turn (`done.json` `name: sase-1ra.7`, `outcome: failed`;
     `agent_meta.json` name `sase-1ra.7--plan`, session `sase-1ra.7`, role `root`).
   - The monitor member (`turn_kind: monitor`, `parent_timestamp` pointing at the root,
     `done.json` `outcome: monitored`, `monitor_state: failed`, no follow-up agent or
     outcome, `staged_monitor_launch.json` present).
   - A closed bead `sase-1ra.7`.
   - A waiter whose markers wait on both agent `sase-1ra.7` (`for_epic=false`) and bead
     `sase-1ra.7`.

   If you have SSH access to athena, you may copy the real marker files from the paths
   above as a reference. Do not depend on them at test time.

2. Run one `wait_checks` pass over the fixture. The required behavior: the waiter must
   not stay silently `WAITING`. A terminal-blocked notice must be upserted within one
   pass, naming waiter `sase-1ra.8` and dependency `sase-1ra.7`. The wait must not be
   treated as satisfied, because a failed session is not success.
3. If the test fails on master, find where the blocker is dropped and fix it there.
   Wait-dependency resolution currently lives under
   `src/sase/core/wait_dependency_resolution/` and the `_chop_wait_checks_*` modules. If
   the drop point turns out to be a sase-core-owned predicate, follow the Rust core
   boundary in `docs/rust_backend.md`: change sase-core, then move
   `sase-core-revision.txt`. If the test already passes on master, keep it as a
   regression test, and say in the commit message that master already handles the shape.
4. Cover the near variants in the same test module: a monitor that failed before any
   follow-up, and a monitor whose follow-up outcome is `dropped`. Do not change the
   superseded-member rules that `sase-1h9.4` deliberately quiets.

### Verification

- The new and changed tests pass in their focused modules.
- `just check` passes.

## Phase `fleet-lineage-batch` (sase-core, then sase's pin)

### F10: every presentation fleet snapshot build overruns the 4 s refresh timeout

apollo's gateway log (`~/.sase/service/procs/gateway/output.log`) shows the symptom.
Retention covers 2026-11-01 onward, and every presentation-scope build in that span is
logged at `warn` as `fleet snapshot build finished`. sase-core's
`crates/sase_gateway/src/fleet_reads/service.rs` logs at `warn` only when a build fails
or runs longer than `SNAPSHOT_REFRESH_TIMEOUT` (4 s).

- **2026-11-08 00:00–03:00 UTC:** about 56 builds per hour, averaging about 7.2 s (max
  8.2 s) for 86 served rows. About 55 waiters per hour timed out.
- **2026-11-02 → 2026-11-06, when about 200–230 rows were served:** builds averaged
  20–30 s, with outliers of 335 s, 2,022 s, and 9,465 s.
- **Cold-cache 504s:** when the gateway has no previous snapshot to fall back on, for
  example on the first request after a gateway start, a waiter that times out gets
  nothing at all. The gateway answers `504 Gateway Timeout` after 4 s (`tower_http` logs
  it at `ERROR` with `latency=4003 ms`). The log shows 5–17 of these a day from
  2026-11-02 through 2026-11-07, plus one 21 s after the 2026-11-08 11:40:46 UTC service
  host restart.

Each waiter that times out gets the previous snapshot, marked stale and partial with
`error = "timeout"`. That is the single-flight design from
`plan:202610/apollo_gateway_snapshot_stampede.md`. So the remote reader that polls
apollo's fleet about once a minute always sees data one build behind. It also fails the
"no long-running-build warnings" pass criterion of that epic's open phase `sase-1j1.6`.

The stampede containment itself is healthy. The gateway runs 18 threads at about 100 MiB
RSS, holds no artifact-index fds, has a 0-byte WAL, and averages about 0.16 of a core.

### Root cause (measured on apollo, 2026-11-08)

- `sase_core_rs.assemble_fleet_catalog({"sase_home": "~/.sase"})` runs the same pipeline
  as the gateway's presentation build: the same index query, liveness, dismissal
  lineage, `select_fleet_presentation`, and projection. It reproduces the gateway
  exactly: 6.9–7.1 s per build, with 86 summaries out of 304 candidates.
- `py-spy record --native` on that probe puts 83% of samples in `pread64`, which means
  SQLite page reads. The presentation index query itself (`freshness=revalidate`,
  `record_shape=full`, limits 512) takes only 0.56–1.18 s.
- The rest is `resolve_agent_session_dismissal_lineage` in
  `crates/sase_core/src/agent_scan/index/lineage.rs`. For each candidate,
  `agent_session_root_dismissed_for_candidate` calls `select_lineage_rows` until the
  timestamp set stops changing. Replaying that walk over the 304 real candidates takes
  727 `select_lineage_rows` calls and 6.4–6.7 s.
- Each call costs 6–10 ms. Its `WHERE` clause ORs five `IN (...)` lists, so SQLite
  range-scans
  `idx_agent_artifacts_project_workflow (project_name=?, workflow_dir_name=?)` and
  fetches every table row of the pair. That is 2,522 rows for sase `ace-run` and 4,066
  for bob-cli `ace-run`. Rows average about 14 KiB because of `record_json`; the index
  file is 95 MiB (24,354 pages) and has no `sqlite_stat1`.
- `agent_session_root_by_agent_session` has the same scan shape (about 5.5 ms per call),
  but the replay hit it only 27 times.

Alternatives measured on a private copy of the index:

| Variant                                                              | 304-candidate walk |
| -------------------------------------------------------------------- | ------------------ |
| Current per-candidate SQL                                            | 6.44 s             |
| `ANALYZE` only                                                       | 6.41 s             |
| `UNION` of five per-column `IN` queries                              | 20.8 s (worse)     |
| Covering index on the lineage columns, queries unchanged             | 0.87 s             |
| One lineage-column load per (project, workflow) pair, walk in memory | 0.07 s             |
| The same batched load with the covering index                        | 0.04 s             |

The `UNION` rewrite is worse because, with no stats, the planner falls back to the same
range scan inside each branch once an `IN` list has more than one element. Do not ship
it.

### Change — batch the lineage walk

1. In `resolve_agent_session_dismissal_lineage`, group the candidates by
   `(project_name, workflow_dir_name)`. For each distinct pair, load the lineage columns
   once, in a single query: `artifact_dir`, `timestamp`, `agent_session`,
   `parent_timestamp`, `retry_of_timestamp`, `retried_as_timestamp`, and
   `retry_chain_root_timestamp`. Never select `record_json` in this query. Build
   in-memory maps from each relation timestamp to the rows that carry it.
2. Run each candidate's expansion against those maps. Results must be identical to
   today's:
   - Keep the `MAX_RELATED_ARTIFACT_QUERY_ITERATIONS` (32) and
     `MAX_RELATED_ARTIFACT_LINEAGE_TIMESTAMPS` (128) bounds and the
     `insert_lineage_timestamp` / `add_related_timestamps` semantics.
   - Keep the root choice: the related row with no `parent_timestamp`, picked in
     `by_timestamp` (`BTreeMap`) order, then the `agent_session` fallback (oldest
     parentless row of the same session in the pair).
   - Keep the seed-level `dismissed_parent_suffix_for_seed` and `seed_definitively_dead`
     suffix checks.
3. Keep the per-root work as it is (`load_record_by_artifact_dir`, `RecordSummary`,
   `record_is_dismissed`); those are primary-key and covering-index lookups. Memoize the
   root verdict by root `artifact_dir` within one call, because many candidates share a
   session root.
4. Leave `query_related_agent_artifact_dirs`, the single-agent path used by the TUI's
   LLM-calls reader, on its current queries unless the covering-index decision says
   otherwise.

> [!decision] covering_index Also add
> `CREATE INDEX idx_agent_artifacts_lineage_cover ON agent_artifacts(project_name, workflow_dir_name, timestamp, parent_timestamp, retry_of_timestamp, retried_as_timestamp, retry_chain_root_timestamp, agent_session, artifact_dir)`
> through the artifact index's existing schema-migration path, with its schema-version
> bump and migration test. It makes the batched load index-only and cuts single-agent
> `select_lineage_rows` calls from about 8 ms to well under 1 ms.

> [!decision] covering_index = no Ship the batched walk alone. Make no index schema
> change.

### Tests (sase-core)

- **Equivalence.** Keep the current per-candidate resolver as a test-only oracle. On
  fixtures, assert that the batched resolver returns identical
  `AgentSessionDismissalLineageResultWire` lists. The fixtures must cover: parent
  chains; retry chains (`retry_of`, `retried_as`, `retry_chain_root`); a multi-member
  session with no parentless row in reach (the `agent_session` fallback); a dismissed
  root; a dead seed with a dismissed raw suffix; a missing seed; candidates spread
  across several projects and workflow dirs; and chains that hit both bounds.
- **Query count.** Use rusqlite's `trace`/`profile` hook or a test counter to assert
  that the number of lineage-column queries equals the number of distinct (project,
  workflow dir) pairs, not the number of candidates. Do not write wall-clock assertions.
- Existing gateway `fleet_reads` tests and `fleet_catalog` tests stay green. Run
  sase-core's usual checks for the touched crates.

### Land and verify

1. Land the sase-core change. Then move sase's `sase-core-revision.txt` pin past it, as
   `docs/rust_backend.md` describes, and run `just check` in sase.
2. Agents cannot run a global reinstall (`decisions:global-install-is-human-only`).
   Propose apollo's reinstall plus a gateway restart
   (`sase service proc restart gateway`) with `/sase_gate`, and measure in the gate's
   follow-up.
3. Measure after the reinstall:
   - Run `assemble_fleet_catalog({"sase_home": str(Path.home() / ".sase")})` from the
     installed sase tool Python; it should finish in under 2 s.
   - For at least 10 minutes, apollo's gateway log should show presentation builds only
     at `info` level (under 4 s), with `timed_out_waiters=0`.
   - The log should show no `504 Gateway Timeout` after the restart, including for the
     first request, which arrives before any snapshot exists.
4. Record the before and after numbers in this phase's bead notes. Also add a note to
   `sase-1j1.6` saying that its long-running-build criterion can now be re-measured.
   Closing `sase-1j1.6` stays with that phase's own owner.

## Phase `dismissal-evidence-lookup` (sase-core, then sase)

### F11: agent_state_refresh can never see a dismissal, so dismissed runs stay "active"

Epic `sase-1cc` / `sase-1pu.5` made the five-minute `agent_state_refresh` job converge
published agents-sidecar run state with local evidence. On apollo that convergence has
effectively stopped:

- **Refresh results since 2026-11-07 00:00 local** (289 ticks, from
  `~/.sase/axe/logs/lumberjack-checks.log`): 8,222 records examined, 40 changed, 4,296
  classified `live`, and 1,053 unresolved. The latest tick examined 100, changed 0, and
  called 82 of them live, while apollo was running exactly one agent.
- **Published apollo records across all project sidecars:** 3,059 `active` and 177
  `waiting`, 1,064 of them started on or after 2026-11-01. Only 5 apollo records are
  `dismissed`.
- **Log noise:** since 2026-11-04 the job has logged
  `<agent>: dismissed index lookup is incomplete; preserved <state>` for 628 distinct
  runs, growing every hour.

Root cause, traced on apollo against the installed code:

1. `_locate_dismissed_bundles_for_run` in
   `src/sase/agents_sync/_publication_state_repair_observations_runs.py` calls
   `query_summaries(root, limit=_DISMISSED_CANDIDATE_LIMIT + 1)` with **no filter**. It
   then treats more than 64 rows as an overflow and returns
   `"dismissed index lookup is incomplete"`. apollo's dismissed-bundle index holds
   14,738 hidden bundles (bob-cli 8,234, sase 5,760), so every call overflows. The bound
   arrived on 2026-11-01 in `0547d23ce3` ("bound exact-run identity and repair
   deadlines", part of `sase-1pu.5.1.5`).
2. `filesystem_observations_for_run` only surfaces that error when the run has **no**
   artifact observations (`if bundle_error is not None and not observations`). When the
   artifact dir still exists, the error is silently dropped and only an `unfinished`
   observation reaches the Rust resolver. The resolver then answers `live_evidence`, and
   the published `active` row never changes.
3. A project filter alone would not fix it, because bob-cli alone has 8,234 bundles. The
   published `V2RunRecord` carries `source_run_id` (a one-way hash of project, workflow
   dir, and durable id) and `local_name`, but not the raw suffix. sase-core's
   `DismissedBundleIndexQueryWire` (`crates/sase_core/src/agent_archive/wire.rs`) can
   filter only by `suffixes`, `cl_name`, `project_name`, `top_level_only`, and
   `visibility`. The `dismissed_bundle_summaries` table already has `agent_name` and
   `source_run_id` columns. `source_run_id` is indexed only as the third column of
   `idx_dismissed_bundle_archive_key (source_username, source_machine, source_run_id)`,
   and `agent_name` has no index.

Measured case: `bob-cli-hg.14--mon` (bob-cli,
`.../artifacts/ace-run/202611/06/20261106074652`, a monitor whose monitored command
errored, so it has no `done.json`). It has a dismissed bundle whose `source_run_id`
matches. Observations and resolver results:

| Dismissal lookup                             | Observations                         | Resolver (published `active`)          |
| -------------------------------------------- | ------------------------------------ | -------------------------------------- |
| Today (unfiltered, overflows, error dropped) | `unfinished`                         | `live_evidence`, unchanged             |
| Error surfaced as an unreadable `dismissed`  | `unfinished`, `dismissed` unreadable | `unreadable_evidence`, unchanged       |
| Exact identity match (simulated in memory)   | `unfinished`, `dismissed`            | `updated_from_dismissed` → `dismissed` |

### Change 1 — exact identity filters in sase-core

Follow the Rust core boundary (`docs/rust_backend.md`).

- Extend `DismissedBundleIndexQueryWire` with optional
  `agent_names: Option<Vec<String>>` and `source_run_ids: Option<Vec<String>>`, both
  `#[serde(default)]`, so existing callers and wire payloads stay valid. When either is
  present, `query_summaries` in `crates/sase_core/src/agent_archive/store.rs` adds one
  clause that matches `s.agent_name IN (...) OR s.source_run_id IN (...)`. That clause
  is ANDed with the existing filters. An empty list matches nothing, mirroring today's
  empty-`suffixes` short-circuit.
- Add `idx_dismissed_bundle_agent_name ON dismissed_bundle_summaries(agent_name)` and
  `idx_dismissed_bundle_source_run_id ON dismissed_bundle_summaries(source_run_id)` to
  `INDEX_DDLS` (`CREATE INDEX IF NOT EXISTS`, so existing stores gain them on open).
  Bump `ARCHIVE_STORE_SCHEMA_VERSION` only if the store's compatibility check requires
  it for new indexes.
- Keep the Python binding (`crates/sase_core_py/src/agent_custody/mod.rs`) passing the
  request through unchanged, and update the sase-side wire docs or types if they
  enumerate the request fields.
- Tests (sase-core): with more than 64 summaries in the store, filtering by agent name
  returns only that agent's bundles; filtering by source run id does the same; the two
  filters OR together; they AND with `project_name` and `visibility`; an empty list
  returns nothing; omitted fields behave exactly as today. Use `EXPLAIN QUERY PLAN` or
  an equivalent assertion to confirm that the new clause uses the new indexes rather
  than a table scan.

### Change 2 — use the filters, and never drop a failed lookup (sase)

- In `_locate_dismissed_bundles_for_run`, pass `project_name=target.project_key`,
  `agent_names={local_name}` when `local_name` is non-empty, and
  `source_run_ids={source_run_id}`, through `query_summaries` in
  `src/sase/ace/dismissed_bundle_index/_api.py`. Add the two keyword arguments there.
  Keep `_summary_is_candidate` as the post-filter and keep the
  `_DISMISSED_CANDIDATE_LIMIT` guard: it now reports genuine overflow only.
- In `filesystem_observations_for_run`, never discard `bundle_error`. When it is set and
  artifact observations exist, append the unreadable `dismissed` observation anyway, so
  the resolver returns an unresolved reason (`unreadable_evidence`) and never
  `live_evidence`. Keep returning the error to the caller, so the diagnostic line and
  the `unresolved` counter stay truthful.
- Tests (sase), in the agents-sync state-repair test modules:
  - A dismissed-bundle store with more than 64 unrelated bundles plus one bundle for the
    target run. A published `active` record whose artifact dir has no `done.json`
    resolves to `dismissed`.
  - A matching bundle found only through `source_run_id` (the agent name was reused by a
    later run) still resolves, while a bundle for the same name but a different run id
    does not.
  - When the dismissal lookup reports an error and the artifact dir exists, the run is
    counted `unresolved`, never `live`.

### Land and verify

1. Land the sase-core change. Move sase's `sase-core-revision.txt` past it, then land
   the sase change and run `just check`.
2. Report-only check before the reinstall:
   `sase agent sync --repair-states -p gh_bobs-org__bob-cli` should report
   `bob-cli-hg.14--mon` as changing to `dismissed`. Run it from the repo or tool Python
   that carries the new code.
3. The reinstall and service restart are human-only
   (`decisions:global-install-is-human-only`). If `fleet-lineage-batch` has not already
   proposed one that covers this commit, propose apollo's reinstall with `/sase_gate`.
   Include a scheduler restart (`sase service proc restart scheduler`) only if chop runs
   do not pick up newly installed code on their own; check that before proposing. Then
   measure in the gate's follow-up.
4. After the reinstall, measure:
   - Within a few `agent_state_refresh` ticks, the `checks` log shows `changed>0`,
     `live` at most the number of genuinely live agents, and no new
     `dismissed index lookup is incomplete` lines.
   - Over the next day, apollo's published `active`/`waiting` count across project
     sidecars falls steadily from about 3,236.
5. Record the before and after numbers in this phase's bead notes.

## Out of scope

These are user or host actions, or work already owned elsewhere. Do not change them
here:

- Relaunching the stranded `sase-1ra` epic. Its phases `.11` (boot reconciler) and `.12`
  (zombie-epic report) own reboot-stranded and zombie epics in general.
- apollo's zombie staged monitor `5kej4t` (`bob-cli-gh.1`), owned by `sase-1ra.8`.
- Salvaging `cd--code`'s pinned sase workspace 10 and bob-cli workspace 24, which is
  already announced as "Workspace held". Also the failed 2026-09-19 custom gate
  `custom-c1f4c258…`, which awaits a user cancel or re-decision.
- The other `PROPOSED FOLLOW-UP` items recorded on `sase-1j1.6`: cgroup limits, index
  connection pooling, moving `settled_unexpired` off the async worker threads, and
  federation coalescing.
- The 311 legacy `--gate`/`--mon` member records that `agent_state_refresh` logs as
  `conflicting_terminal_evidence; preserved failed` (about 450 lines a day since
  2026-11-01). Each is published `failed`, while its `done.json` says `gated` or
  `monitored` with a completed state. The resolver deliberately refuses
  terminal-to-terminal flips. All of these runs date from September and October. Every
  November `--mon` record published `failed` either has `monitor_state: failed` (a real
  failure) or no terminal marker at all, so the publish race that produced them no
  longer reproduces. Leave the policy as it is.
