---
tier: epic
status: done
title: Optional wait argument for tale and epic plan approvals
goal: "Approving a tale or an epic can name agent and bead dependencies, and the work
  that approval starts stays held until every named dependency finishes.

  "
phases:
  - id: bead_work_wait
    title: Shared wait-spec parser and `sase bead work --wait`
    depends_on: []
    size: medium
    description: "bead_work_wait: add the shared `agent,bead=<id>` wait-spec parser and
      a `sase bead work -w/--wait` option that renders the extra waits onto every
      unblocked epic segment.

      "
  - id: tale_coder_wait
    title: Carry approval waits into the tale coder prompt
    depends_on: []
    size: small
    description: "tale_coder_wait: give `PlanApprovalResult` wait fields and stamp a
      canonical `%wait(...)` directive onto the approved tale's coder successor prompt.

      "
  - id: epic_launch_wait
    title: Thread a wait spec through the host-owned epic launch
    depends_on:
      - bead_work_wait
    size: small
    description: "epic_launch_wait: pass an optional wait spec from
      `prepare_epic_launch` through the epic launch monitor into the `sase bead work
      --wait` argv, including resume hints.

      "
  - id: gate_wait_input
    title: Accept `wait` on the plan gate approval options
    depends_on:
      - tale_coder_wait
      - epic_launch_wait
    size: medium
    description: "gate_wait_input: declare `wait` on the tale and epic approval option
      schemas, validate it in the gate command, and route the parsed result into the
      coder prompt and the epic launch.

      "
  - id: plan_approve_cli
    title: "`sase plan approve --wait`"
    depends_on:
      - gate_wait_input
    size: small
    description: "plan_approve_cli: add a `-w/--wait` option to `sase plan approve` that
      validates the spec before anything is mutated and forwards it to the approval
      executor.

      "
  - id: ace_approval_wait
    title: Wait field in the ACE approval modal
    depends_on:
      - gate_wait_input
    size: medium
    description:
      "ace_approval_wait: add an editable wait row to the ACE custom-approval modal and
      forward the spec through both the neutral and legacy ACE approval paths."
proposed_by: bbugyi200.athena.0ga
bead_id: sase-vs
create_time: 2026-09-09 19:49:49
---

- **PROMPT:**
  [prompts/202608/approval_wait_argument.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/approval_wait_argument.md)
- **BEAD:**
  [sase-vs](https://github.com/sase-org/sase--beads/blob/main/pages/sase-vs/README.md)

# Plan: Optional `wait` argument for tale and epic plan approvals

## Goal

A reviewer approving a plan often knows the work should not start yet — another agent is
still rewriting the same module, or a bead has to close first. Today the only recourse
is to approve and then immediately open the ACE wait modal on the agent that was just
launched, which races the agent's own start.

This epic adds one optional `wait` argument to the tale and epic approval options. Its
value is a comma-separated list where each entry is either a SASE agent name or
`bead=<bead_id>`:

```bash
sase gate answer -i <id> -k plan -o approve --set wait='sase-s7.2,bead=sase-64.3'
sase gate answer -i <id> -k epic_plan -o approve --set wait='sase-s7.2'
sase plan approve abcdef12 --kind tale --wait 'sase-s7.2,bead=sase-64.3'
```

The work the approval starts is then held until every named agent reaches a terminal
state and every named bead closes.

## Design decisions

### The spec grammar reuses the existing `%wait` vocabulary

`%wait(a, b)` and `%wait(bead=<id>)` are already the way SASE expresses agent and bead
dependencies, and `sase.xprompt.directive_edit` already owns the canonical
`PromptWaitDirective` payload plus its formatter. The new argument is a flat string form
of that same payload, so it parses into `PromptWaitDirective` and every downstream
consumer keeps using the directive machinery it already uses. Only agent names and
`bead=` entries are accepted; `time=`, `runners=`, `priority=`, `unit=`, and `proc=` are
deliberately rejected so the approval argument stays one narrow thing.

### The tale holds its coder agent; the epic holds its phase agents

For a tale, the approval's follow-up is a single coder agent whose prompt is already a
live xprompt carrying `%model:` and the workspace ref (the plan gate shell's follow-up
policy sets `raw_prompt: True`). Stamping `%wait(...)` onto that prompt is the whole
change.

For an epic, the approval starts a **monitor** that runs `sase bead work <plan>`, which
creates the epic bead graph and launches the phase agents. The user's request named the
monitor as the thing that should wait. This plan deliberately holds the **launched phase
agents** instead, because:

- Monitors have no wait or hold facility today. `sase monitor start` takes no dependency
  arguments and the supervisor execs its command immediately, so holding the monitor
  means new blocking machinery in the monitor engine.
- Holding the monitor hides the epic completely until the dependency clears: no beads,
  no agent rows, nothing in `sase bead show`. Holding the phase agents creates the bead
  graph right away and shows every phase as a `WAITING` row, which is the state the rest
  of SASE already knows how to display, adjust (the ACE wait modal), and release
  (`wait_watch`).
- The epic's land agent already waits on every phase, so it inherits the delay for free.

The observable difference is that the epic's beads and its plan snapshot are created at
approval time rather than after the dependency clears. That is the better default: the
epic becomes visible and reviewable immediately, and no code is written until the
dependency finishes. This deviation is called out here rather than silently made.

### Waits land on unblocked segments only

`render_multi_prompt` already emits `%w:<names>` for intra-epic ordering. The extra
approval waits are appended only to segments whose intra-epic `waits_on` is empty (the
root-wave phases, and the land segment when no phases are launched). Every other segment
already waits transitively on a root segment, so repeating the dependency there would
only add redundant `WAITING` labels and polling.

Naming an agent that has already finished is safe: `initial_dependencies_resolved` in
`src/sase/axe/run_agent_wait.py` releases a waiter whose dependencies are already
terminal, so a stale name costs one resolution pass, not a hang.

### Phase ordering avoids a half-wired argument

The consumers land before the user-facing argument. `bead_work_wait` and
`tale_coder_wait` are independently useful on their own (`sase bead work --wait` is a
real CLI feature; the `PlanApprovalResult` fields stay empty until something sets them),
and `gate_wait_input` is the first phase where a reviewer can type `wait=` — by which
point both consumers exist. No landed phase exposes an argument that is silently
ignored, so no feature flag is required.

### Out of scope

- The mobile/Telegram approval bridge
  (`src/sase/integrations/_mobile_notification_actions.py`) keeps its current option
  set; it can adopt the argument later without changes here.
- Auto-approval (`%auto`, `%auto:tale`, `%auto:epic`) never supplies a wait. The
  in-process `%auto` path continues as a successor inside the same runner and does not
  re-enter the launch-time dependency barrier, so it is intentionally left alone.
- `time=`, `runners=`, and `priority=` wait keywords.

## Shared wait-spec parser and `sase bead work --wait`

Add `src/sase/wait_spec.py`, the one place that turns the flat argument into a
`PromptWaitDirective`:

- `WaitSpecError(ValueError)` — a deterministic parse failure carrying a message
  suitable for direct CLI and gate-command output.
- `parse_wait_spec(text: str) -> PromptWaitDirective` — splits on commas, strips
  surrounding whitespace from each entry, routes `bead=<id>` entries to `beads` and
  every other entry to `agents`, and deduplicates each list while preserving first-seen
  order. It raises `WaitSpecError` for an empty entry, a value containing whitespace, an
  empty `bead=`, and any other `key=value` form (naming the rejected keyword so
  `time=5m` produces a useful message rather than an agent named `time=5m`). Reuse the
  validation shape of `resolve_wait_bead_args` in
  `src/sase/xprompt/_directive_values.py` so both entry points reject the same values.
- `format_wait_spec(spec: PromptWaitDirective) -> str` — the canonical round-trip form
  (`agents first, then bead=<id> entries`, comma-joined), used for argv construction and
  for display.

Then render the extra waits:

- `render_multi_prompt` in `src/sase/bead/work.py` takes
  `extra_waits: PromptWaitDirective | None = None`. For each emitted phase segment whose
  `assignment.waits_on` is empty, and for the land segment when `plan.land_waits_on` is
  empty, append `%w:<comma-joined agents>` when agents are present and one
  `%w(bead=<id>)` line per bead, after the segment's existing wait lines and before its
  `#<xprompt>` line.
- Thread `extra_waits` through `launch_epic_bead_work` in
  `src/sase/bead/cli_work_handler.py` (both the real render and the `--dry-run` render,
  so the preview shows the waits) and through `work_from_plan_file` in
  `src/sase/bead/cli_work_from_plan.py`, including its own recursive timer-owning call
  and the nested `launch_created_epic` call.
- Register `-w/--wait SPEC` on `sase bead work` in
  `src/sase/main/parser_bead_lifecycle.py`, placed alphabetically between `--parent` and
  `--yes`. Help text: one line naming the comma-separated agent names and `bead=<id>`
  entries the launched phases wait for.
- Parse the option in `src/sase/bead/cli_work_entry.py`, exiting 2 with the
  `WaitSpecError` message before any bead or file mutation. The option applies to both
  plan-file targets and existing epic bead targets.

Docs: the `sase bead work` row in `docs/cli.md` and the epic launch section of
`docs/beads.md`.

Tests: parser unit tests (agents only, beads only, mixed, dedup, every rejection),
`render_multi_prompt` tests proving the waits appear on root and land-only segments and
not on dependent segments, and a CLI test that a bad spec exits 2 without launching.

## Carry approval waits into the tale coder prompt

- `PlanApprovalResult` in `src/sase/llm_provider/_plan_utils.py` gains
  `wait_agents: tuple[str, ...] = ()` and `wait_beads: tuple[str, ...] = ()`.
- `plan_approval_result_from_gate_response` reads `wait_agents` and `wait_beads` from
  the translated response, accepting only lists of non-empty strings and normalizing to
  tuples.
- `prepare_accepted_plan_successor` in `src/sase/axe/run_agent_exec_plan_accept.py`:
  when either tuple is non-empty, pass the fully composed successor prompt through
  `set_prompt_wait(prompt, PromptWaitDirective(agents=..., beads=...))` from
  `sase.xprompt.directive_edit` before handing it to `SuccessorRequest`. Using the
  existing rewrite helper rather than hand-assembling a prefix keeps the directive
  placement identical to every other prompt rewrite in the repo.
- Leave the empty case byte-identical: no wait means the prompt is not rewritten at all.

Tests: a golden successor-prompt test for agents-only, beads-only, and mixed waits, and
an unchanged-prompt test for the empty case. Confirm the rewritten prompt still parses
to the expected `%model`, workspace ref, and plan reference.

## Thread a wait spec through the host-owned epic launch

- `build_epic_launch_argv` in `src/sase/bead/epic_launch.py` takes
  `wait_spec: PromptWaitDirective | None = None` and appends
  `["--wait", format_wait_spec(wait_spec)]` when it is non-empty.
- `start_epic_launch_monitor` and `prepare_epic_launch`
  (`src/sase/_plan_approval_epic.py`) take and forward the same optional argument.
- Every resume hint built from `build_epic_launch_argv` — the unusable-store error, the
  monitor-start failure, `_raise_unclaimable_epic_launch`, and `finish_epic_launch` —
  keeps the wait, so a copy-pasted resume reproduces the reviewer's intent.
- Check `_active_epic_launch_for_plan` and `_logical_epic_launch_argv`: confirm a
  wait-carrying argv still matches an in-flight launch for the same plan, and adjust the
  comparison if it keys on the full argv rather than the plan path.

Default `None` keeps every existing caller byte-identical. Tests cover argv construction
with and without a spec, and that the monitor's logical command round-trips through the
code-swap guard wrapper unchanged.

## Accept `wait` on the plan gate approval options

This is the phase that makes the argument reachable.

- `_plan_input_schema` in `src/sase/plan_gate.py`: declare `"wait": {"type": "string"}`
  for the tale `approve` and `commit` options (both AND-group members receive the same
  input, matching how the coder fields are already declared) and for the epic `approve`
  option.
- `_plan_result_schema`: declare `wait_agents` and `wait_beads` as string arrays on the
  approving options, since `additionalProperties` is `False`.
- `execute_plan_gate_command` in `src/sase/_plan_gate_command.py`: read `wait` with
  `plan_gate_optional_text`, parse it with `parse_wait_spec`, and let a `WaitSpecError`
  fall into the existing `except Exception` handler so the command exits 2 with the
  parse message. Pass the parsed spec to `plan_response_json`.
- `src/sase/plan_approval_choices.py`: add `allow_wait_option: bool` to
  `_PlanApprovalChoiceRecord`, true for `approve`, `tale`, and `epic` and false for
  `commit`.
- `src/sase/_plan_approval_protocol.py`: `plan_response_json` and
  `plan_response_json_for_selection` take the parsed spec and emit `wait_agents` /
  `wait_beads` (omitting empty lists) when the selection runs a coder or is an epic
  approval. A commit-only tale selection drops the wait, exactly as it already drops the
  coder fields.
- `translate_plan_gate_response` in `src/sase/_plan_gate_envelope.py`: copy
  `wait_agents` and `wait_beads` from the `approve` option result into the translated
  response, so both tiers read one parsed value rather than re-parsing the raw input.
- `execute_plan_approval_response` in `src/sase/plan_approval_actions.py` takes
  `wait: str | None = None`, parses it once, puts the raw spec into `input_data` for the
  tale `approve`/`commit` options and the epic `approve` option on the neutral path, and
  passes the parsed spec to `plan_response_json` and `prepare_epic_launch` on the legacy
  path.
- `src/sase/notification_gates/adapters.py`: in the epic branch of `apply_side_effects`,
  read `wait_agents` / `wait_beads` from the merged `result` and pass the reconstructed
  `PromptWaitDirective` to `prepare_epic_launch`.

Compatibility: gate bundles are content-hashed, so a plan gate created before this phase
keeps its old schema and rejects `--set wait=`. That is correct — a pre-upgrade bundle
has no command that would honor the value — but the phase must confirm the rejection
message is the ordinary "no selected option accepts that input" error rather than a
crash, and say so in the phase's completion notes.

Tests: gate-spec golden updates for both tiers, `execute_plan_gate_command` accepting a
valid spec and exiting 2 on an invalid one, response-translation tests, an epic adapter
test asserting the argv handed to the monitor carries `--wait`, and a tale end-to-end
test asserting the coder successor prompt carries `%wait`.

## `sase plan approve --wait`

- Register `-w/--wait SPEC` on `sase plan approve` in `src/sase/main/parser_plan.py`,
  placed alphabetically after `-p/--prompt`, with help text naming the comma-separated
  agent names and `bead=<id>` entries. Add an epilog example alongside the existing
  `--prompt` one.
- `handle_plan_approve_command` and `_approve_plan_from_cli` in
  `src/sase/main/plan_approve_handler.py` forward the raw spec to
  `execute_plan_approval_response(wait=...)`.
- Validate the spec with `parse_wait_spec` before `resolve_pending_plan` runs, so a typo
  exits 2 with the parse message and no notification is consumed.

Docs: the `sase plan approve` rows in `docs/cli.md` and `docs/configuration.md`, and the
approval walkthrough in `docs/sdd.md`.

Tests: option parsing, forwarding, and the early-exit-on-bad-spec behavior.

## Wait field in the ACE approval modal

- `ApproveOptionsModal` in `src/sase/ace/tui/modals/approve_options_modal.py` gains a
  "Wait for:" row rendered under the model row, showing the current spec or `none`. Bind
  `w` to edit it. Reuse the existing agent-completion candidates
  (`visible_agent_completion_candidates`) for the editor's suggestions so the field
  offers the same names the wait modal does; a plain validated text input is acceptable
  if wiring completion into this modal proves invasive — the phase should record which
  it chose.
- Reject an invalid spec in the modal rather than at submission, using `parse_wait_spec`
  for the check.
- `ApproveOptionsResult` and `ApproveOptionsEditPrompt` carry the spec, as does the ACE
  `PlanApprovalResult` in `src/sase/ace/tui/modals/plan_approval_results.py`.
- `src/sase/ace/tui/actions/agents/_notification_plan_gate.py` passes `wait=` into
  `execute_plan_approval_response`; `_submit_legacy_epic_launch_task` in
  `src/sase/ace/tui/actions/agents/_notification_modals.py` passes the parsed spec into
  `prepare_epic_launch`.
- Update the modal footer key hints and `src/sase/default_config.yml` if the `w` binding
  needs a configured entry.

Docs: the plan approval section of `docs/ace.md`.

Tests: modal state tests for setting, clearing, and rejecting a spec; a decision-path
test asserting the spec reaches the executor. Refresh the ACE PNG snapshots with
`just test-visual --sase-update-visual-snapshots` if the modal layout shifts.

## Verification

Every phase runs `just check` before finishing. Run `just install` first if the
workspace clone's virtualenv is stale. `just check-full` runs through `/sase_monitor`
with the `TESTING`/`TESTED` status pair before the epic's combined tree lands, since the
change touches the gate schemas, the epic launch chain, and the ACE modals.
