---
tier: epic
title: Epic phase sizes and parented child epics
goal: 'Epic plans must declare a per-phase size (small/medium/large) that sase plan
  validate enforces; medium and large phases launch with a plan-first prompt (#plan
  appended), large phases route to a new @smartest model alias, and epics proposed
  by phase or land agents automatically become child epic beads of the bead they are
  responsible for (foo-5.2 begets foo-5.2.1, the lander of foo-5 begets foo-5.4) with
  first-class sase bead show support for the resulting hierarchy.

  '
phases:
- id: core-plan-schema
  title: Core plan schema for phase size and parent_bead
  depends_on: []
  description: '''Core plan schema: required phase size and managed parent_bead''
    section: extend sase-core''s plan validator with a required per-phase size enum
    (authoring errors, launch-mode legacy downgrade) and a managed epic-level parent_bead
    field, updating schema specs, diagnostics, and Rust parity tests.'
- id: core-bead-model
  title: Core bead model for size, nested plans, and cascades
  depends_on: []
  description: '''Core bead model: phase size, nested plan beads, recursive cascades''
    section: add a size field to phase beads across the Rust wire/mutation/event/projection
    layers, expose it on work-plan phase assignments, make close/rm cascades recursive
    for nested plan children, and lock in wave-filter behavior with Rust tests.'
- id: plan-schema-python
  title: Python plan validation mirror
  depends_on:
  - core-plan-schema
  description: '''Python plan validation mirror and floor bump'' section: mirror size
    and parent_bead in the frozen Python dataclasses and rehydration, keep the validate
    command''s schema table and minimal example correct, bump the sase_core_rs dependency
    floor, and update the plan-schema reference docs.'
- id: bead-work-routing
  title: Size-aware launch routing and @smartest
  depends_on:
  - core-bead-model
  - plan-schema-python
  description: '''Size-aware launch routing and the @smartest alias'' section: copy
    authored sizes onto phase beads at epic creation, append #plan to medium/large
    phase segments, route large phases without an explicit model to a new @smartest
    builtin alias, add --size to bead create/update, and keep dry-run previews in
    parity.'
- id: parented-epics
  title: Automatic parent association for proposed epics
  depends_on:
  - core-bead-model
  - plan-schema-python
  description: '''Automatic parent association for proposed epics'' section: stamp
    parent_bead at sase plan propose time from the existing SASE_PHASE_BEAD_ID/SASE_EPIC_BEAD_ID
    env vars, create parented child epic beads with hierarchical IDs in the epic-launch
    paths, add a --parent override, and verify env inheritance for follow-up agents.'
- id: bead-show
  title: Bead show for sized phases and child epics
  depends_on:
  - core-bead-model
  description: '''sase bead show for sized phases and child epics'' section: render
    phase size, split an epic''s CHILDREN into phases versus child epics, show full
    parent lineage for nested beads, and reconcile the documented output with what
    the command actually prints.'
- id: guidance-docs
  title: Authoring guidance, skills, and docs
  depends_on:
  - bead-work-routing
  - parented-epics
  description: '''Authoring guidance, skills, and docs'' section: teach the sase_plan
    skill template the size property and its selection rules, regenerate deployed
    skills, and sweep sdd/bead/configuration docs for size semantics, the @smartest
    alias, and parented child epics.'
- id: smoke
  title: End-to-end smoke exercises
  depends_on:
  - bead-work-routing
  - parented-epics
  - bead-show
  - guidance-docs
  description: '''End-to-end smoke exercises'' section: drive the full feature in
    a scratch project — validation failure modes, dry-run segment and model assertions
    per size, child-epic naming from env vars, recursive cascades, and bead show output
    — and report any gaps.'
  model: haiku
create_time: 2026-07-19 21:09:04
status: done
bead_id: sase-7z
---

# Plan: Epic phase sizes and parented child epics

## Context

Everything below was verified against the current checkout and the sase-core linked repo. Paths starting with `crates/`
live in the sase-core linked repo (open it with the `/sase_repo` skill before reading or editing; per the Rust Core
Backend Boundary memory, shared domain behavior belongs there).

**Plan validation is authoritative in Rust.** `sase plan validate` (parser `src/sase/main/parser_plan.py:356`, handler
`src/sase/main/plan_validate_handler.py:23`) delegates through `src/sase/sdd/plan_validate.py:99` to the `sase_core_rs`
binding (`crates/sase_core_py/src/lib.rs:2156`) and into `crates/sase_core/src/plan/validate.rs:106`. The phase schema
is `PlanPhaseWire { id, title, depends_on, description, model }` (`validate.rs:75-82`) with the accepted-key allowlist
`PHASE_FIELDS` (`validate.rs:22-23`), per-field checks in `validate_phases` (`validate.rs:476`), and the expected-schema
table driven by `plan_frontmatter_schema` (`validate.rs:115`, phase specs at `validate.rs:169-203`). The Python side
(`ValidatedPlanPhase`, `src/sase/sdd/plan_validate.py:54-63`) is a rehydration mirror with zero validation logic.
Managed fields (`create_time`, `status`, `prompt`, `bead_id`) are accepted but never required. `sase bead work`
plan-file mode re-validates with the same schema (`src/sase/bead/cli_work_from_plan.py:56`), and the committed-plan
sweep (`just validate-committed-plans`) requires the full schema only for plans in `YYYYMM` directories at or after
`202608`.

**Epic launch pipeline.** Epic approval delegates to `sase bead work <plan-file>` → `create_and_launch_epic_from_plan`
(`src/sase/bead/epic_from_plan.py:33`): the epic bead is created with `design=plan_ref`, `tier=EPIC`, and today no
parent (`epic_from_plan.py:84-93`); phase beads are created in frontmatter order with `parent_id=epic.id` and
`model=phase_spec.model or ""` (`epic_from_plan.py:108-118`). Rust builds the wave plan (`build_epic_work_plan`,
`crates/sase_core/src/bead/work.rs:33-203`; phase assignment copies `phase.model` at `work.rs:178`) and
`render_multi_prompt` (`src/sase/bead/work.py:268-354`) emits one segment per phase: `%model:<bead model>` or the
`@phase_worker` fallback (`work.py:323-328`), unconditional `%auto` (`work.py:329`), `%w:` waits, then
`#<work_phase_xprompt>:<bead_id>` (`work.py:332`); the land segment is analogous (`work.py:335-352`) with lander-role
routing in `epic_land_model_directive_value` (`work.py:247-265`). Bare `%auto` resolves to auto-approving whatever plan
the agent proposes, and an auto-approved epic re-enters `create_and_launch_epic_from_plan` via `prepare_epic_launch`
(`src/sase/_plan_approval_epic.py:16`).

**The association env vars already exist.** `src/sase/bead/work.py:30-33` defines `SASE_BEAD_ID`, `SASE_EPIC_PLAN_REF`,
`SASE_EPIC_BEAD_ID` (every segment, including land), and `SASE_PHASE_BEAD_ID` (phase segments only), built per segment
by `epic_work_segment_env`/`_bead_env` (`work.py:378-450`) and merged into the child process environment at
`src/sase/agent/launch_spawn.py:270-278`.

**Epic beads with parents are already structurally supported.** `sase bead create --type "plan(<file>,<parent>)"` exists
(`src/sase/bead/cli_crud.py:30-59`), nothing in wire validation or the SQLite schema bars a plan bead from having a
parent (`crates/sase_core/src/bead/wire.rs:207-239`), and Rust `create_issue` allocates `"{parent_id}.{N}"` for any bead
created with a parent (`next_child_id`, `crates/sase_core/src/bead/mutation.rs:891-898`) — which reproduces the
requested naming exactly: phase `foo-5.2`'s child epic becomes `foo-5.2.1`, and a lander-created epic under `foo-5` with
phases 1-3 becomes `foo-5.4`. The gaps are behavioral: `sase bead close`/`rm` cascade only one level
(`mutation.rs:370-386`, `425-431`), orphaning a child epic's phases; `sase bead show`
(`src/sase/bead/cli_query.py:64-155`) lists child epics undistinguished among phases; and the wave planner correctly
filters an epic's children to phases only (`work.rs:66-88`) — child epics are launched by their own approval flow, not
recursively.

**`@smartest` does not exist yet.** Builtin/role aliases live in `src/sase/llm_provider/config.py:44-96` (names,
fallback chain, descriptions) with config surface `llm_provider.model_aliases.builtin`
(`src/sase/default_config.yml:428-457`) and schema (`src/sase/config/sase.schema.json`). The `#plan` xprompt is defined
inline at `src/sase/default_config.yml:652-655`, and the `/sase_plan` skill source is
`src/sase/xprompts/skills/sase_plan.md` (epic schema example and phase-property prose at lines 40-74).

**Propose-time stamping point.** `sase plan propose` (`src/sase/main/plan_propose_handler.py:15`) runs inside the agent
process (SASE_AGENT-guarded), validates, prettier-formats the plan in place, then archives it via `move_plan_to_sase` —
so it can read the env vars and persist the association into the plan file that `sase bead work` later consumes.

## Design

### Phase size semantics

Every epic phase must declare `size: small | medium | large`:

| Size     | Prompt                                                  | Model routing                                          |
| -------- | ------------------------------------------------------- | ------------------------------------------------------ |
| `small`  | unchanged (today's behavior)                            | explicit phase `model`, else `@phase_worker`           |
| `medium` | `#plan` appended after the `#work_phase_bead` reference | explicit phase `model`, else `@phase_worker`           |
| `large`  | same as medium                                          | explicit phase `model`, else `@smartest` (new builtin) |

An explicit phase `model:` always wins over the size-derived default, mirroring how an explicit epic `model:` beats the
`@epic_lander`/`@big_epic_lander` threshold rule. Because every phase segment already carries bare `%auto`, the plan a
medium/large phase agent authors is auto-approved and follows the authored tier's follow-up path.

`size` is copied onto the phase bead at epic creation (next to the existing model copy), so bead-ID relaunches and
retries reproduce the same prompt and routing without re-reading the plan file. A stored missing/empty size means
`small` everywhere.

Compatibility: the authoring gate (`sase plan validate`) errors on a missing or invalid size with stable diagnostic
codes. The launch path must keep pre-feature epics resumable: plan-file validation invoked by `sase bead work` treats a
missing size as `small` with a warning (the mechanism — a validation-mode parameter on the binding versus a call-site
severity remap — is the core phase's decision, but the rule itself belongs in Rust per the backend-boundary memory). The
committed-plan sweep picks the field up automatically as part of the full schema, which only applies to
`YYYYMM >= 202608` directories, so already-archived epics are unaffected.

### Automatic parent association

When an agent proposes an epic-tier plan, `sase plan propose` stamps a SASE-managed `parent_bead: <bead-id>` frontmatter
field before archiving, sourced from `SASE_PHASE_BEAD_ID` when set (phase agents), else `SASE_EPIC_BEAD_ID` (the land
agent), else no stamp (agents outside bead work keep today's top-level behavior). The epic-launch paths
(`cli_work_from_plan.py` and `prepare_epic_launch`) resolve `parent_bead` in the active bead store and create the epic
bead with that `parent_id`, so Rust's existing `next_child_id` yields the hierarchical name; the child's phases then
nest naturally (`foo-5.2.1.1`, ...), and the scheme recurses for free. A parent that cannot be resolved fails with a
remedy, following the stale-`bead_id` precedent, and `sase bead work` grows an explicit `--parent` override (with a way
to force top-level) for manual control. `parent_bead` is accepted (never required) on epics and warned about on tales,
matching the existing epic-only-field treatment.

Follow-up agents must keep the association: a medium phase agent that proposes a _tale_ hands off to a coder follow-up
whose commits and potential later epics should still attribute to the phase bead, so env-var inheritance across the
plan-family handoff needs a regression test (the spawn path copies the runner's environment, so this is expected to
already hold).

### Non-goals

- No recursive `sase bead work` into child epics — a child epic is launched by its own approval flow, and the parent's
  wave filter continues to exclude plan-type children (locked in by test).
- No reparenting command; `parent_id` remains create-time-only.
- No size on tales or on the land agent.

## Phase sections

### Core plan schema: required phase size and managed parent_bead

In the sase-core linked repo. Add `size` to `PHASE_FIELDS` (`validate.rs:22`), `PlanPhaseWire` (`validate.rs:75-82`),
the phase field specs in `plan_frontmatter_schema` (near `validate.rs:197`, with guidance strings alongside
`PHASE_MODEL_DESCRIPTION` at `validate.rs:25-26`), and `validate_phases` (`validate.rs:476`): required, enum
`small|medium|large`, with distinct diagnostics for missing versus invalid values. Provide the launch/consumption
downgrade path described in the Design section so `sase bead work` can accept legacy size-less epics as all-small
(warning severity). Add the managed epic-level `parent_bead` field: accepted and type-checked on epics, never required,
epic-only warning on tales, exposed on the normalized plan wire. Keep the JSON payload additive — avoid a
`PLAN_WIRE_SCHEMA_VERSION` bump if possible (the Python adapter hard-rejects other versions,
`src/sase/sdd/plan_validate.py:17,131-136`), so the already-released Python side keeps working until its mirror phase
lands. Update the parity suite (`crates/sase_core/tests/plan_validate_parity.rs`) and the exact-schema test
`schema_is_ordered_and_contains_exact_phase_guidance` (`validate.rs:1377`); run the crate checks (`just rust-check`).

### Core bead model: phase size, nested plan beads, recursive cascades

In the sase-core linked repo. Add an optional `size` field to the bead model: `IssueWire`
(`crates/sase_core/src/bead/wire.rs:125-198`) with validation that size is only legal on phase-type issues (the
`changespec`-is-plan-only rule at `wire.rs:207-239` is the pattern), create/update mutations (`mutation.rs:137-198` and
the update path), event codec/reducer, the `issues.jsonl` projection, the SQLite compatibility cache, and search's
indexed fields. Old event streams without size must reduce cleanly to "no size". Expose size on the epic work plan's
phase assignments (next to the model copy at `work.rs:178`) so the Python renderer can see it. Make `close_issues` and
`remove_issue` cascade recursively through plan-type children (`mutation.rs:370-386`, `425-431`, `868-878`) so closing
or removing an epic with nested child epics cannot orphan grandchild phases, and extend `sase bead doctor`'s orphan
check to nested plans. Add a regression test that `build_epic_work_plan` continues to exclude plan-type children from
waves (extending the existing `parent_does_not_change_epic_launch_tag` fixture family, `work.rs:231-233,337-346`).
Update the Python-facing binding payloads additively; run `just rust-check` and `just bead-perf-smoke`.

### Python plan validation mirror and floor bump

Mirror the new wire fields in this repo: `ValidatedPlanPhase.size` and the plan-level `parent_bead` in
`src/sase/sdd/plan_validate.py` (dataclasses at lines 27-63, rehydration at 152-199), confirm
`src/sase/main/plan_validate_render.py` renders the new schema rows, minimal-example output, and diagnostics without
special-casing, and thread the launch/consumption validation mode through `validate_plan_file` for later use by
`cli_work_from_plan`. Bump the `sase_core_rs` dependency floor to the release carrying both core phases (per the
established floor-bump process used by prior epics). Update the plan-schema reference documentation (`docs/sdd.md` "Plan
Frontmatter Schema and Validation" table and the phase-fields prose) to list `size` as required and `parent_bead` as
SASE-managed. Pytest coverage for rehydration, render, and both validation modes.

### Size-aware launch routing and the @smartest alias

All in this repo. Define the `smartest` builtin alias in `src/sase/llm_provider/config.py` (name constant, fallback
`@default`, description; follow the `phase_worker` pattern at lines 44-96), the commented example in
`src/sase/default_config.yml:428-457`, `src/sase/config/sase.schema.json`, and any alias-listing surfaces (doctor hint
text, model completion). Copy the authored phase `size` onto phase beads in `create_and_launch_epic_from_plan`
(`src/sase/bead/epic_from_plan.py:108-118`). In `render_multi_prompt` (`src/sase/bead/work.py:268-354`): append `#plan`
on its own line after the `#work_phase_bead:<bead_id>` reference for medium/large assignments, and change the model
selection (`work.py:323-328`) to explicit bead model → size-large `@smartest` → `@phase_worker`. `sase bead work`
consuming a plan file uses the launch validation mode so legacy size-less epics stay resumable. Add `--size` to
`sase bead create` and `sase bead update` (phase beads only; short aliases and alphabetical listing per the CLI-rules
memory). Keep the dry-run/approval preview (`src/sase/bead/cli_work_from_plan_render.py`) in parity with the emitted
prompt, including the routed model per size. Tests: segment rendering per size (small has no `#plan`; medium/large do),
model precedence (explicit model beats `@smartest`), bead-ID relaunch reproducing size behavior from stored beads, and
CLI flag validation.

### Automatic parent association for proposed epics

In `sase plan propose` (`src/sase/main/plan_propose_handler.py`, before the prettier/archive step at lines 93-104): when
the validated plan is epic-tier, stamp `parent_bead` from `SASE_PHASE_BEAD_ID`, falling back to `SASE_EPIC_BEAD_ID`; no
env vars → no stamp. In the epic-launch paths (`src/sase/bead/cli_work_from_plan.py`,
`src/sase/_plan_approval_epic.py:16`, `src/sase/bead/epic_from_plan.py:84-93`): resolve `parent_bead` against the active
bead store and pass `parent_id` into epic bead creation so `next_child_id` allocates `foo-5.2.1`-style IDs; an
unresolvable parent fails with a remedy message (stale-`bead_id` precedent at `docs/beads.md`'s plan-file mode); dry-run
output previews the parented ID. Add `-p/--parent` to `sase bead work` as an explicit override, including a
force-top-level form. Verify and regression-test env-var inheritance: the tale follow-up coder spawned after a phase
agent's auto-approved tale retains `SASE_PHASE_BEAD_ID`/`SASE_EPIC_BEAD_ID` (spawn merge at
`src/sase/agent/launch_spawn.py:270-278`), and a nested child epic's own phase agents get their own child-scoped values
so deeper nesting recurses. Tests: propose stamping (phase agent, land agent, no-env, tale plans left unstamped),
end-to-end parented creation via `prepare_epic_launch`, `--parent` override, and unresolvable-parent failure.

### sase bead show for sized phases and child epics

In `src/sase/bead/cli_query.py:64-155` (data via the Rust read facade, `get_epic_children` at
`crates/sase_core/src/bead/read.rs:78-83,249-256` already returns typed children). Render: a `Size:` value for phase
beads (alongside Model at lines 78-81); the CHILDREN section split into phases (with status and size) and child epics
(with tier and status) instead of one undistinguished list (`cli_query.py:92-98`); for any bead with ancestors, the full
lineage chain up to the root epic (e.g. `foo-5.2.1 ← phase foo-5.2 ← epic foo-5`) in the PARENT section
(`cli_query.py:82-91`); and sensible plan-file lines for child epics (they take the own-`design` branch at
`cli_query.py:139-140`). Keep any machine-readable output in parity, reconcile `docs/beads.md:221-224` (which documents
an "epic count" the command does not print) with the implemented output, and cover child-epic-under-phase,
child-epic-under-epic, and deep-nesting cases with CLI-level tests. If small read-side additions are needed (e.g. an
ancestor query), they follow the backend boundary and go into the core read module.

### Authoring guidance, skills, and docs

Update the `/sase_plan` skill source (`src/sase/xprompts/skills/sase_plan.md`): add `size` to every phase in the epic
example and extend the phase-property prose — every phase must declare a size; use `medium` when the phase is
potentially a lot of work and justifies its own plan file; use `large` when you suspect that plan file would itself be
sized large enough to be assigned an epic tier; use `small` otherwise; explicit `model` still wins over size-derived
routing. Regenerate and deploy skills (`sase skill init --force`, then `chezmoi apply` per the generated-skills memory).
Doc sweep: `docs/sdd.md` (size semantics and model-routing paragraph in the Model Field section, `parent_bead` in the
managed-fields list), `docs/beads.md` (size field and CLI flags, parented child epics and naming, recursive cascade
behavior, `bead work --parent`), configuration/alias docs for `@smartest`, and the glossary/memory touchpoints only if
the user approves memory edits (memory files require explicit user permission — flag, don't edit). Verify the `#epic`
and `#plan` xprompt texts in `src/sase/default_config.yml` still read coherently with plan-first phases.

### End-to-end smoke exercises

In a scratch bare-git project with an initialized bead store: author an epic with three phases sized small/medium/large;
confirm `sase plan validate` rejects a missing and an invalid size with the documented codes and prints the updated
expected schema; `sase bead work --dry-run` and a real launch preview showing the small segment without `#plan` and with
`@phase_worker`, the medium segment with `#plan`, and the large segment with `#plan` plus `@smartest` (and an
explicit-model phase overriding it); with `SASE_PHASE_BEAD_ID`/`SASE_EPIC_BEAD_ID` set, propose and launch a child epic
and confirm `foo.2.1`-style naming under a phase and next-sibling naming (`foo.4`) under the epic, plus a deeper nested
level; exercise recursive close/rm without orphans and `sase bead doctor` cleanliness; review `sase bead show` output
for the epic, a sized phase, and a nested child epic. Report any gaps as findings rather than fixing consequential code.

## Testing

Each phase carries its own unit/CLI tests as described above; Rust phases run `just rust-check` (and
`just bead-perf-smoke` for the bead model), Python phases run `just check`. The committed-plan sweep
(`just validate-committed-plans`) must stay green — pre-`202608` months keep the legacy tier-only check, so no existing
archived plan needs backfill. The smoke phase is observation-only.

## Risks

- **Rust/Python lockstep**: the binding payload changes must stay additive so the released `sase_core_rs` keeps working
  with current Python until the mirror phases land; the floor bump happens in the first consuming Python phase. A
  wire-version bump would break that ordering and should be avoided or coordinated deliberately.
- **Legacy epics**: an epic authored before this feature but re-archived into a `>=202608` month directory would need
  `size` backfilled to pass the SDD writer gate; the launch-mode downgrade keeps normal retries working.
- **Name/clan interactions**: a child epic's own launch creates clan `foo-5.2.1` whose agent names nest inside the
  parent epic's `foo-5` hood; force-reuse wipes for a relaunched parent only target its own open-phase numbers and
  `.land`, so no collision is expected — the smoke phase verifies this.
- **TUI surfaces**: bead-adjacent presentation (ACE bead pickers/views) may assume plan beads are roots; the bead-show
  phase should grep those surfaces and keep changes presentation-only per the backend boundary.
- **Env-var absence**: agents launched outside bead work have no association vars and keep today's top-level epic
  behavior by design.
