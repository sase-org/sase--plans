---
tier: tale
title: Harden gate_shell_reclaim against its two-minute timeout
goal:
  One gate_shell_reclaim pass reads the artifact index at most twice, finishes inside a
  pass-wide budget well under the 120s chop timeout, and leaves progress output in its
  run log even when killed.
size: medium
proposed_by: bbugyi200.athena.0kd
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0kd](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0kd.md)
- **COMMITS:**
  - [817c5c5](https://github.com/sase-org/sase/commit/817c5c5679154fffe7892e7124aa496782d9b8a7)
    — fix(gate_shell): harden reclaim timeout handling with shared snapshot and
    pass-wide deadline

# Harden the `gate_shell_reclaim` chop so its two-minute timeout cannot return

## Diagnosis (already confirmed; do not re-investigate)

**Symptom.** The axe error digests report
`housekeeping / gate_shell_reclaim: timed out after 120s`, exit code `-9`, and
`Subprocess Output: [output absent]`. Every retained run from 09:24 to 14:11 on
2026-09-12 timed out at about 120.1s. That makes 23 timeout digests in total.

**Root cause.**

- `reconcile_incomplete_gate_handoffs()` called `collect_successor_evidence()` for every
  settled gate. Each call did a full agent-artifact-index read through
  `sase.gate_shell.handoff._family_records()`.
- The index now holds about 10.9k records, so one full read takes about 6.5s: roughly
  4.5s in the Rust `query_agent_artifact_index` and 1.5s decoding the wire in Python.
- With up to 32 gates per project, one pass needed well over 120s.
- axe SIGKILLed the chop before it saved its reconcile cursor. Every tick therefore
  re-diagnosed the same first gates and never made progress: a permanent wedge.

**Already landed.** Commit `335d71223` ("fix(gate-shell): read the artifact index once
per handoff reconcile pass") fixed the per-gate reads:

- The pass shares one `GateShellSnapshot`.
- It saves the cursor after every gate.
- It defers gates past a 60s budget.

Since axe restarted, the 14:49 and 15:49 runs both succeeded in about 26s, and the 15:49
digest has no `gate_shell_reclaim` entry.

**Remaining gaps.** These can bring the timeout back as the index and gate volume grow.
This plan closes them.

1. **Two full index reads per pass.** The chop still reads the index twice, a fixed cost
   of about 13–14s that grows linearly with the index:
   - `reclaim_pending_gate_shells()` calls `list_gate_shells()`, which reads the full
     index.
   - `reconcile_incomplete_gate_handoffs()` then calls `load_gate_shell_snapshot()`,
     which reads it again.
2. **The budget does not cover the whole pass.**
   - The 60s reconcile budget starts only after the reclaim phase and the snapshot read,
     so neither counts against it.
   - The budget is checked only between gates.
3. **Changed gates still cost a full read each, with no bound.**
   - A settled gate whose `agent_meta.json` changed after the snapshot (within
     `_SNAPSHOT_MTIME_SLACK_SECONDS`) still falls back to `_family_records()`.
   - That is another full ~6.5s read per such gate, with no limit on how many a pass
     does. Busy hours make this common.
   - One pass can blow through the budget by several full reads, because the budget
     check runs before each gate rather than before the expensive read.
4. **A timeout loses all output.**
   - `ChopLogger` (`src/sase/chops/sdk.py`) `print()`s without flushing.
   - axe runs the chop with its stdout on a pipe (`src/sase/axe/chop_script_runner.py`),
     so child Python stdout is block-buffered.
   - The timeout SIGKILL drops the buffer, which is why every timeout digest says
     `[output absent]`. The next regression would be just as undiagnosable, and the same
     blind spot affects every other Python chop that times out.
5. **Backlog.** The `gh_sase-org__sase` project still has about 151 settled gates ahead
   of its reconcile cursor. Full 32-gate batches will run for several more hourly ticks,
   so the pass must hold up under a full batch.

## Goal

Make one `gate_shell_reclaim` pass:

- read the artifact index **once** in the common case and **at most twice** in the worst
  case;
- finish **inside a single wall-clock budget measured from chop start**, well under the
  120s timeout;
- leave durable progress lines in the run log even if it is killed.

Do not change the chop's structured result contract: the counter keys, status and reason
semantics, and summary-line format all stay as they are.

## Changes

### 1. Share one snapshot across both phases (`src/sase/gate_shell/reclaim.py`, `src/sase/scripts/sase_chop_gate_shell_reclaim.py`)

- Add an optional keyword `snapshot: GateShellSnapshot | None = None` to both
  `reclaim_pending_gate_shells()` and `reconcile_incomplete_gate_handoffs()`.
  - When it is given, iterate `snapshot.gate_shells` instead of calling
    `list_gate_shells()` or `load_gate_shell_snapshot()`.
  - When it is `None`, keep today's behavior, so other callers and existing tests are
    unaffected.
- In the chop script `_run`:
  - take `started = time.monotonic()`;
  - call `load_gate_shell_snapshot()` once and time it;
  - pass the same snapshot to both phases.
- Why a stale snapshot is safe for the reclaim phase:
  - `settle_gate_shell()` re-reads `agent_meta.json` under the per-gate follow-up lock
    and already handles a shell that became terminal concurrently
    (`settle_already_terminal_handoff`).
  - `_reclaim_one()` inspects the gate bundle files live.
  - So the snapshot is exactly as fresh as the `list_gate_shells()` call it replaces.
- Gates the reclaim phase settles during this pass are still non-terminal in the
  snapshot, so reconcile skips them this pass. That is intended:
  - Settlement performs the handoff itself; reconcile is only the safety net.
  - It also avoids the full re-read that a just-settled gate would otherwise trigger
    through the mtime check.

### 2. One pass-wide deadline (`reclaim.py`, chop script)

- Replace the reconcile-only budget with a budget measured from chop start.
  - Add a module constant, e.g. `_PASS_TIME_BUDGET_SECONDS = 75.0`, with a comment
    explaining how it relates to the chop's `timeout: "2m"` in
    `src/sase/default_config.yml`.
  - Worst case is the budget, plus one in-flight gate's classify and persist, plus a 5s
    follow-up lock wait. That stays well under 120s.
- Add an optional `deadline: float | None = None` (same clock as `clock`) to
  `reconcile_incomplete_gate_handoffs()`.
  - When given, it replaces `clock() + time_budget_seconds`.
  - Keep `time_budget_seconds` and `clock` so the existing budget test keeps its shape.
- The chop computes `deadline = started + _PASS_TIME_BUDGET_SECONDS` and passes it, so
  the snapshot read and the reclaim phase count against the budget.
- The reclaim phase itself stays unbudgeted: it only touches pending gates (a handful),
  and deferring settlement would add a new counter to the result contract.

### 3. Bound the changed-gate path to one snapshot refresh (`reclaim.py`)

Replace the per-gate full re-query:

- Today: `_snapshot_family_records()` returns `None`, and then
  `collect_successor_evidence()` calls `_family_records()`, a full index read.
- New rule: at most **one** snapshot refresh per pass.

When `_diagnose_one` finds the gate's metadata changed after the current snapshot:

- **If no refresh has happened yet this pass, and the deadline leaves room for another
  read:**
  - Reload the snapshot once with `load_gate_shell_snapshot(project=...)`.
  - "Room" means `clock() + <duration of the initial snapshot read> < deadline`. Use the
    measured duration if the chop passed it, otherwise a conservative default.
  - Use the refreshed snapshot for this gate and for the rest of the pass.
  - Refresh only the family-member lookup source. Do not rebuild the reconcile batch
    list or re-sort the cursor order mid-pass.
- **Otherwise:**
  - Do not query the index.
  - Stop that project's batch at this gate without advancing the cursor past it, and
    count it plus the rest of the project's batch as `deferred`.
  - The next tick's fresh snapshot will postdate the change.
  - Never classify a changed gate against a snapshot older than its change: that can
    persist a false `gate_followup_error` through `apply_decision`.
- Keep the check inside the per-gate follow-up lock, and pass the refreshed family
  records explicitly as `family_records=`, so reconcile never falls back to
  `_family_records()`.
  - Leave `collect_successor_evidence()`'s `None` fallback in place for its other
    callers (settlement and resume).
- Implementation shape is the implementer's choice, e.g. a small mutable pass-state
  object holding `snapshot`, `refreshed`, and `read_seconds`, passed through
  `_reconcile_project` and `_diagnose_one`.
- Mind the 1.0s mtime slack. A gate modified within one second before the refresh still
  counts as changed, so tests need to control the snapshot's `taken_at` (monkeypatch the
  wall clock `load_gate_shell_snapshot` uses, or set `agent_meta.json` mtimes
  explicitly) rather than relying on real elapsed time.

### 4. Make chop output survive a timeout kill (`src/sase/axe/chop_script_runner.py`, chop script)

- In `_compose_chop_subprocess_env()`, default `PYTHONUNBUFFERED=1` with `setdefault` on
  the copied environ, before `extras` are applied.
  - An explicit value from the ambient env or from `extras` still wins.
  - This makes every Python chop's stdout reach the run log as it is written, so a
    killed run keeps its partial output and the digest's `Subprocess Output` excerpt
    stops being empty.
  - Update any test that asserts the exact composed environment.
- In the `gate_shell_reclaim` chop, log short progress lines with `runtime.log.info` at
  each phase boundary:
  - snapshot read: seconds, record count, gate-shell count;
  - reclaim phase done: seconds;
  - snapshot refreshed: which gate, seconds;
  - reconcile done: seconds.
- Progress lines must **not** parse as a summary line. `parse_summary` accepts
  `name: key=value ...` only when the prefix has no whitespace. Use a prefix with
  spaces, e.g. `gate shell reclaim progress: ...`, or a body that is not all `key=value`
  tokens.
  - Confirm `ChopLogger` does not treat them as summaries: `last_summary` must still
    return the real summary.
  - Confirm the result counters stay exactly the current 12 keys.

### 5. Documentation

- Update the `gate_shell_reclaim` `description` in `src/sase/default_config.yml`. It
  should say the pass reads the artifact index once, refreshes it at most once for gates
  that changed mid-pass, and defers remaining gates once a pass-wide budget (measured
  from chop start) is spent.
- Keep the existing `timeout: "2m"`. Do **not** raise the timeout: that would hide the
  problem instead of bounding it.

## Tests

Extend `tests/gate_shell/test_reclaim.py`, reusing its `_serve_index`,
`_serve_family_query`, `_stub_decisions`, and `_settled_gate` helpers, and
`tests/test_axe_chop_output_contract.py`:

- **One read for the whole chop.**
  - Run the chop's `_run` path, or both phase functions with a shared snapshot, against
    served pending and settled gates.
  - Assert `_project_records` is read exactly once, and `handoff._family_records` is
    never called.
- **Refresh at most once.**
  - With two or more settled gates changed after the initial snapshot, assert exactly
    two index reads in total (the initial read plus one refresh) and zero
    `_family_records` calls.
  - Assert a successor present only in the refreshed index is seen as `attached_agent`.
- **Changed after the refresh, or no budget left.**
  - Assert the gate is deferred, is not classified, and the cursor stays before it.
  - Assert the next pass, with a newer snapshot, diagnoses it.
- **Pass-wide deadline.**
  - With a fake clock where the snapshot read consumes most of the budget, assert
    reconcile defers instead of scanning.
  - Assert a refresh is skipped when `clock() + read_seconds >= deadline`.
- **Update the existing test.** Change
  `test_reconcile_requeries_evidence_for_a_gate_changed_after_the_snapshot` to the
  refresh semantics: the evidence comes from the refreshed snapshot, not from a
  `_family_records` query.
- **Chop contract tests.** The four `test_gate_shell_reclaim_*` tests monkeypatch the
  two phase functions with zero-argument lambdas. Update the stubs to accept the new
  keyword arguments, and stub `load_gate_shell_snapshot` so they never read a real
  index. Keep all existing counter and reason assertions unchanged.
- **Env composer.** In `tests/test_axe_chop_script_runner.py`, which already covers
  `_compose_chop_subprocess_env`, add assertions that `PYTHONUNBUFFERED` defaults to `1`
  and that an ambient or `extras` value overrides it. Update any existing assertion
  there that compares the exact composed environment.

## Verification

- Run `just install` if the workspace venv is stale, then `just check`.
- Also run
  `pytest tests/gate_shell tests/test_axe_chop_output_contract.py tests/test_axe_chop_script_runner.py`
  directly.
- Read-only sanity check (it must not settle gates or write cursors): in the repo venv,
  time `sase.gate_shell.store.load_gate_shell_snapshot()` once to confirm the
  single-read cost the budget assumes (about 6–7s today).
- Do not run the real chop against the live `~/.sase` state from the implementation
  workspace: it mutates gate metadata and cursors. The next scheduled axe tick exercises
  it. Success means the housekeeping run history shows `status: success` with the
  progress lines present in the run log.

## Out of scope — follow-up to file with `/sase_new_task` if confirmed

- **Reconcile cursor coverage gap.**
  - The cursor orders gates by creation `(timestamp, artifacts_dir)`.
  - A gate that stays pending (e.g. a plan approval waiting hours) while newer gates
    settle will be behind the cursor by the time it settles, so the handoff safety net
    never diagnoses it.
  - It is not biting today: no pending gate is older than any project's cursor, because
    the `sase` cursor is still draining a backlog. It will bite once the backlog clears.
  - Fixing it means keying the cursor on settlement order or re-scanning recently
    settled gates. That changes reconcile semantics, so it belongs in its own task.
- **Cheaper family lookups in the Rust core.**
  - The index query's `candidate_filter` supports only project, cl, model, provider, and
    type; there is no agent family filter.
  - A family or gate-shell pushdown in `sase-core` (`query_agent_artifact_index`) would
    make both the snapshot and any refresh cheap.
  - That is a cross-repo core change and is not needed to bound this chop.
