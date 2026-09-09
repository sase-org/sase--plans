---
tier: epic
status: done
title: Retire the notification and family status overrides
goal: "A plan or question status reaches the Agents tab from exactly one place — the
  gate shell's own recorded start/stop pair and declared accent. The notification-driven
  pending-plan and question overrides, the `_agent_pre_question_status` map, the
  `_agent_status_overrides` re-export facade, the synthetic planner children, and the
  timestamp-reconstruction passes in `apply_status_overrides` are gone, with every
  surviving symbol kept for a stated reason rather than by omission.

  "
phases:
  - id: gate-contract
    title: Pin the post-gate-shell family projection contract
    depends_on: []
    size: small
    description:
      "gate-contract: add guard tests that a family whose newest member is a plan or
      question gate shell renders the gate's status on the container row and `DONE` on
      the planner, for pending and settled branches alike, driven from real gate-shell
      member metadata rather than the override map."
  - id: notification-strip
    title: Retire the notification-driven status writes
    depends_on:
      - gate-contract
    size: medium
    description:
      "notification-strip: delete the pending-plan and `UserQuestion` override writes in
      `_notification_status_overrides.py`, decide the external-response reconciliation
      against the gate executor's own dismissal, remove the now-writerless
      `_agent_pre_question_status` map, and delete the
      `models/_agent_status_overrides.py` re-export facade."
  - id: planner-strip
    title: Retire the synthetic planner children
    depends_on:
      - notification-strip
    size: medium
    description:
      "planner-strip: decide whether a plan family still shows its planner's own work
      without a materialized row, then delete `ensure_synthetic_planner_children`,
      `sync_planner_child_from_parent`, `planner_child_status`, and the
      `is_synthetic_planner` guards that become unreachable with them."
  - id: override-strip
    title: Retire the timestamp-reconstruction status passes
    depends_on:
      - planner-strip
    size: medium
    description:
      "override-strip: delete the `DONE` to `PLAN` / `QUESTION` / `FEEDBACK`
      reconstruction passes and their policy helpers, keep the handoff-labelling symbols
      the gate shell does not replace with a recorded reason for each, drop only the
      `status_buckets` constants that lose their last consumer, and triage every failing
      test into deleted or rewritten."
proposed_by: bbugyi200.athena.sase-ud.13.1.3
parent_bead: sase-ud.13.1.3
bead_id: sase-ud.13.1.3.1
create_time: 2026-09-09 19:51:40
---

- **PROMPT:**
  [prompts/202608/status_strip.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/status_strip.md)
- **PARENT:**
  [202608/gate_shell_status_collapse.md](https://github.com/sase-org/sase--plans/blob/main/202608/gate_shell_status_collapse.md)
- **BEAD:**
  [sase-ud.13.1.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.13.1.3.1.md)

# Plan: Retire the notification and family status overrides

This is phase `status-strip` of `plan:202608/gate_shell_status_collapse.md`, the
sub-epic of `sase-ud.13`. Read that plan's `status-strip` section, its "Design" section
(especially "Strip to what is still reachable is a measurement, not a list" and "Deleted
behaviour needs deleted tests, not weakened ones"), and its risks R-B, R-C, and R-D
before starting a phase here.

The parent plan calls this "the riskiest phase" and prescribes an order: prove the
replacement signal first, delete second. This plan turns that order into phase
dependencies so it cannot be skipped, and records the measurements that were already
taken so no phase has to re-derive them.

## Why this is an epic and not a tale

A tale caps at `medium`, and the measured work does not fit inside one. The deletion
front spans three separate layers with no shared file: the ACE actions layer
(`_notification_status_overrides.py` at 351 lines plus a map with 23 touch points across
12 modules), the model policy layer (`_agent_status_family_policy.py`,
`_agent_status_family_planner.py`, and roughly a dozen passes inside the 514-line
`apply_status_overrides`), and the test surface (about 90 locally failing tests before
counting the wider suite, across ~40 files). Each front also carries its own open
decision — the fate of the external-response reconciliation, the fate of the synthetic
planner row, and the fate of the handoff-labelling symbols — and each of those decisions
needs its own evidence.

The phases are ordered rather than parallel because `notification-strip` deletes the
`_agent_status_overrides` re-export facade whose import list `planner-strip` and
`override-strip` would otherwise have to edit twice, and because `planner-strip`'s
`planner_child_status` calls two of the helpers `override-strip` deletes.

## Measurements already taken

Everything below was measured on the working tree at commit `a646bdaf6` (the
`flag-removal` phase's landing commit). Do not re-derive it; do re-check anything that
looks stale.

### The family node already speaks

Building a plan family from real gate-shell member metadata — a plan-chain root, an
optional concrete planner member, and a gate member with `agent_family_role="gate"`,
`gate_id`, `gate_state`, `gate_start_status`, `gate_stop_status` — and running
`_apply_status_overrides` over it produces exactly the projection the grandparent plan's
§8 table promises:

| Family shape             | Container row   | Planner row | Gate row        |
| ------------------------ | --------------- | ----------- | --------------- |
| pending tale gate        | `TALE`          | `DONE`      | `TALE`          |
| pending epic gate        | `EPIC`          | `DONE`      | `EPIC`          |
| pending question gate    | `QUESTION`      | `DONE`      | `QUESTION`      |
| settled `approve+commit` | `TALE APPROVED` | `DONE`      | `TALE APPROVED` |
| settled `reject`         | `PLAN REJECTED` | `DONE`      | `PLAN REJECTED` |
| settled `feedback`       | `FEEDBACK`      | `DONE`      | `FEEDBACK`      |
| settled question gate    | `ANSWERED`      | `DONE`      | `ANSWERED`      |

R-B does not reproduce. The signal reaches the container through
`_mirror_root_from_child` in `_agent_status_apply.py`, not through
`concrete_agent_statuses` — which does still filter gate shells via
`row_is_family_shell`, exactly as the parent plan warned. The grandparent plan's §8
claim that gate shells "must not be filtered" at `concrete_agent_statuses` is **wrong
about the mechanism and right about the outcome**; do not change that filter.

Re-running the same fixtures with `ensure_synthetic_planner_children`,
`sync_planner_child_from_parent`, `has_unreviewed_submitted_plan`,
`has_unanswered_completed_question`, `superseded_by_feedback_round`, and
`is_awaiting_plan_review` all neutralized produces **byte-identical container, planner,
and gate statuses**. The only difference anywhere is that the synthetic `--plan` member
row stops being materialized.

### What legacy families lose (R-C)

Pre-gate-era artifacts have no gate member. With the same six helpers neutralized:

| Legacy family (no gate member)                | Today          | Stripped                   |
| --------------------------------------------- | -------------- | -------------------------- |
| planner submitted a plan, never reviewed      | `PLAN`         | `DONE`                     |
| planner asked a question, never answered      | `QUESTION`     | `DONE`                     |
| tale-approved root with a running coder child | `WORKING TALE` | `WORKING TALE` (unchanged) |

The first degradation is the one the parent plan explicitly rules acceptable. The second
looks like the case the parent plan rules unacceptable ("a live family losing its
question signal is not"), but it is not: `has_unanswered_completed_question` requires
`agent.status == "DONE"`, so it only ever relabels a _completed_ row. A live blocked
question came from `handle_questions_flow`, which `flag-removal` (`sase-ud.13.1.2`)
already deleted, and a live row with a `pending_question.json` marker still reads
`QUESTION` from the loader, not from this pass. Confirm both claims in `override-strip`
before deleting, and record the confirmation in that phase's bead note.

### Per-pass test blast radius

Measured by neutralizing one helper group at a time and running
`tests/test_agent_loader_status_override_*.py` plus `tests/ace/tui/models/` (835 tests,
all passing at baseline):

| Neutralized                                                            | Failures |
| ---------------------------------------------------------------------- | -------- |
| `ensure_synthetic_planner_children` + `sync_planner_child_from_parent` | 16       |
| `active_approved_plan_handoff_status`                                  | 25       |
| `is_completed_plan_handoff_child` + `is_completed_epic_followup_child` | 12       |
| `approved_followup_planner_status`                                     | 11       |
| `has_unreviewed_submitted_plan`                                        | 10       |
| `has_unanswered_completed_question`                                    | 8        |
| `is_answered_continuation_asker` + `is_answered_root_asker_step`       | 4        |
| `superseded_by_feedback_round`                                         | 2        |
| `is_awaiting_plan_review`                                              | 0        |

`is_awaiting_plan_review` has no independent consumers; it dies with
`has_unreviewed_submitted_plan` and the feedback branch. The wider suite (37,786 tests)
will add more, mostly in `tests/ace/tui/widgets/`.

### Reachability facts

- Every symbol in `_agent_status_family_policy.py` and `_agent_status_family_planner.py`
  has exactly one production consumer: `_agent_status_apply.py`, plus the re-export
  lists in `_agent_status_family.py` and `_agent_status_overrides.py`. Nothing outside
  `src/sase/ace/tui/models/` imports any of them.
- `_agent_status_overrides` (the app-level map) keeps writers this epic does not touch:
  `agent_workflow/_prompt_bar_submit.py:166` writes `"RUNNING"`, and
  `agents/_notification_question_modal.py:339` writes `"ANSWERED"`. **The map itself is
  not deletable in this epic.** The two plan-status writers in
  `_notification_plan_gate.py` and `_notification_modals.py` belong to `ladder-collapse`
  (`sase-ud.13.1.4`).
- `_agent_pre_question_status` has exactly one writer — the `UserQuestion` branch of
  `_apply_notification_status_overrides` — and every other site is a `.pop()` or a
  protocol declaration. No site ever reads the saved status back onto a row. Deleting
  that one writer makes the entire map dead.
- In `status_buckets.py`, only `WORKING_PLAN_STATUS_TO_APPROVED` is already unreferenced
  anywhere in `src/`, `tests/`, or `tools/`. `PENDING_PLAN_REVIEW_STATUSES` (6 other
  modules), `ACTIVE_PLAN_HANDOFF_STATUSES` (9), `APPROVED_PLAN_STATUSES` (2), and
  `WORKING_PLAN_STATUSES` (1) all keep consumers outside the deleted passes.
- `notification_gates/executor._settle_gate_notification` calls `mark_already_handled`
  and `mark_dismissed` for every terminal transition of every gate kind from every
  client, and its docstring says so explicitly.

## Design

**The gate shell is the status.** After this epic the only publisher of a plan or
question status is the gate shell's recorded start/stop pair plus its declared accent,
projected by `gate_status_presentation` and mirrored onto the container by
`_mirror_root_from_child`. Nothing reconstructs a status from artifact timestamps,
unread notifications, or synthesized rows.

**Keeping a symbol is a legitimate outcome, but only with a reason.** The parent plan's
"What this deletes" table is the author's estimate, and the grandparent's is stale in at
least two places: it lists `active_approved_plan_handoff_status` and
`done_handoff_status` as deletions, but both label rows the gate shell does not replace
(the gate settles at `TALE APPROVED` and only then launches the coder, which then needs
`WORKING TALE` and `TALE DONE`). Every symbol named in this plan gets a per-symbol
verdict recorded in its phase's bead note — deleted because its inputs can no longer
occur, or kept because they can, and why.

**Deleted behaviour needs deleted tests, not weakened ones.** A test asserting an
override this epic retires is evidence of the retired world and is deleted with it. A
test asserting a projection the gate shell still owes the user is rewritten to drive it
through a gate-shell fixture. Never neutralize an assertion to make a suite pass, and
never delete a test whose subject survives.

**Additive first.** `gate-contract` adds the guard tests before anything deletes, so a
regression in the replacement signal fails loudly at the phase that caused it rather
than at review time.

## Risks

- **R-1 — A kept symbol is deleted anyway.** `active_approved_plan_handoff_status`,
  `is_completed_plan_handoff_child`/`done_handoff_status`,
  `is_completed_epic_followup_child`, and `approved_followup_planner_status` label
  concrete follow-up rows, not reconstructed planner state. Deleting them regresses the
  post-approval half of every plan family with no gate-shell test to catch it, because
  the gate is already settled by then. `gate-contract` adds the settled-gate-plus-coder
  cases specifically to make that failure loud.
- **R-2 — The synthetic planner row is load-bearing for something other than status.**
  It carries `artifacts_dir`, `response_path`, `diff_path`, and `extra_files` copied
  from the root, so it may be the row through which the file/chat panels reach the
  planner's own artifacts. `planner-strip` must check that before deleting, not after.
- **R-3 — A `pop()` without a writer looks harmless and is not.** The
  `_agent_pre_question_status` removal touches kill, dismiss, marking, and finalize
  paths. A partial removal that leaves a protocol declaration behind fails Symvision;
  one that removes a `pop()` from a shared helper without removing the map fails at
  runtime. Remove the writer, then the map, then every declaration, in one change.
- **R-4 — Overlap with `ladder-collapse`.** `sase-ud.13.1.4` owns
  `_agent_list_render_agent_status.py`, `plan_approval_choices.py`, the optimistic
  `_agent_status_overrides` writes in `_notification_plan_gate.py` and
  `_notification_modals.py`, and `"MONITORED"` in `_TERMINAL_STATUSES`. No phase here
  touches any of them.
- **R-5 — Pre-existing red.** `just check-full` currently fails at
  `tools/check_test_cost_budgets` on master; that is task bead `sase-j0`, actively
  tracked and unrelated. Confirm any check-full failure is that one before treating it
  as this epic's.

## Working agreements for every phase

- `just check` before replying. `just check-full` — and `just test-visual` for any phase
  that changes which rows a family emits — through the `/sase_monitor` skill, never
  inline.
- Do not edit `sase/memory/`; epic phase `sase-ud.14` owns the memory and decision
  record.
- Record discovered follow-up work as `PROPOSED FOLLOW-UP:` notes on your own phase
  bead. Do not create beads. Do not close any ancestor bead.
- Record every per-symbol verdict — deleted or kept, and why — in your phase's bead
  note. That note is the evidence the parent epic's land agent needs.

---

# Phases

## gate-contract — Pin the post-gate-shell family projection contract

Purely additive. Nothing is deleted in this phase.

Add a test module (suggested:
`tests/test_agent_loader_status_override_gate_shell_family.py`) that builds plan and
question families from real gate-shell member metadata and asserts the projection
through `_apply_status_overrides`. Prefer driving the gate row's fields through
`enrich_agent_from_meta_wire` with a `FamilyShellWire`/`FamilyShellGateWire` payload —
the shape `tests/ace/tui/models/test_gate_rows.py` already uses — so the guard covers
the real wire projection rather than hand-assigned attributes. Never build a fixture
from the `_agent_status_overrides` map.

Cases, all of which pass on today's tree:

- A pending tale gate as the family's newest member: container `TALE`, planner member
  `DONE`, gate row `TALE`.
- The same for a pending epic gate (`EPIC`) and a pending question gate (`QUESTION`).
- A settled gate rendering its branch status on the container: `TALE APPROVED`,
  `FEEDBACK`, `PLAN REJECTED`, and `ANSWERED` for the question gate.
- A settled gate followed by a **running** coder child: container and coder both
  `WORKING TALE`. This is R-1's guard — it is the case that fails if a later phase
  deletes `active_approved_plan_handoff_status`.
- The same with a **completed** coder child: container and coder both `TALE DONE`. This
  is the guard for `is_completed_plan_handoff_child` and `done_handoff_status`.
- The container's mirrored gate pair — `gate_start_status`, `gate_stop_status`,
  `gate_state`, `gate_accent` — equals the gate row's, so `ladder-collapse` can rely on
  `gate_status_presentation` reaching the container.

If any case fails, that failure is this phase's actual work: find where the gate's
status is dropped and fix it. Do not weaken the case to match the tree.

Verification: `just check`.

## notification-strip — Retire the notification-driven status writes

**Delete the status writes.** In
`src/sase/ace/tui/actions/agents/_notification_status_overrides.py`, remove the
pending-plan and `UserQuestion` override branches of
`AgentNotificationStatusMixin._apply_notification_status_overrides`, together with the
`pending_plan_status` computation, the `PENDING_TALE_STATUS`/`PENDING_EPIC_STATUS`
imports, the `pending_plan_status_for_agent` call, and the `changed_agents` refresh loop
if nothing else feeds it. The gate shell already publishes `TALE`/`EPIC`/`QUESTION` on
both the gate row and the container; these writes paint the same value on top.

**Decide the external-response reconciliation, then act.** Split
`prepare_plan_notification_reconciliation` / `_prepare_external_plan_response` by bundle
kind rather than deleting wholesale:

- The **neutral** (gate-bundle) branch is a notification-lifecycle duplicate:
  `notification_gates/executor._settle_gate_notification` already marks handled and
  dismisses for every terminal transition of every gate kind from every client,
  including Telegram and mobile. Default verdict: delete it, and prove the deletion with
  a test that an externally settled gate's notification still disappears from ACE's
  inbox.
- The **legacy** branches — `plan_approved.marker`, `persist_plan_approved`, and the
  vanished-request `clear_override` case — serve pre-gate on-disk bundles that the
  executor never sees. Default verdict: keep them. If a check of
  `resolve_notification_bundle` shows no reachable legacy producer remains, deleting
  them is also acceptable; say which and why in the bead note either way.
- `_auto_dismiss_external_plan_response` is a synchronous wrapper for non-poller callers
  and tests. Keep or delete it with whatever it wraps.

**Delete `_agent_pre_question_status`.** Once the `UserQuestion` branch is gone the map
has no writer. Remove, in one change: the initialization in
`actions/_state_init_agents.py`, every protocol declaration (`agents/_core.py`,
`_marking.py`, `_dismissing.py`, `_loading_state.py`, `_killing.py`,
`_notifications.py`, `_dismiss_memory.py`, `_kill_identity.py`), and every `.pop()`
(`_marking.py`, `_dismissing.py` ×2, `_notification_question_modal.py`,
`_notification_plan_gate.py`, `_loading_finalize.py` ×3, `_kill_flow.py`,
`_kill_identity.py`). Re-confirm before deleting that no site reads the saved status
back onto a row.

**Do not delete the `_agent_status_overrides` map.** It keeps two writers this epic does
not retire (`_prompt_bar_submit.py`'s `"RUNNING"`, `_notification_question_modal.py`'s
`"ANSWERED"`) plus two that belong to `sase-ud.13.1.4`. Leave
`should_clear_loaded_agent_status_override` in `agents/_loading_helpers.py` alone unless
a clause is provably unreachable after this phase.

**Delete the model facade.** Remove `src/sase/ace/tui/models/_agent_status_overrides.py`
and repoint its importers at the real modules: `models/agent_loader.py` takes
`apply_status_overrides` from `_agent_status_apply`, `is_coder_followup_suffix` and
`is_feedback_suffix` from `_agent_status_roles`, and `is_root_plan_workflow` from
`_agent_status_family`. Preserve the facade's one piece of behaviour — it passes
`diff_badge_classifier=_classify_diff_badges` — by checking that
`_agent_status_apply.apply_status_overrides` still defaults to
`classify_persisted_diff_badges`. Three test references need the new seam:
`tests/test_plan_inventory_scanning.py:367` patches
`...models._agent_status_overrides._classify_diff_badges` (the equivalent is
`_agent_status_apply.classify_persisted_diff_badges`),
`tests/test_agent_loader_status_override_plan_entry.py:8`, and
`tests/test_agent_loader_live_file_change_hint.py:22`.

**Tests.** `tests/ace/tui/test_agent_notification_status_overrides.py` (557 lines, 10
tests) and `tests/_notification_toasts_helpers.py` are the direct coverage; triage each
case against the delete/rewrite rule. Twenty-three test files declare
`_agent_pre_question_status` on fake app objects — that removal is mechanical, but read
each one rather than sed-ing it, because several also assert override behaviour.

Verification: `just check`, then `just check-full` through `/sase_monitor`.

## planner-strip — Retire the synthetic planner children

**Answer R-2 first.** With `ensure_synthetic_planner_children` neutralized, a modern
plan family renders container + gate + coder and nothing else; the `--plan` member row
disappears while every status stays identical. Before deleting, establish whether that
row is the only way the Agents tab reaches the planner's own artifacts, response, diff,
and chat. Two facts point at "no": `_root_represents_member` in
`models/agent_family_members.py` already makes a plan-family root its own
`container_proxy` in `_family_shell_anchors`, and the synthetic row's `artifacts_dir`,
`response_path`, `diff_path`, and `extra_files` are copies of the root's. Confirm it in
the family-row projection and, if a panel does lose access, keep
`ensure_synthetic_planner_children` and record why in the bead note — that is a
legitimate outcome and the phase then reduces to deleting `planner_child_status`'s
unreachable branches.

**On the delete verdict**, remove from `models/_agent_status_family_planner.py`:
`ensure_synthetic_planner_children` and `sync_planner_child_from_parent`; and from
`models/_agent_status_family_policy.py`: `planner_child_status`,
`answered_asker_freeze_time`, `_approved_planner_status`,
`_approved_epic_planner_status`, and `EPIC_CREATED_STATUS` if
`is_completed_epic_followup_child`'s literal is its last consumer. Keep
`copy_missing_plan_metadata`, `copy_missing_display_metadata`, and
`pull_plan_metadata_from_family_members` — they serve root mirroring, not planner
materialization. Prune the two synthetic-planner branches of `apply_status_overrides`
and the comment above `mark_derived_plan_family_roots` that explains the ordering
constraint between them, which stops being true.

**Then chase `is_synthetic_planner`.** With no producer the field is permanently
`False`, so every guard reading it is dead: `_is_excluded_family_shell`,
`_concrete_planner_child`, and `_concrete_continuations` in
`models/agent_family_members.py`, the `step_type == "agent"` filters in
`_concrete_agent_rows`, and the field on `models/agent.py`. Let `just lint`'s Symvision
stage find the tail; read `sase memory read symvision.md` first if you have not. Nine
test files reference the field.

**Tests.** Sixteen tests in `tests/test_agent_loader_status_override_*.py` and
`tests/ace/tui/models/` fail when this group is neutralized, concentrated in
`test_agent_loader_status_override_promoted_plan_family.py`,
`tests/ace/tui/models/test_agent_family_members.py`, and
`tests/ace/tui/models/test_agent_tree_rendering.py`; the widget suite adds more in
`test_agent_list_runtime_ordering.py`, `test_agent_render_cache.py`,
`test_agent_display_kind_headers.py`, `test_agent_display_clan.py`,
`test_agent_render_cache_patching.py`, and `test_agent_display_model_fields.py`. A test
that asserts a synthetic row exists is deleted with it; a test that asserts a family's
visible row order or status is rewritten against the concrete rows.

Verification: `just check`, then `just check-full` and `just test-visual` through
`/sase_monitor`. This phase changes which rows a family emits, so the PNG goldens must
be inspected in `.pytest_cache/sase-visual/` before any rebaseline — and a colour change
here means something is wrong, because this phase changes no accents.

## override-strip — Retire the timestamp-reconstruction status passes

**Delete**, after confirming each one's inputs can no longer occur:

- `has_unreviewed_submitted_plan` and its `DONE` → `PLAN`/`TALE`/`EPIC` pass.
- `is_awaiting_plan_review` (no independent consumer once the above and the feedback
  branch of `apply_status_overrides` are gone).
- `has_unanswered_completed_question` and its two `DONE` → `QUESTION` passes, plus the
  `has_inherited_family_question` guard that exists only to qualify them. Before
  deleting, confirm on the tree that a live blocked question cannot reach this pass: it
  requires `agent.status == "DONE"`, the blocking `handle_questions_flow` was deleted by
  `sase-ud.13.1.2`, and a live `pending_question.json` row gets `QUESTION` from the
  loader instead. Record the confirmation in the bead note.
- `superseded_by_feedback_round`, `_is_planner_family_row`, and the `FEEDBACK` pass —
  the gate's `feedback` branch publishes `FEEDBACK` directly.
- `feedback_child_progressed_past_review` and `pending_plan_status_for_agent` if the
  above were their last consumers.
- `WORKING_PLAN_STATUS_TO_APPROVED` in `src/sase/agent/status_buckets.py`, which is
  already unreferenced everywhere.

**Verify, and keep unless the evidence says otherwise** — each of these labels a
concrete follow-up row that the gate shell does not replace, and `gate-contract`'s
settled-gate cases cover them:

- `active_approved_plan_handoff_status` (`WORKING PLAN` / `WORKING TALE` /
  `EPIC APPROVED` / `PLAN COMMITTED` on a **running** coder child).
- `is_completed_plan_handoff_child` and `done_handoff_status` (`PLAN DONE` / `TALE DONE`
  on a completed one).
- `is_completed_epic_followup_child` (`EPIC CREATED`).
- `approved_followup_planner_status` (the sticky approved status on a concrete follow-up
  planner).
- `is_answered_continuation_asker` and `is_answered_root_asker_step` (`ANSWERED` on the
  asker row that handed off) — these are not in the parent plan's expected-deletion
  list; treat keeping them as the default.

**Do not touch** `PENDING_PLAN_REVIEW_STATUSES`, `ACTIVE_PLAN_HANDOFF_STATUSES`,
`APPROVED_PLAN_STATUSES`, or `WORKING_PLAN_STATUSES`: each keeps consumers in
`_revive_artifacts.py`, `_loading_helpers.py`, `agent_time.py`,
`zoom_panel_rendering.py`, `_workflow_loaders.py`, `agent_detail.py`,
`file_panel/_diff.py`, `_mobile_agent_summary.py`, or `event_refresh/_constants.py`.
`"MONITORED"` in `_TERMINAL_STATUSES` belongs to `sase-ud.13.1.4`.

**Prune `apply_status_overrides` and keep its docstring honest.** Its compatibility
paragraph documents `DONE -> QUESTION` and `DONE -> PLAN` for non-family agents; both
stop being true here. Rewrite it to describe what the pass still does: family containers
mirror their newest member, timestamps and metadata propagate to the root, and follow-up
handoff rows get their semantic labels.

**Tests.** Twenty locally failing tests when this group is neutralized
(`has_unreviewed_submitted_plan` 10, `has_unanswered_completed_question` 8,
`superseded_by_feedback_round` 2), spread across
`test_agent_loader_status_override_plan_entry.py`, `_feedback.py`, `_questions.py`,
`_question_families.py`, `_question_continuations.py`, `_followup_*.py`, and `_tale.py`.
Classify every one: a test asserting a reconstructed status is deleted; a test asserting
a projection the gate shell still owes is rewritten against a gate-shell fixture built
the way `gate-contract` builds them. Thirty-two test files mention `WORKING PLAN` /
`WORKING TALE`, but that status survives — expect most of them to be untouched, and
treat a failure in one as a signal that a kept symbol was deleted.

Verification: `just check`, then `just check-full` and `just test-visual` through
`/sase_monitor`.
