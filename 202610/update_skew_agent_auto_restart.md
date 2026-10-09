---
tier: epic
title: Update-Skew Agent Auto-Restart
goal: When a live sase update breaks a running agent before its model turn, sase puts
  it back once, under the same name, exactly as `,x` plus an unmodified submit would,
  and tells the user what it did and why in one calm amber ↻ story. Every other update-shaped
  failure is surfaced with its reason and never silently swallowed. The refresh-path
  bug behind the 2026-10-09 incident can no longer recur.
decisions:
  decision_record:
    ask: Add a decisions-web record for this design (update-skew restarts are at most
      once per lineage, pre-provider only)?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: refresh-exec-first
  title: Exec-first runner refresh and import firewall
  depends_on: []
  size: small
  description: 'refresh-exec-first: remove every lazy sase import between the HEAD-moved
    check and os.execv in refresh_runner_code_after_wait, move the %auto reconcile
    into the refreshed process, and add an import-firewall test.'
- id: failure-facts
  title: Runner boot identity, lifecycle breadcrumbs, and failure facts
  depends_on: []
  size: medium
  description: 'failure-facts: persist the runner''s boot code identity and lifecycle-phase
    breadcrumbs in agent_meta, and record stdlib-only structured failure facts plus
    a skew-suspect prefilter in done.json at every runner failure writer.'
- id: core-verdict
  title: sase-core failure classifier, ledger state machine, and recovery wire
  depends_on:
  - failure-facts
  size: medium
  description: 'core-verdict: add the agent_auto_restart domain to sase-core (facts,
    witness, and verdict wires, the family-based classifier, ledger transitions, episode
    ids, the done-marker recovery field and status bucket, and the ↻ report glyph),
    then the Python adapter and the core pin bump.'
- id: witness-scan
  title: Skew witnesses and the read-only scan command
  depends_on:
  - core-verdict
  size: medium
  description: 'witness-scan: collect managed roots and the W1-W3 witnesses (boot
    identity drift, update journal, file-level proof with the culprit commit), build
    legacy inputs from logs, and ship `sase agent auto-restart scan` for corpus replay.'
- id: healer
  title: The healer, at-most-once ledger, and auto-restart CLI
  depends_on:
  - witness-scan
  size: medium
  description: 'healer: implement `sase agent auto-restart run`, which claims the
    ledger, classifies the failure, checks quiescence, runs the fresh-interpreter
    probe, applies the skip rules and storm breaker, preserves evidence, and relaunches
    headlessly through plan/execute_agent_restart with provenance. Also adds the config
    block, the beta flag, and the list/show/resume commands.'
- id: trigger
  title: Runner doorbell, scheduler job, and waiter safety
  depends_on:
  - healer
  size: medium
  description: 'trigger: have the dying runner drop a stdlib doorbell, mark recovery
    pending, and silence its failure notification. Add the fs-triggered scheduler
    job that sweeps and submits the healer as a durable proc. Keep waiters and wait_checks
    correct while a recovery is in flight.'
- id: episode-notify
  title: One upserted ↻ notification and live report per update episode
  depends_on:
  - healer
  size: medium
  description: 'episode-notify: replace the healer''s minimal notifications with the
    designed experience: one upserted amber ↻ episode notification with a live ChopReport,
    title refresh without re-toasting, exactly one information toast, and the loud
    escalation copy.'
- id: ux-surfaces
  title: Agents-tab ↻ RESTARTING state, provenance line, help, and update hint
  depends_on:
  - healer
  size: medium
  description: 'ux-surfaces: render in-flight recoveries as amber ↻ RESTARTING with
    a dim reason, add decline hints and the replacement''s ↻ provenance line with
    its preserved error report as a `v` file hint, and update the help modal and the
    `sase update` advisory hint, with visual snapshots.'
- id: land
  title: Remove the beta flag, document, and replay the incident end to end
  depends_on:
  - refresh-exec-first
  - trigger
  - episode-notify
  - ux-surfaces
  size: small
  description: 'land: delete the agent_auto_restart flag''s Off branches and close
    its bead, write the user-facing doc, add the end-to-end incident replay acceptance
    test, and record the deferred follow-ups.'
proposed_by: bbugyi200.athena.research.47.linker.w0
decided_by: auto
create_time: 2026-10-09 15:02:02
status: wip
bead_id: sase-1j6
---

- **PROMPT:** [prompts/202610/update_skew_agent_auto_restart.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/update_skew_agent_auto_restart.md)
- **BEAD:** [sase-1j6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1j6/README.md)

# Plan: Update-Skew Agent Auto-Restart

## Context

On 2026-10-09, five sase agents died with
`ImportError: cannot import name 'auto_launch_prefix' from 'sase.monitor.continuation_delivery'`.
All five crashed in `refresh_runner_code_after_wait()`
(`src/sase/axe/run_agent_runner_refresh.py`), the function whose job is to survive a
code swap during a `%wait`:

1. It noticed sase's HEAD had moved.
2. Before its `os.execv`, it lazily imported modules from the new tree.
3. The old in-memory code asked for a symbol that `9fd8a081f4` had just deleted.

None of the five had started its model turn, so `,x` plus an unchanged `<ctrl+g><enter>`
was exactly the right repair.

The research report
`research:202610/update_skew_agent_auto_restart/update_skew_agent_auto_restart.md` (read
it with `sase artifact read`) measured about 43 skew failures on this host since July,
about 4.8% of failed agents. The user has accepted **all** of its adjusted requirements:

| #   | Requirement this epic implements                                                                                                                                                                                                 |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | **At most one automatic restart per lineage**, claimed in a durable ledger before any mutation. A replacement that fails again is reported and never retried.                                                                    |
| A2  | **Phase-aware.** Pre-provider deaths get automatic `,x` semantics with forced name reuse. Post-provider deaths are not relaunched: the held workspace stays and the user is notified. Plan-handoff and gate deaths ask the user. |
| A3  | **Signature ∧ witness ∧ probe.** A matching error pattern alone is never enough.                                                                                                                                                 |
| A4  | **Quiescence gate.** Act only when no update holds the code-swap writer lock, the tree has been quiet for `quiescence_seconds`, and a fresh interpreter passes the probe.                                                        |
| A5  | **One upserted notification per update episode**, with a live report and one +1 per agent. Escalations stay loud.                                                                                                                |
| A6  | **Prevention ships first:** the exec-first refresh fix plus an import-firewall test (phase `refresh-exec-first`).                                                                                                                |
| A7  | **Preserve evidence before the wipe.** Copy the error report, traceback, log tail, facts, and verdict into the recovery bundle, and link it everywhere.                                                                          |
| A8  | **User intent wins.** Skip killed, dismissed, mid-`,x`, and question- or gate-holding agents. Add a storm breaker and a permanent config kill switch.                                                                            |
| A9  | **Headless.** Use `plan_agent_restart` / `execute_agent_restart` (the provider-drain seam) with the same `prepare_kill_and_edit_prompt` rewrite. Never synthesize TUI keys.                                                      |

The open questions in the research are settled as follows:

- Post-provider deaths are notify-only in this epic. Resume-in-place is a follow-up.
- Name reuse is always forced, even for prompts without `%id`.
- A replacement broken by a _later_ update is not retried.
- Release pinning is a separate epic.

## Mental model

> **If a sase update breaks a running agent before it did any model work, sase puts it
> back once, under the same name, and tells you exactly what it did and why.**

Everything in this epic serves that sentence:

- **One verb.** An automatic restart is `,x` plus an unmodified submit, run headlessly.
- **One budget.** Each lineage gets one automatic restart.
- **One color and glyph.** Amber `↻` (`UPDATE_CAUTION_ACCENT`, `#FFAF5F`) already means
  "sase changed under running code" in the Updates panel. Red stays reserved for things
  that need the user.
- **One story per update.** A grouped notification replaces N red rows and N toasts.

## Architecture

```text
dying runner (torn code; only boot-imported, stdlib-only helpers)
  ├─ done.json: failure_facts + recovery.state="pending"     (phase failure-facts / trigger)
  ├─ ~/.sase/agent_auto_restart/doorbell/<key>.json          (atomic tmp+rename)
  └─ failure notification sent silent=True
          │  fs trigger (≤10 s), or the max_quiet sweep (60 s)
          ▼
scheduler job agent_auto_restart (fresh subprocess every tick)
  ├─ enumerate candidates: doorbells, recent failed rows (legacy and log-only), stale claims
  ├─ feature off or paused → re-surface pending failures loudly, clear doorbells
  └─ work found → submit durable proc `sase agent auto-restart run -p -j`
                  (concurrency key agent-auto-restart; cwd ~)
          ▼
healer (fresh process, after quiescence)
  claim ledger (O_EXCL) → gather witnesses W1–W3 → core classify_agent_failure
  → quiescence + W4 probe (else deferred) → skip rules + storm breaker
  → evidence bundle → plan_agent_restart(follow_live_autonomy=True)
  → wipe-scope guard → execute_agent_restart (forced reuse, provenance env)
  → ledger launched → upsert episode notification and rewrite the live report
```

The healer is the only actor that mutates agent state. The runner only records facts and
rings the doorbell. The job only enumerates and submits. Recovery never runs in the
dying process, because the torn interpreter is the hazard. It never runs in the TUI,
which may be closed, stale, or racing the prompt bar.

**Why a job and not a runner `Popen`.** The research proposed having the runner `Popen`
`code_swap_guarded_exec.py`. On this codebase that cannot work reliably:
`sweep_own_agent_scope()` (`src/sase/agent/_scope_sweep_own.py`) kills every process
left in the runner's cgroup scope at exit, and a doorbell child would be one of them.

A file-drop doorbell plus an `fs`-triggered scheduler job avoids that problem. It also:

- gives quiescence for free;
- merges the doorbell responder and the sweeper into one code path;
- runs every check in a fresh subprocess;
- follows the scheduler's "jobs propose, others launch" contract via a durable proc,
  exactly as the provider drain does
  (`src/sase/llm_provider/usage_limit_disable.py::_submit_drain`).

## Shared contracts

Every phase must honor these shapes. Each JSON object carries `schema_version: 1`.

### `agent_meta.json` additions (phase `failure-facts`)

- **`code_identity`**: the boot snapshot of the code this process imported:
  `{schema_version, captured_at, roots: [{name, role: host|core|plugin, version, commit, source_root, install_type}]}`.
  - Covers the `sase` host, `sase-core-rs`, and each editable `sase-*` plugin.
  - `commit` is null for non-editable installs; version drift still counts as a witness.
- **`booted_at`**: ISO-8601 timestamp of this process image's boot. A refresh re-exec is
  a new image, so it gets a new identity.
- **`lifecycle_phase`** plus **`lifecycle_phase_at`**: the latest boundary crossed.
  - Values:
    `booting → waiting → preparing → provider_running → provider_done → finalizing`,
    plus `handoff` while plan, question, monitor, gate, or pipe markers are processed.
  - **`lifecycle_phases`** keeps a bounded history (at most 16 entries of
    `{phase, at}`).
- **`auto_restart`** (phase `healer`; written only on a replacement):
  `{of_artifacts_dir, of_timestamp, lineage_root, episode_id, ledger_key, evidence_dir, signature, from_rev, to_rev, culprit_commit, culprit_subject, restarted_at}`.

### `done.json` additions

- **`failure_facts`** (phase `failure-facts`), with these fields:
  - `exception_chain`: at most 8 links, following `__cause__`/`__context__` with cycle
    detection. Each link is `{type, qualname, module, message ≤2 KiB}`.
  - `import_error`: `{name, path, missing_symbol}`.
  - `attribute_error`: `{module, attribute}`, only when the target is a module.
  - `frames`: at most 64 entries of `{file, function, line}`, innermost last.
  - `last_frame_file`.
  - `lifecycle_phase`: the in-memory current phase.
  - `skew_suspect: bool`.
  - `captured_at`.
- **`recovery`**:
  `{state, reason, reason_text, requested_at, updated_at, episode_id, ledger_key}`.
  - `state` is one of `pending | deferred | launching | declined | launched`.
  - The dying runner writes `pending` (phase `trigger`). The healer advances the rest
    through the existing marker-mutation path
    (`update_agent_artifact_index_for_marker_mutation`).
  - `launched` is rarely observed, because the forced-reuse wipe removes the old row.

### Ledger: `~/.sase/agent_auto_restart/` (phase `healer`)

Use `sase_subdir("agent_auto_restart")`. Nothing here may live inside an artifacts
directory, because a wipe could remove it.

| Path                                             | Content                                                                                                                                                                                                        |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ledger/<project>__<lineage_root>.json`          | One record per lineage. Created with `atomic_write_json(..., exclusive=True)` from `src/sase/notification_gates/durability.py` (no-clobber `os.link`). Later transitions use atomic replace under `file_lock`. |
| `doorbell/<project>__<artifacts_timestamp>.json` | `{schema_version, artifacts_dir, project, agent_name, failed_at, skew_suspect}`. Deleted once a ledger record owns the failure.                                                                                |
| `episodes/<episode_slug>.report.json`            | The live ChopReport for one episode. It is a projection of the ledger records carrying that `episode_id`, re-rendered under a file lock on every change.                                                       |
| `state.json`                                     | `{paused, paused_at, paused_reason, paused_episode}`, owned by the storm breaker.                                                                                                                              |

**Lineage root.** Take the first that exists: `agent_meta.auto_restart.lineage_root`,
then `retry_chain_root_timestamp`, then the row's own artifacts timestamp. A manual `,x`
carries none of these, so it starts a fresh lineage.

**Ledger record.**
`{schema_version, key, lineage_root, state, claimed_at, claimer: {pid, process_identity}, failed_artifacts_dir, agent_name, project, episode_id, verdict, witnesses, planned_name, plan_digest, launched_artifacts_dir, evidence_dir, decline_reason, deferrals, history: [{state, at, note}]}`.
`process_identity` comes from `src/sase/core/process_identity.py`, so liveness checks
are safe against PID reuse.

**States and transitions.** These are owned by sase-core (phase `core-verdict`):

```text
claimed  → deferred | declined | launching
deferred → claimed (a later healer pass re-attempts) | declined (max_defer_seconds elapsed)
launching → launched | settled_failed
launched → settled_ok | settled_failed
```

**Write order.** The claim is recorded before any mutation, and any uncertainty resolves
to "attempt spent, notify".

**Crash rule.** When a claimer died while `launching`, a later pass looks for a row
named `planned_name` whose artifacts are newer than the claim. If one exists, the pass
adopts it as `launched`. Otherwise the record becomes `settled_failed` and the failure
is re-surfaced. A pass never re-launches, because forced reuse would wipe the
replacement it just started.

### Episodes

- **Episode id.** Use the **culprit commit** from W3 when it is known, e.g.
  `sase@9fd8a08`. That groups all five of the 2026-10-09 deaths, including the one that
  woke after a later install. Otherwise use a stable digest of the target revision set.
- **Dismissed episodes.** If the user dismissed an episode's notification row, a later
  restart in that episode starts a new row (`<episode>#2`). A restart is never
  invisible.

## Signature catalog and decision rules

Catalog **families, not symbols**: yesterday's symbol is gone today. A family matches
only when the module or symbol has a **managed origin**:

- Its path resolves under a managed root: the `sase` checkout, an editable `sase-*`
  plugin, or `sase_core_rs`.
- It **never** resolves under the agent's workspace directory. Agents that work on the
  sase repo itself raise workspace ImportErrors all day; those must never match.

| Tier                        | Family                                                                                                                                                                                                                                                                                                                                                               | Recover when                     |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| 1 Torn Python code          | `ImportError: cannot import name 'X' from 'M'`; `ModuleNotFoundError: No module named 'M'`; `AttributeError: module 'M' has no attribute 'X'` (module target only); `… from partially initialized module 'M'` (only if the probe passes, otherwise it is a real circular import); `SyntaxError`/`IndentationError` in a managed file (only if the file compiles now) | signature ∧ (W1 ∨ W2) ∧ W4       |
| 2 Rust binding or wire skew | `sase_core_rs is importable but does not expose binding 'X'`; `… wire schema mismatch: got N, expected M`; `wire is stale`; `No module named 'sase_core_rs'`; partially-initialized `sase_core_rs`                                                                                                                                                                   | signature ∧ W4, after quiescence |
| 3 Data-format skew          | `was written by a newer or unknown sase version`; `uses a format this process does not understand`                                                                                                                                                                                                                                                                   | signature ∧ (W1 ∨ W2)            |
| 4 Usually a real bug        | signature-mismatch `TypeError`; `NameError` in a managed module                                                                                                                                                                                                                                                                                                      | **Never.** Annotate only.        |

**Witnesses:**

| ID  | Witness                                                                                           | Notes                                                                                                                                                                                                                                                                                                                                                    |
| --- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| W1  | Persisted boot `code_identity` ≠ current identity                                                 | Primary witness.                                                                                                                                                                                                                                                                                                                                         |
| W2  | A `sase update` journal row (`~/.sase/logs/dev_update.jsonl`) between `booted_at` and the failure | Secondary. It misses HEAD moves that were never journaled.                                                                                                                                                                                                                                                                                               |
| W3  | File-level proof                                                                                  | The symbol or module exists at the boot revision and not at HEAD, or the reverse. `git log -S<symbol> <boot>..HEAD -- <file>` names the **culprit commit**, which becomes the episode id and the notification headline.                                                                                                                                  |
| W4  | Fresh-interpreter probe                                                                           | Imports the **current** versions of every managed module on the failing traceback plus the target module, and checks any `sase_core_rs` binding the error names. It does **not** retry the missing import, which would never succeed for removal-type skew like this incident. For addition-type skew it may also assert that the symbol exists at HEAD. |

**Phase gate.** A Tier 1–3 match authorizes relaunch only for `pre_provider` deaths
(`booting`, `waiting`, `preparing`):

| Death phase                                       | Mode                   | What happens                                                       |
| ------------------------------------------------- | ---------------------- | ------------------------------------------------------------------ |
| `provider_running`, `provider_done`, `finalizing` | `notify_post_provider` | Not relaunched; the user is notified with held-workspace guidance. |
| `handoff`                                         | `ask`                  | Not relaunched; the user is asked to decide.                       |

**Legacy rows** have no breadcrumbs or facts: agents booted before this epic, and
log-only deaths during module import. For these, use regex over `error`, `traceback`,
and the runner-log tail. Treat the row as pre-provider only when its frames include no
`run_execution_loop`, `invoke_agent`, or finalizer frames, or when the log shows the
`Refreshing sase runner code after dependency wait` line and no provider start.

**Never restart:**

- Provider errors, 429s, usage limits, auth, or context overflow. `llm_provider.retry`
  and the provider drain own these.
- Kills, SIGTERM, cancels, rejected plans, and completed gate or monitor handoffs.
- OOM, SIGKILL, timeouts, disk full, `PermissionError`, `MemoryError`, `RecursionError`.
- Directive or macro errors and unknown aliases.
- Commit-finalizer, publish, and gate-dispatch failures.
- Workspace or linked-repo materialization failures.
- Failures from tools, tests, or monitor commands.
- Any ImportError from a workspace or third-party module.
- "Runner exited without recording an error" with no signature in the log tail.
- Anything `plan_agent_restart` refuses.

## UX specification

### Agents tab (phase `ux-surfaces`)

**In-flight states.** While `recovery.state` is `pending`, `deferred`, or `launching`,
the row shows a bold amber **`↻ RESTARTING`** in place of red `FAILED`, followed by one
dim hint:

- `sase updated 9c5000f → 9fd8a08 · restarting once the update settles` (pending or
  launching), or
- `↻ waiting for the sase update to finish` (deferred).

A `pending` older than `pending_resurface_seconds` with no ledger record renders as
normal `FAILED` with the dim hint
`auto-restart never ran — is the sase scheduler running?`.

**After relaunch.** The old row disappears exactly as it does after `,x`. The
replacement appears under the **same name** with a `↻` chip. Its identity header gains a
provenance block:

```text
↻ Auto-restarted after sase update 9c5000f → 9fd8a08
  was: ImportError · cannot import name 'auto_launch_prefix' (sase.monitor.continuation_delivery)
  broke 12:04 refreshing after %wait · before its model turn · nothing lost
```

The preserved `error_report.md` is offered as a file hint, so the existing `v`
(view_files) key opens it. **No new keymap.**

**Declined.** The row is a normal `FAILED` plus one dim reason, for example:

- `auto-restart skipped — already restarted once`
- `auto-restart skipped — no sase update during this run (looks like a real bug)`
- `not restarted — died after its model turn; workspace #3 held with its changes`

### Notification (phase `episode-notify`)

**Fields:**

| Field            | Value                                                                                                                                                        |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `sender`         | `agent.auto-restart`                                                                                                                                         |
| `icon` / `color` | `↻` / `#FFAF5F`                                                                                                                                              |
| `tags`           | `sase-update`, `auto-restart`, tier id                                                                                                                       |
| `dedup_key`      | `agent-auto-restart:<episode>`                                                                                                                               |
| `action`         | `ViewReport`, with `action_data = {report_path, report_title, report}`: the live path plus an inline snapshot so Telegram and mobile render without the file |
| `files`          | Each agent's preserved `error_report.md` and its evidence bundle                                                                                             |

**Notes** for the 2026-10-09 incident:

```text
↻ Restarted 5 agents after sase update 9fd8a08
sase changed while they waited: their in-memory code asked the new files for auto_launch_prefix, which 9fd8a08 ("feat(autonomy): structural inheritance of live record…") removed.
• research.46.final, research.46.image, toobig-7h.test_justfile_lint.0, sase-1j1.6 (sase) — relaunched under the same names
• 0yz (bob-cli) — relaunched under the same name
Nothing was lost: all 5 broke while refreshing after %wait, before their model turn.
Each agent gets one automatic restart. If one breaks again you will get a normal failure notice and sase will not retry it.
```

**Copy rules:**

- `notes[0]` never contains "fail" or "error"; exception text appears only in note 2 or
  later.
- A single-agent episode reads
  `↻ Restarted research.46.final after sase update 9fd8a08`.
- Cost is stated honestly.

**Live report.** Built only from `ChopReport` blocks (`src/sase/chops/report.py`; tones
`ok`, `warn`, `muted`):

- **headline**: `5 agents restarted · 0 left alone`.
- **kv**: Update `9c5000f → 9fd8a08`, the culprit commit subject, the roots changed, and
  the witnesses fired.
- **rows**: `Agent | Project | Broke during | Signature | Action | Now`. The **Now**
  column is live (RUNNING, DONE, or FAILED) and is rewritten whenever the ledger
  settles.
- **bullets**: "Left alone", each with its reason.
- **text**: "Why this happened", in two plain sentences.
- **divider**, then one dim line naming the prevention work.

**Toasts.** Exactly **one** information toast per episode, when it is created. A +1
never toasts. A title refresh must not move delivery cursors, so it never re-toasts or
re-pushes.

**Escalations** are loud, error severity, and go to the Errors bucket:

| Situation                      | Title                                                                                                                                  |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| Post-provider                  | `research.46.final broke after its model turn during a sase update — workspace #3 held with its changes. Review, then ,x to relaunch.` |
| Other decline, or probe failed | `Couldn't restart research.46.final automatically — <reason>. Press ,x on it to retry by hand.`                                        |
| Replacement broke again        | The normal failure notification plus `This was its automatic restart after sase update 9fd8a08 — not retrying.`                        |
| Storm breaker tripped          | `Auto-restart paused: 13 agents broke within one update — this looks like a real bug, not an update race.`                             |

### CLI (phases `witness-scan` and `healer`; read the `cli_rules` memory first)

`sase agent auto-restart`: a bare invocation delegates to `list` through the central
`_default_list_subcommands()`. Subcommands and options are sorted, and every long option
has a short alias.

| Command                                                                           | Purpose                                                                                                                                           |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `list [-a/--all] [-j/--json]`                                                     | Ledger records, newest first, grouped by episode: amber header, state colored (launched green, deferred amber, declined dim, settled_failed red). |
| `resume`                                                                          | Re-arm after the storm breaker trips.                                                                                                             |
| `run (NAME \| -a/--artifacts-dir DIR \| -p/--pending) [-n/--dry-run] [-j/--json]` | The healer. `-p` is the job's target. A manual run still honors the ledger.                                                                       |
| `scan [-j/--json] [-l/--limit N] [-s/--since DURATION]`                           | Read-only replay of the classifier over history. Never writes the ledger.                                                                         |
| `show TARGET [-j/--json]`                                                         | One card: verdict, a ✓/✗ checklist of witnesses, the state timeline, and evidence and bundle paths.                                               |

### Config (phase `healer`)

This is a permanent user choice, so it is a config field and not a flag. It is a new
top-level block next to `agent_scope_teardown`:

```yaml
agent_auto_restart:
  enabled: true
  quiescence_seconds: 30
  max_defer_seconds: 1800
  pending_resurface_seconds: 600
  storm_max_per_episode: 12
  storm_max_per_30m: 20
```

Add it to `src/sase/default_config.yml` and `src/sase/config/sase.schema.json`; the
schema root is `additionalProperties: false`. Add typed getters modeled on
`get_agent_scope_teardown_enabled()` in `src/sase/config/_settings_system.py`, with
their re-exports.

### `sase update` hint (phase `ux-surfaces`)

When live runners hold advisory reader slots, add one line next to
`_advisory_warning_line()` in `src/sase/main/update_render.py`:

```text
3 running agents are still on 9c5000f. If this update breaks one, sase restarts it once automatically (sase agent auto-restart list).
```

Mirror it in the TUI dev-update modal.

## Phase refresh-exec-first: Exec-first runner refresh and import firewall

Prevention (A6). This phase is independent and should land first.

- In `refresh_runner_code_after_wait()` (`src/sase/axe/run_agent_runner_refresh.py`),
  perform **zero** `sase.*` imports between the identity comparison and `os.execv`.
  - Hoist the `sase.agent.multi_prompt_macros` helpers (`read_local_macros_path`,
    `serialize_local_macros`, `set_local_macros_path`, `restore_local_macros_path`) to
    module scope.
  - Hoist `planned_name_is_reserved_for_artifacts`
    (`sase.axe.run_agent_directive_identity`, used by
    `_validated_continuation_planned_name`) to module scope as well.
  - Check those modules for import cycles.
- **Exec first, reconcile after.** Remove `_reconcile_prompt_with_live_auto_state` from
  the pre-exec path.
  - The refreshed process applies the live autonomy record to the prompt file **before
    directive extraction**, gated on the refresh marker (`SASE_RUNNER_CODE_REFRESHED` is
    still set during the refreshed pass's bootstrap).
  - Hook it in `bootstrap_agent_run` (`src/sase/axe/run_agent_runner_bootstrap.py`)
    ahead of `extract_directives_and_write_meta`.
  - Keep the semantics: the live selection wins and `:plan` is preserved; agents without
    a record fall back to the legacy-key translation.
  - Update the module docstring's handoff lists accordingly.
- **Import-firewall test.**
  1. Install a `sys.meta_path` finder that raises on any `sase.*` import after the
     identity check.
  2. Mock `os.execv` and the identity functions.
  3. Assert the function reaches `execv` with local macros, a planned name, and an
     artifacts dir all present.
- Also test that an `A` toggle made during the wait (both on and off, plus `%auto:plan`)
  survives the re-exec through the new refreshed-pass reconcile.

## Phase failure-facts: Runner boot identity, lifecycle breadcrumbs, and failure facts

This phase is observation only: no behavior changes, and no notification or recovery
side effects.

- **New stdlib-only module** `src/sase/axe/runner_failure_facts.py`, imported at runner
  boot from `src/sase/axe/run_agent_runner.py`.
  - It may use only the stdlib and modules already imported at boot. Add an AST test
    that forbids other imports and any function-local import.
  - It exposes
    `capture_failure_facts(exc, *, phase, error_text=None, traceback_text=None) -> dict`,
    which never raises and is bounded as specified under Shared contracts.
  - It also exposes `facts_look_like_update_skew(facts) -> bool`, a cheap, deliberately
    over-inclusive prefilter over the Tier 1–3 families. The core classifier is
    authoritative.
  - Provider-turn errors are wrapped twice: `LLMInvocationError` and then
    `WorkflowExecutionError`. The capture must walk the cause chain so the root
    ImportError is visible.
- **Lifecycle breadcrumbs**, in a new boot-imported helper (for example
  `src/sase/axe/runner_lifecycle_phase.py`).
  - `mark_lifecycle_phase(artifacts_dir, phase)` updates an in-memory current phase,
    which facts capture reads without touching disk. It then best-effort writes
    `lifecycle_phase`, `lifecycle_phase_at`, and bounded `lifecycle_phases` through the
    existing atomic agent-meta writer.
  - Call it at these boundaries:

    | Phase              | Where                                                                                                           |
    | ------------------ | --------------------------------------------------------------------------------------------------------------- |
    | `booting`          | `main` / `bootstrap_agent_run`                                                                                  |
    | `waiting`          | `wait_for_dependencies` and `wait_for_runner_slot`                                                              |
    | `preparing`        | `launch_agent_run`                                                                                              |
    | `provider_running` | entering `run_execution_loop` / `invoke_agent`                                                                  |
    | `provider_done`    | provider returned                                                                                               |
    | `finalizing`       | `run_finalizers` / `finalize_loop`                                                                              |
    | `handoff`          | `_handle_killed_iteration` marker handling, `handle_plan_marker`, `continue_as_successor`, `handle_pipe_marker` |

  - Do not name anything `publish_phase_env`-like; that name is already taken for a
    different concept.

- **Boot code identity.** At module import, next to `_STARTUP_CODE_IDENTITY`, capture
  `code_identity` from the version inventory (`src/sase/version/_collector.py`, using
  plugin records that do not import plugin code). Persist it with `booted_at` during
  bootstrap.
  - Budget: at most about 50 ms of added boot time. Measure it. If the full inventory is
    slower, probe git only for editable roots.
  - Add both keys to the refresh preserved-metadata handling where needed.
- **Persist facts** at every failure writer:
  - `record_runner_error` (`src/sase/axe/run_agent_runner_errors.py`) captures from the
    live exception.
  - `write_error_done_marker` (`src/sase/axe/run_agent_runner_finalize.py`) and
    `build_done_marker` (`src/sase/axe/run_agent_markers.py`) gain an optional
    `failure_facts`.
  - `finalize_loop`'s failed outcomes and `_ensure_failed_done_marker` record phase-only
    facts.
  - Also teach `record_runner_error` to call the existing
    `source_skew.code_swap_explanation` so the log names the swap.
- **Tests:**
  - chain capture, cycles, and bounds;
  - the ImportError, ModuleNotFoundError, and AttributeError extraction rules;
  - prefilter positives and negatives (workspace ImportError text still sets
    `skew_suspect`; origin scoping is core's job);
  - breadcrumbs written in order in a fake run;
  - `done.json` contains facts for raised and loop-level failures.

## Phase core-verdict: sase-core failure classifier, ledger state machine, and recovery wire

Shared verdicts that every frontend must render identically are core logic (the
`rust_core_backend_boundary` rule). Open the linked repo with
`sase repo open sase-core`, read its `AGENTS.md`, and run `sase tool run check` there.

- **New domain** `crates/sase_core/src/agent_auto_restart/`: a `mod.rs` facade plus
  `wire.rs`, `catalog.rs`, `classify.rs`, `ledger.rs`, `episode.rs`, and `tests.rs`. The
  binding domain is `crates/sase_core_py/src/agent_auto_restart/`.
- **Wires**, each with `schema_version`:
  - `AgentFailureFactsWire`: mirrors phase `failure-facts` exactly.
  - `AutoRestartContextWire`:
    `{managed_roots: [{name, root}], workspace_dir, outcome, kill_source, lifecycle_phase, has_pending_question, has_pending_handoff, is_remote, error_text, traceback_text, log_tail}`.
  - `AutoRestartWitnessesWire`:
    `{boot_identity, current_identity, journal_updates: [...], file_proof: {symbol, module, boot_has, head_has, culprit_commit, culprit_subject}?, probe: {ok, failures}?, refresh_log_line: {from, to}?}`.
  - `RecoveryVerdictWire`:
    `{tier, family, signature, origin_module, missing_symbol, phase_class: pre_provider|post_provider|plan_handoff|unknown, mode: relaunch|defer|notify_post_provider|ask|decline, reason (stable slug), reason_text (human copy), witnesses_fired, episode_id?}`.
- **`classify_agent_failure(facts?, context, witnesses) -> RecoveryVerdictWire`**
  implements the catalog, managed-origin scoping, the never-restart list, the phase
  gate, and the decision rules above.
  - It uses structured facts when present and regex over
    `error_text`/`traceback_text`/`log_tail` otherwise.
  - A missing W4 on a Tier 1–2 match yields `defer`, not `relaunch`. This lets the
    Python side run the probe only when it matters.
- **Ledger.** `AutoRestartLedgerRecordWire`, plus
  `advance_auto_restart_ledger(record, event) -> Result<record, error>` enforcing the
  transition table, and `auto_restart_lineage_root(meta, artifacts_timestamp)`.
- **Episodes.**
  `derive_auto_restart_episode(witnesses) -> {id, slug, culprit_short, from_rev, to_rev, label}`.
- **Done marker.** Add an optional `recovery` object to the agent-scan done wire. In the
  status-bucket mapping, in-flight states (`pending`, `deferred`, `launching`) map to an
  active "restarting" bucket instead of failed.
- **Report glyph.** Add `↻` to the ChopReport glyph allowlist in the Rust validator and
  in `src/sase/chops/report.py`.
- **Python side:**
  - `src/sase/core/agent_auto_restart_wire.py` (frozen dataclasses plus `*_from_dict`
    with schema checks; model it on `src/sase/core/retryability_wire.py`).
  - `src/sase/core/agent_auto_restart_facade.py` (uses `require_rust_binding`).
  - The done-wire field in `src/sase/core/agent_scan_wire.py`.
  - Parity tests, and a bump of `sase-core-revision.txt` per `docs/rust_backend.md`.
- **Golden tests in core.**
  - Positive: the five 2026-10-09 tracebacks (refresh-path frames, removal-type
    `auto_launch_prefix`) produce `relaunch` when W1 and W4 hold, and `defer` without
    W4.
  - Negative fixtures:
    - the same ImportError with no witness (`no_update_witness`);
    - a workspace-origin ImportError;
    - a traceback quoted in agent output;
    - provider 429 or usage limit;
    - killed;
    - a directive error;
    - a finalizer failure;
    - `TypeError` (Tier 4);
    - a post-provider ImportError (`notify_post_provider`);
    - a plan-handoff ImportError (`ask`).
  - Every ledger transition, legal and illegal.

## Phase witness-scan: Skew witnesses and the read-only scan command

- **New package** `src/sase/agent/auto_restart/`:
  - `managed_roots.py`: name, role, source root, current commit, and version from the
    runtime version inventory.
  - `witnesses.py`:
    - W1: compare `agent_meta.code_identity` with the current identity.
    - W2: journal rows from `src/sase/dev_update/journal.py` between `booted_at` (or the
      artifacts timestamp for legacy rows) and `finished_at`.
    - W3: bounded git with 2 s timeouts: `git cat-file`/`git grep` at the boot and HEAD
      revisions for the symbol, and
      `git log -S<symbol> --format=%H%x00%s <boot>..HEAD -- <file>` for the culprit.
    - The refresh log line, parsed from the runner log tail.
  - `inputs.py`: assemble facts, context, and witnesses from `done.json`,
    `agent_meta.json`, the runner log (`~/.sase/workflows/YYYYMM/*_ace-run-*.txt`, last
    200 lines), and pending question or handoff markers. Legacy and log-only rows go
    through the regex fallback.
- **CLI.** Add the `auto-restart` group under `sase agent`:
  - parser `src/sase/main/parser_agent_auto_restart.py`, registered in
    `_AGENT_SUBCOMMAND_ORDER`;
  - handler `src/sase/agents/cli_auto_restart.py`, routed from
    `src/sase/main/agent_handler.py`, modeled on the `hold` group;
  - `scan` only in this phase.
- **Scan output.** A colored table
  (`Agent | Project | Died | Phase | Signature | Witnesses | Verdict`) plus a summary
  line of counts by verdict mode. `-j` emits verdict wires.
  - Sources: failed `done.json` rows and dismissed bundles
    (`~/.sase/dismissed_bundles/`) within `--since` (default `7d`).
  - Strictly read-only.
- **Exit criterion.** Run `sase agent auto-restart scan -s 120d` on the host and record
  the counts in a phase-bead note.
  - Historical pre-provider skew rows should classify `relaunch` or `defer`.
  - There must be **zero** `relaunch` verdicts among non-skew failures. Investigate and
    fix any that appear.
- **Tests:** witness fixtures with temporary git repos (removal-type and addition-type
  symbols, with the culprit named), journal windows, and legacy log-tail parsing.

## Phase healer: The healer, at-most-once ledger, and auto-restart CLI

- **Beta flag.** Create `agent_auto_restart` with `sase flag new` (read the `sase_flags`
  memory first; never hand-add a registry member). It is epic scaffolding: with it off,
  no automatic path acts. Every automatic entry point checks both the flag and
  `agent_auto_restart.enabled`. Test both flag states.
- **Config block and getters** as specified above.
- **Ledger store**: `src/sase/agent/auto_restart/ledger.py`. IO uses the contracts
  above, and transitions go through the core binding.
- **Quiescence**: `quiescence.py`. Two checks:
  1. Probe the code-swap lock without blocking. If a writer holds it, defer.
     (`src/sase/dev_update/code_swap_lock.py`; see `code_swap_readers_active` and the
     writer lock.)
  2. Require at least `quiescence_seconds` since the last code change. That is the
     maximum of the lock-file mtime (the writer truncates it on release), the newest
     journal row, and each managed root's HEAD reflog or ref mtime.
- **W4 probe**: `probe.py`.
  - Run `[sys.executable, "-I", "-c", <script>]` with cwd `~` and a 20 s timeout. `-I`
    keeps a sase workspace checkout off `sys.path`.
  - Import every managed module on the failing frames plus the target module, then check
    any named `sase_core_rs` binding with `hasattr`.
  - Return `{ok, failures}`.
- **Healer**: `healer.py`. Steps, in order:
  1. Claim the ledger, or exit if another claim is live.
  2. Gather inputs and witnesses.
  3. Classify.
  4. Run the quiescence gate and the probe; on failure, mark `deferred`.
  5. Apply the skip rules.
  6. Apply the storm breaker.
  7. Write evidence.
  8. Plan the restart.
  9. Guard the wipe scope.
  10. Execute.
  11. Record the outcome.
  12. Notify.

  Write `recovery` on the failed row's `done.json` at each state.

- **Skip rules** (user intent wins). Before executing, re-validate against fresh state.
  - **Decline silently** (the user acted) when the row was dismissed, is no longer
    `FAILED`, or `find_named_agent(name)` now resolves to a different artifacts dir
    (mid-`,x` or a manual relaunch).
  - **Not candidates at all:** remote rows, and killed or stopped outcomes.
  - **Ask** when there is a pending question, plan, or gate marker.
  - **Decline loudly** (`already_restarted`) when the lineage already has a record, or
    the row carries `agent_meta.auto_restart`.
- **Storm breaker**, in `storm.py`. When launches would exceed `storm_max_per_episode`
  or `storm_max_per_30m` (counted from ledger records):
  - set `state.json` paused;
  - decline the remainder with `paused`;
  - send one storm escalation.

  `resume` clears the pause.

- **Relaunch**, through the provider-drain seam (`src/sase/agent/restart.py`):
  - **Live autonomy.** Add
    `plan_agent_restart(..., follow_live_autonomy: bool = False)`. When it is true,
    apply the live autonomy record to the rewritten prompt. Move
    `_rewrite_prompt_from_live_record` (`src/sase/axe/run_agent_retry_spawn.py`) to a
    shared helper and reuse it. `sase agent restart` keeps today's behavior.
  - **Wipe-scope guard.** Decline (`wipe_reaches_others`) if the wipe preview reaches
    anything other than the failed row's own records. The relevant code is
    `restart_needs_confirmation` / `_restart_preview.py`.
  - **Evidence.** Extend `prepare_recovery` (`src/sase/agent/_restart_recovery.py`) so
    the existing `~/.sase/restarts/<stamp>-<name>/` bundle also receives:
    `error_report.md`, `done.json`, `failure_facts.json`, `runner_log_tail.txt`,
    `verdict.json`, and `witnesses.json`. Record the bundle path in the ledger.
  - **Provenance.** Pass `SASE_AUTO_RESTART_PROVENANCE` (inline JSON) through the
    force-reuse plan's segment env. At bootstrap the runner consumes it into
    `agent_meta.auto_restart`. Add it to the refresh module's handoff lists and
    preserved metadata.
  - **Execute.** Call `execute_agent_restart(plan)` from cwd `~` through normal
    admission, holds, and capacity.
  - **Ordering.** `-p` processes dependencies before dependents (topological over
    `%wait` names), then least progress first. Correctness must not depend on that
    order, because waiters stay parked on a failed dependency.
- **Minimal notifications**, in `notify.py`, with three functions:
  - `publish_relaunch(record)`: a plain upsert with the episode sender and dedup key.
  - `publish_escalation(record, kind)`.
  - `resurface_failure(record, reason_text)`: makes the original failure notification
    loud, or rebuilds the standard failure notification from `done.json` and
    `error_report.md` if it is missing, and appends one explanatory note.

  Phase `episode-notify` upgrades the content; the call sites stay.

- **CLI.** `run`, `list`, `show`, and `resume` as specified. Add an operation name
  `AGENT_AUTO_RESTART` to `sase.ops.names`.
- **Tests:**
  - happy path: a pre-provider skew relaunches under the same name with forced reuse, a
    session member, and a bead-bound epic phase;
  - every skip rule;
  - crash windows: the healer dies after the claim, before launch, and after launch. A
    second pass adopts or settles, and never launches twice;
  - user races: a manual `,x` during preparation wins;
  - the storm breaker trips and `resume` re-arms;
  - live autonomy, both `A` off and `%auto:plan`;
  - the evidence bundle survives the wipe;
  - the wipe-scope guard;
  - name-based `%wait` dependents bind to the replacement.

## Phase trigger: Runner doorbell, scheduler job, and waiter safety

- **Runner doorbell**: stdlib-only and boot-imported.
  - Compute the enabled bit at boot from the flag and the config, held in run state.
  - When the bit is on and `facts_look_like_update_skew` is true, the failure path:
    1. sets `done.json` `recovery = {state: "pending", requested_at}`;
    2. atomically drops the doorbell file;
    3. sends the completion notification with `silent=True`.
  - The doorbell write happens **before** the notification, so a broken notification
    path cannot lose it.
  - No doorbell is dropped for user kills.
- **Scheduler job** `agent_auto_restart`, modeled on `orphan_agent_scope_reap`:
  - Script `src/sase/scripts/sase_chop_agent_auto_restart.py` with
    `@builtin_chop("agent_auto_restart")`.
  - Entry point `sase_job_agent_auto_restart` in `pyproject.toml`.
  - Added to the 10-second `waits` routine in `src/sase/default_config.yml` with an `fs`
    trigger on `agent_auto_restart/doorbell` and the code-swap lock file, and
    `max_quiet: "60s"`.
  - Each run:
    1. Enumerate the following, sharing the enumeration module with `run -p`:
       - doorbells;
       - recent failed rows not in the ledger and not dismissed (via the agent artifact
         scan facade within a bounded lookback, never a raw glob), including legacy and
         log-only deaths;
       - stale `claimed`/`launching` records;
       - `deferred` records;
       - `launched` records whose replacement has settled.
    2. If the feature is off or paused, re-surface pending failures loudly, clear their
       doorbells, and mark them `declined`. **Disabling the feature never swallows a
       failure.**
    3. Otherwise, when work exists, submit the durable proc
       `sase agent auto-restart run -p -j` with these settings:
       - label `↻ Auto-restart N agent(s)`, origin and operation `AGENT_AUTO_RESTART`;
       - `concurrency_keys=["agent-auto-restart"]`, cwd `~`, and a 30-minute timeout.
  - Idle ticks must stay at a handful of `stat()` calls. Emit a summary with a reason
    via `runtime.emit_summary`.
  - Settle `launched → settled_ok|settled_failed` from the replacement's outcome so the
    report's **Now** column stays live.
  - Re-surface any `pending` older than `pending_resurface_seconds` that has no ledger
    record.
- **Waiter safety:**
  - Name-based waits bind through forced reuse; verify this with tests.
  - Identity- or ref-based waits (`wait_for_artifacts`) pin the old artifacts dir, which
    the wipe removes. Add a forward lookup so these resolve to the replacement:
    - The ledger maps old artifacts timestamp to new artifacts dir.
    - Consult it wherever wait resolution currently ends in `target artifact is missing`
      (`src/sase/agent/wait_watch/`, `src/sase/axe/run_agent_wait_deps.py`).
    - Move it into core if that resolution lives there.
  - `wait_checks` (`src/sase/scripts/_chop_wait_checks_terminal.py::terminal_blockers`)
    must not report a dependency as a terminal blocker while its `recovery.state` is in
    flight.
- **Tests:**
  - the doorbell is written under an import-poisoned `sys.meta_path` (the runner must
    not import anything new on this path);
  - job enumeration, and the idle-tick cost;
  - feature-off and paused re-surfacing;
  - proc submission dedup;
  - stale-pending re-surfacing;
  - identity-wait forwarding;
  - wait_checks quiet while in flight.
  - Docs: add the job to the job table in `docs/axe.md`.

## Phase episode-notify: One upserted ↻ notification and live report per update episode

- Implement the notification, report, toast, and escalation spec above in
  `src/sase/agent/auto_restart/notify.py`.
  - Use `upsert_notification` (`src/sase/notifications/store.py`) for create and +1,
    with one +1 note per agent.
  - The upsert path is modeled on `src/sase/service/notifications.py`'s guarded upsert.
- **Title refresh.** A +1 deliberately changes neither notes nor delivery cursors
  (`docs/notifications.md`). Refresh `notes` and the inline report snapshot in place
  through `reconcile_notification_rows`.
  - First verify that it owns those fields and does not resurface or bump the activity
    cursor.
  - If it cannot do that, add a narrow `refresh_content` option to the core notification
    store, in this phase, with a pin bump.
- **Live report.** Write `episodes/<slug>.report.json` atomically under a file lock,
  validated with `validate_chop_report`, re-rendered from ledger records on every ledger
  change and every job settlement.
- **Dismissed episodes.** If the episode row was dismissed, create `<episode>#N` instead
  of adding a +1.
- **Escalations.** Post-provider and declined failures re-surface the agent's own
  failure notification (sender `user-agent`, `ViewErrorReport`, so it lands in the
  Errors bucket) with the escalation sentence as an added note. The storm breaker uses
  the `agent.auto-restart` sender at error severity.
- **Toast.** Verify that `_format_notification_toast`
  (`src/sase/ace/tui/actions/agents/_toasts.py`) renders the `ViewReport` episode toast
  at information severity with the `↻` title intact. Add the sender's action label to
  `format_batch_toasts` if it is needed.
- **Tests:**
  - five agents in one episode give one row, four +1s, one toast, and refreshed notes
    with no cursor movement;
  - the live report changes as replacements settle;
  - the dismissed-episode rollover;
  - each escalation's copy and severity;
  - `notes[0]` never contains "fail" or "error".
  - Docs: add the sender to `docs/notifications.md`.

## Phase ux-surfaces: Agents-tab ↻ RESTARTING state, provenance line, help, and update hint

Read the `tui` and `tui_perf` memories first. Render paths never stat or glob; recovery
data arrives through the existing done-wire loaders.

- **Status.**
  - Map in-flight `recovery.state` to a new `RESTARTING` status in the done-snapshot
    loaders (`src/sase/ace/tui/models/_loaders/_done_snapshot_loaders.py`,
    `_agent_status_apply.py`).
  - Register it everywhere a live status must be known:
    - `ACTIVE_AGENT_STATUSES` (`src/sase/agent/status_buckets.py`)
    - `AGENT_LIVE_STATUS_VALUES`
    - `src/sase/ace/tui/widgets/_agent_detail_helpers.py`
    - `src/sase/ace/tui/actions/event_refresh/_constants.py`
    - `_ACTIVE_LEAF_STATUSES`
  - Clan aggregation treats it like a running member, not a failed one.
- **Row rendering** (`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py`):
  - bold `#FFAF5F` `↻ RESTARTING` plus a dim hint;
  - the stale-pending fallback and the declined reasons as dim hints on `FAILED`;
  - add the new fields to `agent_render_key`.
  - Add `UPDATE_RECOVERY_GLYPH = "↻"` next to the accents in
    `src/sase/ace/tui/widgets/update_accents.py` and use it everywhere.
- **Replacement.**
  - Add a `↻` chip on the row prefix, styled like the existing retry badge.
  - Add the provenance block to the identity header
    (`src/sase/ace/tui/widgets/prompt_panel/_identity_header_compact.py` and the
    expanded metadata sections), sourced from `agent_meta.auto_restart`.
  - Register the preserved `error_report.md` and the bundle directory as `v` file hints.
- **Help.** Add `↻ RESTARTING` to the Agents reference legend and the row-glyph section
  (`src/sase/ace/tui/modals/help_modal/agents_reference_sections.py`). Descriptions are
  at most 32 characters; box widths per `src/sase/ace/CLAUDE.md`.
- **The `sase update` hint**, in the CLI and the TUI modal, shown only when the feature
  is enabled.
- **Visual snapshots** for:
  - a RESTARTING row;
  - a deferred row;
  - a declined row;
  - the replacement's provenance header.

  Run `just fix-tui-screenshots` with selectors and inspect every golden.

- **Perf.** No new per-keypress disk reads. Prove with the `SASE_TUI_PERF=1` j/k
  measurement on an Agents tab containing these rows.

## Phase land: Remove the beta flag, document, and replay the incident end to end

- **Remove the flag.** Delete the `agent_auto_restart` flag's Off branches, make the On
  branches unconditional, remove the registry entry, and close the flag bead in the same
  change. The config switch remains the permanent kill switch.
- **Acceptance test.** Replay the incident end to end with fakes for provider and
  launch:
  1. A runner boots with identity A and parks on `%wait`.
  2. A simulated update to identity B removes a symbol on a deferred path.
  3. The runner fails pre-provider.
  4. Expect the doorbell, then the job, then the healer proc, then a relaunch under the
     same name with provenance.
  5. Expect one episode notification and one toast, then a live report whose **Now**
     column settles.
  6. A second failure of the replacement is declined loudly as `already_restarted`.
- **Documentation.** Write `docs/agent_auto_restart.md` covering:
  - the mental model;
  - the signature, witness, and probe rules;
  - phases and modes;
  - the ledger and the at-most-once guarantee;
  - the CLI, config, and notification anatomy;
  - troubleshooting.

  Link it from `docs/axe.md` and `docs/notifications.md`.

- Run `sase agent auto-restart scan -s 120d` one final time and record the result.
- **Follow-ups.** Record each of these as a `PROPOSED FOLLOW-UP:` note on the epic's
  bead:
  - P1, a CI import-recording test that forbids post-provider first imports and turns
    `preload_post_gate_modules` into a test-enforced invariant;
  - P2, a release-pinning epic: immutable per-process releases;
  - post-provider resume-in-place mode;
  - sunsetting the `llm_provider.retry.sase` entry into this feature;
  - `%auto` parity for manual `,x` and `sase agent restart`.

> [!decision] decision_record When accepted, the land phase adds a `decisions` web
> record (for example `decisions:update-skew-auto-restart`) stating the claim:
> update-skew recovery relaunches at most once per lineage, only pre-provider, gated on
> signature ∧ witness ∧ probe, out of process. It also records why the alternatives were
> rejected, the cost, and the reopen condition (release pinning makes Tier 1–2 skew
> impossible). Use the `/sase_memory_write` skill, then run `sase memory init`. When
> declined, the land phase records it as a `PROPOSED FOLLOW-UP:` note instead.

## Verification

- Every phase runs `sase tool run check` in this repo. Phase `core-verdict`, and phase
  `episode-notify` if it touches core, also run `sase tool run check` in the sase-core
  checkout.
- Phase `ux-surfaces` runs targeted `just fix-tui-screenshots`.
- No phase runs `just check-full` unless explicitly instructed.
- **Overall success criteria:**
  - Replaying the 2026-10-09 incident brings the agents back under their own names
    within about one minute of the update settling, with one notification and one
    information toast, and no red toasts.
  - The scan shows zero `relaunch` verdicts among non-skew failures.
  - No path silently swallows a failure.
