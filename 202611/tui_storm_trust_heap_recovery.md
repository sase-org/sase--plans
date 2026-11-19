---
tier: epic
title: Recover TUI speed - stop the hijack storm, fix the trust-root defect, shrink
  the heap
goal: '`sase tui` reaches a usable Agents list in about 6-7 s instead of about 14
  s. Selection changes can no longer start archive scans that fill memory or the default
  executor. Numbered workspaces inherit their project''s trust acknowledgment. Long
  sessions keep a bounded heap and responsive j/k navigation. Existing telemetry proves
  each stage.

  '
decisions:
  hotfix_reinstall:
    ask: Should the storm-hotfix phase propose a gated global reinstall as soon as
      it lands?
    default: true
    why: The live TUI runs a storm-capable build; one approval stops the storm before
      the epic lands
    answer: true
  trust_ack_compat:
    ask: How should the trust-root fix treat acknowledgments recorded under the old
      `<repo>/sase` key?
    choices:
      cutover: Drop the old key; you re-run `sase config trust allow` once in the
        primary checkout
      compat: Honor old-key acks behind a sunset flag; adds a flag bead and a legacy-digest
        trust path
    default: cutover
    why: One legacy ack exists; fail-closed until one re-ack, after which every workspace
      inherits it
    answer: cutover
  tui_perf_memory:
    ask: Should the epic add a default-executor rule for selection-driven computes
      to the tui_perf memory note?
    memory:
    - tui_perf.md
    default: false
    answer: false
phases:
- id: storm-hotfix
  title: Stop the hijack-decision storm
  depends_on: []
  size: medium
  description: 'storm-hotfix: gate passive footer scheduling with the cheap check.
    Key the decision cache on stable row identity plus a fingerprint instead of the
    roster generation. Run single-flight, latest-wins computes on one dedicated worker,
    and resolve sessions, clans, and running rows from the in-memory roster so the
    TUI never scans the archive.'
- id: telemetry-guardrails
  title: Executor-saturation gauge and telemetry retention
  depends_on: []
  size: small
  description: 'telemetry-guardrails: add a default-executor saturation gauge to the
    existing memory heartbeat, keep at least 24 h of tui_stalls rows, and cap the
    unbounded tui_trace file with rotation.'
- id: trust-memo
  title: Semantics-preserving trust and config memoization
  depends_on: []
  size: medium
  description: 'trust-memo: memoize the per-primary registry-membership scan and the
    lower-layer macro/workflow merge, and check the merged-config cache before reading
    and evaluating the local layer. An O(N) merge-count test guards construction.'
- id: trust-root
  title: One checkout root for trust enforcement and acknowledgment
  depends_on:
  - trust-memo
  size: medium
  description: 'trust-root: make enforcement and `sase config trust` derive the same
    checkout root and identity for canonical `<repo>/sase/sase.yml` configs, so primaries
    and numbered workspaces match their project acknowledgment. Handles the legacy
    key per the trust_ack_compat decision.'
- id: loader-memos
  title: Per-load memos for the Agents loader decode stage
  depends_on: []
  size: medium
  description: 'loader-memos: build the project alias map once per load, hoist the
    held-claim TTL config read, enrich each unique agent once with batch-scoped retention
    and artifacts-dir memos, and compute instruction-bundle file lists lazily for
    the detail panel.'
- id: indexed-name-lookups
  title: Index-backed agent name, session, clan, and dismissed-bundle lookups
  depends_on: []
  size: large
  description: 'indexed-name-lookups: replace the archive-walking named-agent, session,
    and clan lookups and the 65k-file dismissed-bundle rglob with indexed sase-core
    queries, so `sase agent hijack --explain` and the persist-hijack proc stop paying
    17-38 s.'
- id: provider-discovery-deferral
  title: Provider-tab discovery after first paint as one generation
  depends_on:
  - trust-root
  size: large
  description: 'provider-discovery-deferral: build fixed-tab bindings synchronously.
    Publish provider tab descriptors after first paint as one coherent generation
    for keymaps, Artifacts composition, help, the command catalog, and navigation,
    avoiding the deferral traps the research lists.'
- id: startup-critical-path
  title: Lazy action imports and startup background work off the critical path
  depends_on:
  - storm-hotfix
  - trust-memo
  - loader-memos
  size: large
  description: 'startup-critical-path: re-measure, then lazy-import the heavy agents
    action closure under the import-budget ratchet. Skip or defer startup monitor_reconcile
    when nothing can settle, defer AXE full snapshots until Services is visited, and
    make idle axe.collect ticks cheap.'
- id: heap-census
  title: Post-storm GC census and heap-owner attribution
  depends_on:
  - storm-hotfix
  - telemetry-guardrails
  - trust-memo
  - loader-memos
  size: medium
  description: 'heap-census: soak a separate TUI instance with the GC census at about
    10 min and 2 h, then record a ranked list of surviving heap owners with code anchors
    for the heap-diet phases. Coordinate with sase-1sl.4.'
- id: slim-list-rows
  title: Slim list rows end to end and a one-generation snapshot cache
  depends_on:
  - heap-census
  size: large
  description: 'slim-list-rows: return and consume a slim list DTO from sase-core
    instead of inflating full nested wire records for list paint. Hydrate only the
    selected identity, drop wire snapshots after Agent construction, and cap the snapshot
    cache at one live generation.'
- id: bounded-log-caches
  title: Replace whole-file log caches with targeted queries
  depends_on:
  - heap-census
  size: large
  description: 'bounded-log-caches: replace the module-level full-file materializations
    of the artifact index, artifact and memory read logs, bead touches, and stranded-report
    parses with targeted indexed or streamed queries, in census order.'
- id: stale-index-rows
  title: Terminalize stale running rows in index maintenance
  depends_on: []
  size: large
  description: 'stale-index-rows: find out why about 12.5k index rows stay running
    with no done marker, and terminalize them in sase-core background maintenance
    using real witnesses, without changing the Agents live predicate.'
- id: interaction-pass
  title: Interaction latency after the heap shrinks
  depends_on:
  - storm-hotfix
  - slim-list-rows
  - bounded-log-caches
  size: large
  description: 'interaction-pass: re-measure j/k and detail selection, then move detail-header
    lanes off the pump with per-agent caches and keep `,j` on the highlight path.
    Make clan runtime aggregation, delta-load graph copies, and bead-confirmation
    warmups proportional to changed or visible rows.'
- id: soak-verify
  title: End-to-end soak against the research baselines
  depends_on:
  - storm-hotfix
  - telemetry-guardrails
  - trust-memo
  - trust-root
  - loader-memos
  - indexed-name-lookups
  - provider-discovery-deferral
  - startup-critical-path
  - heap-census
  - slim-list-rows
  - bounded-log-caches
  - stale-index-rows
  - interaction-pass
  size: medium
  description: 'soak-verify: run a separate-instance soak with the whole epic, compare
    every stage''s telemetry against the research baselines and targets, document
    the new gauge and baselines in the perf runbook, and record residual gaps.'
proposed_by: bbugyi200.athena.research.62.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 07:57:48
status: done
bead_id: sase-1t3
---

- **PROMPT:** [prompts/202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/tui_storm_trust_heap_recovery.md)
- **BEAD:** [sase-1t3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/README.md)

# Recover TUI speed: stop the hijack storm, fix the trust-root defect, shrink the heap

## Problem

The consolidated research report
`research:202611/tui_hijack_storm_trust_root_and_heap/tui_hijack_storm_trust_root_and_heap.md`
(read it with `sase artifact read`) found three separate slowdowns stacked on top of
each other. Each one has a narrow fix:

| #   | Problem                                                  | Since                                    | Cost                                                                                                                                                                              |
| --- | -------------------------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A   | Hijack-decision storm                                    | `d26b581253` (2026-11-16)                | RSS 20-35 GB plus up to 30 GB swap; 31 of 45 threads stuck; 26% of wall time frozen; 50 s GC pauses; every other `asyncio.to_thread` user, including agent loads, waits behind it |
| B   | Trust/config root defect makes construction O(projects²) | `021cd73021` / `ea4121387b` (2026-11-12) | `app_construct_seconds` 0.7-1.0 s → 5.2-6.3 s; about 6 s more on cold agent-identity discovery                                                                                    |
| C   | Chronic heap plus per-load recompute                     | Gradual, Aug-Nov                         | gen-2 GC pauses of about 3 s; warm 131-145-row loads take 2-5 s; `agents_ready` about 11.6 s; j/k p95 400-500 ms                                                                  |

The research's recommended order is: A, then B (memo first, root fix second), then C1
loader memos, then startup deferrals, then the C2 heap diet, then interaction polish.
The phase graph below follows that order, but phases that touch different subsystems can
run in parallel.

## Verified at HEAD `0731d06c5a` (planner's own pass)

**Storm (A).**

- `_apply_agent_footer_update`
  (`src/sase/ace/tui/actions/agents/_display_detail_footer.py:173-175`) calls
  `_schedule_hijack_decision` without first checking the cheap gate.
  `hijack_ingress_available_for_footer` (`_hijack_decision.py:112-124`, called from the
  same footer at `:206-213` and from `actions/agent_workflow/_leader_mode.py:558-562`)
  schedules a second time.
- The cache key is `(identity, roster_generation)` (`_hijack_decision_cache.py:39-43`).
  `_finish_hijack_decision` (`:143-153`) discards a result once the generation has
  moved. `bump_roster_generation` (`_roster_generation.py:29`) runs on every roster
  assignment and every status mutation. So any compute slower than one refresh is never
  stored, and the cache never warms.
- Each compute calls `asyncio.to_thread` inside a pump-free task (`:122-126`). That uses
  the shared default executor and has no bound.
- `resolve_hijack_subject` (`src/sase/agent/hijack/facts.py:70`) makes more archive-wide
  lookups than the research names, not just `find_named_agent`:
  - `find_agent_session_by_member_artifact_dir` and `find_agent_session` go through
    `_iter_agent_session_members`.
  - `find_agent_clan` goes through `_iter_clan_members`.
  - Both of those helpers (`src/sase/agent/names/_lookup_groups.py:126,213`) `json.load`
    every ace-run `agent_meta.json` in every project.
  - `_running_rows` calls `list_running_agents()`.
  - Only `find_agent` and `list_running` go through the `StopDeps` seams
    (`src/sase/agent/hijack/_stop_models.py:51`). The session and clan lookups are
    imported directly.
- On a miss, `find_named_agent` (`_lookup_named.py:47`) runs an unindexed
  `scan_agent_artifacts`. It then `rglob`s and `json.load`s the dismissed bundles
  (`_lookup_named.py:195`; 65,875 files, 591 MB).
- `,k` submits `sase agent persist-hijack` as a durable subprocess proc
  (`src/sase/ace/tui/actions/agent_durable.py:118`). That proc re-resolves the target
  authoritatively before it acts, so the TUI's decision only advises the footer and the
  toast.

**Trust root (B).**

- `_checkout_root_for_local_path` (`src/sase/config/local_trust_enforcement.py:86`)
  probes `<repo>/sase`.
- `_is_project_root` (`src/sase/content_layout.py:405`) accepts any directory that
  contains `sase.yml` (the legacy layout), so discovery returns `<repo>/sase`. That
  makes the canonical-layout grandparent fallback at `:96-105` dead code.
- With the wrong root, `_project_identity`
  (`src/sase/config/local_trust_management.py:141`) never matches a primary. It falls
  into the all-projects registry scan, which runs a full
  `load_merged_config_for_local_path` plus registry reads for every registered project,
  on every evaluation.
- Every trust evaluation also re-merges lower layers in `_lower_macros_and_workflows`
  (`local_trust_enforcement.py:135`).
- `load_merged_config` (`src/sase/config/_sources.py:180`) runs
  `local_config_snapshot()`, which does a YAML read plus a full trust evaluation,
  _before_ it checks the cache.
- Identity derivation is Python-only today. The Rust side only decides and stores (no
  Rust twin), so fix it in place and do not port it in this epic.
- athena's trust store (`~/.sase/local_config_trust/records.json`) holds exactly one
  acknowledgment, keyed `checkout:v1:<primary>/sase`. Every numbered workspace prints
  `Local commands … were skipped (no-acknowledgment)`.

**Index (C).** sase-core's upsert writes `has_running_marker = record.running.is_some()`
(`crates/sase_core/src/agent_scan/index/maintenance.rs:653`). The research found the
column is 0 for all 23,133 rows, while 12,474 rows are
`status='running' AND has_done_marker=0`. Terminalization machinery already exists
(`terminalize_stale_active_agent_artifact_index_rows`, `maintenance.rs:167`; candidate
selection at `refresh.rs:331`).

## Non-goals and guardrails for every phase

- **Off the table:** no rewrite, no Ratatui/Rust UI port, no extra worker threads as a
  fix, no GC-threshold tuning or harder `gc.freeze`, and no archive deletion. All five
  research reports rejected these.
- **`tui_perf` contract:** follow `tui_perf.md` (read it through `/sase_memory_read`):
  - pump-free tasks, last-request-wins, and nav gates;
  - incremental `patch_row`;
  - no disk access on keystrokes;
  - no O(archive) work before ready.
- **Rust boundary:** shared backend behavior changes (slim DTO, index queries, index
  maintenance) go to linked `sase-core`. Open it with `sase repo open sase-core` and
  read its `AGENTS.md` first. Each change ships with the PyO3 binding change and a move
  of sase's `sase-core-revision.txt` pin (`docs/rust_backend.md`). The pin only moves
  forward: when two phases have moved it, keep the newer sase-core commit.
- **Measure, don't guess:**
  - Before/after numbers come from the existing telemetry:
    `~/.sase/logs/tui_startup.jsonl`, `tui_stalls.jsonl` (GC pauses, memory heartbeat),
    `tui_agent_loads.jsonl`, `~/.sase/perf/tui_jk.jsonl`, `SASE_TUI_TRACE=1` spans,
    py-spy, and the existing benches (`tests/ace/tui/bench_tui_jk.py`,
    `tests/perf/bench_tui_trace.py`, `docs/contribute/perf-runbook.md`).
  - Run benches on a quiet host. Do not sum startup JSONL fields that use different
    anchors.
  - Match the tested Textual and sase-core environment to the real TUI's (the research
    found a mismatch between the workspace venv and the installed build).
  - Record numbers in the phase bead's notes.
  - Several costs the research measured were inflated by the storm or the heap (decode 2
    s → 22 s, delta prep, 141 s warmups). Every phase after `storm-hotfix` re-measures
    first. If a target cost is already gone, the phase records that evidence and narrows
    its scope instead of building the fix anyway.
- **Human-only actions:**
  - Global reinstall (`just install`) and restarting the live TUI are human-only.
    Propose a reinstall only through `/sase_gate`.
  - Soaks run a _separate_ TUI instance, matching the accepted `sase-1sl` decision
    `live_tui_restart = separate_instance`.
- **Coordination:**
  - Agent Hijack epic `sase-1t2` is IN_PROGRESS. Its latch, gate, and monitor phases
    touch `_hijack_*` and `sase.agent.hijack`. Read its state, rebase onto its commits,
    and keep its ingress semantics, but do not wait for it.
  - `sase-1sl.4` (residual-leak attribution) is IN_PROGRESS. See `heap-census`.
  - Trust epic `sase-1sb` is closed.
- **Tests:** every phase adds focused unit tests and passes the project's standard
  verification (`lint_and_test.md`).

## Phase `storm-hotfix` — Stop the hijack-decision storm

Requirement: refreshes, cursor moves, and roster deltas can never stack hijack computes,
never use the default executor, and never run O(archive) work. The footer chip and `,k`
keep offering the same choices the backend makes today for every row kind the TUI can
show.

1. **Gate passive scheduling.**
   - Footer and leader paths request a decision only when
     `cheap_hijack_ingress_available(agent)` is true and the Agents tab is visible.
   - Schedule from one place per footer refresh, not both `_apply_agent_footer_update`
     and `hijack_ingress_available_for_footer`.
   - Explicit ingress still requests a decision through the same worker:
     `_start_hijack_agent` (`_hijack_action.py:28`) and kill-modal `h`
     (`_kill_action_flow.py:158`).
2. **Stable cache key.**
   - Key = row identity plus a decision fingerprint. The fingerprint covers every
     row-local field the row-local resolver reads: status, pid, workspace_num,
     artifacts_dir, agent_session/clan fields, container flags, and `hijack_*` fields.
     For container rows, include the member rows' fingerprints. The roster generation is
     never part of the key.
   - A finished result is stored whenever its key is still current. Generation moves no
     longer discard results.
   - Keep the LRU bound.
   - Add a short maximum age (≤30 s, revalidated in the background while the cached
     value is served) for facts that live only on disk (claims, `agent_meta.json`, done
     markers). This follows `tui_perf` rule 10.
3. **One dedicated worker, latest-wins.**
   - Use one app-owned `ThreadPoolExecutor(max_workers=1)`, created lazily and shut down
     with `cancel_futures=True` at teardown, following the precedent in
     `src/sase/ace/tui/models/_loaders/_json_cache.py:85-115`.
   - Await it from the pump-free task through `run_in_executor(<dedicated>, …)` or
     `asyncio.wrap_future`, never `asyncio.to_thread`.
   - At most one compute runs and one passive request waits. A newer passive request
     replaces the waiting one.
   - Explicit requests keep their `then` callbacks and go ahead of passive ones.
   - Keep the single-flight-per-key behavior.
4. **Row-local resolution.**
   - Add optional `StopDeps` seams for the lookups `resolve_hijack_subject` imports
     directly: session by member artifact dir, session by name, and clan by name.
     `find_agent` and `list_running` already have seams.
   - The defaults keep today's archive-backed behavior, so `sase agent hijack --explain`
     and the `persist-hijack` proc stay authoritative.
   - On the UI thread at scheduling time, the TUI captures an immutable snapshot of the
     roster rows it needs: the selected row, its session and clan members, and live
     rows. It passes deps that answer from that snapshot.
   - Remaining work is O(session members): per-member `agent_meta.json`, done markers,
     claims, and liveness.
   - When the snapshot cannot answer, the TUI path returns what it has. It never falls
     back to `find_named_agent`, `iter_ace_run_artifact_dirs`, `scan_agent_artifacts`,
     or the dismissed-bundle rglob.
5. **Tests.**
   - Storm regression: 50 footer refreshes with roster-generation bumps against a slow
     fake decider leave at most one compute in flight and one queued.
     `asyncio.to_thread` and default-executor submission are patched to fail.
   - Cache warm-up: after a generation bump, a refresh with an unchanged row fingerprint
     hits the cache, and a late result still refreshes the footer for the selected row.
   - Parity: extend `tests/ace/tui/test_agent_hijack_tui_parity.py`. On a fixture
     archive, row-local decisions match default-deps decisions (mode and code) for a
     plain running agent, a session container with members, a clan container, a terminal
     row, a named proc, a remote row, and a WAITING/booting workspace holder.
   - Teardown cancels the worker.
6. **Verify:** use a separate TUI instance, or py-spy against a fixture-driven run. At
   most one `sase-hijack-decision` thread appears, and heartbeat RSS stays near the
   pre-storm baseline of about 4 GB. Record the numbers.

> [!decision] hotfix_reinstall
>
> After the phase's change is in place, end the turn through `/sase_gate`. The gate
> proposes the human-only global reinstall (`just install` from the primary checkout
> once the hotfix commit is on master), so the live TUI stops storming after Bryan
> restarts it. Never run the install directly.

> [!decision] hotfix_reinstall = no
>
> Propose no reinstall. The fix reaches the live TUI whenever the human next reinstalls.

> [!decision] tui_perf_memory
>
> Add one rule to `tui_perf.md`: off the event loop is not off the default executor.
> Computes driven by selection or refresh run single-flight per row, keyed on stable
> identity rather than a roster generation, on a bounded dedicated worker, and never do
> O(archive) work. Then run `sase memory init`.

## Phase `telemetry-guardrails` — Executor-saturation gauge and telemetry retention

The storm ran for an hour and nothing in the logs named it. This phase adds the
research's single instrumentation guardrail plus two retention fixes. It must add no
per-keystroke work.

1. **Executor gauge.** Register a provider through `register_heartbeat_provider`
   (`src/sase/ace/tui/util/gc_telemetry.py:119`) that adds one dict per
   `tui_memory_heartbeat` row:
   - max workers, busy workers, queue depth, and oldest queued age, for the loop's
     default executor;
   - the same fields for the `sase-loader` executor;
   - the same fields for the `storm-hotfix` dedicated worker, if it exists.

   For example, install an app-owned instrumented default executor with
   `loop.set_default_executor` at startup. Keep the overhead to a counter per submit.

2. **Stall-log retention.** `~/.sase/logs/tui_stalls.jsonl` currently keeps about 2 MB ×
   2 files, roughly 6 h, which lost the Nov 11 spike. Keep at least 24 h at observed
   rates.
3. **Trace-file cap.** `~/.sase/perf/tui_trace.jsonl` has no bound (585 MB on athena).
   Add size-based rotation.
4. **Tests and docs.** Add tests for the gauge fields under a saturated fake executor
   and for both rotation policies. Document the gauge in
   `docs/contribute/perf-runbook.md`.

## Phase `trust-memo` — Semantics-preserving trust and config memoization

Requirement: construction-time config work is O(projects), not O(projects²). No trust
verdict, identity, or digest may change.

1. **Memoize `_project_identity`.** Cache the per-primary registry-membership result for
   each primary: the merged config, the `WorkspaceStore`, and the registry read. Key it
   on the primary path, a project-records token, the registry file's `mtime_ns`/size,
   and the config token. Memoize `list_project_records` per project-records token.
2. **Memoize `_lower_macros_and_workflows`.** Key it per `current_config_token()` plus
   the default/plugin cache generation. Preserve `_trust_evaluation_guard` reentrancy.
3. **Reorder `load_merged_config`.** Check a cheap first-level key before
   `local_config_snapshot()`. The key is the config token, the local path, the local
   file's stat signature, and the trust store's stat signature. Only a miss reads YAML
   and evaluates trust. `sase config trust allow/revoke` and config edits must still
   invalidate, through stat drift or an explicit `clear_config_cache()`.
4. **Tests.**
   - Count `merge_config_sources` calls while constructing `AceApp` (or
     `init_late_startup_state` → `build_app_bindings`) with N fake registered projects,
     and assert O(N), not a time limit.
   - Assert that a cache-hit `load_merged_config()` neither loads YAML nor evaluates
     trust.
   - Allow, revoke, and edit sequences invalidate correctly.
   - Existing trust suites pass unchanged.
5. **Verify:** `app_construct_seconds` p50 moves from about 5.2 s toward ≤2.5 s. The
   cold `provider` substage in `tui_agent_loads.jsonl` drops.

## Phase `trust-root` — One checkout root for trust enforcement and acknowledgment

Requirement:

- Enforcement (`_checkout_root_for_local_path`, `_current_selected_config_path`) and
  management (`_requested_checkout`, `local_trust_management.py:88`) resolve the same
  checkout root, and therefore the same identity, logical path, and digest, for the same
  selected config.
- For canonical `<repo>/sase/sase.yml`, the root is `<repo>`. For legacy
  `<repo>/sase.yml`, the root is `<repo>`.
- A primary checkout matches its project record. A registered numbered workspace
  resolves to the project identity through the registry. An unregistered checkout gets
  checkout scope.
- The `skipped (no-acknowledgment)` hint names a command that records the key
  enforcement actually reads.

Approach:

- Add one shared helper, "owning checkout root for a selected config path". It walks up
  from the config file and picks the nearest ancestor whose
  `resolve_project_config_read_path` selects exactly that file, using VCS markers to
  break canonical-versus-legacy ties.
- Do not change `discover_project_root` globally; it has many unrelated callers.

Test matrix:

- primary;
- numbered workspace;
- unregistered checkout;
- symlinked checkout;
- a repo literally named `sase`, including `/x/sase/sase.yml` ambiguity;
- canonical and legacy collision (`LayoutCollisionError`);
- enforcement/acknowledgment identity parity on each case.

Fail-closed behavior must hold whenever identity derivation errors.

> [!decision] trust_ack_compat = cutover
>
> Identity keys change without a compatibility branch, and no feature flag is needed.
> Fail-closed behavior skips local commands in the primary until the one legacy
> acknowledgment is re-recorded. Print a clear one-line upgrade hint naming
> `sase config trust allow` for the primary. Record in the bead note that Bryan must
> re-run it once per host (athena; apollo/mac if they hold legacy keys). After that,
> every registered workspace inherits it.

> [!decision] trust_ack_compat = compat
>
> Create a `sunset` flag with `sase flag new` (read `sase_flags.md` first). With the
> flag on, when the new identity has no acknowledgment, look up the legacy
> `checkout:v1:<root>/sase` key and accept it only if its digest matches the digest
> recomputed with the legacy logical path (`sase.yml`). Never widen trust beyond what
> that legacy record covered. Test both flag states, and keep the off branch deletable.

Verify: `app_construct_seconds` approaches about 1 s, and the cold `provider` substage
drops below 1 s. `sase` commands in a numbered workspace stop printing the skipped
warning once the primary is acknowledged.

## Phase `loader-memos` — Per-load memos for the Agents loader decode stage

Requirement: one Agents load does each piece of per-project or per-config work once, not
once per agent. Rows render identically.

- `load_project_alias_map` (`src/sase/project_aliases.py:239`, reached through
  `src/sase/macro/project_identity.py:13`) resolves about 303-322 times per load, about
  44% of decode. Build it once per load, or memoize it per project-records token.
- `configured_held_claim_ttl_days()`
  (`src/sase/workspace_provider/retention_record.py:186`) runs a full
  `load_merged_config()` for each retention agent (7 × 0.42 s). Hoist it to once per
  load.
- `enrich_agent_from_meta_wire`
  (`src/sase/ace/tui/models/_loaders/_meta_enrichment_wire.py:57`) runs about 3 times
  per agent. Run it once per unique agent, and memoize `load_agent_workspace_retention`
  (`models/agent_workspace_retention.py:30`) and `get_artifacts_dir`
  (`models/artifact_files.py:34`) per batch.
- Instruction-bundle file lists are built about 268 times per load. Compute them lazily
  for the detail panel.

Prefer memos scoped to one load over module globals (`tui_perf` rule 8: over-broad keys
serve stale rows). Tests count each call per load with N agents. The loader oracle and
parity suites must stay green. Verify through the `decode` substage in
`tui_agent_loads.jsonl`, plus the tiering bench: warm full load about 5 s → about 1 s.

## Phase `indexed-name-lookups` — Index-backed agent name, session, clan, and dismissed-bundle lookups

The TUI stops using these lookups after `storm-hotfix`. The CLI and the `persist-hijack`
proc started from `,k` still pay 17-38 s per call.

- Back `find_named_agent` / `_find_named_agent_candidates` (`_lookup_named.py:47,86`)
  and the session and clan member iterators (`_lookup_groups.py:126,213`) with indexed
  queries on sase-core's agent artifact index.
- Add a dismissed-bundle index (name and workflow name → bundle) that dismissal
  maintains, replacing the rglob at `_lookup_named.py:195`.
- Keep results identical: exact before workflow match, live before dead before done,
  dismissed-prefix handling, and current-owner name candidates. Prove it with parity
  tests against the scan implementation on fixtures.
- Index-missing and stale-schema states follow the established cheap-detect plus bounded
  fallback pattern.
- Target: warm `sase agent hijack --explain NAME` resolution < 1 s on athena.
- Scope: core, binding, and pin move.

## Phase `provider-discovery-deferral` — Provider-tab discovery after first paint as one generation

`AceApp.__init__` runs this chain synchronously and resolves every project's providers
to generate numbered Artifacts shortcuts:

- `init_late_startup_state` (`actions/_state_init_late.py:34`)
- `build_app_bindings` (`keymaps/bindings.py:54`)
- `resolve_artifacts_subtabs` (`artifact_tabs.py:89`)
- `load_project_provider_records` (`_artifact_tab_discovery.py:42`)

Re-measure after `trust-root` first. If construction is already about 1 s, this phase
becomes protection against project-count growth, and its scope should shrink to match.

Requirement:

- Build fixed-tab bindings synchronously.
- Publish provider descriptors after first paint as **one coherent generation**,
  consumed by keymaps, Artifacts composition, help, the command catalog, and navigation.

Avoid these traps from the research:

- `ArtifactsView.__init__` also resolves subtabs while hidden.
- Early digit shortcuts must resolve against the same generation.
- An empty cache entry must never read as authoritative discovery.
- Cold `AgentIdentitySnapshot.current()` shares the path. Deferring keybindings alone
  just moves the cost.

## Phase `startup-critical-path` — Lazy action imports and startup background work off the critical path

Re-measure each item at HEAD first. Some deferral machinery already exists
(`_flush_deferred_monitor_reconcile`, `actions/_startup_loads_maintenance.py:392`), so
drop any item whose cost has already gone.

- **Lazy imports.** About 1.4 s of the 2.1 s module import is `actions.agents._core`
  pulling `gate_turn`, `notification_gates`, and `macro.models` eagerly. Make them lazy
  inside the existing import-budget ratchet (`tests/ace/tui/test_app_import_budget.py`),
  and lower the budget.
- **Startup `monitor_reconcile`.** It took 11-15 s with `settled: 0`. Skip it when the
  roster signature shows nothing can settle, and defer it past `visible_ready`. If a
  scan is truly needed, make it a bounded index query.
- **AXE.**
  - Startup currently collects full snapshots across 63 chops (about 1,349 file reads,
    about 4.1 s). Defer full snapshots until the Services tab is visited (see
    `_run_axe_startup_init`, `axe_display/_refresh_full.py:99`, and the `axe_full` rule
    in `event_refresh/_auto_refresh_surfaces.py:239`).
  - Idle `axe.collect` ticks spend 2-3 s of CPU with zero file opens. Cache
    `load_axe_config` (`src/sase/axe/config.py:99`) per config token, and skip
    orchestrator probes on ticks where the token is stable.
- **Targets.** process→`on_mount` < 3 s p95; `axe_ready` 3-4 s earlier; quiet ticks near
  zero CPU. Measure with `tui_startup.jsonl` and `SASE_TUI_TRACE` spans.

## Phase `heap-census` — Post-storm GC census and heap-owner attribution

The only existing census was taken mid-storm and overstates wire records. The census
proves a large surviving graph exists, but not who owns it, so identify owners before
the invasive heap-diet phases.

1. Read the `sase-1sl.4` bead and its notes first. If it already holds a post-storm
   census from a build that includes `storm-hotfix`, reuse it. Never change its status.
2. Otherwise, run a separate TUI instance of this epic's code on athena with
   `SASE_TUI_GC_CENSUS=1` (Python 3.14+, see `src/sase/ace/tui/util/gc_policy.py`),
   using `/sase_monitor` for the long waits. Take censuses at about 10 min and about 2
   h, along with:
   - heartbeat RSS and allocated blocks;
   - gen-2 pause p50/p95;
   - the new executor gauge.
3. Attribute the survivors:
   - `AgentArtifactRecordWire` count versus visible rows;
   - snapshot-cache generations (the research saw 8 fat copies);
   - module-level full-log caches: `artifacts/index.jsonl` (`ArtifactFile` 46.9k ×2),
     `artifact_reads.jsonl` (27k), `memory_reads.jsonl` (23k), `agent_bead_touches.json`
     (16k), and stranded-report issue parses (8.5k);
   - full-history reconciliation, pending-worker snapshots, catalogs, and duplicate
     roster or detail representations.
4. Write a ranked owner list (retained objects and bytes, plus code anchors) as an
   epic-bead note for `slim-list-rows` and `bounded-log-caches`. Add a cross-reference
   note on `sase-1sl.4`. Code changes are limited to fixing census tooling, if it is
   broken.

## Phase `slim-list-rows` — Slim list rows end to end and a one-generation snapshot cache

sase-core already offers `record_shape="list"`
(`src/sase/agent/listing_snapshot.py:103`,
`models/_agent_loader_artifact_index.py:237`). Python still inflates full nested
`AgentArtifactRecordWire` trees through `_record_from_dict`
(`src/sase/core/agent_scan_wire_conversion.py:358`) for list paint.

- Return and consume a slim DTO: identity, status, markers, names, timestamps, and
  tribe/clan scalars.
- Hydrate the full record only for the selected identity, building on the
  `_projected_record.py` hydration path.
- Drop wire snapshots after `Agent` construction.
- Cap the snapshot cache the census names at one live generation.
- Ship core, binding, and pin move in one change set.
- Target: `AgentArtifactRecordWire` survivors O(visible rows). List rendering, filters,
  grouping, and incremental `patch_row` paths behave the same.

## Phase `bounded-log-caches` — Replace whole-file log caches with targeted queries

Work in the census's ranked order. Replace whole-file Python materialization with
targeted indexed or streamed queries, Rust-side where the behavior is shared.
Candidates:

- the artifacts index (`artifacts/index.jsonl`);
- `artifact_reads.jsonl`;
- `memory_reads.jsonl`;
- `agent_bead_touches.json`;
- stranded-report issue parses.

Render paths stay stat- and glob-free (`tui_perf` rule 8). These sources feed the
detail-header lanes that `interaction-pass` moves off the pump. Keep their per-agent
query shape, so that phase can key caches per agent.

## Phase `stale-index-rows` — Terminalize stale running rows in index maintenance

Work in sase-core.

- Explain why about 12.5k rows stay `status='running'` with no done marker while
  `has_running_marker` (written as `record.running.is_some()`) is 0 everywhere.
  - Do runners still write `running.json`?
  - Does the oldest-first, `LIMIT`-bounded candidate selection (`refresh.rs:331`) starve
    on rows it keeps skipping?
- Terminalize them in coalesced background maintenance, off the TUI, using real
  witnesses: a done marker on disk, PID/process-identity liveness, and live workspace
  claims.
- Do **not** switch the Agents live predicate to require `has_running_marker=1`; today
  that would empty the inbox. Record a follow-up only if the writer now populates
  markers.
- Target: stale running rows fall to roughly the truly live count, and the reduction in
  revalidate churn is measured.

## Phase `interaction-pass` — Interaction latency after the heap shrinks

A 16 ms key-to-paint budget cannot be met on a multi-GB heap, so re-measure with
`SASE_TUI_PERF=1` and `SASE_TUI_TRACE=1` first. Then address:

- **Detail-header lanes.** `build_detail_header_summary` runs p50 0.5-1.25 s, p95 up to
  18.6 s.
  - Resolve `artifact_reads`, `memory_reads`, `agent_page_url`, and `skill_uses` off the
    pump with per-agent caches keyed by mtime.
  - Paint the chrome immediately and fill lanes as they land.
  - Skip unchanged content by stable content identity before building or hashing large
    renderables.
- **`,j`.** It runs p95 about 1.5 s. Keep it on highlight plus debounced detail, never a
  full refresh.
- **`aggregate_clan_runtime` on timer ticks.** Compute only changed or visible
  durations, or batch the work in Rust. Interval-union semantics must not change.
- **Delta-load `prep`.** `copy_agent_graph` takes 2.3-2.9 s for 2-8 changed rows. Switch
  to targeted per-row copy-on-write.
- **`bead_confirmation_warmup`.** It runs 141-161 s under contention. Bound it to
  visible and WAITING rows, and make stranded sweeps lazy.

Targets: j/k p95 toward 16 ms (`tui_jk.jsonl`, `bench_tui_jk.py`), and detail selection
under 100 ms.

## Phase `soak-verify` — End-to-end soak against the research baselines

Run a separate-instance soak of at least 1 h on athena under real load with the whole
epic, using `/sase_monitor` for the waits. Record a stage-by-stage table in the epic
bead comparing the research baseline, the target, and the measured value:

| Stage             | Measure                                                                              | Target                                           |
| ----------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------ |
| Storm             | hijack threads (py-spy), RSS, default-executor gauge, ≥1.5 s hitches near keypresses | ≤1 hijack thread; no default-executor saturation |
| Construction      | `app_construct_seconds`                                                              | about 1 s                                        |
| Provider substage | cold `provider` substage                                                             | < 1 s                                            |
| Loads             | warm full load / `agents_ready`                                                      | about 1 s / about 5-6 s                          |
| Startup           | process→`on_mount`                                                                   | < 3 s p95                                        |
| Heap              | idle RSS after 30 min, gen-2 pause p95, allocated blocks, hitch share of wall        | < 1.5-2 GB; ≤ 0.5-1 s; < 8M; < 0.5%              |
| Interaction       | j/k p95, detail selection                                                            | toward 16 ms; < 100 ms                           |
| Overall           | `sase tui` to a usable Agents list                                                   | about 6-7 s or better                            |

Update `docs/contribute/perf-runbook.md` with the recovered baselines (read
`docs_writing.md` first). Record each missed target as a `PROPOSED FOLLOW-UP:` note with
its evidence.

## Human-only operational notes (not phase work)

- Until `storm-hotfix` is installed, quit the live TUI when it is not in use. The
  research's alternative is pinning the global install to `082f2bc705`, the last steady
  build, through a gate.
- Unset `SASE_TUI_GC_CENSUS` on the live TUI. The census itself is an 11.5 s UI-thread
  walk.
- After `trust-root` lands under `cutover`, re-run `sase config trust allow` once in the
  primary checkout on each host that had acknowledged local commands.
