---
tier: epic
title: "Truthful %auto: P0 autonomy safety tales"
goal: "No %auto spelling silently grants more than it says, pressing A to turn auto off
  really turns it off, epic phase and land workers park nested epic plans for a human
  instead of launching them, and the docs, the macros.md memory row, and /sase_questions
  describe the behavior that actually ships. The three P0 task beads (sase-1hg,
  sase-15s, sase-1hh) are closed when the epic lands.

  "
decisions:
  inherit_mode:
    ask:
      Should in-process coder and replan successors keep the planner's %auto:tale or
      %auto:epic mode?
    default: true
    why:
      Otherwise epic workers' coders lose question auto-answers once workers emit
      %auto:tale
    answer: true
  deploy_skill:
    ask:
      Should the land agent deploy the updated /sase_questions skill with sase skill
      init?
    default: true
    answer: true
  macros_row:
    ask:
      Rewrite the %auto directive row in the macros.md memory note to describe the
      shipped behavior?
    memory:
      - macros.md
    default: true
    requested:
      Can you help me implement all of "safety tales" (i.e. P0) recommended by the
    answer: true
phases:
  - id: grammar
    title: Fail-closed %auto grammar in sase-core and Python
    depends_on: []
    size: medium
    description:
      "grammar: add one sase-core classifier for %auto spellings and use it in the Rust
      typed launch extractor, editor metadata, editor/LSP diagnostics, and the Python
      extractor, so named arguments, parenthesized forms, extra positionals, and unknown
      colon values fail at launch, and :manual/:off mean Manual. Commit sase-core and
      sase in the same turn so the host moves the core pin. Close task bead sase-1hg
      when done."
  - id: live_meta
    title: Live agent meta is the only %auto source
    depends_on: []
    size: medium
    description:
      "live_meta: make the plan and question auto readers consult only the live
      agent_meta.json, never the SASE_AGENT_AUTO_* env snapshot, and stop runner
      write-backs of stale in-memory meta from undoing an A toggle. Update the env rows
      in docs/configuration.md. Close task bead sase-15s when done."
  - id: tier_mismatch
    title: A plan-tier mismatch asks instead of erroring
    depends_on:
      - live_meta
    size: medium
    description:
      'tier_mismatch: treat :tale/:plan on an epic plan and :epic on a tale plan as "not
      covered", so the plan parks for a human. Apply this at propose, gate-spec build,
      and gate creation. Fix the auto-handled dismissal and the "auto-approved" notes so
      a parked gate stays visible.'
  - id: epic_workers
    title: Epic phase and land workers run under %auto:tale
    depends_on:
      - tier_mismatch
    size: medium
    description:
      "epic_workers: emit %auto:tale instead of bare %auto for every phase and land
      segment. Seed in-process coder and replan successors from the planner's live auto
      state (per inherit_mode). Flip the bead-work rendering tests and update
      docs/beads.md."
  - id: prompt_bar
    title: Prompt bar shows %auto grammar errors
    depends_on:
      - grammar
    size: small
    description:
      "prompt_bar: show an invalid %auto spelling inline in the ACE prompt input bar's
      context line, and block submit with the same message the launch path raises."
  - id: docs_truth
    title: Docs, memory, and /sase_questions describe shipped behavior
    depends_on:
      - grammar
      - live_meta
      - tier_mismatch
      - epic_workers
      - prompt_bar
    size: medium
    description:
      "docs_truth: rewrite the %auto passages in docs/macros.md and docs/ace.md to match
      the landed behavior. Rewrite the macros.md memory %auto row (macros_row). Add the
      recommended-option-first guidance to the /sase_questions skill source with a test.
      Close task bead sase-1hh when done."
proposed_by: bbugyi200.athena.0yg
decided_by: auto
create_time: 2026-10-08 13:39:25
status: wip
---

- **PROMPT:**
  [prompts/202610/auto_p0_safety_tales.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/auto_p0_safety_tales.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                      | Why                                                                       |
| ------------ | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| derives-from | [research:202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md][1] | Implements the roadmap's P0 safety tales as one epic                      |
| implements   | bead:sase-15s                                                                 | The live_meta phase makes the A toggle truthful                           |
| implements   | bead:sase-1hg                                                                 | The grammar phase lands the fail-closed %auto grammar                     |
| implements   | bead:sase-1hh                                                                 | The docs_truth phase rewrites the %auto docs and the macros.md memory row |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md

<!-- sase:links:end -->

# Plan: Truthful `%auto` (P0 autonomy safety tales)

## Context

`%auto` is in about 70% of prompts. Today four of its spellings and states silently
grant more automation than they say:

- **D1/D2.** `%auto(plan=ask)`, `%a(epic=ask)`, and `%auto(plan, epic)` fail open to
  full bare-`%auto` automation. `%auto:off` _enables_ auto with an opaque argument.
- **D5.** After the TUI `A` toggle turns auto off, the runner's exported
  `SASE_AGENT_AUTO_APPROVE=1` keeps approving the next plan.
- **D7.** Epic phase and land workers carry a literal bare `%auto`. That is how nested
  epics have been auto-approved and launched: 170 times by mid-October, including the
  `sase-11e`/`sase-xe` nesting loops.
- **D3/D4/D8.** The docs and the `macros.md` memory row misdescribe what `%auto` does.
  `/sase_questions` never tells agents that under `%auto` their first option is
  auto-chosen.

The research roadmap recommends shipping five independent "safety tales" (P0) before the
autonomy-record epics (E1+). This epic packages those five tales as phases. The roadmap
explicitly allows one epic with the same content instead of five tales.

| Roadmap tale                                                                                                                 | Phase                           | Task bead closed                          |
| ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------- | ----------------------------------------- |
| Fail-closed `%auto` grammar (Python, Rust typed extractor, editor metadata, LSP), plus the prompt bar showing the same error | `grammar`, `prompt_bar`         | `sase-1hg` (by `grammar`)                 |
| Readers use live agent meta, so the env snapshot is no longer authoritative                                                  | `live_meta`                     | `sase-15s`                                |
| Epic phase/land workers emit `%auto:tale`, and a tier mismatch becomes `ask` at every call site                              | `tier_mismatch`, `epic_workers` | none (new work; its phase beads track it) |
| `docs/macros.md` and the `macros.md` memory row describe observed behavior                                                   | `docs_truth`                    | `sase-1hh`                                |
| `/sase_questions`: put the recommended option first, because under `%auto` it is chosen automatically                        | `docs_truth`                    | none (new work)                           |

The roadmap's two "new" tales are **not** filed as separate task beads: their phase
beads track them.

### Target behavior after this epic

**Bold** marks a deliberate change from today.

| Prompt or state                                         | Tale plan gate                                                  | Epic plan gate                               | Question gate                                   |
| ------------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------- | ----------------------------------------------- |
| no `%auto`                                              | ask                                                             | ask                                          | ask                                             |
| `%auto`, `%a`, `%auto+`, `%auto:true`                   | approve + archive                                               | approve + launch clan                        | first option                                    |
| `%auto:tale`, `%auto:plan`                              | approve + archive                                               | **ask** (today: `sase plan propose` exits 1) | first option                                    |
| `%auto:epic`                                            | **ask** (today: `invalid_auto_argument` error)                  | approve + launch clan                        | first option                                    |
| `%auto:manual`, `%auto:off`                             | **ask**                                                         | **ask**                                      | **ask** (today: auto-answered)                  |
| `%auto:foo`, any `%auto(…)` / `%a(…)`, `%auto:x(…)`     | **launch error** (today: bare automation or an opaque argument) | ←                                            | ←                                               |
| bare `%auto`, then `A` off                              | **ask** (today: the env snapshot still approves)                | **ask**                                      | **ask**                                         |
| epic phase or land worker                               | approve + archive                                               | **ask** (today: approve + launch)            | first option                                    |
| in-process coder or replanner of a `%auto:tale` planner | **approve + archive** if `inherit_mode` (today: ask)            | ask                                          | **first option** if `inherit_mode` (today: ask) |

These gates are never auto-resolved, and this epic does not change that: launch, sudo,
custom, hitl, task/flag triage, snooze, stale-cleanup, and plugin gates.

### Out of scope (E1 and later)

- The `agent_meta.autonomy` record, the core `evaluate()`, and `sase autonomy …`.
- Explicit option-ID selection.
- The awareness block.
- Any `sunset` flag.
- Structural inheritance through gate-turn, pipe, monitor, and handoff follow-ups. This
  is the `%auto` half of `sase-11g`; `sase-11g` stays open for E1.
- Profiles.
- Giving `%auto(...)` a meaning. This epic only rejects it; E3 later defines it.
- Removing the `SASE_AGENT_AUTO_APPROVE` export. E1 retires it behind its sunset flag.

### Constraints every phase follows

- **Rust core boundary.** The `%auto` grammar is shared behavior, so its classifier
  lives in `sase_core`. Python, the typed extractor, and editor diagnostics call it;
  none re-implements it (`rust_core_backend_boundary`). Open the core repo with
  `sase repo open sase-core`. Read its `AGENTS.md` (no bare `cargo`; use
  `just test -p sase_core <filter>` and `sase tool run check` there).
- **No feature flags.** Every change here removes a fail-open or lying branch on
  purpose; no old branch stays reachable.
- **Changelog.** `CHANGELOG.md` is generated by release-please and must never be
  hand-edited. Use a conventional commit subject that names the behavior change in plain
  words (for example
  `feat(bead): run epic phase and land workers under %auto:tale so nested epics wait for review`).
- **Docs ownership.** Only `docs_truth` edits `docs/macros.md` and the `%auto` toggle
  passages in `docs/ace.md`. This avoids conflicts between parallel phases. Other phases
  update only the docs files named in their section.
- **Memory.** Only `docs_truth` edits memory, and only `sase/memory/macros.md` under
  `macros_row`, through `/sase_memory_write`.
- **TUI.** For TUI changes, read `tui.md` and its children through `/sase_memory_read`
  first.
- **Verification.** Run `just install` first if the venv is stale; it rebuilds
  `sase_core_rs` from the linked core checkout. Run `sase tool run check` in every repo
  you changed. Do not run `just check-full` (it is explicit-only).
- **Closing task beads.** A phase that names a task bead closes it after verifying the
  bead's own acceptance, with `sase bead close <id> --note "<what you verified>"`. That
  bead is not an ancestor of the phase, so closing it is part of the phase's definition
  of done. Never close the parent epic.

## Phase `grammar`: Fail-closed `%auto` grammar (closes `sase-1hg`)

### Closed vocabulary

| Spelling (also `%a`)                                                                                                        | Extracted result                                                                                                                                                                                    |
| --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `%auto`, `%auto+`, `%auto:true`                                                                                             | enabled, mode `plan`, argument none (unchanged)                                                                                                                                                     |
| `%auto:plan`, `%auto:tale`, `%auto:epic`                                                                                    | enabled, mode and argument = that value (unchanged)                                                                                                                                                 |
| `%auto:manual`, `%auto:off`                                                                                                 | **Manual**: `auto_enabled=False`, `auto_mode=None`, `auto_argument=None`, exactly as if no `%auto` were present (the token is still stripped from the cleaned prompt, and no meta keys are written) |
| any other colon value, backtick literals included (`%auto:foo`, `%auto:first`, `%auto:epic_plan`, `%auto:x(plan=ask)`)      | `DirectiveError` at launch                                                                                                                                                                          |
| any parenthesized form: `%auto()`, `%auto(tale)`, `%auto(plan=ask)`, `%a(epic=ask)`, `%auto(plan, epic)`, unclosed `%auto(` | `DirectiveError` at launch ("parenthesized arguments are not supported yet")                                                                                                                        |

A duplicate `%auto` stays an error, including `%auto` together with `%auto:off`. The
gate adapters keep their own strict allowlists (`epic_plan`, `first`) for hand-built
`sase gate create` specs; only the prompt grammar closes.

### sase-core (do this first)

1. Add one classifier (for example an `auto_directive` module beside the agent-launch
   directive code).
   - **Input:** an occurrence's form (bare, plus, colon value, or paren) and its raw
     value.
   - **Output:** `{enabled, mode, argument}` or a diagnostic. The diagnostic has a
     stable code (for example `invalid-auto`) and a message that names the offending
     spelling and lists the valid modes.
   - Every surface below shows this exact message text.
2. Use it in the typed extractor's `"auto"` branch
   (`crates/sase_core/src/agent_launch/typed_units.rs` ~400, which today keeps only the
   first arg).
   - Reject paren forms via `DirectiveOccurrence.has_paren_form`. Reject unknown values
     via `typed_plan_diagnostic`, following `parse_tab_target` (~735).
   - For Manual, set `auto_enabled = false` and `auto_mode = None`, so that
     `agent_unit_dispatch_prompt` re-renders no `%auto`.
   - The Rust colon regex stops at `=`, so `%auto:x(plan=ask)` arrives as `x(plan` and
     is rejected as an unknown value. That is fine.
3. Update the editor metadata (`crates/sase_core/src/editor/directive/metadata.rs`,
   ~770):
   - Suggestions become `plan`, `tale`, `epic`, `manual`, `off`, each with a one-line
     doc.
   - The description and hint state the closed vocabulary instead of "interpreted by the
     gate kind".
   - If `editor/wire.rs` has a closed-choice positional role, use it instead of
     `GateOwned`.
   - Keep the syntax forms colon/bare/plus (no paren form).
4. Add an Error-severity editor diagnostic for each invalid `%auto`/`%a` occurrence,
   using the same classifier. Wire it into `analyze_document_with_snapshot`
   (`editor/diagnostics.rs`; model it on `wait_directive_diagnostics`). The macro LSP
   (`crates/sase_macro_lsp`) then publishes it, and sase-nvim gets it through the LSP.
5. Expose the classifier, and the vocabulary list, through a `sase_core_py` binding.
6. Add or update tests:
   - typed-plan tests for every accepted and rejected row (mirror the invalid-tab tests
     in `agent_launch/tests/typed_plan.rs`);
   - the `auto_metadata_describes_gate_owned_resolution…` test in
     `editor/directive/tests.rs`;
   - the directive candidate tests;
   - diagnostics tests;
   - one LSP diagnostics test;
   - a binding round-trip test.

### sase (Python)

1. In `src/sase/macro/_directive_collect.py`, give `auto` its own paren branch for both
   the closed and the unclosed `(` cases. It raises `DirectiveError` with the core
   message before the generic single-value `else` (~214), following the `%tab`/`%final`
   rejections nearby.
2. In `src/sase/macro/_directive_values.py`, `resolve_auto_mode`/`resolve_auto_argument`
   call the binding on the macro-expanded value and raise `DirectiveError` on a
   diagnostic. They must keep running after `expand_single_directive_args`.
3. In `src/sase/macro/_directive_types.py`, `AUTO_COMPATIBILITY_ARGUMENT_SUGGESTIONS`
   becomes the real allowlist, including `manual`/`off`.
   - Source it from the binding, or keep it parity-tested against the core.
   - Rewrite the "not a parser allowlist" comment and the `PromptDirectives` docstring.
4. Change `auto_launch_prefix` (`src/sase/monitor/continuation_delivery.py` ~216) to
   re-emit a stored `auto_approve_argument` only when it is `plan`, `tale`, or `epic`.
   Any other legacy value falls through to the existing action/approve checks, so an old
   `foo`/`off` meta can never produce a follow-up prompt that now fails at launch.
5. Confirm with a test that `set_prompt_auto_mode` replaces an existing
   `%auto:manual|off` token rather than adding a duplicate.
6. Tests:
   - **`tests/test_directives_flags.py`:** a parametrized probe matrix. Every rejected
     row raises `DirectiveError`; every accepted row keeps its fields; Manual rows are
     disabled. Flip `test_auto_argument_is_retained_for_adapter_validation` into a
     rejection test.
   - **Update** `tests/ace/tui/widgets/test_directive_arg_completion_fixed.py`,
     `tests/test_macro_directive_contract.py`,
     `tests/ace/tui/widgets/test_directive_completion_interactions.py`, and
     `tests/test_macro_directive_completion_parity.py`.
   - **Add a parity test** that runs the same matrix through `extract_prompt_directives`
     and through the typed launch planner binding (`plan_typed_launch_units`) and
     asserts the same accept/reject outcome and fields.
7. Do not edit `docs/macros.md` (`docs_truth` owns it).

**Landing the two repos.** Commit both repos in this phase's single final declaration.
The host then commits sase-core first and writes `sase-core-revision.txt` automatically.
Do not hand-edit the pin.

**Acceptance.**

- Every form in `sase-1hg`'s probe list raises `DirectiveError` in Python and a typed
  diagnostic in Rust.
- `%auto`, `%a`, `%auto+`, `%auto:true`, `:plan`, `:tale`, and `:epic` still launch with
  unchanged fields.
- The LSP reports an error on `%auto(plan=ask)`.

Then close `sase-1hg`.

## Phase `live_meta`: the A toggle tells the truth (closes `sase-15s`)

1. **Readers.** In `src/sase/main/plan_approve_handler.py`, `_raw_auto_plan_argument`,
   `get_auto_plan_approval_action`, and `is_auto_approve_active` read only
   `agent_meta.json` under `SASE_ARTIFACTS_DIR`.
   - Remove every read of `SASE_AGENT_AUTO_APPROVE`,
     `SASE_AGENT_AUTO_APPROVE_PLAN_ACTION`, `SASE_AGENT_AUTO_PLAN_ACTION`,
     `SASE_AGENT_AUTO_APPROVE_ARGUMENT`, and `SASE_AGENT_AUTO_PLAN_ARGUMENT`.
   - Missing, unreadable, or non-dict meta means no auto (fail closed).
   - Rewrite the docstrings.
   - These readers feed:
     - the plan gate (`plan_gate_turn/create.py`);
     - the question gate (`axe/run_agent_exec_questions.py` →
       `user_question_gate_spec`);
     - `sase plan propose`;
     - `sase plan validate`.
2. **The export stays, but is inert.** Keep
   `os.environ["SASE_AGENT_AUTO_APPROVE"] = "1"` in
   `src/sase/axe/run_agent_runner_launch.py` (~308); E1 retires it behind its sunset
   flag. Nothing in SASE reads it after this phase. In `docs/configuration.md`:
   - reword its row as a launch-time snapshot that SASE does not consult and the `A`
     toggle does not update;
   - delete the rows for `SASE_AGENT_AUTO_APPROVE_PLAN_ACTION` and
     `SASE_AGENT_AUTO_PLAN_ACTION`, since they are no longer read.
3. **Write-backs must not undo a toggle.** The `A` toggle
   (`ace/tui/actions/agents/_approve.py` → `sase agent persist-directive`) rewrites
   `agent_meta.json` keys `approve`, `auto_approve_plan_action`, and
   `auto_approve_argument`, and strips `%auto` from the prompt artifacts. Exploration
   found these runner paths that write stale in-memory copies back over it:
   - the full overwrites from `bootstrap.agent_meta` in `run_agent_runner_launch.py`
     (~79, ~234, ~318, ~324);
   - the workspace-rebind `_persist_agent_meta` in `run_agent_workspace_identity.py`,
     which goes through `refresh_linked_repos_for_workspace` → `write_agent_meta`;
   - the memory-wins merges in `run_agent_markers.py` (~16-35, ~48-101) and
     `run_agent_wait_markers.py` (~245-266);
   - the post-wait re-exec in `run_agent_runner_refresh.py` (~129). It writes the
     in-memory `submitted_prompt`, which still holds `%auto`, back to the prompt file
     and re-extracts directives.

   Add one helper (for example in `src/sase/axe/agent_meta.py`) that overlays the
   on-disk values of the three auto keys, including their absence, onto an in-memory
   meta dict. Apply it at every runner write-back that runs after initial directive
   extraction. Do not apply it at the first extraction write in
   `run_agent_directives_extract.py`.

   For the post-wait re-exec, make the live meta's auto state win over the in-memory
   prompt's `%auto` before re-extraction: for example, rewrite the prompt with
   `set_prompt_auto_mode` from the live keys, or re-read the persisted prompt artifact.

   Confirm each site against the code. A site that cannot run after the toggle is
   possible needs no change, but say so in your close note.

4. **Out of scope here.** The in-process coder and replan successor seeding belongs to
   `epic_workers`. Gate-turn, pipe, and monitor follow-ups belong to E1 (`sase-11g`).
5. **Tests.**
   - **Readers:** the env var set while meta lacks the keys gives `None`/`False`.
   - **Toggle state preserved:** meta with only `approve` removed stays off through each
     fixed write-back helper call site.
   - **The bead's scenario:** bare `%auto` agent, `A` off while running, and the next
     plan gate is created manual (not auto-resolved).
   - **Fixtures that inject auto state only through env vars:** these must switch to
     meta. Includes `tests/test_plan_command_handler.py`, whose mismatch rows set only
     `SASE_AGENT_AUTO_APPROVE_PLAN_ACTION`, and
     `tests/plan_chain_golden/test_marker_and_loop_golden.py`. Keep those rows' current
     expectations; `tier_mismatch` flips them.

**Acceptance.** Launch bare `%auto`, press `A` off, and the next plan gate parks. Then
close `sase-15s`.

## Phase `tier_mismatch`: a mismatch asks

1. **One helper.** In `src/sase/_plan_gate_metadata.py`, add a single helper that
   classifies an auto argument against a plan tier:

   | Plan tier | Applies                           | Not covered         |
   | --------- | --------------------------------- | ------------------- |
   | tale      | `None`, `""`, `plan`, `tale`      | `epic`, `epic_plan` |
   | epic      | `None`, `""`, `epic`, `epic_plan` | `plan`, `tale`      |

   Any other value is still invalid (`invalid_auto_argument`).
   `validate_plan_auto_argument` keeps raising only for invalid values.

2. **Every call site treats "not covered" as manual:**
   - `src/sase/main/plan_propose_handler.py` (~168-197): do not exit 1. Print one line
     saying, for example, "`%auto:tale` does not cover epic plans; this plan waits for
     review". Print the "auto-approved: every decision takes its default" note only when
     auto applies.
   - `src/sase/main/plan_validate_handler.py` (~94/116 JSON `auto_approved`, ~198/206
     note): use the same rule.
   - `src/sase/plan_gate.py` `build_plan_approval_gate_spec` (~76): build the spec with
     auto disabled.
   - `src/sase/notification_gates/service.py` `create_gate`: normalize a "not covered"
     plan/epic_plan spec to manual (`dataclasses.replace`) **before**
     `validate_gate_spec`. Then `validation.py` (~287), the notification-id assignment
     (~97), the fingerprint, the resume path, and `_resolve_auto_gate` all agree. This
     also covers hand-built `sase gate create` specs.
     `GateAdapter.resolve_auto_selection` stays strict.
3. **Keep a parked gate visible.**
   - `src/sase/plan_gate_turn/create.py` (~149-158) calls
     `mark_auto_approved_plan_handled` and decides the desktop notification from its
     local `auto_enabled`, which is computed before the tier is known. Key both on the
     created gate's effective auto state instead. Otherwise the human's new EpicApproval
     notification gets dismissed.
   - Confirm a parked cross-tier gate shows as pending review in the agent list.
     `integrations/_agent_list_entry_status.py` (~170) hides pending TALE/EPIC whenever
     `auto_approve_plan_action` is set; make it consult the real gate state if it hides
     this case.
4. **Tests.**
   - **Flip:**
     - the `(VALID_TALE, "epic")` and `(VALID_EPIC, "tale")` rows of
       `test_plan_command_rejects_invalid_or_auto_mismatched_plan_without_side_effects`
       in `tests/test_plan_command_handler.py`; they become successful proposals that
       hand off to a manual gate;
     - the cross-tier case of
       `test_auto_rejects_unknown_and_cross_tier_arguments_before_publication` in
       `tests/test_plan_gates_execution.py`; it becomes a manual gate with a
       notification, while `foo` still raises.
   - **Add** in `tests/plan_gate_turn/test_create.py`: an epic plan created under
     `%auto:tale` gives a manual gate that is not marked handled. Repeat for
     `%auto:plan` on an epic and `%auto:epic` on a tale.

**Acceptance.** A fixture epic plan gate whose auto argument is `tale` parks for a
human, and `sase plan propose` of an epic under `%auto:tale` exits 0 and hands off.

## Phase `epic_workers`: workers run under `%auto:tale`

1. **Emit `%auto:tale`.** In `src/sase/bead/work_prompt.py`, `render_multi_prompt` emits
   `%auto:tale` instead of `%auto` for each phase segment (~187) and the land segment
   (~226). `render_task_prompt` stays unchanged.

   This deliberately reverses `496c7945ea`: it returned to bare `%auto` only because
   epic plans were rejected, and `tier_mismatch` removes that reason. Tale plans
   proposed by phase or land workers are still approved and archived. Epic plans park
   for a human.

2. **Seed in-process successors from live meta.** Two successors are seeded from the
   launch snapshot `ctx.agent_meta` today:
   - the in-process coder after plan approval:
     `_plan_followup_base_meta(ctx.agent_meta)` in
     `src/sase/axe/run_agent_exec_plan_accept.py` (~564);
   - the in-process feedback replanner in `src/sase/axe/run_agent_exec_plan.py` (~242).

   Seed both from the predecessor's live `agent_meta.json` (the current phase artifacts
   dir), using `live_meta`'s overlay helper. A toggle made while the planner ran then
   reaches the successor.

   `create_followup_artifacts` (`src/sase/axe/run_agent_helpers_artifacts.py`) copies
   only allow-listed keys. Change only these two call paths. Gate-turn, monitor, pipe,
   and retry members keep today's behavior.

   > [!decision] inherit_mode The two successors also inherit the live
   > `auto_approve_argument` and `auto_approve_plan_action` (never `plan`). A
   > `%auto:tale` planner's coder (the epic workers of step 1 included) keeps `:tale`:
   > its tale plans are approved and archived, its epic plans ask, and its questions get
   > the first option. Add a note on `sase-11g` saying this in-process slice landed
   > here.

   > [!decision] inherit_mode = no The two successors inherit only the live `approve`.
   > The coder of a `%auto:tale` planner, including every epic phase worker's coder,
   > runs with no auto, so its questions and plans park for a human.

3. **Tests.**
   - **Flip** the bare-`%auto` rendering assertions:
     - `tests/test_bead/test_work_epic_plan.py` `test_diamond_render_snapshot`;
     - `tests/test_bead/work_test_helpers.py` `assert_bare_auto_directives` (rename it
       for `:tale`) and its callers;
     - the exact `"%auto\n"` strings in `tests/test_bead/test_work_rendering_models.py`
       and `tests/test_bead/test_work_rendering_changespec.py`;
     - `tests/test_bead/test_cli_work_epic_dry_run.py`.
   - **Add** successor tests for `inherit_mode`, and one where `A` turned auto off
     before approval, so the coder has no auto.
4. **Docs.** Update `docs/beads.md` (~2796-2803 and ~2858-2861): phase and land segments
   carry `%auto:tale`, and a nested epic plan waits for human approval.

**Acceptance.** `sase bead work <epic> --dry-run` shows `%auto:tale` on every phase and
land segment, with no bare `%auto` line.

## Phase `prompt_bar`: the prompt bar shows the error

1. **Context line.** `src/sase/ace/tui/widgets/_prompt_input_bar_dispatch.py` already
   renders "Target error" and "Tab error" segments (`_dispatch_context_text` ~361,
   `_tab_context_segment` ~445). Add an `%auto` segment that shows the core message when
   the draft's `%auto`/`%a` spelling is invalid. A valid or Manual spelling shows
   nothing.
   - Use a cheap scan: the Python collector plus the core classifier from `grammar`, as
     `scan_tab_directive`/`scan_dispatch_directive` do. Do not run full
     `extract_prompt_directives`, which allocates names.
   - This runs on every text change, so honor `tui_perf.md`.
2. **Submit preflight.** `_maybe_preflight_dispatch_submission` (~297) and
   `_commit_or_preflight_submission` in `_prompt_input_bar_submission_actions.py` (~332)
   block submit, keep the draft, and show the same message. The error then never becomes
   a launch-worker failure toast.
3. **Tests.** Widget tests cover the segment (invalid, valid, Manual) and the blocked
   submit. Add a PNG golden only if one fits the existing prompt-bar visual set; follow
   `tui_screenshot.md` if you do.
4. **Docs.** If `docs/ace.md` documents the prompt-bar context line, add the `%auto`
   segment there.

**Acceptance.** Typing `%auto(plan=ask)` in the prompt bar shows the same error text the
launch path raises, and Enter does not submit.

## Phase `docs_truth`: docs, memory, and `/sase_questions` (closes `sase-1hh`)

Write against the landed code, not this plan. Verify each claim in source first; the
section above can drift during implementation.

1. **`docs/macros.md`.**
   - Rewrite the Auto Directive section (~3025-3064):
     - the closed vocabulary, with `:manual`/`:off` as Manual;
     - parenthesized forms are reserved and fail at launch;
     - bare `%auto` approves and archives tale plans (the same as Enter), approves and
       launches epic plans, and answers every question with its first option;
     - `:tale`/`:plan` cover tale plans only, and `:epic` covers epic plans only. The
       other tier parks for review. All three still auto-answer questions;
     - the never-auto gate kinds;
     - the `A` toggle takes effect at the next gate.
   - Fix the stale lines:
     - ~2003 (table row);
     - ~2062 (completion matrix);
     - ~2416-2420 (cheat sheet);
     - ~2937-2938;
     - Plan Approval ~3198-3199 and ~3244-3254. That passage wrongly says `%auto:epic`
       does not answer questions.
2. **`docs/ace.md`.**
   - Fix the `A` toggle passages (~1448, ~6512-6523): toggle-off really stops the next
     gate, and toggle-on means bare `%auto`.
   - Fix any other `%auto` sentence the landed phases falsified. `docs/sdd.md` and
     `docs/cli.md` have "auto-approved: every decision takes its default" lines; adjust
     them if `tier_mismatch` changed when that note prints.
3. **Memory note.**

   > [!decision] macros_row Use `/sase_memory_write` to rewrite the `%auto` row of the
   > directive table in `sase/memory/macros.md`. Keep it to one table row:
   >
   > - the directive cell lists `%auto[:plan/tale/epic/manual/off]`;
   > - the effect cell says: auto-resolves plan, epic, and question gates; bare approves
   >   and archives tales, launches epics, and answers questions with the first option;
   >   `:tale`/`:plan` and `:epic` limit which plan tier auto-resolves (the other tier
   >   asks); `:manual`/`:off` turn auto off; other forms fail at launch.
   >
   > Then run `sase memory init`.

   > [!decision] macros_row = no Do not edit memory. Instead file a `memory` task bead
   > through `/sase_new_task` with the proposed row text, and leave `sase-1hh` open for
   > the land agent to report.

4. **`/sase_questions`.** Add a short section to
   `src/sase/macros/skills/sase_questions.md`:
   - order each question's options with your recommended option first;
   - when the agent runs under `%auto`, SASE answers question gates automatically by
     choosing the first option of every question, and no human reads them.

   Add a distinctive phrase from it to the `sase_questions` row in
   `tests/main/test_init_skills_sources.py`; that test checks the source and every
   provider render. Do not edit rendered `SKILL.md` copies.

**Acceptance.**

- `sase memory read macros.md -r "<why>"` shows a row that matches the target-behavior
  table above.
- `docs/macros.md` no longer claims opaque arguments reach adapters, or that
  `%auto:epic` skips questions.
- The skill-source test passes.

Then close `sase-1hh` (under `macros_row = no`, see that branch above).

## Landing (for the epic's land agent)

The work is landed only when every associated bead is closed:

1. **Phase beads.** All six phase beads are closed.
2. **Task beads.** `sase-1hg`, `sase-15s`, and `sase-1hh` are closed. For any that is
   still open, re-verify the acceptance in its phase section and close it with
   `sase bead close <id> --note "<what you verified; phase that delivered it>"`. Under
   `macros_row = no`, `sase-1hh` closes only once its memory row has landed. Otherwise,
   record that it stays open, and why, in the epic close note.
3. **Spot-check acceptance on the landed tree.**
   - The Python and Rust probe matrices.
   - `sase bead work <this epic> --dry-run` shows `%auto:tale`.
   - A cross-tier fixture gate parks.
   - The prompt bar rejects `%auto(plan=ask)`.
4. **Skill deployment.**

   > [!decision] deploy_skill From the landed, clean tree, deploy the updated skill as
   > `generated_skills.md` says: `sase skill init --force`, then `chezmoi apply` if it
   > was skipped. If the deploy is refused, record the refusal in the close note; do not
   > override it with `--allow-dirty`.

   > [!decision] deploy_skill = no Do not deploy. State in the close note that
   > `/sase_questions` still needs `sase skill init`.

5. **`sase-11g`.** It stays open as E1 scope; never close it here. Under `inherit_mode`,
   confirm the note from `epic_workers` is on it.
