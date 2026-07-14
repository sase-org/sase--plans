---
create_time: 2026-07-14 12:43:46
status: done
prompt: 202607/prompts/plan_validate_command_1.md
bead_id: sase-61
tier: epic
---
# Plan: Agent-Facing `sase plan validate` + Structured Epic Frontmatter

## Context

Today a SASE plan file is free-form markdown until approval time. `sase plan propose`
(`src/sase/main/plan_propose_handler.py`) checks only that env vars are set and the file exists; the tier is chosen by
the human at approval (`sase plan approve --kind tale|epic`, TUI `t`/`E` keys) and stamped into frontmatter afterward
(`src/sase/plan_approval_actions.py` `_archive_plan_for_approval`, `src/sase/sdd/_write.py`). For epics, the runner then
spawns an LLM agent running the `bd/new_epic` xprompt (`src/sase/default_config.yml`, referenced from
`src/sase/axe/run_agent_exec_plan_accept.py` and `src/sase/agent_family/standard_plan_chain_definition.py`) which reads
the plan prose and shells out to `sase bead create` / `sase bead dep add` / `sase bead work` to build the epic bead,
phase beads, and dependency graph.

This has two costs:

1. **No validation loop for agents.** A planning agent has no way to check that its plan file is structurally sound
   before proposing it. Malformed frontmatter is silently tolerated (`src/sase/sdd/frontmatter.py` swallows YAML errors;
   `classify_plan_file` in `src/sase/sdd/plan_tiers.py` silently falls back to `tale`).
2. **Epic bead creation is non-deterministic.** The phases-to-beads translation is delegated to an LLM (`bd/new_epic`),
   so titles, descriptions, dependency edges, and per-phase models depend on how well that agent reads the plan prose.

This epic introduces an agent-facing `sase plan validate` command backed by a deterministic, tier-aware frontmatter
schema, teaches planning agents a validate → edit → revalidate loop, enforces validation at the proposal, epic-approval,
and committed-plan (CI/commit-path) boundaries, and makes epic frontmatter rich enough that SASE creates the epic bead,
phase beads, and dependencies programmatically and launches `sase bead work` automatically on epic approval — retiring
the `bd/new_epic` xprompt.

### Settled requirements (do not re-litigate during implementation)

- `sase plan validate` validates exactly one explicit `PLAN_FILE` argument. No inference from agent context; no bulk
  mode in v1.
- The caller must explicitly provide the tier; supported tiers are exactly `tale` and `epic`.
- V1 is deterministic schema validation only: syntax, frontmatter, tier-specific structural fields. No judgments about
  prose quality, feasibility, testing quality, risks, or completeness.
- The command is independent in purpose and UX from `sase validate` and `sase sdd validate`. Neither existing command
  becomes the user-facing surface for this validator.
- Every supported plan tier requires a top-level `goal` frontmatter property describing the outcome the plan is designed
  to achieve.
- Epic frontmatter must carry all structured data needed to programmatically create the parent epic bead and every
  ordered phase bead, including dependency wiring — as schema requirements, not quality checks.
- All validation problems are reported in one run; each diagnostic is actionable and identifies the file and location
  when possible; any failure exits nonzero.
- Validation is enforced at three boundaries: before a plan enters the proposal approval queue; during epic approval
  before notifications, bead creation, or PR setup; and in CI/commit-path checks for committed plan files.
- On epic approval, SASE creates the epic bead, phase beads, and dependencies from frontmatter and runs the
  `sase bead work` launch path automatically; `bd/new_epic` is retired.
- Each phase entry supports an optional `model` field whose schema description states that agents should only set it
  explicitly when the user's prompt requested it, or when the phase's agent does not do real consequential work (for
  example phases that exercise/test the feature itself).

## The Frontmatter Schema

### Authoritative representation

The authoritative schema is a **typed Rust model plus rule set in `sase-core`**
(`crates/sase_core/src/plan/validate.rs`, new), following the two existing Rust validator precedents:
`config/validate.rs` (`ConfigDiagnosticWire {severity, code, message, path}`) and `editor/frontmatter.rs` (LSP-shape
diagnostics with locations). This is required by the Rust core backend boundary: the same validation must behave
identically for the CLI, the TUI approval flow, the auto-approval runner path, CI sweeps, and the deterministic
bead-creation flow, and the parsed epic structure feeds bead creation whose data model already lives in Rust
(`crates/sase_core/src/bead/`). The Rust plan module already owns tolerant plan discovery (`plan/read.rs`:
`split_frontmatter`, `parse_frontmatter_map`, `plan_kind_from_tier`); validation is its strict sibling.

The Rust side also exposes a machine-readable **schema spec** per tier (ordered field records: name, type, required,
description, example) so Python renders the "expected schema" block and JSON consumers get it verbatim. One source of
truth; no hand-maintained parallel JSON Schema document. Field descriptions live in this spec — including the phase
`model` guidance text quoted below.

Two bindings in `crates/sase_core_py` (JSON-in/JSON-out, like `py_plan_search`):

- `plan_validate(content, tier)` → `{ok, diagnostics: [...], plan: {...}|null}` — `plan` is the normalized parsed
  structure when validation passes (consumed later by deterministic bead creation).
- `plan_frontmatter_schema(tier)` → the ordered field-spec list.

Watch the two-vocabulary hazard: plan-file tiers are `tale`/`epic` (`sdd/plan_tiers.py`, `plan/read.rs`); bead tiers are
`plan`/`epic` (`bead/wire.rs`). This feature is about the plan-file vocabulary; only the bead-creation phase maps
between them.

### Fields — both tiers

| Field   | Required | Rule                                                                                                 |
| ------- | -------- | ---------------------------------------------------------------------------------------------------- |
| `tier`  | yes      | `tale` or `epic`; must equal the tier the caller passed (mismatch is an error)                       |
| `goal`  | yes      | Non-empty string describing the outcome the plan is designed to achieve                              |
| `model` | no       | Non-empty string (same syntax `%model` accepts). Tale: coder follow-up model. Epic: land-agent model |

System-managed fields are accepted but never required: `create_time`, `status`, `prompt`, `bead_id`. Unknown top-level
keys are **errors** (typo protection), with one carve-out below.

Structural syntax checks (both tiers): file readable as UTF-8; frontmatter opens at byte 0 with `---`; closing `---`
marker present; YAML parses; YAML value is a mapping; non-empty markdown body after the frontmatter.

### Fields — epic only

| Field        | Required | Rule                                                               |
| ------------ | -------- | ------------------------------------------------------------------ |
| `title`      | yes      | Non-empty string; becomes the epic plan bead's title               |
| `phases`     | yes      | Non-empty list of phase mappings, in execution/creation order      |
| `changespec` | no       | Non-empty string; forwarded to the epic bead's ChangeSpec metadata |
| `bug_id`     | no       | Integer; requires `changespec`                                     |

Each `phases[]` entry:

| Field         | Required | Rule                                                                                                                                                                                                                                                                                                                                                                            |
| ------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`          | yes      | Non-empty string slug, unique across phases                                                                                                                                                                                                                                                                                                                                     |
| `title`       | yes      | Non-empty string; becomes the phase bead's title                                                                                                                                                                                                                                                                                                                                |
| `depends_on`  | yes      | List (may be `[]`) of phase `id`s. Every reference must resolve, no self/duplicate references, and references may only point to **earlier-listed** phases — so list order is always a valid topological order and matches bead child-ID allocation order                                                                                                                        |
| `description` | no       | String; becomes the phase bead's description (a deterministic pointer to the plan file and phase is generated when absent)                                                                                                                                                                                                                                                      |
| `model`       | no       | Non-empty string. Schema description text: _"Model for this phase's agent. Only set this explicitly when the user's prompt requested a specific model, or when this phase's agent does not do real consequential work (for example, a phase that exercises or tests the feature's own functionality). Otherwise omit it so the configured `@phase_worker` role alias applies."_ |

`depends_on` is required (even as `[]`) so agents consciously declare the graph instead of silently getting all-parallel
or all-sequential behavior. Unknown keys inside a phase entry are errors.

On a **tale**, the epic-only fields (`title`, `phases`, `changespec`, `bug_id`) are downgraded to _warnings_ (inert,
ignored) rather than errors, so a human downgrading an epic-authored plan to a tale at approval time does not create a
file that can never pass CI.

### Example epic frontmatter

```yaml
---
tier: epic
title: Workspace GC rewrite
goal: >
  Stale workspace checkouts are garbage-collected automatically without touching claimed workspaces, and the TUI shows
  reclaim progress.
model: opus # optional — land-agent model
changespec: ws_gc # optional
phases:
  - id: core
    title: GC planner and safety checks
    depends_on: []
  - id: cli
    title: sase workspace gc command
    depends_on: [core]
  - id: smoke
    title: End-to-end GC smoke exercises
    depends_on: [cli]
    model: haiku # allowed: this phase only exercises the feature
---
```

## CLI Design

```
sase plan validate PLAN_FILE -t {tale,epic}
```

- Registered as a `validate` subparser in the existing `plan` group (`src/sase/main/parser_plan.py`), alphabetically
  last; dispatched via `src/sase/main/plan_command_handler.py` to a new `src/sase/main/plan_validate_handler.py`.
- `PLAN_FILE` is a positional path (matching `sase plan propose`). The tier is a **required** `-t/--tier` option with
  choices `tale`/`epic` — required options are idiomatic here (`sase bead create -t/-T`), and `-t/--tier` matches the
  existing `sase plan list -t/--tier` spelling. Also `-j/--json` and `-q/--quiet` (print nothing on success), matching
  repo conventions.
- Hermetic: no `SASE_AGENT`/`SASE_ARTIFACTS_DIR`/project requirements — it validates any path.
- Follow the CLI rules memory: excellent `-h` output with examples, alphabetized subcommands/options, short aliases for
  every public long option, Rich-colored human output via `src/sase/output.py` (unlike the plain-text
  `sase sdd validate`).

### Diagnostics behavior

- **One pass, all problems**: the engine never fail-fasts; every diagnostic from a run is reported.
- Each diagnostic carries `severity` (`error`/`warning`), a kebab-case `code` (e.g. `frontmatter-unclosed`,
  `tier-mismatch`, `required-missing`, `unknown-key`, `phase-id-duplicate`, `dep-unknown`, `dep-forward`,
  `bug-id-without-changespec`, `tale-inert-field`), a `field_path` (e.g. `phases[2].depends_on[0]`), a human message,
  and a best-effort 1-based `line` number in the file (frontmatter YAML locations are resolvable; body/whole-file checks
  may omit it).
- Human output: `PLAN_FILE:LINE: severity [code] message` lines; **on failure, the expected tier-specific frontmatter
  schema is always printed** (rendered field table from `plan_frontmatter_schema` plus a minimal valid example) so the
  agent can self-correct — this is the core of the validate → edit → revalidate loop. Then a summary line.
- JSON output: `{schema_version, ok, tier, path, diagnostics: [...], expected_schema: {...}}`.
- Exit codes: `0` valid; `1` any validation error; `2` usage errors (argparse).

## Enforcement Boundaries

1. **Proposal queue** — `handle_plan_propose_command` reads the _authored_ `tier` from the file's frontmatter (now
   required at propose time), runs the engine, and on failure prints the full diagnostics + expected schema and exits 1
   **without** archiving the file, writing `.sase_plan_pending`, or killing the runner group — the planning agent stays
   alive to fix and re-propose. When an auto-approval action is pinned (`SASE_AGENT_AUTO_APPROVE_PLAN_ACTION` =
   `tale`/`epic`), a mismatch with the authored tier is also a propose-time error (catches the "authored tale under
   `%auto:epic`" dead end early).
2. **Epic approval** — a single gate helper validates the pending plan file against the `epic` schema before any side
   effects, called from: the host-side approval executor (`execute_plan_approval_response` in
   `src/sase/plan_approval_actions.py`, which serves the CLI, ACE TUI, and remote/Telegram choices), the auto-approval
   resolution path (`%auto:epic`), and defensively in the runner's epic branch (`handle_accepted_plan` in
   `src/sase/axe/run_agent_exec_plan_accept.py`) before SDD write/commit, notifications dismissal, bead creation, or PR
   setup. A failed gate refuses the approval with diagnostics and leaves the proposal pending.
3. **Committed plan files** — (a) commit-path: the SDD plan writers (`write_sdd_files` in `src/sase/sdd/_write.py`,
   `_archive_plan_for_approval`, and `handle_sase_plan` in `src/sase/workflows/commit/commit_hooks.py`) validate before
   writing or committing a plan into the SDD store; (b) CI: a sweep over the committed plans store (see Phase 6) runs in
   the repo's CI lint job, which already checks out the plans sidecar. The sweep is an internal script + Justfile
   recipe, not a new user-facing bulk mode on the command and not an extension of `sase validate`/`sase sdd validate`
   UX.

### Approval-kind ergonomics

Since plans now author their tier, `sase plan approve` and the TUI approval modal default the kind from the authored
tier (explicit `--kind`/key still overrides). A cross-tier override revalidates against the _target_ tier before
proceeding: upgrading a tale-authored plan to epic fails (no `phases`) with diagnostics — correct, the plan lacks the
required structure; downgrading an epic-authored plan to tale passes (epic-only fields become inert warnings).

## Deterministic Epic Bead Creation (replaces `bd/new_epic`)

On a passing epic approval, the runner's epic branch stops spawning the `#bd/new_epic:<plan_ref>` follow-up agent and
instead runs a new deterministic host-side flow (new module, e.g. `src/sase/bead/epic_from_plan.py`), consuming the
normalized `plan` structure returned by `plan_validate`:

1. Create the epic plan bead via `BeadProject.create` (`src/sase/bead/project.py`): title = frontmatter `title`,
   description = `goal`, `--type plan(<plan_ref>)` using the existing `build_epic_plan_ref` reference, `tier=epic`,
   model = top-level `model` (omitted when absent), ChangeSpec metadata from `changespec`/`bug_id` when present.
2. Write `bead_id` back into the plan file's frontmatter and commit the update to the SDD store (same as `bd/new_epic`
   instructed).
3. Create one phase bead per `phases[]` entry, strictly in list order (child-ID suffixes are allocation-ordered), with
   title/description/model from the entry; generate the deterministic description pointer when `description` is absent.
4. Add dependency edges via `BeadProject.add_dependency` per `depends_on` (frontmatter phase id → allocated bead child
   id).
5. Invoke the `sase bead work` launch path internally (equivalent of `sase bead work <epic_id> --yes`), reusing its
   existing wave scheduling, pre-claiming, force-reuse naming, launch, rollback, and bead-state commit/push behavior
   (`src/sase/bead/cli_work_handler.py`, `src/sase/bead/work.py`).

Boundary note: frontmatter parsing/normalization is Rust (shared with validation); bead creation goes through the
existing Rust mutation facade; orchestration, plan-ref resolution, SDD commits, and agent launching remain Python host
logic — matching the documented bead backend split (`docs/beads.md`).

Retirement: remove the `bd/new_epic` xprompt from `src/sase/default_config.yml`, rewire the `epic` role in
`src/sase/agent_family/standard_plan_chain_definition.py`, and retire the `@epic_creator` role alias from docs/config
defaults (tolerate the config key for compat). `bd/work_phase_bead` and `bd/land_epic` remain — `sase bead work` still
uses them.

## Migration & Compatibility

- **Legacy committed plans** (~2,700 files in the plans store) predate `goal`/`phases`. The CI sweep and commit-path
  gate apply the full schema only to plan files in month directories at or after a cutover constant (e.g. `202608`);
  earlier months keep only today's checks (valid `tier`). No mass rewrite of historical plans.
- **Reads stay tolerant**: `classify_plan_file`'s tale fallback and the tolerant Rust discovery in `plan/read.rs` are
  unchanged — validation is strict only at the three enforcement boundaries.
- **Scratch plan files** (`sase_plan_*.md` committed at the repo root historically) are not swept; the propose gate
  covers new ones going forward.
- **Automated flows**: `sase bead work` phase/land segments carry `%auto:tale`, so those agents' submitted plans must
  now conform — the updated `/sase_plan` skill (Phase 3) is the same skill those agents use, and the propose-gate's
  schema-printing failure output is self-teaching for any agent running with a stale skill. Skill sources are generated
  and deployed (`src/sase/xprompts/skills/`, `sase skill init`) — Phase 3's agent must follow the generated-skills
  long-term memory procedure.
- **Cross-repo sequencing**: Phase 1 lands in `../sase-core` and is released by release-plz (conventional `feat:`
  commit; never hand-edit crate versions); Phase 2 bumps the `sase-core-rs` pin in `pyproject.toml`. Later phases are
  pure-Python consumers.

---

## Phase 1: sase-core — plan frontmatter schema model + validation engine

**Goal**: The Rust core owns the authoritative tier-aware plan frontmatter schema, a one-pass diagnostics engine, and
the schema-spec metadata, exposed to Python.

- New `crates/sase_core/src/plan/validate.rs`: typed serde model for common + epic frontmatter (phases with
  `id`/`title`/`depends_on`/`description`/`model`), all rules from "The Frontmatter Schema" above, diagnostics with
  `severity`/`code`/`message`/`field_path`/optional `line` (reuse the location techniques from `editor/frontmatter.rs`;
  reuse `split_frontmatter`/ `parse_frontmatter_map` from `plan/read.rs`).
- Schema-spec metadata per tier (ordered field records with descriptions/examples), including the exact phase-`model`
  guidance description text.
- Bindings in `crates/sase_core_py/src/lib.rs`: `plan_validate(content, tier)` and `plan_frontmatter_schema(tier)`;
  extend `PLAN_WIRE_SCHEMA_VERSION` handling per convention.
- Tests: Rust unit tests covering every rule and the all-problems-in-one-pass behavior (multiple simultaneous errors),
  plus a `plan_validate` parity fixture test in `crates/sase_core/tests/` mirroring the existing `*_parity.rs` pattern.
- Work happens in the sase-core repo (open it with `sase repo open sase-core`); commit as `feat:` so release-plz cuts a
  release Phase 2 can pin.

Dependencies: none.

## Phase 2: `sase plan validate` CLI command + Python facade

**Goal**: Agents can run `sase plan validate PLAN_FILE -t <tier>` and get complete, colored, location-bearing
diagnostics plus the expected tier schema on failure, with JSON parity.

- Bump the `sase-core-rs` pin in `pyproject.toml` to the Phase 1 release.
- New thin facade (e.g. `src/sase/sdd/plan_validate.py`) modeled on `src/sase/xprompt/frontmatter_schema.py`: call the
  bindings, rehydrate diagnostics/schema into frozen dataclasses; no validation rules reimplemented in Python.
- New `validate` subparser in `src/sase/main/parser_plan.py` + `plan_validate_handler.py` + dispatch in
  `plan_command_handler.py`, per the CLI design section (required `-t/--tier`, `-j/--json`, `-q/--quiet`, exit codes,
  Rich rendering, expected-schema block on failure, updated group help/epilog examples). Update `docs/cli.md` and add a
  schema section to `docs/sdd.md`.
- Tests: CLI tests (valid tale, valid epic, tier mismatch, every diagnostic family, JSON shape, exit codes, quiet mode)
  following `tests/test_plan_search_cli.py` patterns; facade tests; a golden test asserting the failure output contains
  the expected schema rendering.

Dependencies: Phase 1.

## Phase 3: Propose-time gate + agent planning-loop updates

**Goal**: Invalid plans cannot enter the proposal queue, and planning agents are taught the tier-aware author → validate
→ edit → revalidate → propose loop.

- Gate `handle_plan_propose_command` per "Enforcement Boundaries" #1 (authored-tier read, engine run, diagnostics +
  schema on stderr, exit 1 with no queue mutation and no runner kill; auto-mode tier cross-check).
- Update the `/sase_plan` skill source (`src/sase/xprompts/skills/sase_plan.md`): how to choose the tier, the required
  frontmatter per tier (with the epic example), the phase-`model` guidance, the `sase plan validate` loop, and only then
  `sase plan propose`. Follow the generated-skills long-term memory procedure for regeneration/deployment
  (`sase skill init`).
- Audit other prompt surfaces that instruct plan submission (e.g. `bd/work_phase_bead`, `bd/land_epic` in
  `src/sase/default_config.yml`) and align wording where they reference plans.
- Tests: propose-gate tests (valid/invalid per tier, missing tier, auto-mode mismatch, marker file not written on
  failure, runner group not killed on failure).

Dependencies: Phase 2.

## Phase 4: Epic-approval validation gate

**Goal**: An epic approval cannot trigger notifications-dismissal, SDD commits, bead creation, or PR setup unless the
plan file passes epic validation; approval kinds default from the authored tier.

- Shared gate helper (e.g. in `src/sase/plan_approval_actions.py` or a sibling module) per "Enforcement Boundaries" #2,
  wired into: `execute_plan_approval_response` (covers CLI approve, ACE TUI modal, remote/Telegram), the auto-approval
  path resolving `%auto:epic`, and the runner's `handle_accepted_plan` epic branch as a defensive re-check.
- Failure UX: CLI prints diagnostics + expected schema and exits 1 with the proposal left pending; the TUI surfaces the
  failure (modal or notification) without consuming the approval; auto-epic failures surface as an actionable error on
  the agent row rather than a silent stall.
- Approval-kind defaulting from authored tier + cross-tier override revalidation, per the ergonomics section.
- Tests: host-gate tests for each entry path (CLI, executor-level for TUI/remote, auto-epic), downgrade/upgrade override
  behavior, and a regression test that a failed gate leaves the proposal approvable after the plan is fixed.

Dependencies: Phase 2 (parallel with Phase 3).

## Phase 5: Deterministic epic bead creation + auto `sase bead work` + retire `bd/new_epic`

**Goal**: Approving an epic programmatically creates the epic bead, phase beads, and dependency edges from frontmatter
and launches the epic via the `sase bead work` path — no `bd/new_epic` agent.

- Implement `epic_from_plan` per "Deterministic Epic Bead Creation" above (creation order, `bead_id` write-back + SDD
  commit, dep wiring, internal `bead work --yes` launch reusing existing scheduling/rollback/commit/push behavior).
- Rewire the epic branch of `handle_accepted_plan`; remove the `bd/new_epic` xprompt; update
  `standard_plan_chain_definition.py`'s epic role and the agent-family/naming expectations that assumed an epic-creator
  follow-up agent; retire `@epic_creator` from docs and default config (tolerating existing user config).
- Failure handling: bead-creation or launch errors roll back cleanly (existing `rollback_work_launch` pattern) and
  surface an actionable error; the epic approval is not half-applied.
- Update `docs/beads.md`, `docs/sdd.md` (rewrite the Model Field section: per-phase models now live in `phases[].model`,
  not body annotations), and agent-family docs.
- Tests: unit tests for the frontmatter→bead mapping (titles, descriptions incl. generated pointers, models, ChangeSpec
  metadata, dep edges, ordering); integration test driving the epic branch of `handle_accepted_plan` with a fake
  launcher asserting beads + `bead work` launch; an end-to-end test from a valid epic plan file through approval to a
  `bead work --dry-run`-level wave plan.

Dependencies: Phase 4.

## Phase 6: CI and commit-path enforcement for committed plan files

**Goal**: Committed plan files at or after the cutover month always satisfy the schema, in both the SDD commit path and
CI, without breaking legacy history.

- Commit-path gates in `write_sdd_files`, `_archive_plan_for_approval`, and `commit_hooks.handle_sase_plan` (validate
  the plan against its frontmatter tier before writing/committing to the SDD store; post-approval these should already
  pass — the gate is defensive).
- Committed-plans sweep: an internal script (e.g. under `src/sase/scripts/`) that resolves the plans root, enumerates
  `YYYYMM/*.md` at/after the cutover constant, runs the engine per file (tier from frontmatter), and prints an aggregate
  diagnostics report with nonzero exit on any error; legacy months get only the existing tier check. A new Justfile
  recipe runs it, and a new step in the CI lint job (`.github/workflows/ci.yml`, which already checks out the plans
  sidecar) invokes that recipe. Do not route it through `sase validate`/`sase sdd validate`.
- Document the cutover policy and the sweep in `docs/sdd.md`.
- Tests: sweep-script tests (cutover boundary, legacy leniency, aggregate reporting, exit codes) and commit-path gate
  tests.

Dependencies: Phase 2 (parallel with Phases 3–5).

---

## Out of Scope (v1)

- Bulk/multi-file modes, directory modes, or tier inference on `sase plan validate`.
- Any prose-quality, feasibility, risk, or completeness judgments.
- Rewriting or backfilling legacy plan files to the new schema.
- Changes to `sase sdd validate` link checking or `sase validate`'s existing checks.
- Editing long-term memory files (`memory/*.md`) — CLI/skill documentation lives in `docs/` and skill sources only.

## Risks

- **Cross-repo coupling**: Phases 2+ block on a sase-core release; the pin bump is the only synchronization point.
  Mitigation: Phase 1 is self-contained and releasable independently.
- **Agent-family expectations around the epic follow-up**: removing the epic-creator agent changes the plan-approval
  transition for epics; Phase 5 must adjust `evaluate_plan_approval_transition`-adjacent logic and TUI expectations
  deliberately.
- **Stale deployed skills** at rollout produce propose failures; acceptable because the failure output prints the
  expected schema (self-teaching), but Phase 3 should land the skill update and the propose gate together.
