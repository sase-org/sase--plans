---
tier: epic
title:
  "Plan Decisions finish gaps: visible chosen-branch tint, a stale reopen that can
  submit, generic rails back to 42, the owed gate tests, and a real Telegram
  launch-failure signal"
goal:
  Close the gaps the sase-1hi.10.7 land audit found in its own phases. ACE shows the
  chosen branch green and bold on screen. A stale review reopened after the modal closed
  can still submit. Generic gates keep their old rail width. The gate route tests the
  repair plans required exist and assert behaviour. Telegram says "coder could not
  start" only when sase recorded a real coder launch failure, and keeps each decision's
  choices next to its question.
phases:
  - id: ace
    title:
      Rendered chosen-branch tint, stale reopen through the real open path, and generic
      rails back to 42
    depends_on: []
    size: medium
    description:
      "ace: make the chosen-branch green/bold tint survive Textual rendering and test it
      on rendered output, reopen a closed stale review through handle_plan_approval so
      its submit is handled, load the stale bundle off the UI thread, and scope the rail
      width change so generic gates and decision-free plans return to 42 cells."
  - id: goldens
    title: Refresh and inspect the plan and generic gate goldens after the ace fixes
    depends_on:
      - ace
    size: small
    description:
      "goldens: run targeted just fix-tui-screenshots for the plan gate, custom gate,
      sudo request, and plan decisions inbox/toast goldens under /sase_monitor, inspect
      every update, and confirm the green chosen branch and the 42-cell generic rails."
  - id: gate_tests
    title: The gate route tests the first two passes skipped, plus one restamp record
    depends_on: []
    size: medium
    description:
      "gate_tests: add the missing stale_review, authored-order stamping, agent memory
      refusal, new-strand grant, decision-host-check-failed, caller classification,
      handoff writer, restamp, memory guard, and receipt inbox tests, and record a
      restamp failure inside side effects exactly once."
  - id: telegram
    title:
      Real coder launch-failure signal, per-decision blockquotes, and the missing
      keyboard, settle, PDF, and retry tests
    depends_on: []
    size: medium
    description:
      "telegram: read sase's gate-turn followup_error as the only coder launch-failure
      signal, stop treating any side_effects error as a failed launch, keep each
      decision's choice lines next to its ask in the expandable stage, and add the
      keyboard, external settle, PDF, stale refresh, and keyboard-removal retry tests."
parent_bead: sase-1hi.10.7
proposed_by: bbugyi200.apollo.sase-1hi.10.7.land
create_time: 2026-10-08 19:15:57
status: wip
---

- **PROMPT:**
  [prompts/202610/plan_decisions_finish_gaps.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/plan_decisions_finish_gaps.md)
- **PARENT:**
  [202610/plan_decisions_landing_finish.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions_landing_finish.md)

# Plan Decisions finish gaps

## 0. Scope, ownership, and ground rules

This child epic finishes **sase-1hi.10.7** ("Plan Decisions landing finish"). That epic
finishes sase-1hi.10, which finishes sase-1hi ("Plan Decisions").

- The authoritative design is **plan:202610/plan_decisions.md** (the parent plan). Read
  Sections 1.2, 1.3, 6.6, and 6.7 with `sase artifact read`.
- The plan this epic completes is **plan:202610/plan_decisions_landing_finish.md**. Its
  phases landed in commits `eb646c3d71` (gate), `3412a9f1bd` (cli), `b49f9bcb28` (tui),
  `4bc1db294c` (goldens), and sase-telegram `db624d3` (telegram).

The sase-1hi.10.7 land agent audited those phases on master `67df4cfba5`. Every item
below is a confirmed gap from that audit. Line numbers are from `67df4cfba5` and may
drift. Confirm each gap before you change anything.

**Already done. Do not redo it:** bead-work answer reuse, durable pre-acceptance error
records, one direct resolver, stripping private gate keys, structured missing-note
grants, strand coverage in the memory guard, the quiet receipt on the ACE direct page,
`_write_acceptance_meta`, the completion snapshot and `-S` helpers, the provenance
chips, the plan-show and `-D` handler tests, the settled-polling probe, the open-modal
stale rebuild, the short Verdict labels, the signature-keyed plan sheet cache, and the
Telegram receipt headers and stale/error recovery.

**Module moves since the audited plan.** `b770238c09` split
`src/sase/sdd/plan_decisions.py` into a facade plus `plan_decisions_host.py`,
`_plan_decisions_shared.py`, and two more modules. `67df4cfba5` split
`tests/ace/tui/test_plan_decision_ace.py` into themed `test_plan_decision_ace_*.py`
modules. Use the new locations.

**Not in this epic.**

- Closing sase-1hi.10.7, sase-1hi.10, and sase-1hi, and marking their plans done. The
  land agents resumed through `parent_bead` do that.
- The parent plan's Section 7 end-to-end smoke and usability report.
- Deploying changed skills.
- The `_gate_source`/`_gate_caller` pops in `plan_approval_actions.py` (~327-334) and
  the fallback reads in `plan_gate_decisions.py` (~313-316). They stay, to defend
  against `response.json` files written before `1914591ab4`.

**Repositories.**

- `ace`, `goldens`, and `gate_tests` work in the sase repo.
- `telegram` works in the linked sase-telegram checkout. Open it with
  `sase repo open sase-telegram` and read its `AGENTS.md`.
- No phase is expected to change sase-core.

**Required reading.** Use `sase memory read`. This epic edits no memory notes.

- Every sase phase: `lint_and_test` and `symvision`.
- `ace` and `goldens`: also `tui`.

**Verification.** Run `sase tool run check` in every repo you change. Never run
`just check-full`.

These failures are on master and are not caused by this epic. Record them as KNOWN and
do not fix them here:

- `tests/test_macro_terminology.py::test_macro_string_literals_avoid_xprompt_terms`
  (sase-1hr)
- The raw-prompt and hint failures sase-1hy, sase-1i9, and sase-1ia. sase-1ia also
  covers
  `tests/ace/tui/widgets/test_agent_display_agent_session_render.py::test_agent_session_conversation_sections_are_always_full`.
- `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
  (sase-1ic)
- `test_candidates_fast_path_child_cpu_budget[snippet]` under the parallel lane
  (sase-1g3)
- `test_post_dispatch_foreign_race_on_external_is_exempt` (sase-1hs)
- The Agents deck PNG nodes `test_agents_deck_blocks_spread_deck_sticky_png_snapshot`
  and `test_agents_decks_left_right_search_committed_png_snapshot` (sase-1ii)
- The unused-public Symvision backlog (sase-1hp), including `BeadBoardSnapshot` (owned
  by open epic sase-1h8)

If any other failure appears, prove it reproduces on a clean base before calling it
pre-existing. Record such failures as a `PROPOSED FOLLOW-UP:` note.

**Symvision.** No phase adds a new unused-public entry. If a test needs a seam, keep it
private or give it a real non-test consumer.

**Phase workers never create beads.** Record anything out of scope as a
`PROPOSED FOLLOW-UP:` note on your phase bead.

## 1. `ace` — tint that renders, a stale reopen that submits, 42-cell generic rails

Read the `tui` note's performance rules first. Do not update PNG goldens here; `goldens`
owns them. A targeted non-updating visual run or `sase screenshot` to check layout is
fine.

1. **The chosen-branch tint must survive rendering** (parent plan 6.6.4).
   - `tinted_document_text` (`src/sase/ace/tui/util/frontmatter_syntax.py` ~100-170)
     highlights the folded text and then calls `text.stylize("bold green", ...)` on the
     chosen header line and `"dim"` on unchosen lines.
   - Rich would let the appended span win. But the document pane is a Textual widget,
     and Textual 8 layers spans by start offset. Every syntax token that starts later on
     the line overrides the tint, and Pygments token styles set "not bold".
   - Measured on the goldens from `4bc1db294c`: the chosen line renders
     `fill:#f8f8f2; italic`, with no green and no bold. Only the leading `>` is green.
     Dim lines do render dim (pixels 176,177,171 against 248,248,242).
   - Fix the overlay so the chosen header line renders green and bold. Merge the tint
     into every token span on that line, or split the tint at token boundaries so it is
     layered after each token. Keep the rest of the parent plan's rules:
     - continuation lines of the chosen callout keep their syntax colours;
     - unchosen branches and toggles answered no stay dimmed;
     - nothing is hidden;
     - no extra lexing or YAML parsing per keypress.
   - `test_first_frame_tint_keeps_syntax` inspects only Rich spans, and falls back to
     `_document_renderable` when the widget content is not `Text`, so it cannot catch
     this.
   - Add an assertion on rendered output. Mount the modal, then export the screen (for
     example `app.export_screenshot()`), or render the document widget's lines to
     segments. Assert that:
     - the chosen header line's label text is green and bold;
     - an unchosen line is dim;
     - a non-tinted syntax token (for example a frontmatter key) keeps its theme colour.
2. **A stale review reopened after close must still submit** (parent plan 6.6.6).
   - The closed-modal branch of `_handle_stale_review`
     (`src/sase/ace/tui/actions/agents/_notification_plan_gate.py` ~764-798) reopens
     with a bare `app.push_screen(PlanApprovalModal(**push_kwargs))`.
   - That push has no dismiss callback, and no `action_runner`, `gate_keymaps`,
     `pending_approve_state`, or `copy_plan_path`. This branch is the normal case: the
     modal dismisses on submit, and `stale_review` arrives later. So whatever the
     reviewer submits in the reopened modal is dropped, and its edit action has no
     runner.
   - Reopen through the real open path, `handle_plan_approval`
     (`src/sase/ace/tui/actions/agents/_notification_modals.py` ~66). Pass the reloaded
     data as `_loaded`, or let it load off the pump.
   - Keep restoring the reviewer's values by decision id through `esc_drafts`: ids that
     vanished are dropped, and new ids take `★`. Confirm the reopened modal really reads
     them.
   - `_handle_stale_review` also loads and verifies the bundle synchronously inside
     `on_complete`, on the UI thread. Move the load off the message pump the way
     `handle_plan_approval` does, with `spawn_pump_free_task` and `asyncio.to_thread`.
   - Extend
     `test_plan_decision_ace_stale.py::test_stale_review_reloads_revision_keeping_values`,
     or add a sibling test. After the closed-path reopen, submit the reopened modal and
     assert that the `handle_plan_approval` dismiss handling ran: the submission reaches
     the plan response path with the restored `decision_*` values and the new
     `review_revision`. Also assert that the stale reload does not read the bundle on
     the message pump.
3. **Generic rails go back to 42 cells.**
   - `b49f9bcb28` changed the unscoped rule `.gate-review-body > .gate-review-actions`
     in `src/sase/ace/tui/styles.tcss` (~1766) from `width: 42` to `width: 44`.
   - `CustomGateModal` and `SudoRequestModal` share that rule, so every generic gate's
     rail grew by 2 cells, and `4bc1db294c` regenerated their goldens. That is the
     regression the parent epic forbade ("Generic, non-plan gates must not change").
   - The parent plan (6.6.1) says the plan rail is 42 cells without decisions and 50
     with them (`.gate-review-actions--decisions`).
   - Restore `width: 42` on the generic rule. The `#plan-verdict` rules from
     `4bc1db294c` dropped the button borders, so first measure whether the compact
     Verdict now fits a 42-cell plan rail without decisions. If it does, leave plan
     modals at 42. Only if it does not, give decision-free plan modals their own
     `PlanApprovalModal`-scoped width rule. Never widen a generic modal.
   - Extend
     `test_plan_decision_ace_render.py::test_compact_verdict_stays_inside_rail_with_stylesheet`,
     or add a sibling. Under the real stylesheet:
     - a `CustomGateModal` rail is 42 cells wide;
     - a decision-free plan rail matches what you chose;
     - every Verdict control still lies inside the rail at 120x40 and 90x40.
4. Run `sase tool run check`. Measure keypress cost for the tint change against the
   `tui` note's budget, and record the numbers in your closing note.

## 2. `goldens` — refresh and inspect

1. After `ace` lands, run targeted `just fix-tui-screenshots` through `/sase_monitor`,
   using the `TESTING`/`TESTED` pair and a generous timeout. Target every visual test
   file that owns a plan gate, custom gate, sudo request, plan decisions inbox card, or
   plan toast golden. Start with
   `tests/ace/tui/visual/test_ace_png_snapshots_plan_gate.py` and the `custom_gate`,
   `sudo_request`, `notification_gate`, and `plan_toast` snapshot files. Find any others
   with `grep -l` on the golden names. A targeted run cannot prune stale goldens, and
   this phase needs no pruning.
2. If the run reports `partial`, read the WARNING block and the manifest's `skipped`
   list. The two Agents deck nodes owned by sase-1ii are expected skips if they are
   selected.
3. Open and inspect every updated PNG. Confirm:
   - **Decisions goldens** (`plan_gate_tale_decisions{,_memory,_unverified}_120x40`,
     `plan_gate_tale_decisions_stacked_90x40`, `plan_gate_epic_decisions_120x40`): the
     chosen callout header is green and bold, the unchosen branch is dimmed, and other
     lines keep their syntax colours.
   - **Generic goldens** (`custom_gate_*`, `sudo_request_modal*`): the rail is back to
     its pre-`b49f9bcb28` width. Compare against `git show b49f9bcb28^:<png>` where the
     image predates `4bc1db294c`.
   - **Tale and epic Verdict**: both toggles and Tale/Epic, Reject, and Feedback stay
     inside the rail.
   - **Stacked 90x40**: the Decisions panel stays visible.
   - The unverified `⚠` warning and the memory chips are intact.
4. Any other changed golden is unexpected. Explain it or fix the cause; do not accept it
   blindly.
5. Record what each image shows in your phase note.

## 3. `gate_tests` — the owed route tests

The gate code from `eb646c3d71` is correct, but the plan's owed tests mostly do not
exist. Several named tests do not check what their names say. Add real tests, mostly in
`tests/test_gate_finish_phase.py` or focused siblings. Keep each test file under the
`toobig` limit, and reuse the existing fixtures in
`tests/test_plan_gate_decision_repairs.py`, `tests/test_plan_decisions_handoff.py`, and
`tests/test_plan_gates_execution.py`.

1. **`stale_review`.**
   - Through `execute_gate_selection` (`src/sase/notification_gates/executor.py`
     ~172-245): exactly one `errors/*.json` with code `stale_review`, whose message
     names both the submitted and current revision, and no `response.json`.
   - Through `sase gate answer`, attached and detached. A detached answer re-runs
     `sase gate answer --no-detach`; drive that path. Assert the same single record.
2. **Authored-order stamping on every route.** Assert the stamped file's answers appear
   in the plan's authored decision order, with the right `decided_by`/ `decided_via`.
   Cover:
   - tale approve+commit, approve-only, and commit-only;
   - epic;
   - direct file (`sase plan approve <file>`);
   - hand-run `sase bead work`;
   - the retry re-stamp (`recover_plan_stamp_from_response`).
     `tests/test_plan_direct_approval_run.py` already pins the direct-file order. Reuse
     it; do not duplicate it.
3. **Agent memory refusal end to end.** An agent caller answering a memory decision
   `yes` through `sase gate answer` is refused with `memory_decision_requires_human`. A
   durable `errors/*.json` carries that code, and no stamp or `response.json` is
   written.
4. **A grant for a new strand of an existing web.** Through the resolver, not the guard:
   `glossary:plan-decision` resolves with `exists: false` at
   `sase/memory/glossary/plan-decision.md`, and the grant is allowed.
5. **`decision-host-check-failed`** appears as a diagnostic for both
   `sase plan validate` and `sase plan propose` when the host check raises.
6. **Bead-work caller classification.** A human shell stamps `reviewer`/`cli`, and an
   agent shell (`SASE_AGENT` set) stamps `agent`/`cli`.
7. **Bead-work reuse edge cases**:
   - an agent caller on a reviewer-accepted memory-yes plan succeeds with no resolution;
   - the fresh-stamp branch surfaces `resolved["errors"]`;
   - no `.plan-decisions.json` sibling is written from the reader's environment.
8. **Handoff writer side**:
   - the bundle-to-sibling write;
   - never overwriting an existing sibling;
   - the archive copy plus its commit.
9. **`recover_plan_stamp_from_response` directly.** Cover success, a conflicting stamp,
   missing answers, and a failure recorded with `stage="restamp"`. Today only
   `test_recover_stamp_never_invents_answers` exists.
   - When the restamp fails inside side effects, `adapter_plan.py` (~36-80) records it,
     and `executor_side_effects.py` records it again through `record_failure_outcome`.
   - Make that failure produce exactly one durable record, and test it.
10. **Memory guard through its public entry.** Drive `memory_guard_for_new_markers`
    (`src/sase/finalizers/commit_memory_guard.py`), not internal helpers:
    - a frozen sibling loaded with cwd `/tmp` covers `sase/memory/tui.md`;
    - a granted new strand is covered;
    - a nested hand-written `AGENTS.md` stays "other".
11. **The quiet receipt on the real ACE page.**
    `test_quiet_receipt_visible_on_direct_page` only checks the post function's return
    value. Replace or extend it:
    - post a real `%auto` receipt into a temp notifications store, using the env var or
      fixture the store actually reads;
    - load the modal's direct page through
      `src/sase/ace/tui/actions/agents/_notification_provider_direct.py`, and assert the
      receipt row is present;
    - assert the unread count, toast, and bell paths in `lifecycle.py` skip it.
12. **The single resolver test.** Since the split,
    `test_direct_resolver_freezes_definitions_once` patches the facade's
    `build_definitions`, so it would miss a rebuild inside `plan_decisions_host.py`.
    Patch where the resolver actually looks the name up.
13. Run `sase tool run check`.

## 4. `telegram` — sase-telegram

Work only in the linked checkout. Every decision import stays on
`sase.sdd.plan_decisions` with its feature-detect fallback. Any other sase import is
feature-detected the same way.

1. **Read sase's real coder launch-failure signal.**
   - sase tolerates a tale coder launch failure after acceptance and records it on the
     gate-turn member as `gate_followup_error`, with `gate_followup_error_stage` and
     `gate_followup_error_type`.
   - Callers read it as `followup_error` from
     `sase.gate_turn.store.find_gate_turn_by_gate_id(project, gate_id)`. sase's own
     consumer is `_gate_turn_followup_fields` in `src/sase/_plan_approval_response.py`
     (~203-235), which maps it to `coder_error`.
   - Today `inbound_handlers/gate_completions.py` reads none of that:
     - It reads `bundle_path/"meta.json"` and `response["meta"]` (~531). Neither exists.
     - It passes `response.json` to `current_post_response_failure` (~409, ~519).
       `response.json` has no `acceptance_id`, while journal events carry the one from
       `decision_receipt.json`, so that never matches on current bundles.
     - It treats any `errors/*.json` with stage in `_LAUNCH_FAILURE_STAGES` (~485:
       side_effects, follow_up, coder, launch) as a failed launch. The `side_effects`
       stage also covers restamp, archive/commit, and dismissal failures, so a commit
       failure on approve+commit wrongly reads "coder could not start".
   - Fix:
     - Show "Approved with these choices · coder could not start · retry" only when the
       gate-turn record's `followup_error` is set, or a recorded error is explicitly a
       coder launch failure.
     - Pass the real acceptance identity, from `decision_receipt.json`, to any journal
       reader.
     - Report other post-acceptance failures, such as archive or commit, as what they
       are, not as a coder launch failure.
     - Keep the one later re-edit when the failure lands after the first receipt edit.
   - Replace `test_launch_failure_claim_only_for_recorded_coder_failure`, which
     hand-writes a `code: "coder_launch_failed"` that no sase code emits. Drive these
     shapes:
     - a stubbed `find_gate_turn_by_gate_id` record with `followup_error` (claims the
       launch failure);
     - an archive/commit `side_effects` error (does not claim it);
     - a clean acceptance (does not claim it).
2. **Keep each decision's choices next to its question.**
   - The expandable stage `_split_expandable_parts` (`decision_sheet.py` ~129-185) puts
     every decision's choice lines into one blockquote after all the asks, with no
     decision number or id. That separates choices from their decision, against the
     parent plan's "never cut a decision in half" (6.7.1).
   - Emit one expandable blockquote per choice decision, directly under that decision's
     ask, `★` line, and memory lines. Keep asks, `★` lines, and memory lines outside the
     quotes.
3. **Budget tests through the public renderer.**
   - `test_sheet_three_degradations_only` exercises the stages through helper flags, and
     reaches the blockquote only with `budget=10`.
   - Add fixtures sized so that `render_decision_sheet` at the real 1,800-character
     budget picks stage 1, then stage 2, then stage 3, one fixture per stage.
   - Assert that no MarkdownV2 escape is split anywhere in the output: every `\` is
     followed by an escapable character, not only at the end.
4. **Keyboard pins.**
   - `test_keyboard_exact_rows_tokens_and_style` pins only rows 0–1 and the sub-keyboard
     labels.
   - Pin every row of the Section 1.3 decision keyboard, including each button's label
     and callback token: decision rows, toggles, primary, `↺ Reset`, Reject, and
     Feedback.
   - Assert `style: success` on the primary button of the rendered keyboard, not on the
     `primary_button()` helper.
5. **External settle through the real path.**
   - The ACE and CLI settle tests rewrite `response.json` on a bundle submitted from
     Telegram. The real external path,
     `keyboard_cleanup._settle_externally_resolved_decision`, is only tested negatively.
   - Add positive tests: a gate answered in ACE and one answered from the CLI, with no
     Telegram submit. Each edits the card once into the receipt and removes the
     keyboard.
6. **PDF and the `new` chip.**
   - `test_pdf_accepted_values_callouts_and_cleanup` checks only `"mode" in content`.
     Drive the accepted-values path through `load_stamped_decisions` on a stamped plan.
   - Assert the `new` memory chip text for a grant that creates a new note. Remove the
     unused temp dir.
7. **Stale refresh bumps the revision.** In the stale-then-Refresh test, bump the
   bundle's review revision before Refresh. Assert that the re-rendered card and its
   callback tokens carry the new revision, and the draft is kept.
8. **Keyboard-removal retry.**
   - `_disable_decision_controls` (`gate_response.py` ~193-208) now saves the retry
     before the edit and clears it only on success, but no test covers that. Test that a
     failed removal leaves the retry record and that the sweep retries it.
   - In `_deliver_stale_recovery` (`gate_completions.py` ~733), a sent message followed
     by a failed keyboard restore returns False, so the next poll sends the message
     again. Make the retry restore only the keyboard, and test it.
9. Run `sase tool run check` in sase-telegram.

## 5. Completion evidence

Each phase's closing note lists:

- the items it fixed, each with its test;
- `sase tool run check` results, with any KNOWN failures named;
- confirmation that `sase bead epic-symbols <phase-bead>` is empty.

This epic's land agent re-runs the audit checks above. After it closes this epic, it
resumes sase-1hi.10.7's interrupted landing through the `parent_bead` link: closing
sase-1hi.10.7, then sase-1hi.10, then sase-1hi.
