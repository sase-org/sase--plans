---
tier: epic
goal:
  Agents can deterministically validate a plan file against a tier-specific schema with
  a single `sase plan validate` command, and SASE enforces that validation automatically
  at the proposal, epic-approval, and CI/hook boundaries — replacing today's
  LLM-inferred plan structure with a checked, machine-readable contract that makes epic
  bead creation programmatic instead of heuristic.
phases:
  - id: rust-core-validator
    title: Rust core plan schema + validation engine + bindings
  - id: cli-command
    title: Python facade and the `sase plan validate` CLI command
    depends_on:
      - rust-core-validator
  - id: agent-teaching
    title: "Teach agents the schema: /sase_plan skill, bd/new_epic xprompt, docs"
    depends_on:
      - cli-command
  - id: lifecycle-enforcement
    title: Enforce validation at propose and epic approval
    depends_on:
      - agent-teaching
  - id: ci-enforcement
    title:
      Enforce committed-plan schemas in SDD validation and CI with epoch grandfathering
    depends_on:
      - agent-teaching
  - id: e2e-verification
    title: End-to-end verification and closeout
    depends_on:
      - lifecycle-enforcement
      - ci-enforcement
create_time: 2026-09-09 19:53:16
status: wip
---

- **PROMPT:**
  [prompts/202607/plan_validate_command.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/plan_validate_command.md)

# Implementation Plan: `sase plan validate` — deterministic, tier-aware plan schema validation

(The frontmatter above dogfoods the epic schema this plan proposes.)

## Product Context

Plan files are the central artifact of the SASE planning pipeline, but nothing validates
their structure at any point where it matters:

- `sase plan propose` (`src/sase/main/plan_propose_handler.py`) checks only that env
  vars are set and the file exists, then archives the plan and SIGTERMs the planner. A
  malformed plan sails straight into the PlanApproval notification queue.
- Epic approval (`src/sase/axe/run_agent_exec_plan_accept.py`, epic branch) commits the
  plan, initializes the bead store, and launches a `bd/new_epic` follow-up agent
  (`src/sase/default_config.yml`, `bd/new_epic` xprompt) that must **guess** the phase
  list, phase order, per-phase `model:` annotations, and inter-phase dependencies from
  free-form markdown ("Make sure that each phase bead has the appropriate dependencies
  set up"). No SASE code ever parses phases; the entire epic → bead contract is LLM
  judgment.
- CI validates committed sidecar plans only for frontmatter parseability, `tier`, and
  prompt↔plan links (`validate_sdd_tree` in `src/sase/sdd/links.py`, via
  `sase sdd validate` → `sase validate` → `just validate` → the CI `lint` job). Nothing
  checks that a plan declares its goal or, for epics, its phase graph.

The result: agents cannot know whether a plan is structurally complete before proposing
it, epic bead creation is fragile, and the human reviewer is the only schema check in
the pipeline.

This epic adds an agent-facing `sase plan validate` command that performs
**deterministic schema validation only** — syntax, frontmatter, and tier-specific
structural fields — and enforces it automatically at all three lifecycle boundaries. The
intended agent loop:

1. Author the plan with tier-appropriate frontmatter.
2. Run `sase plan validate <file> --tier <tale|epic>`.
3. On failure, read the actionable diagnostics **plus the expected tier-specific
   frontmatter schema** printed with them, edit the plan, and rerun.
4. Only when validation passes, run `sase plan propose <file>`.

## Settled Requirements (do not re-litigate)

1. The command validates exactly one explicit `PLAN_FILE` argument. No plan inference
   from agent context; no bulk mode in v1.
2. The calling agent explicitly provides the tier; only `tale` and `epic` are supported.
3. V1 is deterministic schema validation only: syntax, frontmatter, tier-specific
   structural fields. It must not judge prose quality, feasibility, testing quality,
   risks, or completeness.
4. The command is independent in purpose and UX from `sase validate` and
   `sase sdd validate`; neither existing command is the user-facing surface for this
   validator.
5. Every supported plan tier requires a top-level `goal` frontmatter property describing
   the outcome the plan achieves.
6. An epic plan must declare, as schema-required structured data, everything needed to
   programmatically create the parent epic bead and every ordered phase bead, including
   dependency information sufficient to wire the phase graph.
7. All discovered problems are reported in one run; each diagnostic is actionable and
   identifies the file and location when possible; any validation failure exits nonzero.
8. Successful validation is enforced automatically at three boundaries: before a plan
   enters the proposal approval queue; during epic approval before notifications, bead
   creation, or PR setup; and in CI/hook checks for committed plan files.

## Verified Current State (reconnaissance summary)

- **CLI group**: `sase plan {approve,list,propose,reject,search}` registered in
  `src/sase/main/parser_plan.py`, dispatched via
  `src/sase/main/plan_command_handler.py`. `sase plan list` already spells the tier
  filter `-t/--tier` with choices `("tale", "epic")`; `sase bead create` uses
  `--tier {plan,epic}`. There is no `validate` subcommand.
- **Tier machinery**: `src/sase/sdd/plan_tiers.py` (`PLAN_TIERS = ("tale", "epic")`,
  `normalize_plan_tier`, `read_plan_tier`, `classify_plan_file` defaulting to `tale`).
  Tier frontmatter is stamped at approval/commit time
  (`src/sase/plan_approval_actions.py` `_archive_plan_for_approval`;
  `src/sase/sdd/_write.py`; `src/sase/workflows/commit/commit_hooks.py`
  `handle_sase_plan`). Authored scratch plans today carry **no** frontmatter.
- **Proposal queue**: "entering the queue" == `notify_plan_approval`
  (`src/sase/notifications/senders.py`) appending a `PlanApproval` notification,
  triggered runner-side after `sase plan propose` writes its `.sase_plan_pending` marker
  and kills the runner group. The propose handler is the single user-facing entry point.
- **Epic approval**: all approval surfaces (CLI `plan_approve_handler.py`, TUI
  notification modals, remote/Telegram) converge on the runner's `handle_accepted_plan`
  (`src/sase/axe/run_agent_exec_plan_accept.py`). For `action=="epic"` it commits SDD
  files, calls `ensure_beads_initialized`, and spawns the `bd/new_epic` follow-up with
  the VCS/PR workflow prefix. Bead creation and PR setup happen inside that follow-up
  agent.
- **Beads**: `sase bead create -t TITLE -T "plan(<file>)" --tier epic [-m MODEL]`
  creates the parent; `-T "phase(<parent_id>)"` creates ordered children (IDs
  `<parent>.<N>` allocated by creation order); `sase bead dep add <issue> <depends_on>`
  wires the graph; `sase bead work <epic>` builds a Kahn-wave schedule from those
  dependencies. The bead domain model lives in Rust (`sase_core::bead`), with thin
  Python facades.
- **Existing validators**: `validate_sdd_tree` (`src/sase/sdd/links.py`) is the
  collect-all precedent — `SddIssue(severity, code, path, message)`
  (`src/sase/sdd/_link_models.py`), text format
  `"{severity}: {path}: {message} ({code})"` with errors on stderr, `-j/--json`, exit
  `0 if ok else 1`, plus a closed `LEGACY_INVALID_SDD_ERROR_ALLOWLIST` grandfathering
  pattern.
- **Rust boundary**: every deterministic validation engine already lives in the
  sase-core linked repo (`crates/sase_core`): a JSON-schema-subset validator
  (`config/validate.rs`, exposed as `config_validate`), a frontmatter validation engine
  with positional diagnostics (`editor/frontmatter.rs`, backing the xprompt LSP), and
  bead schema constraints (`bead/schema.rs`). Plan discovery/search already parses plan
  frontmatter in Rust (`plan/read.rs`), consumed through the thin
  `src/sase/plan_search/facade.py` adapter. `src/sase/config/inventory.py` and
  `src/sase/xprompt/frontmatter_schema.py` document the "validation decision in Rust, IO
  in Python" adapter pattern. The Python-only SDD link validator is the lone un-migrated
  exception.
- **CI/hooks**: CI `lint` job checks out the plans/research sidecars, writes an SDD
  store record, and runs `just validate` (→ `sase validate` → `init --check` +
  `sdd validate`). `just check` also runs `just validate`. Project hooks:
  `commit_hooks.before: just fix`; ChangeSpec
  `vcs_provider.default_hooks: [just lint, just test]`.

## Design Decisions

### D1. The authoritative schema lives in the Rust core

Per the rust_core_backend_boundary litmus test, plan schema validation is core backend
logic: the CLI, the approval runner, CI, the TUI, and plausibly the LSP/mobile frontends
all need identical verdicts. The precedent is uniform — config validation, xprompt
frontmatter validation, and the bead schema all live in `sase_core` behind thin Python
adapters, and plan frontmatter parsing for search is already Rust.

Therefore: a new `sase_core::plan::validate` module (reusing `plan/read.rs` frontmatter
splitting and the diagnostic patterns of `editor/frontmatter.rs`) is the single schema
authority. It exposes two bindings through `sase_core_py`:

- `plan_validate(request)` → structured diagnostics for one plan text + explicit tier.
- `plan_validate_schema(tier)` → ordered field descriptors (name, required, type,
  allowed values, description, example) used to render the "expected schema" block in
  failure output — mirroring the existing `frontmatter_field_schema` pattern so help
  text and validation can never diverge.

Python gets a thin adapter (`src/sase/plan_validation/facade.py`, modeled on
`plan_search/facade.py` and `xprompt/frontmatter_schema.py`): rehydrate binding dicts
into frozen dataclasses, own file IO and path/relpath handling, reimplement no rules.

### D2. Tier schemas

Common to both tiers (all deterministic; codes are stable machine strings):

- File must be readable UTF-8 markdown (`file-unreadable`).
- A `---`-delimited YAML frontmatter block is required, must close, and must parse to a
  mapping (`frontmatter-missing`, `frontmatter-unclosed`, `frontmatter-parse`,
  `frontmatter-not-mapping`).
- `tier`: required; must be `tale` or `epic`; must equal the explicitly passed tier
  (`tier-missing`, `tier-invalid`, `tier-mismatch`).
- `goal`: required non-empty string describing the outcome the plan is designed to
  achieve (`goal-missing`, `goal-empty`, `goal-type`).
- Exactly one markdown H1 heading with non-empty text is required in the body
  (`title-missing`, `title-duplicate`). The H1 is the deterministic title source for the
  epic plan bead.
- System-managed/optional fields are accepted without complaint: `create_time`,
  `status`, `prompt`, `bead_id`, `model` (top-level model, already documented in
  `docs/sdd.md`).
- Unknown top-level keys produce **warnings** (`unknown-field`), not errors, for forward
  compatibility.

Epic-only — the structured data that makes bead creation programmatic:

```yaml
tier: epic
goal: <outcome statement>
phases:
  - id: short-slug # optional; defaults to the 1-based position; unique, [a-z0-9_-]+
    title: <phase title> # required, non-empty, unique across phases
    description: <text> # optional; forwarded to `sase bead create -d`
    model: <model-ref> # optional; forwarded to `sase bead create -m`
    depends_on: [ref, ...] # optional; each ref is an earlier phase's id or 1-based index
```

- `phases`: required non-empty list of mappings (`phases-missing`, `phases-empty`,
  `phases-type`); forbidden on tale plans (`phases-on-tale`).
- List order **is** the bead creation order (child suffix allocation order) and the
  canonical phase numbering.
- `depends_on` refs must resolve to a **strictly earlier** phase (`phase-dep-unknown`,
  `phase-dep-forward`, `phase-dep-self`, `phase-dep-duplicate`). Earlier-only references
  make the graph acyclic by construction and match how `sase bead work` Kahn-schedules
  phases; phases without `depends_on` are roots and may run in parallel.
- Uniqueness and shape checks: `phase-title-missing`, `phase-title-duplicate`,
  `phase-id-invalid`, `phase-id-duplicate`, `phase-model-type`, plus `unknown-field`
  warnings on unknown per-phase keys.

This gives `bd/new_epic` (and any future fully-programmatic creator) everything it
needs: parent bead title (H1), tier, optional epic model (top-level `model`), ordered
phase titles/descriptions/models, and the dependency graph.

V1 deliberately does **not** cross-check `## Phase` body headings against the
frontmatter list (see Non-Goals) — the frontmatter is the machine contract; the body
remains prose.

### D3. CLI shape

```
sase plan validate PLAN_FILE -t/--tier {tale,epic} [-j/--json]
```

- `PLAN_FILE`: required positional, exactly one, no default, no inference.
- `-t/--tier`: **required** option. `--tier` is the established tier spelling
  (`sase plan list -t/--tier`, `sase bead create --tier`); `--kind` is reserved for
  approval kinds which include non-tier values. Required-named options follow the
  `sase bead create -t/-T` precedent, and every long option gets a short alias per CLI
  rules.
- `-j/--json`: machine-readable output, matching `sase sdd validate` / `sase plan list`.
- No env-var guards (unlike `propose`): the command must run anywhere, including CI and
  interactive shells.
- Registered alphabetically in `src/sase/main/parser_plan.py` (approve, list, propose,
  reject, search, validate) with help/examples meeting the CLI-rules bar; dispatched via
  `plan_command_handler.py` to a new `src/sase/main/plan_validate_handler.py`.

Exit codes: `0` on pass; `1` on any validation failure (including unreadable/missing
file, reported as a diagnostic); `2` for argparse usage errors (missing `--tier`, bad
tier value) — argparse's native behavior.

### D4. Diagnostic behavior

- **Collect-all**: every problem found in one run, following `validate_sdd_tree`'s
  accumulation model.
- Text format mirrors the SDD validator so output stays greppable and CI-friendly:
  `"{severity}: {path}:{line}: {message} ({code})"` — errors to stderr, warnings to
  stdout, plain (non-Rich) output. Line/column comes from the Rust engine's positional
  spans where available (frontmatter fields); diagnostics that have no meaningful
  position (e.g. `frontmatter-missing`) omit the line component.
- **Schema-on-failure**: whenever validation fails, the command prints the expected
  tier-specific frontmatter schema — rendered from `plan_validate_schema(tier)` — as a
  fenced YAML template plus a short field table, so an agent can fix the file without
  any other reference. On pass, print a single
  `plan validation passed: <path> (tier: <tier>)` line.
- JSON shape:
  `{"file", "tier", "ok", "errors": [...], "warnings": [...], "schema": {...}}` with
  each diagnostic as `{"severity", "code", "line", "column", "message"}` (line/column
  nullable), versioned by field stability like `validation_to_json`.

### D5. Enforcement at the three boundaries

1. **Before the proposal queue** — in `handle_plan_propose_command`
   (`src/sase/main/plan_propose_handler.py`), after the file-existence guard and
   **before** prettier formatting, `move_plan_to_sase`, the `.sase_plan_pending` marker,
   and the runner-group SIGTERM. The tier is read from the plan's now-required `tier`
   frontmatter (no CLI tier on `propose`). On failure: print the same diagnostics +
   expected schema and exit 1 — the planner agent survives, edits the plan, and reruns.
   Since `propose` is the only writer of the pending marker, this single gate covers
   every path into the PlanApproval queue, including `%auto` flows and xprompt swarms.
2. **During epic approval** — at the runner choke point `handle_accepted_plan`
   (`src/sase/axe/run_agent_exec_plan_accept.py`), in the epic branch **before**
   `ensure_beads_initialized` and before the `bd/new_epic` follow-up prompt (which
   carries the VCS/PR prefix) is built. This is the last point SASE code controls before
   notifications, bead creation, and PR setup, and it covers every approval surface
   (CLI, TUI, remote/Telegram, `%auto:epic`). On failure the run aborts with an
   actionable error surfaced through the standard axe error-notification path, and no
   beads/follow-up/PR side effects fire. Additionally, `sase plan approve --kind epic`
   performs the same check host-side in `plan_approve_handler.py` before writing
   `plan_response.json`, so CLI approvers get an immediate failure (exit 2, notification
   stays pending) instead of a downstream runner abort. Cross-tier approval of a valid
   plan (e.g. `--kind tale` on a `tier: epic` plan) fails the `tier-mismatch` check by
   design — the plan's declared tier is authoritative and the approver's kind must agree
   with it.
3. **CI/hooks for committed plan files** — `validate_sdd_tree` (`src/sase/sdd/links.py`)
   gains a per-plan-file schema check that calls the same Rust validator (tier read from
   the committed plan's frontmatter, which the archive path always stamps). Enforcement
   then rides the existing rails with zero new CI plumbing: `sase sdd validate` →
   `sase validate` → `just validate` → `just check` locally and the CI `lint` job (which
   already materializes the plans sidecar). This keeps requirement 4 intact:
   `sase plan validate` remains the independent agent-facing UX; the SDD/CI surface
   merely shares the same core engine, exactly as doctor shares `validate_sdd_tree`
   today.

### D6. Migration and compatibility

- **Existing committed plans** (hundreds across monthly sidecar dirs) predate the schema
  and lack `goal`/`phases`. A single epoch constant, `PLAN_SCHEMA_EPOCH_YYYYMM`, gates
  the committed-plan check: plans in monthly directories `>=` the epoch must satisfy the
  full tier schema; older plans keep today's checks (frontmatter parses + valid `tier`).
  This is deterministic, self-documenting, and avoids a per-file allowlist crawl (the
  existing `LEGACY_INVALID_SDD_ERROR_ALLOWLIST` pattern stays for its two historical
  files). The epoch is set during the ci-enforcement phase to the first month strictly
  after it lands, with a test asserting the real sidecars validate clean at that epoch.
- **Scratch plans and old-skill agents**: during rollout, an agent still following the
  old skill (no frontmatter) hits the propose gate and receives the full expected schema
  in the failure output — the loop is self-correcting even before the updated skill
  reaches every runtime. Phase ordering still lands the teaching phase before the
  enforcement phases to minimize that window.
- **`bd/new_epic` backward compatibility**: the rewritten xprompt consumes the
  structured `phases` frontmatter, with a short fallback note for legacy plans that lack
  it (possible only for pre-epoch epics re-driven by hand, since the approval gate
  guarantees new epics validate). All agent runtimes get the same instructions
  (uniform-runtimes rule).
- **Stamping paths preserve authored fields**: `_archive_plan_for_approval`,
  `_write.py`, and `handle_sase_plan` all use `set_frontmatter_fields`, which updates
  rather than replaces frontmatter, so authored `goal`/`phases` survive archive/commit.
  The archive path keeps stamping `tier`, which after the mismatch check is always a
  no-op re-assertion of the authored value.
- **Proposal-queue inventory bonus**: `sase plan list` tier columns (currently `-` for
  proposals) start showing real tiers because authored plans now declare `tier` — no
  code change required.

## Phasing Overview

Six phases. Each is completed by a distinct agent instance, is independently landable,
and leaves the repo releasable (green `just check`; run `just install` first in a fresh
workspace). Phase 1 works in the sase-core linked repo (open it with the `/sase_repo`
skill) and follows that repo's own checks and release process.

| Phase | id                    | Deliverable                                                  | Depends on |
| ----- | --------------------- | ------------------------------------------------------------ | ---------- |
| 1     | rust-core-validator   | `sase_core::plan::validate` engine + `sase_core_py` bindings | —          |
| 2     | cli-command           | Python facade + `sase plan validate` CLI                     | 1          |
| 3     | agent-teaching        | `/sase_plan` skill, `bd/new_epic` xprompt, docs/templates    | 2          |
| 4     | lifecycle-enforcement | Propose gate + epic-approval gate                            | 3          |
| 5     | ci-enforcement        | Committed-plan schema checks in `validate_sdd_tree` + epoch  | 3          |
| 6     | e2e-verification      | End-to-end tests, real-sidecar audit, closeout               | 4, 5       |

Phases 4 and 5 may run in parallel. The 3 → 4/5 ordering is a rollout choice (teach
before enforcing), not a technical dependency.

## Phase 1 — Rust core plan schema + validation engine + bindings

**Goal:** the single authoritative implementation of the tale/epic plan schemas,
producing positional, coded diagnostics, exposed to Python.

**Where:** the sase-core linked repo (`crates/sase_core`, `crates/sase_core_py`). Open
it via the `/sase_repo` skill.

**Scope:**

- New `sase_core::plan::validate` module implementing every rule in D2 against raw
  markdown text plus an explicit tier. Reuse the existing frontmatter splitting in
  `plan/read.rs` (factor shared helpers rather than duplicating) and follow the
  diagnostic conventions of `editor/frontmatter.rs` (severity, stable code, span). Wire
  types live beside the existing plan wire records (`plan/wire.rs` conventions).
- A schema-description API returning ordered field descriptors per tier (name, required,
  type, allowed values, description, example) — the data source for schema-on-failure
  rendering and for docs, so rules and guidance cannot diverge.
- `sase_core_py` pyfunctions `plan_validate(request)` and `plan_validate_schema(tier)`
  following the existing request/response dict wire style.
- Comprehensive Rust unit tests: every diagnostic code has at least one triggering
  fixture and one passing fixture; epic dependency-graph cases (forward ref, self ref,
  unknown ref, duplicate ids/titles, index and slug refs); YAML edge cases (unclosed
  frontmatter, non-mapping, empty file).
- Version bump and release per the sase-core repo's release process so downstream phases
  can raise the `sase-core-rs` floor.

**Acceptance:** sase-core checks green (fmt/clippy/tests per its Justfile); bindings
callable from a local editable install; no changes in the sase repo yet.

## Phase 2 — Python facade and the `sase plan validate` CLI command

**Goal:** the agent-facing command, end to end, per D3/D4.

**Scope:**

- Raise the `sase-core-rs` version floor in `pyproject.toml` to the Phase 1 release.
- New thin adapter `src/sase/plan_validation/facade.py`: frozen dataclasses for
  diagnostics/schema descriptors, `require_rust_binding` calls, file reading and path
  presentation. No rule logic in Python (document this in the module docstring like
  `config/inventory.py` does).
- Parser registration in `src/sase/main/parser_plan.py` (alphabetical order, `-t/--tier`
  required with `choices=("tale", "epic")`, `-j/--json`, excellent help + examples),
  dispatch in `plan_command_handler.py`, handler in
  `src/sase/main/plan_validate_handler.py` implementing text/JSON output,
  schema-on-failure rendering, stderr/stdout split, and exit codes.
- Update `docs/cli.md` and the `sase plan` group help/examples.
- Focused tests: parser tests alongside the existing plan-parser suite (`tests/main/`),
  handler tests covering pass/fail/JSON/schema-block/exit codes with golden-ish
  assertions, facade rehydration tests, and a fixture-driven matrix that exercises each
  diagnostic code through the real binding.

**Acceptance:** `sase plan validate <file> -t epic` on this very plan file passes; on a
deliberately broken copy it reports every seeded defect in one run, prints the epic
schema, and exits 1. `just check` green.

## Phase 3 — Teach agents the schema: /sase_plan skill, bd/new_epic xprompt, docs

**Goal:** every plan-authoring agent learns the frontmatter contract and the
validate/edit/revalidate loop; the epic bead creator consumes structured data instead of
guessing.

**Scope:**

- Rewrite `src/sase/xprompts/skills/sase_plan.md` (the generated-skill source template;
  per `memory/generated_skills.md`, deployment happens via `sase skill init --force` +
  chezmoi, uniformly across all runtimes): teach tier selection, the required
  frontmatter for each tier (rendered examples matching `plan_validate_schema`), the
  loop — run `sase plan validate <file> --tier <tier>`, fix using the printed schema +
  diagnostics, rerun until pass — and only then `sase plan propose`.
- Rewrite the `bd/new_epic` xprompt in `src/sase/default_config.yml`: read `phases` from
  the plan frontmatter; create the parent bead from the H1 title, `--tier epic`, and
  top-level `model`; create phase beads in list order (serially, preserving suffix
  allocation) with each phase's `title`/`description`/`model`; wire `sase bead dep add`
  edges exactly from `depends_on` (resolving ids and indices); keep a brief legacy
  fallback for pre-schema plans.
- Documentation: `docs/sdd.md` (authored-schema section replacing the loose per-phase
  `model:` convention text), `docs/beads.md` (epic creation now schema-fed), SDD guide
  templates (`src/sase/sdd/templates/plans-README.md`, `templates/README.md`) which
  currently document only the `tier` requirement.
- Tests: skill/xprompt content tests if precedent exists (there are existing tests over
  default_config xprompts); otherwise structural assertions that the xprompt references
  the schema fields, plus docs lint via `just check` (`fmt-md-check`).

**Acceptance:** regenerated skill text instructs the exact loop; `bd/new_epic` no longer
contains any "infer the phases" language; `just check` green.

## Phase 4 — Enforce validation at propose and epic approval

**Goal:** boundaries 1 and 2 of requirement 8.

**Scope:**

- **Propose gate** in `handle_plan_propose_command`: validate with tier from frontmatter
  (a missing/invalid `tier` is itself a schema failure) before any side effect
  (prettier, move, marker, SIGTERM). Failure prints diagnostics + schema and exits 1,
  leaving the agent alive and the workspace untouched.
- **Epic-approval gate** in `handle_accepted_plan`'s epic branch, before
  `ensure_beads_initialized` and follow-up prompt construction: re-validate the plan
  file that will seed `bd/new_epic` with tier `epic`. Failure aborts the epic side
  effects and surfaces the diagnostics through the standard axe error-notification path.
- **CLI approve pre-check** in `plan_approve_handler.py` for `--kind epic` (and the
  `tier-mismatch` check for `--kind tale` on epic-declared plans): fail fast with
  diagnostics, exit 2, notification remains pending and re-approvable after the plan is
  fixed.
- Tests: propose-gate unit tests (valid passes through; invalid exits 1 with schema
  block and no marker/move/SIGTERM); approval-gate tests in the existing axe exec-plan
  test area (valid epic proceeds; invalid epic creates no beads and launches no
  follow-up; tale flow untouched); CLI approve tests alongside the existing approve-CLI
  suite; `%auto:epic` path covered via the runner gate tests.

**Acceptance:** an invalid plan cannot enter the queue from `propose`, and an invalid
epic cannot reach bead creation, notifications, or PR setup from any approval surface.
`just check` green.

## Phase 5 — Enforce committed-plan schemas in SDD validation and CI with epoch grandfathering

**Goal:** boundary 3 of requirement 8, riding existing rails.

**Scope:**

- Extend `validate_sdd_tree` in `src/sase/sdd/links.py`: for each plan-kind file in a
  monthly directory `>=` `PLAN_SCHEMA_EPOCH_YYYYMM`, run the Rust validator with the
  tier from frontmatter and map its diagnostics into `SddIssue`s (prefix or reuse the
  stable codes; keep severity mapping 1:1). Pre-epoch plans keep today's checks
  unchanged. Set the epoch to the first month strictly after this phase lands.
- Confirm the enforcement chain needs no new plumbing: `sase sdd validate`,
  `sase validate`, `just validate`, `just check`, and the CI `lint` job all pick the
  check up automatically. Add a brief note to `docs/sdd.md`'s validation section.
- Tests: epoch boundary tests (pre-epoch lenient, post-epoch strict) over synthetic SDD
  trees; JSON projection includes the new issues; a guard test that the current real
  plans-sidecar contents remain valid at the chosen epoch (CI's lint job materializes
  the sidecars, so this also runs in CI); doctor bridge (`checks_config_sdd.py`)
  continues to aggregate correctly.

**Acceptance:** a post-epoch committed plan missing `goal` (or an epic missing `phases`)
fails `sase sdd validate`, `just validate`, `just check`, and CI; all existing committed
plans still pass. `just check` green.

## Phase 6 — End-to-end verification and closeout

**Goal:** prove the full agent loop and the three boundaries work together; close
remaining gaps.

**Scope:**

- End-to-end tests driving the real CLI: author an invalid epic plan →
  `sase plan validate` reports all defects in one run with the schema block and exits 1
  → apply fixes → passes → `sase plan propose` succeeds (in a harnessed env with
  `SASE_AGENT`/`SASE_ARTIFACTS_DIR` set) → approval-path validation admits it; mirror
  the negative path at each boundary. Reuse the existing exec-plan/approval test
  harnesses rather than inventing new fixtures.
- A dogfood check that this epic's own plan file (as committed to the plans sidecar)
  validates as `tier: epic`.
- Audit for stragglers: any remaining docs (`docs/cli.md`, `docs/sdd.md`,
  `docs/beads.md`, README command tables), help-text sorting,
  `sase plan search`/inventory interactions with the new frontmatter fields, and the
  cross-repo gotcha list (config schema sync is n/a — no new config keys; nvim syntax
  unaffected unless status tokens changed).
- Run the full local gate (`just install`, `just check`, `just test-visual` only if TUI
  rendering was touched — it should not be) and confirm CI parity.

**Acceptance:** E2E suite green locally and in CI; no doc or help drift; epic closes
with all phase beads done.

## Non-Goals (v1)

- No judgment of prose quality, feasibility, risk coverage, testing adequacy, or
  completeness — schema only.
- No bulk/multi-file or directory mode on `sase plan validate`; exactly one `PLAN_FILE`
  per invocation (CI iterates via `validate_sdd_tree`, not via this command).
- No cross-check of `## Phase` body headings against the frontmatter `phases` list
  (future enhancement; the frontmatter is the contract).
- No replacement of the agent-driven `bd/new_epic` flow with fully programmatic bead
  creation — the schema makes that possible later, but v1 only feeds the existing agent
  structured data.
- No LSP/editor integration for plan files (the Rust engine's positional diagnostics
  make this a natural follow-up in `sase_xprompt_lsp`).
- No changes to research-note or prompt-snapshot validation.

## Risks and Mitigations

- **sase-core release coupling.** Phase 2+ requires a published `sase-core-rs` with the
  new bindings. Mitigation: Phase 1 ends with the release per that repo's process
  (established precedent: bead backend, plan search); Phase 2 raises the version floor;
  `require_rust_binding` fails loudly if a stale core is installed.
- **Rollout window with old skill instructions.** Agents authored plans without
  frontmatter until the new skill deploys. Mitigation: teaching phase lands before
  enforcement phases; the propose-gate failure output contains the complete expected
  schema, so even un-updated agents self-correct in one iteration.
- **Breaking CI on historical plans.** Hundreds of committed plans predate the schema.
  Mitigation: epoch grandfathering (D6) plus a real-sidecar guard test in Phase 5.
- **Approval-kind vs declared-tier conflicts.** Reviewers could previously flip a plan's
  tier at approval time; now the declared tier wins and mismatched approvals fail.
  Mitigation: explicit, actionable `tier-mismatch` messaging at the CLI/host pre-checks;
  reviewers ask the planner (or edit the archived file) to change tiers deliberately.
- **Frontmatter round-tripping.** Prettier formatting at propose and
  `set_frontmatter_fields` stamping at archive/commit must not corrupt authored
  `phases`. Mitigation: both already preserve/merge frontmatter; Phase 4/5 tests include
  round-trip assertions (validate → propose-format → archive-stamp → still valid).
- **TUI responsiveness.** Validation runs in CLI/runner processes, not the TUI event
  loop; the optional TUI approval pre-check is deferred precisely to avoid
  `memory/tui_perf.md` concerns.

## Verification (every phase)

- `just install` first (ephemeral workspaces), then `just check` before finishing; Phase
  1 uses the sase-core repo's own fmt/clippy/test gates instead.
- Phases 2, 4, 5, 6 add or extend pytest suites under `tests/` colocated with the areas
  they touch; Phase 1 adds Rust unit tests; Phase 3 verifies generated-skill/xprompt
  content and docs formatting.
- Phase 6 is the explicit CI-parity and end-to-end gate for the epic as a whole.
