---
tier: tale
title:
  Fix the host_completion_reasons wire-key regression that parks every waiting agent
goal:
  Agents waiting on finished dependencies release again even when a host-completion
  recovery monitor recorded plural reasons, and apollo's parked 9m--1 resumes.
size: small
decisions:
  apollo_unblock:
    ask:
      Should the implementing agent unblock apollo's parked waiters through a /sase_gate
      after the fix is verified?
    choices:
      gate:
        Propose the apollo marker scrub (plus any kill) as a gate you confirm; 9m--1
        resumes on approval
      manual: Leave apollo alone; you run the unblock steps listed in this plan yourself
    default: gate
    why:
      the fix can't reach runners that already imported the broken module; the marker
      must be cleared
    answer: gate
  stale_a013_followup:
    ask: Should bob-cli-a0.13--1 be stopped before apollo is unblocked?
    choices:
      kill: Stop it first; 9m--1 already owns bob-cli-a0.13's reader sweep and closeout
      release:
        Let it resume alongside 9m--1; its 16:04Z reader-launch follow-up will likely
        duplicate 9m's work
    default: kill
    why:
      9m's own next-action prompt treats bob-cli-a0.13--1 as stranded and finishes its
      work itself
    answer: kill
proposed_by: bbugyi200.athena.12m
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-20 15:21:40
status: wip
---

# Fix the `host_completion_reasons` wire-key regression that parks every waiting agent

## Why `9m` is waiting (diagnosis)

On apollo, the `9m` agent session's current member is `9m--1`. It is the follow-up turn
launched when the `9m--mon` monitor ("docs readers") finished at 18:47Z. Its prompt
starts with `#fork:9m--code`, so it registers a wait on `9m--code`. `9m--code` finished
at 18:32Z (`done.json` outcome `completed`), so the wait should have released right
away. Instead, the runner log
(`~/.sase/workflows/202610/gh_bobs-org__bob-cli_ace-run-261020_144736.txt`) repeats:

```text
Wait dependency check failed (index): TypeError: AgentSessionTurnWire.__init__() got an unexpected keyword argument 'host_completion_reasons'; staying parked
```

The same error has parked `bob-cli-a0.13--1` on apollo since 16:04Z (its log is
`gh_bobs-org__bob-cli_ace-run-261020_120434.txt`). `9m` already calls this "the stranded
bob-cli-a0.13--1 launcher defect" in its own next-action prompt. It is the same bug.

Reproduced on apollo with the full traceback:

```text
stored_tribe_names_for_resolution()                     # sase/core/agent_tribe_evidence.py
  -> _build_stored_tribe_names -> WaitDependencyIndex.add_many
  -> done_outcome_from_data -> effective_done_outcome -> _effective_shell_outcome
  -> agent_session_turn_from_mapping(done_data)
  -> _agent_session_turn_from_flat_keys -> AgentSessionTurnWire(**...)  # TypeError
```

The same TypeError reproduces locally on master (`682e52fc71`, which is also the commit
apollo runs) with a single call:
`agent_session_turn_from_mapping({"monitor_id": "x", "monitor_host_completion_reasons": ["a"]})`.

### Root cause

Commit `e77280c873` ("feat(monitor): show host-completion recovery reasons in follow-up
prompts", bead `sase-1n8.3`) did two things:

- It started persisting a plural `monitor_host_completion_reasons` list. The list goes
  into `agent_meta.json` through `host_completion_state.record_status(reasons=...)` and
  is copied into `done.json` by `supervise._finish_monitor`, `reconcile`, and
  `proc_adapter.settle_monitor_followup`.
- It added `"monitor_host_completion_reasons": "host_completion_reasons"` to
  `_MONITOR_SHARED_KEYS` in `src/sase/core/agent_scan_wire_agent_session_turn.py`, but
  never added a `host_completion_reasons` field to the frozen `AgentSessionTurnWire`
  dataclass.

`_agent_session_turn_from_flat_keys` splats every projected key into the constructor. So
**any** flat on-disk marker that carries the plural key raises `TypeError`. Only
monitors that go through host-completion recovery write the key. The first one on apollo
was the sase project's `99--mon-1` (artifact dir
`gh_sase-org__sase/artifacts/ace-run/202610/20/20261020105054`, written 15:47Z). It is
the only marker on apollo that carries the key. Athena has none yet.

### Blast radius

`stored_tribe_names_for_resolution()` scans `agent_meta.json` / `done.json` across
**every project** on the machine. One poisoned marker therefore makes
`_build_runner_wait_index` raise for every waiting runner, in every project.
`run_agent_wait_deps._resolve_marker_release` deliberately fails closed ("staying
parked"), so every named-agent waiter on the machine stays parked forever. The failure
only shows up in each runner's output log; the TUI shows an ordinary WAITING row.

- `95.w1` and `bob-cli-a0.land` are currently waiting for legitimate reasons (an epic
  follow and open beads). They will wedge the same way once their dependencies are
  satisfied.
- The same projection is also used by `gate_turn/store`, `monitor/store`,
  `dismissed_agent_completion`, `wait_dependency_resolution/_artifact_state`,
  `agent_scan_wire_conversion`, and `_agent_list_entry_build`. Fixing the projection
  fixes all of them.

Dry-run check on apollo: with the bad mapping removed in-process, both `9m--1` and
`bob-cli-a0.13--1` resolve as `releasable=True`.

## The fix: drop the dead mapping, don't add a field

Remove the `"monitor_host_completion_reasons": "host_completion_reasons"` entry from
`_MONITOR_SHARED_KEYS`. Do not add a `host_completion_reasons` field to
`AgentSessionTurnWire`, for three reasons:

- Nothing reads `AgentSessionTurnWire.host_completion_reasons`. The only consumer of the
  plural list is `sase.monitor.followup.launch_followup_agent`, which reads
  `meta["monitor_host_completion_reasons"]` straight from the raw meta dict.
- The Rust wire (`sase_core` `agent_scan/wire.rs` / `scanner.rs`) mirrors exactly
  `host_completion_status` / `_message` / `_reason`, with no plural field. Removing the
  mapping restores Python/Rust parity. Adding the field would need a Rust wire change, a
  binding change, and a `sase-core-revision.txt` pin bump, all for a field with no
  reader.
- This exactly restores the projection as it was before `e77280c873`. The parts of that
  commit that work are untouched.

### Previous fixes that must keep working

- **`e77280c873` / `sase-1n8.3` (recovery reasons in follow-up prompts).** Keep every
  writer of `monitor_host_completion_reasons` (`host_completion_state.record_status`,
  the done-marker copy loops in `supervise.py`, `reconcile.py`, and `proc_adapter.py`)
  and the reader in `monitor/followup.py`. Do not change them.
  `tests/monitor/test_monitor_host_completion_recovery_prompt.py` must still pass
  unchanged, including `test_recover_persists_full_reasons_in_receipt_and_meta`.
- **The wait lane's fail-closed behavior** (`sase-1hf` wait-lane repair `c3f9d2915c`,
  the scoped runner wait indexes in `814dbcafb4` / `fe77e92d56` / `d289b664a6`).
  `run_agent_wait_deps.py` must keep staying parked on index or confirmation errors.
  Releasing on a broken index could release a waiter whose dependency is still running.
  Do not edit `run_agent_wait_deps.py`, `agent_tribe_evidence.py`, or the wait index to
  swallow per-row errors in this tale; that resilience question is the follow-up bead
  below.
- **The wire-mirror cutover** (`tests/test_agent_session_wire_mirrors.py`,
  `tests/test_python_turn_wire_cutover.py`). The dataclass shape does not change, so
  these must pass untouched.

## Changes

### 1. `src/sase/core/agent_scan_wire_agent_session_turn.py`

- Delete the `"monitor_host_completion_reasons": "host_completion_reasons",` line from
  `_MONITOR_SHARED_KEYS`.
- Add one short comment above the `_MONITOR_SHARED_KEYS` dict (match the module's
  comment style). It should say that every target must be an `AgentSessionTurnWire`
  field, and that `monitor_host_completion_reasons` is deliberately not projected
  because `sase.monitor.followup` reads it from raw meta and the Rust wire has no such
  field. This keeps the line from being re-added.
- No other source changes.

### 2. New test module `tests/test_agent_session_turn_flat_key_map.py`

Keep it small and follow the style of `tests/test_agent_tribe_evidence_cache.py`.

1. **Structural guard (prevents this whole class of bug).** For each of the four maps,
   assert every target name is a field of the right dataclass:
   - `_MONITOR_SHARED_KEYS` and `_GATE_SHARED_KEYS` →
     `AgentSessionTurnWire.__dataclass_fields__`, minus the `kind` / `monitor` / `gate`
     structural fields.
   - `_MONITOR_SPECIFIC_KEYS` → `AgentSessionTurnMonitorWire`.
   - `_GATE_SPECIFIC_KEYS` → `AgentSessionTurnGateWire`.

   Use `dataclasses.fields()`. A parametrized test is fine. These private maps are
   test-only imports. Check the Symvision rules (`sase memory read symvision.md`) if the
   private-import lint complains, and prefer the repo's established pattern for tests
   that import module privates.

2. **Flat-marker regression.** Call `agent_session_turn_from_mapping` on a flat
   recovery-monitor `done.json`-shaped dict, modeled on apollo's `99--mon-1`:
   `monitor_id`, `monitor_state: "failed"`,
   `monitor_host_completion_status: "recovery"`,
   `monitor_host_completion_reason: "no_new_verdict_insufficient:new_failures"`,
   `monitor_host_completion_reasons: ["no_new_verdict_insufficient:new_failures"]`,
   `monitor_followup_outcome: "launched"`, and `outcome: "monitored"`. Assert it returns
   a `kind == "monitor"` turn with `host_completion_status` and `host_completion_reason`
   preserved, and that it does not raise. Also assert that
   `sase.core.dismissed_agent_completion.effective_done_outcome` on the same dict does
   not raise.
3. **Machine-wide wedge regression.** Under `tmp_path`, build a projects root with one
   project containing one `ace-run` artifact dir. Give it an `agent_meta.json` and a
   `done.json` that both carry the plural key; use the layout `iter_agent_artifact_dirs`
   expects, as the existing wait-index tests and fixtures do. Call
   `agent_tribe_evidence.stored_tribe_names_for_resolution(projects_root=..., agent_tribes_path=..., legacy_store_path=...)`
   and assert it returns without raising. Clear `evidence_mod._CACHE` before and after,
   as in `tests/test_agent_tribe_evidence_cache.py`. Optionally, also build a scoped
   runner wait index for a waiter that names a completed agent in the same tmp project,
   and assert it resolves as releasable. Only add this if an existing fixture makes it
   cheap; otherwise the tribe-scan assertion is enough, because that call is the one
   that wedged apollo.

Confirm that tests 2 and 3 fail on the current code (TypeError) before the source edit,
and pass after it.

### 3. Verification

- Run the new module plus the neighbors directly:
  `tests/test_agent_session_turn_flat_key_map.py`,
  `tests/monitor/test_monitor_host_completion_recovery_prompt.py`,
  `tests/test_agent_session_wire_mirrors.py`, `tests/test_python_turn_wire_cutover.py`,
  and `tests/test_agent_tribe_evidence_cache.py`.
- Then run `sase tool run check`, per `lint_and_test.md`. Do not run `check-full`.

### 4. Follow-up bead (file it, do not implement it)

Use `/sase_new_task` to file a `bug` task bead (default size `large`) for wait-check
failures being invisible and machine-wide:

- One malformed or unparseable marker in **any** project wedges every named-agent waiter
  on the machine through the global stored-tribe scan.
- The only signal is a repeated "staying parked" line in each runner's output log. The
  waiting marker, `sase agent show`, and the TUI all show an ordinary WAITING row.

Candidate directions for whoever picks the bead up (they decide):

- persist the latest wait-check error into the waiting marker and surface it in the
  TUI/agent detail, and/or
- isolate per-row parse failures in the _tribe-name_ scan only, without weakening the
  fail-closed dependency-index behavior.

Cite this incident: `9m--1` and `bob-cli-a0.13--1` on apollo, poisoned marker from
`99--mon-1`.

## Unblocking apollo

The parked runner processes (`9m--1`, pid 559134; `bob-cli-a0.13--1`, pid 162582)
imported the broken module, so landing this fix and updating apollo's sase checkout will
**not** release them. `sase agent restart 9m--1` is not a good substitute: its dry run
says the stored monitor-follow-up prompt has no VCS tag and would relaunch it as a
**home** agent, losing the bob-cli project and workspace.

The least disruptive unblock is to remove the plural key from the single poisoned marker
pair on apollo. Each runner's next wait tick then re-resolves.

1. **Stop the stale follow-up first.**

   > [!decision] stale_a013_followup = kill Run
   > `ssh apollo 'sase agent kill bob-cli-a0.13--1'` and confirm it is gone before the
   > scrub. Once the scrub lands, its wait resolves releasable just like `9m--1`'s. It
   > would then resume its 16:04Z "10 reader slots approved" follow-up, duplicating the
   > reader sweep and `bob-cli-a0.13` closeout that `9m--1` already owns.

   > [!decision] stale_a013_followup = release Skip the kill; both waiters release on
   > the next tick after the scrub. Call out the duplicate-work risk in the gate or
   > summary.

2. **Scrub the poisoned marker on apollo.** In
   `~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202610/20/20261020105054/`,
   edit `agent_meta.json` and `done.json`:
   - Back up both files first (`*.bak-host-completion-reasons`).
   - Remove only the `monitor_host_completion_reasons` key. Leave everything else
     untouched, including `monitor_host_completion_reason`, which holds the same single
     value.
   - Rewrite atomically (temp file + rename), keeping the file mode (`agent_meta.json`
     is 0600) and the existing 2-space JSON formatting.
   - Make the step idempotent: skip if the key is already absent.

   This loses nothing. `99--mon-1`'s follow-up `99--3` already launched and consumed the
   reasons, and the singular reason field keeps the value.

3. **Confirm the release.** Within a wait tick or two, `9m--1`'s runner log moves past
   "Waiting for agents: 9m--code", and `ssh apollo 'sase agent show 9m--1'` shows it
   running its turn. Then grep both machines' `~/.sase` for
   `monitor_host_completion_reasons` in `agent_meta.json` / `done.json` to confirm no
   other poisoned marker exists.
4. **Deploy the code fix.** Getting the fix onto apollo and athena (their sase checkouts
   or installs) is a human action per `decisions:global-install-is-human-only`. Until
   then, any new host-completion recovery monitor re-poisons the machine. Mention this
   in the final summary.

> [!decision] apollo_unblock = gate After verification passes and the follow-up bead is
> filed, make a `/sase_gate` the last action of the turn. Follow that skill's
> instructions for final-declaration and commit ordering. The gate's approve option runs
> steps 1–3 over `ssh apollo`: the kill only when `stale_a013_followup = kill`, then the
> scrub as one small, idempotent Python script that backs up and rewrites atomically,
> then a read-only status check. A decline option leaves apollo untouched. The gate
> description must quote the poisoned path, the key being removed, and why that is safe.

> [!decision] apollo_unblock = manual Do not touch apollo. Finish with a summary that
> repeats steps 1–4 as copy-pasteable commands for the user.
