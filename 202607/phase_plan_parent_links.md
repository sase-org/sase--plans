---
tier: epic
title: Stamp phase plans with their bead and parent epic plan
goal: 'Plans proposed from epic bead work automatically record which bead the proposing
  agent is working (`bead`) and which epic plan file spawned that work (`parent`),
  for both phase (tale) plans and child epic plans.

  '
phases:
- id: core-schema
  title: Accept bead and parent plan frontmatter in sase-core
  depends_on: []
  size: small
  description: '''Phase 1: sase-core schema support'' section: extend the Rust plan
    validator, frontmatter schema, and wire payload to accept and expose the managed
    `bead` and `parent` fields on both plan tiers.'
- id: propose-stamping
  title: Stamp bead and parent at sase plan propose
  depends_on:
  - core-schema
  size: small
  description: '''Phase 2: propose-time stamping'' section: stamp `bead` and `parent`
    frontmatter from the epic bead-work environment in the plan propose handler and
    surface the new fields through the Python validation adapter.'
create_time: 2026-07-20 11:40:47
status: wip
---

# Plan: Stamp phase plans with their bead and parent epic plan

## Context

Epic bead work launches one agent per phase plus a land agent (`src/sase/bead/work.py`). Every launch segment receives
the same environment family via `_bead_env`:

- `SASE_PHASE_BEAD_ID` — the phase bead this agent works (phase segments only)
- `SASE_EPIC_BEAD_ID` — the owning epic bead (all segments)
- `SASE_EPIC_PLAN_REF` — the epic plan file reference (all segments), produced by `plan_ref_for_store()` and therefore
  already workspace-relative when possible

The recent child-epic support (sase repo commit 814026c20, sase-core commit 9150852) taught `sase plan propose` to stamp
`parent_bead` onto **epic**-tier proposals from that environment, which is how child epics get created under their
parent bead. Two gaps remain:

1. **Phase plans** (the tale-tier plans that medium/large phase agents author via `#plan`) are deliberately left
   unstamped, so nothing records which phase bead a proposed plan belongs to or which epic plan spawned it.
2. **Child epic plans** record the parent _bead_ (`parent_bead`) but not the parent _epic plan file's path_, so the plan
   files themselves are not linked into a hierarchy.

This epic closes both gaps by stamping two new SASE-managed frontmatter fields in `handle_plan_propose_command`
(`src/sase/main/plan_propose_handler.py`):

| Field    | Stamped on                                 | Value                                                                                                                    |
| -------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `bead`   | tale proposals from bead work              | `SASE_PHASE_BEAD_ID`, falling back to `SASE_EPIC_BEAD_ID` (land agents), mirroring the existing `parent_bead` precedence |
| `parent` | tale **and** epic proposals from bead work | `SASE_EPIC_PLAN_REF` verbatim (already relative when possible)                                                           |

Existing managed fields keep their meanings: `bead_id` remains the epic bead created _from_ an epic plan at launch, and
`parent_bead` remains the bead a child epic is created _under_. Plans proposed outside bead work (no epic environment)
remain completely unstamped.

The Rust plan validator (`sase-core/crates/sase_core/src/plan/validate.rs`) rejects unknown top-level frontmatter keys
as errors at proposal, commit, and launch boundaries, so the schema work must land in sase-core before the stamping can
ship — hence the phase split and dependency. This matches the `parent_bead` rollout, where sase-core 9150852 landed
before sase 814026c20.

## Design decisions

- **Field names**: `bead` and `parent`, as requested. Neither name is used in plan frontmatter today (`memory/notes.py`
  uses a `parent` key only for Obsidian memory notes, an unrelated document type).
- **`parent` value is the plan ref, verbatim.** `SASE_EPIC_PLAN_REF` already carries `plan_ref_for_store()` output
  (workspace-relative like `sase/repos/plans/202607/foo.md`, `.sase/sdd/...` for separate-repo stores, absolute only as
  a last resort). Reusing it verbatim keeps the value consistent with the `sdd_plan_path` agent-meta convention and
  requires no new path resolution.
- **Both new fields are accepted on both tiers** as optional non-empty strings. This keeps validation simple (no new
  inert-field warning machinery), and re-proposals after feedback rounds — which re-validate in authoring mode — stay
  valid once a file has been stamped.
- **Both fields are exposed through the normalized wire payload** (`ValidatedPlanWire` and the Python `_ValidatedPlan`),
  mirroring `parent_bead`, so future consumers (ACE plan linking, hierarchy views) can read them without re-parsing
  frontmatter. The wire schema version stays at 2; the `parent_bead` addition set that precedent for additive optional
  fields.
- **Env wins over authored values** when stamping (same overwrite semantics `parent_bead` has today via
  `set_frontmatter_fields`); with no epic environment present, authored values pass through untouched.

## Phase 1: sase-core schema support

All work in the sase-core repo (open it with the `/sase_repo` skill; `crates/sase_core/src/plan/validate.rs` plus
binding parity coverage).

- Add `bead` and `parent` to the accepted key sets for both tiers and validate each with `optional_non_empty_string`
  during `validate()`, for tale and epic alike.
- Add both fields to `ValidatedPlanWire` as `Option<String>` and populate them from the parsed mapping. Do not bump
  `PLAN_WIRE_SCHEMA_VERSION` (additive optional fields; `parent_bead` precedent).
- Extend `plan_frontmatter_schema()` with specs for both fields, described as SASE-managed (like `bead_id`): `bead` —
  "bead id of the agent that proposed this plan, written by SASE"; `parent` — "parent epic plan file reference written
  by SASE". Place them with the managed/system fields so authoring guidance keeps steering humans and agents away from
  writing them by hand.
- Tests: in-crate validator tests (accepted on both tiers, non-empty-string enforcement, normalized payload exposure,
  unknown-key behavior unchanged for genuinely unknown fields), `tests/plan_validate_parity.rs`, and the binding
  assertions in `crates/sase_core_py/src/lib.rs` that check the returned dict and schema listing (mirroring the existing
  `parent_bead` assertions).
- Run the sase-core checks/tests per that repo's instructions before committing.

## Phase 2: propose-time stamping

All work in the sase repo. Requires the phase 1 sase-core change to be on sase-core master, since dev installs build the
`sase_core_rs` binding from the linked checkout and CI checks sase-core out at master.

- `src/sase/main/plan_propose_handler.py`: extend the existing association block. Read `SASE_PHASE_BEAD_ID`,
  `SASE_EPIC_BEAD_ID`, and `SASE_EPIC_PLAN_REF` (import the env names from `sase.bead.work`); build the stamp mapping
  per tier:
  - tale: `bead` (phase-over-epic precedence) and `parent` (plan ref), each only when non-empty;
  - epic: existing `parent_bead` plus new `parent` when non-empty. Stamp once via `set_frontmatter_fields` before
    prettier formatting, as the `parent_bead` code does today. Rewrite the now-stale comment that says tales
    deliberately remain unstamped so it documents the new tale stamping and its bead/plan sources.
- `src/sase/sdd/plan_validate.py`: add `bead` and `parent` to `_ValidatedPlan` and `_validated_plan_from_dict` so Python
  callers see the normalized values.
- Tests (`tests/test_plan_command_handler.py`): rework the `test_plan_command_stamps_epic_parent_from_bead_work_env`
  parametrization — the `tale-remains-unstamped` case becomes tale-stamps-bead-and-parent; add `SASE_EPIC_PLAN_REF`
  coverage for: tale with full phase env, tale proposed by a land agent (epic-bead fallback), epic gaining `parent`
  alongside `parent_bead`, missing plan ref (bead-only stamp), and the outside-bead-work case staying unstamped. Assert
  `bead`/`parent` surface on validation results where the adapter fields are exercised.
- Run `just check` before finishing.

## Testing

- Phase 1: sase-core validator unit tests, parity tests, and binding tests cover acceptance, normalization, schema
  listing, and rejection of empty values on both tiers.
- Phase 2: propose-handler tests cover every stamp matrix cell above; the full `just check` run guards the rest of the
  plan pipeline (archive, commit-boundary validation, launch re-validation) against unknown-key regressions.
- End-to-end confidence comes from the launch boundary already re-validating archived epic plans
  (`validate_plan_file(..., "epic", mode="launch")`): stamped child epic plans must keep passing it, which the phase 1
  schema change guarantees and phase 2 tests exercise through the archived copies.

## Risks and notes

- **Sequencing**: phase 2 hard-depends on phase 1 being merged in sase-core; the phase dependency plus the
  linked-checkout build (`just install` rebuilds `sase_core_rs` locally) make this safe to verify before the sase change
  lands.
- **Committed-plan cutover**: `PLAN_SCHEMA_CUTOVER_YYYYMM = "202608"` makes committed SDD plans strictly validated
  starting next month. Landing the schema acceptance first means stamped tale plans committed at plan-accept time will
  not trip the cutover.
- **No CLI surface changes**: `sase plan propose` gains no new flags; the stamping is automatic and env-driven, so no
  parser or docs updates beyond the schema descriptions are needed.
- Launch-time backfill of `parent` for hand-authored plans passed directly to `sase bead work <plan-file>` is
  intentionally out of scope; linking is a propose-time concern.
