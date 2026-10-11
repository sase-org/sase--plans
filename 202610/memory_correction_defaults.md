---
tier: tale
title: Default memory decisions on for objective corrections
goal:
  Planners can default a memory decision to true, with a host-verified `corrects:`
  quote, when the edit objectively corrects a false or misleading claim in the note
  (including one the plan's own work makes false), and the skills, explain text, and
  docs say so.
size: medium
decisions:
  plan_skill_hint:
    ask:
      Also add one sentence to /sase_plan telling planners to plan memory corrections
      their work requires?
    default: true
    why:
      Planners rarely load /sase_memory_write unless they already noticed the stale note
    answer: true
proposed_by: bbugyi200.athena.0zq
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 13:52:13
status: wip
---

# Default Memory Decisions On For Objective Corrections

## Goal

Today a plan's memory decision may default to `true` only when it carries a `requested:`
quote of the user's complete affirmative request. Every other memory decision defaults
to `false`. The user wants a second legitimate reason for a `true` default: **the edit
is an objective correction of a false or misleading claim in the note**. That includes a
claim the plan's own work will make false. For example, the plan renames a command,
flag, recipe, or config key that a memory note documents, or it removes behavior a note
describes.

The rule is not only prose. It is enforced mechanically:

- The Rust grammar rejects a memory default of `true` without `requested:`
  (`decision-requested-missing`).
- The Python host rejects a `requested:` quote it cannot verify
  (`decision-requested-unverified`).
- The Rust payload clamps the effective default to `false` unless the host verified it.

Rewording the skills alone would just make agents fail validation. This plan therefore
adds a verified correction justification end to end, then updates the agent-facing
wording to match.

## Design

### The `corrects:` field

A memory toggle may carry `corrects: <quote>` instead of `requested:`. The quote is one
line of 3–300 characters, copied verbatim from the note's **current** text. It names the
claim that is false or misleading, or that the plan's work makes so. With a verified
`corrects:` quote, a memory decision may default to `true`.

Rules:

- `corrects` is allowed on memory toggles only, as `requested` is.
- A decision carries at most one of `requested` and `corrects`. When the user asked for
  the change, `requested` wins.
- `why` stays forbidden on memory decisions. The quote and the `ask` explain the
  default.
- Nothing forbids `corrects` on a `default: false` decision. It does nothing there, just
  as `requested` does nothing there today.

New guidance, rewording, preferences, and additions that fix nothing false are not
corrections. They keep defaulting to `false`. A memory decision whose only target is a
note that does not exist yet cannot carry a verified correction, because there is no
claim to quote.

### Verification

The Python host verifies `corrects:` the same way it verifies `requested:`, with the
existing lexical matcher binding `plan_decision_quote_match`. The searched texts differ:

- `requested:` is matched against human-authored text.
- `corrects:` is matched against the current raw text of the memory files that decision
  covers. These are the same files `resolve_memory_records` counts for the decision:
  flat notes, plus a web's descriptor and strands. Missing (future-grant) targets
  contribute no text.

The matcher already normalizes case and whitespace and strips edge punctuation from each
token. So Markdown backticks and bold markers around words do not block a match, and
line wraps in the note collapse.

No new Rust binding is needed. Only the host-supplied text set changes.

### Provenance and the single verified flag

Add two provenance values to the existing four (`asked`, `not_asked`, `quote_not_found`,
`inherited`):

- `correction`: the `corrects:` quote matched the note text.
- `claim_not_found`: a `corrects:` quote is present but did not match, for example
  because the note drifted. This fails closed.

A verified correction reuses the existing host fact `requested_verified: true`. That
follows the `inherited` precedent, where inherited epic authorization is also sent as
`requested_verified: true` without a quote. Do **not** add a second verified flag. The
Rust clamp, the resolver's agent boundary, and the two frozen-definition paths already
key off the frozen `requested_verified` and `provenance`, so they keep working
unchanged:

- the adapter re-freeze in `src/sase/notification_gates/adapter_plan.py`
- the rebuild and digest check in `src/sase/notification_gates/kind_validation/plan.py`

Update the doc comments to state the broadened meaning: "the authored true default is
verified (`asked`, `correction`, or `inherited`)". Keep the field name for wire
compatibility.

The frozen definition gains `corrects` with `skip_serializing_if = "Option::is_none"`.
Existing frozen gates therefore keep their digests. The sheet's memory `quote` becomes
`requested`, falling back to `corrects`. Readers tell the two apart by provenance.

Under no-human review, the default is applied as is:

- A verified correction is on. The coder block authorizes the edit and files nothing.
- A `claim_not_found` row is off. It joins the "unrequested" set (`not_asked`,
  `quote_not_found`), so the coder records the skipped change as a follow-up and the
  correction is not silently lost.
- A human who turns any row off still files nothing.

## Changes

### 1. sase-core (linked repo)

Open it with `sase repo open sase-core` and work only in the printed path. All paths
below are under `crates/sase_core/src/plan/`.

1. **`decisions/grammar.rs`**
   - Add `corrects` to `DECISION_FIELDS` after `requested`.
   - Update the hard-coded allowed-field list in the `decision-unknown-field` message.
   - Parse `corrects` beside `requested`, with the same one-line 3–300 rule and the same
     string check. Add three codes:
     - `decision-corrects-not-memory`: "decision `{id}` field `corrects` is allowed on
       memory toggles only"
     - `decision-corrects-invalid`: "…must be one line of 3-300 characters" and "…must
       be a string, found {type}"
     - `decision-corrects-with-requested`, on field path `.corrects`: "decision `{id}`
       carries both `requested` and `corrects`; keep `requested` when the user asked for
       the change"
   - Relax `decision-requested-missing` so it fires only when neither quote is present.
     Keep the code, and change the message to "…must quote the human request in
     `requested` or the claim it corrects in `corrects`".
   - Change the `decision-why-on-memory` message tail to "its `requested` or `corrects`
     quote explains it".
2. **`decisions/wire.rs`**: add `corrects: Option<String>` to `PlanDecisionWire`,
   serde-skipped when `None`. Fill it in the grammar where `PlanDecisionWire` is built.
3. **`validate.rs` (`plan_frontmatter_schema`)**: add a `decisions.<id>.corrects` spec
   directly after `requested`.
   - Type: "quoted false claim for a memory decision defaulting to true".
   - Description: "Memory toggles only: one line of 3-300 characters copied from the
     note's current text, naming a claim that is false or misleading or that the plan's
     own work makes so. Lets the memory default be true without a human request;
     exclusive with `requested`."
   - Change the `requested` description to end "Required when the authored memory
     default is true and no `corrects` quote is given."
   - Change the `why` description tail to "the requested or corrects quote explains
     those."
4. **`decisions/resolver.rs`**
   - Add `correction` and `claim_not_found` to `DECISION_PROVENANCES`, and to the
     "unknown decision provenance" message's expected list.
   - Add `corrects` to `PlanDecisionDefinitionWire`, serde-skipped when `None`, and copy
     it in `plan_decisions_payload`.
   - Update the `PlanDecisionHostFactWire` doc comment as described in Design. The clamp
     logic does not change.
5. **`decisions/sheet.rs`**
   - In the sheet memory wire, change `quote` to
     `definition.requested.clone().or_else(|| definition.corrects.clone())`.
   - Add both provenances to the `check_sheet_row` list and its message.
   - Add `claim_not_found` to the `unrequested` `matches!` in the prompt-block declined
     rule, and update its comment: verified `asked`, `correction`, and inherited rows
     left off stay quiet.
6. **`decisions/quote.rs`**: update the module doc. The host may also supply memory-note
   text for `corrects:` verification. Rust still never reads memory files itself.
7. **Tests**
   - Extend `decisions/tests.rs`:
     - `corrects` is accepted on a default-true memory toggle with no `requested`
     - each of the three new codes fires
     - `requested-missing` still fires with neither quote
     - updated `why` and unknown-field messages
     - exact wire ordering with `corrects`
   - Extend the resolver tests:
     - `correction` plus `requested_verified: true` keeps `true`
     - `claim_not_found` plus `false` clamps
     - `corrects` is copied into the definition and is absent from JSON when `None`, so
       the digest of a definition without `corrects` is unchanged
   - Extend the sheet tests:
     - the quote falls back to `corrects`
     - the provenance table in `auto_follow_ups_only_for_unrequested_memory_off` gains
       `correction` (quiet) and `claim_not_found` (follow-up)
     - the unknown-provenance test still rejects other strings
   - `sheet.rs` is already about 1,464 lines and AGENTS.md caps files at 1,500. Put new
     sheet tests in a sibling test module rather than growing `sheet.rs` past the cap.
   - Update the pinned schema-order tests:
     `schema_is_ordered_and_contains_exact_phase_guidance` in `validate.rs`, and the
     field list in `crates/sase_core_py/src/plans/tests.rs`.
   - Update the binding doc comment in `crates/sase_core_py/src/plans/decisions.rs` that
     lists the host-fact keys, if it describes provenance values.
8. **Commit type**: this is a wire change older cores reject, because they reject the
   new provenance strings. Per sase-core's AGENTS.md, mark the sase-core commit `feat!:`
   with a `BREAKING CHANGE:` footer. Never edit versions or changelogs.

### 2. sase (this repo)

1. **`src/sase/sdd/plan_validate.py`**: add `corrects: str | None` to `_PlanDecision`
   and parse it from the core payload next to `requested`.
2. **`src/sase/sdd/plan_decisions_wire.py`**: send `corrects` to the core alongside
   `requested`, only when it is not `None`.
3. **`src/sase/sdd/_plan_decisions_shared.py`**
   - Add a public helper, for example `memory_quote_texts(selectors)`. It returns
     `[{source: "memory", ref: <canonical relative path>, text: <raw_text>}]` for every
     existing file `resolve_memory_records` would count for those selectors:
     - each flat note's `content.raw_text`
     - each web section's `web.raw_text`
     - each node's `strand.raw_text`
   - Raw text deliberately includes frontmatter, so a stale `description:` can be
     quoted.
   - Future-grant (missing) targets contribute nothing.
   - Add `CORRECTS_UNVERIFIED_CODE = "decision-corrects-unverified"` and export it.
4. **`src/sase/sdd/plan_decisions_host.py`**
   - **`_build_host_facts`**: the branch order is `requested` (unchanged), then
     `corrects`, then `not_asked`. The `corrects` branch matches the quote against
     `memory_quote_texts(selectors)`:
     - a match gives `{requested_verified: True, provenance: "correction"}`
     - no match, no texts, or a matcher exception gives
       `{requested_verified: False, provenance: "claim_not_found"}`
   - Keep the fail-closed style.
   - **`validate_host_checks(strict_quotes=True)`**: for a default-true memory decision
     whose verification failed, emit an ERROR diagnostic that depends on which quote it
     carries:
     - `requested`: keep `decision-requested-unverified` exactly as today.
     - `corrects`: emit `decision-corrects-unverified` on field path
       `decisions.<id>.corrects`. The message is
       `memory decision '<id>' corrects quote was not found in <comma-joined note paths>`.
       When the matcher returns a closest hint from the note texts, append
       `; closest: "<sentence>"`. Generalize `_closest_sentence_for` to take the text
       list instead of duplicating it.
5. **`src/sase/notification_gates/kind_validation/plan.py`**: add `correction` and
   `claim_not_found` to the frozen-provenance whitelist.
6. **Display**
   - **`src/sase/sdd/_plan_display_decisions.py`**: add chips
     `"correction": "corrects note"` and `"claim_not_found": "⚠ claim not found · off"`.
   - **`src/sase/main/plan_decide.py` card**
     - Treat `claim_not_found` like `quote_not_found`: drop ` · off` when the value is
       true.
     - Render `correction` with a quote as ` · corrects: '<quote>'`.
   - **`src/sase/ace/tui/modals/plan_decision_rows.py`**
     - `_is_unverified_row` is also true for `claim_not_found`.
     - The warning line names the source by provenance. `quote_not_found` keeps
       `— not in your messages`. `claim_not_found` reads
       `⚠ "<quote>" — not in the note · off until you turn it on`.
     - The expanded detail prints `corrects: "<quote>"` for `correction` rows instead of
       `you asked:`.
     - Read the `tui` reference memory first. New PNG goldens are not required; cover
       the text in unit tests.
   - **`src/sase/sdd/plan_decision_handoff.py` `_receipt_memory_lines`**
     - An on `correction` row renders `corrects: '<quote>'`, not `you asked:`.
     - An off `claim_not_found` row renders
       `⚠ 🧠 <id> = no · claim not found — left off because no human reviewed this plan`.
7. **`src/sase/main/feedback_prompt.py`**: change the provisional memory note to
   `(not authorization; default it on only by quoting human feedback text in `requested:`or the false claim in`corrects:`)`.
8. **`src/sase/main/plan_explain.py` `PLAN_DECISIONS_GUIDANCE`**: replace the
   memory-default sentence with:

   "A memory decision defaults to `true` only with one quote that justifies it:
   `requested:` (3–300 chars) quotes the user's complete affirmative request, or
   `corrects:` (3–300 chars) copies verbatim from the note's current text a claim that
   is false or misleading, or that the plan's own work makes so. Otherwise it defaults
   to `false`; never carry both."

   Keep the rest of the paragraph.

9. **`src/sase/macros/skills/sase_memory_write.md`**
   - Replace the Routing paragraph "Authoring a plan whose steps change memory?" with:

     ```markdown
     **Authoring a plan whose steps change memory?** Author one memory decision per
     changed note in the plan's `decisions:` frontmatter. Default it to `true` when
     either holds:

     - The user asked for the change: quote their complete affirmative request in
       `requested:`.
     - The change objectively corrects a false or misleading claim in the note,
       including one the plan's own work will make false (a renamed command, a removed
       flag, a changed default): copy that claim verbatim from the note into
       `corrects:`.

     Otherwise default it to `false`; new guidance, rewording, and additions that fix
     nothing false are not corrections. Use one quote, never both. Do not write "do not
     edit memory" disclaimers: a plan with no memory decision authorizes no memory
     edits.
     ```

   - In Authorization, extend the last sentence to "…or your own conclusion that a note
     is wrong. In a plan, that conclusion only sets a memory decision's default (see
     Routing)." It still never authorizes a direct edit.
   - Keep every phrase pinned by `tests/main/test_init_skills_sources.py`.
   - Never write the literal `%auto` in a skill source. A test forbids it; say "no human
     reviewed the plan".

> [!decision] plan_skill_hint

10. **`src/sase/macros/skills/sase_plan.md`**: in step 4, after "Order decisions by
    importance with memory decisions last.", add: "If the planned work would leave a
    memory note stating something false, plan that correction too, under a memory
    decision authored with your `/sase_memory_write` skill." Pin the new phrase in
    `test_init_skills_sources.py` if that test pins `sase_plan` phrases.

> [!decision] plan_skill_hint = no

10. Leave `sase_plan.md` unchanged.

11. **Docs**
    - **`docs/sdd.md`** (the memory-decision paragraphs, about lines 669–683): add the
      `corrects:` route and its verification against the note's current text. Add the
      `correction` and `claim_not_found` provenances. Keep the statements that agent
      context and proposal reject unverified true defaults.
    - **`docs/cli.md`** (about 608–612 and 647–649): "human-request quotes" becomes
      "human-request and corrected-claim quotes".
    - **`docs/notifications.md`** (the no-human-review receipt around 1022–1036, and the
      memory boundary around 1523): mention correction rows and `claim_not_found`.
    - Grep `docs/` for `requested` quote prose and update any other enumeration of the
      rule.
12. **Tests**
    - Update the schema field list in `tests/test_plan_validate.py`
      (`test_facade_rehydrates_valid_tale_and_ordered_schema`).
    - Add `tests/test_plan_decisions_corrects.py`, using the fixture style of
      `tests/test_plan_decide_cli_card.py` and `tests/test_plan_decisions_gate.py`, with
      a temporary project memory note. Cover:
      1. A verified `corrects` gives provenance `correction` and
         `effective_default: true` through `build_definitions`.
      2. A quote absent from the note gives `claim_not_found` and is clamped to `false`.
      3. Strict `validate_host_checks` reports `decision-corrects-unverified` on
         `decisions.<id>.corrects` with a closest hint.
      4. A `corrects` quote on a missing (future-grant) note never verifies.
      5. With no human review, a verified correction stays on and its coder block has no
         follow-up sentence. A `claim_not_found` row is off and gets the
         `/sase_new_task` follow-up.
      6. A frozen gate with both new provenances passes kind validation, and its digest
         round-trips.
    - Add display assertions for the new chips, the card, the TUI row text, and the
      receipt lines. Extend the existing modules where natural.
    - Add a test for the existing `decision-requested-unverified` diagnostic. It has no
      direct coverage today.

## Verification

1. In the sase-core checkout, run `sase tool run check`.
2. In sase, run `just rust-install` so the venv binding is built from the edited linked
   sase-core checkout. It can be long; use `/sase_monitor` if needed. Then run the new
   and touched tests.
3. Run `sase tool run check`, following the `lint_and_test` reference memory.
4. Validate a scratch tale plan in agent context with
   `sase plan validate <file> --explain`:
   - A memory decision with `default: true` and a `corrects:` quote copied from an
     existing note passes.
   - The same decision with an altered quote fails with `decision-corrects-unverified`
     and a closest hint.
   - With both `requested` and `corrects`, it fails with
     `decision-corrects-with-requested`.

   Do not propose it.

When one turn commits both repos, the host commits sase-core first and moves
`sase-core-revision.txt` automatically. Do not bump the pin by hand.

## Out of Scope

- **Skill deployment.** Edited skill sources reach the managed chezmoi locations through
  `sase skill init --force` from the clean, landed tree. That is a post-landing step,
  not part of this change.
- **Memory notes.** No memory note restates this rule. The generated `sase.md` pointer
  ("…or propose a plan that would — use your `/sase_memory_write` skill") stays
  accurate. This plan edits no memory.
- **Agents outside plans.** Authorization for agents not working from a plan is
  unchanged. An agent that finds a stale note outside a plan still files a `memory` task
  bead.
