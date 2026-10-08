---
tier: tale
title: Telegram Plan Decisions review and receipts
goal: "Telegram reviewers can see and change every Plan Decision, approve the displayed
  revision, and receive durable answered receipts with the same values and summary as
  the other SASE review surfaces.

  "
size: medium
proposed_by: bbugyi200.apollo.sase-1hi.7
bead: sase-1hi.7
create_time: 2026-10-08 01:55:06
status: wip
---

- **PARENT:**
  [202610/plan_decisions.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions.md)
- **BEAD:**
  [sase-1hi.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/sase-1hi.7.md)

# Plan: Telegram Plan Decisions review and receipts

## Scope and accepted design

Implement the complete `telegram` phase of `plan:202610/plan_decisions.md`, assigned to
`sase-1hi.7`. The epic's Sections 1.3, 1.6, 3, and 6.7 are authoritative. The accepted
research artifacts are
`research:202610/plan_frontmatter_decisions/plan_frontmatter_decisions.md` and
`research:202610/plan_decisions_cross_surface_ux/plan_decisions_cross_surface_ux.md`;
the later report replaces the earlier Telegram step-wizard proposal with controls on the
review card.

This is a medium tale because the frozen definitions, value resolver, Decision Sheet,
summary, revision check, and supervised gate execution already exist. One coder can
implement the connected state transitions in a single transport and verify them
together. No reviewer choices remain to make in this implementation plan: the parent
epic already settled the product behavior.

Work primarily in the linked `sase-telegram` repository. Open it with
`sase repo open sase-telegram -r "Implement assigned Telegram Plan Decisions phase"` and
read its `AGENTS.md`. Use repo-relative paths below; never depend on the planner's
checkout location. Read source requirements with `/sase_memory_read`, and read the
parent design and research through `sase artifact read`.

Any necessary primary SASE changes are limited to thin public document adapters and
shared PDF presentation. Decision semantics remain in the existing Rust core. Consume
decision behavior through `sase.sdd.plan_decisions`; do not import Rust bindings or
duplicate resolution, quote matching, callout grammar, or summary generation in the
plugin. No new Rust binding is anticipated. If one proves necessary, open `sase-core`
through `/sase_repo`, honor its instructions and the backend boundary, and advance
`sase-core-revision.txt` past the binding commit.

## Existing seams and implementation constraints

- `gate_flow.py` verifies the gate envelope and atomically stores
  `telegram_gate_progress.json`, but currently retains neither decisions nor
  `review_revision` in `GateView`/`GateProgress`.
- `formatting.py` has a bounded Properties/body preview and generic query-driven
  keyboards. It currently dumps every frontmatter property. Extend its entry points
  while putting substantial decision-specific rendering in focused modules.
- `gate_inputs.pending_fields()` reads declared inputs; Plan Decisions use raw
  `input_schema` properties, so they must stay out of the generic input step flow.
- `inbound.resolve_gate_response()` submits the shared `gate answer --no-detach`
  supervised process. Its operation payload currently lacks `review_revision`. Preserve
  that path; the shared executor already rejects `stale_review` when the revision
  reaches it.
- `gate_response._execute_gate_callback_response()` currently dismisses the keyboard and
  clears progress immediately after submitting a process. Submission can precede a stale
  rejection; retain enough durable review context to restore the refresh action when the
  process rejects the displayed revision.
- `gate_completions.py` delivers generic replies, while `keyboard_cleanup.py` and the
  inbound pre/post-poll cleanup handle external answers and durable retries. Unify
  decision-plan receipt handling across those paths.
- `outbound.get_unsent_notifications()` currently excludes all silent rows. The existing
  auto receipt has `action=None`, `silent=True`, `muted=False`, the tag
  `plan_decisions_receipt`, and a dedup key per request. Admit that specific silent
  informational receipt and send it quietly.
- `pdf_convert.md_to_pdf()` now delegates to `sase.attachments.markdown_pdf`. Preserve
  the renderer's engine fallback, cleanup, filenames, and resource paths.
- `agent_launch._launch_agents_with_notifications()` calls
  `launch_agents_from_cwd(prompt)` without the supported `origin` argument.
- Inbound modules follow the existing layered import structure. Tests patch them through
  `tests/inbound_namespace.py`; register new handler modules there when needed instead
  of restoring a monolithic handler.

## 1. Shared API access and persisted review state

Add a small plugin adapter that feature-detects the public Decision Sheet and summary
API in `sase.sdd.plan_decisions`. Honor `is_enabled()` where present; support its later
removal by the policy phase. An installed SASE without that API, and plans without
decisions, retain the current review behavior. An available API failing to build a sheet
for a decision-bearing gate must show unavailable controls and keep approval disabled
rather than silently omitting decisions.

Extend verified `GateView` with the envelope's frozen `payload.decisions` and
`review_revision`. Build sheets with `sheet_binding(definitions, values, revision)` and
summaries with `summary_binding(sheet, verdict, form)`. Start with the core's effective
defaults, including clamped memory defaults. Keep author order and use the sheet's
`value`, `default`, `changed`, and memory metadata directly.

Extend `GateProgress` with the displayed revision, draft decision values, open choice
index, and any submission/receipt state required below. Persist those in the existing
atomic progress write; legacy JSON without these fields loads safely. Validate saved
values against the frozen definitions using the shared API. Malformed state must not
select an undeclared choice or grant memory consent. Keep a stale saved revision until
the reviewer explicitly refreshes it. Opening a choice or replaying a value-set callback
must survive a receiver restart.

Store Telegram drafts only in Telegram progress. They never update a gate's definitions,
accepted answers, or ACE's draft state.

## 2. Static question sheet and bounded message rendering

For decision-capable plan reviews, remove `decisions`, `decided_by`, and `decided_via`
from the Properties card. Insert the static Decisions question sheet after the
header/notes and before Properties. Include decision/memory counts, numbered IDs and
full asks, choices, the `★` default, optional `why`, and memory
selectors/types/provenance/quote. Use `yes`/`no`, `◉`/`○`, `★`, `●`, and `🧠` as
specified by the epic. Memory provenance reads `you asked`, `not asked`,
`⚠ quote not found · off`, or `approved in epic`; core notes mention
`loaded every turn`.

Reserve message space for this sheet before budgeting Properties and the body. Around
1,800 characters, degrade the sheet in the prescribed order: drop labels of non-default
choices, then remaining choice labels, then use an expandable blockquote for choice
lines. Every ask and its complete default line survive; never slice through a decision.
Keep the complete escaped message within Telegram's 4,096-character limit, including
MarkdownV2 markup. Reduce optional notes/properties/body first if escaping or long
memory quotes exhaust space. Preserve full scope information in the attachment. Cover
the maximum five decisions and escaping-heavy content in budget tests.

During edits, only change the inline keyboard. Explicit stale refresh is the exception
that re-renders the review for a newer revision; settlement replaces the question sheet
with its receipt once.

## 3. Live keyboard and replay-safe callbacks

Add decision rows above the existing AND option controls. Pair toggle rows where labels
fit; choice rows show `◉ id: value`, `●` for changes, and `▾`. A choice opens an
in-place radio sub-keyboard with all keys, selection marks, `★`, changed marks, and
`↩ Back`. Choosing a value returns to the main card without submitting.

For decision plans, render the primary button as `✅ Tale · defaults` or
`✅ Tale · N changes`, with the core short summary's memory indicator when appropriate;
use the corresponding Epic label for epic gates. Preserve sealed option/group labels and
query semantics. Keep approve-only, commit-only, and combined tale subsets valid.
Request `style: success` only when supported by the installed Telegram client library,
using a compatibility-safe construction. Show `↺ Reset` after a change; it restores the
core effective defaults.

Encode short, server-resolved index/value tokens with the displayed revision, such as
`d2=0r4`, `d0=k1r4`, `d0>r4`, `d<r4`, and `dzr4`. Include revision binding on
decision-plan verdict, feedback, and AND-control callbacks too. Validate the complete
encoded callback against the 64-byte limit. Values are explicit sets; replaying the same
token sets the same value instead of flipping it. Prefer explicit sets for decision-plan
AND controls as well, while preserving generic gate compatibility.

Opening a choice answers with its ask. Setting it answers `grouping → mode`; memory
toggles answer, for example, `🧠 tui_note yes — authorizes editing tui.md`. Editing,
back, and reset never submit a process, accept a gate decision, or launch an agent.
Validate indexes and keys against verified server data, rejecting malformed/forged
callbacks safely.

## 4. Submission, stale review, and feedback

Merge the current decision vector as `decision_<id>` into each selected plan option's
inputs, including feedback's provisional inputs. Keep declared-input collection for
other fields and merge decisions after that collection too. Give `approve` and `commit`
the same decision vector, even when only one is selected. Send `review_revision` as
operation metadata, not an undeclared option input. Add a compatible optional field to
`ResponseAction`, preserve it through awaiting-feedback state, and pass it through the
supervised process payload. Retain `source="telegram"` and host-owned caller
classification.

Compare each callback's revision with both persisted displayed state and the current
verified envelope. Any mismatch answers exactly
`This plan changed since this card was shown.` and presents `↻ Refresh review`. It
performs no submission. Explicit refresh reloads current prose and the sheet, retains
still-valid draft values, persists the newly displayed revision, and returns to the main
keyboard; another tap is required to approve.

The same refresh path must handle `stale_review` reported after the supervised process
starts. Persist the source message/chat, bundle, revision, and draft before submitting,
and keep that context until authoritative acceptance or rejection. Never turn
`process submitted` into an approval claim. Reuse the completion queue to surface the
rejection and restore refresh controls.

Feedback asks for a reply to the review message and carries the current vector as
provisional values. When multiple prompts await replies, unreplied text prompts the user
to reply to the appropriate message and returns without launching an agent. Preserve the
existing single-waiter plain-text convenience, reply-ID isolation, explicit slash
commands, and stale-flow cleanup. A feedback reply still submits its original displayed
revision and can become stale.

## 5. Settled review cards and completion replies

Use one decision-plan receipt path for Telegram completion and external TUI/CLI/mobile
settlement. Persist a deterministic receipt edit before removing the local pending
action or progress. Keep retry context after transient API failures and receiver
restarts. Coordinate the existing markup cleanup so it cannot discard the card before
the answered edit is queued.

Render accepted values from authoritative normalized inputs/results or the stamped plan,
using the shared sheet API. Use `effective_response_input` when reading per-option
response inputs. Do not reconstruct accepted values from a Telegram draft or a hash-only
acceptance receipt. Fast acceptance may happen before `response.json` or the stamped
plan is available: disable the controls and retain a receipt job until the accepted
vector is readable.

The receipt contains the verdict, decider/surface/time, numbered accepted values,
`● (★ default)` for changes, memory selectors, and the full core summary. Remove the
keyboard in the same `edit_message_text` request. Repeat poll ticks and completion
retries produce one logical answered-card edit; freeze its text and timestamp, treat
Telegram's already-identical-message response as success, and retry only unfinished
work.

Keep acceptance and implementation status distinct. A completion reply includes the full
summary and says a coder launched only when the shared outcome proves it. Inspect
post-response failures as well as responses, because a response can exist before a
successor-launch failure. An accepted plan whose coder could not start reads
`Approved with these choices · coder could not start · retry`, retaining its immutable
accepted values. Rejection, feedback, cancellation, and generic gates retain their
appropriate existing behavior.

## 6. Quiet auto receipts, PDFs, and launch provenance

Allow silent informational rows tagged `plan_decisions_receipt` through the outbound
filter, preserving muted/read filtering and normal suppression of other silent rows. Set
`disable_notification=True` for that receipt. Add an optional compatible argument to
`telegram_client.send_message` and pass it through every chunk, retry, and Markdown
parse fallback. Preserve per-request deduplication and advance the outbound cursor only
after successful delivery.

Make plan PDFs show a Decisions table before Properties/body and turn decision callouts
into labelled blockquotes with every branch retained. The current plugin delegates
conversion to the shared SASE renderer, so extend that renderer's presentation
preprocessing instead of duplicating Pandoc commands. Obtain definitions/sheets and
core-validated callout spans through public thin helpers in `sase.sdd.plan_decisions` if
the existing API needs a document-loading facade. Use frozen review data for a pending
gate PDF and stamped answers for accepted documents. Strip the decision bookkeeping
fields from the Properties table. Preserve original source bytes, relative-resource
resolution, engine fallbacks, and temporary-file cleanup. Old installed SASE retains
today's PDF behavior.

Pass `origin="typed"` to the canonical launch pipeline for prompts newly authored by
Telegram users, including supported text/media entry paths. Audit retry and generated
successor paths so agent-authored prompts do not become human quote evidence. Do not
grant memory consent by relabelling a generated prompt.

## 7. Verification and documentation

Add focused tests using real shared decision fixtures and verified gate envelopes where
possible, while mocking Telegram API calls and actual process launches. Cover both tale
and epic gates, with choices, toggles, requested/unrequested memory decisions, and the
maximum-size sheet.

- Formatting order, hidden Properties bookkeeping, glyphs, choices/defaults/why,
  provenance, MarkdownV2 escaping, and staged budget reduction.
- Main/sub-keyboards, reset, single/combined tale option subsets, conditional success
  styling, and all encoded tokens within 64 bytes.
- Repeated set callbacks, invalid tokens, zero side effects from edits, persisted
  drafts/open choices, legacy/corrupt progress, and restart recovery.
- Identical decision inputs on approve/commit, revision metadata reaching the supervised
  request, immediate and asynchronous stale rejection, explicit refresh, and an edit
  occurring after callback validation but before execution.
- Provisional feedback values/revision, replying to one of multiple prompts, unreplied
  text not launching, single-waiter compatibility, and slash commands.
- Telegram and external settlement, pending execution, authoritative answers,
  accepted-but-launch-failed status, repeated polls, API retry, and restart.
- Auto receipt filter exception, quiet delivery across split/fallback sends, normal
  silent/muted/read filtering, and cursor/dedup behavior.
- PDF preprocessing/table/callouts, pending versus accepted documents, resource paths
  and cleanup; launch `origin="typed"` and generated-prompt provenance.
- Missing decision API, disabled transitional flag, and decision-free/generic gate
  compatibility. Update existing formatting/keyboard pins in
  `tests/test_custom_gates.py` and the related integration fixtures.

Update `docs/outbound.md` and `docs/inbound.md` with the static sheet, live answer
controls, approve-as-displayed behavior, stale refresh, receipt states, quiet auto
receipts, and feedback reply rules. Correct the stale outbound list of plan buttons. No
manual live Telegram message or agent launch is required for tests.

Read `lint_and_test.md` before completing any primary SASE edits, and read
`symvision.md` before adjusting epic-symbol entries. Format the changes, run
`sase tool run check` inside the linked Telegram checkout, and also run it in the
primary checkout if shared PDF/adapters were changed. Run each check once after the
final relevant edits; repair introduced failures and rerun. Use `/sase_monitor` for a
command that needs a handoff, preserving the original bead and completion obligations in
its follow-up. Do not run `just check-full`.

If a failed check reproduces identically on an unchanged clean base, record its
commands/evidence and any existing tracking bead in a `PROPOSED FOLLOW-UP:` note on
`sase-1hi.7`; it does not keep this phase open. Record all discovered out-of-scope work
through `sase bead note sase-1hi.7 'PROPOSED FOLLOW-UP: ...'`, never by creating new
beads.

## Completion

Before closing, run `sase bead epic-symbols sase-1hi.7` again. The planner observed no
entries; still resolve any that appear during implementation, or re-key their Justfile
entries to a still-open parent/later phase when a real later consumer exists. Verify any
resulting primary checkout change.

After the phase's entire scope is implemented and checked, close only `sase-1hi.7` with
`sase bead close sase-1hi.7 --note "<implemented behavior and verification evidence>"`.
Do not hand-set status or close `sase-1hi`, any ancestor, or another phase. Report the
check results and limitations, then submit the root SASE final declaration for every
changed repository so host-owned finalizers commit the work.
