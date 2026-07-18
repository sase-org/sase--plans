---
tier: epic
title: Query-driven sase gates with configurable options
goal: 'Every sase gate is declared by a single boolean option query whose branches
  become the buttons and toggles on every surface (Telegram, ACE TUI, mobile), every
  button and submit control is configurable (label + icon), the tale-tier plan review
  shrinks from seven ambiguous Telegram buttons to five unambiguous ones, and all
  per-kind gate special cases (preset keyboards, choice enums, extras-vs-choices duality)
  are deleted.

  '
phases:
- id: query-model
  title: Gate option query language and envelope v2
  depends_on: []
  description: '''Gate option query language and envelope v2'' section: implement
    the query parser, the unified option/branch/group model, the schema_version 2
    envelope, and the selected-option execution and wait contract in src/sase/notification_gates/.'
- id: gate-cli
  title: First-class sase gate CLI
  depends_on:
  - query-model
  description: '''First-class sase gate CLI'' section: add the `sase gate create`
    and `sase gate wait` command group, retire `sase notify create --gate` and `sase
    notify wait`, and bring help output up to CLI standards.'
- id: producers
  title: Migrate gate producers to option queries
  depends_on:
  - query-model
  description: '''Migrate gate producers to option queries'' section: rebuild the
    plan, epic, launch, and HITL gate producers on envelope v2, collapse the tale
    plan choice set to the (approve AND commit) OR reject OR feedback query, and derive
    the runner response protocol from the selected option set.'
- id: ace-tui
  title: ACE TUI gates on the branch model
  depends_on:
  - producers
  description: '''ACE TUI gates on the branch model'' section: unify the plan-approval
    and custom-gate modals on the shared branch renderer, update keymaps and default_config.yml,
    and refresh visual snapshots.'
- id: mobile-wire
  title: Mobile wire and gateway migration
  depends_on:
  - producers
  description: '''Mobile wire and gateway migration'' section: replace the per-kind
    mobile choice enums in sase-core with the generic gate branch wire, accept selected-option
    submissions, and update contract fixtures and docs/mobile_gateway.md.'
- id: telegram
  title: Telegram unified gate keyboards
  depends_on:
  - producers
  description: '''Telegram unified gate keyboards'' section: render every gate kind
    through one branch-driven keyboard in sase-telegram, delete the plan preset keyboard
    and compatibility paths, and handle toggle/submit callbacks generically.'
- id: skill-docs
  title: Skill and documentation rewrite
  depends_on:
  - gate-cli
  - producers
  description: '''Skill and documentation rewrite'' section: rewrite the sase_gate
    skill template around queries and options, regenerate and deploy skills, and sweep
    remaining gate documentation.'
- id: smoke
  title: End-to-end gate query exercises
  depends_on:
  - ace-tui
  - mobile-wire
  - telegram
  - skill-docs
  description: '''End-to-end gate query exercises'' section: exercise a tale plan
    gate and a custom gate end to end across the CLI, executor, and each rendering
    surface.'
  model: haiku
create_time: 2026-07-17 19:39:53
status: done
bead_id: sase-6p
---

# Plan: Query-driven sase gates with configurable options

## Problem and product context

A tale-tier plan review currently shows **seven** Telegram buttons: `📖 Tale`, `✅ Approve`, `❌ Reject`,
`💬 Send Feedback`, two checkbox add-ons (`💾 Commit plan file to the plans sidecar`, `▶️ Run coder follow-up`), and a
hardcoded `✅ Approve with selected add-ons` submit button. The layout is confusing because the underlying model is
redundant: the tale gate registers six terminal choices (`approve`, `run`, `tale`, `commit`, `reject`, `feedback`) that
mostly encode _combinations_ of two booleans (`commit_plan`, `run_coder`), and then re-encodes the same two booleans as
"extras" on the `approve` choice. Telegram, the ACE TUI, and the mobile wire each carry their own hardcoded projection
of this choice set (`render_plan_gate_keyboard` presets, `PLAN_APPROVAL_CHOICE_RECORDS`, `PlanActionChoiceWire`), plus
legacy compatibility paths.

The fix is not plan-specific. Sase gates in general need one declarative way to say "these are the resolution paths,
these paths are composed of independently selectable commands, and this is how each control looks."

## Design overview

### The gate option query

A gate declares its structure with a single **option query** in the gate request:

```
(approve AND commit) OR reject OR feedback
```

Grammar (deliberately flat — no NOT, no nesting beyond one level):

```
query  := branch ( OR branch )*
branch := option | "(" option ( AND option )* ")"
option := [a-z][a-z0-9_-]*
```

`AND` binds tighter than `OR`; parentheses are conventional but optional. Every option id must appear exactly once in
the query and must have exactly one matching entry in the request's `options` list (and vice versa). Parse errors point
at the offending position, in the spirit of the ACE query parser diagnostics.

Semantics:

- Each **OR branch** is one mutually exclusive way to resolve the gate.
- A **singleton branch** renders as a single button; activating it runs that option's command and resolves the gate.
- An **AND group** renders one toggle per member option (each defaulting to selected, per `default_selected`, which
  defaults to true) plus one submit button. Submitting runs the currently selected members in query order and resolves
  the gate. At least one member must be selected to submit.

The query is parsed **once, at creation time**, inside the gate service. The hashed `request.json` envelope stores both
the canonical query string (for display and debugging) and the normalized branch structure; consumers render from the
structure and never re-parse.

### Options — the only kind of control

The `choices` + `extras` duality is deleted. A request carries one flat, ordered `options` list. Each option:

```json
{
  "id": "commit",
  "label": "Commit plan file to the plans sidecar",
  "icon": "💾",
  "command": { "argv": ["commands/commit_plan"] },
  "default_selected": true,
  "feedback": "disabled",
  "input_schema": {},
  "result_schema": {}
}
```

`label` and `icon` are how _every_ control is configured, uniformly. `feedback` keeps today's
`disabled | optional | required` modes; when an AND group resolves, the effective mode is the strongest mode among the
selected members. Commands keep today's trust model unchanged: shell-free `argv`, first element names a hashed `command`
resource, JSON on stdin, one JSON value on stdout, `result_schema` validation, execution only through the shared
verified executor.

### Groups and configurable submit buttons

Each AND group owns one submit button. Its label and icon default to the group's **first member's** label and icon, and
can be configured explicitly in the request:

```json
"groups": [
  { "options": ["approve", "commit"], "label": "Approve", "icon": "✅" }
]
```

Group configs are matched to query branches by member-id set; configuring a group that is not an AND branch of the query
is a validation error. This is what replaces the hardcoded "Approve with selected add-ons" string: the tale plan gate
configures its submit button as `Approve` with the `✅` icon.

### Envelope v2 and the resolution contract

`schema_version` bumps to `2`. The envelope stores `query`, `options`, `groups`, and the normalized `branches` (a list
of ordered option-id lists). The response contract becomes uniform across all gates:

```json
{ "status": "answered", "selected_option_ids": ["approve", "commit"], "feedback": null, "response_path": "..." }
```

`choice_id` and `selected_extra_ids` are removed everywhere: executor, poller, wait output, mobile bridge, Telegram
progress state. A singleton branch resolution is simply `selected_option_ids: ["reject"]`. The executor validates that
the submitted set is a non-empty subset of exactly one branch, then runs the selected commands in query order under the
existing response lock, hash verification, and atomic-write discipline.

This is a clean break, not a dual-format transition: creation rejects v1 requests with an error that names the v2 shape,
and all in-repo producers migrate in this epic. Gates are short-lived by design, so the only migration cost is that
gates pending across the upgrade must be re-created (see Risks).

### Rendering contract (all surfaces)

Every surface renders from the same branch structure with the same rules:

- Branches render in query order.
- AND-group members render as toggles (`☑️`/`⬜` prefix + icon + label), one per row on Telegram, followed by the
  group's submit button.
- Consecutive singleton branches share a row where the surface allows it.
- When the query contains **exactly one** AND group, that group renders expanded immediately (the common case, including
  tale plans). When a query contains multiple AND groups, groups start collapsed as their submit buttons; activating one
  expands its toggles and collapses the others. This keeps arbitrarily rich gates navigable without cluttering the
  simple ones.

The tale plan review therefore renders exactly five controls:

```
☑️ ✅ Approve
☑️ 💾 Commit plan file to the plans sidecar
✅ Approve                    ← group submit (configured label + icon)
❌ Reject   💬 Send Feedback
```

### First-class CLI: sase gate

Gate creation and waiting move out of `sase notify` into a dedicated group, matching how users already talk about "the
sase gate command":

```bash
sase gate create < gate-request.json > gate-descriptor.json
sase gate wait --id <request-id> --kind custom --json
```

`sase notify create --gate` and `sase notify wait` are removed. The new group follows the CLI rules memory: excellent
`-h` output, alphabetized subcommands, a short alias for every public long option, colored output.

### Rust core boundary

Gate creation has exactly one writer — the Python gate service — so the query parser and envelope authoring stay in
`src/sase/notification_gates/`. What is genuinely shared across frontends is the **envelope v2 wire contract**
(branches, options, groups, selected-option submissions), and that contract is mirrored in the sase-core notifications
wire, which the mobile gateway already uses to read `request.json` directly. If a second gate-producing frontend ever
appears, the parser graduates to sase-core; until then a cross-language parser would be duplication, not sharing.

### What gets deleted

- The `choices`/`extras` split in models, envelope, executor, and poller.
- `plan_gate_choice_ids` redundancy: tale's `approve/run/tale/commit` choice set and the parallel
  `commit_plan`/`run_coder` extras.
- Telegram's `render_plan_gate_keyboard` presets, `_neutral_plan_choices` remote-id allowlist, the legacy request-JSON
  keyboard path, the "Tale"-button-means-approve compatibility mapping, and the hardcoded "Approve with selected
  add-ons" string.
- The per-kind mobile choice enums (`PlanActionChoiceWire` variants `Run`, `Tale`-adjacent mappings,
  `CustomGateChoiceWire` extras) in favor of one generic gate wire.
- The parts of `PLAN_APPROVAL_CHOICE_RECORDS` that exist only to name boolean combinations.

## Gate option query language and envelope v2

Implement the foundation in `src/sase/notification_gates/`:

- A small `query.py` module: tokenizer + parser for the grammar above, producing normalized branches with positional,
  human-readable errors. Model the diagnostics style on `src/sase/ace/query/parser.py` without reusing its
  ChangeSpec-specific machinery.
- Rework `models.py`: `GateOption` (id, label, icon, command, default_selected, feedback, schemas) replaces
  `GateChoice`/`GateExtra`; add `GateGroup` (member ids, label, icon) and branch normalization on `GateSpec`.
  `GateSpec.from_mapping` accepts only `schema_version: 2` with `query` + `options` (+ optional `groups`) and rejects v1
  shapes with an error that shows the expected schema.
- `service.py`: validate query/options/groups cross-references, embed `query`, `options`, `groups`, and `branches` in
  the hashed envelope.
- `executor.py`: `execute_gate_selection(bundle, selected_option_ids, feedback)` validates the non-empty single-branch
  subset rule, enforces the strongest-selected feedback mode, runs selected commands in query order, and writes
  `selected_option_ids` into `response.json`.
- `poller.py` / wait projection: surface `selected_option_ids` and drop `choice_id`/`selected_extra_ids`.
- Update `registry.py` adapter validation hooks to validate against branch structure rather than fixed choice tuples
  (the per-kind expected queries land in the producers phase).

Cover the parser (precedence, parens, duplicate ids, unknown ids, empty groups), envelope round-trip and hashing, subset
validation, execution order, and feedback-mode resolution in `tests/test_notification_gates.py` and a new parser test
module.

## First-class sase gate CLI

- Add the `gate` command group in `src/sase/main/parser_commands.py` (or a new `parser_gate.py` if that matches the
  surrounding layout) with `create` and `wait` subcommands; handlers adapt the existing `notify_handler` / `cli_wait`
  logic.
- `sase gate wait` keeps `--id/-i`, `--kind/-k`, `--timeout/-t`, `--json/-j` and the
  `answered=0 / cancelled=3 / timeout=4` exit contract, but its JSON projection uses the v2 resolution fields.
- Remove the `--gate` flag from `notify create` and remove `notify wait`; privileged gate actions remain impossible to
  create through raw notifications. Update `sase notify` group help accordingly.
- Follow the CLI rules memory (read it via the long-term memory procedure): alphabetized listings, short aliases,
  colored and scannable help.
- Update CLI tests (`tests/test_plan_approve_cli.py` and the notify/gate CLI coverage) to the new command group.

## Migrate gate producers to option queries

Rebuild every in-repo gate producer on envelope v2:

- **Tale plans** (`src/sase/plan_gate.py`): query `(approve AND commit) OR reject OR feedback`.
  - `approve` — label `Approve`, icon `✅`, default-selected; its command carries today's run-coder approval semantics.
  - `commit` — label `Commit plan file to the plans sidecar`, icon `💾`, default-selected; its command carries today's
    commit-plan semantics.
  - `reject` — label `Reject`, icon `❌`; `feedback` — label `Send Feedback`, icon `💬`, feedback mode `required`.
  - Group submit configured explicitly: label `Approve`, icon `✅`.
- **Epic plans**: query `approve OR reject OR feedback`, with `approve` keeping today's epic side effects (commit to the
  plans sidecar, launch beads). The standalone `epic` choice id disappears in favor of the uniform `approve`.
- **Launch approval and HITL** gates: express their existing simple choice sets as singleton-branch queries so the whole
  system speaks one language. The question-gate kind keeps its specialized multi-question UI; only its envelope moves to
  v2.
- **Runner protocol**: derive `{action, commit_plan, run_coder}` in `plan_approval_actions.py` from the selected set —
  `run_coder` iff `approve` selected, `commit_plan` iff `commit` selected, `action: "approve"` for any resolution of the
  approve branch — so `{approve, commit}` reproduces today's "Tale", `{approve}` today's "Approve"/"Run", and `{commit}`
  today's "Commit". Rejection and feedback are unchanged. Collapse `PLAN_APPROVAL_CHOICE_RECORDS` accordingly: status
  labels and consequence text become functions of the selected set rather than a six-row registry.
- Keep tier validation strict in `registry.py`: a tale plan gate must carry exactly the tale query shape, an epic gate
  the epic shape.
- Preserve the auto-approval path (`tests/test_plan_auto_approval.py`) by expressing auto resolutions as selected-option
  sets.

Update `tests/test_plan_gates.py`, `tests/test_plan_approval_actions.py`, `tests/test_plan_approval_choices.py`,
`tests/test_plan_approval_responses.py`, `tests/test_epic_approval.py`, `tests/test_launch_approval.py`, and
`tests/test_user_question_gates.py`.

## ACE TUI gates on the branch model

- Rework the plan-approval and custom-gate modals under `src/sase/ace/tui/` to render from branches with the shared
  rendering contract: toggles for group members, a submit control per group, plain buttons for singleton branches.
- Simplify keybindings to match (toggle selection, submit, activate singleton); update `src/sase/default_config.yml` for
  any keymap changes per the gotchas memory.
- Read the TUI performance memory via the long-term memory procedure before touching modal rendering or refresh paths.
- Update `tests/ace/tui/test_notification_plan_gate.py`, `test_notification_custom_gate.py`,
  `test_custom_gate_modal.py`, and `test_gate_debug_modal.py`; regenerate affected PNG snapshots with
  `--sase-update-visual-snapshots` and verify them intentionally.

## Mobile wire and gateway migration

This phase changes the sase-core repo; open it with the `/sase_repo` skill and work only in the path it prints. In
`crates/sase_core/src/notifications/`:

- Replace `CustomGateChoiceWire`/`CustomGateExtraWire` and the per-kind action choice enums (`PlanActionChoiceWire`,
  launch/HITL equivalents) with a generic gate wire: branches of options (id, label, icon, default_selected, feedback)
  plus per-group submit metadata, read from the v2 envelope.
- Mobile action submissions carry `selected_option_ids` (+ feedback); the gateway forwards them to the Python bridge
  unchanged. Plan-specific response assembly moves fully behind the bridge (producers phase), so
  `insert_plan_options`-style wire logic disappears.
- Update the mobile contract snapshot fixtures and `gate_action_string_mapping` tests in `mobile.rs`, then update this
  repo's Python bridge (`src/sase/integrations/_mobile_notification_actions.py`, `_mobile_notification_models.py`) and
  `docs/mobile_gateway.md` to the new contract.
- Run the sase-core test suite and this repo's mobile gateway tests (`tests/test_mobile_gateway.py`).

## Telegram unified gate keyboards

This phase changes the sase-telegram repo; open it with the `/sase_repo` skill and work only in the path it prints.

- Replace `render_custom_gate_keyboard` and `render_plan_gate_keyboard` with one branch-driven renderer implementing the
  rendering contract, including single-group inline expansion and multi-group collapse/expand.
- `gate_flow.py` progress state becomes `selected_option_ids` (+ expanded group); keep the compact
  `c<index>`/`x<index>`-style callback tokens so the 64-byte callback limit still holds with a generic
  branch/option/submit token scheme.
- `inbound.py` resolves every gate kind through the same selection path (`execute_gate_selection` via the shared
  executor); delete the legacy request-JSON keyboard, the remote-id allowlist, the "Tale"→approve compatibility mapping,
  and the per-kind callback branches for plan choices.
- Update `tests/test_gate_flow.py`, `tests/test_custom_gates.py`, `tests/test_inbound.py`, `tests/test_formatting.py`,
  and `tests/test_callback_data.py`, including a rendering test that pins the five-control tale layout shown in the
  design.

## Skill and documentation rewrite

- Rewrite `src/sase/xprompts/skills/sase_gate.md` around the v2 interface: design guidance for choosing a query,
  configuring option labels/icons, group submit buttons, and feedback modes; a compact worked example
  (`(restart AND verify) OR reject` reads naturally); and the `sase gate create` / `sase gate wait` workflow with the
  `selected_option_ids` result contract. The trust rules (no polling bundle files, no running bundle commands by hand,
  no `auto` for custom gates) carry over verbatim.
- Follow the generated-skills memory: run `sase skill init --force`, then deploy the regenerated skill files via
  `chezmoi apply` (the chezmoi repo must be opened through `/sase_repo` if its files need inspection).
- Sweep remaining references to `notify create --gate`, `notify wait`, `choice_id`, and `selected_extra_ids` in docs and
  skill text.

## End-to-end gate query exercises

Exercise the feature as a user would, without mocking the seams:

- Author a custom gate request with the query `(restart AND verify) OR reject`, create it with `sase gate create`,
  resolve it through the executor with a partial selection, and assert the `sase gate wait --json` projection (`status`,
  `selected_option_ids`, exit code).
- Create a tale plan gate from a real tale plan file and assert the envelope's branches, the configured `✅ Approve`
  group submit, and the derived runner protocol for the `{approve}`-only and `{commit}`-only selections.
- Render the tale gate through the Telegram keyboard builder (via the sase-telegram checkout opened with `/sase_repo`)
  and assert the five-control layout; project it through the mobile wire and assert the branch structure matches.
- Confirm creation rejects a v1-shaped request with the expected guidance.

## Testing strategy

Each phase lands its own unit coverage as described in its section; the epic-wide invariants are: (1) the envelope
branch structure is the only source of truth any renderer consumes, (2) the executor never runs a selection that is not
a non-empty subset of one branch, and (3) the tale plan gate's five-control layout is pinned by tests in both Telegram
and TUI form. Run `just check` in each touched repo before finishing a phase (`just install` first in fresh workspaces).

## Risks and migration

- **In-flight gates across the upgrade.** v1 bundles pending when v2 deploys will fail verification-time loading;
  surfaces already degrade gracefully ("Controls unavailable"). Mitigation: answer or cancel pending gates before
  deploying; re-create any that were lost.
- **Coordinated three-repo rollout.** sase must land before sase-core and sase-telegram consume the v2 contract; the
  phase dependency order encodes this. Keep the envelope change and its consumers within one release window so no
  surface reads a contract it does not understand.
- **Semantic shift: no mandatory group member.** Any non-empty subset of an AND group is submittable, so "commit without
  approve" becomes expressible — intentionally, replacing today's standalone `Commit` choice. The runner protocol
  mapping table in the producers phase is the contract that keeps this well-defined.
- **Telegram callback budget.** Generic tokens must stay within 64 bytes; the compact index scheme already in place is
  retained and tested.
