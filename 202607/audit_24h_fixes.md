---
tier: epic
title: 24h log-audit bug sweep
goal: 'The highest-frequency active failure loops found in the 2026-07-19/20 log audit
  stop recurring (hooks suffix oscillation, SDD clone wedge, axe restart churn, TUI
  error storms), verified regressions in the wait/runner-slot and retry flows are
  fixed, and log/test hygiene gaps that corrupted or bloated production state are
  closed.

  '
phases:
- id: hooks-strip-loop
  title: Converge hooks suffix-transform writes
  depends_on: []
  size: medium
  description: '''Converge hooks suffix-transform writes'' section: stop the strip/restore
    oscillation by merging hook-suffix rewrites on a fresh locked read and making
    strips idempotent.'
- id: retry-family-names
  title: Retry family-phase agents under the family base name
  depends_on: []
  size: medium
  description: '''Retry family-phase agents under the family base name'' section:
    normalize kill-and-edit/retry/mobile-retry seeding to the family base name and
    make rejected forced-reuse launches non-destructive.'
- id: runner-slot-waits
  title: Fix runner-slot wait regressions
  depends_on: []
  size: medium
  description: '''Fix runner-slot wait regressions'' section: make Run Now actually
    release parked slot waiters, preserve wait priority across question-yield requeue,
    and stop the slot poll from clobbering foreign waiting-marker keys.'
- id: axe-status-degrade
  title: Degrade TUI axe status gracefully
  depends_on: []
  size: small
  description: '''Degrade TUI axe status gracefully'' section: surface invalid axe
    config as a status instead of raising every auto-refresh tick, and dedupe repeated
    pump-task failure logs.'
- id: sdd-clone-selfheal
  title: Self-heal wedged SDD sidecar clones
  depends_on: []
  size: medium
  description: '''Self-heal wedged SDD sidecar clones'' section: recover machine-managed
    SDD clones from dirty-index pull failures with a snapshot-then-reset policy and
    rate-limited logging.'
- id: axe-restart-damper
  title: Journal and damp axe fleet restarts
  depends_on: []
  size: medium
  description: '''Journal and damp axe fleet restarts'' section: record every orchestrator
    restart with source and reason, clear dead-owner maintenance markers on the ensure
    path, and add restart-storm detection with notification.'
- id: bead-sync-conflicts
  title: Reduce bead stream sync conflicts
  depends_on: []
  size: medium
  description: '''Reduce bead stream sync conflicts'' section: union-merge append-only
    bead event streams during sync rebases, recount the event manifest after merges,
    and cover concurrent-writer conflict recovery.'
- id: fork-resolution
  title: Harden fork parent resolution
  depends_on: []
  size: medium
  description: '''Harden fork parent resolution'' section: accept fork parents that
    share one transcript, and turn incomplete-clan fork targets into a wait instead
    of a late workflow crash.'
- id: display-polish
  title: Close display and help-binding gaps
  depends_on: []
  size: small
  description: '''Close display and help-binding gaps'' section: humanize the remaining
    canonical project-key leaks, honor the configurable statistics help key for closing,
    and document the new binding.'
- id: log-hygiene
  title: Bound and harden log sinks
  depends_on: []
  size: medium
  description: '''Bound and harden log sinks'' section: rotate the unrotated JSONL/text
    sinks, make runs.jsonl appends atomic, cap hook output captures, and clean orphaned
    atomic-write temp files.'
- id: test-isolation
  title: Keep tests out of production state
  depends_on: []
  size: medium
  description: '''Keep tests out of production state'' section: guard telemetry and
    notification writers against unisolated pytest runs so test fixtures stop polluting
    the real metrics DB and axe logs.'
create_time: 2026-07-20 16:31:12
status: wip
---

# Plan: 24h log-audit bug sweep

## Context

A structured audit of all SASE log sources (`~/.sase/logs/*`, `~/.sase/axe/**`, `~/.sase/notifications`,
`~/.sase/telemetry`, run artifacts) plus a code review of the ~150 commits landed between 2026-07-19 16:00 and
2026-07-20 16:00 EDT surfaced a set of verified bugs and hygiene gaps. Every phase below is grounded in observed
production evidence (counts are for that 24h window) and, where stated, in a code-level root cause that was verified
against the current tree. Phases are independent; each lands as its own CL with its own tests. When a fix touches
shared/domain behavior, respect the Rust core boundary: wire/domain logic belongs in `../sase-core/crates/sase_core`
with Python calling through `sase_core_rs`; presentation and glue stay here.

Evidence pointers used throughout: `~/.sase/logs/tui.log`, `~/.sase/logs/tui_stalls.jsonl`,
`~/.sase/logs/tui_toasts.jsonl`, `~/.sase/logs/runs.jsonl`, `~/.sase/logs/events.jsonl`,
`~/.sase/axe/logs/lumberjack-*.log`, `~/.sase/axe/recent_errors.json`, `~/.sase/axe/error_digests/`,
`~/.sase/prompt_history/2607.json`.

## Converge hooks suffix-transform writes

**Evidence.** The hooks lumberjack logged ~22,700 `Stripped error marker from HOOK ... fix-hook Failed` lines in 24h,
all for the archived (Submitted) changespec `sase_fix_just_linters_14` in
`~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase-archive.sase`, still firing every ~3-6s tick. Successive reads of
that file show hook status-line suffixes oscillating between `(!: fix-hook Failed)` and `(fix-hook Failed)` — strips
persist and are then reverted.

**Root cause.** Two transforms in `src/sase/ace/scheduler/suffix_transforms.py` fight each other:

- `strip_error_markers` updates one status line at a time through `update_hook_status_line_suffix_type`
  (`src/sase/ace/hooks/persistence.py:167`), which re-reads the changespec from disk under `changespec_lock` before
  writing — safe.
- `strip_terminal_status_markers` builds a full replacement HOOKS list from the changespec object it was handed — the
  tick-serialized snapshot, which can be a tick stale — and writes it wholesale via `update_changespec_hooks_field`
  (`src/sase/ace/hooks/persistence.py:100` area). A stale snapshot that still contains `error` suffixes re-writes them
  over lines the fresh-read path just stripped. The loop never converges, and each tick logs "successful" strips.

**Fix.**

1. Make `strip_terminal_status_markers` (and any other caller of `update_changespec_hooks_field` that derives its hook
   list from a snapshot) recompute its transformation from the freshly parsed on-disk hooks inside the lock, mirroring
   `update_hook_status_line_suffix_type`. The cleanest shape: give persistence a
   `transform_hooks(project_file, changespec_name, fn)` helper that re-reads under lock, applies a pure
   `fn(list[HookEntry]) -> list[HookEntry] | None`, and skips the write when `fn` returns None/an equal list. Route both
   transforms through it.
2. Idempotence: when the fresh read shows nothing to change, emit no update message — this alone removes the per-tick
   log spam even in pathological states.
3. Apply the same fresh-read discipline to the COMMITS/COMMENTS branches of `strip_terminal_status_markers` if they
   share the snapshot-write pattern.

**Tests.** Regression test that simulates the interleave: parse a changespec, hold the stale object, apply a fresh-read
strip of one line, then run `strip_terminal_status_markers` with the stale object and assert the stripped line stays
stripped and that a second run produces zero updates (idempotence).

## Retry family-phase agents under the family base name

**Evidence.** Three launches failed on 2026-07-20 (toasts 19:27:22/19:27:37/19:28:54Z) with
`Agent name 'sase-8a.3--plan' cannot contain '--'`. `~/.sase/prompt_history/2607.json` holds the exact cancelled prompts
(`%id(3--plan, clan=sase-8a)` etc.). The failed planner's artifacts directory
(`.../artifacts/ace-run/202607/20/20260720134644/`) was destroyed even though the launch was rejected.

**Root cause (verified by live repro).** Kill-and-edit (`x`) on a failed auto-plan phase agent seeds the prompt bar from
the concrete family row name:

1. `_kill_and_edit_agent` (`src/sase/ace/tui/actions/agent_workflow/_entry_relaunch.py:73-106`) calls
   `prepare_kill_and_edit_prompt(raw_prompt, "sase-8a.3--plan")`
   (`src/sase/ace/tui/actions/agent_workflow/_entry_name_prompts.py`), which rewrites the clan member id to `3--plan`
   with forced reuse (`src/sase/xprompt/directive_edit.py:98-156`).
2. On submit, `run_agent_launch_body` first runs `wipe_names_for_forced_reuse(["sase-8a.3--plan"])`
   (`src/sase/ace/tui/actions/agent_workflow/_launch_body_impl.py:59-96`, `src/sase/agent/names/_wipe.py:68-79`) —
   destroying the old registry entry and artifacts — and only then validates names (`_launch_body_single.py:178`), where
   `validate_user_agent_name` (`src/sase/agent/launch_validation.py:106-113`) correctly rejects `--` (the family
   separator is reserved; only trusted launchers may pass it via `SASE_INTERNAL_AGENT_NAME_BYPASS`,
   `src/sase/agent/_family_attach_launch.py:44-46`).

`allocate_retry_name("sase-8a.3--plan")` returns `sase-8a.3--plan.r0`, so the plain retry path (`_entry_relaunch.py:57`)
and mobile retry (`src/sase/integrations/_mobile_agent_lifecycle.py:74-75`) carry the same latent bug.

**Fix.**

1. At every retry/relaunch seeding boundary — `prepare_kill_and_edit_prompt` / `force_name_reuse_in_prompt` callers in
   `_entry_name_prompts.py`, `_retry_edit_agent` in `_entry_relaunch.py`, and mobile retry in
   `_mobile_agent_lifecycle.py` — normalize the seed name with `agent_family_base` (`src/sase/plan_chain.py:327`): retry
   `sase-8a.3--plan` as `sase-8a.3`. The retained `%auto` re-derives the `--plan` row through the trusted family-attach
   path, which is the designed flow. Do NOT loosen validation and do NOT set the bypass for user-editable prompt-bar
   launches — that would let arbitrary prompts claim derived `--` names.
2. Reorder `run_agent_launch_body` so launch-name validation (at minimum the syntax check) runs before
   `wipe_names_for_forced_reuse`, making rejected relaunches non-destructive.

**Tests.** Kill-and-edit seeding for a `--plan` family row asserts the seeded `%id` uses the base member id; retry-name
allocation for a family row name; a launch-body test asserting a name-syntax rejection performs no wipe.

## Fix runner-slot wait regressions

Three verified defects from the wait-priority commits (`b04002091`, `46c2f0622`):

1. **Run Now on a parked runner-slot waiter is a silent no-op.** `b04002091` added `not result.run_now` to the
   special-case guards in `src/sase/ace/tui/actions/agents/_wait_actions.py:126-138`, so run-now falls through to the
   branch that writes `ready.json` (`{"unwait": true}`) — but the runner-slot loop (`wait_for_runner_slot` /
   `_try_claim_runner_slot`, `src/sase/axe/run_agent_wait.py:357-370`) never reads `ready.json`, and `unwait` has no
   reader anywhere in `src/`. The TUI optimistically clears `slot_requested_at`/`wait_runners`/`wait_priority` and
   toasts success; the agent stays parked, and the stray `ready.json` makes a second Run Now fail with `FileExistsError`
   (`src/sase/ace/tui/actions/agents/_directive_persistence.py:272-279`). Fix: route run-now for parked slot waiters
   through the live runner-wait update path (`_apply_live_runner_wait` semantics with `runners=None`, i.e. rewrite the
   marker to the implicit global-cap threshold) or an explicit slot-release marker the runner actually consumes; assert
   end-to-end that the runner proceeds. Also make the existing test
   (`tests/ace/tui/test_agent_wait_resume.py::test_apply_wait_run_now_releases_parked_runner_slot`) assert release, not
   just file writes.
2. **Question-yield requeue discards `%wait(priority=...)`.** `reacquire_runner_slot`
   (`src/sase/axe/run_agent_wait.py:262-272`) is called with hard-coded `wait_priority=None` from
   `src/sase/axe/run_agent_exec_questions.py:162-170`, so a priority-0 agent re-parks at default priority 10 (and a
   deliberately low-priority agent jumps ahead). `base_meta` at the call site already carries the persisted
   `wait_priority` (written at `src/sase/axe/run_agent_directives.py:444-445`) — thread it through. Add `wait_priority`
   to `AgentMetaWire` (`src/sase/core/agent_scan_wire_markers.py:95-171`) so scan consumers can read it (respect the
   Rust core boundary if the wire struct lives in `sase_core`).
3. **The slot poll clobbers foreign `waiting.json` keys.** The runner rebuilds the marker from a fixed seven-key set and
   rewrites on any inequality (`src/sase/axe/run_agent_wait.py:322-333`), dropping `wait_for_beads` that the TUI edit
   path deliberately preserves (`_wait_actions.py:284-293`, `_directive_persistence.py:242-268`). Preserve
   unknown/condition keys on rewrite; if bead metadata on slot markers is intentionally unsupported, instead remove the
   dead bead threading from `_apply_live_runner_wait` and its meta patch so `agent_meta.json` stops resurrecting
   resolved bead gates in `sase agent list` (`src/sase/integrations/_agent_list_entry_builder.py:237-239`).

While in this area, add a short docs note that runner-slot admission is priority-then-FIFO with no aging
(`src/sase/core/runner_slots/_admission.py:109-151`), so default-priority waiters can be starved by a stream of
higher-priority arrivals — an intentional trade-off that should be documented where `%wait` priorities are described.

## Degrade TUI axe status gracefully

**Evidence.** 342 identical ERROR tracebacks in `~/.sase/logs/tui.log` between 19:41 and 20:38 on 2026-07-19 — every
auto-refresh tick for ~57 minutes — all `AxeConfigError: Invalid axe configuration` raised because the then-running
(older) binary rejected newer config keys/guards (`axe.lumberjack_log_temp_max_age_seconds`, guard provider
`agent_clan`). The config itself was valid for the newer code; the storm is a version-skew window that will recur on
every schema-advancing deploy.

**Fix.**

1. In the axe status collection path used by TUI auto-refresh (`src/sase/ace/tui/actions/axe_display/_data.py:170` →
   `src/sase/axe/_process_status.py:45` → `src/sase/axe/config.py:118`), catch `AxeConfigError` and return a structured
   degraded status ("axe config invalid: <first diagnostic>") that the Axe pane renders, instead of letting the
   exception escape to the pump-task runner every tick.
2. In `src/sase/ace/tui/util/pump_tasks.py` (the `_done` callback that logs `pump-free task ... failed`), dedupe
   repeats: log the full traceback on first occurrence of a (task-name, exception-signature) pair, then rate-limit
   identical repeats to a one-line counter (e.g. once per 5 minutes). This bounds any future recurring pump-task
   failure, not just this one.

Keep the changes read-path only; per the TUI perf rules no new blocking work on the event loop.

**Tests.** Status collection with an invalid axe config returns the degraded status (no raise); pump-task failure dedup
logs once then suppresses.

## Self-heal wedged SDD sidecar clones

**Evidence.** 652 `Failed to pull workspace SDD clone` errors (comments lumberjack) + 228 (checks) in 24h, every ~60s
tick against the same machine-managed plans clone, plus 2
`SddRepositoryHealthError: git rebase failed: ... index contains uncommitted changes` and repeated
`cannot lock ref 'refs/remotes/origin/main'` contention. One dirty clone wedged both lumberjacks for most of a day; the
TUI also hit it (`sase.sdd._store_link` warnings).

**Fix.** In `src/sase/sdd/_store_link.py` (`ensure_sidecar_sdd_clone` / `_pull_sdd_clone`) and
`src/sase/sdd/_repository_transaction.py`:

1. When a pull-with-rebase fails because the clone has uncommitted/tracked changes, treat the clone as machine-managed
   and self-heal: snapshot the dirty state first (commit to a timestamped `sase/recovery/<ts>` branch or a stash entry,
   so nothing is silently lost), then hard-reset to the remote branch and re-pull. Bound the recovery to once per
   tick-window per clone to avoid a new hot loop; on repeated failure, surface one axe error-digest entry instead of
   per-tick spam.
2. Rate-limit the per-tick failure logging regardless (identical message once per N minutes per clone).
3. Verify the recovery path also clears the rebase-in-progress state (`rebase-merge`/`rebase-apply`) left by a crashed
   earlier rebase.

**Tests.** A fixture clone with dirty tracked changes + diverged remote: pull fails, self-heal snapshots then resets,
second pull succeeds, and the snapshot ref contains the dirty content; repeated failure does not re-attempt within the
same window.

## Journal and damp axe fleet restarts

**Evidence.** 43 clusters in ~22h where all 9 lumberjacks restarted together (45 starts each), with
`stale_running_cleanup` reaping up to 12 stranded running-claims right after restarts. All 45 maintenance-skip lines
cite one marker: `reason: Deploy retired fix_just chop package and scheduler configuration, pid: 1662906` — a pid that
is long dead. dev-update only restarted axe twice in the window, so most restarts are unattributed.
`~/.sase/axe/logs/axe.log` stopped receiving writes at 2026-07-19 22:40 despite 20+ later restarts, and startup hit
`Failed to start Prometheus HTTP server on port 9464: Address already in use` 3 times (overlapping daemons).
`ensure_axe` (`src/sase/axe/ensure.py:64`) only starts axe when the published PID is missing, so the orchestrator PID
was repeatedly dying or being unpublished.

**Fix.** This phase is part investigation, part hardening; land the hardening regardless of what the investigation
concludes:

1. **Restart journal.** Every orchestrator start/stop/restart appends a JSONL record (who: source argument, pid,
   desired-state, active maintenance marker if any) to a bounded journal under `~/.sase/axe/`. The existing restart
   verification (commit `10eeaf723`) covers restarts it performs; extend coverage so `sase axe start/stop`, `ensure_axe`
   healing, and dev-update restarts all journal — the point is that the next restart storm is attributable from one
   file.
2. **Stale maintenance clearing on the ensure/start path.** Lumberjack ticks already call `clear_stale_maintenance()`
   (`src/sase/axe/lumberjack.py:147`), but nothing clears a dead-owner marker while the fleet is down or before healing;
   call it from `ensure_axe` and orchestrator startup. Also guard against PID reuse: record the owner's process start
   time (or a boot id) in the marker and treat a recycled PID as dead.
3. **Restart-storm damper.** If the journal shows more than N orchestrator starts within M minutes (e.g. 5 in 30m),
   `ensure_axe` should back off (longer rate-limit) and emit a single notification naming the journaled sources, instead
   of silently thrashing; `stale_running_cleanup` churn right after restarts is the cost of each thrash.
4. **Prometheus bind conflict.** Startup should log the bind failure once with the owning pid if determinable and
   continue; it must not contribute to restart failure loops.

**Tests.** Journal records for start/stop/ensure-heal; dead-owner maintenance marker cleared by `ensure_axe`; storm
damper triggers after the threshold with one notification.

## Reduce bead stream sync conflicts

**Evidence.** In 24h: 55 `bead.sync.rebase` merge conflicts + 18 `bead.sync.push` remote rejections (non-fast-forward),
concentrated on `beads/events/streams/sase-*.jsonl` and `beads/issues.jsonl` (`~/.sase/logs/tui_git_ops.jsonl`); 18
`commit_conflict` + 18 `commit_failed(sync_conflict)` events; one epic rollback itself conflicted
(`Could not apply ... rollback work launch sase-84; non-bead conflicts remain: beads/events/streams/sase-84.jsonl, beads/issues.jsonl`),
and one epic launch was blocked by `bead event manifest stream_count mismatch: 300 != 301` — the conflict handling
corrupted the store's manifest invariant.

**Fix.** Scope this to the append-only stream files where a safe automatic merge exists:

1. Event streams (`beads/events/streams/*.jsonl`) are append-only journals: during bead-sync rebase conflict handling,
   resolve stream-file conflicts by unioning lines (preserve order by timestamp/ULID if entries carry one, else
   ours-then-missing-theirs, deduped exactly). Either install a `merge=union`-style gitattribute scoped to the streams
   directory in the sidecar repo during clone/init, or (preferred, more controlled) add explicit conflict resolution in
   the sync/rebase-continue path that rewrites conflicted stream files as the line-union and stages them.
2. After any merged/unioned sync, recompute the event manifest (`stream_count` and friends) from the actual streams
   before validation runs, so a correct union can never trip `stream_count mismatch`; treat a recount that disagrees
   with the manifest as a repairable condition with a logged repair, not a hard launch blocker.
3. `beads/issues.jsonl` is not union-safe (rewrites); leave its conflicts to the existing retry/rebase flow, but make
   the rollback path robust: a rollback commit that conflicts should fall back to store-level reconstruction rather than
   leaving the repo mid-rebase.
4. Mind the Rust core boundary: if manifest/stream invariants are enforced in `sase_core`, implement the recount/repair
   there and expose it through the binding.

**Tests.** Two writers append to the same stream on diverged clones; sync unions both sets of events, manifest recount
passes, no conflict escapes to the operator. Rollback-conflict fallback covered with a conflicting rollback commit.

## Harden fork parent resolution

**Evidence.** 8 failed runs in 24h aborted in the workflow `resolve` step (`src/sase/scripts/agent_chat_from_name.py`
`_resolve_agent_chat_sources`, RuntimeError "Invalid fork parents"), two modes:

- `parents 'fi--plan' and 'fi--code' resolve to the same transcript` — a planner and coder phase of one family
  legitimately share a chat file when the coder resumed the planner's session; rejecting the pair as duplicates is
  wrong.
- `#fork:<clan>`/`#fork:<agent>` dispatched while the parent was incomplete
  (`Clan 'sase-7z' is not complete: 3/9 members done` / `No agent with chat history found for: sase-7z`) — the agent
  claims a workspace, runs the resolve step, and dies late with an opaque error.

**Fix.**

1. In `_resolve_agent_chat_sources`, when two resolved parents map to the same transcript path, dedupe silently (keep
   the first) instead of raising; only raise when the _user-specified_ parents are textually identical.
2. Make fork targets participate in wait gating: when a `#fork` parent is a clan (or agent) that exists but is not yet
   complete, the launch should park on the same wait machinery used by `%w`/bead gates (or, minimally, fail fast at
   directive-resolution time _before_ claiming a workspace, with a message that names the incomplete members).
   Investigate where `#fork` resolution happens relative to wait admission in `src/sase/axe/run_agent_runner.py` / the
   xprompt VCS workflow steps and pick the earliest gate that has the clan state available.

**Tests.** Same-transcript parent pair resolves to one source; forking an incomplete clan parks or fails pre-workspace
with the member-status message; forking a complete clan still works.

## Close display and help-binding gaps

Four small verified gaps from the sase-89/8a review:

1. Mobile agents API subtitle joins the raw canonical project key
   (`src/sase/integrations/_mobile_agent_summary.py:227-229`) — humanize with `project_display_name_for` like
   `src/sase/integrations/_mobile_helper_catalog.py:53` does, keeping raw keys only in machine-readable context fields.
2. XPrompt browser group headers render canonical keys (`src/sase/ace/tui/modals/xprompt_browser_helpers.py:95,122,141`)
   — humanize the display label while leaving source labels/namespacing untouched.
3. `StatisticsHelpModal` hard-codes its close keys and footer hint
   (`src/sase/ace/tui/modals/statistics_help_modal.py:38-48,76`) while the open key is configurable
   (`ace.keymaps.statistics.help`): bind the configured key as a close key too and render it in the footer via the
   keymap display name.
4. The new statistics `help` binding is missing from the docs enumerations in `docs/configuration.md` (sample + actions
   table), `docs/telemetry.md` (key table + override YAML), and `docs/ace.md` (override example).

**Tests.** Mobile summary/browser header humanization asserts; a modal test with a remapped help key asserts open and
close symmetry.

## Bound and harden log sinks

**Evidence.** Rotation is inconsistent: `tui_stalls`/`tui_git_ops`/`tui_launch_timing` rotate, while
`~/.sase/logs/dev_update.jsonl` (4.3MB, unbounded), `tui.log` (1.6MB since 07-06), `runs.jsonl` (1.3MB), `events.jsonl`,
`tui_agent_loads.jsonl`, `tui_toasts.jsonl` grow forever. `runs.jsonl` contains an interleaved corrupt record (one
record's JSON written inside another's `status` field — non-atomic concurrent appends). One hook-output capture reached
17.9MB / 241k lines (`~/.sase/hooks/202607/...-260719_202105.txt`) with no truncation. Orphaned atomic-write temp files
persist (`~/.sase/notifications/.notifications.jsonl.<pid>.<ts>.tmp` from 07-18,
`~/.sase/pending_actions/.actions.json.*.tmp` from 07-17).

**Fix.**

1. Route the unrotated sinks through the same size-based rotation used by the rotating JSONL sinks (single `.1`
   generation is fine); include `tui.log`.
2. Make `runs.jsonl` (and any other multi-process JSONL append sink) write each record with a single `O_APPEND`
   `os.write` of a fully serialized line (or under the existing lock helper) so concurrent writers cannot interleave;
   keep readers tolerant of a torn historical line.
3. Cap hook output captures: keep head + tail (e.g. first 200KB + last 300KB with an elision marker), preserving the
   tail where pytest/lint summaries live.
4. Best-effort GC for stale atomic-write `.tmp` siblings older than a day in the writers' directories, done in the
   writer helper itself.

**Tests.** Rotation trigger for a previously unrotated sink; concurrent-append stress writes N records from two
processes with zero torn lines; capture truncation preserves head/tail and marker; tmp GC leaves fresh temps alone.

## Keep tests out of production state

**Evidence.** `~/.sase/telemetry/metrics.sqlite` is 87MB with 113,425 samples labeled
`test-provider`/`workflow=test-workflow` and a frozen fixture instance timestamp (`20260316_120000`), written up to the
present; in-window "LLM error" counters are dominated by `fakey`/`test-provider` noise. The production
`~/.sase/axe/logs/axe.log` contains crash-notify lines whose paths point into
`/tmp/pytest-of-bryan/.../home*/.sase/notifications` — pytest-spawned lumberjacks wrote through to the real axe log and
notification senders.

**Fix.**

1. Add a hard guard at the telemetry write boundary (`src/sase/telemetry/`): when running under pytest
   (`PYTEST_CURRENT_TEST`/equivalent) and the resolved SASE home is the real user home (not a tmp fixture home), refuse
   the write (raise in tests, no-op with a single warning otherwise). The correct fixtures already isolate SASE home —
   the guard catches the leaking ones, then fix the specific fixtures the guard flags (the `agent_runner` fixture with
   the frozen `20260316_120000` instance is the known offender).
2. Apply the same guard shape to the axe crash-loop notify path and any writer that resolves state paths at call time
   (`src/sase/axe/` commit `cc99b7a3b` made paths call-time-resolved — the guard complements that by refusing
   cross-boundary writes).
3. One-time cleanup: provide a small maintenance command or documented `sase doctor`-adjacent action that deletes
   `test-workflow`/`test-provider`/`fakey` samples from the live metrics DB and vacuums it (87MB → real data only). Do
   not run it implicitly.

**Tests.** Under a simulated unisolated pytest env the telemetry writer refuses and flags; isolated fixture homes still
record; the cleanup action removes only test-labeled samples.

## Testing strategy

Each phase adds targeted regression tests alongside the existing suites (`just test`; visual snapshots only where a TUI
surface changes, e.g. the statistics help modal). Phases touching axe daemon behavior should prefer the existing
fakey/e2e harnesses (`tests/fakey/`, axe outage-recovery smoke tests) over new bespoke scaffolding. Run `just check` per
phase before completion.

## Risks

- **SDD clone self-heal and bead union-merge modify recovery behavior on real repositories.** Both phases must snapshot
  state before destructive steps (recovery branch/stash; rebase abort before reconstruction) and be covered by
  fixture-repo tests before landing.
- **Wait/runner-slot changes affect admission ordering.** Keep the normalization helpers (`normalize_wait_priority`) as
  the single source of truth and extend the existing e2e queue tests (`tests/fakey/test_runner_slots_e2e.py`) rather
  than adding parallel logic.
- **Restart-damper thresholds** must not delay legitimate healing after a real crash; default conservatively (damp only
  on repeated starts with no intervening stable uptime) and notify when damping engages.
- **Version-skew storms** (the axe-config error burst) recur by design around deploys; the degraded status must render
  clearly so a skewed binary is visible in the TUI rather than silent.

## Deferred follow-ups (documented, not in this epic)

- Codex usage-limit circuit breaker (4 long runs burned ~1.7h after the provider quota was exhausted; needs
  provider-level design: parse reset time, defer codex launches until then).
- Telegram `/kill` callback-data 64-byte overflow — real bug, but lives in the `sase-telegram` plugin repo
  (`sase_tg_inbound.py:2701`).
- Telegram bot-token `pass` failure storm: add chop-env preflight backoff for repeatedly unreadable secrets (64
  errors/hour while the secret store was locked).
- `%wait(bead=...)` existence validation at parse/launch time (a typo currently parks the agent forever with no
  diagnostic).
- Stranded wait dependency `telegram-recovery-q-20260706` (state cleanup + surfacing never-satisfiable waits in the
  TUI).
- TUI stall hot paths from `tui_stalls.jsonl` (artifact-path watcher scans, synchronous `lazy_syntax` highlighting,
  `select_prompt_file`, ~90s full agent disk loads at ~700 agents) — a dedicated perf epic guided by
  `sase/memory/tui_perf.md` rather than a phase here.
- Stale `%tribe:chop` emitter: one auto-agent prompt still used the removed `%tribe` directive; the template appears to
  live outside this repo (overlay config) — locate and migrate it.
