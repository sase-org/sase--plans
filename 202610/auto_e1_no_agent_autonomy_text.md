---
tier: tale
title: Agents are not told their autonomy
goal:
  No SASE agent prompt, generated worker macro, or skill tells an agent its autonomy
  level; the dead core awareness text is gone; and the %auto roadmap records the
  reversal.
size: medium
decisions:
  memory_record:
    ask:
      Add a decision record "Agents Are Not Told Their Autonomy" to the decisions memory
      web?
    memory:
      - decisions:agents-unaware-of-autonomy
    default: false
    answer: false
proposed_by: bbugyi200.athena.0za.f0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-10 05:41:34
status: wip
---

# Plan: Agents are not told their autonomy

## Why

E1's `gates` phase shipped the policy baseline's "agent awareness" requirement (R8).
Every root agent turn whose live autonomy record is not `manual` gets a block appended
to the bottom of its prompt. For a bare `%auto` agent, it looks like this:

```text
SASE autonomy: standard (advisory; covers host checkpoints only, your shell is not restricted)
- Tale plans: approved and archived automatically, then implemented without review.
- Epic plans: archived and launched automatically, then implemented without review.
- Questions: answered automatically with each question's first option; no human reads them, so put your recommended option first.
- Launch, sudo, and custom gates: wait for a human.
```

The user has reversed this. Agents are meant **not** to know their autonomy level. At
the very least, SASE must not draw their attention to it or spend tokens on it every
turn. Agents should do the same work whether a human or a policy settles their gates.
They learn only each gate's result, after the fact. Humans still see autonomy everywhere
they look: `sase autonomy`, `sase agent list/show`, `gate show`, and the TUI.

The block has one producer and one consumer:

- **Producer:** core `autonomy_awareness_text` (sase-core
  `crates/sase_core/src/autonomy/sentences.rs`), bound as the Python binding
  `autonomy_awareness_text`.
- **Consumer:** sase `src/sase/autonomy/gates.py::with_awareness_block`. Its only caller
  is `src/sase/macro/workflow_executor_steps_prompt.py`, in the `is_anonymous_workflow`
  branch just before `invoke_agent`.

Removing the block also strands text that points agents at it or keys on `%auto`. The
`%auto` token itself is already stripped from the cleaned prompt, so an agent cannot act
on that text anyway:

- **Two epic-worker macros** (added by `plan:202610/auto_e1_epic_worker_roles.md`) say
  "...depends on your autonomy profile; the SASE autonomy block in your prompt says
  which."
- **Three generated skills** (`sase_plan`, `sase_questions`, `sase_memory_write`) give
  instructions conditioned on `%auto`.

Read for context:

- `sase artifact read plan:202610/auto_e1_epic_worker_roles.md "<why>"`;
- `sase artifact read research:202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md "<why>"`;
- `sase artifact read research:202610/auto_directive_autonomy_policy/auto_directive_autonomy_policy.md "<why>"`.
  See its "Agent awareness" section and R8/D8.

## 1. sase: stop appending the block

**`src/sase/macro/workflow_executor_steps_prompt.py`:**

- Delete the whole `if self.workflow.is_anonymous_workflow:` block that imports and
  calls `with_awareness_block`, including its comment.
- Do not touch the other `is_anonymous_workflow` branch earlier in the method.
- `expanded_prompt` then reaches `invoke_agent` and `save_chat_history` unchanged.

**`src/sase/autonomy/gates.py`:**

- Delete `AWARENESS_MARKER`, `_awareness_block_for_record`,
  `_awareness_block_for_artifacts_dir`, and `with_awareness_block`, plus their `__all__`
  entries.
- Drop "and the agent awareness block" from the module docstring.
- Remove imports that become unused (`Path` is likely one). Let ruff, mypy, and
  symvision confirm.
- Nothing else in sase calls the `autonomy_awareness_text` binding. Confirm with
  `rg -n "awareness" src tools`. The only hits allowed afterwards are unrelated TUI
  "update awareness" fixtures under `tests/ace/tui`.

## 2. sase: macro and skill wording

**`src/sase/default_config.yml`:** In `bd/land_epic` (the child-epic path paragraph) and
`bd/work_phase_bead` (the last sentence), delete this sentence:

> Whether such a child epic plan launches automatically or waits for human approval
> depends on your autonomy profile; the SASE autonomy block in your prompt says which.

Replace it with nothing. The worker does not need to know who approves its plan, because
its turn ends at `sase plan propose` either way. Re-wrap only the lines the deletion
touches, and keep the rest of each macro byte-identical.

**Skill sources in `src/sase/macros/skills/`** (generated skills):

- **`sase_plan.md`, step 4.** Delete "Under `%auto`, embed only memory decisions and
  make every other choice yourself: auto-approved plans take every default without
  review." Under any autonomy, a plan's decisions carry defaults the planner would
  defend.
- **`sase_plan.md`, step 6.** Delete "`%auto` remains synchronous and continues in this
  process without a detached agent." Nothing the planning agent does depends on it.
- **`sase_questions.md`, "Recommended Option First".** Make the rule autonomy-neutral:
  "Put your recommended option first in every question's options list, then order the
  remaining options by preference." Drop the `%auto` sentence.
- **`sase_memory_write.md`, "Declined Changes".** Key the rule off the decisions block
  the agent actually receives, not `%auto`. For example: "When your plan's decisions
  block says no human reviewed the plan and an unrequested memory decision is off:".
  Keep the three bullets.
  - Leave the host-written block itself
    (`Auto decisions for this plan (final · no human reviewed this plan):`) unchanged.
    That block is a gate result the coder needs, not a policy announcement.

**Deploying the skills:**

- Preview with `sase skill init --diff` only.
- Do **not** deploy (no `--force`, no `chezmoi apply`). Per `generated_skills.md`,
  skills deploy from a landed tree.
- Tell the user in your final reply that `sase skill init --force` is due after landing.

## 3. sase: docs

**`docs/macros.md`, Auto Directive section:** After "Autonomy covers host checkpoints
only; the agent's shell is not restricted.", add one sentence:

> Agents are never told their profile: SASE adds no autonomy text to an agent's prompt,
> and the `%auto` token itself is stripped, so an agent learns only each gate's result.

**`docs/sdd.md`, `docs/configuration.md`, `docs/beads.md`:** Grep for any statement that
the agent is told its autonomy, and correct it if one exists. None is known. Leave
`docs/blog/` alone, and never edit the generated `CHANGELOG.md`.

## 4. sase: tests

**`tests/autonomy_contract/test_gates_evaluate.py`:**

- Delete `test_awareness_block_per_profile_once_and_live`.
- Replace `test_awareness_hook_only_on_root_agent_turns` with a regression test, for
  example `test_agent_prompts_carry_no_autonomy_text`:
  - It reuses `_run_prompt_step` and `_write_autonomy_meta`.
  - It covers anonymous root turns with selections `""` (standard), `"tale"`, `"epic"`,
    and `None` (manual), plus a helper workflow with `"tale"`.
  - For each, the prompt handed to `invoke_agent` equals `"Do the work"` exactly, and
    contains neither `SASE autonomy` nor `autonomy`. Use string literals, since the
    marker constant is gone.
- Update the module docstring, which lists "the agent awareness block".

**`tests/test_bead_macro_tags.py`:** Add a guard. For `bd/work_phase_bead`,
`bd/land_epic`, and `bd/work_task`, `_builtin_prompt_body(name)` must contain neither
`autonomy` (case-insensitive) nor `%auto`.

**`tests/main/test_init_skills_sources.py`:**

- Drop the expectation `"Under `%auto`, embed only memory decisions"`. Keep
  `"Never embed a decision you should make yourself"` and
  `"Put your recommended option first"`.
- Add a test that no skill source under `src/sase/macros/skills/` contains `%auto`. Its
  assertion message should say agents are never told their autonomy.
  - `sase_gate.md`'s custom-gate `"auto": true` field is unrelated and contains no
    `%auto`.

**Other tests:** Update any other test that pins the deleted wording. Find them with
`rg -n "autonomy block|embed only memory|remains synchronous|no human reads them" tests`.

## 5. sase-core: delete the dead text and binding

Open the core checkout with
`sase repo open sase-core -r "Delete the unused agent awareness text and binding"`, and
read its `AGENTS.md`. The awareness text has no other consumer. Its rendered wording
("no human reads them...") is agent-addressed prose that this tale retires. Leaving a
public "awareness" API invites re-adding the block.

- **`crates/sase_core/src/autonomy/sentences.rs`:**
  - Delete `autonomy_awareness_text` and the helpers only it uses: `plan_consequence`,
    `epic_consequence`, `question_consequence`, and `kind_is_automatic`.
  - Remove imports that become unused. Clippy runs with `-D warnings`.
  - Fix the module doc, which mentions "the agent awareness block".
- **`crates/sase_core/src/autonomy/mod.rs`:** Drop `autonomy_awareness_text` from the
  `pub use`.
- **`crates/sase_core/src/autonomy/summary.rs`:** Change the `AUTONOMY_COVERAGE` doc to
  "Coverage line carried by every summary." The coverage line itself stays, because the
  human-facing summaries use it.
- **`crates/sase_core_py/src/autonomy/mod.rs`:** Delete `py_autonomy_awareness_text` and
  its `register_*` line.
- **Tests:**
  - Delete `awareness_text_renders_tale_snapshot_and_manual_none` in
    `crates/sase_core/src/autonomy/tests.rs`.
  - In `crates/sase_core_py/src/autonomy/tests.rs`, rename
    `autonomy_summary_sentence_and_awareness_bindings` to
    `autonomy_summary_and_sentence_bindings` and drop only its awareness assertions.
- **Check:** `rg -n "awareness" crates` matches nothing except the unrelated
  `agent_cli_update_awareness` fixture and the note-attachment JSONL fixture.
- **Commit subject:** removing a Python binding is a breaking change under core's
  conventions. Use a Conventional Commit marked breaking, for example
  `feat(autonomy)!: remove the agent awareness text and its autonomy_awareness_text binding`,
  with a `BREAKING CHANGE:` footer naming the removed binding.
  - Released sase never calls the binding: its `sase-core-rs` window predates the
    autonomy module. Unreleased sase stops calling it in this same turn.
  - The host moves sase's `sase-core-revision.txt` pin when the turn commits both repos.
  - Never edit versions or core's `CHANGELOG.md`.

## 6. Research roadmap: second 2026-10-09 amendment

Open the sidecar with
`sase repo open sase--research -r "Amend the %auto roadmap: agents are not told their autonomy"`.

- Edit only `202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md`.
- Leave the `__final`/`__<model>` reports, the baselines, and the images alone.
- Keep the report's voice and Markdown style, and the style of the first amendment:
  short inline "(…amendment…)" notes rather than deleting history.
- Where this list says "this tale's plan reference", use the `@plan:` ref from your
  launch prompt.

1. **Amendment callout.** Add a second callout directly under the existing epic-worker
   callout. Title it "Amendment (2026-10-09): agents are not told their autonomy".
   Cover:
   - The user reversed the policy baseline's R8 "agent awareness" requirement, and with
     it E1.5's awareness block. SASE adds no autonomy text to agent prompts.
   - Generated worker macros and skills no longer mention `%auto` or autonomy.
   - Agents learn only each gate's result. Humans still see autonomy on every inspect
     surface.
   - Reasons: it costs tokens every turn and draws the agent's attention to it, and
     agents should do the same work either way.
   - Name this tale's plan reference.
   - This amendment and the first override the `__final` source report and both accepted
     baselines wherever they disagree.
   - Change the existing "The infographic predates it." to say it predates both
     amendments.
2. **The yardstick.** Add a bullet: **Prompt silence.** For every row, the agent's
   prompt carries no autonomy text; this tale's regression test is the seed.
3. **P0 table, the `/sase_questions` (D8) row.** Annotate it: its `%auto` clause was
   removed by the amendment, and the autonomy-neutral "recommended option first" rule
   stands.
4. **E1 section.**
   - In the Result list, replace "The agent is told its policy through the awareness
     block." with a line saying agents are not told their policy. The block E1.5 shipped
     was removed by the amendment.
   - E1.5 row: annotate "The awareness block, labeled as soft" as shipped, then removed
     by the amendment.
   - Plan Decisions: mark `awareness_scope` as settled by the amendment (no agent gets a
     block).
   - Add an "**Amendment: no awareness block.**" bullet beside the existing epic-worker
     amendment bullet. It lists exactly what this tale shipped:
     - the hook and `gates.py` helpers deleted;
     - core `autonomy_awareness_text` and its binding deleted;
     - the macro and skill wording;
     - the prompt-silence and skill/macro guard tests.
5. **E2 section.**
   - Result: "Every agent names its profile on every surface" becomes "on every human
     surface (never in the agent's own prompt)".
   - E2.2 `tui_status`: replace "the awareness text verbatim" with "the core summary's
     per-kind cells and coverage line".
   - Exit criteria: add "prompt silence still holds".
6. **E3 section.**
   - E3.2 `vocabulary`: change "typed refusals the agent was warned about" to "typed
     refusals the agent learns from the gate's response (it is never forewarned)".
   - Note that `/sase_questions`' `recommended` guidance stays autonomy-neutral ("mark
     your recommended option"), never "under `%auto`…".
   - E3.4 `compose`: replace "The awareness block covers the new values" with "Agents
     still see no autonomy text; the prompt-silence test covers the new profiles".
   - Exit criteria: add "prompt silence holds for every new profile".
7. **Launch recipe.** Point the E2 (and later) planning reference at
   `research:202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md`, not
   `__final.md`, and say why: only this file carries the amendments. Leave the E1 line
   as history.
8. **Cross-cutting constraints.** Add a bullet to "Every epic plan should state these
   cross-cutting constraints": **Agents are never told their autonomy.**
   - No prompt block, no skill or macro text keyed on `%auto`.
   - Gate responses report results only.
   - Pre-authorization ideas that relied on the awareness block (the policy baseline's
     `preauthorized` list) must become host-enforced gate rules or be dropped.
9. **Risks.** Add a row. Risk: agents cannot tailor work to their autonomy (they may ask
   questions that get auto-answered, or author epics that park). Mitigations:
   - the autonomy-neutral "recommended option first" rule;
   - Plan Decisions defaults;
   - gate results;
   - E3's `question: recommended`/`decide`.
10. **Alternatives rejected.** Add a row: "Tell agents their autonomy (the policy
    baseline's R8 awareness block, shipped in E1.5)". It costs tokens every turn and
    focuses agents on autonomy. The user rejected it on 2026-10-09.
11. **What would change this recommendation.** Add a bullet: if E1's log shows
    auto-answered questions routinely taking poor first options, adopt E3's
    `question: recommended` or `decide`. Do not re-add a prompt block.
12. **Scope and method.** Where it says both baselines are binding, add "except where
    the 2026-10-09 amendments override them".

Run `git -C <printed path> status` and confirm the change is pending for the final
declaration. Do not hand-commit.

## 7. Verify and land

- **Before you finish,** read `lint_and_test.md` through `/sase_memory_read`, because
  tracked sase files change.
- **sase-core:** run `sase tool run check` in the core checkout. It takes about 5
  minutes; give it a 10+ minute timeout.
- **sase:** run `just fix`, then `sase tool run check`.
  - Editing `default_config.yml` escalates the scoped lane to the full suite (30–50
    minutes here), so hand the final check to a verify monitor through `/sase_monitor`.
  - The last runs on this host reported three load-sensitive failures:
    `tests/tool/test_detach.py::test_watchdog_reports_ended_join_monitor`,
    `tests/monitor/test_monitor_supervise_timeout.py::test_run_supervisor_escalates_term_ignoring_chatty_child`,
    and
    `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`.
  - If they recur, rerun them alone and check for flake beads before treating them as
    yours. Never weaken an assertion. Never run `just check-full`.
- **Spot-checks:**
  - `rg -n "SASE autonomy|with_awareness_block|autonomy_awareness_text" src tests`
    returns nothing.
  - `sase autonomy explain -p '%auto'` and `sase autonomy list` still work (use
    `.venv/bin/python -m sase` if the global install is older; never reinstall
    globally).
  - `sase skill init --diff` shows only the intended skill edits.
- **sase commit subject:** a Conventional Commit that names the behavior change in plain
  words, for example
  `fix(autonomy): stop telling agents their autonomy profile in prompts, macros, and skills`.
- **Final declaration:** declare all three changed repos: sase, sase-core, and the
  research sidecar.

> [!decision] memory_record Use `/sase_memory_write` and add the
> `decisions:agents-unaware-of-autonomy` strand in the house decision-record shape.
> Model it on `decisions:host-owned-completion`.
>
> - **Title:** "Agents Are Not Told Their Autonomy".
> - **Claim:** SASE never puts an agent's autonomy policy into its prompt context. There
>   is no appended block and no macro or skill text keyed on `%auto`. Agents learn only
>   gate results; humans see autonomy on inspect surfaces.
> - **Why, over the alternatives:** the policy baseline's R8 awareness block, which
>   shipped in E1.5 and was then removed, and an "unattended profiles only" scope. Both
>   cost tokens every turn and draw focus, and agents should behave the same either way.
> - **Cost:** agents cannot tailor work to their autonomy. This is mitigated by the
>   autonomy-neutral guidance and gate-side vocabulary.
> - **Reopens when:** gate-side fixes such as `question: recommended` cannot correct
>   outcomes that are measurably worse because the agent was unaware.
>
> Link `[[decisions/gates-never-block]]` if it fits. Then run `sase memory init`.

## Out of scope

- **Gate results the agent needs.** These stay as they are:
  - the host-written "Auto decisions for this plan (final · no human reviewed this
    plan)" coder block and its memory-row suffixes;
  - `sase plan validate`/`propose`'s "auto-approved: every decision takes its default"
    and tier-mismatch notes (output of the agent's own command, documented in
    `docs/sdd.md`);
  - auto-answered question follow-ups.
- **Fork continuation replay.** It replays prior raw prompts, including their directive
  tokens, inside a disabled region. It appends no instruction.
- **Already-running agents.** They keep the block already in their saved prompt. Nothing
  is rewritten.
- **The `autonomy.roles` config, gate evaluation, the decision log, and all human-facing
  autonomy surfaces.**
