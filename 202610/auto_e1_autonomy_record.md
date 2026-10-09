---
tier: epic
title: "%auto E1: one autonomy record"
goal: "Every automatic gate outcome comes from one Rust evaluate() applied to one
  persisted, revisioned agent_meta.autonomy record that every agent-session member
  inherits. `sase autonomy explain` predicts each decision exactly, every decision is
  logged, the agent is told its policy, and `%auto` otherwise behaves exactly as it does
  today.

  "
decisions:
  decision_record:
    ask:
      Add a decisions memory record that autonomy is one record evaluated in core, not
      gate UI defaults?
    memory:
      - decisions:autonomy-one-record
    default: false
    answer: false
phases:
  - id: contract
    title: Autonomy behavior contract suite
    depends_on: []
    size: medium
    description:
      "contract: table-driven %auto behavior contract that runs every spelling and state
      through real launch, successor, and gate code, with Python/Rust/LSP parity and
      strict-xfail rows for the deliberate E1 changes."
  - id: core_policy
    title: Core autonomy record, compatibility profiles, and evaluate()
    depends_on: []
    size: medium
    description:
      "core_policy: sase-core autonomy module with the v1 record, policy, request, and
      decision wires, the %auto compatibility translation, evaluate() over explicit
      option IDs, the agent-scan field, and the Python bindings."
  - id: core_summary
    title: Core summary, sentences, mutation, and decision log
    depends_on:
      - core_policy
    size: medium
    description:
      "core_summary: sase-core summary wire, decision and awareness sentences, revision-
      checked mutate_autonomy and inherit with tighten-only agent actors, the built-in
      profile catalog, and the host decision log store."
  - id: record
    title: Persist the record and read it everywhere
    depends_on:
      - contract
      - core_policy
    size: medium
    description:
      "record: resolve and persist agent_meta.autonomy at every launch behind the
      autonomy_record_only sunset flag, and move every Python, TUI, listing, and scan
      reader of %auto state onto it."
  - id: inherit
    title: Structural inheritance and a truthful A toggle
    depends_on:
      - record
      - core_summary
    size: medium
    description:
      "inherit: every host-composed session successor inherits the live record
      structurally, auto_launch_prefix goes away, retries cannot resurrect a toggled-off
      auto, and A writes through mutate_autonomy and restores the last profile."
  - id: gates
    title: Gates decide through evaluate()
    depends_on:
      - record
      - core_summary
    size: medium
    description:
      "gates: adapters declare capability sets, plan, epic, and question gates resolve
      through core evaluate() with a policy block and a decision-log row, and the agent
      receives the awareness block."
  - id: cli
    title: sase autonomy CLI, inspect surfaces, and acceptance
    depends_on:
      - inherit
      - gates
    size: medium
    description:
      "cli: sase autonomy explain, list, log, and show; autonomy in agent list, agent
      show, and gate show; the explain-equals-runtime property test; a fakey lifecycle
      e2e; and the docs."
proposed_by: bbugyi200.athena.0yj
decided_by: auto
create_time: 2026-10-09 05:12:49
status: wip
---

- **PROMPT:**
  [prompts/202610/auto_e1_autonomy_record.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/auto_e1_autonomy_record.md)

# Plan: `%auto` E1 — one autonomy record

## Context

This is epic **E1** of the `%auto` autonomy roadmap
(`research:202610/auto_autonomy_epic_roadmap/auto_autonomy_epic_roadmap.md`). It builds
on the accepted baselines
`research:202610/auto_directive_autonomy_policy/auto_directive_autonomy_policy.md`
(requirements R1–R11, defects D1–D8) and
`research:202610/auto_autonomy_profiles_ux/auto_autonomy_profiles_ux.md`. Read them with
`sase artifact read <ref> "<reason>"` when a phase needs their detail.

The P0 safety epic `sase-1id` has landed:

- `%auto` grammar fails closed through the core classifier `classify_auto_directive`;
- readers use live `agent_meta.json` only;
- a plan-tier mismatch parks the gate;
- epic workers emit `%auto:tale`;
- the prompt bar shows `%auto` errors;
- `/sase_questions` says to put the recommended option first.

**E1's one-sentence result.** One Rust evaluator decides every automatic gate from one
live, revisioned session record. `sase autonomy explain` predicts each decision exactly,
every decision is logged, and the state survives every continuation. **No other behavior
changes.**

### What is still wrong after P0

- **State is three overlapping meta keys plus an inert env var.**
  - The keys are `approve`, `auto_approve_plan_action`, and `auto_approve_argument`
    (plus a coupled `plan` key).
  - `SASE_AGENT_AUTO_APPROVE=1` is exported at `axe/run_agent_runner_launch.py` and read
    by nothing.
  - The Rust agent scan does not carry the argument (the `sase-1id` closeout adds it as
    a stopgap).
  - `%auto:tale` and `%auto:epic` agents report `approve=false` in agent listings.
- **Python decides auto in four places**, with duplicated rules:
  - `GateAdapter.resolve_auto_selection` picks the gate's `primary_branch` filtered by
    `default_selected`. That is the D3 root cause.
  - `GateAdapter.automatic_input` builds the automatic input.
  - Plan-tier coverage is recomputed in about five places: `plan_gate.py`,
    `notification_gates/service.py:_normalize_cross_tier_plan_spec`,
    `plan_gate_turn/create.py`, the propose and validate handlers, and
    `_plan_gate_metadata.recorded_auto_covers_plan` for the TUI. The adapter's allowed
    sets duplicate `_plan_gate_metadata`'s.
  - Plan Decisions defaults are taken by the Rust `resolve_binding(..., "auto")`. That
    one is fine and stays.
- **Inheritance is ad hoc (D6, the `%auto` half of `sase-11g`):**

  | Successor                                                                            | Today                                                                                                       | After E1                             |
  | ------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- | ------------------------------------ |
  | In-process coder / feedback replanner                                                | Full state (P0 `inherit_mode`)                                                                              | Inherits the record                  |
  | In-process question successor after an auto-answer                                   | `approve` only, so a `:tale` agent turns manual                                                             | Inherits                             |
  | Pipe / handoff successor                                                             | `approve` only: `:plan` widens to epics; `:tale`/`:epic` turn manual                                        | Inherits                             |
  | Monitor follow-up                                                                    | `auto_launch_prefix` re-emitted from member meta: bare is fine, `:plan` widens, `:tale`/`:epic` turn manual | Inherits                             |
  | Gate follow-up (launch, custom, sudo, question, plan manual approval, workflow HITL) | Always manual                                                                                               | Inherits                             |
  | Coder after a manually approved plan gate, or after `sase plan approve`              | Manual                                                                                                      | Inherits                             |
  | Runner auto-retry                                                                    | Replays the launch `%auto` even after `A` off                                                               | Token rewritten from the live record |

- **`A` toggle-on always restores bare `%auto`**, silently widening `:tale` users.
- **No surface says why auto acted.** Auto-answered questions leave no decision record
  beyond the bundle, the agent is never told its policy (D8), and `sase gate show`
  prints nothing about auto.

### Target behavior after E1 (the contract)

**Bold** marks a deliberate change from today.

| Prompt or state                                                                                                                                                                                                                   | Tale plan gate                                                      | Epic plan gate        | Question gate |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | --------------------- | ------------- |
| No `%auto`                                                                                                                                                                                                                        | ask                                                                 | ask                   | ask           |
| `%auto`, `%a`, `%auto+`, `%auto:true`                                                                                                                                                                                             | approve + archive                                                   | approve + launch clan | first option  |
| `%auto:tale`, `%auto:plan`                                                                                                                                                                                                        | approve + archive                                                   | ask                   | first option  |
| `%auto:epic`                                                                                                                                                                                                                      | ask                                                                 | approve + launch clan | first option  |
| `%auto:manual`, `%auto:off`                                                                                                                                                                                                       | ask                                                                 | ask                   | ask           |
| `%auto:foo`, any `%auto(…)`, `%auto:x(…)`                                                                                                                                                                                         | launch error                                                        | ←                     | ←             |
| Bare `%auto`, then `A` off                                                                                                                                                                                                        | ask                                                                 | ask                   | ask           |
| `%auto:tale` agent, `A` off then on                                                                                                                                                                                               | **approve + archive** (today: bare is restored)                     | **ask**               | first option  |
| Epic phase or land worker                                                                                                                                                                                                         | approve + archive                                                   | ask                   | first option  |
| In-process coder or replanner of a `%auto:tale` planner                                                                                                                                                                           | approve + archive                                                   | ask                   | first option  |
| Host-composed successor of an agent: a gate follow-up (including the coder after a human approves its parked plan in the TUI or with `sase plan approve`), a pipe/handoff successor, a monitor follow-up, or a question successor | **the agent's own row** (today: manual, or `:plan` widened to bare) | ←                     | ←             |
| `A` off on the live member, then its next host-composed successor                                                                                                                                                                 | **ask** (today: monitor follow-ups re-emit `%auto`)                 | **ask**               | **ask**       |
| Any agent: reorder `primary_branch` or turn off `default_selected`                                                                                                                                                                | **unchanged outcome** (today: the plan outcome changes)             | ←                     | ←             |

Launch, sudo, custom, hitl, task/flag triage, snooze, stale-cleanup, plugins-required,
and any unknown gate kind always ask. Question auto-answers stay "first option"; the
`recommended` value and its option flag are E3 work.

### Design every phase shares

**Compatibility profiles.** These are built in and not configurable in E1; config
profiles arrive in E3. The four names are reserved.

| Profile    | Selected by                            | `plan` (tale)                                   | `epic`                                                    | `question`           |
| ---------- | -------------------------------------- | ----------------------------------------------- | --------------------------------------------------------- | -------------------- |
| `manual`   | no `%auto`, `:manual`, `:off`, `A` off | `ask`                                           | `ask`                                                     | `ask`                |
| `standard` | `%auto`, `%a`, `%auto+`, `%auto:true`  | `approve_archive` → options `[approve, commit]` | `approve` → `[approve]`, input `epic_launch_mode: launch` | `first` → `[submit]` |
| `tale`     | `:tale`, `:plan`                       | `approve_archive`                               | `ask`                                                     | `first`              |
| `epic`     | `:epic`                                | `ask`                                           | `approve`                                                 | `first`              |

- `on_ask` is always `park` in E1.
- The policy keys are the profile-facing names `plan`, `epic`, and `question`. Core maps
  the gate kinds `plan` → `plan`, `epic_plan` → `epic`, and `question` → `question`.
  Every other kind is not auto-allowable.

**The record** (`agent_meta.autonomy`, core-owned wire, `schema_version: 1`):

- `profile`: one of the four names.
- `selection`: the text after `%auto` that produced it (`""`, `plan`, `tale`, `epic`,
  `manual`), kept so that legacy projections and `explain` stay exact.
- `policy`: `{gates: {plan, epic, question}, on_ask}`.
- `overrides`: always `{}` in E1.
- `source`: `prompt | tui | cli | inherited | legacy`.
- `inherited_from`: the predecessor agent name, if inherited.
- `last`: the last non-manual `{profile, selection}`, used by the `A` restore.
- `revision`: starts at 1; +1 per applied mutation.
- `digest`: `canonical_json_sha256` of `policy`.
- `updated_at` and `updated_by`: `{kind: human | agent | host, surface, principal}`,
  modeled on the goal ledger's actor wire.

Every agent launched after the `record` phase gets a record. A prompt without `%auto`
gets a `manual` record.

**evaluate(record, request) → decision.** The request is
`{gate_kind, option_ids, capabilities, request_id}`, where `capabilities` are the
decision values the gate's adapter can execute. The algorithm:

1. Map the gate kind to a policy key. A privileged or unknown kind gives `ask`, with
   rule `not_auto_allowable` or `unknown_kind`.
2. A missing value gives `ask` (rule `unmentioned`).
3. The value `ask` gives `ask` (rule `gates.<key>`).
4. A value outside `capabilities` gives `ask` (rule `not_capable`).
5. If any option ID the value requires is absent from `option_ids`, the result is `ask`
   (rule `missing_options`). The host never executes part of a selection.
6. Otherwise the result is `auto`, with that value and its option IDs in canonical
   order.

The decision
`{outcome: auto | ask | deny, value, option_ids, rule, reason, profile, selection, revision, digest, source}`
never depends on `primary_branch`, `default_selected`, or option order. `deny` is
reserved for E3 and never produced in E1.

**Actors.**

- A human actor (TUI, CLI) may set any selection.
- An agent actor may only narrow: for every kind, the new value must be the same or
  `ask`.
- A `%auto` written into a host-composed successor prompt (monitor, gate, or pipe
  follow-up text) is agent-authored.

**One inheritance rule.**

- Host-composed successors inherit their predecessor's **live** record unchanged
  (`source: inherited`). This covers in-process coders and replanners, question
  successors, pipe/handoff successors, monitor and gate members and their follow-ups,
  coders after manual or CLI plan approval, and anything else the host composes.
- An explicit `%auto` in such a prompt can only narrow. A widening request keeps the
  inherited record and is noted in the run log.
- Human-authored launches resolve from their own prompt, exactly as today, with absence
  meaning Manual. This covers new prompts, a `%id(…, session=…)` launch typed by a
  human, and a TUI or CLI retry/restart of a stored prompt.

**Legacy compatibility: the `autonomy_record_only` sunset flag** (default on).

- **On:** only `agent_meta.autonomy` is written. No legacy meta keys, and no
  `SASE_AGENT_AUTO_APPROVE` export.
- **Off:** every record write also writes the legacy keys (`approve`,
  `auto_approve_plan_action`, `auto_approve_argument`, `plan`), reproduced exactly from
  the record by `autonomy_legacy_projection`, and the runner exports
  `SASE_AGENT_AUTO_APPROVE=1` as before.
- **In both states,** readers that find no record translate the legacy keys
  (`autonomy_record_from_legacy_meta`, `source: legacy`). That is how pre-E1 agents and
  history keep reading.
- **Projections keep consumers working with the flag on.**
  - The Rust agent scan derives its legacy wire fields (`approve`,
    `auto_approve_plan_action`, the argument) from the record, so fleet facts and the
    mobile gateway are unaffected.
  - The Python `AgentListEntry` keeps `approve`, `auto_approve_plan_action`, and
    `auto_badge` as record projections, because sase-telegram reads them.
  - Done markers keep their derived `approve` boolean.
- A TUI process started before the upgrade writes only legacy keys from `A`, and a
  record overrides them. The `record` phase's commit subject tells users to restart
  running TUIs after upgrading.
- Removing the flag deletes the Off branch, per `sase_flags.md`.

### Deliberately not in E1

- **Shipping the resolved record through `%dispatch`'s fleet intent.**
  - In E1, the token-to-policy translation is fixed core code, so a remote E1 host
    resolves the dispatched prompt's `%auto` to the identical record.
  - `FleetLaunchIntentWire` is `deny_unknown_fields`, so adding a field now would break
    dispatch to hosts that have not upgraded.
  - This moves to E3, where config profiles first make remote drift possible. `cli` adds
    a test that a dispatched prompt's `%auto` resolves to the same record.
- **E2 and E3 work:**
  - E2: named chips and colors, the TUI Context section, announcements, the brake,
    Telegram, and `sase autonomy pause|resume`.
  - E3: profiles and the `autonomy:` config, a meaning for `%auto(...)`, `deny` /
    `on_ask: deny` / `decide`, and roles.
- **E4 and E5 work:** bulk or remote steering and `sase autonomy set` (E4); launch
  auto-approval (E5).
- **The `recommended` question value and its `recommended: true` option flag** (E3). E1
  keeps `first`; `/sase_questions` already tells agents to put their recommended option
  first.
- **A `preauthorized` list in the awareness block** (E3, with config profiles).
- **The `%queue` half of `sase-11g`.** It stays on that bead.

### Constraints every phase follows

- **Rust core boundary.**
  - Schema, translation, evaluate, summary, sentences, mutation, inheritance, and the
    decision log belong in `sase_core`. Python calls them through `sase_core_rs` from
    one thin adapter package (for example `src/sase/autonomy/`) and never re-implements
    them.
  - Open the core repo with `sase repo open sase-core` and read its `AGENTS.md`: no bare
    `cargo`; use `just test -p sase_core <filter>` and `sase tool run check` there.
- **Core pin.**
  - A phase that commits both repos in one declaration gets `sase-core-revision.txt`
    written by the host. Never hand-edit it.
  - A sase-only phase that calls a binding from an earlier core-only phase runs
    `just ratchet-core-revision` first if the pin does not yet include that commit
    (`docs/rust_backend.md`, "The CI source revision pin").
- **Behavior.** Only the bold rows of the contract table may change. Every other row
  stays green in every phase.
- **Flags.** `record` creates `autonomy_record_only` with `sase flag new`, and every
  phase that touches its branches tests both states. No `beta` flag.
- **Docs ownership.**
  - `cli` owns `docs/macros.md` (Auto Directive section) and `docs/cli.md`.
  - `inherit` owns the `A` passages in `docs/ace.md` plus `docs/monitors.md` and
    `docs/agent_sessions.md`.
  - `record` owns the flag row and env-var row in `docs/configuration.md`.
  - `gates` owns auto-resolution passages in `docs/sdd.md` and `docs/notifications.md`.
- **Memory.** Only `cli` may edit memory, and only if `decision_record` is accepted,
  through `/sase_memory_write`. No other memory note changes in this epic: the
  `macros.md` `%auto` row stays accurate because `%auto` behavior is unchanged.
- **TUI.** Read `tui.md` (and its children) through `/sase_memory_read` before touching
  TUI code. Existing auto glyphs (`⚡`, `⚡T`, `⚡E`, `⚡ PLAN/TALE/EPIC`) must render
  pixel-identically in E1. If a golden must change, regenerate it with
  `just fix-tui-screenshots` through `/sase_monitor` and inspect the report.
- **CLI.** New commands and options follow `cli_rules.md`, which you read through
  `/sase_memory_read`.
- **Coverage line.** Every inspect view `cli` adds ends with
  `Covers host checkpoints only · the agent's shell is not restricted`. Never show a
  padlock or a "restricted" badge.
- **Changelog.** `CHANGELOG.md` is generated by release-please and must never be
  hand-edited. Name behavior changes in plain words in the conventional commit subject.
- **Verification.** Run `just install-venv` if the venv is stale. Run
  `sase tool run check` in every repo you changed. Never run `just check-full`.
- **Contract suite.** Each phase extends it for what it delivers, and removes the
  strict-xfail markers it makes pass.
- **In-flight collisions.** `sase-1hi.10.7` (Plan Decisions receipts), `sase-18i`
  (`plan_approve_handler.py`), `sase-11t` (gate handoff), `sase-1ab.10` (turn rename),
  and `sase-10h` (gate capacity) touch the same files. Rebase onto whatever has landed
  and never wait for them. `gates` must preserve the Plan Decisions take-defaults
  behavior and the quiet auto-approval receipt exactly as master has them.

## Phase `contract`: autonomy behavior contract suite

Create the yardstick that every later phase (and E2/E3) extends. Put it in one
directory, for example `tests/autonomy_contract/`.

1. **Row table** (a data-only module with a docstring explaining how later phases add
   rows and remove xfail markers).
   - Each row is `(id, prompt or state, context)`.
   - Each row expects one outcome per column: tale plan, epic plan, and question.
   - The outcome values are `ask`, `approve+archive`, `approve+launch`, `first`, and
     `launch_error`.
   - Include every row of the contract table above.
2. **Driver.** Run each row through production code, with no mocks on the decision path
   and no provider invocation:
   - **Launch:** `extract_prompt_directives`, then `build_agent_meta`
     (`axe/run_agent_directive_metadata.py`), writing `agent_meta.json` into a temp
     artifacts dir under a temp `SASE_HOME`.
   - **Contexts,** each through the real helper:
     - the `A` toggle: `persist_agent_directive_update` with the TUI payload;
     - in-process coder and replanner: the `live_plan_successor_meta` /
       `create_followup_artifacts` path;
     - monitor member and follow-up: `create_monitor_member`, plus the follow-up prompt
       and the attach-child meta path;
     - gate member and follow-up: `create_gate_turn_member` plus follow-up composition;
     - pipe: the `handle_pipe_marker` meta seeding;
     - epic workers: the `bead/work_prompt.py` rendering.
   - **Outcome:** build real tale, epic, and question gate specs with the production
     builders (`build_plan_approval_gate_spec`, `user_question_gate_spec`) from the
     row's live meta, run them through `create_gate`, and read `.creation_result.json` /
     `response.json`: either auto-resolved option IDs or parked. Reuse
     `tests/fakey/_gate_capacity_helpers.py` patterns.
   - **Keep the driver refactor-proof.** Call the highest-level production entry point
     each context has, behind one small context-adapter function per context. When a
     later phase replaces a helper (for example `inherit` deletes
     `live_plan_successor_meta`), it updates only that adapter function, never the row
     table.
3. **Parity.**
   - Run every spelling row through the Rust typed launch planner
     (`plan_typed_launch_units`) and the editor-diagnostics binding as well.
   - Assert the same accept/reject result and fields everywhere.
   - Fold `tests/test_auto_grammar_parity.py` into the suite.
4. **Strict xfail rows** (`pytest.mark.xfail(strict=True, reason="E1 <phase>")`). Put
   each group in its own module so that the parallel phases don't conflict:
   - **`test_inheritance_contract.py`** (removed by `inherit`):
     - the gate, pipe, monitor, and question successors of a `%auto:tale` agent keep the
       tale outcomes;
     - a monitor follow-up of `%auto:plan` is not widened;
     - `A` off on the live member means the next host-composed successor is manual.
   - **`test_toggle_contract.py`** (removed by `inherit`): for a `%auto:tale` agent, `A`
     off then on gives the tale outcomes.
   - **`test_ui_default_independence.py`** (removed by `gates`): for tale plans, epic
     plans, and questions, reordering `primary_branch` or setting
     `default_selected: false` on the primary options leaves the automatic option IDs
     unchanged. The tale-plan case fails today (auto drops `commit`).
   - Mark only the cases that actually fail on today's master. A case that already
     passes, such as the single-option question gate, is a plain passing test, because a
     strict xfail that passes is a failure.
5. **Consolidation.**
   - Move the P0 assertions that duplicate a contract row into the suite. The sources
     are `tests/test_auto_grammar_parity.py`, `tests/test_plan_auto_live_meta.py`,
     `tests/test_axe_plan_successor_auto_inherit.py`, the cross-tier cases in
     `tests/test_plan_gates_execution.py`, and `tests/plan_gate_turn/test_create.py`.
   - Keep their unit-level assertions where they are.
   - Lose no assertion.

**Acceptance.**

- The suite passes, with only the named strict-xfail rows outstanding.
- Each xfail row fails for its documented reason.
- The suite runs in the default `just test` lane in seconds.

## Phase `core_policy`: core record, compatibility profiles, and evaluate()

Work in the sase-core repo, opened with `sase repo open sase-core`.

1. **Module.** Add `crates/sase_core/src/autonomy/` (`pub mod autonomy;` in `lib.rs`),
   with wires, profiles, legacy, and evaluate submodules:
   - the wires from "Design every phase shares": record, policy, request, decision, and
     actor, all at `schema_version: 1`;
   - `resolve_autonomy_selection(selection: Option<&str>, source, actor, now)`. It
     validates through the existing `classify_auto_directive`. `None` gives `manual`;
     `""`/`true` give `standard`; `plan`/`tale` give `tale`; `epic` gives `epic`;
     `manual`/`off` give `manual`. Anything else returns the classifier's `invalid-auto`
     diagnostic. The result is a revision-1 record with its digest.
   - `autonomy_record_from_legacy_meta(meta)`. It translates the legacy keys exactly as
     today's readers do (`main/plan_approve_handler.py`
     `get_auto_plan_approval_action`/`_raw_auto_plan_argument` and the
     `_plan_gate_metadata.py` coverage tables). It covers every shape today's writers
     produce:
     - bare: `approve`
     - `:plan`: `approve` + argument
     - `:tale`/`:epic`: action + argument + `plan`
     - toggle-on bare
     - revive action-only
     - the legacy action `"plan"`
   - `autonomy_legacy_projection(record)`. It returns the legacy keys and the prompt
     mode for `set_prompt_auto_mode`, reproducing today's writer output exactly from
     `record.selection`.
   - `evaluate(record, request)`, implementing the algorithm above. Unknown kinds give
     `ask`, and combined selections are all-or-nothing.
2. **Agent scan.**
   - `AgentMetaWire` (`agent_scan/wire.rs`) gains a trailing optional `autonomy`,
     omitted when absent so that field-order parity holds.
   - `agent_meta_from_object` reads it.
   - When `autonomy` is present, the scanner fills the existing legacy wire fields
     (`approve`, `auto_approve_plan_action`, and the argument field, if the `sase-1id`
     closeout added it) from `autonomy_legacy_projection`. Every Rust consumer (fleet
     facts, the gateway and mobile projections) then keeps working when Python stops
     writing legacy keys.
   - `fleet_owner_facts.rs` computes "auto-approved for this plan tier" with `evaluate`
     on the record, or on the translated legacy record when there is none, replacing
     `approve || action`.
   - Mirror the field in sase: `src/sase/core/agent_scan_wire_markers.py`,
     `_agent_meta_from_dict`, `tests/test_core_agent_scan_wire_agent_meta.py`, and the
     `python_wire_parity.rs` fixture.
3. **Bindings.**
   - Add `crates/sase_core_py/src/autonomy/mod.rs`, registered from `lib.rs`, exposing
     `autonomy_resolve_selection`, `autonomy_record_from_legacy_meta`,
     `autonomy_legacy_projection`, `autonomy_evaluate`, and
     `autonomy_wire_schema_version`.
   - Use dicts in and dicts out, like `plan_decision_summary`, with validation errors
     raised as `ValueError`.
   - Update `tools/check_sase_core_rs_bindings` / `tools/validate_sase_core_rs` in sase
     if they enumerate bindings.
4. **Tests.**
   - Every compatibility row × gate kind.
   - Unknown, privileged, unmentioned, not-capable, and missing-option cases all give
     `ask`, never a partial selection.
   - Option-order independence.
   - A legacy-translation round trip for every legacy shape listed above.
   - Projection reproduces today's writer output byte for byte.
   - A digest stability golden.
   - Binding round trips.

**Landing.** Commit both repos in one declaration; the sase side is the wire mirror. The
host pins core automatically.

**Acceptance.**

- `just test -p sase_core autonomy` is green.
- The wire parity tests are green in both repos.
- sase behavior is unchanged.

## Phase `core_summary`: core summary, sentences, mutation, and decision log

This phase is sase-core only, plus bindings.

1. **`autonomy_summary(record) -> AutonomySummaryWire`.** The wire is
   `{profile, class, sentence, short, cells[{kind, glyph, effect, rule}], coverage, source, selection, revision}`.
   - **Class:**
     - `autopilot`: plan, epic, and question are all automatic.
     - `attended`: at least one of them asks.
     - `unattended`: `on_ask` is deny; this is unreachable in E1.
     - `manual`
   - **Glyphs:** `✓` automatic, `✋` waits.
   - **Effect wording** (exact strings, from the UX baseline):
     - "Tales: approve + archive, then implement"
     - "Epics: archive + launch workers"
     - "Questions: take the first option"
     - "Waits for you"
   - **Coverage:** the constant
     `Covers host checkpoints only · the agent's shell is not restricted`.
   - **`short`:** the one-liner. For example, `standard` gives
     `tales ✓ · epics ✓ · questions first`.
2. **`autonomy_decision_sentence(decision, context)`.** It produces one line per
   decision, used by `log`, `explain`, and `gate show`. For example:
   - `✓ tale approved + archived · standard · gates.plan`
   - `✋ epic plan waits for you · tale · gates.epic`
3. **`autonomy_awareness_text(record) -> Option<String>`.** It returns `None` for
   `manual`. Otherwise it returns at most six lines:
   - a header naming the profile and saying the block is advisory, with the coverage
     line;
   - one consequence line each for tale plans, epic plans, and questions, addressed to
     the agent. A question line for `first` reads: "answered automatically with each
     question's first option; no human reads them — put your recommended option first";
   - "Launch, sudo, and custom gates: wait for a human."

   For example, `tale` renders as:

   ```text
   SASE autonomy: tale (advisory; covers host checkpoints only, your shell is not restricted)
   - Tale plans: approved and archived automatically, then implemented without review.
   - Epic plans: wait for a human review before anything launches.
   - Questions: answered automatically with each question's first option; no human reads them, so put your recommended option first.
   - Launch, sudo, and custom gates: wait for a human.
   ```

   Add a snapshot test per profile.

4. **`mutate_autonomy(record, {selection, expected_revision, actor})`.**
   - `selection` is `manual`, `restore`, or any `%auto` value text.
   - It returns `{status: applied | unchanged | refused | stale, record, reason}`:
     - `stale` when `expected_revision` differs from the current revision;
     - `refused` when an agent actor would widen any kind;
     - `unchanged` when the result equals the current record (no bump).
   - Setting `manual` stores the previous non-manual `{profile, selection}` in `last`.
   - `restore` applies `last`, else `standard`.
   - Every applied change bumps `revision` and recomputes `digest`, `updated_at`, and
     `updated_by`.
5. **`autonomy_inherit(predecessor, explicit_selection, actor)`.**
   - It returns the successor record: the predecessor's policy, `profile`, `selection`,
     and `last` unchanged, with `source: inherited` and `inherited_from`.
   - An explicit selection is applied as an agent-actor narrowing. A refused widening
     keeps the inherited record and returns the refusal reason.
6. **`autonomy_profiles()`.** The catalog of `manual`, `standard`, `tale`, and `epic`:
   `layer: builtin`, `kind: manual | default | compatibility`, the selections that pick
   each, the per-kind cells, and the generated one-liner.
7. **Decision log store.**
   - Location: `<sase_home>/autonomy/decisions.jsonl`, with bounded rotation (one `.1`
     segment, a size cap of about 2 MiB like `append_jsonl_record`), a lock, and one
     append per entry.
   - `AutonomyLogEntryWire` is
     `{schema_version, at, agent, agent_session, project, gate_kind, gate_id, creator_role, decision}`.
   - `append_autonomy_decision(sase_home, entry)`.
   - `read_autonomy_decisions(sase_home, {since, agent, gate_kind, outcome, limit})`
     returns entries newest first, reads both segments, and tolerates a torn last line.
8. **Bindings and tests.**
   - Add bindings for all of the above (in the same `autonomy` binding module).
   - Unit tests cover each status of `mutate_autonomy` (stale, refused widening by an
     agent, human widening, restore with and without `last`, unchanged), inheritance
     narrowing, class rules, sentences, and the log (rotation, torn line, filters).

**Landing.** Commit sase-core only. Later sase phases that call these bindings run
`just ratchet-core-revision` if the pin is behind this commit.

**Acceptance.** `just test -p sase_core autonomy` and the binding tests are green.

## Phase `record`: persist the record and read it everywhere

**Before you start,** confirm that the `sase-1id` land closeout is on master: the
post-wait re-exec no longer uses `run_agent_runner_refresh._live_auto_prompt_mode`, and
the core `AgentMetaWire` handles the argument. If it is not, note that on your phase
bead and proceed. This phase supersedes both items.

1. **Flag.** Run:

   ```text
   sase flag new autonomy_record_only -k sunset \
     --when-enabled "Only agent_meta.autonomy carries %auto state: launches and autonomy mutations write no legacy approve/auto_approve_plan_action/auto_approve_argument/plan meta keys and the runner exports no SASE_AGENT_AUTO_APPROVE." \
     --when-disabled "Every autonomy record write also writes the legacy %auto meta keys derived from the record, and the runner exports SASE_AGENT_AUTO_APPROVE=1 for bare-auto agents, for consumers that still read them." \
     --remove-when "No maintained consumer in sase, sase-telegram, the mobile gateway, or chezmoi reads the legacy %auto meta keys or SASE_AGENT_AUTO_APPROVE, and a sase release with the autonomy record has shipped."
   ```

   Then paste the registry entry, run `tools/sync_feature_flags_schema`, and add the
   flag row in `docs/configuration.md`. Reword the `SASE_AGENT_AUTO_APPROVE` row there
   as "exported only while `autonomy_record_only` is off; nothing in SASE reads it".

2. **Adapter.** Add one thin Python adapter module (for example
   `src/sase/autonomy/record.py`) over the bindings:
   - `read_record(meta)`: the record, else the legacy translation, else `None` (Manual);
   - `live_record(artifacts_dir)`: missing or unreadable meta means `None`, failing
     closed;
   - `record_meta_patch(record)`: sets `autonomy`, and with the flag off also the
     projected legacy keys; with the flag on it removes them;
   - `auto_applies(record, gate_kind)`: `evaluate` with that kind's standard option IDs
     and capabilities.
3. **Launch.**
   - `build_agent_meta` (`axe/run_agent_directive_metadata.py`) resolves the prompt's
     `%auto` through `autonomy_resolve_selection` (`source: prompt`, `updated_by` a host
     actor with surface `launch`) and writes the record via `record_meta_patch` for
     every launch, including `manual`. `inherit` later makes host-composed successors
     inherit instead.
   - The `SASE_AGENT_AUTO_APPROVE` export in `axe/run_agent_runner_launch.py` moves into
     the flag-off branch.
   - `AgentInfo.approve` and the done marker's `approve` boolean are derived from the
     record.
4. **Readers.** Every reader of `%auto` state goes through the adapter:
   - **Readers in `main/plan_approve_handler.py`:** `get_auto_plan_approval_action`,
     `get_auto_plan_approval_argument`, and `is_auto_approve_active`. They keep their
     signatures for now, as projections of the live record; `gates` replaces their gate
     callers.
   - **Coverage helpers:** `plan_auto_covers_tier`, `effective_plan_auto_argument`, and
     `recorded_auto_covers_plan` in `_plan_gate_metadata.py`, and their callers in the
     propose and validate handlers, `integrations/_agent_list_entry_status.py`, and the
     TUI meta-enrichment loaders. They become `auto_applies(record, kind)`. Delete each
     helper once it has no caller.
   - **Live overlay:** `axe/agent_meta.py`'s `AUTO_STATE_KEYS` /
     `overlay_live_auto_keys` overlays `autonomy`, plus the legacy keys while the flag
     is off. Every existing call site keeps working.
   - **Post-wait re-exec:** the prompt reconcile in `axe/run_agent_runner_refresh.py`
     rewrites the token from the record's projected prompt mode.
   - **Projections from the record (or legacy translation):**
     - agent listing and entry models: `approve`, `auto_approve_plan_action`, and
       `auto_badge` in `integrations/_agent_list_entry_*.py` and the
       `agent/_running_listing_*.py` files;
     - TUI models, loaders, and widgets: `models/_agent_state_session.py`, the
       `_loaders/_meta_enrichment_*` and done loaders, the row prefix, the render cache
       key, and both header renderers;
     - session promotion's `--plan` suffix: `agent/_agent_session_promotion.py`,
       `_agent_session_attach_resolution.py`;
     - revive (`ace/tui/actions/agents/_revive_artifacts.py`), which restores the
       record;
     - the agents-sync export of `approve` (format unchanged).

     sase-telegram reads `AgentListEntry.auto_badge` and `auto_approve_plan_action`, so
     those attributes must keep their values.

5. **Tests.**
   - Both flag states for launch, toggle-written meta, and every reader group.
   - Legacy-only fixtures (no record) still read correctly in both states.
   - Listing and TUI fields keep today's values for every spelling, because the
     projection reproduces the legacy keys exactly.
   - The `tests/ace/tui/visual/test_ace_png_snapshots_agents_auto_approve.py` goldens
     stay pixel-identical.
   - Fixtures that inject legacy keys keep passing through the translation.
   - Extend the contract suite with a flag-off parametrization of every launch row.

**Acceptance.**

- A fresh `%auto:tale` launch writes `agent_meta.autonomy` with
  `{profile: tale, selection: tale, revision: 1}`, and with the flag on, no legacy keys.
- `rg` finds no production reader of the legacy keys outside the adapter, the legacy
  translation, and the flag-off write.
- The contract suite is green, with only its strict-xfail rows outstanding.

## Phase `inherit`: structural inheritance and a truthful A toggle

1. **Writer 1: `create_followup_artifacts`** (`axe/run_agent_helpers_artifacts.py`).
   - Seed the successor's record with `autonomy_inherit` from the predecessor's **live**
     record, read from the predecessor's artifacts dir at creation time.
   - This replaces the `approve`-only copy and the P0 helpers `live_plan_successor_meta`
     / `plan_successor_auto_relationships`. Delete them once they have no callers.
   - It covers the in-process coder and replanner, the question successor
     (`_continue_after_auto_answered_question`), the pipe successor
     (`axe/run_agent_exec_pipe.py`), gate-turn members (`gate_turn/member.py`), and
     monitor members (`monitor/member.py`).
2. **Writer 2: the out-of-process session-attach child.**
   - `spawn_agent_session_successor` (`agent/detached_child.py`), used for monitor and
     gate follow-ups and the plan gate-turn coder, marks its
     `AgentSessionAttachLaunchPlan` (`agent/_agent_session_attach_types.py`) as
     host-composed.
   - The coder launched by `sase plan approve` (`main/plan_direct_approval_prompt.py`
     path) is host-composed too.
   - In `_add_agent_session_metadata` (`axe/run_agent_directive_metadata.py`), a
     host-composed child inherits the parent member's live record through
     `autonomy_inherit`. The prompt's own `%auto`, if any, is passed as the explicit
     narrowing selection.
   - A human `%id(…, session=…)` launch keeps resolving from its own prompt.
   - Note the `parent=lane` resolution caveat: the parent is usually the settled turn
     member, so writer 1 must already have put the record on that member.
3. **Delete `auto_launch_prefix`.** Remove it from `monitor/followup.py` and
   `monitor/continuation_delivery.py`; inheritance replaces it. Keep
   `queue_launch_prefix` (the `%queue` half of `sase-11g`).
4. **Retry.**
   - Runner auto-retry (`axe/run_agent_retry_spawn.py` `_build_resume_prompt`) rewrites
     the `%auto` token in `original_prompt` from the failed agent's live record (its
     projected prompt mode) before spawning, so a retry after `A` off has no `%auto`.
   - TUI and CLI restart keep replaying `raw_prompt.md`, which the toggle already
     rewrites.
5. **`A` toggle** (`ace/tui/actions/agents/_approve.py` →
   `sase agent persist-directive`, `ops/commands/_agent_directive.py`,
   `ace/tui/actions/agents/_directive_persistence.py`).
   - The payload becomes an autonomy mutation
     `{selection: "manual" | "restore", expected_revision}`: `manual` when the live
     profile is not `manual`, otherwise `restore`. A pre-E1 agent with only legacy keys
     is read through the translation and gets a record on its first toggle.
   - Under the existing per-agent lock, the command:
     1. reads the live record;
     2. calls `mutate_autonomy` with a human/tui actor;
     3. writes `record_meta_patch`;
     4. rewrites the stored prompt token from the projection (`set_prompt_auto_mode`, as
        today), so that Retry and fork replay what the user sees.
   - A `stale` result re-reads and retries once, then reports.
   - The in-memory row patch renders from the read-back record.
   - The toast names the target, for example `⚡ tale restored · sase.x` or
     `✋ sase.x is now manual — its next plan, epic, or question will wait for you`.
   - Eligible statuses and the `A` key are unchanged.
6. **`sase-11g`.** Add a note saying that its `%auto` half landed here and the `%queue`
   half remains. Never close it.
7. **Docs.**
   - `docs/ace.md`, the `A` passages: `A` changes the selected agent's record from its
     next gate, later successors inherit it, and turning it back on restores the last
     profile, else `standard`.
   - `docs/monitors.md` and `docs/agent_sessions.md`: successors inherit autonomy
     structurally, and an explicit `%auto` in agent-authored follow-up text can only
     narrow.
8. **Tests.**
   - Remove the xfail markers in `test_inheritance_contract.py` and
     `test_toggle_contract.py`.
   - For each successor kind: inheritance, `A` off on the live member before successor
     creation, narrowing by an explicit `%auto`, and a refused widening.
   - Retry after `A` off.
   - A human `%id(…, session=…)` without `%auto` stays manual.
   - Both flag states.

**Acceptance.**

- Turning `A` off in one session member stays off in the next.
- The contract suite has no inheritance or toggle xfail rows left.
- `rg auto_launch_prefix src/` is empty.

## Phase `gates`: gates decide through evaluate()

1. **Capability sets.**
   - In `notification_gates/adapter.py` and `adapter_registry.py`, replace `auto_policy`
     (`approval | first | forbidden`) with `auto_capabilities: frozenset[str]`:
     - `plan`: `{approve_archive}`
     - `epic_plan`: `{approve}`
     - `question`: `{first}`
     - every other kind: empty
   - Delete `resolve_auto_selection` and `_default_branch_selection`.
   - `automatic_input(spec, decision)` builds the input from `decision.value`: the
     question answers, or the epic launch mode `launch`.
   - Update the tests that construct `GateAdapter(auto_policy=…)` or assert
     `auto_policy == "forbidden"`.
2. **Spec.**
   - `_GateAuto` (`notification_gates/model_request.py`) gains an optional `policy`: the
     creator's record snapshot.
   - Creators attach the live record: the plan gate-turn create path
     (`plan_gate_turn/create.py` via `build_plan_approval_gate_spec`) and the question
     path (`axe/run_agent_exec_questions.py` → `user_question_gate_spec`).
   - `enabled` means a non-manual profile.
   - A hand-built spec with `enabled` and `argument` but no `policy` is translated
     through `autonomy_resolve_selection(argument)`.
3. **Service** (`notification_gates/service.py`, `validation.py`).
   - At creation, build the request from the spec (kind, option IDs, adapter
     capabilities) and call `autonomy_evaluate`. Evaluate exactly once, from the record
     snapshot read at creation.
   - `auto` executes `decision.option_ids` with `automatic_input`, using
     `source="auto_resolution"`, as today.
   - `ask` normalizes the spec to manual before fingerprinting, so the notification and
     pending row are published as today.
   - This replaces `_normalize_cross_tier_plan_spec`, the parking in
     `build_plan_approval_gate_spec`, and the adapter's allowed-argument sets. Delete
     the now-unused coverage tables in `_plan_gate_metadata.py`.
   - Replace the remaining `plan_approve_handler` projections used by gate creation and
     by `plan propose` / `plan validate` with the adapter's evaluate.
   - `validation.py` still pins `primary_branch` for the UI, but auto validation calls
     evaluate.
   - Plan Decisions take-defaults behavior, `decided_by: auto`, and the quiet receipt
     (`sdd/plan_decision_handoff.py`) are untouched.
4. **Policy block.**
   - Write
     `policy: {profile, selection, rule, outcome, value, option_ids, source, revision, digest}`
     into the request envelope's `auto` block, into `.creation_result.json`'s
     `auto_resolution`, and into `response.json` for automatic outcomes.
   - Every plan, epic, and question gate created by an agent carries one, manual agents
     included (rule `manual`).
5. **Decision log.**
   - For every evaluation whose profile is not `manual` (both `auto` and `ask`), append
     an `AutonomyLogEntryWire` through the binding.
   - Derive `creator_role` from existing agent meta: `epic_worker` for bead-work phase
     and land agents, `workflow` for workflow-launched agents, `top_level` otherwise.
   - Log failures warn and never block gate creation.
6. **Awareness block.** Every agent whose live profile is not `manual` gets
   `autonomy_awareness_text` of its live record on each of its turns. Manual agents get
   nothing.
   - **Where:** the root-agent prompt path. The candidates are `invoke_agent` in
     `llm_provider/_invoke.py`, after the continuation-prompt capture and before
     `save_prompt_to_file`, or `_execute_prompt_step` in
     `macro/workflow_executor_steps_prompt.py` under `is_anonymous_workflow`. Pick the
     one that meets every requirement below and say which in your close note.
   - **Requirements:**
     - It reaches every root agent turn, including in-process successors and
       out-of-process follow-ups.
     - It reads the record live at invocation time (via the turn's artifacts dir), so a
       toggle shows up on the next turn.
     - The block lands in the saved prompt artifact and never enters continuation
       replay.
     - It never accumulates across successors.
     - Mentor, CRS, fix-hook, workflow-handler, and standalone-query invocations get no
       block.
   - The text comes only from core; Python never composes it.

7. **Docs.** Update the `docs/sdd.md` / `docs/notifications.md` sentences that describe
   how auto picks options (explicit policy option IDs, not the primary branch) and the
   policy block.
8. **Tests.**
   - Remove the `test_ui_default_independence.py` xfail markers.
   - Policy block on auto, ask, and manual gates.
   - One log row per non-manual evaluation, including auto-answered questions.
   - A hand-built legacy spec.
   - A missing option means a manual gate, never a partial execution.
   - The awareness block is present for `standard`, `tale`, and `epic`, absent for
     `manual` and helper workflows, appears once per turn, and updates after a toggle.
   - The receipt and Plan Decisions tests unchanged.
   - Update `tests/test_plan_gates_execution.py`, `tests/plan_gate_turn/test_create.py`,
     `tests/test_notification_gates.py`, and the fakey custom gate test, which builds
     `GateAdapter(auto_policy="first")`.

**Acceptance.**

- `rg "primary_branch|default_selected" src/sase/notification_gates/` shows no use on
  the auto path.
- Every gate in the contract suite carries a policy block.
- The contract suite is green with no xfail rows except any not yet removed by
  `inherit`.

## Phase `cli`: sase autonomy CLI, inspect surfaces, and acceptance

1. **`sase autonomy`** (new group, per `cli_rules.md`).
   - Register it in `main/parser_registry.py`, `main/parser_full_registrars.py`, and the
     `entry.py` dispatch. Model it on `sase flag`. The bare group delegates to `list`.
   - Subcommands:
     - `explain [AGENT] [-g/--gate ID] [-j/--json] [-p/--prompt TEXT]`:
       - with an agent (resolved the way `sase agent show` resolves names): its live
         record's summary, per-kind cells with rule and source, the decisions so far
         (from the log), and the coverage line;
       - with `-p`: a static dry run that resolves the prompt's `%auto` with the
         side-effect-free scan, never executes `$(...)`, and labels unresolved dynamic
         parts;
       - with `-g`: the policy block of that settled gate, which is the revision that
         decided it.
     - `list [-j/--json]`: the built-in profile catalog with one-liners. Mark `standard`
       as the default.
     - `log [AGENT] [-j/--json] [-k/--kind KIND] [-s/--since WHEN] [-o/--outcome OUTCOME]`:
       entries newest first, rendered with `autonomy_decision_sentence`. Reuse an
       existing `--since` parser (for example `sase.vcs_log.dates`).
     - `show PROFILE [-j/--json]`: the profile matrix, layer (`builtin`), the selections
       that pick it, and the coverage line.
   - Use colored Rich output, sorted subcommands and options, a short alias for every
     option, and help examples.
   - Regenerate `tests/completion/snapshots/cli_spec.json` (`just sync-completion-spec`)
     and add completion kinds (agent name, kind, outcome).
2. **Existing commands.**
   - `sase agent list` gains an `AUTO` column (the profile name; blank for manual).
     `--json` gains
     `autonomy: {profile, class, selection, overrides, source, revision}`, keeping
     `approve` while `autonomy_record_only` exists.
   - `sase agent show` gains an Autonomy section (summary cells plus the coverage line).
   - `sase gate show` gains a `Policy:` line after `Query:`, plus a `policy` key in
     `--json`.
3. **explain-equals-runtime property test.** For every contract row,
   `sase autonomy explain -p "<prompt>" --json` (and `explain <agent>` for the context
   rows) equals the decision recorded in that row's gate policy block, per gate kind.
   Also add the `%dispatch` test: the prompt `%dispatch` forwards resolves to the same
   record on the receiving side.
4. **Fakey lifecycle e2e.**
   - Use the `tests/fakey/` harness patterns (see
     `tests/fakey/test_gate_capacity_plan_e2e.py` and `_runner_slot_harness.py`).
   - Scenario: a `%auto:tale` planner, its auto-approved tale plan, the coder (inherits
     `tale`), a monitor started by the coder, the monitor follow-up (inherits `tale`), a
     gate created by the follow-up, and the gate follow-up (inherits `tale`).
   - Assert the epic plan proposed in the gate follow-up parks.
   - In a second scenario, turn `A` off mid-chain and assert the next follow-up parks.
   - If the provider-driven chain is impractical, drive the real successor and gate
     creation functions with fakey invocations. Say which in your close note.
5. **Docs.**
   - `docs/macros.md` Auto Directive: the record, the four built-in profiles, structural
     inheritance, `A` restore-last, `sase autonomy explain`, and the coverage line.
   - `docs/cli.md` rows for the new group and the new agent/gate fields.
   - The `docs/configuration.md` env-var row if `record` left it stale.
6. **Memory.**

> [!decision] decision_record Through `/sase_memory_write`, add the decisions strand
> `decisions:autonomy-one-record` ("Autonomy Is One Record Evaluated In Core"),
> following the existing decision-record shape: Claim, Why (alternatives rejected: the
> meta-triad plus env snapshot; adapter `primary_branch`/`default_selected` selection;
> prompt re-emission for inheritance), Cost, and Reopens when. Link
> `[[decisions/rust-core-required]]` and `[[decisions/gates-never-block]]`. Then run
> `sase memory init`.

> [!decision] decision_record = no Edit no memory. Record a `PROPOSED FOLLOW-UP:` note
> on your phase bead with the proposed strand: its title "Autonomy Is One Record
> Evaluated In Core" and a one-line claim. The claim is that `%auto` autonomy is one
> revisioned `agent_meta.autonomy` record, inherited structurally and evaluated only by
> core `evaluate()` over explicit option IDs, never derived from gate UI defaults.

**Acceptance.**

- The property test passes for every row.
- `sase autonomy explain`, `list`, `log`, and `show` work with and without `--json`.
- The CLI help and completion snapshot tests pass.
- The e2e passes.

## Landing (for the epic's land agent)

The work is landed only when:

1. **Beads.** All seven phase beads are closed, the `autonomy_record_only` flag bead
   exists, and `sase-11g` carries the `%auto` note and stays open.
2. **Exit criteria** hold on the landed tree:
   - the contract suite has zero xfail rows, and no expectation edits beyond the bold
     rows;
   - explain parity holds on every row;
   - the UI-default-independence test passes;
   - no production code reads `SASE_AGENT_AUTO_*`, and the export lives only in the
     flag-off branch;
   - every gate in the acceptance run carries a `policy` block;
   - turning `A` off in one session member stays off in the next;
   - both flag states are tested.
3. **Demo**, spot-checked live:
   - `sase autonomy explain -p '%auto:tale'`;
   - `sase autonomy explain <a running agent>`;
   - `sase autonomy log --since 1h` lists recent automatic tale, epic, and question
     decisions;
   - `sase agent list` shows the AUTO column;
   - a running `%auto` agent's saved prompt artifact ends with the awareness block, and
     a manual agent's does not.
4. **Memory.** If `decision_record` was accepted, the strand exists and
   `sase memory read decisions:autonomy-one-record -r "<why>"` prints it. Otherwise, the
   `cli` phase bead carries the `PROPOSED FOLLOW-UP:` note.
5. **Watch metrics** (not exit criteria; they feed E3's plan): automatic decisions per
   day by kind and creator role, top-level epic auto-launches, and parked nested epics
   with how long they wait. All of these come from `sase autonomy log`.
