---
tier: epic
title: Canonical sase-size memory and size-driven agent routing
goal: "One generated long-term memory note owns every sase-size instruction, tale plans
  carry a validated `size`, and coder follow-ups route through the size-specific
  phase-worker aliases instead of the retired coder alias bucket.

  "
phases:
  - id: memory-parent
    title: Robust long-note parent support
    depends_on: []
    size: medium
    description:
      "memory-parent: make a long memory note parented by another long note a
      first-class, validated arrangement across `sase memory init`, `sase memory read`,
      and memory proposals."
  - id: sizes-memory
    title: Generated sase_sizes.md memory note
    depends_on:
      - memory-parent
    size: medium
    description:
      "sizes-memory: add the generated `sase/memory/sase_sizes.md` long note parented by
      `sase/memory/sase_beads.md` and make it the only place sase-size guidance lives."
  - id: core-tale-size
    title: Required tale size in sase-core
    depends_on: []
    size: medium
    description:
      "core-tale-size: require and validate a tale plan's `size` frontmatter in the
      sase-core plan validator, expose it on the wire, and release it."
  - id: plan-size-adopt
    title: Adopt tale size in sase
    depends_on:
      - sizes-memory
      - core-tale-size
    size: medium
    description:
      "plan-size-adopt: raise the sase-core floor, plumb the validated tale `size`
      through the Python adapter, and point every remaining size instruction at the
      memory note."
  - id: coder-alias
    title: Retire the coder alias bucket
    depends_on:
      - plan-size-adopt
    size: large
    description:
      "coder-alias: delete the `coder` and `<provider>_coder` implicit aliases and route
      coder follow-ups through the phase-worker alias for the tale plan's size."
  - id: task-plan-handoff
    title: Verify plan handoff for large task beads
    depends_on: []
    size: small
    description:
      "task-plan-handoff: audit and regression-test that every task-bead launch path
      appends `#plan` for `large` and `xlarge` task beads."
proposed_by: bbugyi200.athena.wt
create_time: 2026-08-09 16:43:05
status: wip
---

- **PROMPT:**
  [prompts/202608/sase_sizes_memory.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/sase_sizes_memory.md)

# Plan: Canonical sase-size memory and size-driven agent routing

Sase sizes (`xsmall`, `small`, `medium`, `large`, `xlarge`) currently drive phase-bead
model routing, task-bead model routing, and the plan-first `#plan` handoff — but the
instructions that teach agents how to _choose_ a size are scattered across the Rust plan
validator's field descriptions, `src/sase/main/plan_explain.py`, the generated
`sase/memory/sase_beads.md` note, the `/sase_new_task` skill, and CLI `--help` text.
Tale plans carry no size at all, so a tale's coder follow-up is routed by planner
_provider_ (`@claude_coder` / `@codex_coder`) rather than by the work's scope.

This epic makes one generated long-term memory note the canonical, single source of
sase-size truth; gives tale plans a required, validated `size`; and retires the coder
alias bucket in favour of the existing size-specific phase-worker aliases.

**Authorization note.** The user explicitly requested the memory changes in this epic's
originating prompt, which satisfies the "Memory File Edits Require Explicit User
Permission" gotcha. Additionally, both `sase/memory/sase_beads.md` and the new
`sase/memory/sase_sizes.md` are _generated_ notes: the source of truth is the packaged
template under `src/sase/main/init_memory/templates/`, and the canonical note is
produced by `sase memory init`. Implementing agents edit the template, then run
`sase memory init` — they never hand-edit the canonical note.

## Robust long-note parent support

`MemoryNote.parent` already accepts a non-`AGENTS.md` value, and three pieces of the
machinery already understand it: `_children_by_parent_for_init` in
`src/sase/memory/inventory_reachability.py` walks children during init reachability,
`_render_managed_agents` in `src/sase/amd/_memory.py` restricts the `AGENTS.md` Tier 2
list to notes whose parent is `AGENTS.md`, and `render_children_section` in
`src/sase/memory/notes.py` appends a `## Children` section to `sase memory read` output
using the same `**\`path\`\*\*` + description shape as the Tier 2 list. No repository
note uses the arrangement yet, so the gaps below are latent.

Close them before anything depends on the arrangement:

1. **Validate parent targets.** Add parent validation to the `sase memory init` planning
   path (alongside `unreferenced_memory_files`, so the blocker surfaces with the other
   init blockers rather than as a confusing "unreferenced memory files" report). A
   note's `parent` must be either `AGENTS.md` or an existing memory note; a parent that
   names a missing file, a `type: short` note, or the note itself is a blocker with a
   message that names the offending note, the bad parent value, and the reason. Detect
   parent cycles (`a -> b -> a`, and longer) and report every note in the cycle in one
   blocker. Normalize through `canonical_memory_reference` first so `memory/x.md` and
   `sase/memory/x.md` are the same target.
2. **Support generated notes with a non-`AGENTS.md` parent in one init pass.** Today
   `generated_long_notes` is a `dict[str, str]` of relative path to description, and
   both `render_generated_beads_memory_content`
   (`src/sase/main/init_memory/root_rendering.py`) and the synthesized `MemoryNote` in
   `_render_managed_agents` hardcode `AGENTS_PARENT`. On a fresh root — where the
   generated note does not exist on disk yet — a child note would therefore be listed in
   `AGENTS.md` Tier 2 on the first pass and only settle on the second. Carry the parent
   alongside the description (a small frozen record keyed by relative path is cleaner
   than a second parallel mapping) so the first pass is already correct, and generalize
   the beads-specific renderer into one helper that renders any packaged generated long
   note from `(template filename, relative path, parent)`.
3. **Preserve an authored parent in memory proposals.** `_canonical_memory_content` in
   `src/sase/memory/proposals/write.py`'s sibling `review.py` hardcodes
   `parent=AGENTS_PARENT`, so an approved proposal can never land as a child note.
   Honour a `parent:` in the drafted body when it names a valid existing long note, and
   keep `AGENTS.md` as the default.
4. **Make the `## Children` section actionable.** `render_children_section` currently
   emits only the reference list. Prefix it with the same instruction the `AGENTS.md`
   Tier 2 section uses — that these are long-term child notes and that the agent MUST
   use the `/sase_memory_read` skill to read them — so an agent reading the parent knows
   both that the children exist and how it is allowed to read them.

Cover each of the four with tests: a child note reachable through its parent, a blocker
for each invalid parent shape (missing target, short-note target, self-parent, two- and
three-note cycles), a single-pass init assertion that a generated child note is absent
from `AGENTS.md` Tier 2 and present in the parent's `## Children`, and the rendered
`## Children` instruction text.

## Generated sase_sizes.md memory note

Add `src/sase/main/init_memory/templates/memory-sase-sizes.template.md` and wire it into
`sase memory init` exactly the way the bead memory note is wired: rendered for project
roots (`include_bead_memory=True` call sites in `src/sase/main/init_memory_handler.py` —
generalize that flag's name if it now gates two notes), retired via the byte-identical
check in `_retired_note_paths` for roots that stop managing it, and included in the
memory README render. Its frontmatter is `type: long` with
`parent: sase/memory/sase_beads.md`.

The note is the canonical source of truth for sase sizes. It must cover:

- **What a size is.** One scale (`xsmall`, `small`, `medium`, `large`, `xlarge`) shared
  by epic phases, task beads, and tale plans. Size selects the work model (through the
  `@<size>_phase_worker` aliases, unless an explicit `model` overrides it) and decides
  whether the agent plans before implementing.
- **Per-size meaning**, factored out of the existing prose in
  `src/sase/main/plan_explain.py` and the sase-core `PHASE_SIZE_DESCRIPTION`: `xsmall`
  for the very simplest tasks needing almost no reasoning, such as launching SASE agents
  purely to observe their output while testing a SASE agent feature; `small` for focused
  work implemented directly; `medium` for substantial work still implemented directly
  from its description; `large` for work that needs a separate planning handoff and may
  itself justify an epic plan; `xlarge` rarely, admitting the work is too large to plan
  effectively alone or deliberately deferring planning of part of a feature — chosen
  only when fairly confident the agent will itself author an epic plan.
- **The plan-first rule.** Only `large` and `xlarge` receive `#plan` and plan before
  implementing; `xsmall`, `small`, and `medium` implement directly.
- **Tale plan files.** A tale plan MUST declare `size: xsmall | small | medium`. A tale
  is by definition work a single follow-up coder agent implements directly, so `large`
  and `xlarge` are not valid tale sizes — work that big is an epic.
- **Authoring a plan is itself large or xlarge.** Creating a plan with `/sase_plan` is
  always associated, implicitly or explicitly, with `large` (the agent authors a `tale`)
  or `xlarge` (the agent authors an `epic`). That is why `large` and `xlarge` beads
  receive `#plan`: the size names the _handoff_, and the tale plan's own `size` then
  names the scope of the follow-up implementation.
- **Task-bead defaults.** When creating a new task bead, default to `large` unless
  either: the agent is very confident it has identified the precise root cause, in which
  case it uses `xsmall`, `small`, or `medium` and describes that root cause precisely in
  the bead description; or the agent is very certain the work is large enough to need an
  epic plan and multiple agents, in which case it uses `xlarge`.
- **Model routing.** The five `@<size>_phase_worker` aliases, and that an explicit
  `model` always wins over size-derived routing.

Keep the note tight — every token in an agent's context either helps or hurts.

Then remove the guidance this note replaces and reference it instead:

- `src/sase/main/init_memory/templates/memory-sase-beads.template.md`: drop the
  enumerated size list from the task-bead section and point at the child note (it is now
  rendered automatically in that note's own `## Children` section, so the pointer can be
  a single short sentence naming `/sase_memory_read`).
- `src/sase/xprompts/skills/sase_new_task.md`: replace the trailing size enumeration
  with a `sase memory read sase_sizes.md` step, and state the default-to-`large` rule at
  the point where the skill tells the agent to choose `--size`.
- `src/sase/main/parser_bead_lifecycle.py`: shorten the two `--size` help strings to
  name the memory note rather than restating routing behaviour.

Run `sase memory init` afterwards so `sase/memory/sase_sizes.md`,
`sase/memory/sase_beads.md`, `AGENTS.md`, the provider shims, and
`sase/memory/README.md` are all regenerated in the same commit. `/sase_new_task` is a
generated skill — follow `sase/memory/generated_skills.md` (via `/sase_memory_read`) for
its deploy step.

## Required tale size in sase-core

Open the sase-core repo with `/sase_repo` (`sase repo open sase-core`). All plan
frontmatter validation lives in `crates/sase_core/src/plan/validate.rs`.

- Add `size` as a **tale-only** top-level field: required, and one of `xsmall`, `small`,
  `medium`. Reject `large` and `xlarge` with a message that says a tale is single-agent
  work and that larger work belongs in an epic. Mirror the existing tale/epic key
  handling in `validate_top_level_keys`: introduce a `TALE_FIELDS` list so a top-level
  `size` on an epic produces an `epic-inert-field` warning symmetric with today's
  `tale-inert-field`, rather than an `unknown-key` error.
- Add `size: Option<String>` to `ValidatedPlanWire`, populated for tales.
- Honour `PlanValidationMode`: `Authoring` reports a missing tale `size` as an error;
  `Launch` reports it as a warning and normalizes to `medium` (the largest size a tale
  may declare, so a legacy sizeless tale keeps at least today's capability rather than
  silently downgrading its follow-up model). Match the existing `phase-size-missing` /
  `phase-size-invalid` code naming with `plan-size-missing` / `plan-size-invalid`.
- Add the `size` entry to `plan_frontmatter_schema` for the tale tier, with example
  `"medium"`.
- **Shorten the guidance prose in the field descriptions.** `PHASE_SIZE_DESCRIPTION` and
  `PHASE_MODEL_DESCRIPTION` currently carry the full per-size explanation; the new tale
  `size` description must not duplicate it. Reduce all three to a one-line statement of
  the accepted values plus a pointer telling the agent to read
  `sase/memory/sase_sizes.md` with the `/sase_memory_read` skill. Update
  `plan_validate_parity.rs` and the in-module tests that assert on those descriptions
  (including the test that asserts the alias names appear in `PHASE_SIZE_DESCRIPTION`).
- Bump `PLAN_WIRE_SCHEMA_VERSION` in `crates/sase_core/src/plan/wire.rs` to 3 and update
  every consumer and parity test in the repo.
- Update the existing tale fixtures across `crates/sase_core` (and `crates/sase_gateway`
  if it constructs tale documents) to carry a `size`.

Land the change and cut a sase-core release; the following phase raises sase's floor to
it.

## Adopt tale size in sase

- Raise the `sase-core-rs` window in `pyproject.toml` to the release from the previous
  phase, following the precedent of prior floor bumps.
- Bump `PLAN_WIRE_SCHEMA_VERSION` and add `size: str | None` to `_ValidatedPlan` in
  `src/sase/sdd/plan_validate.py`, populated in `_validated_plan_from_dict`.
- Rewrite `TALE_PLAN_EXPLANATION` and `EPIC_PLAN_EXPLANATION` in
  `src/sase/main/plan_explain.py`: the tale example gains `size: medium`, and every
  paragraph explaining what the sizes mean, which sizes get `#plan`, and how size
  selects a model is replaced by a short pointer instructing the agent to read
  `sase/memory/sase_sizes.md` with the `/sase_memory_read` skill. Leave
  `PLAN_HEADER_BLOCK_NOTE` and the phase-shape rules (unique slug IDs, dependency
  ordering, the `<id>: ` description prefix) in place — they are not size guidance.
- Update `src/sase/xprompts/skills/sase_plan.md`: add reading `sase_sizes.md` to the
  exploration step, require the tale frontmatter to declare `size`, and state that
  authoring a plan is itself `large` (tale) or `xlarge` (epic) work. Redeploy the
  generated skill per `sase/memory/generated_skills.md`.
- Refresh any tale plan fixtures under `tests/` that must now validate, and add tests
  for a valid tale size, a missing tale size, and a rejected `large` tale size.

Legacy tale plans already committed to the plan archive keep working: approval and
launch surfaces validate with `mode="launch"`, which warns and normalizes rather than
failing. Verify that specifically for `require_plan_approval_validation`
(`src/sase/plan_approval_actions.py`) and the `sase plan` handlers, and note in the
phase's completion which surfaces use authoring mode and would therefore reject a legacy
sizeless tale.

## Retire the coder alias bucket

Replace provider-derived coder routing with size-derived routing. The tale plan's `size`
now names the follow-up's scope, and the `@<size>_phase_worker` aliases already exist,
so the `coder` / `<provider>_coder` family is redundant.

Launch path:

- `src/sase/axe/run_agent_exec_plan_accept.py` — `_resolve_coder_alias_followup`
  currently emits `@<planner_provider>_coder`, falling back to `@coder`. Replace it with
  a resolver that reads the approved plan (`plan_result.plan_file`, or the committed SDD
  plan path when the plan was committed) and emits `@<size>_phase_worker` for the tale's
  validated size. Use launch-mode validation so a legacy sizeless tale normalizes
  instead of failing, and keep the existing precedence: an explicit approval-picker
  `coder_model` still wins, and a `%model` directive inside a custom coder prompt still
  supersedes both.
- `src/sase/ace/tui/modals/approve_options_modal.py` — `_contextual_coder_lane` shows
  the planner-provider coder alias in the approval modal. Show the size-derived
  phase-worker alias instead, resolved from the plan under approval.

Alias registry and surfaces:

- `src/sase/llm_provider/model_alias_defaults.yml` — remove the `coder` entry.
- `src/sase/llm_provider/model_alias_policy.py` — remove `CODER_MODEL_ALIAS_NAME`,
  `PROVIDER_CODER_ALIAS_SUFFIX`, and the `coder` element of
  `_ROLE_ALIAS_NAME_CONSTANTS`; update the module docstring and the block comment
  describing implicit aliases.
- `src/sase/llm_provider/model_alias_config.py` — remove
  `coder_model_alias_for_provider`, `is_provider_coder_alias`,
  `provider_coder_model_alias_names`, and the `provider_coder` branch of
  `model_alias_kind`; `special_model_alias_names` collapses to the role aliases.
- `src/sase/llm_provider/alias_view.py` — drop the `provider_coder` `AliasKind`, the
  `coders` bucket constants and spec, and the hidden-provider coder filtering;
  `BUILTIN_MODEL_ALIAS_BUCKET_NAMES` becomes just `phase_worker`.
- `src/sase/xprompt/model_completion.py` and `src/sase/ace/tui/model_alias_styles.py` —
  remove the coder alias from completions and the `provider_coder` style entry.
- `src/sase/doctor/checks_config_model_aliases.py` — add `coder` (and any configured
  `<provider>_coder` key) to `_RETIRED_BUILTIN_ALIAS_NAMES` with a migration message
  pointing at the `@<size>_phase_worker` aliases, following the existing `phase_worker`
  precedent in `src/sase/doctor/checks_config_common.py`, so an existing user config
  that sets them reports an actionable finding instead of an unknown-key error.
- `src/sase/default_config.yml` and `src/sase/config/sase.schema.json` — drop `coder`,
  `<provider>_coder`, and the `coders` bucket from the documented alias list, the
  commented examples, and the bucket descriptions.
- `docs/` — `configuration.md`, `llms.md`, `ace.md`, `cli.md`, `sdd.md`, `xprompt.md`,
  `integrations.md`, and `fakey.md` all mention coder aliases; update the ones that
  document the alias family. Leave the blog posts alone.

Run `just check-full` for this phase: the sweep touches the broadening set (config
schema, alias registry, TUI panels).

## Verify plan handoff for large task beads

`render_task_prompt` in `src/sase/bead/work.py` already appends `#plan` when
`phase_requires_plan(size)` is true (`large` or `xlarge`), so the behaviour the user
asked about most likely already exists. Confirm it rather than reimplement it:

- Trace every path that launches a task-bead agent — `sase bead work <task-id>`, the
  `TaskTriage` gate action, and the ACE TUI task-launch action — and confirm each
  renders its prompt through `render_task_prompt` rather than assembling a prompt of its
  own.
- Confirm `_phase_size` normalizes a missing legacy size to `small` (no `#plan`), and
  that `task_model_directive_value` and `phase_requires_plan` agree on the same
  normalized size.
- Add regression tests asserting the rendered task prompt contains `#plan` for `large`
  and `xlarge` and omits it for `xsmall`, `small`, `medium`, and a sizeless legacy task.

If a launch path is found that bypasses `render_task_prompt`, fix it to use the shared
renderer and say so in the phase's completion notes.

## Verification

Every phase runs `just install` first (workspace virtualenvs are ephemeral), then
`just check`. The `coder-alias` phase runs `just check-full`, as does the epic's land
agent before submitting the combined tree. The sase-core phase runs that repo's own
check target inside the checkout `/sase_repo` prepared.
