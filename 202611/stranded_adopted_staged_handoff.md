---
tier: tale
title: Stop delivered staged handoffs from owning beads forever
goal:
  A finished agent session that handed off to a monitor, gate, or plan successor and
  left its bead in_progress is classified stranded, while in-flight stages and live
  successors still own the bead.
size: medium
proposed_by: bbugyi200.athena.18g
proposed_with: claude/opus
create_time: 2026-11-05 06:45:29
status: wip
---

# Stop delivered staged handoffs from owning beads forever

## Problem

A phase agent that finishes without closing its `in_progress` bead should show **⚐
STRANDED** in ACE. Agent sessions that ever handed off to a monitor, gate, or plan
successor never do. Real example: `bob-cli-h4.4` (bob-cli project). Eleven turns
(`--plan`, `--mon`, `--1`, `--mon-0`, … `--5`) all finished by 2026-11-04 19:43 EST, and
the bead was still `in_progress`. The Agents tab showed the row as plain `DONE`. There
was no STRANDED overlay, no stranded marker on the clan, and its waiters
(`bob-cli-h4.5`, `bob-cli-h4.6`, `bob-cli-h4.land`) showed no stranded wait target.

### Root cause

The stranded-work reducer lives in the linked `sase-core` repo:
`crates/sase_core/src/stranded_work/classify.rs`. Its `owner_kind()` returns
`Some("handoff")` for **any** member whose `staged_handoff` fact is set. It ignores that
member's adoption receipt:

```rust
if normalize_optional(&member.staged_handoff).is_some() {
    return Some("handoff".to_string());
}
```

The host intentionally **keeps** `.sase_handoff_staged.json` after a successful
adoption, next to a `.sase_handoff_adopted.json` receipt with `outcome: "adopted"`. The
staged-recovery service (`src/sase/axe/run_agent_exec_staged_recovery*.py`) needs both
files to re-deliver idempotently. Nothing ever deletes the staged record after adoption.
As a result, every finished turn that ever ran `sase monitor start`, a gate, or
`sase plan propose` counts as a live `handoff` owner forever. `bead_owner()` picks the
newest such member, the verdict is `owned/handoff`, and the bead can never reach
`stranded`.

The Rust test `owned_staged_handoff_and_held_receipt` in
`crates/sase_core/src/stranded_work/tests.rs` pins this behavior with the comment "A
staged record still owns even when a success receipt is present: the stage is unresolved
until the later reconciler clears it." That premise is false.
`sase.axe.staged_handoff_reconcile.reconcile_staged_handoffs` only adopts or delivers
stages. It never clears an adopted stage.

### Evidence gathered while planning

- On disk, every finished turn that handed off still has both files, for example
  `bob-cli-h8.5--plan`, `--1`, and `--4`, and `bob-cli-h8.1--plan` and `--1`. The
  receipts look like
  `{"outcome":"adopted","refs":{"member_artifacts_dir":…,"monitor_id":…}}`.
- `bob-cli-h4.4`'s own artifact directories were removed after the screenshot. Its
  published turn records show the same shape: `--plan`, `--1`…`--4` each handed off to a
  monitor, and `--5` was the last agent turn (a stitch-resume repair that never closed
  the bead).
- Live reproductions on current data (`collect_stranded_work_input` plus
  `reduce_stranded_work`):
  - `bob-cli-h8.7`: every member done, verdict `owned/handoff` (owner
    `bob-cli-h8.7--plan`, staged `gate`, receipt `adopted`).
  - `sase-1qz.6.1.4`: every member done, verdict `owned/handoff` (owner `--plan`, staged
    `monitor`, receipt `adopted`).
  - With adopted-and-delivered stages ignored, `bob-cli-h8.7` becomes
    `stranded/unclosed`, `sase-1qz.6.1.4` becomes `stranded/kept`, and `sase-1ra.11`
    becomes `delegated` to its live child epic. `sase-1r2.7`, `sase-1n6.11.4`,
    `sase-1ra.11.1.1`, and `bob-cli-h8.12` stay owned through their live monitor
    successors. Parent-epic verdicts change only through propagation.

## Design

A staged record means "a handoff is in flight", and it owns the bead only until the
stage is delivered. Delivered means:

- the member has a **matching `adopted` receipt** (the same check as
  `classify_receipt(...) == "adopted"`, which also verifies `staged_id`); **and**
- the stage needs no further delivery. This mirrors `_already_needs_delivery()` in
  `src/sase/axe/run_agent_exec_staged_recovery.py`:
  - For kinds outside `SUCCESSOR_KINDS` (`monitor`, `gate`), or a successor kind whose
    receipt `refs` carry no `successor_artifacts_dir`, the adopted receipt alone proves
    delivery. The monitor or gate member, or the plan gate, already exists and has its
    own session-member facts.
  - For `SUCCESSOR_KINDS` (`pipe`, `plan`, `questions`) whose receipt names a
    `successor_artifacts_dir`, `observe_destination(staged_record, artifacts_dir=…)`
    must report `owned` or `terminal`. `pending` and `absent` stay owned, because the
    recovery service must still deliver the successor exactly once.

Once a stage is delivered, the successor owns the bead through its own member facts, if
it is live. If nothing else owns the bead, the normal settle-grace and stranded-cause
rules apply. Malformed staged state, a held (`handoff_not_adopted`) receipt, a
mismatched receipt, and a staged record without a receipt all keep owning, exactly as
today. Per the Rust-core boundary, the ownership rule belongs in `sase-core`. Python
only collects the filesystem fact.

## Changes

### 1. `sase-core` (open with `sase repo open sase-core`)

- `crates/sase_core/src/stranded_work/wire.rs`: add
  `#[serde(default)] pub staged_delivered: bool` to `StrandedWorkMemberFactsWire`, with
  a doc comment that states the delivery definition above. The struct is
  `deny_unknown_fields` and the new field defaults to `false`, so older Python input
  keeps today's behavior. Leave `STRANDED_WORK_WIRE_SCHEMA_VERSION` at 2, because the
  change is additive and defaulted.
- `crates/sase_core/src/stranded_work/classify.rs`, `owner_kind()`: a staged record owns
  unless `staged_receipt_outcome == "adopted"` (after normalization) **and**
  `staged_delivered`. Keep the existing rule that a receipt outcome other than `adopted`
  owns. Keep the `workspace_retention` early return. Rewrite the comment block so it
  describes the delivered-stage exemption and no longer mentions a reconciler that
  clears stages.
- `crates/sase_core/src/stranded_work/tests.rs`:
  - Update `owned_staged_handoff_and_held_receipt`. Staged plus `adopted` without
    `staged_delivered` stays `owned/handoff`. Staged plus `adopted` plus
    `staged_delivered` is not owned: it is `stranded` when cold and `settling` within
    the grace window.
  - Add a regression test for the `bob-cli-h4.4` session shape. Members are a done
    `--plan` agent with a delivered `monitor` stage; a done `--mon` monitor with
    `followup_outcome: "launched"`; done agent turns `--1`…`--4`, each with a delivered
    monitor stage; failed monitors with launched follow-ups; and a newest done agent
    turn `--5` with no stage and no declared action. Assert `stranded/unclosed`. Add one
    variant where `--5` declared `keep` and assert `stranded/kept`.
  - Add a test that a delivered stage still lets a live successor member own the bead
    (`owner` is the successor, not the stager).
  - Add a test that `staged_delivered: true` without an `adopted` receipt (defensive)
    still owns through `handoff`.
- `crates/sase_core_py`: confirm that the PyO3 stranded-work binding round-trips the new
  field through serde with no code change. Extend its tests if they enumerate member
  fields.
- `CHANGELOG.md` files in sase-core: follow that repo's `AGENTS.md` and existing
  convention. Do not hand-edit them if they are release-generated.

### 2. `sase` (this repo)

- `src/sase/bead/stranded_facts.py`:
  - Extend `_staged_facts()`, or add a sibling helper called from `_member_facts()`, so
    it also returns `staged_delivered: bool`, computed as in **Design**. Reuse the
    existing helpers instead of re-reading JSON by hand: `classify_receipt` and
    `adopted_destination_refs` from `sase.axe._run_agent_exec_staged_recovery_shared`,
    and `observe_destination` from `sase.axe.run_agent_exec_staged_recovery_facts`.
    Promote `_already_needs_delivery` from `sase.axe.run_agent_exec_staged_recovery` to
    a shared public helper (for example `adopted_stage_needs_delivery(receipt, kind)` in
    `_run_agent_exec_staged_recovery_shared.py`) and use it from both the recovery
    service and the collector, so the two definitions cannot drift. Wrap the reads in
    the module's existing fail-open style: any read or contract error yields
    `staged_delivered = False`.
  - Add `"staged_delivered": …` to the member facts dict.
  - Update the module docstring to say that adopted-and-delivered stages no longer own.
- `tests/test_stranded_staged_handoff.py`: update the module docstring and add tests
  that use the existing helpers (`_write_plan_record`, `_write_receipt`,
  `wire_collector`, `_done_records`):
  - A staged `monitor` record plus a matching `adopted` receipt whose `refs` name a
    `member_artifacts_dir`: the member reports `staged_delivered is True`, and the
    verdict is `stranded`, not `owned/handoff`.
  - A staged `plan` record plus an `adopted` receipt with only `plan_gate_artifacts_dir`
    refs: delivered, so not owned through `handoff`.
  - A staged `plan` record plus an `adopted` receipt whose `refs` name a
    `successor_artifacts_dir` whose staged-successor `delivery_state` is pending or
    absent: `staged_delivered is False`, and the verdict is `owned/handoff`. With
    `delivery_state` `started`, `owned`, or `terminal`, it is delivered and not owned.
    Use the existing staged-successor writers in `sase.agent.staged_handoff` instead of
    hand-written JSON.
  - A receipt whose `staged_id` does not match the staged record: not delivered, so
    still owned.
  - Keep the existing four tests passing unchanged in intent.
- `docs/reference/work-units/bead-lifecycles.md`, section "Stranded beads": add a
  sentence or two after the workspace-hold paragraph. An adopted staged handoff whose
  successor has been delivered no longer owns the bead; the successor owns it through
  its own session member, and with no other live owner the bead can become stranded.
  Read the `docs_writing` reference memory first.
- `sase-core-revision.txt`: do **not** hand-bump it. The pin moves automatically when
  one declaration commits both repos (see `docs/contribute/rust-backend.md`, "The CI
  source revision pin"). Commit the `sase-core` change in the same turn. Make sure the
  local dev binding is rebuilt before running the Python tests, so they exercise the new
  reducer field.

### 3. Discovered follow-up (file, do not fix here)

The same survey surfaced a second "dead member owns forever" shape. A monitor member
that is already **done** but still reports `followup_pending = true` (a stale
`next_action`, with `followup_outcome` absent) owns its bead through `owner_kind()`'s
monitor/gate branch. Examples: `sase-1ih.2` (`--mon-0` done, `turn_state=running`),
`sase-kp` (`sase-kp.land--mon` done, `turn_state=failed`), `sase-ru.6` and `sase-s5`
(`turn_state=stopped`). File this as a task bead via `/sase_new_task`, citing these bead
IDs and the `owner_kind()` branch. It needs its own root-cause check of why the
follow-up never launched, so it stays out of scope here.

## Verification

- `sase-core`: run the stranded-work unit tests and the crate's standard checks per its
  `AGENTS.md`.
- `sase`: follow the `lint_and_test` reference memory (`just check` through
  `sase tool run`). Make sure `tests/test_stranded_staged_handoff.py` and the other
  `tests/*stranded*` suites pass.
- Manual spot check after the binding rebuild: run the collector and reducer for
  `gh_bobs-org__bob-cli` (bead `bob-cli-h8.7`) and `gh_sase-org__sase` (bead
  `sase-1qz.6.1.4`). Confirm that they report `stranded` instead of `owned/handoff`, and
  that beads with a live monitor successor (`sase-1r2.7`, `bob-cli-h8.12`) are still
  `owned`.
