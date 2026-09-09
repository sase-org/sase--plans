---
status: done
tier: epic
title: Collapse the gate-shell status machinery and remove the beta flag
goal: "The gate shell is the only thing that publishes a plan or question status: the
  `gate_shell_handoff` beta flag and its blocking Off branch are gone, the notification
  and family-policy status overrides that existed only to give a blocked plan chain a
  visible row are gone, the agent-list colour ladder is one shared pair-accent path over
  declared gate accents, and the flat `monitor_*` / `gate_*` wire blocks are one nested
  `family_shell` record at wire schema v7.

  "
phases:
  - id: accent-pin
    title: Pin the plan and epic gate accents
    depends_on: []
    size: small
    description:
      "accent-pin: transcribe the agent-list ladder's hand-tuned plan, tale, epic,
      feedback, and rejection accents onto the plan gate shell spec, which today
      declares different colours, and add a guard test that every ladder-pinned status
      label resolves to the same accent through a gate spec."
  - id: flag-removal
    title: Remove the gate_shell_handoff flag and the blocking Off branch
    depends_on: []
    size: large
    description:
      "flag-removal: make the gate-shell handoff unconditional in the plan and questions
      marker handlers, delete the flag module, registry member, and config schema
      property, delete the blocking wait machinery the Off branch was the last consumer
      of, retarget the runner tests that drove the Off branch, and close flag bead
      sase-uo."
  - id: status-strip
    title: Retire the notification and family status overrides
    depends_on:
      - flag-removal
    size: large
    description:
      "status-strip: delete the notification-driven pending-plan and question status
      overrides and the `_agent_status_overrides` facade, strip the family policy and
      synthetic-planner modules to what the gate shell left reachable, and prove the
      family node still shows the gate's status without them."
  - id: ladder-collapse
    title: Collapse the agent-list status colour ladder
    depends_on:
      - accent-pin
      - status-strip
    size: medium
    description:
      "ladder-collapse: fold the hand-written status branches in the agent-list row
      renderer into the shared pair-accent path, delete the plan-approval `status_label`
      plumbing that fed the optimistic overrides, drop the vestigial `MONITORED`
      terminal-status special case, and rebaseline the PNG goldens."
  - id: wire-v7
    title: One nested family_shell wire record at schema v7
    depends_on: []
    size: medium
    description:
      "wire-v7: fold the flat `monitor_*` and `gate_*` field blocks on `AgentMetaWire`
      and `DoneMarkerWire` into one nested `family_shell` record in both the Rust core
      and the Python wire, bump the agent-scan wire schema to 7, and keep every existing
      reader working through a compatibility projection."
proposed_by: bbugyi200.athena.sase-ud.13
parent_bead: sase-ud.13
bead_id: sase-ud.13.1
create_time: 2026-09-09 19:50:32
---

- **PROMPT:**
  [prompts/202608/gate_shell_status_collapse.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gate_shell_status_collapse.md)
- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.13.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.13.1.md)

# Plan: Collapse the gate-shell status machinery and remove the beta flag

This is the payoff phase of epic `sase-ud` ("Gate shells — a decision that outlives the
agent that asked"), whose plan is `plan:202608/gate_shells.md`. Every earlier phase
added a mechanism; this one deletes the workarounds those mechanisms made unnecessary.
Read the epic plan's `status-collapse` section and its §8 (`#fork` and family status),
§9 (colour regression), and R4 (colour flattening) before starting a phase here — this
plan refines that section against the tree as it actually is, and records where the epic
plan's expectations no longer match it.

## Why this is an epic and not a tale

`sase-ud.13` is a `large` phase bead, so it plans before implementing. The measured work
does not fit a tale's `medium` ceiling: the flag removal alone touches six source files
and about forty patch sites across thirteen test files; the status strip faces roughly
114 test functions in the fourteen `test_agent_loader_status_override_*` files plus PNG
goldens; and the wire fold spans two repositories and fifty-seven wire fields. The three
are also cleanly separable, and one of them — the wire fold — the parent epic already
marks droppable. Splitting them buys a fresh context window per phase and lets the wire
decision be made on its own evidence. `sase-ud.13` closes when this sub-epic's phases
do.

## The problem

A plan or question submitted from inside an agent used to block that agent. Because the
blocked agent could not publish anything, three separate mechanisms were built to give
the user a visible "you must act" row:

1. **Notification-driven overrides** — `_notification_status_overrides.py` scans unread
   `PlanApproval` / `EpicApproval` / `UserQuestion` notifications each poll and writes
   `TALE` / `EPIC` / `QUESTION` into an in-memory override map keyed by agent identity.
2. **Family status policy** — `_agent_status_family_policy.py` reconstructs, from
   artifact timestamps, whether a `DONE` planner row is "really" awaiting review
   (`is_awaiting_plan_review`, `has_unreviewed_submitted_plan`,
   `has_unanswered_completed_question`), and what the plan chain's terminal and handoff
   labels should be (`done_handoff_status`, `active_approved_plan_handoff_status`,
   `superseded_by_feedback_round`, `planner_child_status`).
3. **Synthetic planner children** — `_agent_status_family_planner.py` materializes an
   `Agent` row that does not exist on disk, purely so the plan chain has something to
   carry those statuses.

On top of those, `_agent_list_render_agent_status.py` carries a hand-written ladder of
about twenty `elif` branches pinning one colour per status literal.

The gate shell replaced the premise. A plan or question gate is now a real, named,
durable family member with its own artifact directory, its own status pair, and its own
declared accent. It publishes the decision itself. The three mechanisms above are
scaffolding around a hole that no longer exists.

## Verified against the tree

Everything in this section was checked in the working tree at epic phase `sase-ud.13`;
do not re-derive it, but do re-check anything that looks stale.

- The flag is `gate_shell_handoff` (`beta`, `default=off`), bead `sase-uo`. Its only
  decision points are `src/sase/axe/run_agent_exec_plan.py:117-125` and
  `src/sase/axe/run_agent_exec_questions.py:144`. `src/sase/gate_shell/flag.py` holds
  the single accessor. The registry member is in `src/sase/feature_flags/registry.py`,
  and the config property is in `src/sase/config/sase.schema.json` (kept in sync by
  `tools/sync_feature_flags_schema`, linted by `tools/check_feature_flags`).
- Only two test files reference the flag:
  `tests/test_axe_run_agent_exec_plan_gate_shell.py` and
  `tests/test_axe_run_agent_exec_questions_gate_shell.py`.
- `sase.llm_provider._plan_utils.handle_plan_approval` has exactly one caller: the plan
  marker handler's Off branch. `sase.plan_gate.create_plan_approval_gate` has exactly
  one caller: `handle_plan_approval`. `plan_approval_result_from_gate_response` and
  `mark_auto_approved_plan_handled` are **also** used by `sase.plan_shell` and must
  survive.
- `sase.axe.run_agent_helpers_questions.handle_questions_flow` has exactly one caller
  (the questions Off branch, via the `sase.axe.run_agent_helpers` facade re-export).
  `sase.user_question_actions.create_user_question_gate` has exactly one caller:
  `handle_questions_flow`. `user_question_gate_spec` is also used by
  `sase.question_shell.create` and must survive.
- `sase.notification_gates.poller.wait_for_gate` keeps a live consumer after this epic:
  `sase/notifications/cli_wait.py` (`sase gate wait`). Do not delete it.
- Around forty test call sites across thirteen files patch
  `sase.llm_provider._plan_utils.handle_plan_approval` and then call
  `handle_plan_marker`. They are testing what happens _after_ a plan result, not the
  wait itself.
- **R4 currently fails.** The epic plan's §9 asserts that "the built-in plan and
  question gates pin today's exact values". The question gate does (`QUESTION`
  `#FFAF00`, `ANSWERED` `#5FD7FF` — both match the ladder). The plan gate does not. See
  the accent table below.
- The epic plan's §8 says gate shells "must not be filtered" at
  `concrete_agent_statuses`. In the tree they **are** filtered: that function drops
  every row for which `row_is_family_shell(row)` is true, and `row_is_family_shell` is
  `row.is_monitor or row.is_gate` (`src/sase/ace/tui/models/agent_family_members.py`).
  The family node instead picks up the gate's status through `_mirror_root_from_child`
  in `src/sase/ace/tui/models/_agent_status_apply.py`. Treat §8's claim as an
  expectation to verify, not as a fact.
- `"MONITORED"` appears in exactly one place outside the monitor status contract itself:
  the `_TERMINAL_STATUSES` set in `src/sase/agent/status_buckets.py`. A custom monitor
  stop label such as `TESTED` is not in that set and still buckets correctly, which is
  the evidence that the literal is vestigial.

### The accent table

The ladder in `src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` and the plan
gate shell spec in `src/sase/plan_shell/create.py` disagree on every plan-family status:

| Status label     | Ladder (today's colour) | Plan gate spec declares | Match |
| ---------------- | ----------------------- | ----------------------- | ----- |
| `PLAN`           | `#FF87AF`               | (no branch)             | n/a   |
| `TALE`           | `#FF87AF`               | `#FFD75F`               | no    |
| `EPIC`           | `#D787FF`               | `#AF87FF`               | no    |
| `PLAN APPROVED`  | `#00D7AF`               | `#0BD68B`               | no    |
| `TALE APPROVED`  | `#00D7D7`               | `#0BD68B`               | no    |
| `EPIC APPROVED`  | `#5FD7AF`               | `#AF87FF`               | no    |
| `PLAN COMMITTED` | `#5FD75F`               | `#0BD68B`               | no    |
| `PLAN REJECTED`  | `#D7AF5F`               | `#FF5F5F`               | no    |
| `FEEDBACK`       | `#FF5FD7`               | `#5FD7FF`               | no    |
| `QUESTION`       | `#FFAF00`               | `#FFAF00` (question)    | yes   |
| `ANSWERED`       | `#5FD7FF`               | `#5FD7FF` (question)    | yes   |

## Design

**The gate shell is the status.** After this epic there is exactly one publisher of a
plan or question status: the gate shell's own recorded start/stop pair plus its declared
accent, projected by `gate_status_presentation` in
`src/sase/ace/tui/widgets/_agent_list_styling.py`, which `append_agent_row_status`
already consults _before_ its ladder. Nothing else reconstructs a status from
timestamps, notifications, or synthesized rows.

**"Strip to what is still reachable" is a measurement, not a list.** The epic plan's
"What this deletes" table names specific functions in `_agent_status_family_policy.py`.
Treat that as the author's estimate. The operative instruction in the `status-collapse`
section is to strip these modules _to what is still reachable_, and reachability is
determined by the tree plus the test suite, not by the table.
`active_approved_plan_handoff_status`, for example, labels a **running coder child** — a
row the gate shell does not replace, because the gate settles at `TALE APPROVED` and
only then launches the coder. Delete a symbol when its inputs can no longer occur; keep
it when they can, and say why in the phase's bead note.

**Deleted behaviour needs deleted tests, not weakened ones.** A test that asserts an
override this epic retires is evidence of the retired world and should be deleted with
it. A test that asserts a projection the gate shell still owes the user should be
rewritten to drive it through a gate-shell fixture. Never neutralize an assertion to
make a suite pass.

**Wire v7 is independent and droppable.** The epic plan says so explicitly: it is
behaviour-free by construction, nothing else depends on it, and if it turns out riskier
than budgeted the correct action is to drop it. It is a separate phase here with no
dependencies so that decision can be made on its own evidence.

## Risks

- **R-A — Silent hue change.** Collapsing the ladder before the accents are pinned
  changes plan status colours with no test failure. `accent-pin` exists to make that
  impossible, and `ladder-collapse` depends on it.
- **R-B — The family node goes quiet.** If the overrides are retired and the gate's
  status does not reach the family container row, every blocked family reads `DONE` and
  the "you must act" signal is destroyed. `status-strip` must demonstrate the opposite
  with a test before it deletes anything.
- **R-C — Legacy artifacts.** ACE renders agent directories written before this epic. A
  plan family on disk from the blocking era has no gate-shell member. Decide explicitly,
  per deleted symbol, whether those rows degrade acceptably (a historical planner
  reading `DONE` instead of `TALE` is acceptable; a live family losing its question
  signal is not).
- **R-D — Test blast radius.** Fourteen `tests/test_agent_loader_status_override_*.py`
  files hold about 114 test functions, thirty-three files mention `WORKING PLAN` /
  `WORKING TALE`, and nine mention synthetic planner rows. Budget for reading them, not
  just running them.
- **R-E — Cross-repo drift.** `wire-v7` changes `crates/sase_core/src/agent_scan/` in
  the linked `sase-core` repo and the Python wire together. A schema bump landed on one
  side alone breaks every reader.

## Working agreements for every phase

- Open the linked `sase-core` repo only through the `/sase_repo` skill and use the path
  it prints.
- `just check` before replying; `just check-full` and `just test-visual` through the
  `/sase_monitor` skill for any phase that touches TUI rendering or the wire.
- Do not edit `sase/memory/` — epic phase `sase-ud.14` owns the memory and decision
  record. Skill templates in `src/sase/xprompts/skills/` may be edited where a phase
  names them, but do not run `sase skill init --force`; the deploy needs a clean, merged
  tree.
- Record discovered follow-up work as `PROPOSED FOLLOW-UP:` notes on your own phase
  bead. Do not create beads.

---

# Phases

## accent-pin — Pin the plan and epic gate accents

Land this before anything deletes a colour branch. It is a small, self-contained
correction of a fact the epic plan assumed was already true.

- In `src/sase/plan_shell/create.py`, change `plan_gate_shell_block`'s declared accents
  to the ladder's values from the accent table above: the tale block's `accent` to
  `#FF87AF`, `approve+commit`'s `TALE APPROVED` to `#00D7D7`, `approve`'s
  `PLAN APPROVED` to `#00D7AF`, `commit`'s `PLAN COMMITTED` to `#5FD75F`, `reject`'s
  `PLAN REJECTED` to `#D7AF5F`, and the feedback branch's `FEEDBACK` to `#FF5FD7`; the
  epic block's `accent` and its `approve` branch to `#D787FF`, and `EPIC APPROVED` to
  `#5FD7AF`. Leave the `timeout`, `stopped`, and `failed` branches on the shared
  warning/failure colours — those labels (`PLAN TIMED OUT`, `EPIC CANCELLED`, …) have no
  ladder branch to preserve.
- `_coder_branch` and `_feedback_branch` currently hard-code one accent each across both
  tiers. Since `approve+commit` and `approve` now need different accents, give
  `_coder_branch` an accent parameter rather than duplicating the helper.
- Add a guard test that pins the correspondence rather than the literals in isolation:
  for every status label the agent-list ladder pins a colour for, assert that the
  built-in plan, epic-plan, and question gate specs that can produce that label declare
  the same accent. This is the test that makes `ladder-collapse` safe and that would
  have caught the current drift.
- Do not change the question gate spec in `src/sase/question_shell/create.py`; it
  already matches.

Verification: `just check`, plus `just test-visual` if any snapshot renders a plan gate
row.

## flag-removal — Remove the gate_shell_handoff flag and the blocking Off branch

Removing a SASE flag means deleting the Off branch, making the On branch unconditional,
removing the registry entry, and closing the flag bead in the same change. Nothing here
is conditional on the other phases.

**Make the handoff unconditional.**

- `src/sase/axe/run_agent_exec_plan.py`: drop the `gate_shell_handoff_enabled` import
  and check from `handle_plan_marker`, inline `_handle_plan_via_gate_shell`'s body or
  call it unconditionally, and delete the `handle_plan_approval` call and its `killed` /
  `plan_rejected` fallbacks. `_continue_after_plan_result` stays — the `%auto`
  short-circuit still reaches it through `plan_result_from_gate_creation`.
- `src/sase/axe/run_agent_exec_questions.py`: delete the Off-branch body of
  `handle_questions_marker` (everything from `normalize_handoff_interruption_state`
  through the `_update_sdd_prompt_snapshot_qa` call) and make
  `_handle_questions_via_gate_shell` the whole function.
  `_continue_after_auto_answered_question` stays. Re-check which of
  `_interrupted_phase_meta`, `_question_interrupted_suffix_and_role`,
  `_question_successor_fallback_token`, and `_update_sdd_prompt_snapshot_qa` still have
  callers; the auto-answered path uses several of them.
- Delete `src/sase/gate_shell/flag.py`. `src/sase/gate_shell/__init__.py` does not
  re-export it today, so no package-level edit is expected.
- Remove `FeatureFlag.gate_shell_handoff` and its `_FEATURE_FLAG_DEFINITIONS` entry from
  `src/sase/feature_flags/registry.py`, then run `tools/sync_feature_flags_schema` (or
  edit `src/sase/config/sase.schema.json` to match) and confirm
  `tools/check_feature_flags` passes.
- Update the stale flag prose in `src/sase/xprompts/skills/sase_plan.md` (the "With the
  flag disabled …" sentence) and `src/sase/xprompts/skills/sase_questions.md` (the whole
  enabled/disabled split under "Handoff And Continuation"), so both describe one
  unconditional behaviour. Keep the edits minimal — phase `sase-ud.14` owns the full
  consistency pass over these templates.

**Delete what the Off branch was the last consumer of.** Work outward from the two
handlers and let `just lint`'s Symvision stage find the tail; read
`sase memory read symvision.md` first if you have not. Expected, but verify each:

- `sase.llm_provider._plan_utils.handle_plan_approval` and its now-dead private helpers.
  Keep `plan_approval_result_from_gate_response` and `mark_auto_approved_plan_handled`.
- `sase.plan_gate.create_plan_approval_gate` and its `__all__` entry.
- `src/sase/axe/run_agent_helpers_questions.py` in full, plus the
  `handle_questions_flow` re-export and `__all__` entry in
  `src/sase/axe/run_agent_helpers.py`.
- `sase.user_question_actions.create_user_question_gate` and its `__all__` entry. Keep
  `user_question_gate_spec`.
- Any `pending_question.json` write path that only `handle_questions_flow` reached. The
  marker's _readers_ stay: ACE and the scan wire still project it for records written by
  earlier releases.

**Tests.**

- In `tests/test_axe_run_agent_exec_plan_gate_shell.py` and
  `tests/test_axe_run_agent_exec_questions_gate_shell.py`, delete the Off-branch cases
  and the `monkeypatch.setattr(..., lambda: True)` lines from the On-branch cases.
- Retarget the roughly forty sites that patch
  `sase.llm_provider._plan_utils.handle_plan_approval`. The equivalent seam on the
  gate-shell path is `sase.plan_shell.plan_result_from_gate_creation`, which returns the
  same `PlanApprovalResult`. Add a stub of `sase.plan_shell.create_plan_gate_shell`
  returning a creation whose `should_handoff` is `False` to `PLAN_PATCHES` in
  `tests/_axe_run_agent_exec_plan_helpers.py` so the shared harness keeps driving
  `_continue_after_plan_result`, then swap each per-test patch target. Files:
  `tests/_axe_run_agent_exec_plan_followup_prompt_helpers.py`,
  `tests/plan_chain_golden/test_marker_and_loop_golden.py`, and the
  `tests/test_axe_run_agent_exec_plan_*` and
  `tests/test_plan_approval_launch_reliability_integration.py` set.
- Delete `tests/test_axe_run_agent_helpers_questions.py` and the
  `create_plan_approval_gate` cases in `tests/test_plan_gates_execution.py` along with
  the code they cover; keep any case that exercises a surviving symbol.

**Close the flag bead.** `sase bead close sase-uo --note "<what you verified>"` in this
same change, per the flag-removal rule. This is the one bead outside your own phase bead
that you should close; do not close `sase-ud` or any other ancestor.

Verification: `just check`, then `just check-full` through `/sase_monitor`.

## status-strip — Retire the notification and family status overrides

The riskiest phase. Work strictly in the order below: prove the replacement signal
first, delete second.

**Step 1 — Prove the family node still speaks.** Before deleting anything, add tests
that pin the post-gate-shell contract:

- A family whose newest member is a _pending_ plan gate shell renders `TALE` (or `EPIC`)
  on the family container row, and its planner member renders `DONE`.
- The same for a pending question gate shell rendering `QUESTION`.
- A settled gate shell renders its branch's status (`TALE APPROVED`, `FEEDBACK`,
  `PLAN REJECTED`) on the container.

Build these from real gate-shell member metadata, not from the override map. If any of
them fails on today's tree, that failure is the actual work of this phase: find where
the gate's status is dropped and fix it. The epic plan's §8 predicts
`concrete_agent_statuses` as the suspect, but the tree currently filters gate shells
there through `row_is_family_shell` and routes the container's status through
`_mirror_root_from_child` instead — so confirm before changing either.

**Step 2 — Delete the notification overrides.** Retire
`src/sase/ace/tui/actions/agents/_notification_status_overrides.py`. Separate its two
jobs before deleting:

- The pending-plan / `UserQuestion` override writing into `_agent_status_overrides` and
  `_agent_pre_question_status` is what this epic retires.
- The external-response reconciliation (`prepare_plan_notification_reconciliation`,
  `_prepare_external_plan_response`, and the auto-dismiss of a `PlanApproval` answered
  from Telegram or mobile) is a _notification lifecycle_ concern, not a status concern.
  Determine whether the gate-shell settlement path already dismisses those
  notifications; if it does, delete this too, and if it does not, keep the dismissal and
  delete only the status writes. Say which, and why, in the bead note.
- Then delete the `_agent_status_overrides` / `_agent_pre_question_status` map plumbing
  that has no remaining writer, and delete
  `src/sase/ace/tui/models/_agent_status_overrides.py` (a re-export facade whose callers
  should import from `_agent_status_apply` and `_agent_status_family` directly).

**Step 3 — Strip the family policy and planner modules.** For each symbol in
`src/sase/ace/tui/models/_agent_status_family_policy.py` and
`src/sase/ace/tui/models/_agent_status_family_planner.py`, decide reachability against
the gate-shell world and record the decision:

- `is_awaiting_plan_review`, `has_unreviewed_submitted_plan`,
  `has_unanswered_completed_question`, `superseded_by_feedback_round`,
  `planner_child_status`, `done_handoff_status`, and `ensure_synthetic_planner_children`
  / `sync_planner_child_from_parent` are the epic's expected deletions — they exist to
  reconstruct a status the gate shell now publishes directly.
- `active_approved_plan_handoff_status`, `is_completed_plan_handoff_child`,
  `is_completed_epic_followup_child`, and `approved_followup_planner_status` label
  concrete follow-up agent rows the gate shell does **not** replace. Verify before
  deleting; keeping them is a legitimate outcome.
- Remove from `src/sase/agent/status_buckets.py` only the constants that lose their last
  consumer. `PENDING_PLAN_REVIEW_STATUSES`, `APPROVED_PLAN_STATUSES`,
  `WORKING_PLAN_STATUSES`, `ACTIVE_PLAN_HANDOFF_STATUSES`, and
  `WORKING_PLAN_STATUS_TO_APPROVED` are the epic's candidates; several are also read by
  query filters and `AUTO_APPROVE_ELIGIBLE_STATUSES`, so check each.
- Prune `apply_status_overrides` in `src/sase/ace/tui/models/_agent_status_apply.py` to
  the passes that survive, and keep its docstring honest about what it still does.

**Step 4 — The tests.** Read the fourteen `tests/test_agent_loader_status_override_*.py`
files and classify every failing test: delete the ones asserting a retired override,
rewrite the ones asserting a projection the gate shell still owes, and leave the rest.
The same triage applies to the synthetic-planner tests and to `tests/ace/tui/`
family-projection tests. Do not weaken an assertion to make it pass.

Verification: `just check`, then `just check-full` and `just test-visual` through
`/sase_monitor`.

## ladder-collapse — Collapse the agent-list status colour ladder

With accents pinned and the overrides gone, the hand-written ladder in
`src/sase/ace/tui/widgets/_agent_list_render_agent_status.py` is redundant for every
status a gate shell owns.

- Delete the `elif` branches for `PLAN`, `TALE`, `EPIC`, `FEEDBACK`, `PLAN APPROVED`,
  `TALE APPROVED`, `PLAN COMMITTED`, `EPIC APPROVED`, `EPIC CREATED`, `PLAN REJECTED`,
  `QUESTION`, `ANSWERED`, and the `PLAN DONE` / `TALE DONE` members of the `DONE`
  branch, and let `gate_status_presentation` supply their style. Keep `WORKING PLAN` /
  `WORKING TALE` only if `status-strip` kept the symbol that produces them.
- Keep the branches that are not gate-shell statuses: `STARTING`, `RUNNING`, `SETTLING`,
  `DONE`, `STOPPED`, `FAILED`, `FAILED (RETRIED)`, `RETRYING`, `QUEUED` (with its
  queue-position and slot annotations), and `WAITING` (with its dependency counts and
  countdown).
- Delete the `status_label` plumbing in `src/sase/plan_approval_choices.py`
  (`_PlanApprovalChoiceRecord.status_label`, its four values, and
  `approval_choice_status_label`) together with `plan_approval_status` and
  `plan_approval_choice_for_status`'s status-only use in
  `src/sase/ace/tui/actions/agents/_notification_plan_response.py`, and the two
  optimistic `app._agent_status_overrides[...] = status` writes in
  `_notification_plan_gate.py` and `_notification_modals.py`. The settled gate shell's
  status pair is now what the row shows; verify the answer path still repaints promptly
  without the optimistic write, and if it does not, fix the refresh rather than
  restoring the override. `approval_choice_persist_action` and
  `plan_approval_choice_for_status`'s persist-action use stay.
- Drop `"MONITORED"` from `_TERMINAL_STATUSES` in `src/sase/agent/status_buckets.py`
  once you have confirmed a monitor shell row with the default stop label still buckets
  as `Done` through its recorded `status_bucket`, the same way a custom label such as
  `TESTED` already does. If it does not, leave the literal and note why.
- Rebaseline `tests/ace/tui/visual/snapshots/png/` with `--sase-update-visual-snapshots`
  **only** after inspecting the diffs in `.pytest_cache/sase-visual/` and confirming
  every colour change is one this plan intends. An unexplained hue change means
  `accent-pin` missed a status.

Verification: `just check`, then `just check-full` and `just test-visual` through
`/sase_monitor`.

## wire-v7 — One nested family_shell wire record at schema v7

Independent of every other phase, and explicitly droppable: if the cost lands well above
this estimate, stop, revert cleanly, and record a `PROPOSED FOLLOW-UP:` note on this
phase's bead rather than landing a half-migrated schema.

`AgentMetaWire` carries twenty-seven `monitor_*` fields and thirty `gate_*` fields;
`DoneMarkerWire` carries a smaller terminal subset of both. About nineteen of them are
the same concept under two names — `id`, `state`, `label`, `reason`, `start_status`,
`stop_status`, `timeout_seconds`, `elapsed_seconds`, `output_path`, `output_truncated`,
`request_fingerprint`, `next_action`, `next_output`, `next_model`, `followup_agent`,
`followup_outcome`, `followup_error`, `followup_degraded_reason`, and
`followup_prompt_path`. The rest are kind-specific: `command`, `cwd`, `exit_code`,
`starter_agent`, `tail_lines`, `pgid`, `supervisor_identity`, `settled`, and
`idle_timeout_seconds` for monitors; `kind`, `accent`, `creator_agent`, `next_fork`,
`next_suffix`, `next_role`, `next_raw_prompt`, `workspace_policy`, `bundle_path`,
`notification_id`, and `decision_path` for gates.

- Define one `family_shell` record — a `kind` discriminator, the shared fields, and a
  kind-specific sub-block — in the Rust core (`crates/sase_core/src/agent_scan/wire.rs`,
  with the writer in `agent_scan/scanner.rs` and any reader in `agent_runtime.rs`) and
  in the Python wire (`src/sase/core/agent_scan_wire_markers.py`, with JSON handling in
  `agent_scan_wire_conversion.py`). Bump `AGENT_SCAN_WIRE_SCHEMA_VERSION` to 7 on both
  sides in the same change.
- Keep every existing reader working through a compatibility projection rather than a
  flag-day rename. The Python-side consumers to check are
  `src/sase/core/agent_scan_facade.py`,
  `src/sase/integrations/_agent_list_entry_builder.py`, and
  `src/sase/agents/_wait_live_rows.py`.
- Keep the existing round-trip coverage: `tests/test_core_agent_scan_wire_schema.py`
  asserts the version literal, and the `tests/test_core_agent_scan_wire_*.py` set covers
  the marker projections. Extend, do not replace.
- Run the Rust core's own test suite in the linked repo as well as `just check-full`
  here, and check `tools/check_sase_core_rs_bindings` / `tools/validate_sase_core_rs`
  still pass.
