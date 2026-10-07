---
tier: epic
title: 'Plan Decisions: typed reviewer choices answered inside the plan review'
goal: 'A tale or epic can declare up to five typed, defaulted Plan Decisions (toggles,
  2-5-way choices, and memory consents) in a `decisions:` frontmatter map. The reviewer
  answers them in the same plan review on ACE, Telegram, or the CLI, and the primary
  action always approves exactly the values on display. `%auto` takes verified defaults
  and posts a quiet receipt. Answers are recorded once in the gate response and stamped
  into the archived plan. Every implementer receives them mechanically, and a plan
  authorizes a memory edit only through an accepted memory decision.

  '
phases:
- id: core
  title: sase-core decisions grammar, resolver, quote matcher, and Decision Sheet
  depends_on: []
  size: large
  description: 'core: in the linked sase-core repo, add the `decisions:` grammar and
    diagnostics (including the system-written `answer`/`decided_by`/`decided_via`
    fields, a new Archived validation mode, branch callouts, and the reserved `phases[].when`),
    the additive validated-plan wire, the frozen payload definition record, the default/clamp
    resolver, the human-quote matcher, the Decision Sheet with its summary sentence
    and implementer prompt block, a definitions digest, and pyo3 bindings with tests.'
- id: provenance
  title: Durable human-authorship provenance for prompts and gate answers
  depends_on: []
  size: medium
  description: 'provenance: record `prompt_origin` on every agent launch and `caller`
    on every gate response, then add one gatherer that returns only human-written
    text from a planner''s chain (typed root prompt, human feedback bullets, human
    free-text Q&A answers), failing closed everywhere.'
- id: gate
  title: Compile, resolve, freeze, and stamp decisions in the plan gate
  depends_on:
  - core
  - provenance
  size: large
  description: 'gate: move the sase-core pin, adapt the Python wire and Archived mode,
    scaffold the `plan_decisions` beta flag, resolve memory scope and verify quotes
    at validate/propose/gate build, compile `decision_<id>` raw schema properties
    and `payload.decisions`, pin them in kind validation, normalize inputs before
    the receipt, bind submissions to the displayed review revision, freeze definitions
    during review, carry provisional values into feedback replans, and stamp immutable
    answers into the durable plan on every approval route.'
- id: handoff
  title: Deliver accepted decisions to coders, phases, notifications, and receipts
  depends_on:
  - gate
  size: medium
  description: 'handoff: append the host-written Reviewer decisions block to tale
    coder prompts, show DECISIONS in `sase bead read` and the phase/land macros, carry
    an epic''s accepted decisions into phase sub-plans, post the quiet `%auto` receipt
    notification, and add the shared Rich decision display builders.'
- id: cli
  title: Decision-aware sase plan and sase gate commands
  depends_on:
  - handoff
  size: medium
  description: 'cli: add `sase plan approve -D/--decide ID=VALUE` with live completions,
    the decision card, dry-run, retry and agent-boundary messages, plus decision output
    for `plan show`, `plan list`, `plan validate`, `plan propose`, and `gate show`.'
- id: tui
  title: ACE Decisions section, compact Verdict, and decision-aware inbox
  depends_on:
  - handoff
  size: large
  description: 'tui: build the ACE Decisions accordion above a compact docked Verdict
    with the outcome sentence, new gate keys, a document pane that folds the frontmatter
    and lights the chosen branch, draft persistence, revision-bound submits on every
    path, a settled-elsewhere state, toast/inbox/gate-card/PLAN-lane decision rendering,
    and new visual goldens.'
- id: telegram
  title: Telegram decision sheet, live keyboard, and settle receipt
  depends_on:
  - handoff
  size: large
  description: 'telegram: in the linked sase-telegram repo, render the static question
    sheet, the live set-value decision keyboard with choice sub-keyboards and the
    summarizing primary button, revision-bound submits with a stale-card refresh,
    the one-time settle edit, quiet `%auto` receipts, the feedback reply fix, decision-aware
    PDFs, and typed-origin agent launches.'
- id: guard
  title: Advisory finalizer memory guard
  depends_on:
  - handoff
  size: medium
  description: 'guard: add a host-side, never-blocking finalizer check that warns
    when an agent launched from an approved plan changes a memory note no accepted
    memory decision (its own or inherited from its epic) covers.'
- id: policy
  title: Planner and memory-skill policy, authoring docs, and flag removal
  depends_on:
  - cli
  - tui
  - telegram
  - guard
  size: medium
  description: 'policy: teach planners when to embed a decision instead of asking
    now, rewrite the memory-write authorization routes around memory decisions, document
    the authoring grammar in `--explain` and the SDD docs, delete the beta flag''s
    Off branch, and record the follow-ups, including the dogfooded memory-decision
    plan for this feature''s own memory notes.'
proposed_by: bbugyi200.apollo.5n
create_time: 2026-10-07 18:48:19
status: wip
bead_id: sase-1hi
---

- **PROMPT:** [prompts/202610/plan_decisions.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/plan_decisions.md)
- **BEAD:** [sase-1hi](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1hi/README.md)

# Plan: Plan Decisions — typed reviewer choices answered inside the plan review

## 0. Sources and ground rules

This epic implements the two accepted research reports. Read them with
`sase artifact read`; do not open the sidecar files directly:

- `research:202610/plan_frontmatter_decisions/plan_frontmatter_decisions.md` (the
  baseline: decisions, not gate options).
- `research:202610/plan_decisions_cross_surface_ux/plan_decisions_cross_surface_ux.md`
  (the cross-surface UX, which refines the baseline in C1–C8).

Bryan accepted every recommendation in both reports. Section 5 lists where this plan
goes beyond them because source checks found a verifiable problem. Where this plan and
the reports disagree, this plan wins.

**Rust boundary.** Grammar, validation, resolution, quote matching, the Decision Sheet,
the summary sentence, and the implementer prompt block live in the linked `sase-core`
repo (`sase repo open sase-core`). Python adapts and orchestrates, and ACE, Telegram,
and the CLI render. A phase that calls a new binding must move `sase-core-revision.txt`
past the sase-core commit that adds it (see `docs/rust_backend.md`, "The CI source
revision pin").

**Naming.** The product term is **Plan Decision**. In code, always say `plan_decisions`
(`plan/decisions.rs`, `sdd/plan_decisions.py`, and so on). That keeps it apart from gate
decision receipts (`notification_gates/decision.py`), `bead_decisions`, and the
`decisions` memory web.

**Required reading per phase.**

- Every sase phase: the `lint_and_test` reference note. For Symvision epic-symbol
  entries, also the `symvision` note.
- `core`: sase-core `AGENTS.md`. Note that `plan/validate.rs` is already above that
  repo's file-size guideline, so add new modules rather than growing it.
- `gate` and `policy`: `sase_flags`.
- `cli`: `cli_rules`.
- `tui`: `tui`, `tui_perf`, `tui_screenshot`.
- `telegram`: sase-telegram `AGENTS.md`.
- `policy`: `generated_skills`.

Read reference notes with `sase memory read`.

## 1. What Bryan sees

The whole design follows one rule: **approve as shown.** Every surface shows the
decisions next to the plan with their current values filled in, and you can change a
value without submitting anything. The primary action takes exactly the values on
display: Enter in ACE, the green button in Telegram, a bare `sase plan approve`, and
`%auto`. Accepting the planner's defaults stays one keystroke.

### 1.1 Authoring (the planner)

```yaml
---
tier: tale
title: Keymap help overlay
goal: Pressing ? in ACE shows every active binding, grouped for scanning.
size: small
decisions:
  grouping:
    ask: How should the overlay group bindings?
    choices:
      pane: By pane, matching the footer hints
      mode: By leader mode; denser, but splits pane actions
    default: pane
    why: pane keeps the footer's order
  tui_note:
    ask: Record the overlay's keymap conventions in the tui memory note?
    memory: [tui.md]
    requested: "and note the convention in the tui memory"
    default: true
---
```

In the body, an optional callout marks the prose each answer selects. Nothing is ever
stripped or executed:

```markdown
> [!decision] grouping = pane Order sections by pane (Agents, Artifacts, Services),
> reusing the footer hint order.

> [!decision] grouping = mode One section per leader mode; pane-local actions sit under
> the mode that owns them.

> [!decision] tui_note Add a "Help overlay" bullet to `tui.md` stating the grouping
> rule.
```

`sase plan validate` prints the Decision Sheet, so the planner sees exactly what the
reviewer will see. `sase plan propose` echoes `Plan Decisions: 2 (🧠 1)`.

### 1.2 ACE Plan Review

```
┌ Plan Review · claude/opus ──────────────────────────────────────────────────────┐
│ e ✏️  Edit plan                               │ keymap_help_overlay.md           │
│                                               │ ---                              │
│ Decisions                           2 · 🧠 1  │ tier: tale                       │
│▌◉ grouping                      ‹ mode › ●    │ title: Keymap help overlay       │
│▌  How should the overlay group bindings?      │ decisions: 2 · answered in panel │
│▌   ○ pane  By pane, matching the footer  ★    │ ---                              │
│▌   ◉ mode  By leader mode; denser, but        │ > [!decision] grouping = pane    │ ← dimmed
│▌           splits pane actions                │ > Order sections by pane …       │
│▌  ★ pane keeps the footer's order             │ > [!decision] grouping = mode    │ ← lit
│ ☑️ 🧠 tui_note                        yes ★    │ > One section per leader mode …  │
│     tui.md · reference · you asked            │                                  │
│                                               │                                  │
│ Verdict                                       │                                  │
│ ☑️ 🚀 Launch coder    ☑️ 💾 Commit plan         │                                  │
│ 1 ✅ Tale   2 ❌ Reject   3 💬 Feedback         │                                  │
│ → coder + commit · grouping=mode ● · 🧠 tui.md│                                  │
└───────────────────────────────────────────────┴──────────────────────────────────┘
  Enter=Tale · 1 change   space/h/l change   r reset   R reset all   j/k move   e edit
```

- Unfocused rows take two lines: glyph, id, and value, then the ask truncated or the
  memory chips. The focused row (cyan bar) expands to show the full ask, every choice
  with its consequence, the `★` default with its `why`, and for memory rows the note
  chips and the quote.
- The modal opens with the first decision focused, so the question and its alternatives
  are visible without a keypress. Enter still approves at once.

### 1.3 Telegram

```
📋 CLAUDE(opus) Plan Review  @5m--plan
Tale ready for review: keymap_help_overlay.md
2 decisions · 🧠 1

Decisions · 2
1. grouping — How should the overlay group bindings?
   ★ pane · By pane, matching the footer hints — pane keeps the footer's order
     mode · By leader mode; denser, but splits pane actions
2. 🧠 tui_note — Record the overlay's keymap conventions in the tui memory note?
   ★ yes · tui.md (reference) · you asked: "and note the convention in the tui memory"
━━━━━━━━━━━━━━━━━━━━
🧾 Properties …   ▸ Implementation …

[ ◉ grouping: mode ● ▾ ]
[ ☑️ 🧠 tui_note ]
[ ☑️ 🚀 Launch coder agent ] [ ☑️ 💾 Commit plan file … ]
[ ✅ Tale · 1 change ]
[ ↺ Reset ] [ ❌ Reject ] [ 💬 Send Feedback ]
```

- The message text never changes while you edit. Taps edit only the keyboard.
- `☑️`/`⬜` rows flip in place, and `▾` rows open a radio sub-keyboard:
  `[○ pane ★] [◉ mode ●] [↩ Back]`.
- When the gate settles here or anywhere else, the message is edited once into its own
  receipt:

```
✅ Tale approved · you via Telegram · 14:02
1. grouping → mode ● (★ pane)
2. 🧠 tui_note → yes ★
→ coder + commit · grouping=mode ● · 🧠 tui.md
```

### 1.4 CLI

```
$ sase plan approve keymap_help_overlay -D grouping=mode --dry-run
◇ Dry run · tale · keymap_help_overlay · review 4
  grouping   mode ●   -D        was ★ pane
  tui_note   yes  ★   default   🧠 tui.md · you asked
  → coder + commit · grouping=mode ● · 🧠 tui.md
nothing was approved (dry run)
```

- A bare `sase plan approve` approves at once and prints the same card without the
  dry-run line. The card is the confirmation, not a prompt.
- Bad ids or values fail before any side effect. The error prints a did-you-mean and the
  allowed values with `★` on the default.

### 1.5 `%auto` receipt

About three in four plans settle under `%auto` and today they settle in silence. Every
auto-approved plan **with at least one decision** posts one quiet informational
notification:

```
🤖 Auto-approved tale · keymap_help_overlay
grouping = pane ★ (auto)
🧠 tui.md · you asked: "and note the convention in the tui memory"
⚠ 🧠 glossary_term = no · not asked — left off because no human reviewed this plan
```

### 1.6 Shared visual language

- `☑️`/`⬜` mark a toggle as yes/no, and Space flips it.
- `◉` marks a choice row and the selected option; `○` marks an unselected option. Do not
  use `◆` (task beads, epic phases), `⋔` (gate turns), or `🎛` (launch control).
- `★` marks the planner's default: what Enter, ✅, and `%auto` take.
- A gold `●` (`#FFD700`, the Config pane's modified colour) marks a value changed from
  its default. Never rely on colour alone; detail views also show `★ default`.
- `🧠` marks a memory consent. It is always paired with the note selector in text.
- Memory provenance chips read `you asked`, `not asked`, `⚠ quote not found · off`, or
  `approved in epic`. Never say "verified consent".
- Note chips read `core` (warning colour, "loaded every turn"), `reference`, `web`, or
  `new`.
- Toggle values display as `yes`/`no`. YAML stays `true`/`false`.
- The decision **id** is the one name on every surface: it is what ACE and Telegram show
  and what `-D` takes.
- Every surface and the archive keep author order and never re-sort.

**The summary sentence** is built in sase-core and is identical on every surface.

- **Full form:** `→ <verdict> · <id>=<value>[ ●] · … · 🧠 <notes>`.
  - The verdict is `coder + commit`, `coder`, `commit`, or `epic launch`.
  - Non-memory decisions follow in author order: choices as `id=key`, toggles as
    `id=yes|no`.
  - Then one memory clause: `🧠 tui.md, glossary:help-overlay` for the notes that will
    be edited, or `🧠 no memory edits` when every memory decision is off. The clause is
    omitted when the plan has no memory decisions.
- **Short form:** `defaults` or `N change(s)`, plus ` · 🧠` when any memory edit is on.

## 2. Authoring grammar

`decisions:` is an optional top-level map keyed by decision id, valid on tales and
epics. YAML map order is author order. A decision with `choices:` is a **choice**; one
without is a **toggle**. A toggle with `memory:` is a **memory decision**.

| Field                        | Rule (diagnostic code on violation)                                                                                                                                                                                                                                                                |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| map                          | At most 5 decisions (`decision-limit`). Unknown fields inside a decision are errors (`decision-unknown-field`).                                                                                                                                                                                    |
| id                           | `^[a-z][a-z0-9_]*$`, at most 32 chars (`decision-id-invalid`). Not a YAML 1.1 bool/null word (`y n yes no on off true false null`). Not a reserved name: `approve commit reject feedback coder_prompt coder_model wait epic_launch_mode capacity` (`decision-id-reserved`).                        |
| `ask`                        | Required, one line, at most 120 chars (`decision-ask-invalid`). Warning if it does not end in `?` (`decision-ask-not-question`). Phrase toggles so yes means do the work.                                                                                                                          |
| `choices`                    | Choices only: 2–5 entries (`decision-choices-count`). Keys follow the id rules except the reserved list, at most 24 chars. Labels are one-line consequences of at most 100 chars (`decision-choice-invalid`).                                                                                      |
| `default`                    | Required (`decision-default-missing`). A toggle takes a YAML boolean only; `yes`/`on` strings fail with a "use true/false" hint. A choice takes one of its keys (`decision-default-invalid`).                                                                                                      |
| `why`                        | Optional, one line, at most 100 chars. Rejected on memory decisions (`decision-why-on-memory`).                                                                                                                                                                                                    |
| `memory`                     | Toggles only (`decision-memory-on-choice`). A non-empty list of `sase memory read` selectors: `note.md`, `web`, `web:keyword`. Core checks syntax (`decision-memory-selector-invalid`). The host resolves the notes and rejects overlapping grants (`decision-memory-overlap`).                    |
| `requested`                  | Memory decisions only (`decision-requested-not-memory`). One line, 3 to 300 chars. **Required when a memory decision defaults to `true`** (`decision-requested-missing`). The host checks it word for word against human-written text (`decision-requested-unverified`, Section 4).                |
| `answer`                     | System-written. Forbidden in Authoring mode (`decision-answer-forbidden`). In Launch and Archived modes it must be a valid value for the decision.                                                                                                                                                 |
| `decided_by` / `decided_via` | Top-level, system-written. Forbidden in Authoring. `decided_by` is `reviewer`, `auto`, or `agent`. `decided_via` is `tui`, `telegram`, `mobile`, or `cli`, and is absent for `auto`. In Archived mode, `decided_by` and per-decision `answer` must appear together (`decision-answer-incomplete`). |
| `phases[].when`              | Reserved; always an error in v1 (`phase-when-reserved`).                                                                                                                                                                                                                                           |

**Callouts.** A line of the form `> [!decision] <id>`, `> [!decision] <id> = <key>`, or
`> [!decision] <id> = no` outside fenced code starts a callout. The callout runs to the
end of that blockquote.

- A bare toggle id marks the yes branch, and `= no` marks the no branch.
- An unknown id or key is an error (`decision-branch-unknown`).
- A decision whose id appears neither in a callout nor as a whole word in the body gets
  a warning (`decision-unreferenced`).

**Memory heuristic warning (`decision-memory-uncovered`).** It fires when the body
mentions `sase memory init`, or names a `sase/memory/` path on a line with an edit verb
(add, edit, update, create, delete, remove, rewrite), and no memory decision covers that
note. Tune it against the plan archive: the ≥20 archived memory-editing plans should
warn, and read-only references ("read the tui note") should not.

**The Archived validation mode is new.** It is as strict as Authoring, and it also
allows the system-written answer fields. Committed-plan validation
(`just validate-committed-plans`, `validate_plan_for_commit`) moves from Authoring to
Archived. Launch mode also accepts answers.

## 3. Reliability contract

These are release acceptance criteria. Phases cite them by number.

1. **Every value is visible before approval** on ACE, Telegram, and the CLI card. No
   generic input UI ever shows a decision: the ACE input panel, the `c` dialog,
   `✎ n inputs` badges, Telegram's step flow, or a duplicate inbox field list.
2. **Editing a value never approves.** In ACE, Enter cannot edit. In Telegram, picking a
   choice returns you to the card. In the CLI, `-n` previews.
3. **Omitted means default.** The host fills omissions and clamps unverified memory
   defaults _before_ the receipt identity is computed. An omitted value and an explicit
   default then produce the same accepted vector and the same `input_identity`, from
   every surface, including `%auto`, Android, and older clients.
4. **One vector is accepted.** Tale `approve` and `commit` never resolve independently,
   and differing values submitted for them are rejected (`decision_conflict`).
5. **Approval refers to the displayed revision.** ACE and Telegram submit the
   `review_revision` they displayed, and a mismatch is refused (`stale_review`) with a
   refresh path.
6. **Definitions are frozen for a review.** An in-gate edit may change prose but never
   anything under `decisions:`, including `ask` wording, labels, and `why`. To change
   the questions, send feedback.
7. **Accepted answers are immutable.** Retries reuse them. A crash between acceptance
   and stamping never re-asks or re-resolves: recovery re-stamps from `response.json`.
8. **"Approved" and "implementation started" are separate states** on every surface.
9. **Memory edits need a human or a verified quote.** An agent process can never switch
   a memory decision on through any route it can reach: live-gate `sase plan approve`,
   `sase gate answer -O`, or the direct-file route. `%auto` takes a memory default of
   yes only when its quote was verified.
10. **Fail closed.** Unknown provenance, a missing quote source, or an unresolvable epic
    context counts as "not requested".

## 4. Memory consent

**Policy (lands in `policy`).**

- A plan, tale or epic, that changes memory must cover every changed note with a memory
  decision.
- A memory decision defaults to `true` only when the user asked, and `requested:` then
  quotes their words. Otherwise it defaults to `false`.
- A plan with no memory decision authorizes no memory edits, so planners stop writing
  "do not edit memory" disclaimers.
- A specific instruction in the human's current prompt still authorizes an agent that is
  not planning. That prompt is already the human gate.

**Human-written text (`provenance` phase).** A quote is checked only against text a
human wrote in the planner's chain:

- the chain root's `submitted_prompt.md` (the launch-boundary text before alias or macro
  expansion), only when that root's `prompt_origin` is `typed`;
- plan-gate feedback bullets from responses whose `caller` is `human` and whose source
  is not `auto_resolution`;
- `/sase_questions` free text (`custom_feedback`, `global_note`) under the same rule.
  Selected option labels are agent-written and never count.

Prompts written by agents never count: LaunchApproval children, `sase bead work` phase,
land, and task prompts, chop jobs, and session successors.

**Matching (`core`).** Normalize both sides: NFKC, casefold, straight quotes and dashes,
whitespace collapsed, and leading and trailing punctuation stripped from the quote. The
quote must then appear as a contiguous word sequence in one human text.

- On failure the matcher returns the closest human sentence by token overlap, so
  `propose` can print it and the planner can fix the quote in the same turn.
- A match is evidence, not a seal: "edit tui.md" also occurs inside "do not edit
  tui.md". The skill therefore tells planners to quote the complete affirmative request,
  and every surface shows the quote itself.

**When the check runs.**

- At `sase plan validate` inside an agent, and at `sase plan propose`: an unverified
  quote is an error while the planner can still fix it.
- At gate build: an unverified quote fails closed. The default is forced to `false`, the
  provenance becomes `quote_not_found`, and every surface flags it.

**Epic inheritance (`handoff`).** Phase planners always run under `%auto` with
agent-written prompts, so they can never verify a quote. A phase or land agent's
sub-plan therefore **inherits** its epic's accepted decisions:

- An epic memory decision answered yes authorizes those notes for every phase, every
  phase sub-plan's coder, and the land agent. The provenance chip reads
  `approved in epic`.
- Sub-plans must not redeclare an inherited decision. Notes beyond the epic's grant
  still need their own memory decision, which defaults off.

**Declined changes.** When `decided_by: auto` left an unrequested memory decision off,
no human saw it.

- The coder records the skipped change with `/sase_new_task` as a `memory` task bead.
  `/sase_new_task` corroborates an existing bead instead of duplicating it.
- Phase workers record a `PROPOSED FOLLOW-UP:` instead.
- When a human turned it off, nothing is filed.

## 5. Judgment calls beyond the research

Each item was verified against the source. To change any of them, send feedback on this
plan.

1. **Archived validation mode.** Committed-plan validation and
   `validate_plan_for_commit` run in Authoring mode. They would reject every stamped
   archive if Authoring forbids `answer:`, so a third mode is required (Section 2).
2. **YAML 1.1 words are banned as ids and choice keys.** `sase plan propose` rewrites
   frontmatter through PyYAML (`sdd/frontmatter.py`), which reads `no:` as `false`. A
   choice key `no` would silently become `false` between propose and the gate.
3. **Durable provenance is new work.** No agent record says whether a human typed its
   prompt today. `source` on gate responses is self-reported, and question-gate sources
   are lost downstream. The `provenance` phase adds `prompt_origin` and a response
   `caller`, both fail-closed.
4. **Epic inheritance (Section 4).** Without it, an epic's human-approved memory change
   could never be carried out by a large phase, whose sub-plan is auto-approved with
   agent-written prompts.
5. **The normalization hook covers both input paths.** Gate inputs are resolved
   separately for the receipt (`decision.py`) and for execution (`executor.py`). One
   adapter hook runs before both.
6. **The agent memory guard is in the resolver**, not only in `sase plan approve`.
   Live-gate `sase plan approve` and `sase gate answer` have no agent check today.
7. **The guard ships advisory and unflagged.** Agent declarations have no warning
   channel, so the guard emits host-side finalizer warnings. Enforcing mode becomes a
   follow-up decision for Bryan after a soak period.
8. **This epic edits no memory.** The `Plan Decision` glossary strand, a decision
   record, and the `generated_skills` questions line become one follow-up memory plan:
   the feature's first live memory decision, reviewed by Bryan in the new Decisions
   panel.
9. **Some generic gate-input fixes are follow-ups.** Decisions use raw schema properties
   (C1), so the `TypedInputForm` bool toggle and the Telegram step-flow "Keep default"
   change are no longer on the decision path.
10. **`%auto` planners are told.** `sase plan validate` and `propose` say
    "auto-approved: every decision takes its default". The skill then says that under
    `%auto` a planner embeds only memory decisions and makes every other choice itself.
11. **Android.** The separate client is not linked here. It approves with defaults and
    shows the `2 decisions · 🧠 1` note line. Its Decision Sheet rendering is a
    follow-up, and nothing on Android is labelled "as shown".

## 6. Phases

### 6.1 `core` — sase-core

Work in the linked sase-core checkout (`sase repo open sase-core`). Put new code in
`crates/sase_core/src/plan/decisions.rs` (split into a `decisions/` module directory if
it grows), and keep `plan/validate.rs` changes to small hook calls.

1. **Schema.** Add `decisions` to the common field list. Add `decided_by` and
   `decided_via` as system fields, forbidden in Authoring. Implement every rule in
   Section 2 through the existing `Validator` and `push` diagnostics. Extend
   `SourceIndex` so `decisions.<id>` and `decisions.<id>.<field>` resolve to real line
   numbers. Add `PlanValidationMode::Archived` (parsed from `"archived"`).
2. **Field-spec rows.** Add rows to `plan_frontmatter_schema` for `decisions`,
   `decisions.<id>.ask`, `.choices`, `.default`, `.why`, `.memory`, and `.requested`,
   with examples. Update `schema_is_ordered_and_contains_exact_phase_guidance`.
3. **Callouts.** Parse callouts from the body (Section 2), skipping fenced code. Return
   `PlanDecisionCalloutWire { id, key: Option<String>, branch: yes|no|choice, start_line, end_line }`.
4. **Validated wire.** `ValidatedPlanWire` gains these fields, each with
   `skip_serializing_if` so existing parity fixtures stay byte-stable:
   - `decisions: Vec<PlanDecisionWire>`
   - `decision_callouts`
   - `decided_by`
   - `decided_via`

   `PlanDecisionWire` is
   `{ id, kind: toggle|choice, ask, why, choices: [{key, label}], default, memory: Option<{selectors}>, requested, answer }`.
   Do **not** bump `PLAN_WIRE_SCHEMA_VERSION`: the change is additive, as `proposed_by`
   and `links` were. Update `plan_validate_parity.rs` and the binding round-trip tests
   as needed.

5. **Frozen definition record.**
   `plan_decisions_payload(validated, host_facts) -> Vec<PlanDecisionDefinitionWire>`.
   Host facts are supplied by Python:
   - per memory decision: the resolved notes
     `[{selector, kind: note|web|strand, scope: project|home, path, type: core|reference|web|strand, exists, strands?}]`;
   - per decision: `requested_verified` and a provenance of `asked`, `not_asked`,
     `quote_not_found`, or `inherited`.

   The record carries an `effective_default`, which is `false` for a memory decision
   whose default is `true` but unverified. Also add
   `plan_decisions_digest(definitions)`, a canonical digest of the full definition (ask,
   labels, and why included), used for the edit freeze.

6. **Resolver.**
   `plan_decisions_resolve(definitions, submitted, caller: human|agent|auto) -> { values, rows[{id, value, source: default|submitted|clamped, changed}], errors[{id, code, message, allowed, default}] }`.
   - Omitted ids take `effective_default`.
   - Booleans must be JSON booleans. Choice keys canonicalize case-insensitively, with
     no prefix matching.
   - Unknown ids and invalid values are errors that list the allowed values.
   - A memory value of `true` from an `agent` caller is refused
     (`memory_decision_requires_human`) unless the effective default is already `true`.
   - `auto` takes defaults only.
7. **Quote matcher.**
   `plan_decision_quote_match(quote, texts[{source, ref, text}]) -> { verified, matched_source, closest: Option<{source, ref, sentence}> }`
   with the normalization in Section 4. Enforce the 3-word minimum here too.
8. **Decision Sheet.**
   `plan_decision_sheet(definitions, values, review_revision) -> PlanDecisionSheetWire { count, memory_count, changed_count, review_revision, rows[{id, kind, ask, why, choices, default, value, changed, memory?{selectors, resolved, provenance, quote}}] }`.
   `plan_decision_summary(sheet, verdict, form: short|full)` renders the sentence in
   Section 1.6.
9. **Implementer block.**
   `plan_decisions_prompt_block(sheet, decided_by, decided_via, audience: tale_coder|epic_phase|epic_land, inherited: Option<sheet + epic title>)`
   renders the host-written block. Lines use enum keys, booleans, and quoted `ask` text,
   never free-form interpolation:

   ```
   Reviewer decisions for this plan (final · reviewer via ACE):
   - grouping = mode (planner default: pane). Implement the "grouping = mode" branch; ignore "grouping = pane".
   - tui_note = yes 🧠. Memory edits are authorized for tui.md only.
   No other memory note may be edited. Implement only the branches selected above.
   ```

   - The `auto` variant says no human reviewed the plan.
   - Unrequested memory decisions that were left off get the declined-change instruction
     from Section 4: `/sase_new_task` for coders, `PROPOSED FOLLOW-UP:` for phase
     workers.
   - Inherited lines read `Inherited from epic "<title>": …`.

10. **Bindings.** Add pyo3 functions under `crates/sase_core_py/src/plans/` (a new
    `decisions.rs` submodule), registered in `register_plans`:
    - `plan_decisions_payload`
    - `plan_decisions_digest`
    - `plan_decisions_resolve`
    - `plan_decision_quote_match`
    - `plan_decision_sheet`
    - `plan_decision_summary`
    - `plan_decisions_prompt_block`

    Add round-trip tests. Import new items as `sase_core::plan::…`; do not add prelude
    aliases.

11. **Tests.** Cover every diagnostic, both kinds, YAML 1.1 words, Archived and Launch
    behavior, callouts, resolver clamps and the agent refusal, quote normalization edge
    cases, summary forms, and block audiences. Run `sase tool run check` in sase-core.

### 6.2 `provenance` — human-authorship provenance

1. **`prompt_origin`.** Write `prompt_origin` (`typed | generated | unknown`) and
   `prompt_source_surface` into every agent's `agent_meta.json` at launch. Reuse the
   classification that prompt history already applies
   (`history/prompt_store_mutations.py`, `main/query_handler/_launch.py`) instead of
   inventing a second one:
   - `typed`: the ACE prompt bar, `sase run` from a human shell, mobile.
   - `generated`: LaunchApproval dispatch, `sase bead work`, chop and job launches, and
     every session successor.
   - Anything else is `unknown`. Telegram launches stay `unknown` until the `telegram`
     phase passes `origin="typed"`.
2. **`caller`.** Record `caller: human | agent` on every gate `response.json` written by
   `notification_gates/executor.py`. Classify from the submitting process, failing
   closed to `agent`, like `bead/attachments/provenance.py` `current_actor()`. Do not
   change the Rust receipt wire.
3. **Gatherer.** Add `sase/sdd/plan_human_text.py` with
   `human_authored_texts(artifacts_dir) -> tuple[HumanText]` (`source`, `ref`, `text`).
   It walks the planner's own meta and the plan-gate chain
   (`plan_gate_turn_prev_ artifacts_dir`, root lookup through the session), plus each
   question round's bundle, applying the Section 4 rules. Missing files, missing fields,
   or legacy runs yield nothing.
4. **Tests.** Use fixture artifact trees covering typed roots, generated roots, feedback
   replans with human and auto responses, Q&A with human free text versus selected
   labels, and legacy metadata.

### 6.3 `gate` — compile, resolve, freeze, stamp

1. **Pin and wire.** Move `sase-core-revision.txt` past the `core` commit.
   - In `sdd/plan_validate.py`, `_ValidatedPlan` gains `decisions`, `decision_callouts`,
     `decided_by`, and `decided_via`, and the `archived` mode is accepted.
   - Update the tests that pin field order, and the required-binding list in
     `tools/validate_sase_core_rs` and its test.
   - Switch `sdd/committed_plan_validation.py` to Archived mode.
2. **Beta flag.** Run `sase flag new plan_decisions` (kind `beta`) with the three
   authored sentences, and paste the registry entry it prints.
   - **Off:** `sase plan validate` and `sase plan propose` report an error
     `decisions-disabled` for any plan with `decisions:`, and the schema table and
     `--explain` hide the decision rows.
   - **On:** everything below.
   - Test both states. Bryan can opt in early with `sase flag enable plan_decisions`.
3. **Adapter.** Add `sase/sdd/plan_decisions.py` as the single Python entry point. It
   wraps the core bindings and adds two pieces of host logic:
   - **Memory scope resolution** through `memory/selector.py`. A web selector freezes
     its current strand list, and overlapping grants raise `decision-memory-overlap`.
   - **Quote verification** through the `provenance` gatherer and the core matcher.

   Telegram and every other surface import only this module.

4. **Host checks at validate and propose.** Inside an agent context,
   `sase plan validate` and `sase plan propose` run the host checks:
   - selector resolution and overlap;
   - `decision-requested-unverified`, which prints the closest human sentence.

   Outside an agent, validate says that quote verification runs at propose. `propose`
   echoes `Plan Decisions: N (🧠 M)` before its handoff, plus the `%auto` note when the
   plan will be auto-approved (Section 5.10).

5. **Gate build (`plan_gate.py`).**
   - `payload.decisions` holds the frozen definitions, built from fresh host facts.
     Verification fails closed here (Section 4).
   - Raw `input_schema` properties named `decision_<id>` go on tale `approve`, `commit`,
     and `feedback`, and on epic `approve` and `feedback`. A toggle compiles to
     `{"type":"boolean"}` and a choice to `{"enum":[keys]}`. They are never required.
   - Approve and commit result schemas gain a required, fully resolved `decisions`
     object. `_plan_gate_command.py` echoes the resolved values into it.
   - The gate notes gain a second line, `N decisions · 🧠 M`. That line feeds the ACE
     inbox, toasts, Telegram, and Android.
   - In ACE's plan modal, `decision_` joins the host-collected set as a prefix rule in
     `plan_approval_gate_data.py`. Without it, Enter would open the YAML panel while the
     flag is on and before `tui` lands.
6. **Kind validation.** Add `_validate_plan_decisions` in `kind_validation/plan.py`:
   - `payload.decisions` round-trips through core;
   - each option's `decision_*` property set and types equal the set compiled from the
     payload;
   - the result schemas match.

   The query, options, groups, commands, and operations stay sealed exactly as they are.

7. **Normalization hook.** Add
   `GateAdapter.normalize_option_inputs(envelope, selected_option_ids, option_inputs, *, source, caller)`
   and call it once at the top of `execute_gate_selection`, before
   `accept_gate_decision`. Both the receipt and the execution input paths must consume
   its output. The plan implementation:
   - gathers `decision_*` values across the selected options (`decision_conflict` on any
     disagreement);
   - calls the core resolver with `caller` (`auto` for `auto_resolution`);
   - writes the identical resolved vector into every selected option that declares
     decision properties.

   Errors happen before any side effect (contract 3, 4, 9).

8. **Revision binding.** `execute_gate_selection` gains an optional
   `expected_review_revision`. `sase gate answer` accepts it as a `review_revision`
   payload field. A mismatch fails with `stale_review` before any side effect. Absent
   means unchecked, which keeps mobile and older clients compatible (contract 5).
9. **Edit freeze.** The plan adapter's `validate_edited_resource` compares
   `plan_decisions_digest` of the edited plan with the digest of `payload.decisions`. It
   also refuses any `answer`, `decided_by`, or `decided_via` in the edited plan. The
   refusal text is: "Decisions are fixed for this review. Change answers in the
   Decisions panel, or send feedback to change the questions." (contract 6)
10. **Feedback carry.** A feedback submission's provisional decision values reach
    `main/feedback_prompt.py` (and the in-process equivalent in
    `axe/run_agent_exec_plan.py`). They render as
    `### Reviewer's provisional decisions`, listing each changed value as the new
    default. A memory decision the reviewer switched on is listed but explicitly called
    **not** authorization; the replanner may default it on only by quoting human
    feedback text.
11. **Stamping.**
    - **Tales:** in `prepare_plan_terminal_response`, after the durable sync and before
      the archive commit, write the resolved answers into the **durable** plan through
      `sdd/frontmatter.py` (key order is preserved). Each decision gets an `answer:`,
      and the plan gets `decided_by` and `decided_via` (mapped from source and caller).
      The bundle copy stays pristine.
    - **Epics:** stamp before `prepare_epic_launch`, so `sase bead work` archives the
      stamped file.
    - Commit-only and approve-only tales stamp too.
    - Stamping is idempotent. Re-stamping different values is an error (contract 7).
    - Retry paths re-stamp from `response.json` when the durable plan lacks answers.
12. **Direct routes.** `sase plan approve <file>` with no live gate and a hand-run
    `sase bead work` resolve through the same resolver, take defaults, and stamp. Expose
    `resolve_plan_decisions_for_direct_approval(plan, overrides, caller)` for the `cli`
    phase's `-D`.
13. **Docs and tests.**
    - Update `docs/notifications.md`: the plan gate wire, `payload.decisions`,
      `decision_*`, normalization, `stale_review`, and the freeze.
    - Tests cover: the flag in both states; kind-validation drift; identical
      `input_identity` for omitted versus explicit defaults; `decision_conflict`; the
      agent memory refusal through `sase gate answer`; the freeze; stamping on all
      routes; Archived committed validation; `stale_review`; and `%auto` with verified
      and unverified quotes.

### 6.4 `handoff` — deliver accepted decisions

1. **Coder block.** In `axe/run_agent_exec_plan_accept.py`, append the core-rendered
   block after "Implement it now." and before any "Additional instructions". Read the
   block from the stamped durable plan. That one function serves both the gate-turn and
   the in-process `%auto` paths; test both.
2. **Epic context.** Add `epic_decision_context(artifacts_dir)`. It resolves a phase or
   land agent's epic (`epic_plan_ref`, `phase_bead_id`, or through its agent session for
   a phase planner's coder successor) to the epic's stamped sheet. The coder block and
   the guard use it for inheritance (Section 4). It fails closed when the context cannot
   be resolved.
3. **`sase bead read`.** Show a **DECISIONS** section for epic and phase beads, in text
   and JSON (`bead/cli_detail_resolution.py`, `cli_detail_render.py`,
   `cli_detail_json.py`). Read it from the design plan in Launch mode and render it with
   the `epic_phase` or `epic_land` audience.
4. **Macros.** In `src/sase/default_config.yml`:
   - `bd/work_phase_bead` gains, after its `sase bead read` step: "Honor the epic's
     DECISIONS shown by `sase bead read`; they are final, and only memory notes they
     authorize may be edited."
   - `bd/land_epic` step 1 gains: "confirm the work honored the epic's DECISIONS."

   Update the pinned phrase tests in `tests/test_bead_macro_tags.py`.

5. **`%auto` receipt.** When a plan gate with at least one decision auto-resolves,
   create one informational notification. It is not a gate, and its content follows the
   mock in Section 1.5.
   - It appears in the ACE inbox without a toast or an unread bump.
   - It carries a stable tag (`plan_decisions_receipt`) and a dedup key per request id.
   - Choose `silent`/`muted` and the action so the Telegram phase can forward it
     quietly. Document the choice; `notifications/models.py` and the outbound filter in
     sase-telegram are the constraints.
   - Plans without decisions stay silent, as they are today.
6. **Display builders.** Add `sdd/_plan_display_decisions.py` with shared Rich builders,
   used by `sase plan show`, the ACE PLAN lane, and the agent prompt panel:
   - **Pending:** each ask, its choices, `★`, the why line, and memory chips.
   - **Accepted:** the header `decided by reviewer · TUI · 14:02`, the answer marked
     `◉`, `●` where it changed, and the other choices dimmed.
7. **Tests:** coder prompt snapshots (reviewer, auto, inherited, declined memory),
   `bead read` text and JSON, macro phrases, the receipt created only with decisions,
   and the builders.

### 6.5 `cli` — sase plan and sase gate

Read `cli_rules` first: sorted options, short aliases, excellent coloured help, and `-h`
examples for a default, an override, and a retry.

1. **`-D`.** Add `sase plan approve -D/--decide ID=VALUE`, repeatable. Giving the same
   id twice is an error.
   - Toggles use the shared bool spellings from `macro/models.py`. Choices take the
     exact key, case-insensitive.
   - Errors happen before any side effect. They print a did-you-mean and the allowed
     values with `★`, then `nothing was approved`.
   - It works for live gates (submitting `decision_*` inputs together with the displayed
     `review_revision`), for direct-file approval, and for epics (`-k epic`).
2. **Live completions.** Add a completion provider in `completion/candidates/` so
   `-D <TAB>` completes `id=` from the selected proposal, then its values.
3. **Decision card.** Print it on every approval with decisions, as in Section 1.4. With
   `-n`, print it and stop. The source column reads `-D` or `default`, and `was ★ …`
   appears on changes.
4. **Retry and boundary messages.**
   - On an already-approved plan:
     `Retrying implementation with the accepted decisions: grouping=mode; tui_note=no.`
   - `-D` on an already-approved plan is refused:
     `This plan is already approved. Retry uses its accepted answers. Start a new review to change tui_note.`
   - An agent passing `-D <memory>=yes` is refused:
     `memory decisions can only be switched on by a human; this shell runs inside agent <name>.`
5. **Other commands.**
   - `sase plan show`: a DECISIONS section before PROPERTIES, using the `handoff`
     builders. `-f compact` adds `◉3 🧠1`. `-f json` adds the sheet plus `decided_by`
     and `decided_via`.
   - `sase plan list`: a narrow `◉` count column.
   - `sase plan validate`: renders the Decision Sheet after diagnostics, plus the
     `%auto` note.
   - `sase gate show`: plan gates render a decisions table instead of
     `raw schema: decision_…`.
   - `sase gate answer -O` keeps working as the wire-level escape hatch, through the
     same resolver.
6. **Tests:** parser, help text, every error path, the card, completions, and JSON.

### 6.6 `tui` — ACE

Read the `tui`, `tui_perf`, and `tui_screenshot` notes first.

1. **Layout** (`plan_approval_modal.py` and siblings, `styles.tcss`). The rail has three
   parts:
   - **Actions**, one line.
   - **Decisions**, with the header `Decisions  N · 🧠 M` above accordion rows. It
     scrolls if needed, and is omitted entirely when the plan has none.
   - **Verdict**, renamed from "Decision" and docked to the bottom. Line 1 holds the AND
     toggles with short labels (`☑️ 🚀 Launch coder  ☑️ 💾 Commit plan`; the full label
     becomes a tooltip). Line 2 holds compact buttons
     (`1 ✅ Tale  2 ❌ Reject  3 💬 Feedback`). Line 3 holds the full summary sentence.
     Drop the Cancel button; `q` and Esc already cancel.

   The compact Verdict becomes the standard plan-review layout with or without
   decisions, so layouts never jump. The rail widens from 42 to 50 cells only when
   decisions exist, and the narrow breakpoint stays at 100 columns.

2. **Rows.** Follow Section 1.2 and the research row anatomy: collapsed rows take two
   lines; the focused row expands with the full ask, choices with consequences, `★`, the
   why line, memory chips, and the quote.
   - Unverified memory rows read
     `⚠ "<quote>" — not in your messages · off until you turn it on` in the warning
     colour.
   - When a human turns on an unrequested memory edit, the row shows `yes ●` and the
     summary names the note. There is no confirmation dialog.
3. **Keys.** Add these to `ace.keymaps.gate` in `src/sase/default_config.yml`,
   `GateModalKeymaps`, and `_GATE_BINDING_META`. Bind them with priority and reserve
   them so gate-action fallback keys skip them.
   - `space`: next choice, wrapping, or flip a toggle; on AND members it toggles as
     today.
   - `l`/→: next choice, or set yes.
   - `h`/←: previous choice, or set no.
   - `r`: reset to `★`. `R`: reset all.
   - `j`/`k`: move through Actions, then Decisions, then Verdict.
   - Enter **always** approves as shown, and digits submit branch N with the current
     values.

   Footer hints for these keys appear only when the plan has decisions, and the Enter
   badge reads `Enter=Tale · N changes`. Update `docs/ace.md` (Plan Approval
   Keybindings) and `docs/configuration.md`.

4. **Document pane.**
   - Fold `decisions:` in the frontmatter into one dim line,
     `decisions: N · answered in the Decisions panel`. `e` and `Y` still act on the raw
     file.
   - Focusing a decision scrolls to its first callout, or failing that its first id
     mention, using the core callout spans and accounting for the folded lines.
   - Tint the chosen branch green with a bold header, and dim the unchosen branch or a
     toggle answered no. Never hide a branch.
   - Reuse the cached token stream (`util/frontmatter_syntax.py`). Measure keypress cost
     against `tui_perf`.
5. **Every submit path merges the sheet's values** (contract 1, 5). Check each one:
   - Enter, `ctrl+s`, digits, and mouse;
   - `i`, and panel completion;
   - the `c` → `ApproveOptionsModal` path, and the `PendingApproveState` round trip;
   - the programmatic `action_*` methods;
   - `_plan_gate_submission_payload`.

   Each must send `decision_*` values with the displayed `review_revision`. `c` and `i`
   never show decisions, and the `✎ n inputs` badges ignore them.

6. **States.**
   - Esc keeps provisional values per request id for the life of the ACE process.
   - An in-gate edit that touches `decisions:` shows the existing red "Draft not
     accepted" banner with the freeze message and keeps the draft.
   - Feedback (`3`) shows a read-only first line, `Carries: grouping → mode`.
   - Settled elsewhere: replace the controls with `Approved via Telegram · <summary>`
     instead of letting a stale modal submit.
   - `stale_review` reloads the bundle and keeps the values.
7. **Around the modal.**
   - The plan toast gets a second line, `N decisions · 🧠 M`.
   - Inbox rows append `· ◉N 🧠`.
   - The inbox gate card gets a Decisions block: pending rows read `◉ grouping  pane ★`
     and answered rows read `grouping: mode ●`, with no duplicate per-option field list.
   - The PLAN lane and the agent prompt panel render the `handoff` builders.
   - The `%auto` receipt row renders cleanly.
8. **Goldens.** Add these with fixtures that build real gate data with decisions:
   - `plan_gate_tale_decisions_120x40`
   - `plan_gate_tale_decisions_memory_120x40`
   - `plan_gate_tale_decisions_unverified_120x40`
   - `plan_gate_tale_decisions_stacked_90x40`
   - `plan_gate_epic_decisions_120x40`
   - the inbox gate card, pending and answered
   - the plan toast with decisions

   Update the existing `plan_gate_*` goldens once for the compact Verdict, as one
   reviewed group. Run `just fix-tui-screenshots` through `/sase_monitor`, and inspect
   every creation and update.

### 6.7 `telegram` — sase-telegram

Work in the linked checkout (`sase repo open sase-telegram`). Consume decisions only
through `sase.sdd.plan_decisions`. Feature-detect it, and fall back to today's behavior
when the installed sase lacks it.

1. **Message.** In `formatting.py`, strip `decisions`, `decided_by`, and `decided_via`
   from the properties card. Render the static question sheet (Section 1.3) right after
   the header and notes, before Properties. If the sheet exceeds about 1,800 characters,
   shorten it in this order:
   1. Drop the labels of non-default choices.
   2. Drop the remaining labels.
   3. Wrap the choice lines in an expandable blockquote.

   Never cut a decision in half; each `ask` and its `★` line always survive.

2. **Keyboard.**
   - Decision rows go above the AND toggles. Toggles pair up when their labels fit, and
     a choice opens a radio sub-keyboard in place via `edit_message_reply_markup`.
   - The primary label reads `✅ Tale · defaults` or `✅ Tale · N changes` (presentation
     only; the sealed label stays "Tale"), and the same for Epic. Use `style: success`
     where the client library supports it.
   - `↺ Reset` appears after a change.
   - Toasts read like receipts, for example
     `🧠 tui_note yes — authorizes editing tui.md` or `grouping → mode`, and the opening
     toast of a choice shows its `ask`.
3. **Callbacks set values; they never toggle.**
   - Tokens are short and server-resolved, each carrying the card's revision, for
     example `d2=0r4`, `d0=k1r4`, `d0>r4`, `d<r4`, and `dzr4`. Keep them within 64
     bytes.
   - Values and the revision live in `telegram_gate_progress.json`, so they survive a
     restart.
   - A Telegram draft never changes what ACE will submit.
4. **Submit and stale.** Submit the current values for each selected option, plus
   `review_revision`. On `stale_review`, or on a tap when the revision has moved, answer
   "This plan changed since this card was shown." and offer **↻ Refresh review**, which
   re-renders the card for the current revision. It never approves silently.
5. **Settle.** Whether the gate settles here, in ACE, or in the CLI, edit the review
   message text once into the answered receipt (Section 1.3) and remove the keyboard.
   - The completion reply carries the full summary sentence.
   - A launch failure after acceptance reads "Approved with these choices · coder could
     not start · retry".
6. **Quiet `%auto` receipts.** Forward the `handoff` receipt notification with
   `disable_notification`. Add the parameter to `telegram_client.send_message`.
7. **Feedback.** `💬 Feedback` carries the current values as provisional. The feedback
   note prompt asks for a **reply** to the review message, and fixes the bug where an
   unreplied text can fall through and launch a new agent while several prompts are
   waiting.
8. **PDF.** Prepend a rendered Decisions table, and render callouts as labelled
   blockquotes.
9. **Provenance.** Agent launches from Telegram pass `origin="typed"`
   (`inbound_handlers/agent_launch.py`).
10. **Tests and docs.** Update the formatting and keyboard pins (including
    `tests/test_custom_gates.py`), callback idempotency, staleness, settle edits, and
    the budget truncation. Update `docs/outbound.md` (which is stale: it lists "Tale, ✅
    Approve, Epic") and `docs/inbound.md`. Run `sase tool run check` there.

### 6.8 `guard` — advisory finalizer memory guard

1. Add a host-side commit-finalizer check under `src/sase/finalizers/` for agents
   launched from an approved plan: tale coders (plan from `SASE_PLAN`, `sdd_plan_path`,
   or `plan_archive_ref`), phase agents, phase sub-plan coders, and land agents (via
   `epic_decision_context`).
2. **Coverage** is the union of the plan's own memory decisions answered yes and the
   inherited epic grants, using the frozen resolved paths. Generated `AGENTS.md`,
   provider shims, and the memory README count as covered only when a covered note
   changed in the same declaration.
3. **Checked paths** are `sase/memory/**` in every repo the declaration commits. Agents
   not launched from a plan are never checked: the direct-prompt route stays valid.
4. **Output.** Emit
   `FinalizerDiagnosticWire(severity="warning", code="memory_change_uncovered")` naming
   each uncovered path and the plan. It never blocks, it surfaces through
   `finalizer_status`, and it needs no flag.
5. **Tests:** covered, uncovered, inherited, generated-only, and non-plan agents.

### 6.9 `policy` — skills, docs, flag removal, follow-ups

1. **`src/sase/macros/skills/sase_plan.md`.** Add a short "Plan Decisions" step:
   - **The rule.** Ask now with `/sase_questions` only when the answer changes the tier,
     size, phase graph, or architecture. Embed a decision when you can write one
     complete plan that covers every answer, the difference is local, and you would
     defend your default. Never embed a decision you should make yourself.
   - **Prefer decisions.** Prefer decisions over questions whenever that does not
     degrade the plan.
   - **Grammar.** A compact example.
   - **Copy rules.** Use readable ids; `ask` is a question where yes means do the work;
     choice labels state consequences; add a one-line `why`; order by importance with
     memory last; use callouts when branches differ by more than a sentence.
   - **`%auto`.** Under `%auto`, embed only memory decisions.
   - **Epic phases.** Inside an epic phase, do not re-ask the epic's DECISIONS.
2. **`sase_memory_write.md`.** Rewrite the authorization and routing sections:
   - **Plan route.** An approved plan you are implementing authorizes a note only
     through an accepted memory decision covering it, its own or inherited from its
     epic. Plan approval alone authorizes nothing.
   - **Authoring.** Every memory change a plan makes is a memory decision, defaulting to
     `true` only with a `requested:` quote of the user's complete request. Replace the
     "confirm with `/sase_questions` before propose" route, and stop writing "do not
     edit memory" prose.
   - **Bead route.** It still applies to beads implemented directly. A bead worked
     plan-first follows the plan route.
   - **Declined changes.** Section 4's rule.
   - **The guard.** Changes outside accepted decisions are flagged by the finalizer.
3. **`sase_questions.md`.** Add one paragraph: when the question shapes the plan but a
   complete plan can cover every answer, embed a Plan Decision instead.
4. **Skill tests.** Update the phrase tests (`tests/main/test_init_skills_sources.py`,
   `test_init_skills_source_content.py`).
5. **Planner docs.** In `src/sase/main/plan_explain.py`, document the grammar for tales
   and epics in `--explain` prose. In `docs/sdd.md`, add a "Plan Decisions" authoring
   section: the grammar, callouts, memory consent, the lifecycle, the archive fields,
   and the Archived mode.
6. **Flag removal.** Delete the `plan_decisions` Off branch and make the On branch
   unconditional. Remove the registry entry and close its flag bead in this change.
7. **Skill deployment.** Do not deploy skills from this phase. After this commit lands
   on the canonical branch, the land agent previews with `sase skill init --diff` and
   deploys with `sase skill init --force` (per `generated_skills`).
8. **Follow-ups.** Record each as a `PROPOSED FOLLOW-UP:` note on this phase's bead:
   1. **Dogfood memory plan:** a `glossary:plan-decision` strand ("Plan Decision", alias
      "memory decision"), a `decisions:` record for "decisions, not gate options", and a
      `generated_skills` "Plan Mode and Questions" line preferring Plan Decisions. Work
      it plan-first, so its tale carries memory decisions that Bryan answers in the new
      panel.
   2. **Enforcing guard mode** after a soak period: refuse the whole publication and
      list the uncovered paths.
   3. **The Android client** renders the Decision Sheet from the gateway, and the mobile
      wire carries it.
   4. **Memory task beads under `%auto`.** They can never obtain a human consent through
      `sase bead work`. Decide whether to drop `%auto` for `memory` beads, or to count
      human-created bead descriptions as human text.
   5. **Generic gate-input UX:** a `TypedInputForm` bool toggle, plus a Telegram
      step-flow yes/no keyboard and "Keep default: X" for custom gates.
   6. **A CLI review token** (`-V`), if a script ever needs one.

## 7. Verification

- **Every phase:** `sase tool run check` in each repo it changed (sase-core,
  sase-telegram, sase). The `tui` phase also runs visual goldens through
  `/sase_monitor`. Do not run `just check-full` unless explicitly instructed.
- **End-to-end smoke (land agent),** with the flag already removed. Propose a throwaway
  tale with one choice and one requested memory decision, then confirm:
  1. `sase plan validate` renders the sheet.
  2. `sase plan approve <plan> -D <choice>=<other> --dry-run` prints the card.
  3. A `%auto` run stamps `answer:` and `decided_by: auto` and posts one receipt.
  4. The coder prompt contains the block.
  5. `sase plan show` shows the accepted decisions.

  Then delete the throwaway plan.

- **Usability check for Bryan** (report it in the land note; do not block on it). On
  each surface, review a tale with one memory decision and an epic with one choice plus
  one requested memory decision, without explanation. Before submitting, say what Enter
  or the green button will do. Watch whether changed values stand out, whether "no" on a
  memory row is clear, and whether one answer can be changed without approving.
