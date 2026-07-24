---
tier: epic
title: Add xsmall and xlarge epic phase sizes
goal: 'Epic plans can declare `xsmall` and `xlarge` phases end to end — validated
  and persisted, model-routed through new phase-worker aliases, `#plan`-gated correctly,
  beautifully displayed everywhere phase sizes appear, and documented with clear authoring
  guidance for when to reach for each of the five sizes.

  '
phases:
- id: core
  title: Core phase-size support in sase-core
  depends_on: []
  size: medium
  description: '''Core phase-size support in sase-core'' section: extend the Rust
    PhaseSizeWire enum, deserialization, plan validation, schema CHECK constraint
    + relax migration, and the two schema descriptions to accept xsmall/xlarge.'
- id: aliases
  title: Model-alias ladder and phase-worker routing
  depends_on: []
  size: medium
  description: '''Model-alias ladder and phase-worker routing'' section: rename cheaper/cheapest,
    add cheapest/smart and the xsmall/xlarge phase-worker aliases, retarget the phase-worker
    fallbacks, and wire the new names through the bucket, schema, config example,
    doctor, and completion catalog.'
- id: routing
  title: Python phase-size domain and prompt routing
  depends_on:
  - core
  - aliases
  size: medium
  description: '''Python phase-size domain and prompt routing'' section: add XSMALL/XLARGE
    to the Python PhaseSize enum and route them through work.py size->alias mapping
    and phase_requires_plan, the bead wire (de)serialization, DB load, and the CLI
    --size choices.'
- id: present
  title: Five-size phase presentation
  depends_on:
  - routing
  size: medium
  description: '''Five-size phase presentation'' section: extend phase_size_presentation.py
    with an accessible five-step color ramp and canonical order so every chip, clan
    summary, and the distribution badge render xsmall/xlarge beautifully; refresh
    visual snapshots.'
- id: guidance
  title: Authoring guidance and explanation text
  depends_on:
  - core
  - aliases
  - routing
  size: small
  description: '''Authoring guidance and explanation text'' section: rewrite the epic
    `sase plan validate --explain` prose for all five sizes, remove the model-override
    testing exception in favor of xsmall, and keep it consistent with the Rust schema
    descriptions.'
- id: verify
  title: End-to-end verification
  depends_on:
  - core
  - aliases
  - routing
  - present
  - guidance
  size: small
  description: '''End-to-end verification'' section: rebuild the binding, run just
    check, and exercise a scratch five-size epic plan to confirm validation, #plan
    gating, per-size model routing, and rendered chips all behave.'
create_time: 2026-07-23 17:38:11
status: done
bead_id: sase-8w
---

# Plan: Add `xsmall` and `xlarge` epic phase sizes

## Overview

Epic phases today come in three sizes — `small`, `medium`, `large` — defined redundantly in the Rust core
(`PhaseSizeWire`) and Python (`PhaseSize`), routed to a size-specific `@<size>_phase_worker` model alias, gated for a
`#plan` planning handoff only when `large`, and displayed as colored "chips". This epic adds two bookend sizes, `xsmall`
(the very simplest tasks) and `xlarge` (a rare admission that a task is too big to plan alone), and reshapes the
model-alias ladder that phase sizes route through.

The work spans two repos: the Rust core lives in the linked **sase-core** checkout (open it with
`sase repo open sase-core`; in this workspace it materializes under `sase/repos/linked/sase-core/crates/sase_core`), and
everything else lives in this **sase** repo. Because `sase_core_rs` is a compiled binding, any phase that builds on the
Rust changes must `just install` (which rebuilds the binding from the current sase-core checkout) before `just check`.

### Canonical size → routing table (single source of truth)

Every phase below must conform to this table. It is the authority for model routing, `#plan` gating, and display order.

| size   | order | `#plan`? | phase-worker alias     | resolves to (default provider = Claude)     | chip style                       |
| ------ | ----- | -------- | ---------------------- | ------------------------------------------- | -------------------------------- |
| xsmall | 1     | no       | `@xsmall_phase_worker` | `@cheaper` → `claude/sonnet \| …spark`      | `bold black on #5FD7AF` (mint)   |
| small  | 2     | no       | `@small_phase_worker`  | `@cheap` → `claude/opus@medium \| …gpt-5.5` | `bold black on #87D7FF` (sky)    |
| medium | 3     | no       | `@medium_phase_worker` | `codex/gpt-5.6-sol@high`                    | `bold black on #FFD75F` (gold)   |
| large  | 4     | yes      | `@large_phase_worker`  | `@smart` → `@default` → `claude/opus`       | `bold white on #D75F87` (rose)   |
| xlarge | 5     | yes      | `@xlarge_phase_worker` | `@smartest` → `claude/claude-fable-5 \| …`  | `bold white on #AF5FFF` (violet) |

`small`, `medium`, and `large` keep their existing chip colors; `xsmall`/`xlarge` extend the ramp at the cool and hot
ends. `#5FD7AF` and `#AF5FFF` are xterm-256 palette entries, matching the existing color vocabulary.

### Guidance & wording spec (single source of truth for phases `core` and `guidance`)

Both the Rust schema descriptions (`validate.rs`, phase `core`) and the Python `--explain` prose (`plan_explain.py`,
phase `guidance`) must express exactly these semantics. Keep them mutually consistent; the `verify` phase reads the
combined `sase plan validate --explain` output to confirm it.

- **`xsmall`** — only the very simplest tasks that need almost no reasoning; the canonical example is launching sase
  agents purely to observe their output while testing a sase agent feature. This _replaces_ the old advice to set an
  explicit cheap `model` override for such phases.
- **`small`** — focused work implemented directly.
- **`medium`** — substantial work still implemented directly from its phase description.
- **`large`** — work that needs a separate planning handoff and may itself justify an epic plan.
- **`xlarge`** — used rarely; an admission that the task is too large to plan effectively alone, or a deliberate choice
  to plan part of a feature only after other parts are implemented. Choose it only when fairly confident this phase's
  agent will itself author an epic plan.
- **`#plan`**: appended for `large` and `xlarge` only (correcting the current, stale schema text that claims medium also
  receives `#plan`). `xsmall`/`small`/`medium` implement directly.
- **`model` override**: only when the user's prompt requested a specific model. Explicitly remove the "does not do real
  consequential work / exercises or tests the feature" exception and instead point agents at `size: xsmall` for that
  case. An explicit model still wins over the size-derived alias for every size.

### Constraints & risks

- **This plan's own phases must stay `small`/`medium`/`large`.** The validator that checks this file today does not yet
  know `xsmall`/`xlarge`, so none of the phases below may use them.
- **DB CHECK migration.** The `issues.size` column carries a SQLite CHECK constraint
  `size IN ('small','medium','large')` baked into both the `CREATE TABLE` text and the historical `ADD COLUMN`
  migration. Existing bead databases will _reject_ `xsmall`/`xlarge` inserts until the constraint is relaxed, and SQLite
  cannot alter a CHECK in place — this needs a table-rebuild migration. This is the single trickiest task; see phase
  `core`.
- **Alias rename is a semantics shift, by request.** After the rename, a user config that references `@cheaper` resolves
  to what used to be `@cheapest`. Both names still exist, so configs stay schema-valid; keep the doctor and schema copy
  accurate but do not attempt to auto-rewrite user intent.
- **Cross-repo build order.** Phase `core` commits to sase-core; downstream phases pick it up via `just install`. The
  `depends_on` edges enforce that ordering.

---

## Core phase-size support in sase-core

**Repo:** linked **sase-core** (`sase repo open sase-core`). **Goal:** the Rust core accepts, persists, validates, and
describes `xsmall`/`xlarge`.

Edits (paths under `crates/sase_core/src/`):

1. **`bead/wire.rs`** — add `Xsmall` and `Xlarge` variants to `enum PhaseSizeWire` (with
   `#[serde(rename_all = "snake_case")]` producing `xsmall`/`xlarge`); extend `as_str()`; extend
   `deserialize_option_phase_size` to accept `"xsmall"`/`"xlarge"` and to list all five in its `unknown_variant` error
   slice. Order the variants Xsmall, Small, Medium, Large, Xlarge.
2. **`plan/validate.rs`** — update `validate_phase_size`: the `matches!(value, …)` guard and both diagnostic messages
   ("expected `small`, `medium`, or `large`" / "must be `small`, `medium`, or `large`, found …") to cover all five
   sizes, and the asserted `field_type` string near line 1701 (`"small | medium | large"` →
   `"xsmall | small | medium | large | xlarge"`). Reword the `PHASE_SIZE_DESCRIPTION` and `PHASE_MODEL_DESCRIPTION`
   constants per the **Guidance & wording spec** (all five sizes; `#plan` for large+xlarge; large→`@smart`,
   xlarge→`@smartest`; drop the testing/model-override exception in favor of `xsmall`).
3. **`bead/schema.rs`** — add `xsmall`/`xlarge` to the `size IN (...)` CHECK inside `BEAD_SQLITE_SCHEMA` (the
   `CREATE TABLE`) and inside `size_migration_sql()`. Add a **new relax migration** that rebuilds existing tables whose
   `size` CHECK still lists only the three legacy values: a `needs_size_check_relax_migration(create_table_sql)`
   detector (true when the stored schema contains the size column with `'large'` but not `'xlarge'`) plus SQL that
   recreates `issues` with the widened CHECK and copies rows (standard SQLite create-new/copy/drop/rename dance,
   respecting foreign keys and indexes). Wire the detector/SQL into wherever the other `needs_*_migration` helpers are
   invoked at store-open time (find the caller of `needs_size_migration`/`tier_migration_sql`).
4. **Tests** — update the `schema.rs` migration tests (the `size_migration_sql()` assertion and add a relax-migration
   test), the `wire.rs`/`validate.rs` round-trip and validation tests, and any `PhaseSizeWire` exhaustiveness. Run the
   crate's Rust test suite.

Finish by rebuilding the Python binding in this sase repo (`just install`) so downstream phases build against the new
enum. **Acceptance:** sase-core builds and its tests pass; a phase row with `size='xsmall'`/`'xlarge'` deserializes,
validates, and can be inserted into both a fresh and a pre-existing bead DB.

## Model-alias ladder and phase-worker routing

**Repo:** sase. **Goal:** the model-alias policy exposes the new ladder and the two new phase-worker roles, and every
place that enumerates alias names stays correct.

Edits:

1. **`src/sase/llm_provider/model_alias_policy.py`** — the heart of the change:
   - Rename constant `CHEAPER_MODEL_ALIAS_NAME` value `"cheaper"` → `"cheap"` and `CHEAPEST_MODEL_ALIAS_NAME` value
     `"cheapest"` → `"cheaper"`; keep their _default target strings_ attached to the value each name now denotes (so
     `@cheap` = `claude/opus@medium | codex/gpt-5.5` and `@cheaper` = `claude/sonnet | codex/gpt-5.3-codex-spark`). Add
     a **new** cheapest alias (`"cheapest"`) with default `claude/haiku || codex/gpt-5.3-codex-spark`.
   - Add a **`smart`** alias whose implicit default is `@default`, and **`xsmall_phase_worker`** /
     **`xlarge_phase_worker`** alias-name constants.
   - Retarget the phase-worker fallbacks so the resolved routing matches the canonical table:
     `xsmall_phase_worker → @cheaper`, `small_phase_worker → @cheap`, `large_phase_worker → @smart`,
     `xlarge_phase_worker → @smartest`, `smart → @default` (all `@`-reference fallbacks live in `ROLE_ALIAS_FALLBACKS`);
     move `medium_phase_worker` out of `ROLE_ALIAS_FALLBACKS` and give it a **concrete** default
     `codex/gpt-5.6-sol@high` in `IMPLICIT_ALIAS_TARGETS` (verified to resolve via the `IMPLICIT_ALIAS_TARGETS` branch
     of `model_alias_resolution.py`). Add `cheapest`'s and the renamed `cheap`/`cheaper` concrete targets to
     `IMPLICIT_ALIAS_TARGETS`.
   - Add/adjust `ROLE_ALIAS_DESCRIPTIONS` for `smart`, `cheap`, `cheaper`, `cheapest`, `xsmall_phase_worker`,
     `xlarge_phase_worker`, and refresh the existing phase-worker/`smartest` descriptions so they no longer imply
     "smartest for large" (large now uses `@smart`).
2. **`src/sase/llm_provider/alias_view.py`** — import the two new phase-worker name constants and add them to the
   `phase_worker` bucket `fixed_members` in canonical order (xsmall, small, medium, large, xlarge). Confirm
   `model_alias_kind` / the "known builtin" recognition treats `smart`, `cheap`, and the new phase-worker names as
   builtin (so the doctor does not flag them as legacy/user).
3. **`src/sase/llm_provider/config.py`** — re-export the new name constants alongside the existing ones so
   `alias_view.py`, `model_completion.py`, and other importers resolve them.
4. **`src/sase/xprompt/model_completion.py`** — add `smart`, `cheap`, `xsmall_phase_worker`, `xlarge_phase_worker` to
   `_TRAILING_IMPLICIT_ALIASES` in display order and fix the renamed `cheaper`/`cheapest` entries and the module
   docstring.
5. **`src/sase/config/sase.schema.json`** — update the `builtin` description (line ~1384) to list the new/renamed alias
   names accurately.
6. **`src/sase/default_config.yml`** — update the commented `model_aliases:` example (the `*_phase_worker` assignments,
   the cheap/cheaper/cheapest pool examples, and the `phase_worker` bucket note) to mirror the new ladder.
7. **`src/sase/doctor/checks_config_model_aliases.py`** — confirm the builtin-name recognition and the `phase_worker`
   migration message remain correct against the new name set (medium_phase_worker still the migration target).
8. **Tests** — update the alias/completion/models-panel/directives tests that enumerate the alias set and order (e.g.
   `tests/test_xprompt_model_completion.py`, `tests/test_models_panel*.py`, `tests/_models_panel_helpers.py`).
   **Acceptance:** `just check` green; resolving each `@<size>_phase_worker` yields the canonical-table target/effort.

## Python phase-size domain and prompt routing

**Repo:** sase (depends on `core` for persistence + binding, `aliases` for the new alias-name constants). **Goal:**
phases can carry `xsmall`/`xlarge` and route to the right alias and `#plan`.

Edits:

1. **`src/sase/bead/model.py`** — add `XSMALL = "xsmall"` and `XLARGE = "xlarge"` to `PhaseSize`.
2. **`src/sase/bead/work.py`** — extend `alias_by_size` (`PhaseSize.XSMALL → XSMALL_PHASE_WORKER_MODEL_ALIAS_NAME`,
   `PhaseSize.XLARGE → XLARGE_PHASE_WORKER_MODEL_ALIAS_NAME`) and `phase_requires_plan` to return true for `LARGE`
   **or** `XLARGE` (leaving xsmall/small/medium without `#plan`). Leave `_phase_size` legacy-None normalization as-is.
3. **CLI `--size` choices** — `src/sase/main/parser_bead.py` (both the create and update parsers) add `xsmall`/`xlarge`
   in canonical order.
4. **Bead wire + DB** — `src/sase/core/bead_wire.py` (serializer/deserializer) and `src/sase/bead/db.py` load path
   already delegate to `PhaseSize(str(...))`; confirm they round-trip the new values and add coverage.
5. **Tests** — extend work.py routing tests (size→alias, `#plan` gating for xlarge but not xsmall), wire round-trip, and
   CLI parsing tests. **Acceptance:** `just install` + `just check` green;
   `phase_model_directive_value`/`phase_requires_plan` match the canonical table for all five sizes.

## Five-size phase presentation

**Repo:** sase (depends on `routing`). **Goal:** every place a phase size is shown renders all five sizes beautifully
and in canonical order.

Edits:

1. **`src/sase/phase_size_presentation.py`** — extend `PhaseSizeValue`, `PHASE_SIZE_VALUES` (ordered
   `xsmall, small, medium, large, xlarge`), and `PHASE_SIZE_STYLES` with the mint `xsmall` and violet `xlarge` styles
   from the canonical table. `PHASE_SIZE_CHIP_WIDTH` recomputes to 8 automatically (`" xsmall "`/`" xlarge "` are 8
   wide, matching `" medium "`), so no downstream width changes are needed — confirm this.
2. **Distribution badge** — `src/sase/ace/tui/widgets/artifacts/plans_detail.py` `_epic_phase_sizes()` must count and
   render all five sizes in canonical order (drive it from `PHASE_SIZE_VALUES` rather than a hardcoded
   small/medium/large list).
3. **Chip sites** — confirm the epic clan summary (`src/sase/scripts/sase_clan_summary_epic.py`), shared plan renderer
   (`src/sase/sdd/plan_display.py`), and the TUI chip sites (`plans_rendering.py`, `_agent_plan_section.py`,
   `_agent_bead_section.py`, `plans_detail.py`) all flow through `phase_size_chip` and need no per-value branching —
   they should inherit the new sizes for free; add any missing normalization defaults.
4. **Visual + unit tests** — extend presentation unit tests for the two new chips and the badge order, and refresh the
   ACE PNG snapshots (`just test-visual`; accept intentional changes with `--sase-update-visual-snapshots`).
   **Acceptance:** `just check` and `just test-visual` green; an epic containing all five sizes renders distinct,
   accessible chips and an ordered distribution badge.

## Authoring guidance and explanation text

**Repo:** sase (depends on `core`, `aliases`, `routing`). **Goal:** `sase plan validate --explain` teaches agents when
to use each of the five sizes.

Edits:

1. **`src/sase/main/plan_explain.py`** — rewrite `EPIC_PLAN_EXPLANATION` per the **Guidance & wording spec**: change
   `size: small | medium | large` to the five-value form; add the `xsmall` and `xlarge` paragraphs; state `#plan`
   applies to large+xlarge; and in the `model` paragraph remove the "does not do real consequential work / exercises or
   tests the feature" exception, replacing it with a pointer to `size: xsmall`. Update the embedded example YAML so the
   exercise/observe phase uses `size: xsmall` (and drops its `model: haiku` override), demonstrating the new pattern.
2. **Consistency** — read the landed `validate.rs` descriptions (phase `core`) and keep the prose and schema-table text
   mutually consistent.
3. **Tests** — update any test asserting the explanation text (search for `EPIC_PLAN_EXPLANATION` / `plan_explanation`).
   **Acceptance:** `just check` green; `sase plan validate <file> --explain` prints coherent five-size guidance.

## End-to-end verification

**Repo:** sase (depends on all prior phases). **Goal:** prove the feature works together.

Steps:

1. `just install` then `just check` (and `just test-visual`) in this workspace — all green.
2. Author a scratch epic plan (e.g. `/tmp` or a throwaway `sase_plan_*.md`) with one phase of each size — `xsmall`,
   `small`, `medium`, `large`, `xlarge` — and run `sase plan validate` on it; it must pass now that the validator knows
   all five.
3. Dry-run the phase-worker prompt build (`sase bead work --from-plan …` preview / the render path in
   `cli_work_from_plan_render.py`) and confirm: `#plan` appears only on `large` and `xlarge`; each phase's `%model:`
   directive is the matching `@<size>_phase_worker` alias; and resolving those aliases yields the canonical-table
   targets.
4. Confirm the epic clan summary and Plans detail render the five chips and the ordered distribution badge.
5. Fix any small gaps found and re-run `just check`. **Acceptance:** every check above passes; report the observed
   prompts/rendering. Remove the throwaway scratch plan when done.
