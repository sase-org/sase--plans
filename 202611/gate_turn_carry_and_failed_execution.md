---
tier: epic
title: 'Gate turns: explicit carry-vs-commit and visible accepted-but-failed executions'
goal: 'Agents staging inheriting gate turns know dirty work is carried uncommitted
  unless they declare commit, and a gate whose accepted option command failed is visibly
  failed and recoverable on every surface instead of silently parking in a Running
  "settling" state.

  '
phases:
- id: carry-contract
  title: Make carry-vs-commit explicit for inheriting handoffs
  depends_on: []
  size: medium
  description: 'carry-contract: add a carry-default hint to sase final context and
    the staged gate descriptor, and fix the sase_gate/sase_final skills and docs.'
- id: failed-gate-attention
  title: Accepted-but-failed gate turns project as needing a human
  depends_on: []
  size: medium
  description: 'failed-gate-attention: persist execution-failure facts on the gate
    member, project them with failure status and recovery commands in CLI and ACE,
    and optionally resurface dismissed failures.'
decisions:
  failed_bucket:
    ask: Should an accepted-but-failed gate turn stay in the awaiting-human bucket
      rather than Failed?
    choices:
      waiting: Same bucket as a pending gate plus a red failure glyph; cleanup keeps
        treating it as live
      failed: Failed bucket; louder, but cleanup flows must be audited to skip claim-holding
        gates
    default: waiting
    why: The gate is recoverable and still owns its workspace, so it must not look
      terminal
    answer: waiting
  resurface:
    ask: Should reclaim resurface a dismissed gate-failure notification once while
      the failure is current?
    choices:
      once: Resurface it once, an hour after dismissal, while the gate still holds
        its workspace
      never: Row-level attention only; a dismissed failure stays dismissed
    default: once
    why: The inbox is the attention channel, and a failed gate otherwise holds its
      workspace forever
    answer: once
proposed_by: bbugyi200.athena.1et
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 09:19:50
status: wip
bead_id: sase-1ut
---

- **PROMPT:** [prompts/202611/gate_turn_carry_and_failed_execution.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/gate_turn_carry_and_failed_execution.md)
- **BEAD:** [sase-1ut](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1ut/README.md)

# Gate turns: explicit carry-vs-commit, and visible accepted-but-failed executions

## Context: why `bob-cli-oj.1` stalled

Agent session `bob-cli-oj.1` (project bob-cli) is stuck, and nothing in SASE will move
it forward:

1. Agent `bob-cli-oj.1--1` verified a restored `bob-navigation-hotkeys` 2.96.1 in the
   linked `bob-plugins` repo. It then staged a custom **gate turn**
   (`custom-231911ba-9d6a-4c7f-83d1-5911bcf25361`, `workspace: inherit`). The gate's
   `replace` option command deploys the plugin, but only after checking
   `git status --porcelain` is empty ("Source must be committed by the finalizer before
   deployment").
2. After staging, the agent ran `sase final context -f json` and meant to submit
   `action: "commit"`. Because the staged handoff was `carry_allowed`, the context
   returned `submission_required: false`. Following `/sase_final` ("submit a declaration
   only when its context requires one"), the agent skipped the submission. The host then
   synthesized `{"action": "carry"}` for the dirty `bob-plugins` repo
   (`build_context_requirements` → `omit_submission` in
   `src/sase/finalizers/declaration_store.py`; synthesis is covered by
   `tests/test_finalizer_declaration_channel_staged_handoff.py`). The restored source
   moved into the gate turn **uncommitted**.
3. Bryan approved `replace` from the TUI. The option command ran under the durable proc
   and failed on the dirty tree (`command_failed`, stage `command`). The bundle journal
   records `attempt_failed` and `option_failed`.
4. The gate lifecycle classifies this as `accepted_failed`. `reclaim_pending_gate_turns`
   (`src/sase/gate_turn/_reclaim_pending.py`) intentionally leaves it parked for human
   recovery, and an accepted receipt outranks the 24h deadline, so it never expires.
   Meanwhile:
   - The member meta still says `gate_state: "settling"` with the dead answering pid.
   - `sase gate list` shows status `APPROVE DEPLOY` in the **Running** bucket,
     `followup_needs_attention: false`, and `holds_workspace_claim: true`. It reads as
     "still awaiting approval" or "still running", not as "failed, needs you".
   - The only recovery surface, the `GateExecutionFailed` notification (resume / restart
     / cancel commands), is **dismissed** (unread). No dismissal provenance was
     recorded, so nothing points a human back at the gate.

Two defects combine here:

- **A — Carry is silent.** For inheriting gate turns, the skills and the final-context
  JSON never say that omitting the declaration carries dirty work _uncommitted_ into the
  gate and its option commands. An explicit `commit` is legal and preserved in that case
  (`test_existing_explicit_commit_is_preserved_without_synthesis`). The `/sase_final`
  skill documents this only for monitors ("An unprepared inheriting monitor may instead
  receive host-synthesized carry…").
- **B — Accepted-but-failed gates are invisible.** Once the failure notification is
  gone, no projection, status, glyph, or bucket says the gate failed and needs resume /
  restart / cancel. It keeps holding its workspace claim indefinitely.

Out of scope: recovering `bob-cli-oj.1` itself. That is a one-time operational action in
another project's workspace (commit the carried `bob-plugins` tree, then
`sase gate answer --kind custom --id custom-231911ba-9d6a-4c7f-83d1-5911bcf25361 --option replace --resume`)
and is handled outside this plan.

## Phase `carry-contract` — make carry-vs-commit explicit for inheriting handoffs

- size: medium
- depends on: none

Goal: an agent that stages an inheriting gate turn learns at decision time that dirty
repos will be carried uncommitted unless it declares `commit`, and how to opt in.

1. **Final context hint (Python only).** In `build_commit_declaration_context` (called
   from `src/sase/finalizers/_declaration_context.py`), when the handoff is
   carry-allowed and the commit requirement's submission was omitted, add
   machine-readable fields under `commit_declaration`. For example:
   - `omitted_submission_default: "carry"`
   - `carried_repo_ids: [...]`
   - a `rule`-style sentence: committing is still allowed, and it is required when the
     staged gate's option commands or the successor need committed source.

   Keep `manifest_template` populated with `commit` decisions so opting in stays one
   submit. Render the same hint in `src/sase/finalizers/declaration_format.py` (pretty
   output) next to "Allowed repository actions". `submission_required` semantics do not
   change. This is presentation and context data, not finalizer policy; no `sase_core` /
   wire change is needed. If the field turns out to be wire-typed, stop and keep the
   hint in the Python-built `commit_declaration` dict only.

2. **Staged gate descriptor notice.** When `sase gate create --turn` returns a `staged`
   descriptor for a `workspace: inherit` turn and the creator's workspace has dirty
   repository obligations, include a `carry_notice` in the descriptor JSON (and the
   human output). It names the dirty repos and says: "these will reach the gate
   uncommitted unless you submit a `commit` declaration through `/sase_final`; option
   commands that require committed source will otherwise fail." Find the staged
   descriptor builder from `src/sase/gate_turn/staged_contract.py` and its CLI caller.
   Computing dirty state must reuse the existing finalizer dirty-state helper, not a new
   git scan.
3. **Skills.** First read `sase memory read generated_skills.md` and follow it for
   editing the skill sources and regenerating or deploying.
   - `src/sase/macros/skills/sase_gate.md`: in the option-command environment paragraph
     (the one that introduces `SASE_GATE_WORKSPACE_DIR`), state that an `inherit` gate
     turn receives your dirty work _uncommitted_ (host carry) unless you submit an
     explicit `commit` declaration. Commands that deploy, sync, publish, or land
     committed source need that explicit commit.
   - `src/sase/macros/skills/sase_final.md` "Handoff Commands": replace "submit a
     declaration only when its context requires one" with the precise rule. Omitting it
     lets the host carry dirty repos uncommitted into an inheriting successor or gate.
     Submit an explicit `commit` (still accepted) when the staged gate's commands or the
     successor need committed source. Read
     `commit_declaration.omitted_submission_default`. Keep Step 2's loop wording
     consistent with this.
4. **Docs.** Read `sase memory read docs_writing.md` first. Update the gate-turn /
   finalizer reference (`docs/reference/automation/notification-gates.md` and the
   finalizer or monitor doc that already describes host-synthesized carry) with the same
   rule.
5. **Tests.**
   - Final-context JSON and pretty output include the hint only for carry-allowed
     handoffs with dirty repos.
   - The staged gate descriptor includes `carry_notice` for inherit plus dirty, and
     omits it for `release` or clean.
   - The existing explicit-commit-preserved test still passes.

## Phase `failed-gate-attention` — accepted-but-failed gate turns project as needing a human

- size: medium
- depends on: none

Goal: a gate turn whose accepted option failed before publishing a response is visibly
failed and recoverable on every surface, independent of the notification inbox.

1. **Persist failure facts on the member.** Where the executor records a pre-response
   failure outcome for a gate-turn bundle (`record_failure_outcome` /
   `recorded_attempt_failure` in `src/sase/notification_gates/failure_outcome.py` and
   `executor.py`), also write a small `gate_execution_failure` object into the gate
   member's `agent_meta.json`. Include stage, code, message, error report path,
   acceptance/attempt ids, and the resume / restart / cancel commands from
   `gate_failure_action_data`. Clear `pid` / settling ownership at the same time, so the
   member no longer claims a live settler.

   Use the existing meta update helpers and the gate-turn lane lock conventions. Clear
   the object at every point that already calls `dismiss_gate_execution_failed`:
   - resume success,
   - supersede,
   - cancellation,
   - successful side effects.

   Also clear it on terminal settlement. For gates that failed before this change,
   `reclaim_pending_gate_turns` backfills the object when it sees `accepted_failed` or
   `accepted_owner_lost` and the meta lacks it.

2. **Projection.** In `src/sase/gate_turn/projection.py` (`gate_turn_runtime_json`, the
   shared projection for `sase gate list/show/cancel` and ACE), add:
   - `execution_failure` (the persisted object or null),
   - `needs_attention` (true when `execution_failure` is set or the existing follow-up
     attention fires),
   - `recovery_commands`.

   `followup_needs_attention` and `followup_resume_command` keep their current meaning.
   Hot paths must read the persisted meta object, not re-open the bundle or run the
   lifecycle classifier per row (read `sase memory read tui.md` for the performance
   rules).

3. **Status, glyph, bucket.** In `src/sase/gate_turn/status.py` and `state.py`, a
   non-terminal member with `execution_failure`:
   - shows the failure glyph and style (`✗`, `GATE_STATUS_FAILURE_STYLE`),
   - gets a label that makes the failure explicit (for example `<pending_status>` plus a
     failure marker; keep within `GATE_TURN_STATUS_MAX_CHARS`).

> [!decision] failed_bucket = waiting `gate_member_status_bucket` returns the same
> awaiting-human bucket as `pending` (currently `Stopped`) instead of `Running`.

> [!decision] failed_bucket = failed Return `Failed`. Audit the failed-row cleanup,
> archive, and dismiss flows so they skip non-terminal gate turns that still hold a
> workspace claim.

4. **CLI.** `src/sase/main/gate_turn_render.py` and `notification_gates/cli_show.py`
   render the failure line and the recovery commands on list and show rows, styled like
   the existing follow-up attention marker.
5. **ACE.** `src/sase/ace/tui/widgets/prompt_panel/_agent_gate_section.py` (and the
   agent row status) shows the failure and the recovery commands. If an existing action
   already opens `handle_gate_execution_failed` for a notification, reuse it from the
   gate member row so recovery does not depend on finding the dismissed notification. Do
   not build a new modal.

> [!decision] resurface = once In `reclaim_pending_gate_turns`, for `accepted_failed` /
> `accepted_owner_lost` gates whose `GateExecutionFailed` notification was dismissed
> more than an hour ago and never resurfaced, resurface it once through the notification
> store's existing resurface / un-dismiss path. Record the resurface in the member meta
> so it happens at most once per acceptance.

> [!decision] resurface = never No inbox changes. Row-level attention is the only signal
> after dismissal.

6. **Docs.** Read `docs_writing.md` first. Update
   `docs/reference/automation/notification-gates.md` (gate-turn lifecycle / recovery
   section) to describe the accepted-failed state, how it projects, and the recovery
   commands.
7. **Tests.** A fixture gate turn whose accepted option command fails:
   - the projection has `execution_failure` and `needs_attention`, the bucket follows
     the decision, and the label and glyph show failure;
   - `--resume` success and `sase gate cancel` clear it;
   - the reclaim backfill fills it for an old-shape member;
   - with `resurface = once`: exactly one resurface after the delay and none before it.

## Verification (both phases)

Read `sase memory read lint_and_test.md` and run the verification it prescribes
(`just check` through the guarded tool path). Do not run `just check-full`.
