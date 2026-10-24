---
tier: epic
title: Questions and answers on the Agents Main deck
goal: "Selecting an agent session, the turn that asked, or a question gate turn on the
  Agents tab shows that session's /sase_questions rounds in a QUESTIONS section on the
  Main deck's Context card: each question, its recommended (first-listed) option, the
  actual answer when one exists, and how the round was answered (human via TUI,
  Telegram, mobile, or CLI; automatic policy; agent; unknown). Data is projected from
  the existing question gate bundles, nothing new is written at settlement, and it works
  for every past gate-turn round with no backfill.

  "
decisions:
  placement:
    ask: Where should the new QUESTIONS section sit on the Context card?
    choices:
      before_vars: QUESTIONS renders first, so the human's move leads the Context card
      after_vars:
        Existing sections keep their order and QUESTIONS follows OUTPUT VARIABLES
    default: before_vars
    why:
      The research recommends leading with it; it is usually what you selected the row
      to check
    answer: before_vars
phases:
  - id: qa-bugs
    title: Fix the two latent shared question bugs
    depends_on: []
    size: small
    description:
      "qa-bugs: stop build_merged_qa_markdown from dropping free-text answers and
      comments, and make option-less question rounds park for a human instead of
      crashing auto-resolution."
  - id: core-projection
    title: Core question-round projection, binding, and Python read model
    depends_on: []
    size: medium
    description:
      "core-projection: add the pure sase_core question_round projection and its
      project_question_round binding in sase-core, then the sase facade, typed view
      dataclasses, and a cached single-bundle loader, committing both repos in one turn."
  - id: presenter
    title: Rich presenter and GATE-phase swap for question gates
    depends_on:
      - core-projection
    size: medium
    description:
      "presenter: build the pure Rich Text presenter for question rounds and replace the
      raw Option JSON dump in question gates' GATE section and session Reply phases with
      it."
  - id: lane
    title: Question-round scoping and the questions summary lane
    depends_on:
      - core-projection
    size: medium
    description:
      "lane: add in-memory scoping of question gate rows for container, asking turn, and
      gate row, zero-I/O first-paint stubs, and a new questions DetailHeaderSummary lane
      with gate-state-keyed invalidation."
  - id: context-section
    title: Mount the QUESTIONS section on the Context card
    depends_on:
      - presenter
      - lane
    size: medium
    description:
      "context-section: render QUESTIONS on the Context card from stubs then lane data,
      with the session fold heading, de-duplicate the gate row's GATE section, and add
      header tests, a PNG snapshot, and a j/k perf check."
  - id: docs
    title: Reference docs and the questions skill sentence
    depends_on:
      - context-section
    size: small
    description:
      "docs: document the QUESTIONS section and the question GATE phase in the Agents
      reference pages, and add one informational sentence to the sase_questions skill
      source without deploying it."
proposed_by: bbugyi200.athena.research.58.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 19:29:27
status: wip
---

- **PROMPT:**
  [prompts/202610/questions_main_deck.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/questions_main_deck.md)

# Plan: Questions and answers on the Agents Main deck

## Background and accepted requirements

The user originally asked to set SASE output variables automatically whenever
`/sase_questions` is used, so the Main deck would show each question, its recommended
answer, the actual answer, and how it was answered. A research report reviewed that idea
and the user accepted **all** of its adjusted requirements. Every phase agent should
read it first:

```bash
sase artifact read "research:202610/question_answers_in_agents_main_deck/question_answers_in_agents_main_deck.md" "Design of record for the QUESTIONS section epic"
```

Why the design changed:

- The skill cannot set variables. `sase questions` writes a marker and SIGTERMs the
  asking agent before any answer exists.
- Gate-turn variables would not even be visible. `_output_variable_contributors` only
  walks `(agent, *agent.followup_agents)`, so gates asked by later turns never reach it.
- Variables render as sorted YAML, have size caps, and leak into Jinja, `sase var`,
  mobile, and agents-sync.
- The question gate bundle already holds every fact. The feature is therefore a
  read-time projection of that bundle.

Accepted requirements (R1–R10 in the research):

- **R1.** Cover every way a question gate is created or settled: the skill, the direct
  CLI, `%auto`, and every answer channel. Derive the view at read time.
- **R2.** The gate bundle is the only source of truth. **No output variables in this
  epic**, and no skill bookkeeping steps.
- **R3.** Add a dedicated **QUESTIONS** section on the Main deck's Context card, not
  generic OUTPUT VARIABLES YAML.
- **R4.** Show it when the session container, the asking turn, or the gate turn is
  selected. Deduplicate rounds by gate ID.
- **R5.** Answers are atomic per round. Show every round: pending, answered, timed out,
  cancelled, failed, and lost. Unsubmitted modal drafts are out of scope.
- **R6.** Mark `options[0]` as "recommended (listed first)". Show "no recommendation"
  for option-less questions. No schema change.
- **R7.** Show the channel, the actor (human / auto policy / agent / unknown), the
  policy when recorded, and the time.
  - Mark each answer as followed the recommendation, chose another option, or wrote
    "Other".
  - Make auto-policy answers loud.
  - Never infer "automatic" from an answer that happens to match the recommendation.
- **R8.** First paint shows a compact summary from memory. Full details arrive through
  the existing off-thread lane system. No modal, and no stealing focus, scroll, or deck
  page on refresh.
- **R9.** Works for every past gate-turn round with no migration. The pre-gate
  `question_request.json` layout is best-effort.
- **R10.** Fix the shared Q&A renderer's free-text data loss.

### Facts verified for this plan

- **The bundle.** Each round's bundle is
  `~/.sase/interaction_requests/question/<gate_id>/`.
  - `request.json` holds the questions in `payload.questions`: `question`, optional
    `header`, `options[{label, description?}]`, and `multiSelect`. Its `auto` block can
    carry the deciding `policy`.
  - `response.json` holds `option_results[id=="submit"].result.answers[]` (each with
    `question`, `selected`, and `custom_feedback`). It also has `feedback` (the global
    note), `source`, an optional `caller`, an optional `policy` block, and
    `responded_at_unix`.
  - `cancellation.json` holds `reason` (for example `timeout`), `source` (for example
    `gate_shell_reclaim`), and `cancelled_at_unix`.
- **The local corpus (98 bundles).**
  - 47 are `auto_resolution` without `caller`.
  - 21 are `auto_resolution` with **`caller: "agent"`** and a `policy` block. The actor
    rule must therefore check `source` before `caller`.
  - 19 are `tui` without `caller`, and 1 is `telegram`.
  - 9 have only a `cancellation.json`, and 1 has neither a response nor a cancellation.
- **Gate turn rows** (`<session>--gate`, `--gate-0`, …) already carry these fields on
  the TUI `Agent` model:
  - `gate_id`, `gate_kind` (`"question"`), and `gate_state`
  - `gate_label` (the first ~80 characters of the first question)
  - `gate_creator_agent` (the asking turn's name, for example `0qc--0`)
  - `gate_bundle_path`, `gate_timeout_seconds`, and start times

  Their parent is the asking turn, so later gates attach as that turn's
  `runtime_children`. `concrete_agent_session_turn_rows(container)`
  (`src/sase/ace/tui/models/agent_session_members.py`) emits them in causal order.

- **Today's deck.**
  - The Context card shows nothing about questions.
  - A gate row's Context card appends `build_gate_section`, and a session's Reply card
    renders each gate as `build_gate_phase`. Both are in
    `src/sase/ace/tui/widgets/prompt_panel/_agent_gate_section.py` and share
    `_gate_field_parts`. That function shows a Branches block plus `_response_summary`'s
    raw "Option JSON" dump.
  - The section already reads bundle JSON through
    `sase.agent.artifact_files_cache.get_global_cache().read_json`, which caches by
    mtime and size.
- **The off-thread enrichment system.**
  - `DetailHeaderSummary` (in `_agent_display_state.py`) and
    `build_detail_header_summary` (in `_agent_display_header_summary.py`) build it.
  - Per-lane freshness, batches, and the per-identity cache live in
    `_agent_display_header_summary_cache.py`, along with an existing
    `associated_plan_key` invalidation precedent.
  - `_agent_display_async_agent.py` runs the worker and cancels it when the selection
    changes.

### Shared contracts (all phases)

- **Identity** of an entry is the gate ID plus the question index, never the question
  text. Texts can repeat across rounds.
- **Numbering** is presentation only. Questions are numbered Q1…Qn continuously across
  every question round in the session, in causal order.
  - A question keeps the same number whether you view it from the container, the asking
    turn, or the gate row.
  - This equals the successor prompt's numbering when the session has a single question
    chain, which is the common case.
- **Status** comes from `gate_state`:
  - `pending` and `settling` → `pending`
  - `answered` and `completed` → `answered`
  - `timeout` → `timed_out`
  - `stopped` → `cancelled`
  - `failed` → `failed`
  - `lost` → `lost`
  - anything else → `unknown`

  A durably recorded submit result always yields `answered`, even when `gate_state` is
  `failed` because the successor launch failed. The raw `gate_state` is still echoed, so
  the presenter can warn.

- **Actor:**
  - `source == "auto_resolution"` → `auto_policy`.
  - Otherwise use `caller` (`human` or `agent`).
  - If `caller` is missing and `source` is `tui`, `telegram`, `mobile`, or `cli` →
    `human`.
  - Anything else → `unknown`. Never guess.
- **Recommendation:** `recommended_index = 0` when a question has options, otherwise
  `null`, with `recommendation_basis` set to `"option_order"` or `"none"`.
  `recommendation_match` applies only to answered entries:
  - `followed` when the selection includes the recommended label. For multi-select, the
    first option is one recommended choice, not a recommended set.
  - `other_text` when the selection is only "Other".
  - `free_text` for option-less questions.
  - `different_option` otherwise.
- **Unanswered** answers stay `null`. Never fill them with the recommendation.
- **Missing or malformed files** set `data_state` to `partial` or `unavailable`.
  - Never show an answered round as pending.
  - Never rebuild answers from the successor's prompt.

## Phase qa-bugs: fix the two latent shared question bugs

Python only. This phase is independent and may land first.

1. **Free-text loss in `src/sase/main/qa_markdown.py`.** `_render_question` renders
   `custom_feedback` only inside `if options:`, and only when "Other" is selected.
   Because this renderer builds every successor agent's prompt (via `question_rounds()`
   / `build_merged_qa_markdown`), two shapes silently vanish today:
   - the free-text answer to an option-less question
   - a comment attached to a normal pick

   Fix:
   - Render an option-less answer as its own line, for example
     `- **Answer:** "Must run offline"`.
   - Render a comment on a non-"Other" pick as a line after the options, for example
     `- **Comment:** "Need concurrency"`.
   - Keep the output **byte-identical** for every shape rendered today (options with
     "Other" text, options with no feedback, unknown-label surfacing, the multi-select
     note, and the global note), because the TUI modal preview and successor prompts
     depend on it.
   - Extend `tests/test_qa_format.py`. Keep
     `tests/question_gate_turn/test_rounds_rebuild.py` and the user-question modal tests
     green.

2. **Option-less auto-resolution in `src/sase/user_question_actions.py`.**
   `automatic_question_response` builds `selected: []` with `custom_feedback: None`. The
   response schema's `anyOf` rejects that, so gate creation raises
   `incomplete_question_form` under any auto question policy. Never fabricate an answer:
   - When any question in the round has no options, the gate must not auto-execute. It
     parks for a human exactly like an `ask` outcome: a manual gate is published and the
     notification is sent.
   - Implement this at the Python adapter/evaluation seam, not in core autonomy. The
     relevant code is `automatic_input` in `src/sase/notification_gates/adapter.py` and
     `effective_auto_state` in `src/sase/notification_gates/service_evaluation.py`; an
     adapter feasibility check consulted before `auto_execute` would work.
   - The parked gate's persisted policy block and journal must not claim an automatic
     outcome that never happened. If the block schema cannot express "declined: needs
     human", record a `PROPOSED FOLLOW-UP:` instead of inventing fields.
   - Add tests next to `tests/test_user_question_gates.py` that set an explicit auto
     question policy. The built-in profiles now return `question=ask`.
   - **Known, not yours:** `sase-1o3` already tracks pre-existing failures in that file
     and in `tests/autonomy_contract/` (for example
     `test_auto_question_uses_first_options_without_publishing_pending_action`). Do not
     "fix" those expectations here.

Verify with `sase tool run check`.

## Phase core-projection: core question-round projection, binding, and Python read model

This phase is cross-repo. Open the linked checkout with
`sase repo open sase-core -r "<why>"`, read its `AGENTS.md`, and follow its recipe "Add
a core function and expose it to Python". `sase_core::gate_decision` is the model: a
pure module with versioned `*Wire` serde structs and `thiserror` errors. Its `mod.rs`
holds only `mod` / `pub use` lines, and its tests live in `tests.rs`.

**sase-core:**

- Add a new module `crates/sase_core/src/question_round/` containing `mod.rs`,
  `wire.rs`, `project.rs`, and `tests.rs`, with
  `pub const QUESTION_ROUND_WIRE_SCHEMA_VERSION: u32 = 1`.
- Add
  `project_question_round(&QuestionRoundRequestWire) -> Result<QuestionRoundWire, QuestionRoundError>`
  plus `project_question_round_from_json(&serde_json::Value)`. The function does **no
  I/O**.
- Request wire fields:
  - `schema_version`
  - `gate`: `gate_id`, `gate_state?`, `asked_by?`, `asked_at_unix?`, `round_index?`
  - `request?`, `response?`, and `cancellation?` as raw `serde_json::Value`s, so core
    owns tolerant parsing
- Errors are only for an unsupported schema version or a malformed request wire.
  Malformed bundle content degrades `data_state` instead of erroring.
- Output wire (`QuestionRoundWire`):
  - top level: `schema_version`, `gate_id`, `round_index?`, `asked_by?`,
    `asked_at_unix?`, `status`, `gate_state?`, `data_state`, `entries[]`,
    `global_note?`, `provenance?`, `cancellation?`
  - each entry: `index`, `header?`, `question`, `options[{label, description?}]`,
    `multi_select`, `recommended_index?`, `recommendation_basis`,
    `answer?{selected[], custom_feedback?}`, `recommendation_match?`
  - `provenance`: `channel?` (raw `source`), `actor`,
    `policy?{profile?, rule?, value?, source?}`, `responded_at_unix?`
  - `cancellation`: `reason?`, `source?`, `cancelled_at_unix?`
- Apply the rules from "Shared contracts" above.
- **Answer alignment** mirrors `build_qa_round` in `src/sase/main/qa_prompt.py`:
  - Align by index when the counts match, otherwise by question text.
  - An entry with no match gets `answer = null` and makes `data_state` `partial`.
- **Policy:** use `response.policy`. When it is absent and the actor is `auto_policy`,
  fall back to `request.auto.policy`.
- **Legacy layout (best-effort):** also accept a request with top-level `questions` and
  a response with top-level `answers` / `global_note` (the pre-gate
  `question_request.json` / `question_response.json` shape).
- Tests:
  - every status mapping
  - an answered-then-failed round
  - an auto response with `caller: "agent"`
  - a TUI response without `caller`
  - telegram, and an unknown source
  - multi-select, option-less, and "Other" with a comment
  - duplicate question text, and a 1,000+ character question
  - a missing or malformed response with `gate_state` answered (→ `partial`)
  - cancellation-only bundles
- **Binding** `project_question_round`:
  - Put it in the `sase_core_py` domain that binds `gate_decision`
    (`crates/sase_core_py/src/bead_decisions/`), modeled on
    `py_decide_gate_decision_acceptance`.
  - Register it in that domain's `register_*`.
  - Add a round-trip test in its `tests.rs`.
- Run `sase tool run check` inside the sase-core checkout. It takes about 5 minutes, so
  give it a timeout of at least 10 minutes.

**sase:**

- **Facade.** Add `src/sase/core/question_round_facade.py`, mirroring
  `src/sase/core/gate_decision_facade.py`. It calls
  `require_rust_binding("project_question_round")` and defines
  `QUESTION_ROUND_WIRE_SCHEMA_VERSION = 1`.
  - Add the binding to the required-names list in `tools/validate_sase_core_rs`, plus
    any test that enumerates that list.
  - Add facade tests beside the other `tests/test_core_facade/` modules.
- **Typed read model and loader** in `src/sase/question_gate_turn/projection.py`:
  - Frozen dataclasses mirroring the wire: `QuestionRoundView`, `QuestionEntryView`,
    `QuestionOptionView`, `QuestionAnswerView`, `QuestionProvenanceView`, and
    `QuestionPolicyView`.
  - A tolerant `parse_question_round_wire(dict) -> QuestionRoundView`.
  - `NumberedQuestionRound(view, first_number, member_label, deadline_unix)`. The
    `presenter` and `lane` phases both consume this type, so it lives here.
  - `load_question_round(*, gate_id, gate_state, bundle_path, asked_by, asked_at_unix, round_index=None) -> QuestionRoundView`:
    - Reads `request.json`, `response.json`, and `cancellation.json` through
      `get_global_cache().read_json`, then calls the facade.
    - Any binding or parse error returns a `data_state="unavailable"` view. It never
      raises.
    - It is thread-safe for the lane worker.
- **Loader tests** build fixture bundles in tmp dirs using the real shapes listed in
  "Facts verified for this plan".
- **Commit both repos in this one turn.** The host commits the pinned sase-core sibling
  first and writes its pushed SHA into `sase-core-revision.txt` automatically (see
  `docs/contribute/rust-backend.md`, "The CI source revision pin"). Do not hand-edit the
  pin.
- If the workspace venv's `sase_core_rs` lacks the new binding, rebuild the local core
  with `just rust-install`. It is long, so run it through `/sase_monitor` if needed.
- Do not move `validate_user_questions` or `question_rounds()` into core, and do not add
  to them.

Verify with `sase tool run check` in sase.

## Phase presenter: Rich presenter and GATE-phase swap for question gates

Python only; depends on `core-projection`. If the venv's `sase_core_rs` predates the new
binding, run `just rust-install` first.

1. **Presenter module.** Add
   `src/sase/ace/tui/widgets/prompt_panel/_agent_questions_render.py` (the name is
   flexible). It is pure (no I/O, with `now` injected) and returns **Rich `Text` only**:
   no `Syntax` or JSON renderables, because hint annotation and `gate_phase_text`
   require Text parts. API:
   - `render_question_rounds(rounds: Sequence[NumberedQuestionRound], *, level: FoldLevel, now) -> Text`
     returns the section body.
   - `question_rounds_heading_suffix(...) -> Text` returns, for example,
     `· 3 rounds · 1 awaiting answer`, omitting zero parts. It must accept both full
     rounds and the `lane` phase's first-paint stubs. Define a tiny protocol or accept
     `(gate_state, …)` tuples so neither phase blocks the other.
   - A single-round helper for the GATE section and phase.
2. **Visual vocabulary.** The research mock is the target:

   ```text
   QUESTIONS · 3 rounds · 1 awaiting answer
    gate     ANSWERED  human via TUI · 3m after asking
      Q1 Memory edits      → Core memory: Final Declaration (+1)              ≠★
      Q2 Routing           → Yes, opt-in                                       ★
    gate-0   ANSWERED  ⚠ auto policy (standard · gates.question=first) · no human
      Q3 At the Mac        → Done — helper printed > 268, then synced          ★
    gate-1   QUESTION  awaiting answer · asked 14:02 · 23h left
      Q4 Ship flag default?
         ★ Off (recommended, listed first)  ·  On  ·  Remove flag
   ```

   - **Status words** reuse the question gate turn's vocabulary from
     `src/sase/question_gate_turn/create.py`: `QUESTION`, `ANSWERED`,
     `QUESTION TIMED OUT`, `QUESTION CANCELLED`, and `QUESTION FAILED`, plus a `lost`
     word consistent with the gate row. Accents are `#FFAF00` pending, `#5FD7FF`
     answered, and `#FF5F5F` failed. Reuse existing constants where they exist.
   - **Provenance:**
     - `human via TUI|Telegram|mobile|CLI`
     - `agent via <channel>`
     - `answered (source unknown)`
     - auto policy: `⚠ auto policy (<profile> · <rule>=<value>) · no human`, with the
       warning accent on the round header line
     - elapsed time such as `3m after asking` when both timestamps exist
     - for pending rounds, `awaiting answer · asked HH:MM · <remaining> left`, using
       `deadline_unix`
     - for an answered round whose `gate_state` is `failed`, add `· follow-up failed` as
       a warning
   - **Markers:** `★` only for `recommendation_match == followed`; `≠★` for
     `different_option` and `other_text`; none for `free_text`. An option-less pending
     question shows `(free text, no recommendation)`. Multi-select answers list all
     labels.
   - **Degraded data:** `partial` / `unavailable` render a dim
     `answer details unavailable — sase gate show <short id>` line (use
     `short_gate_turn_id`). Never render them as pending.
   - **Fold levels:**
     - `COLLAPSED` renders no body; the mount shows only the heading.
     - `EXPANDED` renders one header line per round and one line per question
       (`Q<n> <header or clipped question> → <answer> <marker>`). The pending round,
       else the latest round, also lists its options with `★` on the recommended one.
     - `FULLY_EXPANDED` expands every round: full wrapped question text, option
       descriptions, "Other" text, per-question comments, the global note, the cancel
       reason, and exact times.
     - One-line rows clip with `…`; long text wraps only when fully expanded.

3. **GATE swap** in `_agent_gate_section.py`. When `agent.gate_kind == "question"`,
   `_gate_field_parts` replaces the Branches block and `_response_summary`'s "Reviewer
   note" / "Option JSON" dump with the presenter's single-round render.
   - Load the round with `load_question_round` from the row's `gate_bundle_path`. This
     is the same cached-read pattern the section already uses on this path; add no new
     reads elsewhere.
   - Session Reply phases (`build_gate_phase`, called from
     `_agent_display_agent_session_render.py` and `_agent_display_render_main.py`)
     should number questions continuously by accumulating `first_number` over the
     question gate phases they render in causal order. Where no offset is available,
     start at Q1.
   - Keep the Decision, Kind, Status, State, Elapsed, Deadline, Gate id /
     `sase gate show`, Bundle, Next action, and Follow-up fields. Other gate kinds stay
     byte-identical.
   - Check the hint-mode path (`_agent_display_hint_sections.py`, which calls
     `build_gate_section`) still annotates.
4. **Tests:**
   - Presenter unit tests on plain text, covering every acceptance scenario below.
   - Gate-section tests for question and non-question kinds.
   - Regenerate any PNG golden that showed a question gate's Option JSON with a targeted
     `just fix-tui-screenshots -- <selectors>` run through `/sase_monitor` (read
     `tui_screenshot.md` and `lint_and_test.md` via `/sase_memory_read` first), and
     inspect the report.

Verify with `sase tool run check`.

## Phase lane: question-round scoping and the questions summary lane

Python only; depends on `core-projection`. Read `tui_perf.md` with `/sase_memory_read`
first. Rules 1, 5, 8, and 10 govern this phase. This phase changes **no rendering**; the
`context-section` phase mounts the section.

1. **In-memory scoping** in a new `src/sase/ace/tui/models/agent_question_rounds.py`.
   Everything here is pure and does zero I/O.
   - `session_question_gate_rows(agent)`:
     - Resolve the session container: the row itself when
       `is_agent_session_container_row`, else `agent_session_roster_container(agent)`.
       For a row with no container, use its own `runtime_children`.
     - Walk `concrete_agent_session_turn_rows(container)` and keep rows where
       `row.is_gate and row.gate_kind == "question"`.
     - Deduplicate by `gate_id` (falling back to identity) and keep the causal order.
     - Cap defensively at 50 rounds, keeping the newest.
   - `question_rows_in_scope(agent)` returns:
     - all rows for a container
     - the row itself for a question gate row
     - for any other turn, the rows whose `gate_creator_agent` equals that turn's name,
       or that are among its `runtime_children`
   - `question_round_stubs(agent)` returns first-paint data per in-scope round:
     `gate_id`, the member label suffix (`gate`, `gate-0`, …), `gate_state`,
     `gate_label`, the asked time, and `deadline_unix` (from `gate_timeout_seconds` and
     the start time).
   - `questions_cache_key(agent)` returns `tuple((gate_id, gate_state), …)` over the
     session's question rows.
2. **The `questions` lane:**
   - Add `"questions"` to `DetailContextLane` and `ALL_DETAIL_CONTEXT_LANES`, and add
     `question_rounds: tuple[NumberedQuestionRound, ...] = ()` to `DetailHeaderSummary`
     (all in `_agent_display_state.py`).
   - Set `_LANE_FIELDS["questions"] = ("question_rounds",)` and give it the default 10 s
     refresh interval.
   - Put it in batch 1 of `LANE_RESOLUTION_BATCHES`: a handful of mtime-cached reads.
     Move it to batch 2 if a trace shows otherwise.
   - It is **not** a SASE CONTEXT lane. Do not add it to `CONTEXT_LANE_ORDER`, and make
     sure `append_agent_context_section`'s pending-lane indicators ignore it.
3. **The resolver** goes in `_build_detail_header_summary_impl`, under a child
   `tui_trace` span `.question_rounds`.
   - Load a `QuestionRoundView` for **every** session question row, which is needed for
     session-wide numbering.
   - Compute each round's `first_number`, then keep only the in-scope rounds as
     `NumberedQuestionRound`s.
4. **Invalidation:**
   - Store `questions_key` on `DetailHeaderSummaryCacheEntry`. On a mismatch,
     `should_refresh_detail_header_summary` must include `"questions"` without dropping
     the entry or the other lanes' freshness.
   - Response rewrites that add `policy` after creation are caught by the mtime/size
     cache on the next 10 s revalidation.
   - Confirm that an answer settling while the row is selected reaches re-enrichment
     through the **existing** refresh path. Settlement calls `touch_turn_refresh_pulse`,
     rows reload with the new `gate_state`, and the key mismatches. Add no new refresh
     code path.
   - Reuse the selection-change cancellation unchanged.
   - `immediate_detail_header_summary` stays zero-I/O and must never mark `questions`
     ready.
5. **Tests:**
   - Scoping tests beside `tests/ace/tui/models/test_agent_session_members_turns.py`.
     Reproduce the on-disk parentage of session `bob-cli-1n.1`: `--gate` has parent
     `--plan`, `--gate-0` has parent `--1`, and `--gate-1` has parent `--2`. Cover all
     three selection kinds, dedupe, causal order, and numbering offsets.
   - Lane and batch-coverage tests in
     `tests/ace/tui/widgets/test_agent_display_header_summary_lanes.py`.
   - Cache-key invalidation tests, and the trace test if spans are enumerated
     (`test_agent_display_header_summary_trace.py`).
   - Keep `tests/perf/bench_detail_header_summary.py` working.

Verify with `sase tool run check`.

## Phase context-section: mount the QUESTIONS section on the Context card

Python only; depends on `presenter` and `lane`. Read `tui.md`, `tui_perf.md`, and
`tui_screenshot.md` with `/sase_memory_read` first.

1. **Mount** `append_agent_questions_section(text, agent, *, summary, fold_level, now)`
   in `build_header_text` (`_agent_display_header.py`). By default, call it immediately
   **before** `append_agent_output_variables_section`.

   > [!decision] placement = after_vars Call it immediately after
   > `append_agent_output_variables_section` instead, so OUTPUT VARIABLES keeps its
   > current position.
   - Use section id `questions` and title `QUESTIONS`, with the presenter's heading
     suffix.
   - On container rows, follow OUTPUT VARIABLES exactly: use
     `append_agent_session_fold_heading` with
     `effective_agent_session_fold_level("questions", …)`, so the default session fold
     shows the body and `za` collapses it. Other rows use `append_section_heading`.
   - Precede the section with `append_major_section_divider`, like its neighbors.

2. **Data source:**
   - When `summary` is present and `"questions"` is in `summary.ready_lanes`, render the
     presenter over `summary.question_rounds`.
   - Otherwise render the first paint from `question_round_stubs(agent)`: the heading
     with counts, plus one dim line per round (label, status word, `gate_label`). This
     path does zero I/O.
   - Render no section at all when the scope has no question rows.
3. **Gate rows.** A gate row's Context card is the header plus `build_gate_section`.
   Once its QUESTIONS section shows its own round, drop the presenter body from
   `build_gate_section` for question gates; the GATE fields stay.
   - Session Reply GATE phases keep the presenter.
   - Apply the same rule in the hint-mode builder.
4. **Behavior checks:**
   - `za` / `zA` reach the `questions` anchor, and `,/` deck search finds question text.
   - `Ctrl+J` / `Ctrl+K` card stops do not change (QUESTIONS is a section, not a card).
   - The lane landing never steals focus, scroll, or the deck page. `sase-1ln`
     (same-subject re-delivery stealing Reply back to Context) is a known bug; do not
     make it worse.
5. **Tests:**
   - Header tests: container, asking turn, and gate row; first paint vs. ready; each
     fold level; placement; and no section when there are no rounds.
   - A new PNG snapshot (for example in
     `tests/ace/tui/visual/test_ace_png_snapshots_agents_agent_sessions.py` with
     fixtures beside `_ace_agents_png_snapshot_agent_session_panel_fixtures.py`) of a
     session with three rounds: a human-answered round with one deviation (`≠★`), an
     auto-policy round, and a pending round. Generate it with a targeted
     `just fix-tui-screenshots -- <selector>` through `/sase_monitor`, inspect the
     report, and refresh any other golden this mount changes.
6. **Performance:** `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` must stay at p95 <
   16 ms with a session holding several question rounds. Use `SASE_TUI_TRACE=1` to read
   the `.question_rounds` span.

Verify with `sase tool run check`.

## Phase docs: reference docs and the questions skill sentence

Depends on `context-section`. Read `docs_writing.md` and `generated_skills.md` with
`/sase_memory_read` first.

1. **`docs/reference/tui/agents-main-deck.md`:** add a QUESTIONS entry beside the
   existing Main deck bullets. It should cover:
   - what the section shows, and that the recommendation means "listed first"
   - the `★` / `≠★` markers and the auto-policy warning
   - the selection rules (container, asking turn, gate row)
   - first paint vs. full detail, and fold behavior
   - the session-wide numbering note

   Also describe the question gate's GATE section / Reply phase now showing the round
   instead of Option JSON. Touch `docs/reference/tui/agent-statuses.md` (the "Main deck"
   bullet) only if its wording becomes wrong.
   - Stay within page budgets, and run `just fix` for the generated docs.

2. **The skill source** (`src/sase/macros/skills/sase_questions.md`): add exactly one
   informational sentence, for example under "Recommended Option First". It should say
   that your questions, with the first option shown as the recommendation, and their
   answers, with how they were answered, appear in the session's QUESTIONS section on
   the Agents tab.
   - Add no bookkeeping steps.
   - Preview with `sase skill init --diff`, and update any test that pins the rendered
     skill text.
   - **Do not deploy.** Deploying (`sase skill init --force`) happens only from a
     landed, clean tree. The land agent proposes it through `/sase_gate` after landing.

Verify with `sase tool run check`.

## Acceptance scenarios (owned by the phase named)

- **presenter, context-section:** one round with two questions pending, then one
  submission with a normal pick, an "Other" answer, and a global note.
- **lane, context-section:** an answered round 1 followed by a pending round 2, viewed
  from the container, the asking turn, and the gate row. Each round appears once, in
  causal order, with stable Q numbers.
- **core-projection, presenter:**
  - `%auto` rounds with and without a `policy` block, including `caller: "agent"`, and
    older responses with no `caller`
  - timeout, cancel, failed, and lost rounds, and an answered round whose successor
    launch failed
  - duplicate question text across rounds, multi-select, option-less questions, and
    1,000+ character questions
- **lane, context-section:** a response arriving while the session is selected, and fast
  `j` / `k` during loading against the p95 < 16 ms target.
- **qa-bugs:** an option-less free-text answer and a comment on a normal pick both reach
  the successor prompt markdown, and an option-less question under an auto policy parks
  for a human.

## Out of scope

- Output variables of any kind, including the research's optional per-gate `questions`
  export. Add one only when a named consumer appears.
- Toasts or Telegram lines for auto-policy answers, and an explicit `recommended` field
  in the question schema.
- Saving unsubmitted modal drafts.
- Clan and tribe Summary cards, mobile, Telegram, and agents-sync pages.
