---
tier: epic
title: Explicit session/turn targets for fork and agent waits
goal: "`#fork:<name>` reliably waits for the whole agent session, even when the parent
  is promoted into a session after the fork launched. Users can explicitly choose
  `session` or `turn` with `#fork(<name>, target=session|turn)`, `%wait(<name>,
  target=session|turn)`, `sase agent wait -T`, and the plan-gate coder options.
  Admission and injected fork history agree on the same bound target.

  "
decisions:
  macros_memory:
    ask:
      Update macros.md for the wait_target type, keyword-only inputs, and the %wait
      target= keyword?
    memory:
      - macros.md
    default: true
    corrects:
      "`agent`/`model`/`effort`, plugin `<dist>@<id>`; defaultless means required."
    answer: true
phases:
  - id: promotion
    title: Promotion-aware bare-name fork waits
    depends_on: []
    size: medium
    description:
      "promotion: bind bare-name forks as session seeds, heal queued legacy kind=agent
      records whose parent was promoted, keep exact members single-turn, pin the 16h
      replay and continuation regressions."
  - id: vocab
    title: Rust wait-target vocabulary and pure rules
    depends_on: []
    size: medium
    description:
      "vocab: add sase-core wait_target module (parse, effective-target rules,
      current-turn selection), the wait_target NamedEnum, pyo3 bindings, and a thin
      Python facade."
  - id: kwonly_core
    title: Keyword-only macro inputs in sase-core
    depends_on: []
    size: medium
    description:
      "kwonly_core: add keyword_only to the Rust macro input contract (catalog, wires,
      frontmatter rules, positional mapping, signatures, gateway contract)."
  - id: wait_runtime
    title: Turn-pinned wait records and evaluation
    depends_on:
      - promotion
      - vocab
    size: medium
    description:
      "wait_runtime: add the shared Python binder that pins a session's current turn
      into v2 identity records, evaluate turn-scoped records, and keep persistence,
      member dirs, and confirmation consistent."
  - id: wait_directive
    title: The %wait target= keyword end to end
    depends_on:
      - wait_runtime
    size: medium
    description:
      "wait_directive: accept and validate %wait(..., target=) in the Python collector
      and Rust editor/typed parsers, feed the binder, round-trip it through
      PromptWaitDirective edits, and update docs."
  - id: agent_wait_cli
    title: sase agent wait --target
    depends_on:
      - wait_runtime
    size: small
    description:
      "agent_wait_cli: add -T/--target session|turn to sase agent wait, using the shared
      binder to wait on one pinned turn."
  - id: coder_options
    title: Turn targets in plan-gate coder options
    depends_on:
      - wait_directive
    size: medium
    description:
      "coder_options: add turn=<agent> to the wait-spec grammar and carry it through the
      approval response, gateless approve, coder prompts, and epic launches as
      %wait(..., target=turn)."
  - id: wait_tui
    title: Show and edit bound turns in the TUI
    depends_on:
      - coder_options
    size: medium
    description:
      "wait_tui: surface bound turns via an additive scan-wire field, accept turn=
      entries in the agent Wait modal, and disable follow-epics for turn targets."
  - id: kwonly_python
    title: Keyword-only macro inputs in sase
    depends_on:
      - kwonly_core
    size: medium
    description:
      "kwonly_python: mirror keyword_only in InputArg, loaders, binder, validators,
      serializers, schemas, macro show/explain, TUI arg assist, and docs."
  - id: fork_history
    title: Fork history follows the bound wait record
    depends_on:
      - promotion
    size: medium
    description:
      "fork_history: have the fork resolve step read the launching agent's bound fork
      records, select sessions by identity, and cut turn-targeted history at the bound
      turn."
  - id: fork_binding
    title: The
    depends_on:
      - wait_directive
      - kwonly_python
      - fork_history
    size: medium
    description:
      "fork_binding: add the keyword-only wait_target input to fork.yml, return
      structured fork references from the launch scan, bind v2 fork records with
      (identity, target) dedupe, and update docs, skills, and memory."
proposed_by: bbugyi200.athena.research.5j.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 17:43:10
status: wip
---

- **PROMPT:**
  [prompts/202610/fork_wait_session_turn_target.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/fork_wait_session_turn_target.md)

# Explicit session/turn targets for fork and agent waits

## Background

The `16h.f0` agent's prompt contained `#fork:16h`, but it started after `16h--plan`
finished, not after the whole `16h` session (plan, gate, code) settled. The research
report covers the full analysis. Read it with
`sase artifact read research:202610/fork_wait_session_vs_turn_target/fork_wait_session_vs_turn_target.md "<why>"`.
In short:

- `resolve_wait_state()` (`src/sase/axe/run_agent_directives_flow.py`) binds every
  implicit `#fork` wait once, at launch, through `fork_wait_dependency()`
  (`src/sase/agent/fork_waits.py`).
- At 15:22, `16h` was still a plain agent. `find_agent_session()` deliberately skips
  plain agents, so the binder recorded `{"kind": "agent", <16h artifact identity>}`.
- About 2½ minutes later, the gate attached and **promoted** `16h` in place: same
  artifact dir, renamed `16h--plan`, `agent_session: 16h`, no `parent_timestamp`.
- `fork_source_status()`
  (`src/sase/core/wait_dependency_resolution/_index_fork_queries.py`) trusts the frozen
  `kind`, so the `"agent"` branch released when that one artifact finished. The visible
  `wait_for: ["16h"]` name check, which is session-aware, was skipped because the name
  also had a fork source.

So this is a bug against the documented contract: a bare name means the session. It also
shows that the session/turn choice is currently made implicitly by timing. This epic
fixes the bug, then makes the choice explicit on every wait surface.

The user accepted all requirements R1–R11 and the recommended answers to the report's
"Decisions for you". The plan treats them as settled:

- History for a turn target covers the session up to and including the bound turn.
- Already-queued legacy waits are healed.
- The input is named `target`.
- The `%wait`/coder-options work lands before the `#fork` syntax.

## Contract (authoritative for every phase)

### Vocabulary

- **Target values:** `session` and `turn`, matched exactly and case-sensitively. One
  shared list, the builtin NamedEnum `wait_target`, serves `#fork`, `%wait`, the CLI,
  and editor completion.
- **Omitted target, decided by the name (R4):**
  - A bare name (`16h`) means `session`.
  - An exact member name (`16h--plan`) means that member, i.e. a turn.
  - Every bound record stores whether the target was given explicitly.
- **Current turn (R8):** the session's current turn at the moment the wait is bound.
  Take the root's current generation chain (the root plus its `parent_timestamp`-linked
  members):
  - Exclude the waiter's own artifact and members that are dependency-parked on their
    own waits.
  - The current turn is the newest remaining member that is still pending (running,
    slot-queued, or a gate/monitor turn without a settled done marker).
  - If none is pending, it is the newest remaining member.
  - It is pinned by artifact identity, so queue delays never move it.
  - A plain, never-promoted agent is a session of one, so its current turn is itself.

### Effective target per reference kind (R10)

| Reference kind                      | omitted                     | `session`                      | `turn`                                         |
| ----------------------------------- | --------------------------- | ------------------------------ | ---------------------------------------------- |
| bare agent/session name             | session (follows promotion) | session                        | current turn, pinned at bind                   |
| exact member `x--suffix`            | that member                 | owning session (root identity) | that member                                    |
| clan (`#fork` / `%wait` clan names) | clan generation (today)     | clan generation                | **launch error**: a clan has no single turn    |
| `@tribe`                            | next member (today)         | next member                    | **launch error**                               |
| named proc / monitor turn           | single run (today)          | single run                     | single run (both values mean the same)         |
| name not found at bind time         | name wait (today)           | name wait                      | **launch error**: turn needs an existing agent |

### Release semantics (R11, unchanged policies)

- Fork waits release when the bound target ends, success or failure. Failures are
  flagged in the injected history, as today.
- Explicit `%wait` stays success-only.
- Implied fork waits never arm epic-follow.
- A `target=turn` wait is never armed for epic-follow:
  - An explicit `for_epic=true` combined with `target=turn` is an error. This mirrors
    the existing `--plan`-row rule.
  - The implicit `for_epic` default skips turn targets.
- Members attached after a verified release can never pull a released waiter back.

### History (R9)

The fork history injected after release follows the bound record and never re-resolves
the name:

- **Session target:** the bound session generation, selected by root identity. All
  members are shown, rendered as today.
- **Bare name with `target=turn`:** the session's members up to and including the bound
  turn. Later members are listed under "Not shown" with the reason "after the bound
  turn".
  - Consequence to document: `#fork(16h, target=turn)` while `16h` is parked at its gate
    waits for the gate decision. Use `#fork:16h--plan` to fork the plan immediately.
- **Exact member:** that member only, as today.

### Wait record format (v2)

v2 is additive. Records without `target` are legacy and keep their current meaning, plus
the healing rule from `promotion`. Fields:

- `record_version: 2`.
- `requested`: the reference as written, after template resolution.
- `name`: unchanged meaning. For a session record it is the session owner name.
- `target`: `"session"` or `"turn"`.
- `target_explicit`: bool.
- Identity fields (`artifact_dir`, `timestamp`, `project_name`):
  - For a turn record, they identify the pinned turn.
  - For a session record, they identify the root.
- `turn_name`: the pinned member's name at bind time, for display and history.
- Fork records only:
  - The existing `kind`. A turn record keeps `kind: "agent"`, so older readers still
    evaluate it as one artifact.
  - `history_only: true` when an explicit `%wait` already gates the same (identity,
    target). Evaluation, member-dir collection, and `fork_source_names` all skip
    `history_only` records. History still reads them.

Explicit `%wait` turn records live in the existing identity-record list
(`wait_identity_deps` / persisted `wait_for_artifacts`). Fork records stay in
`wait_for_fork_sources`.

### Where code lives

The pure rules belong in sase-core behind `sase_core_rs`, per the Rust core boundary:

- target parsing,
- the effective-target table above,
- current-turn selection,
- the NamedEnum,
- `%wait` editor grammar.

Python gathers facts from artifacts and the wait index, then calls the facade. Wait
resolution itself is still Python (`src/sase/core/wait_dependency_resolution/`), and
porting it is out of scope. For any phase that changes sase-core:

1. Open it with `sase repo open sase-core -r "<why>"`.
2. Follow `docs/contribute/rust-backend.md`: `just rust-install` into this checkout's
   `.venv` to exercise new bindings locally.
3. Commit both repos in the phase's declaration so the `sase-core-revision.txt` pin
   moves past the core commit. Use `just ratchet-core-revision` if it has not moved.

No feature flag: each phase lands a complete, user-safe slice.

- `promotion` is a bug fix toward documented behavior.
- `%wait target=` lands whole in `wait_directive`.
- `fork_history` lands before `fork_binding`, so the `#fork` syntax only appears once
  history can honor it.

### Shared regression set

Every phase that touches wait resolution keeps these green:

- `tests/test_wait_dependency_slot_queued_session_members.py`
- `tests/test_gate_wait_dependency_pending_followup.py` (the 11q.w0 handoff-launch
  window)
- `tests/test_gate_wait_dependency_handoff.py`
- `tests/test_monitor_wait_dependency_handoff.py`
- `tests/test_wait_dependency_live_handoff_resolution.py`
- `tests/test_wait_dependency_failed_handoff_turn_supersession.py`
- `tests/test_run_agent_wait_deps_fork_source.py`
- `tests/test_agent_names_extract_naming.py`
- `tests/test_agent_session_durable_json.py`

Fixtures: `tests/_agent_names_fixtures.py` (`make_agent`),
`tests/_axe_chop_wait_checks_helpers.py`, `tests/_monitor_wait_dependency_helpers.py`,
`tests/_gate_wait_dependency_helpers.py`, plus `build_wait_dependency_index(...)` with
`dependency_resolution_status(...)`.

Verify with `sase tool run check`, in sase and, when touched, inside the sase-core
checkout. Never run `just check-full`. Phases that change rendered TUI output run
targeted `just fix-tui-screenshots -- <selectors>` through `/sase_monitor` and inspect
the report.

## Phase: promotion

Fix the incident with no new syntax. Python only.

**Evaluation side (heals queued waits).** In `fork_source_status()`, a record with
`kind: "agent"`, **no `target` field**, a resolvable identity candidate, and both of the
following conditions is evaluated exactly like `kind: "session"`, through
`agent_session_candidate_for_root()`:

- the candidate is now a session root (no `parent_timestamp`, `agent_session_name` set);
- the recorded `name` equals that session's bare owner name.

Exact-member records (`16h--plan`) never match the name test, so they stay single-turn.
Put the predicate in one helper (for example
`fork_source_follows_promotion(dependency, candidate)`) that `dependency_member_dirs()`
(`_index_queries.py`) also uses, so the release confirmation (`_confirmation.py`) sees
the whole session's members. `fork_history` reuses the same helper.

**Session of one.** For `kind: "session"` with a resolvable identity whose candidate is
not (yet) a session root, evaluate the pinned candidate alone with
`_candidate_terminal_for_fork`. Never fall back to name lookup there, because that could
pick up a later namesake.

**Recording side.** In `fork_wait_dependency()`, a bare name that resolves through
`find_named_agent()` to a plain root agent is recorded as a session seed:
`{"kind": "session", "name": <name>, <root identity>}`. The conditions are:

- the agent's `name` equals the requested name;
- the name is not an exact member per `parse_agent_session_name`;
- there is no `parent_timestamp` and no session role.

Exact members, monitors/procs, clans, and fallbacks keep their current records. Do not
add v2 fields here; `fork_binding` does that.

**Tests** (new file, e.g. `tests/test_fork_wait_promotion.py`):

- **16h replay:** a plain running `16h` gets a fork bound as a session seed. Then it is
  promoted (rename to `16h--plan`, add the `16h--gate` member), and the plan finishes
  while the gate is pending. The fork stays waiting. A gate approval plus `16h--code`
  finishing releases it.
- **Legacy heal:** a legacy `kind: agent` bare-name record over a promoted root waits
  for the gate.
- **Legacy exact member:** a legacy `kind: agent` record named `16h--plan` still
  releases when the plan turn ends.
- **Never promoted:** a never-promoted parent releases the fork when it finishes,
  including on failure.
- **Ordering rule:** a successor registered before its predecessor's `done.json` keeps
  the waiter pending.
- **Continuations:** a same-session gate follow-up (bare session name) and a monitor
  follow-up (exact starter) still release, and never wait on themselves.
- **Member dirs:** `dependency_member_dirs` includes the session members for a healed
  record.

**Docs:**

- `docs/reference/agents/agent-sessions.md`: a bare name also covers an agent that
  "becomes a session after the fork or wait was bound".
- `docs/reference/macros/built-ins.md`: the fork-wait section.
- Make the commit a `fix(...)` that states forks of a live planner now wait through
  review and coding. The generated changelog announces it from that commit.

## Phase: vocab

All in sase-core, plus a thin sase facade. There is no user-visible syntax yet.

- New `crates/sase_core/src/wait_target.rs`:
  - `WaitTarget { Session, Turn }`.
  - `WAIT_TARGETS_WITH_DESCRIPTIONS`, one shared constant for the NamedEnum and later
    the `%wait` suggestions:
    - `session`: "Wait for the whole agent session (default for bare names)"
    - `turn`: "Wait for the session's current turn, captured when the wait is bound"
  - `parse_wait_target(raw)`: an exact match, with a did-you-mean error that reuses the
    `check_input_value` style.
  - `effective_wait_target(requested: Option<WaitTarget>, kind: WaitReferenceKind)`:
    - `kind` is one of `bare_agent | exact_member | clan | tribe | proc | unresolved`.
    - It implements the table above.
    - It returns
      `{target, explicit, scope: session | current_turn | member | single | clan | tribe | name}`.
    - Errors carry stable codes (`wait-target-turn-clan`, `wait-target-turn-tribe`,
      `wait-target-turn-unresolved`, `wait-target-invalid-value`). Their messages name
      the reference and suggest the fix.
  - `select_current_wait_turn(members: &[WaitTurnFacts]) -> Option<usize>`:
    - `WaitTurnFacts { timestamp, pending, excluded }`.
    - It sorts by timestamp itself and returns an index into the input, per the R8 rule.
- Add a builtin `wait_target` NamedEnum to `macro_input_types/catalog.rs`, built from
  the shared constant like `effort`.
- Add serde wires and pyo3 bindings in `crates/sase_core_py` (register in the prelude),
  with Rust unit tests for every table cell and selection edge case:
  - all members finished,
  - pending gate,
  - excluded waiter,
  - dependency-parked sibling,
  - empty input.
- sase side:
  - Add `src/sase/core/wait_target_facade.py`, a typed thin adapter with no logic.
  - Add facade tests.
  - Update the Python input-type parity tests (`tests/test_macro_input_type_parity.py`,
    `tests/macro/test_named_input_type_parity.py`).
  - Update any builtin-type listing in `docs/reference/macros/arguments.md`.
  - Move the pin.

## Phase: kwonly_core

Add keyword-only macro inputs to the shared contract in sase-core. The YAML key is
`keyword_only: true`.

**Rules:**

- A keyword-only input never binds positionally; it is set only by `name=value`, in the
  paren form or the line form `#name(args): text`. The colon form cannot set it.
- `keyword_only` combined with `repeatable` is an error.
- At most one repeatable input is allowed, and it must be the last _positional_ visible
  input. Keyword-only inputs may follow it.

**Sites to change:**

- Catalog: `CatalogInput` plus `parse_inputs` / `parse_short_input_value`
  (`macro_catalog/`).
- Wires: `MobileMacroInputWire`, `MacroInputHint`, and the copy in
  `editor/completion/assist_candidates.rs`, all with `#[serde(default)]`.
- Positional mapping must skip keyword-only inputs:
  - `editor/diagnostics.rs` `input_for_position` and `parse_local_inputs`,
  - `editor/argument_spans.rs`,
  - `editor/completion/trigger_context.rs` (colon form, paren form, and
    `selected_for_repeatable`).
  - Example: `#fork(a, b)` maps both values to `name`; `#fork(a, target=x)` maps `x` to
    `target`.
- Frontmatter validation (`editor/frontmatter.rs`):
  - whitelist `keyword_only`,
  - type-check it as a bool,
  - reject the repeatable combination,
  - restate `non_final_macro_frontmatter_repeatable_input` as "last positional input".
- Signatures: keyword-only inputs render with a trailing `=` on the name, e.g.
  `fork(name…?: agent, target=?: session | turn)`. This applies in
  `macro_catalog/entries.rs` `format_inputs`, `assist_candidates.rs` `input_label`, and
  hover. Argument-name completion keeps offering keyword-only inputs.
- Regenerate the gateway contract snapshot (`contracts/api_v1/mobile_api_v1.json`,
  `UPDATE_MOBILE_CONTRACT=1`).

**Tests:** catalog parsing, frontmatter diagnostics, positional mapping (diagnostics,
spans, trigger context), signature rendering, and LSP integration choice completion for
`target=`.

No sase caller exists yet, so this phase lands alone. `kwonly_python` moves the pin.

## Phase: wait_runtime

This phase adds the Python machinery that pins turns. It is reachable only from code,
not syntax, until `wait_directive`.

- New `src/sase/agent/wait_target_binding.py`, the single binder used by `%wait`, the
  CLI, and `#fork`:
  1. Classify the reference: clan (`find_agent_clan`), tribe (`parse_tribe_reference`),
     exact member (`parse_agent_session_name`), session or plain agent
     (`find_agent_session` / `find_named_agent`), proc/monitor (as in `fork_waits.py`),
     or else unresolved.
  2. Call `effective_wait_target` through the facade.
  3. For `current_turn`, gather `WaitTurnFacts` for the root's current generation from
     the wait-dependency index candidates. Use the same facts `_member_is_pending` and
     `is_dependency_parked` use, with `exclude_artifact_dir` = the waiter. Call
     `select_current_wait_turn`.
  4. Return a `WaitTargetBinding` holding the v2 record fields. Errors raise
     `RuntimeError` with the core message, so launch preparation fails fast like today's
     reserved-tribe error.
- Explicit-wait records:
  - turn (current turn or exact member) → an identity record with `target: "turn"`;
  - explicit `session` on an exact member → an identity record pinned to the root with
    `target: "session"`;
  - a bare name with session, implicit or explicit → no identity record. The existing
    session-aware name wait already follows promotion.
- Evaluation: in `identity_status()` (`_index_identity_queries.py`), `target == "turn"`
  evaluates only the pinned candidate:
  - It uses the existing non-session branch and never
    `agent_session_candidate_for_root`.
  - Its failure fallback is scoped to `turn_name`, never the bare session name.
  - `dependency_member_dirs()` contributes only that candidate's dir for turn records.
- Persistence: make sure the v2 keys survive every round trip. Audit:
  - `run_agent_directive_metadata_build.py`,
  - `run_agent_wait.py` / `run_agent_wait_deps.py` (`waiting.json`),
  - `run_agent_wait_markers.py`,
  - `_scope.py` `marker_scope_keys`,
  - `_epic_follow_release.py` `_PREIMAGE_FIELDS`,
  - the AXE chop waiter loaders.

  Binding happens once. Runner re-execs and resumptions reuse the persisted records and
  never rebind, just like the preserved model selection.

- Expose a `resolve_wait_state` hook that accepts `{name: target}` for explicit waits
  and appends binder records to `wait_identity_deps`. `wait_directive` wires the syntax
  into it.
- Tests:
  - binder table cells against fixture artifacts,
  - current turn while a gate is pending, after all members finish, and with the waiter
    excluded,
  - turn-record release at the pinned turn's success,
  - a failed pinned turn not released by a later unrelated session member,
  - record persistence across a runner re-exec,
  - plus the regression set.

## Phase: wait_directive

`%wait(<agents>, target=session|turn)` end to end.

- Python parser:
  - In `src/sase/macro/_directive_collect_main.py`, add `target` to `supported_keys` and
    record `target_raw` per occurrence. Update the "Unsupported keyword on %wait"
    message.
  - In `src/sase/macro/_directive_extract.py`, add `_resolve_wait_targets` mirroring
    `_resolve_wait_for_epic_explicit`. It uses the facade's parser and shares message
    text with Rust. It rejects:
    - an invalid value,
    - `target=` with no agent target in the same `%wait`,
    - conflicting targets for one agent,
    - `target=turn` combined with `for_epic=true`,
    - `target=turn` on an `@tribe`.
  - `PromptDirectives` (`_directive_types.py`) gains `wait_targets: dict[str, str]`
    (explicit only).
  - `_effective_wait_for_epics_of` skips turn-target agents.
- Wire `directives.wait_targets` into `resolve_wait_state()` through the `wait_runtime`
  hook.
- Rust, sase-core:
  - Editor metadata: add `WAIT_KEYWORDS` `target`. Suggestions come from
    `WAIT_TARGETS_WITH_DESCRIPTIONS`; follow the `%hold scope=` pattern. Update
    `argument_hint` to `":agent or (agent, bead=, hood=, time=, for_epic=, target=)"`.
  - Diagnostics in `editor/diagnostics.rs`: `parse_wait_paren_args` learns `target=`,
    and emits the same checks as Python where they are syntactic.
  - Completion tests.
  - Typed launches (`agent_launch/typed_units.rs`): accept and validate `target=` with
    the same messages, and carry it on `RawWaitTarget`. In typed `%proc`/`%if` launches,
    reject `target=turn` with `wait-target-turn-typed-unsupported`; their admission path
    (`launch_admission_runtime.py`) cannot honor a pinned turn. Accept `target=session`
    as a no-op.
  - Clan waves: `clan_waves.rs` `is_time_or_epic_follow` and the Python mirror in
    `src/sase/ace/tui/models/_agent_clan_waves.py` must skip `target=` tokens, so they
    are never read as agent names.
- Edit round-trip:
  - `PromptWaitDirective` (`src/sase/macro/_directive_edit_wait.py`) gains
    `turn_target_agents: tuple[str, ...] = ()`.
  - `_format_wait_directive` groups agents by (follow-epics flag, target). Output looks
    like `%wait(a, target=turn)`, and a turn group never carries `for_epic=true`.
  - The TUI wait-edit paths (`_directive_persistence_wait.py`, `_wait_apply.py`,
    `_wait_runner.py`) must preserve unchanged turn-target agents. Keep their persisted
    bound record rather than rebinding.
- Docs:
  - `docs/reference/macros/directive-launch.md`: the `%wait` keywords, the table and
    consequences above, and the epic-follow interaction.
  - `docs/reference/agents/agent-sessions.md`: the `%wait` target.
  - Regenerate `docs/reference/macros/directives.md` with `just fix`.
- Update the parity and contract tests: `tests/test_macro_directive_contract.py`,
  `tests/test_macro_directive_completion_parity.py`,
  `tests/ace/tui/widgets/test_directive_arg_completion_wait.py`. Add directive
  extraction tests for every rejection and a launch-flow test where
  `%wait(16h, target=turn)` binds the pending gate.

## Phase: agent_wait_cli

Add `-T/--target {session,turn}` to `sase agent wait`. The parser is
`src/sase/main/parser_agent_lifecycle.py` and the implementation is
`src/sase/agents/cli_wait.py` / `wait_watch/`.

- Keep the options alphabetized and write clear help text.
- `turn` uses the `wait_runtime` binder to pin each NAME's current turn at invocation,
  then waits on that single member. Clans and workflows get a usage error carrying the
  core message.
- `session` (the default) keeps today's behavior.
- JSON output reports the bound `turn_name`.
- Regenerate CLI docs with `just fix`.
- Tests cover binding, the clan error, and JSON.

## Phase: coder_options

Choose a per-agent target in the plan gate's Custom Approval wait field. The input
string grammar does not change, so old bundles keep working.

- `src/sase/wait_spec.py`:
  - Add a `turn=<agent>` entry. It is disjoint from plain agents: listing the same agent
    both ways is an error.
  - `parse_wait_spec` returns `turn_agents`. `format_wait_spec` emits agents, then
    `turn=`, then `bead=`, then `hood=`.
  - `wait_spec_from_name_lists` accepts `turn_agents`.
- Approval result:
  - Add an optional `wait_turn_agents` key to the plan-gate result schema
    (`src/sase/plan_gate.py`). Emit it only when non-empty, through
    `_add_optional_wait_fields`, `plan_response_json*`, and the TUI legacy builder.
  - Decode it in `translate_plan_gate_response` and in
    `plan_approval_result_from_gate_response` (`PlanApprovalResult.wait_turn_agents`).
  - Bundles embed their schemas with `additionalProperties: false`. If a bundle's result
    schema lacks the key and turn entries were requested, fail with a clear message:
    "this gate predates turn waits; answer without turn= or re-create the gate". Answers
    without turn entries must keep working on old bundles; test this against a
    pre-upgrade fixture.
- Coder prompts:
  - `prepare_accepted_plan_successor` (`src/sase/axe/run_agent_exec_plan_accept.py`) and
    gateless `compose_coder_prompt` (`src/sase/main/plan_direct_approval_prompt.py`)
    build `PromptWaitDirective(..., turn_target_agents=...)`. The coder's launch then
    binds the turn immediately after approval.
  - Epic approvals: `build_epic_launch_argv` passes `turn=` through `--wait`.
    `sase bead work` parses it, and `bead/work_prompt.py` `_extra_wait_lines` renders
    `%wait(a, target=turn)`.
- TUI: update the `_WaitSpecInputModal` hint and placeholder
  (`src/sase/ace/tui/modals/approve_options_modal.py`) to list `turn=` and the
  already-accepted `hood=`. The "Wait for:" row shows turn entries as `16h (turn)`.
- Help text for `sase plan approve -w` and `sase bead work --wait`.
- Docs: `docs/reference/tui/agent-workflows.md`.
- Tests: `tests/test_wait_spec.py`, `tests/test_plan_gate_wait.py` (including the
  pre-upgrade bundle), the approve-options modal tests, the coder-prompt tests, and the
  epic launch/work rendering tests.

## Phase: wait_tui

Show what was bound, and edit turn targets on waiting agents.

- Add an additive scan-wire field `wait_turn_bindings: [{name, turn_name}]` to
  `AgentMetaWire` and `WaitingMarkerWire` (sase-core `agent_scan`). The scanner fills it
  from `wait_for_artifacts` and non-`history_only` `wait_for_fork_sources` records with
  `target: "turn"`. Convert it in `agent_scan_wire_conversion.py`.
- Render bound turns wherever the TUI shows an agent's wait targets (row wait summary
  and agent details), e.g. `16h → turn 16h--gate`.
- In the agent Wait modal (`modals/wait_modal.py`), the Agents field accepts
  `turn=<agent>` entries with the coder-options grammar and pre-fills existing turn
  targets. Newly added turn entries bind at apply time through the binder.
- `follow_toggle_disabled_reason` (`src/sase/core/wait_epic_follow_view.py`) treats turn
  targets like `--plan` rows.
- Tests: `tests/ace/tui/test_wait_modal*.py`, `tests/test_wait_epic_follow_view.py`, and
  scan-wire conversion. Run targeted PNG goldens for the changed views.

## Phase: kwonly_python

Mirror `kwonly_core` in sase. Start by checking that the pin includes it; ratchet if
not.

- `InputArg.keyword_only` (`src/sase/macro/models.py`), rejecting the repeatable
  combination.
- Loaders: shortform, longform, and workflow YAML (`_loader_parsing_inputs.py`,
  `workflow_loader_parse.py`). Accept the key only when it is exactly `True`.
- `validate_repeatable_input_order` uses the last-positional rule.
- `bind_input_args` / `input_for_position` (`src/sase/macro/input_binding.py`):
  positionals map only onto positional inputs, and the surplus-positional message counts
  positional inputs.
- `workflow_validator_extract.py` repeatable detection.
- Serializers keep the flag: `prompt_frontmatter._input_to_yaml`, `multi_prompt_macros`.
- `macro/input_schemas.py`, then `tools/sync_macro_input_schemas`.
- Rendering:
  - `_catalog_format.format_inputs` uses the same `=` marker.
  - `sase macro show` / `ShowInput` JSON gets an additive field.
  - Also `explain.py` and `_catalog_structured.py`.
- `MacroInputHint` producers (`_input_hint_wire.py`, `highlight.py`,
  `_mobile_helper_catalog.py`).
- TUI arg assist:
  - `_macro_arg_assist_detection_reference.py`: the positional-index mapping skips
    keyword-only inputs.
  - `_macro_arg_assist_detection_contexts.py`: the single-agent-input path still works
    when a keyword-only input follows.
  - `_macro_arg_assist_skeletons.py`: required keyword-only inputs render as `k=`.
- Docs: `docs/reference/macros/arguments.md`.
- Tests: `tests/test_macro_repeatable_inputs.py` and the macro parity/show/arg-assist
  suites.

## Phase: fork_history

Make the `#fork` resolve step follow the binding (R9). Use synthetic v2 records in
tests, because the syntax that produces turn records arrives in `fork_binding`.

- The resolve step, `sase.scripts.agent_chat_from_name.main` →
  `resolve_agent_chat_sources` (`src/sase/scripts/_agent_chat_from_name_sources.py`),
  loads the launching agent's `wait_for_fork_sources` from
  `$SASE_ARTIFACTS_DIR/agent_meta.json`.
- It matches each requested name to a record by `requested`, or by `name` for legacy
  records.
- Session records select the session generation by root identity. Add a lookup by root
  artifact dir next to `find_agent_session`. They never re-resolve the newest namesake.
- Legacy `kind: agent` records that satisfy the `promotion` healing predicate render as
  sessions.
- Turn records on a bare name render the generation's members with
  `timestamp <= pinned`. Later members go under "Not shown" with the reason "after the
  bound turn" (`history/chat_fork/agent_session.py`).
- Exact-member records select that member by identity.
- Clan, proc, and tribe records behave as today.
- No matching record (in-process pipe successors, legacy agents, or `SASE_ARTIFACTS_DIR`
  unset) falls back to today's name resolution unchanged.
- Self-omission of the current artifact stays.
- Tests: `tests/test_agent_chat_from_name_agent_session*.py`,
  `tests/test_fork_history_session.py`, and the 16h scenario. The history after release
  shows plan, gate, and code. A turn record bound at the plan shows only the plan, with
  gate/code "after the bound turn".
- Docs: the fork-history rules in `docs/reference/agents/agent-sessions.md`.

## Phase: fork_binding

The user-facing `#fork(<names>, target=session|turn)`.

- `src/sase/macros/fork.yml`: add the input
  `target: {type: wait_target, keyword_only: true, default: null}`. Its description says
  that when omitted, a bare name means session and an exact member name keeps that one
  member. The resolve step passes `target` to the fallback path from `fork_history`, so
  that in-process successors honor it.
- Launch scan (`src/sase/agent/names/_resume.py`):
  - `_resume_reference_arguments` returns structured
    `ForkReference(name, target, target_explicit)`.
  - It reads named `target=` and `name=` in the paren form.
    `#fork(name=demo, target=turn)` must yield `demo`.
  - It validates the value with the facade.
  - It raises a clear error for `key=value` entries in the colon form, pointing to the
    paren form.
  - `#fork(a, turn)` still means two parents.
  - Keep `fork_agent_names()` as a names-only wrapper.
  - Cover templates, fenced blocks, disabled regions, and multiple references.
- `resolve_wait_state()`:
  - Bind each reference through `fork_wait_dependency(name, target=, target_explicit=)`.
    It now writes v2 records and uses the `wait_runtime` binder for `turn` and for
    explicit `session` on exact members. Clan/tribe `turn` errors fail launch
    preparation.
  - Dedupe on (identity, target): if an explicit `%wait` gates the same normalized name
    with the same effective target, keep the fork record with `history_only: true`
    instead of dropping it. Otherwise keep both.
  - The `promotion` healing predicate must ignore records that carry `target`.
- Rewriters must preserve named arguments:
  - `src/sase/agent/multi_prompt_reference_rewriting.py` (templated-name rebuilds).
  - `multi_prompt_reference_resume.py`: `#fork(target=turn)` with no positional value
    becomes `#fork(<agent>, target=turn)`.
- Tests:
  - every row of the semantics table,
  - `%wait(16h) #fork(16h, target=turn)`: waits for the session, history cut at the
    turn,
  - `%wait:16h #fork:16h`: a history-only record, and the name wait still runs,
  - invalid values,
  - the colon-form error,
  - the `name=` form,
  - the regression set.
- Docs:
  - `docs/reference/macros/built-ins.md`: fork row, waits, examples, and the gate
    consequence.
  - `docs/reference/agents/agent-sessions.md`.
  - The generated skill sources that describe `#fork` waits
    (`src/sase/macros/skills/sase_run.md`, `sase_monitor.md`, `sase_handoff.md`). Read
    the `generated_skills.md` memory first.

> [!decision] macros_memory Update `sase/memory/macros.md` with `/sase_memory_write`
> (then `sase memory init`):
>
> - list the builtin `wait_target` type and `keyword_only` inputs in the Inputs bullet;
> - add `target=session|turn` to the `%wait` row.

## Risks

- Forks of a live planner now wait through human review. That is intended; announce it
  in the `promotion` commit and the docs.
- Session-scope waits inherit any future gate-settlement or handoff windows. The
  ordering test and the shared regression set guard them.
- The approval-bundle schema pinning: covered by the omit-when-empty rule and the
  pre-upgrade fixture in `coder_options`.
