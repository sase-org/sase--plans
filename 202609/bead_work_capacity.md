---
tier: epic
title: Add weighted queue capacity to bead work and epic approval
goal: Let users set the weighted-load threshold for every agent launched for an epic
  through sase bead work or its approval gate, and name the queue directive argument
  capacity consistently.
phases:
- id: capacity_contract
  title: Establish the weighted capacity directive contract
  size: medium
  depends_on: []
  description: 'capacity_contract: rename the directive field in Rust and its Python
    adapters, enforce weighted-load thresholds, update queue presentation, and test
    the binding boundary.'
- id: bead_capacity
  title: Carry capacity through epic work and launch handoffs
  size: medium
  depends_on:
  - capacity_contract
  description: 'bead_capacity: add -c/--capacity, move --cl-name to -C, and preserve
    capacity across all epic rendering, resume, and launch paths.'
- id: epic_gate_capacity
  title: Add capacity to epic approval controls
  size: medium
  depends_on:
  - bead_capacity
  description: 'epic_gate_capacity: expose capacity in gate schemas and the custom
    approval modal, retain it through response translation, and pass it into the epic
    launcher.'
- id: capacity_docs_integration
  title: Document and verify the complete capacity workflow
  size: small
  depends_on:
  - epic_gate_capacity
  description: 'capacity_docs_integration: synchronize CLI and directive documentation,
    reference memory and skill source, and verify the complete gate-to-admission path.'
proposed_by: bbugyi200.athena.0jl
create_time: 2026-09-11 13:40:26
status: wip
bead_id: sase-zp
---

- **PROMPT:** [prompts/202609/bead_work_capacity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/bead_work_capacity.md)
- **BEAD:** [sase-zp](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zp/README.md)

# Bead work queue capacity

## Outcome and scope

`sase bead work epic-a epic-b -c 3` must supply capacity `3` to every newly launched
phase and land agent for both epics. `-C NAME` and `--cl-name NAME` retain the existing
completion-notification behavior and plan-file restriction. The epic approval
notification must expose the same optional capacity control.

An epic is appropriate because the change crosses the Rust scheduling and editor
contract, Python launch orchestration, and durable approval/UI protocol. The four phases
form an explicit chain so each worker can use the preceding phase's tested contract.
Each phase is bounded direct implementation work. The epic land agent owns combined-tree
verification and completion.

The semantics are precise:

- `%queue:N`, `%q:N`, `%queue(N)`, and `%queue(capacity=N)` set the same optional
  capacity field. Preserve the existing nonnegative integer domain, `0..4294967295`;
  fractional agent weights do not require fractional thresholds.
- A capacity of `N` requires the already occupied weighted load to be at most `N` before
  admission. Use the scheduler's existing machine-wide claim accounting, including its
  family/parallel-member ownership rules. Do not sum raw UI rows.
- The candidate's weight is excluded from that explicit pre-admission threshold. The
  separate global budget must still accommodate occupied load plus the candidate's
  weight. Capacity never overrides `max_running_agents`.
- For example, four independent claims of weight `0.25` satisfy capacity `1`; one claim
  of weight `2` does not. Capacity `0` waits for a true drain, including when a live
  claim has a very small positive weight.
- Omission means no additional threshold is authored. Preserve existing default queue
  behavior, priority/FIFO ordering, deference, and weight inheritance.
- The CLI option applies to epic bead IDs and epic Markdown plan targets. An explicit
  capacity on a standalone task target is an actionable error, following existing
  ordered multi-target semantics: earlier successful targets stand and processing stops
  at that task. Do not silently ignore the option.
- This is an invocation control, not a new plan-frontmatter property, saved bead
  setting, global configuration option, or retuning of agents already running.

## Findings and repository access

`src/sase/main/parser_bead_lifecycle.py` assigns `-c` to `--cl-name` today.
`src/sase/bead/cli_work_entry.py` dispatches targets in order; `cli_work_from_plan.py`
and `cli_work_from_plan_resume.py` handle creation/resume; `cli_work_handler.py` renders
previews and selected launches through `src/sase/bead/work.py::render_multi_prompt`.

Unlike `extra_waits`, which are emitted only on unblocked segments, capacity must be
emitted on every selected segment. The shipped `bd/land_epic` xprompt in
`src/sase/default_config.yml` already contributes `%q(w=2.0)`, which must compose with
the new capacity field without losing the land agent's weight.

The queue parser/formatter and admission policy belong to `sase-core`. Its
`crates/sase_core/src/queue_directive.rs` exposes `QueueFieldsWire.runners`;
`runner_capacity.rs` already totals weighted capacity for the global budget but still
compares `occupied_lanes` to the explicit `wait_runners` threshold. A pure terminology
change would therefore leave the requested behavior incorrect.

Use `/sase_repo` before accessing the core repository. Try the configured `sase-core`
reference; if unavailable, use
`sase repo open gh:sase-org/sase-core -r "Implement epic queue capacity"` and use only
the returned path. The external reference was needed during exploration. Read that
checkout's `AGENTS.md`. Shared parsing, validation, and admission policy remain in Rust;
Python is an adapter and frontend/orchestration layer.

## Capacity contract

Owner: `capacity_contract`.

1. Rename the canonical queue field and its parser/formatter parameters to `capacity` in
   `crates/sase_core/src/queue_directive.rs`. Update positional interpretation, named
   keyword, error codes/messages, duplicate detection, and canonical output such as
   `%queue(capacity=3, priority=20, weight=2)`. Preserve `%q`, colon/parenthesized
   syntax, and priority/weight aliases. `%q(3, capacity=3)` and repeated capacity fields
   remain duplicate errors. Reject the obsolete authored `runners=` keyword with a
   migration message naming `capacity=`. Update retired `%wait(runners=...)` guidance to
   recommend `%queue(capacity=...)`. Do not introduce an unflagged compatibility parser.
2. Update typed launch extraction/reconstruction in
   `crates/sase_core/src/agent_launch/{mod,admission}.rs`, editor metadata and
   completion in `crates/sase_core/src/editor/{directive,wire}.rs`, and PyO3
   bindings/tests in `crates/sase_core_py/src/lib.rs`. All generated examples and
   completions must name capacity and describe weighted load.
3. In `runner_capacity.rs`, replace the explicit lane-count condition with the weighted
   pre-admission condition. Reuse compensated sums and the existing floating-point
   comparison policy, with an explicit true-drain treatment of zero. Preserve
   invalid-snapshot fail-closed behavior, global-weight checks, claim reuse/inheritance,
   parked classification, and priority/FIFO ordering. Replace count-based shortfall and
   blocker output for this condition with capacity-based values and terminology,
   including fractional shortfalls; update any wire schema versions/fixtures affected by
   structural changes.
4. Update Python's thin contract consumers atomically with the Rust change:
   `src/sase/xprompt/queue_directive.py`, `_directive_extract.py`,
   `_directive_edit_wait.py`, `_directive_scan.py`, their exported callers, and
   `src/sase/axe/chop_proposal_models.py`. Public directive-edit arguments and
   queue-specific model fields should use capacity. Provide a small reusable Python
   adapter for validating a capacity value through Rust, usable by the CLI and approval
   code. Reject booleans and invalid numeric input without coercing them to a valid
   threshold.
5. Existing persisted launch/wait metadata such as `wait_runners` can retain its storage
   spelling, with a clearly documented adapter mapping to capacity. Do not undertake a
   wholesale history/storage rename. Previously serialized integer thresholds must still
   load and reconstruct canonical capacity syntax; do not bulk-edit archived prompts.
   Truly count-based runner statistics retain their count terminology. The obsolete
   authored keyword is the intentional public syntax migration, not an alias that
   quietly retains count semantics.
6. Update queue-condition labels, editing controls, and readers of changed blocker data
   in the existing wait modal/actions, `agent_runner_slots.py`, and prompt queue/wait
   panels. A weighted-load restriction must not be described as a count of running
   agents. Keep these changes presentation-only and reuse the existing cached Rust
   projection; add no I/O or recomputation to render paths.

Verification: extend Rust queue/directive/editor/admission tests and PyO3 tests, plus
`tests/test_queue_directive.py`, prompt-edit/completion tests, runner-slot tests, and
focused queue-presentation tests. Cover each accepted spelling, omission, zero, maximum
integer, negative/fractional/overflow/boolean rejection, duplicate fields, and
preservation of priority and explicit weight. Prove the
four-light-claims/one-heavy-claim examples, a tiny nonzero claim at capacity zero,
global-budget rejection despite a satisfied capacity threshold, and shared-family claim
accounting. Test typed and ordinary extraction paths.

## Bead work

Owner: `bead_capacity`.

1. Add `-c, --capacity N` with default `None` to the work parser and move only work's
   `--cl-name` short alias to `-C`. Keep options alphabetized by long name. Help must
   explain the maximum already running weighted load, the epic-only scope, omission, and
   zero. Do not change other commands' `-c` flags. Validate malformed/out-of-range input
   before store or launch side effects. Update the structural CLI completion snapshot
   with `just sync-completion-spec`.
2. Carry optional capacity through `cli_work_entry.py`, `cli_work_from_plan.py`,
   `cli_work_from_plan_resume.py`, and `cli_work_handler.py`, including dry-run, JSON,
   creation, checkpoint/resume, and launch-selection/re-render paths. Preserve the same
   value across all epic targets. Enforce the task-target restriction at the existing
   target validation boundary and report it consistently in text and JSON.
3. Extend `render_multi_prompt` to emit one capacity-bearing queue directive in every
   selected phase and land segment, including dependent phases, relaunch subsets, and
   land-only runs. Use the shared formatter. With capacity omitted, preserve the
   rendered queue behavior. With zero, emit it explicitly. Keep dependency waits, clan
   identity, model selection, and plan routing intact.
4. Preserve disjoint queue fields from expanded work xprompts, especially the default
   land weight `2.0`. Retain the shared duplicate-field validation if a custom xprompt
   also authors capacity; report that conflict clearly rather than silently selecting
   one threshold or removing its other queue fields. Exercise both planned bead
   launching and the generic CWD launch path.
5. Extend `src/sase/bead/epic_launch.py` entry points and `build_epic_launch_argv` to
   carry capacity through monitor, detached proc, leased-workspace, and retry paths.
   Include `--capacity N` in failure recovery commands, including those in
   `_plan_approval_epic.py` and plan-file error helpers. Audit `epic_launch_handoff.py`
   and retain the value in any durable payload that reconstructs a launch; payloads for
   completed-launch publication alone need no unrelated field. Always distinguish `None`
   from zero.

Verification: extend the `tests/test_bead/test_cli_work_*`, `test_work_rendering*`,
`test_epic_launch*`, and `tests/test_launch_planned_bead_work.py` suites. Check
short/long options and `-C`, omitted and explicit zero, mixed target ordering and
errors, all phase/land segments, plan-file resume, skipped active agents, land-only
launches, JSON and dry-run without writes, weight/priority composition after xprompt
expansion, duplicate-capacity failure, and capacity-preserving retry argv. Mock launch
boundaries rather than starting real workers for these tests.

## Epic approval

Owner: `epic_gate_capacity`.

1. Add optional integer `capacity` to epic approve input and result schemas in
   `src/sase/plan_gate.py`, using the same bounds as the directive. Keep it out of tale,
   reject, and feedback schemas. Absence means default behavior; JSON zero is an
   explicit value. Validate before committing a plan, persisting a successful response,
   or claiming a launch.
2. Thread capacity through `_plan_gate_command.py`, `_plan_approval_protocol.py`,
   `_plan_gate_envelope.py`, `plan_approval_actions.py`, and
   `notification_gates/adapters.py` into `_plan_approval_epic.py::prepare_epic_launch`
   and its preflight/recovery paths. Cover both command-backed neutral gates and
   direct/legacy approval actions. Translation must retain capacity in the durable
   result rather than depend on ephemeral UI input. Automatic approval and older
   responses with no capacity retain omission behavior. Skip/reject/feedback must not
   launch work.
3. Add an epic-only Capacity control to the existing Custom Approval modal
   (`approve_options_modal.py`) alongside its Wait control. Display Default for omission
   and explain zero as a drain threshold. Support editing, clearing, cancellation,
   inline errors, and retaining capacity while other fields are edited. Use an available
   shortcut such as `c` within that modal, and update the applicable default keymap
   configuration if the binding is configurable. Hide/disable this control for tale
   actions and never submit a stale epic capacity after switching to a tale action.
4. Extend `plan_approval_results.py`, `plan_approval_decisions.py`,
   `plan_approval_modal.py`, and `ace/tui/actions/agents/_notification_plan_gate.py` to
   retain and submit the value. Add `capacity` to `HOST_COLLECTED_PROPERTIES` only with
   that concrete control in place, so the gate does not also show an unnecessary YAML
   input. Keep generic schema-driven CLI/mobile gate responders functional through the
   same schema and protocol. Do not add a separate notification-specific launch
   implementation or an unrelated `sase plan approve` option.

Verification: mirror the end-to-end coverage in `tests/test_plan_gate_wait.py` for
capacity, including command execution, response translation, adapter side effects, and
final launch argv. Extend epic approval/API tests and TUI modal tests for default, zero,
editing/clearing, canceled edits, invalid values, switching action/tier, and
compatibility with responses that omit capacity. Check exactly one launch is scheduled.
Invalid input must leave approval retryable without a terminal successful response or
launch side effect. Update and inspect the relevant epic gate/custom-approval visual
snapshots if changed.

## Documentation and integration

Owner: `capacity_docs_integration`.

1. Update `docs/beads.md` option tables, multi-target examples, and epic approval
   instructions for `-c/--capacity` and `-C/--cl-name`. Update `docs/xprompt.md`
   grammar, examples, completion documentation, and admission explanation to
   capacity/weighted-load terminology and the precise pre-admission/global-budget
   distinction. Document migration from authored `runners=` to `capacity=`.
2. The requested terminology correction includes `sase/memory/xprompts.md`. Use
   `/sase_memory_read` and `/sase_memory_write` before the edit, rewrite the stale
   directive table/paragraph in place, and run `sase memory init` afterward. Do not
   hand-edit generated instruction files or unrelated memory.
3. Update the stale queue example/description in
   `src/sase/xprompts/skills/sase_agents_status.md`. Follow the generated-skills memory:
   preview with `sase skill init --diff` or `--dry-run`; deploy generated skills only
   from the clean landed source through the normal host workflow. Do not edit managed
   provider skill copies from a phase checkout.
4. Add a focused integration test that takes a capacity from an epic gate response
   through launch argv, bead-work rendering, xprompt expansion, Rust extraction, and
   weighted admission. Assert phase and land capacity match and the land weight remains
   `2.0`. Add the omitted-capacity regression case.
5. Audit active source/docs/examples for obsolete queue `runners` references and stale
   work `-c/--cl-name` examples. Historical serialized keys, intentional migration-error
   tests, and actual runner counts are justified exceptions.

## Verification and delivery

Every implementation phase that changes tracked SASE files must read `lint_and_test.md`,
format its changes, and run `just check`. For core changes, run core's `just check` or
`./scripts/check.sh`, which includes workspace-wide Rust and PyO3 checks with Python
3.12 or newer; never substitute `cargo test -p sase_core` alone.

Build/install the changed binding for Python verification using the actual opened core
checkout, e.g. `SASE_CORE_DIR=<opened-core-path> just install`. Check that Python
imports that build. Respect release-plz ownership of Rust versions; do not hand-bump
crate versions or invent an unpublished dependency pin. Follow the existing SASE release
dependency-window reconciliation when the new core contract is published, and record the
minimum supporting core release for that process. Ship a matched Python/core contract;
no Python fallback to the old parser or count policy.

The land agent runs `just check-full` through `/sase_monitor` with the prescribed
TESTING/TESTED status pair, and the relevant visual suite when snapshots changed. It
verifies both repositories' final state and handles any dependency/release coordination
through the established host workflow. Finalizers own commits and publication. A phase
must not claim the feature complete on parser tests alone: the weighted admission
examples, every phase/land prompt, and the durable gate path must all pass.
