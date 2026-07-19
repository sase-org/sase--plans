---
tier: epic
title: Fix chop lifecycle poisoning, collision failures, kill-callback crashes, and
  config deadlock
goal: 'Chop runs finalize from the agents the runner actually launched (no more ambient-env
  registry pollution falsely failing runs and re-firing triggers hourly), explicitly-named
  proposals skip gracefully instead of failing the run when their agent name is taken,
  the Telegram /kill command survives long clan agent names instead of crashing every
  tg_inbound cycle, the orchestrator pid file survives concurrent restart actors,
  and the dead fix_just chop is revived by fixing its chezmoi once_per config.

  '
phases:
- id: linkage-scoping
  title: Explicit chop-launch linkage scoping
  depends_on: []
  description: '''Explicit chop-launch linkage scoping'' section: register chop-agent
    linkage only for explicit runner launches and continuation respawns, scrub ambient
    SASE_CHOP_* from unrelated child agents, and isolate the leaking launcher tests
    from real axe state.'
- id: finalize-matching
  title: Launch-matched lifecycle finalization and registry GC
  depends_on:
  - linkage-scoping
  description: '''Launch-matched lifecycle finalization and registry GC'' section:
    finalize launched runs from records matched to the entry''s launches (following
    retry chains), ignore unmatched records, and garbage-collect orphaned registry
    records.'
- id: collision-skip
  title: Graceful per-proposal skip on agent-name collision
  depends_on: []
  description: '''Graceful per-proposal skip on agent-name collision'' section: treat
    a taken explicit agent name as an idempotent per-proposal skip with a recorded
    reason instead of failing the whole run, releasing once-per keys for skipped proposals.'
- id: kill-callback
  title: Telegram kill-selection callback data hardening
  depends_on: []
  description: '''Telegram kill-selection callback data hardening'' section: replace
    raw agent names in /kill inline-keyboard callback data with short persisted keys
    so long clan member names cannot exceed Telegram''s 64-byte limit, and keep one
    bad button from crashing the whole tg_inbound cycle.'
- id: pidfile-hardening
  title: Orchestrator pid-file lifecycle hardening
  depends_on: []
  description: '''Orchestrator pid-file lifecycle hardening'' section: write orchestrator.pid
    atomically and make stop-path cleanup delete it only when its contents still name
    the process that was stopped, so concurrent restart/ensure actors cannot remove
    a live orchestrator''s pid file.'
- id: chezmoi-config
  title: Chezmoi chop config repair
  depends_on: []
  description: '''Chezmoi chop config repair'' section: remove the once_per key that
    permanently dead-locks fix_just and add agent-hood guards to the audit chops in
    the chezmoi axe config, verifying with doctor and dry runs.'
create_time: 2026-07-19 19:46:59
status: wip
bead_id: sase-7t
---

# Plan: Fix chop lifecycle poisoning, collision failures, kill-callback crashes, and config deadlock

## Context

Epic sase-6v redesigned axe chops into script-only jobs that emit structured results; the runner launches proposed
agents, records durable linkage in each lumberjack's `agent_chops.json`, and a housekeeping pass
(`finalize_launched_chop_runs` in `src/sase/axe/chop_lifecycle.py`) later finalizes `launched` runs from
agent-completion artifacts. A first review of the live lumberjack state on athena (2026-07-19 morning) produced an
earlier version of this plan; a second review the same evening confirmed which findings later epics already fixed and
which are still actively degrading the system.

**Already fixed — do not redo.** Bounded-log hysteresis, rotation-temp reaping, and lumberjack crash recovery landed via
sase-7p (`append_bounded_log` now truncates to half the cap and `reap_stale_log_rotation_temps` runs at orchestrator
start); the multi-GB `.tmp` orphan leak is gone and the disk has recovered. Once-per keys are released for failed
launches (sase-7i.3), waits are relinked across deduped proposals (sase-7i.2), and dismissed bundles are consulted
before fail-closing completions (sase-7i.4). The 07:35 minute-cadence `chop.refresh_docs.<proj>.1` collision storm was
fixed by run-token naming. The morning's lumberjack restart flood and `[Errno 28] No space left on device` errors were
fallout of the since-fixed temp leak.

**Still broken — confirmed live this evening:**

1. **Registry pollution via ambient env.** `spawn_agent_subprocess` (`src/sase/agent/launch_spawn.py`) builds
   `subprocess_env = dict(os.environ)`, scrubs `SASE_AGENT_*` but not `SASE_CHOP_*`, and then calls
   `record_chop_agent_launch_from_env(env=subprocess_env)`. Any agent spawned by a process that carries chop context — a
   chop-launched agent's nested launches, or a chop _script_ that launches agents itself (the runner injects
   `build_chop_launch_env(...)` into every script's env at `src/sase/axe/chop_runner_script.py:222`) — registers under
   the parent's `(chop_name, run_id)`. Two live consequences: (a) tests in `tests/test_axe_chop_agents.py` that drive
   the real launcher via the shared `_spawn_agent_for_env_test` helper without patching `sase.axe.state.JACK_STATE_DIR`
   append fixture records (pid 4321, project `proj`, cl `feature/test`, timestamp `20260101120000`) to the real
   registries whenever a chop-launched agent runs `just check` — the 19:04 `recent_bug_audit[sase]` run and today's
   16:19 `toobig_split[sase]` clan run were both poisoned this way; (b) agents the `tg_inbound` script launches for chat
   commands register under tg_inbound run ids, and because those runs finish `success` (not `launched`) the records are
   never removed — the telegram registry has accumulated 35 stale records and grows daily.
2. **Housekeeping trusts every record.** `finalize_launched_chop_runs` evaluates _all_ registry records matching
   `(chop_name, run_id)`; one dead unrelated pid without artifacts forces the whole run to `action_failed`. Every
   `refresh_docs[sase]` and `recent_bug_audit[sase]` run today was falsely failed by the pid-4321 fixture records.
   Because those chops use `checkpoint: on_action_success`, the bug-audit checkpoint file has an empty `entries` map —
   the `git.commits_since` cursor never establishes, so the trigger re-fires every `run_every` cycle (hourly), launching
   a fresh full audit agent each time. (`recent_improvement_audit` had one clean run, so its cursor exists and it
   correctly skips at "143 of 200 commits".) Registry records are also only removed when their run finalizes, so records
   for terminal or vanished runs leak forever.
3. **Agent-name collisions fail the whole run.** Runner-derived proposal names embed a per-run token, but a
   script-supplied `agent_name` is used verbatim. The bugyi-chops audit scripts intentionally name agents
   `audit_bugs.<project>.<HEAD-short>` as an idempotency key; re-firing at an unchanged HEAD raises
   `AgentNameLaunchCollisionError` (`src/sase/agent/launch_validation.py` — the typed exception now exists), and
   `process_script_chop_result` marks the run `action_failed` (observed at 14:56 today:
   `Agent name 'audit_bugs.sase.7ef34829ef0a' is taken`). The checkpoint again never advances.
4. **Telegram /kill crashes the whole inbound cycle.** `_show_kill_selection`
   (`src/sase_telegram/scripts/sase_tg_inbound.py` in the sase-telegram repo) builds inline-keyboard buttons with
   `callback_data=encode("kill", name, "go")`, and `encode` (`src/sase_telegram/callback_data.py`) raises `ValueError`
   past Telegram's 64-byte limit. Chop-launched clan members now have names like
   `split_file.tests.ace.tui.test_plugins_browser_pane_install.6f7e8258-2` (~70 bytes), so `/kill` crashed the
   tg*inbound script three times between 17:47 and 17:52 today — each crash aborts the _entire* inbound poll cycle
   (exit code 1), not just the kill flow.
5. **Orchestrator pid file lost across concurrent restarts.** After this evening's 19:36 daemon restart the new
   orchestrator ran for minutes with no `~/.sase/axe/orchestrator.pid` at all. `_write_pid`
   (`src/sase/axe/orchestrator.py`) uses a plain truncating `write_text`, and the stop path's `cleanup_pid_files()`
   (`src/sase/axe/_process_probe.py`) unlinks the file unconditionally once an earlier probe considered the daemon dead
   — so interleaved stop/start/ensure actors (`sase axe ensure` timer, restart flows from
   `src/sase/axe/_process_restart.py`) can delete the file a freshly started orchestrator just wrote. Probes fall back
   to the lock-holder pid, so impact is degraded observability rather than outage, but the lifecycle should be
   watertight.
6. **fix_just is permanently dead by config.** The chezmoi `axe:` overlay gives the `fix_just` chop
   `once_per: "{proposal.id}"`. The script's single proposal id is the static string `fix`, so the runner-owned seen
   store blocks every run forever — run history shows nothing but
   `skipped · all 1 proposal(s) skipped by once-per dedupe` through 19:08 today. The `inhibit_if: changespec` guard in
   the same stanza already provides the intended behavior, so the `once_per` line must go.

Non-bugs confirmed, so phases do not chase them: `%wait` proposals launching with `workspace_num 0` and a placeholder
workspace dir is the deferred-workspace design; the 16:19 `toobig_split[sase]` run's 20 sequential wait-chained agents
are working as designed (only the injected fixture record threatens it); telegram tick-overrun warnings are load
reporting, not failures.

**Interplay with in-flight epic sase-7q.** Clan-scoped chop launches (sase-7q.2) landed today and sase-7q.3/7q.4 are
rewriting toobig_split's member identity and its configured guard. This plan must not touch the `toobig_split` chezmoi
stanza, clan member-ID semantics, or the clan batch preflight; the collision-skip phase applies only to the sequential
(non-clan) launch path.

Per the Rust core boundary rule: everything below is launcher/subprocess/IO, state-file housekeeping, plugin script
logic, and config — host-side concerns in this repo, the sase-telegram repo, and the chezmoi repo. No `../sase-core`
changes are required.

## Design overview

The linkage design principle shifts from _"any descendant launch inherits the chop identity"_ to _"linkage is
explicit"_: a registry record is created only for (a) proposal launches the runner itself performs — which already pass
`build_chop_launch_env(...)` through `extra_env` (`launch_chop_proposals` in `src/sase/axe/chop_proposals.py`) — and (b)
continuation respawns of an already-linked agent (retry spawn, model-fallback spawn), which must keep re-registering the
successor pid so housekeeping tracks the live process. Everything else — nested launches by chop agents, launches
performed by chop scripts themselves (tg*inbound chat commands), test-suite launches — neither registers nor inherits
`SASE_CHOP*\*`. Chop scripts keep receiving their own env (`SASE_CHOP_RESULT_FILE`, identity vars, verbose flag)
unchanged; the boundary moves to the agent launcher.

Housekeeping symmetrically stops trusting the registry blindly: records are matched to the run entry's own `launches`
(both sides already carry `artifacts_timestamp`), retry successors are followed through the `retried_as_timestamp` chain
that `_agent_completion` already understands, unmatched records are ignored for status purposes, and orphaned records
are garbage-collected. This heals the currently-polluted registries — pid-4321 fixture rows, pre-cutover legacy rows
(`sase_refresh_docs`, `sase_recent_*_audit`), and the 35 accumulated `tg_inbound` rows — without manual surgery, and
lets the bug-audit checkpoint finally establish so the hourly re-fire loop stops.

The Telegram fix moves /kill button identity out of the callback payload: buttons carry a short generated key, and the
key→agent-name mapping is persisted server-side (the existing `pending_actions` store fits) for resolution when the
callback arrives. Sending the selection also stops assuming every name fits; per-button failures degrade to skipping
that agent with a logged warning rather than aborting the inbound cycle.

## Phases

### Explicit chop-launch linkage scoping

In `src/sase/agent/launch_spawn.py`, `src/sase/agent/env_hygiene.py` usage, and `src/sase/axe/chop_agents.py`:

- Registration: only call the registry when chop metadata was provided explicitly by the caller via `extra_env`, or when
  the spawn is a continuation of the current (already chop-linked) agent — `retry_transfer_from_pid` is set on the retry
  path (`src/sase/axe/run_agent_retry_spawn.py`), and the codex-fallback respawn path
  (`tests/llm_provider/test_codex_fallback_spawn_workspace.py` shows it flows through the same seam) must be audited for
  its equivalent marker. For continuations, reading the current process environment remains correct because the
  respawning process _is_ the linked agent. Restructure `record_chop_agent_launch_from_env` (or add a scoped wrapper) so
  the "which env is authoritative" decision is made by the launcher, not by scanning the merged child env.
- Env propagation: scrub ambient `SASE_CHOP_LUMBERJACK/NAME/RUN_ID/PROMPT_HASH` from spawned children's environments
  (`scrub_chop_context_env` in `src/sase/agent/env_hygiene.py` already exists — apply it beside the existing
  `SASE_AGENT_*` scrub in `spawn_agent_subprocess`), while values supplied via `extra_env` (real proposal launches) and
  continuation respawns still land in the child env so `agent_meta_from_chop_env` keeps stamping `agent_meta.json` for
  genuinely chop-launched agents. Update `test_spawn_agent_subprocess_replaces_ambient_agent_identity_with_launch_env`,
  which currently asserts the ambient pass-through.
- Test hygiene, closing the pollution vector regardless of where the suite runs: make every test in
  `tests/test_axe_chop_agents.py` that drives the real launcher isolate `SASE_HOME` and `sase.axe.state.JACK_STATE_DIR`
  (two tests in the file already show the pattern — extend it to the shared `_spawn_agent_for_env_test` helper), and add
  an autouse conftest fixture that deletes the four `SASE_CHOP_*` vars from the test environment so no test can observe
  or forward a hosting chop agent's identity.

Tests: launcher unit tests covering all three registration cases (explicit extra_env → recorded; ambient-only → not
recorded, env scrubbed; retry continuation → recorded), plus the migrated env-sanitization assertions.

### Launch-matched lifecycle finalization and registry GC

In `src/sase/axe/chop_lifecycle.py` and `src/sase/axe/chop_agents.py`:

- Match records to launches: for each `launched` run, resolve the record set against `entry.launches` by
  `artifacts_timestamp` (helper `_launch_for_record` already exists), then extend the matched set transitively with
  retry successors — a matched record whose `done.json` carries `retried_as_timestamp` claims the record bearing that
  artifacts timestamp. Only the matched set participates in completion evaluation; unmatched records are logged into the
  run output and ignored. Keep the fail-closed rule per launch: a launch with no matching record (and no retry
  successor) still produces the "linkage incomplete" failure detail, and `expected`-count semantics degrade to
  `action_failed`, never hang.
- Registry GC: during the housekeeping pass, delete records whose `(chop_name, run_id)` no longer resolves to a run
  entry, or whose run entry is already terminal. This automatically clears the existing pid-4321 fixture records, the
  pre-cutover legacy records, and the accumulated `tg_inbound` rows, so no manual state surgery is needed. Remove the
  dead no-op conditional in `_registry_path` while in the file if it still exists.

Tests: finalization with extra unmatched records (run still succeeds), retry-chain following, missing-record fail-closed
behavior, and GC of orphaned/terminal-run records (including success-status runs like tg_inbound's).

### Graceful per-proposal skip on agent-name collision

In the launch path (`src/sase/axe/chop_proposals.py`, `src/sase/axe/chop_runner_script_result.py`), reusing the existing
typed `AgentNameLaunchCollisionError` from `src/sase/agent/launch_validation.py`:

- In the sequential (non-clan) loop of `launch_chop_proposals`, catch the typed error per proposal _only when the
  proposal supplied an explicit `agent_name`_ (runner-derived names embed the run token and must never collide — a
  collision there stays a hard failure, as does any collision in the clan batch path, which sase-7q owns). Record the
  proposal as skipped with a name-collision reason (mirroring the once-per duplicate shape in previews/run history),
  release its once-per key like other unlaunched proposals, relink dependent `wait_on` proposals the same way once-per
  dedupe does, and continue with the remaining proposals.
- Aggregate outcomes in `process_script_chop_result`: all proposals skipped → run status `skipped` with the reason
  visible in history; some launched → `launched` as today; non-collision launch errors keep current
  `action_failed`/partial semantics.

This turns the bugyi-chops audits' revision-keyed names into intentional idempotency: re-firing at an unchanged HEAD
becomes a quiet, explained skip instead of an hourly `action_failed` loop.

Tests: collision on explicit names (single and multi-proposal with wait relinking), collision on derived names still
failing, once-per key release on skip, and status aggregation.

### Telegram kill-selection callback data hardening

In the sase-telegram repo (opened through the repo-access skill), primarily
`src/sase_telegram/scripts/sase_tg_inbound.py`, `src/sase_telegram/callback_data.py`, and
`src/sase_telegram/pending_actions.py`:

- Replace the raw agent name in `/kill` selection callback data with a short generated key (bounded well under the
  64-byte budget after the `kill:` prefix), persisting key→agent-name in the pending-actions store when the selection
  keyboard is sent and resolving it in the callback handler. Expire or overwrite mappings using the store's existing
  staleness semantics so abandoned selections do not accumulate.
- Harden the selection sender: a name that still cannot produce valid callback data (or any per-button encoding error)
  skips that agent with a logged warning instead of raising out of `_handle_kill_command` and failing the whole
  tg_inbound run. Keep `/kill <name>` (explicit-name path) behavior unchanged.
- Follow the sase-telegram repo's own build/check instructions (`just check`) for validation.

Tests: encoding round-trip for the new key scheme, a kill-selection flow with an over-64-byte agent name (keyboard sent,
callback resolves to the long name), and a regression test that a single unencodable entry does not abort the selection.

### Orchestrator pid-file lifecycle hardening

In `src/sase/axe/orchestrator.py` and `src/sase/axe/_process_probe.py`:

- Write `orchestrator.pid` atomically (temp file + `os.replace` beside the target) so concurrent readers never observe a
  truncated or empty file, which today can defeat the pid-guard in `_remove_pid` and probe reads.
- Make the stop path's pid-file cleanup ownership-checked: `cleanup_pid_files()` (and its callers in
  `src/sase/axe/_process_stop.py`) should delete `orchestrator.pid` only when the file's current contents name the pid
  that was just confirmed stopped (or a dead pid), never unconditionally — mirroring the existing guard in
  `Orchestrator._remove_pid`. This prevents an interleaved ensure/restart actor from deleting the file a freshly started
  orchestrator just wrote (observed live after the 19:36 restart: running orchestrator, missing pid file).
- Keep the lock-holder-pid fallback behavior in `_process_probe` unchanged; it remains the safety net.

Tests: atomic-write behavior (file always parses as an int), ownership-checked cleanup (live different-pid file is
preserved; dead-pid and matching-pid files are removed), and a restart-interleaving regression exercising
stop-after-new-start leaving the new pid file intact.

### Chezmoi chop config repair

Config-only work in the chezmoi repo (opened through the repo-access skill; touch only
`home/dot_config/sase/sase_athena.yml`, no memory or agent-instruction files):

- Delete `once_per: "{proposal.id}"` from the `fix_just` chop stanza. The `inhibit_if: changespec` guard already
  provides the intended "one fixer at a time" behavior, and the proposal's `-@` agent name cannot collide. No seen-store
  surgery is needed: with no `once_per` and no script-supplied `dedupe_key`, the stored `fix` key is inert.
- Add `inhibit_if: agent_hood` guards to `recent_bug_audit` (hood `audit_bugs`) and `recent_improvement_audit` (hood
  `audit_improvements`) so a still-running audit suppresses the next trigger fire at the source instead of relying on
  downstream name-collision skips. Do NOT touch the `toobig_split` stanza — its guard is being reworked by epic sase-7q.
- Verify on the live deployment: the axe config loads (fail-closed validation passes), `sase axe chop doctor` is clean,
  and `sase axe chop run fix_just -n -V` shows the proposal accepted (not once-per skipped) in the dry-run proposals
  table. Follow the chezmoi repo's own instructions for applying committed changes.

## Testing strategy

Each sase-repo phase lands with the unit coverage listed in its section and must pass `just check`. The linkage-scoping
phase is first because finalize-matching builds on the explicit-linkage semantics; the remaining phases are independent.
The kill-callback phase runs `just check` in the sase-telegram repo. The chezmoi phase validates against the
currently-deployed sase (its edits do not depend on the code fixes). End-to-end healing of the live registries (pid-4321
records, stale telegram rows) arrives with the registry GC once the release containing these fixes is deployed; no phase
should hand-edit live state under `~/.sase/`.

## Risks and mitigations

- **Continuation respawns losing linkage.** The retry path registers via ambient env today; the linkage-scoping phase
  must keep that working (covered by an explicit retry-continuation test) and audit the codex-fallback spawn path, which
  currently patches `record_chop_agent_launch_from_env` in its tests — a sign it flows through the same seam.
- **Finalize matching regressing the retry special-case.** `_agent_completion` already treats
  `failed + retried_as_timestamp` as success for the original attempt; chain-following must keep evaluating the
  successor so a run is not finalized while the retry is still running.
- **Name-collision skip masking real bugs.** Scoping the graceful skip to explicitly-named proposals on the sequential
  path keeps runner-derived-name and clan-batch collisions loud, and the skip reason is persisted in run history and
  previews so `sase axe chop list -v` explains every quiet cycle.
- **Registry GC deleting records housekeeping still needs.** GC only removes records whose run entry is terminal or
  missing; `launched` runs keep their full record set, and the launch-matched evaluation change is what protects them
  from strays in the meantime.
- **Kill-callback key collisions or staleness.** Short keys are generated per selection message and resolved through the
  persisted mapping; the store's existing stale-entry cleanup bounds growth, and an unresolvable key answers the
  callback with a clear "selection expired" message instead of guessing.
- **Stepping on sase-7q.** No changes to clan batch launch semantics, member-ID derivation, or the toobig_split config
  stanza; if the collision-skip phase's touched files conflict with 7q.3 landing, the phase rebases on the landed state
  before finishing.
