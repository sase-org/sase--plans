---
tier: tale
title: "%auto never answers questions"
goal:
  Under every %auto spelling, role, and inherited or pre-existing record, question gates
  wait for a human, and the paused %auto roadmap keeps it that way.
size: medium
decisions:
  macros_md_auto_row:
    ask: Correct the macros.md %auto row to say questions always wait for a human?
    default: true
    memory:
      - macros.md
    corrects:
      Auto-resolves plan, epic, and question gates; bare approves and archives tales,
      launches epics, and answers questions with the first option
    answer: true
proposed_by: bbugyi200.athena.13l
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 06:48:00
status: wip
---

# Plan: `%auto` never answers questions

## Why

**The user's request:** SASE questions (`/sase_questions` → `sase questions` → a
`question` gate) must not be answered by `%auto`, at least not by default. The fix must
cover every case, including the paused `%auto` autonomy roadmap.

**Today every non-manual built-in profile answers each question with its first option.**

- Core `crates/sase_core/src/autonomy/profiles.rs::profile_policy` gives
  `question: first` to three profiles:
  - `standard`: bare `%auto`, `%a`, `%auto+`, and `%auto:true`;
  - `tale`: `%auto:tale` and `%auto:plan`;
  - `epic`: `%auto:epic`.
- Epic phase and land workers run the `autonomy.roles` profile (default `standard`), so
  they answer questions too.
- Host-composed successors inherit the live record, so the behavior carries through
  coders, replanners, pipe/handoff successors, monitor follow-ups, and gate follow-ups.

**Changing `profile_policy` alone is not enough.** Every agent persists its policy in
`agent_meta.autonomy.policy`:

- `evaluate()` reads `record.policy`, not the profile table.
- `autonomy_inherit` copies the predecessor's policy unchanged.

So every agent launched before this lands would keep auto-answering, and so would every
successor descended from it, possibly for days in a long epic chain. Readers must
therefore take a built-in profile's policy from core code, not from the stored snapshot.
That matches E1's own statement that the four profiles are "fixed core code, not
configuration".

**The paused roadmap assumes question auto-answers.** The E1 epic `sase-1ip` is closed.
E2–E5 are paused in the roadmap, and it assumes question auto-answers in several places:

- the contract rows;
- E1's `question_default` decision;
- E3's `question: recommended | decide` vocabulary and its built-in profiles;
- the risks table.

It must be amended so a later epic does not reintroduce question auto-answers.

**Read for context:**

- `sase artifact read research:202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md "<why>"`.
  This is the paused roadmap you will amend; it already carries two 2026-10-09
  amendments.
- `sase artifact read plan:202610/auto_e1_autonomy_record.md "<why>"`: the E1 contract
  and record design.
- `sase artifact read plan:202610/auto_e1_no_agent_autonomy_text.md "<why>"`: the
  precedent tale for a user reversal that amended the roadmap. Copy its amendment style.

## Target behavior

**Bold** marks the deliberate change. Plan and epic outcomes do not change anywhere.

| Prompt or state                                                                                                                                                            | Tale plan         | Epic plan        | Question     |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- | ---------------- | ------------ |
| No `%auto`, `%auto:manual`, `%auto:off`                                                                                                                                    | ask               | ask              | ask          |
| `%auto`, `%a`, `%auto+`, `%auto:true`                                                                                                                                      | approve + archive | approve + launch | **ask**      |
| `%auto:tale`, `%auto:plan`                                                                                                                                                 | approve + archive | ask              | **ask**      |
| `%auto:epic`                                                                                                                                                               | ask               | approve + launch | **ask**      |
| Epic phase or land worker (any role)                                                                                                                                       | per role          | per role         | **ask**      |
| Any host-composed successor                                                                                                                                                | inherited         | inherited        | **ask**      |
| `A` off, then on (restore last)                                                                                                                                            | unchanged         | unchanged        | **ask**      |
| Agent whose record was persisted before this change (`question: first`)                                                                                                    | unchanged         | unchanged        | **ask**      |
| A record naming a non-built-in profile with an explicit `question: first`, unreachable from any launch path today and reserved for E3 config profiles (test fixtures only) | its policy        | its policy       | first option |

**What does not ship:** no new `%auto` spelling, no config key, and no other opt-in. A
question asks because the agent needs a human, and the first option would silently stand
in for one. Question automation becomes opt-in only through E3's config-defined
profiles. Step 6 records that in the roadmap.

## 1. sase-core

Open the core checkout with
`sase repo open sase-core -r "Make built-in autonomy profiles never auto-answer questions"`
and read its `AGENTS.md`:

- never run bare `cargo`;
- use `just test -p <crate> <filter>` while iterating;
- run `sase tool run check` before you finish.

**1a. Profiles** (`crates/sase_core/src/autonomy/profiles.rs`):

- In `profile_policy`, set `question` to `AUTONOMY_VALUE_ASK` for `standard`, `tale`,
  and `epic`. `manual` already asks.
- Update the module and function docs. The wire docs in `wires.rs` should no longer
  imply that any built-in profile answers questions.

**1b. The effective record.** Add one core function next to `profile_policy`, exported
from `mod.rs`:
`pub fn autonomy_effective_record(record: &AutonomyRecordWire) -> AutonomyRecordWire`.

- **Rule:** when `record.profile` is one of `builtin_profiles()` and `record.overrides`
  is empty, return a clone with `policy = profile_policy(profile)` and
  `digest = policy_digest(&policy)`.
- Every other field stays unchanged: `revision`, `source`, `selection`, `last`,
  `inherited_from`, `updated_at`, and `updated_by`. A code-level refresh is not a
  mutation.
- Any other record (an unknown profile name, or non-empty overrides) is returned
  unchanged. That is the forward-compatible lane E3's config profiles will use.
- Document the rule in the doc comment: "built-in profile policies are core code; a
  stored policy for a built-in profile is a cache refreshed on every read".

**1c. Apply it at every core entry that consumes a stored record:**

- **`evaluate`**, as its first step, so `fleet_auto_approved` gets it for free. The
  decision's `digest` is then the effective digest.
- **`autonomy_summary`**, which covers its class, sentence, short form, and cells.
- **`autonomy_inherit`**: normalize the predecessor before building `base`, so
  successors persist the refreshed policy and digest.
- **`mutate_autonomy`**: normalize the incoming record first, so `widened_kind`,
  `same_state`, and the returned record all use the effective policy.
- **The agent scan's `autonomy_record_from_value`** (`agent_scan/scanner.rs`), so the
  scan wire and fleet facts carry the effective record. That includes
  `fleet_owner_facts` and anything the mobile gateway projects.
- Run `rg -n "AutonomyRecordWire" crates` and route any other reader of a stored record
  through the function too.

**1d. Binding.** Expose `autonomy_effective_record` in
`crates/sase_core_py/src/autonomy/mod.rs`:

- model it on `py_autonomy_summary`: dict in, record dict out;
- register it in `register_autonomy`;
- add a round-trip test in `crates/sase_core_py/src/autonomy/tests.rs`.

**1e. Core tests** (`crates/sase_core/src/autonomy/tests.rs`, the core_py tests, and the
scanner tests):

- **Update expectations:**
  - **evaluate table:** the `standard`, `tale`, and `epic` question rows give `ask` with
    rule `gates.question`;
  - **catalog and summary one-liners:** `tales ✓ · epics ✓ · questions ✋`, and likewise
    for `tale` and `epic`;
  - **classes:** `standard` is now `attended`, and no built-in profile is `autopilot`;
  - **sentences:** for example
    `Standard: tales approve + archive; epics launch; questions wait.`;
  - **question cell:** its effect is `Waits for you`.
- **New tests:**
  - **A stale built-in record:**
    - Setup: a `standard` record with `question: first` and the old digest.
    - It evaluates the question kind to `ask`.
    - Its decision digest equals `policy_digest(&profile_policy("standard"))`.
    - Its plan and epic outcomes are unchanged.
    - Its revision is unchanged.
  - **Inheritance:** `autonomy_inherit` from a stale predecessor persists the refreshed
    policy.
  - **Mutation:** `mutate_autonomy` on a stale record (`A` off, then `restore`) returns
    the refreshed policy.
  - **The non-built-in lane:**
    - Setup: a record whose profile is not a built-in name (for example
      `question_fixture`), with `question: first`.
    - It still evaluates to `auto` with option IDs `["submit"]`.
    - `autonomy_effective_record` leaves it unchanged.
    - A built-in record with non-empty `overrides` is also left unchanged.
  - **The scanner:** a stale stored record scans with the refreshed policy.
- **Keep, because a non-built-in profile can still reach them** (E3 will use this path):
  - `AUTONOMY_VALUE_FIRST`;
  - `required_option_ids("first")`;
  - `AUTONOMY_EFFECT_QUESTION_AUTO`;
  - the "answered with the first option" decision sentence.

**Commit.**

- Use a Conventional Commit subject such as
  `feat(autonomy): built-in profiles make every question wait for a human`.
- The body explains the read-time refresh of stored built-in policies.
- The wire is unchanged and a binding is added, so this is not a breaking change.
- Never edit versions or `CHANGELOG.md`.
- The host writes sase's `sase-core-revision.txt` when the turn commits both repos.

**Verify:**

- `just test -p sase_core autonomy`;
- `just test -p sase_core agent_scan`;
- `just test -p sase_core_py autonomy`;
- then `sase tool run check` in the core checkout. It takes about 5 minutes; give it at
  least a 10-minute timeout.

## 2. sase: Python

**Rebuild the binding first.** Rebuild the local `sase_core_rs` from the linked checkout
(for example `just rust-dev-install`, a known-long command) so the new binding imports.
Never reinstall globally.

**2a. `read_record`** (`src/sase/autonomy/record.py`):

- Pass a stored `autonomy` record through the `autonomy_effective_record` binding. Every
  Python surface then sees the effective policy:
  - `sase autonomy explain`, `show`, and `list`;
  - `sase agent show`;
  - the gate snapshot attached by `user_question_gate_spec`;
  - `is_auto_approve_active`.
- Keep its never-raises contract. If the binding call fails, return the stored dict
  unchanged; core `evaluate` normalizes anyway.
- The legacy-translation branch already gets fresh profile policies. Update the
  docstring.

**2b. Leave the automatic-question machinery in place.** It is reachable only through a
non-built-in profile now, and E3 needs it. Do not delete any of these:

- `GATE_AUTO_CAPABILITIES["question"]` and the `_GATE_STANDARD_REQUESTS` tables, which
  declare what the question adapter _can_ execute;
- `GateAdapter.automatic_input`'s `first` branch;
- `automatic_question_response`;
- `_continue_after_auto_answered_question`.

Then confirm that no Python code decides question automation outside core:
`rg -n "first" src/sase/autonomy src/sase/notification_gates src/sase/user_question_actions.py src/sase/axe/run_agent_exec_questions.py`.

**2c. Config comment** (`src/sase/default_config.yml`, the `autonomy:` comment):

- Replace "questions take the first option" with wording that questions always wait for
  you under every profile.
- Re-wrap only the touched lines.

## 3. sase: tests

**Contract rows** (`tests/autonomy_contract/rows.py`):

- Set the question column to `ASK` for every row.
- Extend the module docstring with this deliberate amendment, in the style of the
  existing epic-worker comment.
- Add a `stale_record` row:
  - add a harness context that seeds live meta with a pre-change `standard` record
    (`question: first`, old digest);
  - expected outcomes: tale `APPROVE_ARCHIVE`, epic `APPROVE_LAUNCH`, question `ASK`.
- The parity tests (`explain` equals runtime) then cover the new predictions.

**Tests that exercise the automatic question machinery:**

- Today these use bare `auto=True` or a `standard` record to mean "auto-answer". Move
  them to one shared fixture in `tests/autonomy_contract/harness.py`, for example
  `question_first_fixture_record()`:
  - its profile is not a built-in name (for example `question_fixture`);
  - its policy is `{plan: ask, epic: ask, question: first}`.
- In each one, also assert that the same call under a built-in profile now parks.
- Known sites:
  - `tests/test_user_question_gates.py::test_auto_question_uses_first_options_without_publishing_pending_action`;
  - `tests/fakey/test_gate_capacity_custom_e2e.py`;
  - `tests/autonomy_contract/test_ui_default_independence.py::test_question_ignores_default_selected_off`;
  - `tests/autonomy_contract/test_gates_evaluate.py::test_non_manual_evaluations_log_one_row_including_questions`.
    A `standard` question still logs one row, now with outcome `ask`. Add a fixture-
    profile `auto` row next to it.
  - `tests/autonomy_contract/test_landing_gaps_gates.py`, around the `harness.FIRST`
    expectation;
  - `tests/autonomy_contract/test_autonomy_cli_parity.py`;
  - `tests/test_notification_gates.py`;
  - `tests/test_axe_run_agent_exec_questions_gate_turn.py`.
- Find any others by running the suite. Never weaken an assertion: each one either
  becomes `ask` under built-in profiles or moves to the fixture.

**New regression tests:**

- **Stale record:**
  - Live meta carrying a pre-change `standard` record goes through
    `user_question_gate_spec(..., auto=True)` and `create_gate`.
  - The gate parks, and its `policy` block has outcome `ask` and rule `gates.question`.
  - `read_record` returns the refreshed policy.
- **`sase autonomy list`:** shows questions waiting for every built-in profile.

## 4. sase: docs

Follow `docs_writing.md`: correct the reference pages that describe the old behavior.
Never edit `CHANGELOG.md` or `docs/blog/`.

- **`docs/reference/macros/directive-launch.md`:**
  - In the profile table, the Questions column says "Wait for answers" in every row.
  - Correct "answers every question gate with its first option" and "All three still
    auto-answer every question gate with its first option".
  - Add one sentence: no built-in profile answers questions, including agents launched
    before this change, because built-in profile policies refresh from core when read.
  - Correct the `sase autonomy` section's "both still choose the first option for each
    question".
- **`docs/reference/macros/directive-composition.md`:** correct the `%auto:epic`
  paragraph's question sentence.
- **`docs/reference/macros/directive-syntax.md`:** change the `%auto` comment to
  "Auto-resolve plan and epic gates; questions always wait".
- **`docs/reference/tui/agent-workflows.md`:** "Every enabled built-in profile also
  chooses the first option" becomes "Questions always wait for you, whichever profile is
  enabled".
- **`docs/reference/automation/notification-cli.md`:** append to "Only the plan,
  epic-plan, and question kinds can auto-resolve" that no built-in profile auto-resolves
  questions.
- **`docs/reference/config/agents-and-runners.md`:** regenerate from the config comment
  with `just fix`. Hand-edit it only if it turns out not to be generated.
- Run `rg -n -i "first option|auto-answer|questions take" docs` afterwards. Only
  `docs/blog/` may still match.

## 5. Memory: the `macros.md` `%auto` row

> [!decision] macros_md_auto_row
>
> Use `/sase_memory_write`. In `sase/memory/macros.md`, rewrite the `%auto` row's
> meaning so it reads, for example: "Auto-resolves plan and epic gates; bare approves
> and archives tales and launches epics; `:tale`/`:plan` and `:epic` limit which plan
> tier auto-resolves (the other tier asks); questions always wait for a human;
> `:manual`/`:off` turn auto off; other forms fail at launch". Keep the table aligned
> (`just fmt`), then run `sase memory init`.

> [!decision] macros_md_auto_row = no
>
> Leave the note alone and follow `/sase_memory_write`'s Declined Changes rule.

No other memory note becomes false. `decisions:gates-never-block` still holds, because
plan gates still auto-resolve.

## 6. Research roadmap: the 2026-10-24 amendment

Open the sidecar with
`sase repo open sase--research -r "Amend the %auto roadmap: %auto never answers questions"`.

- Edit only `202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md`.
- Leave the `__final` and `__<model>` reports, both baselines, and the images alone.
- Keep the report's voice and the two existing amendments' style: short inline
  "(…amendment…)" notes, never deleted history.
- Where an edit below says "this tale's plan reference", use the `@plan:` ref from your
  launch prompt.

1. **Amendment callout.** Add a third callout under the existing two, titled "Amendment
   (2026-10-24): `%auto` never answers questions". Cover:
   - The user reversed question auto-answering. Every built-in profile has
     `question: ask`, including E3's planned built-ins `attended` and `overnight`.
   - Built-in profile policies are core code that every reader refreshes, so agents
     launched before the change stopped answering too.
   - Question automation (`first`, and E3's `recommended`/`decide`) is opt-in only,
     through a user- or project-config profile that names a `question:` value
     explicitly. Prompts can never grant it.
   - Why: a question exists because the agent needs a human, and the first option would
     silently stand in for one.
   - Name this tale's plan reference.
   - The three amendments override the `__final` report and both baselines wherever they
     disagree.
   - Change the infographic note to say it predates all three amendments.
2. **The yardstick table.** Set the Question column to `ask` in every row. Add one
   sentence above the table: before the 2026-10-24 amendment, every non-manual row took
   the first option.
3. **E1 section.**
   - Annotate "auto-answered questions included" as history.
   - Mark the `question_default` Plan Decision as settled by the amendment: questions
     ask, neither `first` nor `recommended`.
   - Add an "**Amendment: questions wait.**" bullet listing what this tale shipped:
     - `question: ask` in every built-in profile;
     - core `autonomy_effective_record`, with the read-time refresh of stored built-in
       policies and its binding;
     - the contract-row and test updates;
     - the docs, and the memory row if accepted.
4. **E2 section.** In E2.2 `tui_status`, note that no built-in profile is `autopilot`
   any more (`standard` is `attended`), so class colors must not assume `standard` is
   fully automatic.
5. **E3 section.**
   - In the Result and Vocabulary bullets, make `question: first | recommended | decide`
     config-profile-only opt-ins. No built-in (`standard`, `attended`, `overnight`)
     carries a non-`ask` question value, and `%auto(<profile>, q=…)` can only narrow.
   - In E3.1 `core_profiles`, say that config profiles enter through the non-built-in
     lane of `autonomy_effective_record`. E3 must either keep "built-in profile policies
     are core code, refreshed on read" or decide explicitly how stored built-in policies
     migrate.
   - Add exit criteria:
     - no built-in profile auto-answers questions;
     - a config profile with `question: first` or `recommended` does, and `explain`
       shows its layer.
6. **Risks.**
   - Rewrite the "Agents cannot tailor work to their autonomy" row: questions now always
     reach a human, so they never get auto-answered.
   - Add a row:
     - Risk: questions park unattended work, such as an overnight epic waiting on an
       answer.
     - Mitigations: question gates already notify in ACE and on Telegram; the
       "recommended option first" rule helps the human answer fast; E3's opt-in config
       profiles.
7. **What would change this recommendation.** Replace the "auto-answered questions
   routinely taking poor first options" bullet with this one: if parked questions
   routinely stall unattended work, define a config profile with `question: recommended`
   (E3) for those runs. Never restore question automation to a built-in profile.
8. **Alternatives rejected.** Add a row: "Keep `first` as the built-in question default
   (E1's compatibility behavior)". Give the reason above. The user rejected it on
   2026-10-24.
9. **Scope and method.** Change "except where the 2026-10-09 amendments override them"
   to name the 2026-10-09 and 2026-10-24 amendments.

Run `git -C <printed path> status` and confirm the change is pending for the final
declaration. Do not hand-commit.

## 7. Verify and land

- **Before you finish,** read `lint_and_test.md` through `/sase_memory_read`, because
  tracked sase files change.
- **sase:** run `just fix`, then `sase tool run check`.
  - Editing `default_config.yml` escalates the scoped lane to the full suite, so hand
    the final check to a verify monitor through `/sase_monitor`.
  - Treat UNKNOWN failures as yours. Check for flake beads before calling a failure
    unrelated. Never weaken an assertion, and never run `just check-full`.
- **Spot-checks** (use `.venv/bin/python -m sase` if the global install is older):
  - `sase autonomy explain -p '%auto'`, `-p '%auto:tale'`, and `-p '%auto:epic'` predict
    that questions wait, while tale and epic outcomes are unchanged.
  - `sase autonomy list` shows `questions ✋` for every profile.
  - `rg -n -i "first option|questions take" src docs sase/memory` matches only
    `docs/blog/`, the kept core decision-sentence path, and test fixtures.
- **Commit subject for sase:**
  `feat(autonomy): %auto no longer answers questions; every question waits for a human`.
- **In your final reply,** tell the user two things:
  - A runner process that started before the upgrade keeps the old behavior until it
    refreshes.
  - Question automation is deliberately deferred to E3's config profiles.

## Out of scope

- An opt-in now: no `%auto` spelling, config key, or override (that is E3's).
- `%auto` grammar, completion, LSP, or nvim metadata (none mentions questions).
- Plan and epic gate behavior.
- The TUI (the summary text reaches only CLI surfaces) and sase-telegram (it shows the
  plan auto badge only).
- `/sase_questions`: its "recommended option first" rule stays, because it now helps the
  human.
- Rewriting stored records on disk; readers refresh them.
- Decision-log history.
