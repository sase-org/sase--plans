---
tier: epic
title: Finish update-skew agent auto-restart so it is safe and actually relaunches
goal:
  "Complete epic sase-1j6: the update-skew healer really relaunches pre-provider skew
  deaths once under the same name, touches nothing but skew-shaped failures, never spams
  or resurrects rows, records honest timestamps and provenance, keeps the episode report
  live, and leaves just check free of epic-caused failures."
parent_bead: sase-1j6
decisions:
  quiet_declines:
    ask:
      Should a declined skew-shaped failure that its runner did not silence also get a
      loud second notification?
    choices:
      quiet:
        No - dim TUI decline hint only; the user already got the normal failure notice
      loud: Yes - re-surface it loudly; one extra notification per such declined failure
    default: quiet
    why: a runner that did not silence its failure already notified the user
    answer: quiet
phases:
  - id: sweep-safety
    title:
      Restrict healer targets to skew-shaped failures and stop loud or phantom side
      effects
    depends_on: []
    size: medium
    description:
      "sweep-safety: make the scheduler job and `run -p` consider only doorbells,
      in-flight recovery rows, skew-suspect rows, and recent legacy skew-shaped rows;
      never claim, write, or notify for any other failure; stop write_recovery from
      recreating missing or wiped artifacts dirs; delete doorbells once owned; make the
      disabled and paused paths resurface each silenced failure exactly once; trip the
      storm breaker once; keep idle ticks cheap."
  - id: core-fixes
    title: sase-core classifier, ledger timestamp, meta wire, and notification fixes
    depends_on: []
    size: medium
    description:
      "core-fixes: in sase-core, fix workspace scoping with trailing slashes, stop
      log-tail substrings from proving managed origin, never relaunch an unknown phase,
      extract the missing symbol from the error line, stamp claimed_at and history
      times, carry agent_meta.auto_restart on AgentMetaWire through the scanner and
      index, route agent.auto-restart error rows to the Errors tab, and let notification
      reconcile refresh files."
  - id: probe-quiescence
    title: Correct probe module names and quiescence code-change times
    depends_on: []
    size: small
    description:
      "probe-quiescence: map frame paths in src-layout editable checkouts to importable
      module names so the W4 probe can pass, and measure managed-root code changes from
      ref and reflog mtimes instead of commit times."
  - id: runner-ux-fixes
    title:
      Runner refresh imports, lifecycle facts, config, UX polish, and epic-caused test
      failures
    depends_on: []
    size: medium
    description:
      "runner-ux-fixes: remove the remaining late imports before os.execv, fix
      lifecycle-phase and failure-facts capture gaps, move spare_process_patterns back
      under agent_scope_teardown, use UPDATE_RECOVERY_GLYPH everywhere, and fix the
      epic-caused completion, parser, schema, vocabulary, wire, timezone, lock-path, and
      import-budget test failures."
  - id: healer-relaunch
    title: Make the healer relaunch, settle, and escalate correctly end to end
    depends_on:
      - sweep-safety
      - core-fixes
      - probe-quiescence
    size: medium
    description:
      "healer-relaunch: move the sase-core pin past core-fixes, re-classify with the
      probe witness so a passing probe relaunches, pass real timestamps, escalate
      expired deferrals, take over stale claims properly, escalate a replacement's
      second failure as already_restarted, write in-flight recovery states with
      ledger_key and episode_id, record real from/to revisions in provenance, scrub the
      provenance env, order -p targets topologically, and prove it with an un-mocked
      classifier test."
  - id: episode-polish
    title:
      Live episode report on settlement, honest titles, and per-death escalation keys
    depends_on:
      - healer-relaunch
    size: medium
    description:
      "episode-polish: refresh the live report and the row's inline snapshot on every
      job settlement (consuming refresh_episode_report and dropping its epic-symbol
      row), map every terminal outcome in the Now column, fix episode titles and the
      unknown-episode fallback, key escalations per death, carry every relaunched
      agent's evidence files, and apply the quiet_declines decision."
  - id: acceptance
    title: Real end-to-end incident replay, host dry run, audits, and docs
    depends_on:
      - runner-ux-fixes
      - episode-polish
    size: medium
    description:
      "acceptance: rewrite the incident replay so doorbell, job tick, healer,
      classifier, settlement, report, and escalation all run through real code, dry-run
      the healer over the host corpus, review the new marker audit sites, update
      docs/agent_auto_restart.md, and leave just check with no epic-caused failures."
proposed_by: bbugyi200.athena.sase-1j6.land
decided_by: reviewer
decided_via: tui
create_time: 2026-10-10 08:09:03
status: wip
---

- **PROMPT:**
  [prompts/202610/finish_update_skew_auto_restart.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/finish_update_skew_auto_restart.md)
- **PARENT:**
  [202610/update_skew_agent_auto_restart.md](https://github.com/sase-org/sase--plans/blob/main/202610/update_skew_agent_auto_restart.md)

# Plan: Finish update-skew agent auto-restart

## Context

Epic `sase-1j6` (plan `plan:202610/update_skew_agent_auto_restart.md`) closed all nine
phases. Its land agent verified the source and found that the feature is not done:

- It cannot relaunch anything in real use.
- Installed as it stands, it would send loud notifications for every failed agent of the
  last week and recreate deleted artifacts directories.

Read the parent plan with
`sase artifact read plan:202610/update_skew_agent_auto_restart.md "<why>"` for the full
design: shared contracts, signature catalog, UX spec, and CLI. Every phase here must
still honor it. The parent epic's DECISIONS stay final: no decisions-web record. That
proposal is task `sase-1ji`.

The installed `sase` tool on athena does not yet ship the `agent_auto_restart` job. The
first `just install` of current master activates it. Phase `sweep-safety` therefore has
no dependencies and should land first.

### Verified defects (master `05bab368af`)

1. **Never relaunches.**
   - Without a W4 witness the core returns `defer` (`probe_pending`).
   - `_heal_claimed` in `src/sase/agent/auto_restart/healer.py` then runs the probe, but
     still defers because `verdict.mode == "defer"`.
   - It never re-classifies with the probe result. Every healer test injects
     `classify=lambda: relaunch`, so the real path is untested.
2. **Touches every failure.**
   - `resolve_pending_targets()` feeds the healer every failed `done.json` row and every
     dismissed bundle from the last 7 days, through
     `history.collect_failed_candidates(since_seconds=7*86400)`.
   - Each non-skew row gets a ledger record (`declined`), a `done.json` `recovery`, and
     a loud `resurface_failure()` notification. Ledger records for non-skew rows also
     spend the lineage of `llm_provider` retry chains.
3. **The disabled/paused path is worse.** `sweep._resurface_for_disabled` resurfaces and
   declines every target on each 60-second full sweep.
4. **Phantom rows.** `write_recovery` uses `atomic_write_json`, which runs
   `mkdir(parents=True)`. Writing to a dismissed or wiped artifacts dir recreates it
   with a stray `done.json`. This includes `write_recovery(target, "launched", ...)`
   right after `execute_agent_restart` wiped the failed row.
5. **Doorbells are never deleted once claimed.** `actionable` stays true, so the job
   submits a healer proc on every tick.
6. **Ledger timestamps are always null.**
   - `claimed_at` and `history[].at` are never set: core's `claim_auto_restart_ledger`
     and `advance_auto_restart_ledger` push `at: None`.
   - As a result `max_defer_seconds` never fires, so deferred rows loop forever.
   - The 30-minute storm limit never trips.
   - The crash rule compares against epoch 0.
7. **A replacement's second failure is swallowed.**
   - The replacement's lineage maps to the same ledger key, so `heal_one` returns
     `_recover_existing_claim`'s `already_launched` / `settled_*`.
   - It sends no escalation and writes no recovery, and the doorbell stays.
   - The loud `already_restarted` skip rule is unreachable.
   - The incident replay test hides this by calling `_escalate` by hand.
8. **`claimed` recovery state.** The healer writes `recovery.state = "claimed"`, which
   is not an in-flight state. The row flickers to FAILED and waiters treat it as
   terminal.
9. **Provenance.**
   - The Rust `AgentMetaWire` has no `auto_restart` field, so production rows never show
     the ↻ chip, provenance block, or `v` hint.
   - `_attach_provenance` always writes `from_rev`/`to_rev = None`, and it reads
     `file_proof` from `verdict.witnesses`, which does not exist.
   - `SASE_AUTO_RESTART_PROVENANCE` is never removed from the environment, so agents the
     replacement launches inherit ↻ provenance and its lineage.
10. **Core classifier bugs** (`crates/sase_core/src/agent_auto_restart/`):
    - A trailing `/` on `workspace_dir` defeats workspace scoping; 197 of 229 real rows
      have one.
    - `find_module` searches `log_tail`, so a third-party `No module named 'requests'`
      looks managed whenever the tail mentions a `sase.*` logger.
    - The `unknown` phase falls through to `relaunch`.
    - `missing_symbol` takes the first quoted token anywhere; all 15 post-provider rows
      report `main`.
11. **Probe module names.** For the editable checkout, frame paths become
    `src.sase....`, so the probe import always fails.
12. **Report never settles.** `sweep._settle_launched_records` never calls
    `notify.refresh_episode_report`, which is still on the Justfile `--epic-symbol`
    whitelist. The row's inline report snapshot never refreshes either.
13. **Storm breaker.**
    - It escalates once per agent after a trip, because its dedup key includes the agent
      name.
    - Its row lands in the `sase-update` tab: the Rust tab classifier in sase-core
      `mobile.rs` counts only `axe` and `user-agent` as errors.
14. **Late imports before `os.execv`.** `refresh_runner_code_after_wait()` still
    late-imports two modules on the path to the exec. The pytest firewall misses both
    because one import happens only outside pytest and the other is mocked.
    - `sase.core.managed_tmp_roots`, reached through `serialize_local_macros` →
      `get_sase_managed_tmpdir`.
    - `sase.agent.launch_timing`, reached through
      `planned_name_is_reserved_for_artifacts`.
15. **Epic-caused test failures.** 10 of the land check's 16 failures are caused by the
    epic alone and 2 partly. Against the true pre-epic base `70c51adbdd`:
    - two completion snapshots
    - completion mutex groups (21 vs 20)
    - agents help sort
    - config schema: `spare_process_patterns` moved under `agent_auto_restart` in
      `src/sase/default_config.yml`
    - query-profile status vocabulary (`RESTARTING`)
    - `test_finalizer_status_is_trailing_wire_field`
    - marker mutation audit (`healer.py:_write_evidence`)
    - timezone display guard (`auto_restart/ux.py` uses `datetime.now().astimezone()`)
    - pypi lock path (`code_swap_lock_path` rename)
    - partly: marker path audit (`healer.py:_runner_log_tail`)
    - partly: TUI import budget

    Failures that are not this epic's are tracked elsewhere:
    - `sase-1jj`: bob dry run
    - `sase-1ic`: non-epic import-budget growth
    - `sase-1by`: `session_root_tab` audit rename
    - `sase-1gh`, `sase-1jk`, `sase-1jl`: flakes

## Shared rules for every phase

- **Candidate rule.** A failed row is a healer candidate only if at least one of the
  following holds:
  1. It has a doorbell.
  2. Its `done.json` `recovery.state` is in flight (`pending`, `deferred`, or
     `launching`).
  3. Its `failure_facts.skew_suspect` is true.
  4. It is a **legacy** row: no `failure_facts` and no `recovery`, it died within
     `max_defer_seconds` of the tick, and its error, traceback, or runner-log tail
     matches the Tier 1–3 prefilter. Reuse `facts_look_like_update_skew` or an
     equivalent text prefilter.

  Dismissed bundles are never healer candidates; `scan` may still read them. Rows that
  are not candidates get no ledger record, no `done.json` write, and no notification.

- **Silenced vs not silenced.** A row was silenced when its runner dropped a doorbell or
  wrote `recovery.state = "pending"`. Its failure notification was then sent
  `silent=True`.
  - Every terminal outcome of a silenced row must be made visible exactly once, with the
    parent plan's escalation copy.
  - A row that was not silenced already notified the user. Decision `quiet_declines`
    decides whether it gets a second notification.

> [!decision] quiet_declines = quiet Not-silenced skew-shaped declines get only the dim
> Agents-tab decline hint. The ledger records the decline. No `resurface_failure` or
> escalation is sent.

> [!decision] quiet_declines = loud Not-silenced skew-shaped declines also go through
> the escalation path, exactly as silenced ones do.

- `write_recovery` and every other `done.json` writer in `src/sase/agent/auto_restart/`
  writes only when `done.json` already exists. It never creates directories, and it
  writes nothing under `--dry-run`.
- Read the `tui` memory before touching TUI code, `cli_rules` before CLI changes, and
  `lint_and_test` before finishing. Every phase runs `sase tool run check` in this repo
  and does not run `just check-full`. A phase that edits sase-core also runs
  `sase tool run check` in the sase-core checkout (`sase repo open sase-core`; read its
  `AGENTS.md`).
- New public symbols need a real consumer in the same phase, or an `--epic-symbol` row
  keyed to a still-open bead of this child epic. Never key one to `sase-1j6`.

## Phase sweep-safety

Files: `src/sase/agent/auto_restart/{history,sweep,healer,ledger}.py`, plus tests.

- **Candidate enumeration.** Implement the candidate rule in one shared function used by
  `resolve_pending_targets()` (`run -p`) and `sweep._collect_job_work()`.
  - Use the agent artifact scan facade within the bounded lookback instead of
    `history._iter_done_candidates`'s raw walk of every project's artifacts tree.
  - `scan` keeps its own read-only history path, including dismissed bundles.
- **Idle cost.** Idle ticks must stay at a handful of `stat()` calls. Do not parse every
  ledger file on every tick: gate ledger reads on the ledger directory's mtime, or keep
  a small index. Add a test that bounds idle-tick filesystem calls.
- **Healer pre-check.** Before claiming a ledger record, `heal_one` re-checks that the
  target satisfies the candidate rule. Non-candidates return
  `HealerOutcome(action="skipped", reason="not_update_skew")` with no side effects.
- **Doorbells.** Delete a target's doorbell once a ledger record owns its failure (at
  claim), and when the target is skipped or declined. The job's `actionable` must turn
  false after one healer pass over a handled doorbell.
- **Disabled and paused paths.** `_resurface_for_disabled` only handles silenced rows
  (doorbell or `recovery.state == "pending"`). It resurfaces each one exactly once,
  using a stamp like the existing `resurfaced/` stamps, marks `recovery` declined, and
  clears the doorbell. It never touches other failed rows.
- **`write_recovery` safety.**
  - Return without writing when `done.json` is missing.
  - Drop the post-execute `write_recovery(target, "launched", ...)`; the ledger owns
    `launched` once the old row is wiped.
  - Make every dry-run path write nothing, including the expired-deferral branch of
    `_recover_existing_claim`.
- **No `claimed` state.** Stop writing `recovery.state = "claimed"`. Leave the runner's
  `pending` in place until the healer writes `deferred`, `launching`, or `declined`.
- **Storm breaker.** Trip once. The pass that trips sends one storm escalation for the
  episode. Every later target while paused declines quietly as `paused`, with the dim
  hint and no further escalation. Use the parent plan's copy:
  `Auto-restart paused: N agents broke within one update — this looks like a real bug, not an update race.`
- **Notification policy.** `_decline` resurfaces or escalates only per the "silenced vs
  not silenced" rule and decision `quiet_declines`.
- **Tests:**
  - A non-skew failed row (provider 429 text) and a dismissed bundle produce no ledger
    file, no `done.json` change, and no notification, through both `run -p` and
    `run_job_tick`.
  - A legacy row older than `max_defer_seconds` is ignored.
  - The disabled path resurfaces a silenced row once across three ticks and ignores
    non-silenced rows.
  - `write_recovery` on a missing directory creates nothing.
  - A handled doorbell is deleted, and the next tick is `idle`.
  - The storm sends exactly one escalation for 3 over-limit targets.

## Phase core-fixes

Work only in the linked sase-core checkout (`sase repo open sase-core`). Python callers
move in phase `healer-relaunch`, after the pin moves.

- **Workspace scoping** (`agent_auto_restart/catalog.rs`). Normalize `workspace_dir` and
  file paths before the prefix test: strip trailing slashes and compare whole path
  components. A workspace-origin ImportError must decline as workspace origin, with or
  without a trailing slash, even when W1, W2, and W4 hold. Assert the reason slug.
- **Managed origin.** Derive the origin module and path only from structured facts
  (`import_error`, `attribute_error`, `frames`) or from the exception line itself in
  `error_text` or `traceback_text`. Never search `log_tail` for a module name. Test:
  `ModuleNotFoundError: No module named 'requests'` with `sase.axe` logger lines in the
  tail declines.
- **Phase gate.** `unknown` must never yield `relaunch`.
  - Apply the parent plan's legacy evidence rule. Frames without `run_execution_loop`,
    `invoke_agent`, or finalizer frames, or a refresh log line with no provider start,
    mean `pre_provider`.
  - Otherwise return mode `ask` with reason `phase_unknown` and copy along the lines of
    "couldn't tell whether it reached its model turn".
- **Missing symbol.** Extract it from the matched error line, e.g.
  `cannot import name 'X' from 'M'` or `module 'M' has no attribute 'X'`, not from the
  first quoted token anywhere. Tier 1 with a failed probe and neither W1 nor W2 is
  `decline` (`no_update_witness`), not `defer`.
- **Episode fallback.** When there is no culprit commit, derive the episode from
  boot/current identity revisions before falling back to the refresh log line.
- **Ledger timestamps.** `claim_auto_restart_ledger` and `advance_auto_restart_ledger`
  take an `at` ISO-8601 string.
  - The bindings take it as an optional keyword, so released callers keep working.
  - Claim and `reclaim` set `claimed_at`, and every history entry gets `at`.
  - Test that both timestamps are set, and that `claimed → declined` is accepted (a
    legal transition that is currently untested).
- **Meta wire.** Add an optional `auto_restart` JSON object to the Rust agent-scan
  `AgentMetaWire`.
  - Populate it in the scanner's `agent_meta.json` extraction and carry it through the
    agent artifact index rows that the snapshot and running loaders read.
  - Python already reads `payload["auto_restart"]` in
    `src/sase/core/agent_scan_wire_conversion.py`.
  - Keep wire changes additive, per sase-core `AGENTS.md`.
- **Notifications.**
  - The mobile/tab classifier (`mobile.rs`, ~line 568) routes `agent.auto-restart` rows
    with error severity to the Errors tab, matching Python `priority.py`.
  - If `reconcile_notification_rows` cannot replace a row's `files`, add a narrow option
    so the episode refresh can set the union of every relaunched agent's evidence. Do
    not move timestamps or delivery cursors.
- **Golden tests.** Add a second real incident traceback shape (the waiter variant in
  the parent plan's incident). Add the new negative fixtures above.

## Phase probe-quiescence

Files: `src/sase/agent/auto_restart/{probe,quiescence}.py`, plus tests.

- `probe_modules_for_frames` / `_path_to_module` must yield importable names. For a
  src-layout editable root, `<root>/src/sase/axe/x.py` maps to `sase.axe.x`. Resolve
  each frame file against the package directories of the managed roots, not the checkout
  root. Test with an editable src-layout fixture and with a flat plugin layout.
- `quiescence.py` measures a managed root's last code change as the newest of its
  `HEAD`, current-branch ref, and `logs/HEAD` reflog mtimes, not the HEAD commit time.
  Test that pulling in an older commit still counts as a recent change.

## Phase runner-ux-fixes

- **Refresh path** (`src/sase/axe/run_agent_runner_refresh.py`).
  - Warm `sase.core.managed_tmp_roots` and `sase.agent.launch_timing` at module scope
    next to the existing leaf warm-ups.
  - Add a firewall test that runs in a subprocess (not under pytest's environment
    detection) and does not mock `planned_registered_name_belongs_to_artifact`.
- **Lifecycle and facts.**
  - `run_finalizers` in `src/sase/llm_provider/_invoke.py` marks `finalizing`.
  - `finalize_loop` (`src/sase/axe/run_agent_exec_finalize.py`) captures the phase
    before marking `finalizing`, so a handoff death keeps `handoff`.
  - Attach `failure_facts` only to failed outcomes, and also to the monitor/gate handoff
    finalizer failure branch.
  - Take `frames` and `last_frame_file` from the root exception of the chain in
    `src/sase/axe/runner_failure_facts.py`.
  - Make the boot identity prefer the first matching install record on `sys.path` in
    `runner_lifecycle_phase.py`, and fix that module's import docstring.
- **Config.** Move the `agent_auto_restart:` block in `src/sase/default_config.yml`
  after `agent_scope_teardown`'s `spare_process_patterns` list. Fixes
  `tests/test_config_schema.py::test_default_config_matches_public_schema`.
- **Epic-caused test failures:**
  - `just sync-completion-spec` for the two completion snapshot tests.
  - Expected mutex-group count 21 in `tests/completion/test_build.py`.
  - Add `auto-restart` to the sorted subcommand string in
    `tests/main/test_parser_command_help.py`.
  - Add `RESTARTING` to the pinned status tuple in `tests/test_query_profile_agents.py`.
  - Update
    `tests/test_core_agent_scan_wire_agent_meta.py::test_finalizer_status_is_trailing_wire_field`
    for the trailing `auto_restart` field (default `None`).
  - Use `sase.core.time` helpers instead of `datetime.now().astimezone()` in
    `src/sase/agent/auto_restart/ux.py`.
  - Call `code_swap_lock_path()` in
    `tests/sase_install/test_run_pypi_flow.py::test_lock_path_and_holder_match_sase`.
- **Import budget.** Defer the epic's three new TUI-closure modules.
  - Import `sase.agent.auto_restart.ux` lazily in
    `src/sase/ace/tui/models/_loaders/_done_filesystem_loaders.py`.
  - For `sase.axe.runner_lifecycle_phase` in `src/sase/llm_provider/_invoke.py`, make
    the import function-local only after confirming the runner imports that module at
    boot, so the runner path stays a `sys.modules` hit and never adds a post-provider
    first import.
  - The remaining non-epic growth belongs to `sase-1ic`. If the test is still red only
    because of it, record that in the phase note.
- **UX polish.**
  - Use `UPDATE_RECOVERY_GLYPH` instead of literal `↻` in:
    - `_agent_list_render_agent_prefix.py`
    - `_identity_header_compact.py`
    - the help sections
    - `plugins_browser_dev_update.py`
    - `auto_restart/ux.py`
  - Register the evidence bundle directory, not only `error_report.md`, as a `v` file
    hint.
  - Confirm that session-root mirroring copies the `recovery` fields, so a session
    member row also shows RESTARTING.
  - Re-run the two auto-restart visual goldens with targeted `just fix-tui-screenshots`
    and inspect them.

## Phase healer-relaunch

- **Pin.** Move `sase-core-revision.txt` past the core-fixes commit with
  `just ratchet-core-revision` (see `docs/rust_backend.md`). Rebuild this workspace's
  `sase_core_rs` from the linked checkout before testing.
- **Timestamps.** Pass `at` (ISO-8601, `sase.core.time` timezone) through
  `src/sase/core/agent_auto_restart_facade.py` and the ledger store. With real times,
  verify each of these:
  - `max_defer_seconds` expiry declines and escalates loudly with the "Couldn't restart
    … automatically" copy.
  - The 30-minute storm window counts.
  - `_adopt_or_settle` only adopts a replacement newer than the real claim time.
- **Re-classify.** When quiescence holds and the first verdict is `defer` with
  `probe_pending`, run the probe and attach `probe: {ok, failures}` to the witnesses.
  Classify again and relaunch only if the second verdict is `relaunch`; otherwise act on
  the second verdict. A failing quiescence check still defers before the probe runs.
- **Stale claim takeover.** When a pass takes over a stale `claimed` or `deferred`
  record, rewrite `python_claimer_pid` and `python_claimer_identity` to the current
  process before acting.
- **Replacement broke again.** Before the ledger lookup, a target whose
  `agent_meta.auto_restart` is set, or whose dir differs from the record's
  `failed_artifacts_dir` while the record is `launched` or settled, is the replacement.
  For it:
  - Settle the record `settled_failed` if it is still `launched`.
  - Write `recovery` declined on the replacement's `done.json` and clear its doorbell.
  - Escalate loudly as `already_restarted`, with the normal failure notification plus
    `This was its automatic restart after sase update <sha> — not retrying.`
- **Recovery states.**
  - Write `launching` just before `execute_agent_restart`.
  - Every `recovery` write carries `ledger_key` and, when known, `episode_id`.
- **Provenance.**
  - `_attach_provenance` fills `from_rev`, `to_rev`, `culprit_commit`, and
    `culprit_subject` from the assembled witnesses: W3 `file_proof`, W1 identity
    revisions, and the refresh log line. Fill `broke_detail` from the death phase and
    the refresh line.
  - `src/sase/axe/run_agent_runner_bootstrap.py` pops `SASE_AUTO_RESTART_PROVENANCE`
    from `os.environ` after consuming it. Verify the refresh re-exec keeps
    `agent_meta.auto_restart` through preserved metadata instead.
- **Ordering.** `run -p` orders targets topologically over `%wait` names (dependencies
  first), then least progress first.
- **Tests:**
  - An incident-shaped row runs through the **real** core classifier, with only
    quiescence, the probe subprocess, and plan/execute faked. It relaunches with forced
    reuse and real `from_rev`/`to_rev`.
  - The same row with a failing probe defers, then expires to a loud decline.
  - The replacement's second failure escalates `already_restarted` exactly once.
  - Stale-claim takeover.
  - An env-scrub test.

## Phase episode-polish

Files: mostly `src/sase/agent/auto_restart/notify.py` and
`sweep._settle_launched_records`.

- **Settlement.** Every settlement in `_settle_launched_records` calls
  `refresh_episode_report(episode_id)`. It also refreshes the episode row's `notes` and
  inline `action_data.report` snapshot through the reconcile path, without moving
  delivery cursors, so Telegram and mobile see **Now** settle. Remove
  `--epic-symbol 'sase-1j6(refresh_episode_report)'` from `_lint-symvision` in the
  `Justfile`, along with its comment block if one exists.
- **Now column.** Map every terminal replacement outcome: `stopped`, `noop`,
  `epic_launch_failed`, and any other non-running `done.json` outcome. A finished
  replacement never shows RUNNING.
- **Copy.**
  - Titles and escalations show the short sha (`9fd8a08`), not the raw episode id
    (`sase@9fd8a08`).
  - No path renders "unknown" as an update ref: fall back to "a sase update", and keep
    `episode_id` consistent between ledger records and the row.
  - The post-provider escalation names the held workspace number when known.
  - The storm escalation uses the core-fixes Errors-tab routing.
- **Dedup keys.** Escalation and resurface dedup keys include the failed row's artifacts
  timestamp, so reused names (bead-named agents) never collapse into an old or dismissed
  row as a silent +1.
- **Files.** The episode row carries every relaunched agent's `error_report.md` and
  evidence bundle, using the core-fixes reconcile option. Fall back to the live-report
  links only if core could not support it.
- Apply decision `quiet_declines`.
- **Tests:**
  - A settlement tick rewrites both the report file and the row snapshot with the cursor
    unchanged.
  - Every outcome maps in the Now column.
  - No title contains "unknown" or `sase@`.
  - A reused-name escalation creates a new row after the old one was dismissed.

## Phase acceptance

- **Incident replay.** Rewrite `tests/test_agent_auto_restart_incident_replay.py` so
  that every step after the fixture setup runs through real code:
  1. Fixtures: boot identity A, a `%wait` park, an update to B that removes a symbol,
     and a pre-provider death with real `failure_facts`.
  2. The real runner doorbell and silent notification.
  3. The real `run_job_tick` proc submission, with only `_submit_healer_proc` captured.
  4. A real `heal_one` with the real classifier. Fake only the quiescence result, the
     probe subprocess, and `plan_agent_restart`/`execute_agent_restart`.
  5. A relaunch under the same name with provenance.
  6. One episode row and one information toast, counted rather than only formatted.
  7. A real settlement tick that settles **Now** in the report file and in the row
     snapshot.
  8. The replacement's second failure, declined loudly as `already_restarted`, with the
     reason and the escalation asserted.

  No test may call `_escalate` or `refresh_episode_report` by hand.

- **Host dry run.** Run `sase agent auto-restart run -p -n -j` against the real host
  state.
  - Assert it is read-only: the ledger, `done.json` mtimes, and the notifications store
    are unchanged.
  - Assert that every listed target satisfies the candidate rule.
  - Run `sase agent auto-restart scan -s 120d`.
  - Record both results in the phase note.
- **Audits.** Review the new `done.json` and marker read/write sites in
  `src/sase/agent/auto_restart/` (at least `healer.py:_write_evidence` and
  `_runner_log_tail`, plus anything renamed by earlier phases). Add them to the marker
  mutation and path-passing audit allowlists.
- **Docs.** Update `docs/agent_auto_restart.md` with:
  - the candidate rule
  - silenced vs not-silenced notification behavior and the `quiet_declines` outcome
  - the storm breaker's Errors-tab routing
  - real-timestamp deferral expiry
- **Gate.** `sase tool run check` passes, except for failures attributed to `sase-1jj`,
  `sase-1ic` (non-epic import growth), `sase-1by`, and the known flakes. Name each one
  in the phase note.

## Verification

- Every phase runs `sase tool run check` in this repo. `core-fixes` also runs it in
  sase-core.
- No phase runs `just check-full` unless explicitly instructed.
- **Overall success criteria:**
  - The real classifier path relaunches the 2026-10-09 incident shape once, under the
    same name, with provenance that names `from_rev → to_rev`.
  - A non-skew failure and a dismissed row are never touched.
  - No `done.json` writer recreates a missing directory.
  - Disabling the feature resurfaces each silenced failure exactly once.
  - The live report and the row snapshot settle without a re-toast.
  - The replacement's second failure is escalated loudly as `already_restarted`.
  - `just symvision` passes without any `sase-1j6` epic-symbol rows.
