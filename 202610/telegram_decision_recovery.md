---
tier: tale
title: Finish Telegram Plan Decisions receipts and recovery
goal:
  Telegram reviews retain recoverable drafts after rejected submissions and settle into
  truthful receipts on the original card.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.10.7.5
bead: sase-1hi.10.7.5
create_time: 2026-10-08 16:44:23
status: wip
---

- **PARENT:**
  [202610/plan_decisions_landing_finish.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_finish.md)
- **BEAD:**
  [sase-1hi.10.7.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.7.5.md)

# Finish Telegram Plan Decisions receipts and recovery

## Scope and authority

Implement the assigned phase **sase-1hi.10.7.5** of epic **sase-1hi.10.7**. The phase is
already reserved and in progress; never set its status manually. This is one bounded
implementation in the linked **sase-telegram** repository. Open it with
`sase repo open sase-telegram -r "Implement phase sase-1hi.10.7.5"` and read the printed
checkout's `AGENTS.md` before editing. Use only that checkout.

Read these audited sources for the final design and any newly landed changes:

- `sase bead read sase-1hi.10.7.5 -r "Need phase scope and final epic decisions"`.
- `sase artifact read plan:202610/plan_decisions_landing_finish.md "Need Telegram phase requirements"`,
  Section 5.
- `sase artifact read plan:202610/plan_decisions.md "Need exact Telegram UI contract"`,
  Sections 1.3, 1.5, 1.6, and 6.7.
- `sase artifact read plan:202610/plan_decisions_landing_repairs.md "Need prior repair contract"`,
  Section 6.

The epic's DECISIONS are final. It authorizes no memory edits. Keep decision API imports
on the feature-detected `sase.sdd.plan_decisions` facade. The work here is Telegram
presentation and transport orchestration; reuse host acceptance, error, and launch
facts. Do not independently resolve decisions or move shared backend behavior into this
plugin. No core binding change is expected.

## Confirmed starting behavior

- `decision_receipt.decider_surface` returns surfaces such as `via Telegram`, but reject
  and feedback headers insert another `via`. Auto maps to `(auto, auto)` and approval
  prints both. Header provenance is resolved only after the early reject/feedback
  returns.
- `gate_completions._settle_decision_receipt` restores stale controls without sending
  the stale explanation, ignores other durable pre-response errors, deletes progress on
  missing-response reports, and waits indefinitely when the proc row is absent.
  Successful receipt edits discard the completion job even if launch side effects are
  unfinished, and leave the pending action for a later sweep that edits the card again.
- `text_messages._handle_text_message` removes pending action and gate progress as soon
  as a feedback reply is submitted. Completion then loses the review message id when
  `source_message_id` is null.
- `_disable_decision_controls` writes a keyboard retry and immediately deletes it,
  including after a failed edit.
- `decision_sheet` adds a fourth compaction that drops why and quotes; its final
  expandable candidate omits memory details and wraps the entire sheet.
- Existing tests mostly check helper vectors and the existence of labels. They do not
  pin keyboard rows, execute the three single-verdict callback paths, or exercise the
  actual completion/sweep lifecycle and delayed errors.

## Implementation

### 1. Render one provenance phrase on every outcome

Normalize decider, surface, and response time before branching on approval, reject, or
feedback in `decision_receipt.py`. Share the attribution formatting across outcomes:
`you via Telegram`, `you via ACE`, `you via CLI`, and `you via mobile`; retain recorded
agent attribution. Auto appears once and uses the parent plan's auto-approval wording
for Tale or Epic. Explicit arguments and durable response facts must not create doubled
words.

Preserve author order, changed markers and `(★ default)`, toggle yes/no, provisional
feedback wording, the true coder/commit/epic summary, and timestamps. Use authoritative
accepted vectors rather than a Telegram draft.

### 2. Recover asynchronous rejected submissions without losing the review

In `gate_completions.py`, read the newest submission-relevant durable `errors/*.json`
before falling back to a proc-only diagnostic. Use its structured code and message; the
already-landed gate phase records `stale_review` with the current revision and no
`response.json`.

For stale rejection, send exactly `This plan changed since this card was shown.`,
restore `↻ Refresh review` on the original review card, and keep pending action,
displayed revision, decision draft, and review message/chat context. The refresh token
uses the current request revision; its handler must reload prose and sheet and preserve
only still-valid draft values by decision id. Other pre-response rejections report their
recorded message and leave the review usable for retry.

Introduce an explicit finite grace interval for a missing proc row or a terminal proc
lacking durable output. A still-running visible proc continues waiting. After the grace
interval, report the missing output once and restore usable review controls with the
draft retained. Use an injected/frozen clock in tests, never sleep. A failed Telegram
report or keyboard restoration keeps a durable retry; remove the submission completion
record only once recovery was delivered. Clear any pending keyboard-removal retry when
restoring controls so the next cleanup tick cannot erase Refresh.

### 3. Keep card settlement and launch completion coordinated

Preserve the pending action and progress for submitted decision-plan feedback in
`text_messages.py`; only clear the matched awaiting-feedback entry immediately. Generic
gate behavior stays as it is. Persist the review message id in the completion record as
well as progress, including when source_message_id is null, so receipt retries and
restarts never accidentally edit the feedback reply.

Coordinate `gate_completions.py` and `keyboard_cleanup.py` around one durable receipt
job per submission/review. External settlement from ACE or CLI uses the same rendering
and edit semantics. Successful acceptance edits the card once and removes its keyboard;
a completion reply carries the full summary. Persist which text was successfully edited
and whether the reply was sent, so polls and cleanup sweeps do not duplicate either.
Treat Telegram's already-identical response as a successful edit. Clear pending action
and feedback state after the card edit succeeds, while retaining any job still observing
unfinished side effects.

Do not finalize launch observation merely because response.json exists. The host writes
`journal.jsonl` stage_started/stage_completed/attempt_failed rows for side_effects and
follow_up, and `errors/*.json` has stage/attempt/outcome identities. Coder follow-up
launch errors can be tolerated by shell settlement and recorded as `gate_followup_error`
in gate-turn metadata rather than raised as a journal failure. Inspect the matching host
outcome/metadata with existing host readers and feature detection; do not match
arbitrary error-message words or the nonexistent `successor_launch_failed` code.

Only a recorded failure of the selected coder launch earns
`Approved with these choices · coder could not start · retry`. Commit-only, reject,
feedback, unrelated execution failures, and a running proc never earn that claim. If
failure arrives after the first receipt edit, update that same card once while keeping
the accepted choices immutable. Retain the job until the relevant launch stage is
complete or its recorded failure is delivered; bound missing-outcome observations and
report uncertainty without inventing a launch failure. Cover external settlement as well
as Telegram submission.

Use the existing durable keyboard-cleanup helper when disabling submitted controls:
persist before editing and clear only on success. Recovery or a settled receipt cancels
that disable retry. Clear accepted pending actions so the post-poll sweep cannot perform
a second settlement edit.

### 4. Apply the parent sheet compaction order exactly

Refactor `decision_sheet.py` around whole escaped fields/lines and three ordered
degradations only: drop labels on non-default choices; drop the remaining choice labels;
wrap only the choice lines in real expandable MarkdownV2 blockquotes. Keep asks outside
quotes. Preserve every starred default line, why, memory selector/type/new/provenance
chip, and requested quote through every stage. Remove compact_detail and the fallback
that discards memory details.

Use the existing blockquote helper on escaped choice blocks, not on the assembled sheet.
Compose the result with the protected-sheet path in `formatting.py`. Never slice an
escaped field or cut a decision in half. Exercise the approximately 1,800-character
budget with valid bounded fixtures and assert the overall review stays within Telegram's
4,096-character limit. Expandable markup does not shrink serialized text: if protected
content alone exceeds the compaction trigger, preserve it rather than adding an
unauthorized fourth degradation. Record any unavoidable design-limit conflict as
PROPOSED FOLLOW-UP on this phase, with a valid reproducer, without changing shared
grammar or dropping required fields.

### 5. Pin the interface and exercise routes, not just helpers

Extend or split `tests/test_plan_decisions.py` into focused modules where useful, using
real `build_plan_approval_gate_spec`/`create_gate` bundles and isolated
pending/progress/completion/cleanup stores. Patch Telegram network operations and host
launch/archive side effects, not the callback, normalization, or settlement logic being
verified. Deferred-proc fixtures must return before execution so asynchronous
durable-error tests do not pass only through the synchronous fake.

Required coverage:

- Table-driven receipt headers for approve/reject/feedback on Telegram, ACE, CLI,
  mobile, and auto. Assert complete first lines with exactly one attribution, time,
  Tale/Epic wording, and truthful summaries.
- Actual callbacks for Reject, approve-only, and commit-only on a plan with decisions.
  Assert the submitted selected options, declared decision fields only, displayed
  revision, durable normalized response, and retained recovery context. Existing
  approve+commit and feedback tests remain green.
- Deferred stale error, a schema/error message, and missing response with a missing proc
  past the grace interval. Poll completion, inspect the original card's Refresh/control
  token and kept draft/action, then tap Refresh to demonstrate usable recovery. Repeat
  polls to prove no duplicate report; inject edit/send failures to verify durable
  retries.
- Native Telegram, ACE, and CLI settlement, external reject, and feedback reply with
  source_message_id null. Exercise completion and post-poll cleanup together: the
  original card is edited once, the pending action is removed, draft/progress clears
  only after settlement, and the completion summary is sent once. Repeated polls and a
  receiver restart must preserve these facts.
- A real host launch-failure outcome arriving after initial acceptance causes one
  follow-up card edit with immutable values. Assert no coder-start claim for
  commit-only, reject/feedback, generic errors, or a running proc. Successful outcome
  ends observation without another identical card edit.
- Exact keyboard row labels and callback tokens for default, changed, choice
  sub-keyboard, memory, and epic cards; primary `style: success` when supported and
  compatible fallback otherwise; callback payloads at most 64 bytes.
- Step-by-step budget fixtures distinguish all three degradations; only choice lines are
  quoted, asks and memory lines are outside, all defaults/quotes/new chips survive, and
  MarkdownV2 escapes are complete. Check a formatted card as well as the sheet helper.
- Accepted-plan PDF through the frozen/stamped facade: non-default accepted values,
  correct defaults, labelled callout continuations, source preservation, and
  temporary-file cleanup. Test the new memory chip using a temporary memory fixture,
  never a canonical project memory edit.
- EPIC_DECISIONS_PLAN asserts an Epic primary action and callback, its decision content,
  and epic-specific verdict rather than a header substring. Correct the vector helper
  test's docstring so it states the coverage it actually supplies.

Update the two obsolete `cli_answer._reject_detached_tty_options` comments in
`gate_callbacks.py` and `tests/test_custom_gates.py` to
`cli_answer_submit.reject_detached_tty_options`.

## Verification and completion

Run focused decision, gate-flow, custom-gate, and settlement tests using the linked
checkout's configured environment, then **`sase tool run check`** there. Never use
check-full. If checks need a long-running handoff, use `/sase_monitor` and wait for the
handoff command itself to exit; a yielded session is still running. Report concrete
passing test results in the phase note.

If primary sase files unexpectedly need changes, read `lint_and_test.md` and
`symvision.md` through `sase memory read` before finishing and run that repo's check
too. Avoid creating new unused-public seams. Before closing, from the primary workspace
run `sase bead epic-symbols sase-1hi.10.7.5` and inspect any linked Justfile entry for
this phase. Resolve leftovers or re-key them to an appropriate still-open parent/later
phase. Record the empty result or re-key.

For an out-of-scope failure, establish that it reproduces identically on a clean base
before calling it pre-existing. Append
`sase bead note sase-1hi.10.7.5 'PROPOSED FOLLOW-UP: <summary — reproducer and evidence>'`
and cite any existing task. Never create beads. A proven clean-base failure does not
keep this phase open.

Finish with
`sase bead close sase-1hi.10.7.5 --note "<fixed items, named test coverage, check outcome, and epic-symbol audit>"`.
Close only this phase; never any ancestor. Use `/sase_final` for the root final
declaration so the host commits the linked repo changes. Do not manually commit, branch,
or open a PR. The final answer reports the phase outcome and verification, including
material limitations.
