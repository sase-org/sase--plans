---
tier: epic
title: "Plan Decisions landing repairs: make every surface honor the accepted vector"
goal:
  "Finish epic sase-1hi so Plan Decisions match plan:202610/plan_decisions.md
  everywhere. Stamps keep author order and the true surface. Revisions are checked on
  every submit route. Accepted decisions read the same in any environment. bead read and
  epic inheritance resolve plan: refs. The CLI card and validate JSON are correct. ACE
  has the compact docked Verdict, branch tinting, the edit freeze, and stale handling,
  plus its goldens. Telegram submits, refreshes, and settles correctly for every option.
  The epic-owned Symvision and test regressions are cleared."
phases:
  - id: gate
    title:
      Stamp order and surface, revision binding on every route, kind validation, and
      new-note grants
    depends_on: []
    size: large
    description:
      "gate: keep author order and the true decided_via in stamps, re-stamp from
      response.json on retry, carry review_revision through detached gate answers, make
      kind validation real, fail closed on callers and host checks, let memory grants
      name notes that do not exist yet, and add the missing stamping, stale_review, and
      agent-refusal tests."
  - id: handoff
    title:
      Environment-independent accepted sheets, bead read DECISIONS, epic inheritance,
      guard coverage, and provenance repairs
    depends_on:
      - gate
    size: large
    description:
      "handoff: render accepted decisions from frozen definitions instead of the
      reader's environment, resolve plan: refs for bead read and epic inheritance, make
      guard coverage cwd-independent without false generated-file matches, reach every
      question round in plan_human_text, re-export the helpers Telegram needs, and clear
      the provenance test and Symvision regressions."
  - id: cli
    title:
      Decision card labels, pure validate JSON, scoped completions, CLI tests, and beta
      doc leftovers
    depends_on:
      - handoff
    size: medium
    description:
      "cli: fix the card's clamped-as--D label, duplicate memory chips, and missing
      default stars, keep sase plan validate --json one JSON document, scope -D
      completions to the selected proposal, add the missing plan show/help/handler
      tests, and remove stale beta wording from the docs and README."
  - id: tui
    title:
      ACE compact docked Verdict, branch tinting, edit freeze, carries line, settled and
      stale states
    depends_on:
      - handoff
    size: large
    description:
      "tui: build the compact three-line docked Verdict, tint the chosen branch with a
      fixed classify_callout, show the real Draft not accepted banner, render the
      Carries line, handle settled-elsewhere on refresh and stale_review with a reload,
      fix row text bugs and render-path perf, and clear the four epic-owned Symvision
      symbols."
  - id: goldens
    title: Plan Decisions visual goldens and the compact-Verdict update group
    depends_on:
      - tui
    size: medium
    description:
      "goldens: add the nine Plan Decisions PNG goldens with real gate data and refresh
      the existing plan_gate_* goldens once for the compact Verdict through just
      fix-tui-screenshots under /sase_monitor, inspecting every image."
  - id: telegram
    title:
      Telegram submits every option, refreshes stale cards, and settles with true
      receipts
    depends_on:
      - handoff
    size: large
    description:
      "telegram: send inputs only for selected options, fix the refresh loop and
      stale-after-submit, make settle receipts name the true verdict, decider, surface,
      and defaults, honor the sheet budget, route PDF and receipts through
      sase.sdd.plan_decisions, and repair the nine failing tests plus the missing
      coverage."
proposed_by: bbugyi200.apollo.sase-1hi.land
parent_bead: sase-1hi
create_time: 2026-10-08 05:26:38
status: wip
---

- **PROMPT:**
  [prompts/202610/plan_decisions_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/plan_decisions_landing_repairs.md)
- **PARENT:**
  [202610/plan_decisions.md](https://github.com/sase-org/sase--plans/blob/main/202610/plan_decisions.md)

# Plan Decisions landing repairs

## 0. Scope, ownership, and ground rules

This child epic finishes **sase-1hi** ("Plan Decisions"). The authoritative design is
**plan:202610/plan_decisions.md**. Read it with `sase artifact read`, especially
Sections 1.6, 2, 3, 4, and 6. The sase-1hi land agent audited every phase against that
plan on master `ee3a4f6787`. Every item below is a confirmed gap or bug from that audit,
with source pointers so you can confirm it before changing anything. Line numbers are
from `ee3a4f6787` and may drift.

**Not in this epic.** sase-1hi's resumed landing does these after this epic lands; no
phase here does them:

- the Section 7 end-to-end smoke and the usability report;
- deploying the changed skills (`sase skill init --diff`, then `--force`, per
  `generated_skills`);
- closing sase-1hi and marking its plan done.

The follow-ups the audit triaged are already beads (sase-1hy, sase-1hz, sase-1i0,
sase-1i1, sase-1i2, sase-1i3). Do not re-file them.

**Repositories.**

- `gate`, `handoff`, `cli`, `tui`, and `goldens` work in the sase repo.
- `telegram` works in the linked sase-telegram checkout (`sase repo open sase-telegram`;
  read its `AGENTS.md`).
- No phase is expected to change sase-core. If one must, follow the Rust boundary in the
  parent plan (Section 0): open it with `sase repo open sase-core`, then move
  `sase-core-revision.txt` past the new commit.

**Required reading.**

- Every sase phase: the `lint_and_test` and `symvision` reference notes.
- `tui` and `goldens`: also `tui`.
- `cli`: also `cli_rules`.

Use `sase memory read`. This epic edits no memory notes.

**Verification.** Run `sase tool run check` in every repo you change; never run
`just check-full`.

These failures are known on master and not caused by this epic. Record them as KNOWN and
do not fix them here:

- `tests/test_macro_terminology.py::test_macro_string_literals_avoid_xprompt_terms`
  (sase-1hr);
- `tests/ace/tui/widgets/test_identity_header_raw_prompt.py::test_hinted_raw_prompt_moves_to_identity_and_keeps_its_markers`
  (sase-1hy);
- `test_candidates_fast_path_child_cpu_budget[snippet]` under the parallel lane
  (sase-1g3);
- the unused-public Symvision backlog owned by sase-1hp.

Each sase phase must add **no new** Symvision unused-public entries. It must also clear
the sase-1hi-owned entries assigned to it below. A seam that a later phase of this epic
will consume gets an `--epic-symbol` row keyed to the consuming phase's bead, per the
`symvision` note.

**Phase workers never create beads.** Record anything out of scope as a
`PROPOSED FOLLOW-UP:` note on your phase bead.

## 1. `gate` — stamps, revisions, kind validation, new-note grants

1. **Stamp in author order.**
   - `src/sase/plan_gate_stamp.py:51-68` builds `updated` by iterating the resolved
     vector, and both the core resolver and `_plan_gate_command.py:104`
     (`json.dumps(..., sort_keys=True)`) sort ids. So an authored `zeta, alpha` is
     stamped back as `alpha, zeta`.
   - Iterate the authored `decisions:` map, and apply the same fix to
     `stamp_direct_file`.
   - Test that the authored order survives every stamping route.
2. **Stamp the true surface.**
   - ACE (`_notification_plan_response.py` / `_notification_plan_gate.py`) and live-gate
     `sase plan approve` (`main/plan_approve_handler.py`) both reach
     `execute_gate_selection` through `_plan_approval_response.py:135,160` with
     `source="plan_response"`. `_stamp_coordinates` (`plan_gate_stamp.py:104-127`) then
     maps that to `decided_via: tui`, so CLI approvals are mislabelled.
   - Thread the real surface (`tui` or `cli`) through that shared path.
   - Reject an unknown source in the normalization hook, before acceptance. Today it
     raises `plan_archive_failed` after the decision is already accepted.
3. **Re-stamp on retry from `response.json`** (contract 7).
   - Persist what stamping needs (the source and caller, or the mapped
     `decided_by`/`decided_via`) in the gate response. Today `_gate_source` is read at
     `plan_approval_actions.py:328` but never written.
   - Make every retry path re-stamp the durable plan when it lacks answers, never
     re-resolving or re-asking.
4. **Carry the revision through detached answers.**
   - Plan gates are turn-backed, so a bare `sase gate answer` detaches (`cli_answer.py`
     around :317). `_submit_detached_answer` (:349-357) drops `review_revision` and
     `source`.
   - Forward both. Make a non-integer `review_revision` an input error, not "absent".
5. **Real kind validation.**
   - `_validate_plan_decisions` (`kind_validation/plan.py:216-228`) compares the digest
     with itself, so it can never fail.
   - Round-trip `payload.decisions` through the core bindings: re-derive the digest and
     sheet, and reject any malformed definition.
   - Check that the reject option declares no `decision_*` property.
6. **Fail closed.**
   - `caller_for_decide` (`plan_decide.py:66-75`) fails open to `human`. Use the same
     classifier as `gate_response_caller` (`notification_gates/executor.py:82-96`).
   - The `sase plan validate`/`propose` host checks swallow exceptions
     (`plan_propose_handler.py:124`, `plan_validate_handler.py:106`). Report a
     diagnostic instead.
   - Hand-run `sase bead work` stamping (`bead/cli_work_from_plan.py:301-339`) always
     stamps `decided_by: agent` and swallows every error. Classify the caller properly
     (a human shell is `reviewer` via `cli`) and surface failures.
7. **One direct resolver.**
   - Direct-file approval uses `plan_decide.resolve_direct_decisions`, while the exposed
     `resolve_plan_decisions_for_direct_approval` sits unused.
   - Keep one function, used by `sase plan approve <file>`, `-D`, and hand-run
     `sase bead work`.
8. **Memory grants for new notes.**
   - `sdd/plan_decisions.py:217-276` raises `decision-memory-unresolvable` for a note or
     strand that does not exist yet, and hard-codes `"exists": True`. So the `new` chip
     (parent plan Section 1.6) never appears, and no plan can authorize creating a note.
     This blocks sase-1i3's new `glossary:plan-decision` strand.
   - Resolve a well-formed selector for a missing note, or a missing strand of an
     existing web, to `exists: false` with its would-be path and type. Keep errors for
     bad syntax, unknown scopes, and overlap.
   - Surfaces that already render `exists` from the core sheet show `new`. Verify that
     ACE, the CLI card, and Telegram do; small renderer fixes for the `new` chip belong
     to the `cli`, `tui`, and `telegram` phases.
9. **Tests the gate phase owes.** Add the coverage the parent plan's Section 6.3.13
   asked for and the audit found missing:
   - `stale_review` through `execute_gate_selection` and through `sase gate answer`,
     both attached and detached;
   - stamping on every route: tale approve+commit, approve-only, commit-only, epic,
     direct file, hand-run `sase bead work`, and retry re-stamp;
   - the agent memory refusal end to end through `sase gate answer` (today it is tested
     only at the resolver);
   - a behavioral edit-freeze test, replacing the source grep in
     `tests/test_plan_decisions_gate.py:317`;
   - identical `input_identity` for omitted and explicit defaults on each route you
     touch.

## 2. `handoff` — accepted sheets, bead read, inheritance, guard, provenance

1. **Accepted sheets must not depend on the reader.**
   - `load_stamped_decisions` (`sdd/plan_decision_handoff.py`) calls
     `build_definitions(validation, "")`, which re-resolves host facts from the current
     process's `SASE_ARTIFACTS_DIR` and cwd. As a result:
     - a reviewed "you asked" memory decision redisplays as `quote_not_found`;
     - a verified default-`true` memory row shows `yes ●` (changed) in `sase plan show`;
     - when a selector stops resolving, the sheet and coder block silently vanish.
   - Render accepted decisions from the frozen definitions the reviewer saw. The gate
     bundle's `payload.decisions` is reachable from the response and archive metadata,
     and you may persist it with the stamp.
   - When frozen definitions are unreachable, fall back to a neutral accepted rendering:
     the authored default as `★`, the stamped answer as the value, and no provenance
     warning. Never re-verify a quote against the reader's environment.
   - Every consumer must take this path: the coder block, `sase plan show`, the ACE PLAN
     lane, `sase bead read`, the guard, and Telegram.
2. **`sase bead read` DECISIONS actually appears.**
   - `_resolve_design_file` does not understand `plan:` refs, the real design format.
     `describe_design_reference` already resolves them; reuse that.
   - Phase beads force `tier="tale"` (`bead/cli_detail_decisions.py:92`), but they must
     read their epic's design as an epic.
   - The JSON path omits `plan_roots`/`design_cwd` (`bead/cli_detail_json.py:56`).
   - Render the stored `epic_phase`/`epic_land` audience. Today it is computed but
     ignored.
   - Replace the absolute-path tale fixtures with a real `plan:` ref, a phase bead, and
     an epic bead.
3. **`epic_decision_context`** (`plan_decision_handoff.py:165-216`):
   - Resolve `epic_plan_ref` `plan:` refs through the plan-ref resolver.
   - Add the `phase_bead_id` route and the agent-session route for a phase planner's
     coder successor (parent plan Section 6.4.2).
   - It must still fail closed.
4. **Guard coverage** (`finalizers/commit_memory_guard.py`):
   - Use the frozen resolved paths from item 1, not a cwd-relative re-resolution. Repro:
     finalize from `/tmp`, and an accepted `tui.md` grant resolves to nothing.
   - `_is_generated_root_path` (:73-78) treats any `AGENTS.md`/`CLAUDE.md`/shim at any
     depth as generated, so hand-written nested files (for example
     `src/sase/ace/AGENTS.md`) are treated as covered or flagged wrongly. Limit it to
     the generated instruction outputs and the memory README of the repo whose memory
     changed.
   - Add tests for both cases.
5. **Every question round is quotable.**
   - `sdd/plan_human_text.py` reaches only each plan-turn link's latest question bundle
     through `question_response_path` (sase-1hi.2 note #2).
   - Walk every round of a multi-round `/sase_questions` chain, adding the link it needs
     (bundle to member, or previous bundle) at the point where question rounds are
     created.
   - Test a three-round chain where only the first round holds the quote.
6. **Telegram's API surface.**
   - sase-telegram must consume decisions only through `sase.sdd.plan_decisions`. Today
     it imports `sase.sdd.plan_decision_handoff.load_stamped_decisions` and
     `sase.notification_gates.model_results.effective_response_input`.
   - Re-export the accepted-sheet loader from item 1, the summary builder, and the
     response-input reader that Telegram needs from `sase.sdd.plan_decisions`. Keep the
     names stable for the `telegram` phase.
7. **Provenance regressions from sase-1hi.2.** These still fail on master:
   - `tests/axe/test_agent_meta_atomic.py::test_generic_and_specialized_agent_meta_writers_use_atomic_publication`
   - `tests/test_multi_prompt_launcher_macro_groups.py::test_launch_agents_from_cwd_segment_extra_env_shares_macro_group_counter`
   - `tests/test_multi_prompt_launcher_macro_groups.py::test_launch_agents_from_cwd_force_reuse_marker_applies_to_first_swarm_slot_only`

   Each sees the new `prompt_origin`/`prompt_source_surface` meta or env rows. Update
   the expectations, or exclude the provenance keys where a test pins an exact payload.

8. **Symvision.** Resolve `HumanText` (`sdd/plan_human_text.py`) and
   `prompt_origin_for_launch` / `read_launch_provenance` (`agent/launch_provenance.py`):
   wire, privatize, or delete them.
9. **Docs.** Document the `%auto` receipt's silent/muted and action choice in
   `docs/notifications.md`. Today it is only a docstring in `adapters.py`.

## 3. `cli` — card, validate JSON, completions, tests, docs

1. **Card** (`plan_decide.py:380-410`):
   - The resolver source `clamped` is shown as `-D`. Even with no `-D`, an unverified
     memory row reads `tui_note   no   -D`. Label it as a clamp, for example
     `default · ⚠ quote not found · off`.
   - Memory chips repeat or contradict themselves: `you asked · you asked: '…'`, and
     `⚠ quote not found · off · you asked: '…'`.
   - Unchanged default rows lack `★`, unlike the parent plan Section 1.4 mock.
   - Render the `new` chip from `gate` item 8.
2. **`sase plan validate --json` is one JSON document.**
   - `plan_validate_handler.py:78,96` print human text (the quote-verification line and
     the Decision Sheet) to stdout around the envelope.
   - Put the sheet and the `%auto` note inside the JSON envelope (or send them to
     stderr), and keep the human output unchanged.
   - Rewrite the limitation paragraph in `docs/sdd.md` (around :534-545) to match.
3. **Completions.** `plan_decision_candidates`
   (`completion/candidates/catalog_plans.py`) merges ids from every pending proposal.
   Scope it to the proposal named on the command line when there is one; keep the merged
   fallback.
4. **Tests.** Add the coverage parent plan Section 6.5.6 asked for:
   - `sase plan show` DECISIONS in text, `-f compact` (`◉N 🧠M`), and `-f json`;
   - `sase plan approve -h` help text;
   - handler-level live-gate `-D` submission, which sends `decision_*` plus the
     revision;
   - direct-file `-D` submission;
   - drop the duplicated retry example in `-h`.
5. **Beta leftovers.** The docs refreshes `688c909b28`/`c223047cf4` landed while the
   flag still existed:
   - `docs/cli.md:573` still says "With the Plan Decisions beta enabled";
   - `README.md:141` links `sdd/#plan-decisions-beta` and calls the feature beta.

   Point the link at `#plan-decisions`, then sweep `docs/` and `README.md` for any other
   beta or "no controls" wording.

## 4. `tui` — ACE

1. **Compact docked Verdict** (parent plan Section 6.6.1). It is not built yet:
   - the Verdict is just the rail's last child, with no dock;
   - `gate_branch_controls.py`/`gate_branch_layout.py` still use full toggle labels;
   - the Tale button sits apart from the Reject/Feedback row.

   Build the three docked lines:
   - Line 1: AND toggles with short labels (`☑️ 🚀 Launch coder  ☑️ 💾 Commit plan`),
     the full label as a tooltip.
   - Line 2: `1 ✅ Tale  2 ❌ Reject  3 💬 Feedback`.
   - Line 3: the full summary sentence.

   Use this layout with or without decisions, so it never jumps. Keep 42→50 widening
   only when decisions exist and the 100-column breakpoint.

2. **Rows** (`ace/tui/modals/plan_decision_rows.py`):
   - After a human turns an unverified (`quote_not_found`) row on, the expanded row
     prints `you asked: "<quote>"` (:189-191). Keep the `⚠ … not in your messages`
     wording.
   - Give the expanded choice header its value and `●`.
   - Show the unverified warning chip on collapsed rows too.
   - Render the `new` chip.
3. **Branch tinting** (parent plan Section 6.6.4):
   - Wire `classify_callout` (`plan_decision_document.py`) into the document pane: tint
     the chosen branch green with a bold header, and dim unchosen branches and toggles
     answered no. Never hide a branch.
   - Fix `classify_callout` for `= no` callouts; it ignores `branch` (:159-161).
   - Cache the fold map. `_scroll_to_focused_decision` (:306-331) re-parses YAML on
     every keypress.
   - Remove the unused `render_plan_document(sheet=…)` parameter, or give it its caller.
4. **Edit freeze banner.**
   - The `_apply_edit_outcome` override (`plan_approval_modal.py:815-825`) returns
     before the base class's `controls.set_draft(...)` and `_sync_submission_block()`
     (`gate_action_runner.py:286-291`). So the red "Draft not accepted" banner never
     shows and submit is not blocked.
   - Fix it, and test that submit stays blocked until the draft is resolved.
5. **Feedback `Carries:` line.** `PlanFeedbackContext.carries` is computed
   (`_notification_modals.py:240`) but never rendered. Render the read-only
   `Carries: grouping → mode` first line.
6. **Settled and stale states.**
   - Detect a gate settled elsewhere on notification refresh, not only when the modal
     opens.
   - Label the outcome truthfully: a reject or feedback response must not render as
     "Approved via …" (`_notification_plan_gate.py:166`).
   - Handle `stale_review`: reload the bundle and keep the reviewer's values. Today it
     falls through to the generic error toast.
7. **Render-path performance.**
   - `_load_plan_sheet` (`prompt_panel/_agent_plan_section.py:57,91,115`) runs `os.stat`
     on every render and, on a cache miss, `validate_plan_file` on the UI thread.
   - Move it behind a cached loader or worker, following the `tui` note's perf rules,
     and use the `handoff` accepted-sheet loader.
   - Gate-card toggles use `☑️`/`⬜`, not `◉` (parent plan Section 1.6).
8. **Fixtures.** `_definitions()` in `tests/ace/tui/test_plan_decision_ace.py` is
   hand-drawn, and its memory shape differs from the real payload. Build fixtures with
   `build_plan_approval_gate_spec`.
9. **Symvision.** Privatize `is_unverified_row`, `collapsed_row_text`, and
   `expanded_row_text`, updating the test imports. Wire `classify_callout` through
   item 3.

## 5. `goldens` — visual snapshots

Infrastructure:

- Tests are `pytest.mark.visual` and call
  `ace_png_visual.assert_page_png(page, name, …)`.
- PNGs live in `tests/ace/tui/visual/snapshots/png/`.
- `just fix-tui-screenshots` regenerates them. Run it only through `/sase_monitor`.

Build real gate data: write the plan to tmp and call
`build_plan_approval_gate_spec(plan, "visual-session")`. Then pass
`decision_definitions=spec["payload"]["decisions"]`,
`gate=GateBranchData.from_envelope(spec)`, the plan content, `review_revision`, and
`request_id`. With no artifacts dir, a memory quote fails closed to `quote_not_found`,
which is the unverified case.

1. **Modal goldens.** Add these in
   `tests/ace/tui/visual/test_ace_png_snapshots_plan_gate.py`, extending
   `_snapshot_plan_gate`:
   - `plan_gate_tale_decisions_120x40`
   - `plan_gate_tale_decisions_memory_120x40`
   - `plan_gate_tale_decisions_unverified_120x40`
   - `plan_gate_tale_decisions_stacked_90x40`
   - `plan_gate_epic_decisions_120x40`
2. **Inbox gate card**, pending and answered. Add these in
   `test_ace_png_snapshots_notification_gates.py` through `_modal_with_cached_summary`.
3. **Plan toast with decisions.** Add it in `test_ace_png_snapshots_plan_toast.py`, with
   notes `["Tale ready…", "2 decisions · 🧠 1"]`.
4. **Existing goldens.** Refresh `plan_gate_tale_five_controls_120x40`,
   `plan_gate_frontmatter_120x40`, `plan_gate_epic_action_120x40`, and
   `plan_gate_tale_stacked_90x40` once, as one reviewed group, for the compact Verdict.
5. **Review.** Open and inspect every created or updated PNG before committing, then
   record what each shows in your phase note.

## 6. `telegram` — sase-telegram

Work only in the linked checkout. Nine tests fail today, and six of them pass on the
pre-phase tree `70701a0~1`:

- `test_custom_gates.py`: `test_task_triage_close_uses_required_feedback_flow`,
  `test_task_triage_launch_collects_optional_feedback`,
  `test_registry_drives_resolution_guard_and_inbound_kind_lookup`,
  `test_gate_callback_acknowledges_before_durable_submission`, and
  `test_required_feedback_uses_generic_two_step_text_flow`.
  - The generic required-feedback toast changed to "Reply to the review message with
    feedback" for every gate, not just plan reviews.
  - Callback code reads `view.decisions` on views that lack it.
- `test_telegram_client.py::TestSendMessage::test_long_message_splits_and_attaches_markup_only_to_last`:
  `KeyError: 'reply_markup'` after the `disable_notification` change.
- `test_plan_decisions.py`, three tests:
  - `test_memory_provenance_and_escaping`: create a `sase/memory/tui.md`
    (`type: reference`) under a tmp project and chdir there.
  - `test_submit_merges_identical_vectors_and_revision_metadata`: patch
    `sase.plan_approval_actions._archive_plan_for_approval`, the same way
    `test_custom_gates.py` does.
  - `test_missing_api_disabled_flag_and_generic_compatibility`: the flag is gone. Drop
    the `override_flags(plan_decisions=…)` blocks and the no-op flag fixture.

Fix these behaviors:

1. **Submit only selected options.**
   - `gate_callbacks.py:~179` and `gate_input_steps.py:~174` add `approve`/`commit`
     inputs even when those options are not selected, and merge `decision_*` into every
     selected option, including reject.
   - The executor refuses inputs for unselected options (`unknown_option`), and reject
     has `additionalProperties: false`. So Reject and approve-only or commit-only tales
     fail from Telegram on any plan with decisions.
   - Send inputs only for selected options that declare decision properties.
   - Fix `test_feedback_carries_provisional_vector_and_revision`, which currently
     expects `option_inputs["approve"]` on a feedback-only selection. Add Reject,
     approve-only, and commit-only submission tests.
2. **Refresh and staleness.**
   - `apply_decision_token` checks the revision before it handles `refresh`
     (`decision_callbacks.py:57`), so ↻ loops forever once progress saved an older
     `displayed_revision`.
   - Refresh must re-render the message text (prose and sheet) for the current revision,
     not just the keyboard.
   - A stale rejection after submit must keep a working ↻. Today `gate_response.py:78`
     removes the pending action first.
3. **Settle receipts** (parent plan Section 1.3; `keyboard_cleanup.py:174`,
   `decision_receipt.py:114`, `gate_completions.py:182-309`):
   - Use the true verdict (`coder + commit`/`coder`/`commit`/`epic launch`), the true
     decider and surface (`you via Telegram`, `via ACE`, `via CLI`, `auto`), and Tale vs
     Epic.
   - Give reject and feedback outcomes their own wording; never "✅ Rejected approved".
   - Show the default value in `(★ <default>)`, toggles as yes/no, and the time.
   - Report a missing `response.json` instead of returning False every tick.
   - Treat only a real launch failure as "coder could not start". A still-running
     process is not one.
   - Clear the pending action after an external reject.
   - Edit the card even when `source_message_id` is null, so feedback replies do not
     send a new message.
4. **Sheet budget** (`decision_sheet.py:151-153`, `formatting.py:923,962`):
   - Step 3 must produce a real expandable blockquote.
   - The final fallback must never cut a decision in half; every `ask` and its `★` line
     survive.
   - No fixed-length cut may split a MarkdownV2 escape.
   - Test the 1,800-character budget, the degrade order, and that every ask survives.
5. **Smaller items.**
   - Use the `style: success` helper in `decision_keyboard.py` on the primary button
     when the client library supports it; otherwise delete the helper.
   - Make the AND `x{i}` toggles set values instead of flipping.
   - PDF (`decision_pdf.py`): read decisions through `sase.sdd.plan_decisions` and the
     frozen gate data, not its own YAML parse, and label every line of a callout.
   - Import only from `sase.sdd.plan_decisions`, using the `handoff` re-exports, with
     the existing feature-detect fallback.
   - Render the `new` memory chip.
6. **Docs and tests.**
   - Remove the stale "✅ Approve → `run` payload" paragraph in `docs/outbound.md`
     (around :129-130).
   - Pin the decision keyboard layout.
   - Cover: refresh success, stale-after-submit, settle edits from Telegram, ACE, and
     the CLI, external reject, and an epic decisions gate (`EPIC_DECISIONS_PLAN` is
     currently unused).
   - Run `sase tool run check` in sase-telegram.

## 7. Completion evidence

Each phase's closing note lists:

- the items it fixed, each with its test;
- `sase tool run check` results, with any KNOWN failures named;
- confirmation that `sase bead epic-symbols <phase-bead>` is empty, or what remains
  re-keyed to which later phase.

The epic's land agent then re-runs the audit checks above. After it closes this epic, it
resumes sase-1hi's landing through the `parent_bead` link.
