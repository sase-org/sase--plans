---
tier: epic
title: Fix chop lifecycle poisoning, launch collisions, and log hygiene
goal: "Chop runs finalize from the agents the runner actually launched (no more
  ambient-env registry pollution falsely failing runs and re-firing triggers),
  explicitly-named proposals skip gracefully instead of failing the run when their agent
  name is taken, the dead fix_just chop is revived by fixing its chezmoi once_per
  config, and lumberjack log rotation stops rewriting 50MB per line and leaking multi-GB
  temp files.

  "
phases:
  - id: linkage-scoping
    title: Explicit chop-launch linkage scoping
    depends_on: []
    description:
      "'Explicit chop-launch linkage scoping' section: register chop-agent linkage only
      for explicit runner launches and continuation respawns, scrub ambient SASE_CHOP_*
      from unrelated child agents, and isolate the leaking launcher tests from real axe
      state."
  - id: finalize-matching
    title: Launch-matched lifecycle finalization and registry GC
    depends_on:
      - linkage-scoping
    description:
      "'Launch-matched lifecycle finalization and registry GC' section: finalize
      launched runs from records matched to the entry's launches (following retry
      chains), ignore unmatched records, and garbage-collect orphaned registry records."
  - id: name-collision-skip
    title: Graceful per-proposal skip on agent-name collision
    depends_on: []
    description:
      "'Graceful per-proposal skip on agent-name collision' section: treat a taken
      explicit agent name as an idempotent per-proposal skip with a recorded reason
      instead of failing the whole run, releasing once-per keys for skipped proposals."
  - id: log-hygiene
    title: Bounded lumberjack log hysteresis and tmp cleanup
    depends_on: []
    description:
      "'Bounded lumberjack log hysteresis and tmp cleanup' section: truncate capped logs
      with hysteresis so appends stay cheap, and clean up orphaned atomic-replace temp
      files."
  - id: chezmoi-config
    title: Chezmoi chop config repair
    depends_on: []
    description:
      "'Chezmoi chop config repair' section: remove the once_per key that permanently
      dead-locks fix_just and add agent-hood guards to the audit chops in the chezmoi
      axe config, verifying with doctor and dry runs."
create_time: 2026-09-09 19:53:06
status: wip
---

- **PROMPT:**
  [prompts/202607/chop_lifecycle_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/chop_lifecycle_fixes.md)

# Plan: Fix chop lifecycle poisoning, launch collisions, and log hygiene

## Context

Epic sase-6v redesigned axe chops into script-only jobs that emit structured results;
the runner launches proposed agents, records durable linkage in each lumberjack's
`agent_chops.json`, and a housekeeping pass (`finalize_launched_chop_runs` in
`src/sase/axe/chop_lifecycle.py`) later finalizes `launched` runs from agent-completion
artifacts. A review of the live lumberjack state and logs on athena (2026-07-19) found
the new machinery mostly working — triggers, guards, `for_each` fan-out, map-form
config, and `vars` all behave as designed — but four implementation bugs and one
migrated-config bug are actively degrading the system:

1. **Registry pollution via ambient env.** `spawn_agent_subprocess`
   (`src/sase/agent/launch_spawn.py`) calls
   `record_chop_agent_launch_from_env(env=subprocess_env)`, and the launch-env builder
   deliberately forwards ambient `SASE_CHOP_LUMBERJACK/NAME/RUN_ID/PROMPT_HASH` into
   every child. Consequently _any_ agent spawned by a process descended from a
   chop-launched agent registers itself in the real registry under the parent's
   `(chop_name, run_id)`. The dominant real-world trigger: chop-launched agents working
   on the sase repo run `just check`, and several tests in
   `tests/test_axe_chop_agents.py` (e.g.
   `test_spawn_agent_subprocess_prepares_vcs_and_local_xprompt_env` and the
   `_spawn_agent_for_env_test` helpers) drive the real launcher without patching
   `sase.axe.state.JACK_STATE_DIR` or scrubbing chop env — appending fixture records
   (pid 4321, project `proj`, cl `feature/test`, timestamp `20260101120000`) to the real
   `~/.sase/axe/lumberjacks/<jack>/agent_chops.json`. Three lumberjack registries were
   polluted this way on 2026-07-19 alone.
2. **Housekeeping trusts every record.** `finalize_launched_chop_runs` evaluates _all_
   registry records matching `(chop_name, run_id)`; one dead unrelated pid without
   artifacts forces the whole run to `action_failed`. Every `refresh_docs[sase]` and
   `recent_bug_audit[sase]` run on 2026-07-19 was falsely failed by the fixture records.
   Because those chops use `checkpoint: on_action_success`, the false failures also
   prevent the `git.commits_since` watermark from ever being established, so the audit
   trigger re-fires every `run_every` cycle (hourly) instead of per-200-commits — an
   expensive agent-launch loop. Registry records are also only ever removed when their
   run finalizes, so records for runs that vanished from history leak forever (21 of 23
   records in the telegram registry are stale).
3. **Agent-name collisions fail the whole run.** Runner-derived proposal names embed a
   per-run token (`chop.refresh_docs.sase.9_216215.1`), but a script-supplied
   `agent_name` is used verbatim (`prepare_chop_proposals` in
   `src/sase/axe/chop_proposals.py`). The bugyi-chops audit scripts intentionally name
   agents `audit_bugs.<project>.<HEAD-short>` as an idempotency key; when the trigger
   re-fires at an unchanged HEAD the launch raises `Agent name '…' is taken`,
   `process_script_chop_result` marks the run `action_failed`, and the checkpoint again
   never advances. Run history shows this exact failure
   (`Agent name 'audit_bugs.sase.7ef34829ef0a' is taken`).
4. **Bounded-log churn and temp-file leak.** `append_bounded_log`
   (`src/sase/axe/_state_lumberjack.py`) rewrites the entire capped file through a
   `NamedTemporaryFile` + `os.replace` on _every_ append once the log sits at its cap.
   The hooks and telegram lumberjack logs sit at exactly 50MB, so each appended line
   costs a 50MB read+write, and any interruption mid-replace leaks a 50MB
   `.lumberjack-*.log.*.tmp` file. `~/.sase/axe/logs/` currently holds roughly 90 such
   orphans (~3.2GB). Nothing ever cleans them.
5. **fix_just is permanently dead by config.** The chezmoi `axe:` overlay gives the
   `fix_just` chop `once_per: "{proposal.id}"`. The script's single proposal id is the
   static string `fix`, so after the first launch (2026-07-18) the runner-owned seen
   store blocks every subsequent run forever — run history shows nothing but
   `skipped · all 1 proposal(s) skipped by once-per dedupe` since. The once-per store
   has no TTL; keys release only on launch/action failure. The guard the config actually
   wants already exists in the same stanza (`inhibit_if: changespec` with the
   `sase_fix_just_` prefix), and the proposal's agent name uses the `-@` indexed-name
   template, so the `once_per` line is simply wrong and must go.

Non-bugs confirmed during the review, so later phases do not chase them: `%wait`
proposals launching with `workspace_num 0` and a placeholder workspace dir is the
deferred-workspace design; the pre-07:57 minute-cadence bursts and
`chop.refresh_docs.sase.1` collisions were fixed by the already-landed run-token naming
and `run_every` bookkeeping; trigger `{target.name}` templating and lumberjack-level
env/secret refs work.

Per the Rust core boundary rule: everything below is launcher/subprocess/IO and
state-file housekeeping — host-side Python concerns that stay in this repo. No
`../sase-core` changes are required.

## Design overview

The linkage design principle shifts from _"any descendant launch inherits the chop
identity"_ to _"linkage is explicit"_: a registry record is created only for (a)
proposal launches the runner itself performs — which already pass
`build_chop_launch_env(...)` through `extra_env` (`launch_chop_proposals` in
`src/sase/axe/chop_proposals.py`) — and (b) continuation respawns of an already-linked
agent (retry spawn, model-fallback spawn), which must keep re-registering the successor
pid so housekeeping tracks the live process. Everything else — nested launches by chop
agents, test-suite launches — neither registers nor inherits `SASE_CHOP_*`.

Housekeeping symmetrically stops trusting the registry blindly: records are matched to
the run entry's own `launches` (both sides already carry `artifacts_timestamp`), retry
successors are followed through the `retried_as_timestamp` chain that
`_agent_completion` already understands, unmatched records are ignored for status
purposes, and orphaned records are garbage-collected. This heals the currently-polluted
registries without manual surgery.

## Phases

### Explicit chop-launch linkage scoping

In `src/sase/agent/launch_spawn.py` and `src/sase/axe/chop_agents.py`:

- Registration: only call the registry when chop metadata was provided explicitly by the
  caller via `extra_env`, or when the spawn is a continuation of the current (already
  chop-linked) agent — `retry_transfer_from_pid` is set on the retry path
  (`src/sase/axe/run_agent_retry_spawn.py`), and the codex-fallback respawn path must be
  audited for its equivalent marker. For continuations, reading the current process
  environment remains correct because the respawning process _is_ the linked agent.
  Restructure `record_chop_agent_launch_from_env` (or add a scoped wrapper) so the
  "which env is authoritative" decision is made by the launcher, not by scanning the
  merged child env.
- Env propagation: scrub ambient `SASE_CHOP_LUMBERJACK/NAME/RUN_ID/PROMPT_HASH` from
  spawned children's environments the same way stale `SASE_AGENT_*` identity vars are
  already scrubbed, while values supplied via `extra_env` (real proposal launches) and
  continuation respawns still land in the child env so `agent_meta_from_chop_env` keeps
  stamping `agent_meta.json` for genuinely chop-launched agents. Update
  `test_spawn_agent_subprocess_replaces_ambient_agent_identity_with_launch_env`, which
  currently asserts the ambient pass-through.
- Test hygiene, closing the pollution vector regardless of where the suite runs: make
  every test in `tests/test_axe_chop_agents.py` that drives the real launcher isolate
  `SASE_HOME` and `sase.axe.state.JACK_STATE_DIR` (two tests in the file already show
  the pattern), and add an autouse conftest fixture that deletes the four `SASE_CHOP_*`
  vars from the test environment so no test can observe or forward a hosting chop
  agent's identity.

Tests: launcher unit tests covering all three registration cases (explicit extra_env →
recorded; ambient-only → not recorded, env scrubbed; retry continuation → recorded),
plus the migrated env-sanitization assertions.

### Launch-matched lifecycle finalization and registry GC

In `src/sase/axe/chop_lifecycle.py` and `src/sase/axe/chop_agents.py`:

- Match records to launches: for each `launched` run, resolve the record set against
  `entry.launches` by `artifacts_timestamp` (helper `_launch_for_record` already
  exists), then extend the matched set transitively with retry successors — a matched
  record whose `done.json` carries `retried_as_timestamp` claims the record bearing that
  artifacts timestamp. Only the matched set participates in completion evaluation;
  unmatched records are logged into the run output and ignored. Keep the fail-closed
  rule per launch: a launch with no matching record (and no retry successor) still
  produces the "linkage incomplete" failure detail, and `expected`-count semantics
  degrade to `action_failed`, never hang.
- Registry GC: during the housekeeping pass, delete records whose `(chop_name, run_id)`
  no longer resolves to a run entry, or whose run entry is already terminal. This
  automatically clears the existing pid-4321 fixture records, the pre-cutover legacy
  records (`sase_refresh_docs`, `sase_recent_*_audit`, stale `tg_inbound` rows), and any
  future strays, so no manual state surgery is needed. Remove the dead no-op conditional
  in `_registry_path` while in the file.

Tests: finalization with extra unmatched records (run still succeeds), retry-chain
following, missing-record fail-closed behavior, and GC of orphaned/terminal-run records.

### Graceful per-proposal skip on agent-name collision

In the launch path (`src/sase/axe/chop_proposals.py`,
`src/sase/axe/chop_runner_script_result.py`) and the agent name-reservation code that
raises the `Agent name '…' is taken` error:

- Introduce a typed exception (e.g. `AgentNameTakenError`) at the reservation site so
  the runner can distinguish collisions from other launch failures without string
  matching.
- In `launch_chop_proposals`, catch the typed error per proposal _only when the proposal
  supplied an explicit `agent_name`_ (runner-derived names embed the run token and must
  never collide — a collision there stays a hard failure). Record the proposal as
  skipped with a name-collision reason (mirroring the once-per duplicate shape in
  previews/run history), release its once-per key like other unlaunched proposals,
  relink dependent `wait_on` proposals the same way once-per dedupe does, and continue
  with the remaining proposals.
- Aggregate outcomes in `process_script_chop_result`: all proposals skipped → run status
  `skipped` with the reason visible in history; some launched → `launched` as today;
  non-collision launch errors keep current `action_failed`/partial semantics.

This turns the bugyi-chops audits' revision-keyed names into intentional idempotency:
re-firing at an unchanged HEAD becomes a quiet, explained skip instead of an hourly
`action_failed` loop.

Tests: collision on explicit names (single and multi-proposal with wait relinking),
collision on derived names still failing, once-per key release on skip, and status
aggregation.

### Bounded lumberjack log hysteresis and tmp cleanup

In `src/sase/axe/_state_lumberjack.py` and lumberjack/orchestrator startup:

- Hysteresis: when an append would cross `max_bytes`, truncate to a configured fraction
  of the cap (e.g. keep the newest half plus the new payload with the truncation marker)
  instead of trimming to exactly the cap, so the next ~25MB of appends are plain cheap
  appends rather than 50MB rewrites per line. Preserve the existing truncation marker
  and tail semantics; document the fraction as a module constant.
- Tmp cleanup: on lumberjack start (`ensure_lumberjack_dirs` or the orchestrator boot
  path), remove `.{log_name}.*.tmp` orphans in the axe logs directory older than a
  conservative age threshold, logging a count. This reclaims the ~3.2GB currently leaked
  and keeps future interruptions from accumulating.

Tests: cap-crossing keeps the file at/below the cap and well under it after truncation
(hysteresis observable), subsequent appends do not rewrite, and stale tmp files are
removed while fresh ones are left alone.

### Chezmoi chop config repair

Config-only work in the chezmoi repo (opened through the repo-access skill; touch only
`home/dot_config/sase/sase_athena.yml`, no memory or agent-instruction files):

- Delete `once_per: "{proposal.id}"` from the `fix_just` chop stanza. The
  `inhibit_if: changespec` guard already provides the intended "one fixer at a time"
  behavior, and the proposal's `-@` agent name cannot collide. No seen- store surgery is
  needed: with no `once_per` and no script-supplied `dedupe_key`, the stored `fix` key
  is inert.
- Add `inhibit_if: agent_hood` guards to `recent_bug_audit` (hood `audit_bugs`) and
  `recent_improvement_audit` (hood `audit_improvements`) so a still-running audit
  suppresses the next trigger fire at the source instead of relying on downstream
  name-collision skips.
- Verify on the live deployment: the axe config loads (fail-closed validation passes),
  `sase axe chop doctor` is clean, and `sase axe chop run fix_just -n -V` shows the
  proposal accepted (not once-per skipped) in the dry-run proposals table. Follow the
  chezmoi repo's own instructions for applying committed changes.

## Testing strategy

Each sase-repo phase lands with the unit coverage listed in its section and must pass
`just check`. The linkage-scoping phase is first because finalize-matching builds on the
explicit-linkage semantics; the remaining phases are independent. The chezmoi phase
validates against the currently-deployed sase (its two edits do not depend on the code
fixes). End-to-end healing of the live registries (pid-4321 records, stale telegram
rows) arrives with the registry GC once the release containing these fixes is deployed;
no phase should hand-edit live state under `~/.sase/`.

## Risks and mitigations

- **Continuation respawns losing linkage.** The retry path registers via ambient env
  today; the linkage-scoping phase must keep that working (covered by an explicit
  retry-continuation test) and audit the codex-fallback spawn path, which currently
  patches `record_chop_agent_launch_from_env` in its tests — a sign it flows through the
  same seam.
- **Finalize matching regressing the retry special-case.** `_agent_completion` already
  treats `failed + retried_as_timestamp` as success for the original attempt;
  chain-following must keep evaluating the successor so a run is not finalized while the
  retry is still running.
- **Name-collision skip masking real bugs.** Scoping the graceful skip to
  explicitly-named proposals keeps runner-derived-name collisions loud, and the skip
  reason is persisted in run history and previews so `sase axe chop list -v` explains
  every quiet cycle.
- **Log truncation losing recent context.** Hysteresis keeps the newest tail (same
  guarantee as today's cap) and only reduces how much _old_ tail survives a truncation
  event; the truncation marker still records that trimming occurred.
