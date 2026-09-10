---
tier: tale
title: Resolve waits across settled gate-shell family members
goal:
  A settled gate shell no longer blocks its family's wait aggregate, so an agent waiting
  on a family that raised a gate starts as soon as that family finishes.
size: medium
proposed_by: bbugyi200.athena.0f3
create_time: 2026-09-09 20:00:07
status: wip
---

# Plan: Resolve waits across settled gate-shell family members

## Problem

`bob-cli-1n.5` stayed in `WAITING` long after every dependency it named had finished,
and had to be started by hand. It waited on agents `bob-cli-1n.1`, `bob-cli-1n.3`,
`bob-cli-1n.4` and on beads with the same three IDs. All three beads were closed and all
three agents were done, yet the wait never released. The same defect currently strands
`bob-cli-1n.land`, which will never resolve `bob-cli-1n.1` no matter how long it waits.

This is not a chop-cadence or bead-sync delay. It is a permanent, deterministic block:
any family that ever raised a gate shell can never satisfy a `%wait` on its name again.

## Root cause

A gate shell settles by writing a done marker with `outcome: "gated"` plus a
`gate_state` discriminator (`src/sase/gate_shell/settlement.py:198-210`). Wait
resolution never translates that outcome:

- `effective_done_outcome` (`src/sase/core/dismissed_agent_completion.py:98`) translates
  only the monitor shell's `outcome: "monitored"` into `completed` / `failed` via
  `monitor_state`. Every other outcome string is returned verbatim, so `"gated"` reaches
  the classifiers unchanged.
- `"gated"` is absent from `WAIT_SUCCESS_OUTCOMES`, `FAILURE_OUTCOMES`, and
  `KNOWN_DONE_OUTCOMES` (`src/sase/core/dismissed_agent_completion.py:26-38`).
- Therefore `artifact_is_resolved`
  (`src/sase/core/wait_dependency_resolution/_artifact_state.py:123`) returns `False`,
  and `is_done` / `is_failed` are both `False`. A settled gate member is stuck in a
  limbo that is neither resolved nor terminal.
- `_family_entity` requires `all(candidate.is_resolved ...)` across the whole family
  generation (`src/sase/core/wait_dependency_resolution/_index_entities.py:83-88`). One
  limbo gate member pins the family aggregate to `is_resolved=False` forever, so
  `WaitDependencyIndex.is_resolved(<family name>)` never returns `True`
  (`src/sase/core/wait_dependency_resolution/_index_queries.py:245-289`).

This is the exact defect fixed for monitor shells in `6a0c35c8e`
(`fix(monitor): resolve waits from terminal monitor state`), `2e2facb94`, and
`f929b5e2c`. Gate shells were introduced along the same family-shell abstraction but
were never given the matching wait-resolution semantics, and no wait-dependency test
covers a gate family member.

### Reproduction (run against the live artifacts, on unmodified `master`)

```python
from sase.bead.store_locator import closed_bead_ids_for_project
from sase.core.wait_dependency_resolution import (
    build_wait_dependency_index, dependency_resolution_status,
)

proj = "gh_bobs-org__bob-cli"
idx = build_wait_dependency_index(proj)
closed = closed_bead_ids_for_project(proj)
idx.is_resolved("bob-cli-1n.1")   # -> False
idx.is_resolved("bob-cli-1n.3")   # -> True
idx.is_resolved("bob-cli-1n.4")   # -> True
"bob-cli-1n.1" in closed          # -> True
```

The `bob-cli-1n.1` family generation:

| timestamp      | name                   | outcome     | resolved  | done  |
| -------------- | ---------------------- | ----------- | --------- | ----- |
| 20260827124955 | `bob-cli-1n.1--plan`   | `completed` | True      | True  |
| 20260827130544 | `bob-cli-1n.1--gate`   | `gated`     | **False** | False |
| 20260827130554 | `bob-cli-1n.1--1`      | `None`      | True      | False |
| 20260827131341 | `bob-cli-1n.1--gate-0` | `gated`     | **False** | False |
| 20260827131351 | `bob-cli-1n.1--2`      | `None`      | True      | False |
| 20260827132000 | `bob-cli-1n.1--gate-1` | `gated`     | **False** | False |
| 20260827132009 | `bob-cli-1n.1--3`      | `None`      | True      | False |
| 20260827132654 | `bob-cli-1n.1--gate-2` | `gated`     | **False** | False |
| 20260827132701 | `bob-cli-1n.1--4`      | `None`      | True      | False |
| 20260827133046 | `bob-cli-1n.1--gate-3` | `gated`     | **False** | False |
| 20260827133053 | `bob-cli-1n.1--5`      | `None`      | True      | False |
| 20260827134356 | `bob-cli-1n.1--gate-4` | `gated`     | **False** | False |
| 20260827134403 | `bob-cli-1n.1--6`      | `completed` | True      | True  |

The `wait_checks` chop log records the same finding every tick, and flags the outcome as
unrecognized:

```
[wait_checks] Unknown done outcome blocks waiter .../20260827125001:
  dependency=bob-cli-1n.1 artifact=.../20260827130544 outcome='gated'
[wait_checks] Terminal dependency still blocks waiter .../20260827125001:
  dependency=bob-cli-1n.1 artifact=.../20260827130544 outcome='gated'
```

Monkeypatching `effective_done_outcome` to translate `gated` via `gate_state` flips
`is_resolved("bob-cli-1n.1")` to `True` and reduces `bob-cli-1n.land`'s blockers to
`("bob-cli-1n.5", "bob-cli-1n.6")` — the two phases genuinely still in flight. That
confirms the translation gap is the whole root cause.

## Fix

Give gate shells the wait-resolution semantics monitor shells already have. The gate and
monitor shells already share `sase/shells/settlement.py`, a common `FamilyShellWire`
projection (`sase/core/agent_scan_wire_family_shell.py`), and the same follow-up outcome
vocabulary (`"launched"` / `"launched-degraded"`), so every step below is a
generalization rather than a new mechanism.

### Step 1 — Translate `gated` done markers

In `src/sase/core/dismissed_agent_completion.py`:

- Add `GATE_OUTCOME = "gated"` and
  `GATE_SUCCESS_STATES = frozenset({"answered", "completed", "stopped"})`, mirroring
  `MONITOR_OUTCOME` / `MONITOR_SUCCESS_STATES`.
- Define these locally rather than importing `sase.gate_shell.state`.
  `dismissed_agent_completion` is a low-level `sase.core` module and
  `sase/core/agent_scan_wire_family_shell.py:28-34` already documents the same
  duplicate-the-constant choice to avoid a cycle. Step 6 adds a drift guard so the two
  copies cannot diverge.
- Extend `effective_done_outcome` so `outcome == GATE_OUTCOME` resolves through
  `gate_state`: `SUCCESS_OUTCOME` when the state is in `GATE_SUCCESS_STATES`, otherwise
  `FAILURE_OUTCOME`. Fail closed exactly as the monitor branch does — a missing,
  non-string, or unrecognized `gate_state` must map to `FAILURE_OUTCOME`, never to
  success.
- Add `GATE_OUTCOME` to `KNOWN_DONE_OUTCOMES` so callers that inspect the raw outcome
  (the `wait_checks` "Unknown done outcome" log, `wait_watch/_classify`) stop treating a
  settled gate as unrecognized.
- Read `gate_state` in a shape-tolerant way. On-disk `done.json` carries the flat
  `gate_state` key, while wire-projected mappings nest it under `family_shell`. Route
  through `family_shell_from_mapping` (`sase/core/agent_scan_wire_family_shell.py:251`)
  so both shapes work, and require the projected `shell.kind` to match the outcome
  (`"gate"` for `gated`, `"monitor"` for `monitored`) so a mismatched marker fails
  closed instead of reading the wrong state field. Keep the existing
  flat-`monitor_state` behavior working; `test_monitor_wait_dependency.py` passes
  `{"outcome": "monitored", "monitor_state": ...}` directly and must keep passing.
- Export the new names from `__all__`.

### Step 2 — Do not release a family mid gate handoff

Step 1 alone introduces a regression window, so it must land with this step.
`settle_gate_shell` writes its done marker (`settlement.py:112`) _before_
`settle_shell_claim_and_followup` launches the follow-up member
(`settlement.py:132-149`). Once Step 1 makes that marker read as `completed`, the family
briefly contains only resolved members while its successor does not yet exist, and a
waiter could be released early. Monitor shells already close this window; generalize it:

- In `src/sase/core/wait_dependency_resolution/_artifact_state.py`, generalize
  `monitor_followup_handoff_agent` into a shell-generic helper that also recognizes a
  gate marker: `outcome == GATE_OUTCOME`, `gate_state` in the terminal gate set
  (`answered`, `completed`, `failed`, `timeout`, `stopped`, `lost` — mirroring
  `TERMINAL_GATE_STATES`), `gate_followup_outcome` in
  `SUCCESSFUL_MONITOR_FOLLOWUP_OUTCOMES`, and a non-empty `gate_followup_agent`. Both
  shells already write those follow-up fields through the shared
  `sase/shells/settlement.py`, so one predicate covers both once it reads the projected
  `FamilyShellWire` instead of the `monitor_*` keys.
- Rename `ArtifactCandidate.monitor_followup_agent`
  (`src/sase/core/wait_dependency_resolution/_types.py:47`) and the two helpers
  `_family_members_after_monitor_handoffs` / `_family_monitor_handoffs_have_successors`
  (`src/sase/core/wait_dependency_resolution/_index_entities.py:186-205`) to
  shell-neutral names, and update the three call sites in `_index_entities.py` and
  `_index_queries.py:195-199`. Behavior is unchanged for monitors; gates now get the
  same treatment.

This also fixes the converse case: a gate that ends `timeout` / `lost` / `failed` but
still launches its follow-up would otherwise map to `failed` under Step 1 and block the
family permanently, even though the family kept running. Dropping a handed-off shell
member from the aggregate is what keeps that correct.

### Step 3 — Classify gate members in `sase agent wait`

`src/sase/agent/wait_watch/_classify.py:205-221` builds a synthetic
`{"outcome": ..., "monitor_state": ...}` mapping and only forwards the monitor state, so
`sase agent wait` currently classifies a settled gate member as `TERMINAL_OTHER`, which
counts as a **failed** wait (`_types.py::WaitTargetState.failed`). Pass the record's
`family_shell` through instead of hand-rolling the monitor-only dict, so gate members
are classified by the same translation Step 1 installs.

### Step 4 — Classify dismissed gate records

`load_archived_agent_completions` falls back to the dismissed bundle when `done.json` is
gone. `_archived_outcome_from_bundle`
(`src/sase/core/dismissed_agent_completion.py:235`) recognizes only the monitor stop
status, so a dismissed gate shell yields `outcome=None`, is skipped entirely, and
re-strands every waiter on its family. Add the gate branch: when the archived status is
`DEFAULT_GATE_SHELL_SETTLED_STATUS` (`"GATED"`, from
`sase/notification_gates/model_shell.py:20`), classify via the bundle's `gate_state`
through `effective_done_outcome`. Keep the documented fail-closed behavior for a custom
`gate_stop_status`, which is indistinguishable from an arbitrary user status in the
archive summary — the monitor branch makes the same trade-off and says so in a comment.

### Step 5 — Fail closed for a pending gate member

`artifact_is_resolved` (`_artifact_state.py:123-134`) falls back to
`_completed_handoff_workflow_state` for a plan-chain artifact with no done marker, and
guards that fallback with `_is_monitor_member_meta` — the guard added by `f929b5e2c` to
stop a live monitor member from releasing waiters early. There is no gate equivalent.

A pending gate observed on the live host has `workflow_state.json::status == "running"`,
so this is not currently reachable, and it should be treated as hardening rather than a
second root cause. Generalize the guard to any family-shell member (monitor **or** gate,
via `sase.gate_shell.state.is_real_gate_member` / the existing role predicates) and pin
the behavior with a regression test so the fallback can never misfire on a gate whose
workflow state is finalized ahead of its done marker.

### Step 6 — Tests

- New `tests/test_gate_wait_dependency.py`, mirroring
  `tests/test_monitor_wait_dependency.py`:
  - each gate state resolves or blocks as specified (`answered` / `completed` /
    `stopped` resolve; `failed` / `timeout` / `lost` block; missing, non-string, and
    unrecognized states fail closed);
  - a family whose generation contains a settled gate member resolves — the direct
    regression for this bug, using a `--plan` root plus interleaved `--gate*` and
    numbered members like the `bob-cli-1n.1` table above;
  - the same for a clan aggregate;
  - a gate that recorded a follow-up agent not yet present in the generation keeps the
    family blocked (Step 2's handoff window), and releases once the successor appears;
  - a dismissed gate record classifies from its archived bundle (Step 4);
  - a pending gate member never resolves through the workflow-state fallback (Step 5).
- Extend `tests/test_axe_chop_wait_checks_plan_families.py` (or a sibling) so the
  `wait_checks` chop writes `ready.json` for a waiter blocked only by a settled gate
  member, and stops logging it as an unknown done outcome.
- Extend `tests/test_agent_wait_watch.py` for Step 3: a gate member is not reported as a
  failed wait.
- Add a drift guard asserting `GATE_SUCCESS_STATES` equals the `"Done"` bucket of
  `sase.gate_shell.state.GATE_STATE_BUCKETS`, and that `GATE_OUTCOME` matches the
  literal `settle_gate_shell` writes. Mirror the existing
  `tests/test_done_outcome_classification.py` conventions.

## Verification

- `just check` for the scoped lanes while iterating.
- `just check-full` through `/sase_monitor` before landing — this touches `sase/core/`,
  which several suites import.
- Re-run the reproduction above against the live `gh_bobs-org__bob-cli` artifacts and
  confirm `idx.is_resolved("bob-cli-1n.1")` is now `True` and that `bob-cli-1n.land`'s
  blockers are only the phases still genuinely in flight.

## Notes for the implementer

- **No migration is needed.** Stranded waiters poll `ready.json` and re-resolve their
  own dependency set on a fallback interval (`src/sase/axe/run_agent_wait.py:191-217`),
  so `bob-cli-1n.land` and any similar waiter release on their own once the
  `wait_checks` chop and the runner are executing the fixed code. Do not hand-write
  `ready.json` markers.
- **Do not change gate semantics.** `outcome: "gated"` stays the raw marker value for
  display and diagnostics, exactly as `"monitored"` did after `6a0c35c8e`. Only the
  _wait-resolution_ translation changes.
- **Out of scope:** TUI status rendering, gate lifecycle changes, and the Rust core.
  `sase-core` was checked: it carries `"gated"` only in scan-wire fixtures
  (`crates/sase_core/src/agent_scan/wire.rs:1217`, `scanner.rs:1508`) and holds no
  wait-dependency resolution logic, so no cross-repo change is required.
- The three monitor commits `6a0c35c8e`, `2e2facb94`, and `f929b5e2c` are the reference
  implementation for Steps 1, 2, and 5 respectively; read them before writing code.
- Consider filing a follow-up task bead to audit whether any _other_ family-shell kind
  is added in future without wait-resolution semantics — the drift guard in Step 6
  covers gates, but nothing forces a new shell kind to register a translation.
