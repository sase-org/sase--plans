---
tier: tale
title: Repair Telegram Plan Decisions submission, refresh, and settlement
goal:
  Telegram submits exactly the displayed selected options, recovers stale reviews, and
  renders truthful complete decision receipts and PDFs.
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.10.6
bead: sase-1hi.10.6
create_time: 2026-10-08 10:17:08
status: wip
---

- **PARENT:**
  [202610/plan_decisions_landing_repairs.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_repairs.md)
- **BEAD:**
  [sase-1hi.10.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.10.6.md)

# Repair Telegram Plan Decisions for sase-1hi.10.6

## Scope and ownership

Implement the assigned phase **sase-1hi.10.6**, already in progress. This is a single
implementation tale because the defects have known causes in one plugin and share the
same existing gate data. Keep the assignment and status managed by SASE.

Read these audited sources before implementation:

- `sase bead read sase-1hi.10.6 -r "Need the phase scope and accepted epic decisions"`.
- `sase artifact read plan:202610/plan_decisions_landing_repairs.md "Need Telegram acceptance criteria"`,
  especially Sections 0, 6, and 7.
- `sase artifact read plan:202610/plan_decisions.md "Need authoritative Telegram and reliability contracts"`,
  especially Sections 1.3, 1.6, 3, 4, and 6.7.
- `sase memory read lint_and_test.md symvision.md -r "Need verification and epic-symbol rules"`.

Use `/sase_repo` to run
`sase repo open sase-telegram -r "Implement assigned Telegram Plan Decisions repairs"`
from the host workspace, then read that checkout's `AGENTS.md`. All implementation,
tests, and documentation belong in the printed linked checkout. File paths below are
relative to its root unless explicitly labelled **sase**. Reopen repositories in a
successor; never reuse an absolute workspace path from a prior turn.

Honor inherited DECISIONS without re-asking or redeclaring them. This repair does not
require memory changes. The Rust-owned sheet, summary, validation, and resolver remain
the source of domain behavior; the plugin adapts and renders their records. The
completed dependency sase-1hi.10.2 already exposes `load_stamped_decisions`,
`effective_response_input`, `sheet_binding`, and `summary_binding` through
`sase.sdd.plan_decisions`. Use that surface exclusively for Plan Decisions. Generic gate
imports retain their existing purpose.

Close only sase-1hi.10.6 after implementation and verification. Do not create beads or
close sase-1hi.10, sase-1hi, or any ancestor plan. Record discoveries as
`sase bead note sase-1hi.10.6 'PROPOSED FOLLOW-UP: <summary — evidence>'`. The
ancestor's end-to-end smoke, usability report, and skill deployment belong to its land
agent.

## Findings confirmed before proposal

Both source checkouts were clean during planning. The linked checkout has no
`.venv/bin/python`, and the host checkout's virtualenv lacks `telegram`; no plugin test
results were claimed during planning. Prepare the coordinated environment before
establishing the baseline.

- `inbound_handlers/gate_callbacks.py::_start_or_submit_gate_selection` and
  `inbound_handlers/gate_input_steps.py::_advance_gate_input` merge every decision into
  all options and add unselected approve/commit entries. The executor refuses those
  entries, and reject has no decision properties.
- `decision_callbacks.py::apply_decision_token` validates the old displayed revision
  before handling refresh. Its caller only edits reply markup, so even a successful
  refresh leaves old prose visible.
- `inbound_handlers/gate_response.py` drops the pending action as soon as a proc is
  submitted, making the later restored Refresh button unusable.
- `decision_receipt.py` imports internal SASE modules, defaults to Telegram/Tale,
  displays Python booleans and the literal `(★ default)`, and requests the summary with
  the wrong verdict. Settlement assumes every branch has accepted values.
- `inbound_handlers/gate_completions.py` waits indefinitely for a missing response,
  considers any non-success proc a launch failure, and returns early on a null source
  message id instead of trying the active review id.
- `decision_sheet.py` and `formatting.py` slice escaped strings. The supposed final
  expandable blockquote is a single `>` prefix, and long sheets lose whole asks.
- `decision_keyboard.py::_button` supports success styling but has no caller. AND
  callbacks still toggle server state. PDF decisions come from a separate YAML
  interpretation and only the first callout line receives a label.

## Implementation

### 1. Submit only the selected schemas

Share a small plugin helper between the direct callback and completed input-flow paths.
Start with the inputs of the actual selected option ids. For each selected option,
inspect its raw `input_schema.properties` and merge only the corresponding `decision_*`
fields it declares. Never manufacture approve/commit entries. Preserve ordinary declared
inputs and send the displayed `review_revision` independently. When approve and commit
are both selected, send identical decision values to both. Reject receives an empty
input object; feedback receives its own provisional vector.

Keep decision fields out of generic input collection. Plan-review feedback asks for a
reply to the review card; generic required and optional feedback retain their existing
toast wording and flow. Repair lightweight generic view fixtures to carry the real empty
decision fields, preferably by using `GateView`, rather than weakening the production
gate contract to accommodate incomplete test doubles. Preserve the early callback
acknowledgement before durable submission.

### 2. Make refresh recover the whole review

Handle the explicit refresh token before comparing it with the saved displayed revision.
Every other edit and verdict callback remains revision checked. Reload the verified
current bundle, retain only still-valid Telegram draft values, reset the open choice,
and bind the refreshed controls to the current revision. Refresh never submits an
answer.

Reuse the normal plan-review formatter to edit both message text and keyboard in one
call using the current `plan.md`, frozen definitions, and notification identity.
Preserve/retrieve enough presentation context from the pending action and bundle to
avoid losing the header and notes. Commit the displayed revision after the card edit
succeeds; an API failure must remain retryable without claiming new prose was shown.
Ordinary value edits continue to edit only the keyboard.

Persist the original review message and chat for direct submission, input-step
completion, and feedback replies. Disable controls while a submitted proc runs, but
retain its pending action and progress until its outcome is known. Handle both
synchronous `stale_review` errors and asynchronous recorded errors by restoring a
working Refresh button and retaining draft values. A later refresh must work after a
receiver restart, and a second tap is required to approve the refreshed review.

### 3. Use one truthful receipt path

Have `decision_receipt.py` load terminal response/outcome data and render it for both
`inbound_handlers/gate_completions.py` and `inbound_handlers/keyboard_cleanup.py`. Use
authoritative response inputs from the facade's `effective_response_input`, examining
selected decision-bearing options rather than just the first selected option. Use frozen
gate definitions for the sheet, with the facade's accepted-plan loader as recovery where
necessary. Never treat a Telegram draft as an accepted vector or reverify accepted
memory provenance in the receiver's environment.

Derive these presentation fields from durable facts:

- Tale versus Epic from the gate kind.
- Approval summary verdict from selected ids: `coder + commit`, `coder`, `commit`, or
  `epic launch`. Pass that actual verdict to `summary_binding`.
- Reject and feedback are distinct outcomes with their own headers, including when they
  have no accepted decision values. Feedback values stay labelled provisional.
- Decider and surface from response `caller` and `source`, with stamped
  `decided_by`/`decided_via` for recovery. Map reviewer/Telegram to `you via Telegram`,
  `tui` to `via ACE`, CLI to `via CLI`, and `auto_resolution` to `auto`; retain agent
  attribution when recorded. Do not guess Telegram for unknown provenance.
- Time from `responded_at_unix`, consistently formatted for the receipt. Do not use a
  fresh polling timestamp.
- Toggle values as yes/no, authored defaults as `★`, and changes as `● (★ pane)` or the
  actual yes/no default, in author order. The completion includes the core full summary
  sentence.

Separate acceptance from launch completion. A running or not-yet-visible proc keeps its
completion job pending and never says the coder failed. Only recorded launch failure
evidence can add `Approved with these choices · coder could not start · retry`; reject,
feedback, commit-only, and generic execution failures must not acquire that claim. Once
the proc is terminal, a missing response must produce an actionable failure report
rather than spin forever; while it is running, wait for durable response/error
publication.

Try non-null `source_message_id`, then `active_message_id`, then saved review/action
message ids. Never edit the feedback reply as though it were the review card. Edit the
existing review into its receipt and remove its keyboard in the same request. Clear
pending action, awaiting feedback, progress, and completion records only after the
appropriate terminal delivery succeeds, including an external reject. Treat Telegram's
already-identical error as success. Preserve retry state on API failure; cleanup retries
must retry the receipt edit, not merely erase controls. Persist receipt delivery
progress so polling/restarts do not duplicate the edit or the completion message.

### 4. Preserve complete questions within the message budget

Build structured decision blocks before escaping, so budgets are enforced by rendering
whole fields/lines rather than slicing MarkdownV2 strings. Honor the approximately
1,800-character sheet allocation in this order: full labels, then remove non-default
labels, then remaining labels, then real expandable choice-line blockquotes using the
existing `**>`/line prefixes/closing `||` convention. Reserve each complete ask and its
complete starred default line before optional detail. For extreme valid inputs, compact
optional why/provenance/choice detail by whole lines; retain all asks and default
values, with full choice keys available in the radio keyboard and the attached plan.
Avoid duplicated provenance wording.

Remove the independent sheet slice and decision-message final slice in `formatting.py`.
Budget header/notes/Properties/body around the protected question blocks, shortening
optional sections first and re-escaping source prefixes when needed. Never split an
escape or leave malformed formatting. Keep reviews within Telegram's 4,096-character
limit. Render `exists: false` records with the `new` memory chip while retaining the
other type/provenance chips.

### 5. Finish keyboard, PDF, and compatibility details

Wire the style-detecting button factory to the primary Tale/Epic action only; use
`style: success` when supported and the ordinary button otherwise. Pin the layout:
decision rows first, paired short toggles, AND members, primary submission, then Reset
when changed and Reject/Feedback. A choice sub-keyboard shows radio values and Back, and
selecting a value returns to the main review without submission.

Emit explicit set-state tokens for AND members, e.g. `x0=0r4` for a decision plan, and
update token parsing/server application together. Replaying the same token must leave
the same selection. Keep every callback within 64 UTF-8 bytes and bind new decision-plan
controls to the revision actually displayed. Maintain legacy generic gate callbacks
where compatibility requires them, while new keyboards emit set-state tokens. Decision
value callbacks already set values; preserve that.

For PDFs, use the frozen bundle's `payload.decisions` when the source is a pending
review (pass/derive its gate context), and the facade's `load_stamped_decisions` sheet
when accepted. Do not independently interpret the YAML decision schema. Frontmatter
parsing may still serve non-decision Properties/bookkeeping removal. Render an ordered
Decisions table and label every line in each validated decision callout block, including
continuations and yes/no/choice branches. Preserve source bytes, sibling temporary
resources, renderer fallbacks, and cleanup on all exits.

Keep capability detection for older installed SASE versions and graceful behavior when
the facade is absent. Remove the obsolete Plan Decisions flag fixture and
`override_flags(plan_decisions=...)` test blocks; the flag is removed. Repair the
split-send test to assert absence-or-None markup on earlier chunks and markup only on
the last. Verify `disable_notification` on every chunk and parse-mode fallback. Remove
the stale Approve/run-payload paragraph in `docs/outbound.md`, and update
`docs/inbound.md` for refresh, provisional feedback, and receipts where needed.

## Tests and verification

Use real `build_plan_approval_gate_spec`/`create_gate` fixtures and execute selections
through the host executor, with all Telegram API calls, plan archival, and agent launch
side effects mocked. Reuse `tests/inbound_namespace.py` for handler patches. Patch
`sase.plan_approval_actions._archive_plan_for_approval` as existing custom-gate tests
do. Memory fixtures create a temporary project with a `type: reference` `tui.md` and
chdir there; these are test data, not canonical memory edits. Build a real typed-prompt
source for tests that need verified provenance.

Repair all nine audited test failures: the five named custom-gate tests, the
long-message client test, and the memory, merged-vector, and removed-flag Plan Decisions
tests. Change feedback's assertion to its selected `feedback` input, not an unselected
`approve` entry. Extend focused files or split new cohesive test modules as needed
instead of growing handler files with duplicate helpers.

Add behavioral coverage for:

1. Approve+commit, approve-only, commit-only, reject, feedback-only, and epic
   submission; no unselected inputs; decision fields excluded from input prompts; both
   direct and collected-input paths; identical normalized vectors/defaults.
2. A real prose edit advancing the bundle revision while saved progress retains the old
   revision; stale edits/submits; successful Refresh editing current prose and controls;
   stale-after-submit recovery; restart and edit-failure retry behavior.
3. Exact receipt text and cleanup for Telegram, ACE, CLI, auto, Tale, Epic, external
   reject, and feedback; changed/default yes/no values; response publication gaps;
   running proc versus a genuine launch failure; null source-id fallback; failed receipt
   edit and already-identical retry without repeated successful delivery.
4. Keyboard layout and supported/unsupported style constructors, choice open/set/
   back/reset, AND replay idempotency, actual displayed revisions, and token length.
5. The 1,800-character degradation order, real expandable syntax, five maximum-size
   decisions with escaping/long quotes, every complete ask/default surviving, and a long
   header/Properties/body still fitting 4,096 without broken escapes.
6. Accepted PDFs in a different cwd without quote sources, pending PDFs using frozen
   gate facts, multiline callout labelling, all branches retained, unmodified source
   files, and temporary cleanup on renderer failures.
7. Missing facade and no-decision generic gates, plus quiet receipt delivery across
   split sends and plain-text fallback.

Before setup that consumes sase-core files, open it through `/sase_repo` and read its
`AGENTS.md`; this is dependency build preparation, not authorization to change core. Run
`just install` in the linked plugin if required, using `/sase_monitor` for a long setup
and a continuation that resumes this exact phase. Establish the focused baseline, then
run focused changed-area tests as implementation progresses. Run formatting and
`sase tool run check` in every changed repo for final verification; never run
`just check-full`. Route long commands through `/sase_monitor` with the required
continuation instead of ending the turn while a command still runs.

Failures that reproduce identically on the clean base do not block completion. Record
`PROPOSED FOLLOW-UP:` evidence with the exact node, clean-base reproduction, and an
existing tracking bead where applicable. The epic lists known sase failures on sase-1hr,
sase-1hy, sase-1g3, and sase-1hp; do not repair those here. New or changed failures
remain this phase's responsibility.

Immediately before closing, run `sase bead epic-symbols sase-1hi.10.6` from the host
workspace. It was empty during planning; verify again. Resolve each remaining row or
re-key its **sase** Justfile entry to a still-open parent/later phase, reading the
symbol rules and running that repo's required check if the Justfile changes. Then close
only
`sase bead close sase-1hi.10.6 --note "<fixed items and test mapping; check results with KNOWN failures; epic-symbol cleanup>"`.
Finish through `/sase_final` as the root implementation agent; host finalizers own
commits and publication.
