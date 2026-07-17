---
tier: tale
title: Phase-description authoring hints in sase plan validate
goal: 'sase plan validate instructs plan-authoring agents to give every epic phases[]
  entry a description that names the phase''s section in the plan body and briefly
  summarizes it, instead of pointing at the plan file that sase bead show already
  displays.

  '
create_time: 2026-07-17 10:48:35
status: done
prompt: 202607/prompts/phase_description_hints.md
---

# Plan: Phase-description authoring hints in `sase plan validate`

## Context

- All plan schema and content rules live in Rust: `sase-core` `crates/sase_core/src/plan/validate.rs`, reached through
  the `plan_validate` / `plan_frontmatter_schema` bindings. The Python side (`src/sase/main/plan_validate_handler.py`,
  `src/sase/main/plan_validate_render.py`) only renders wire results. Open the Rust repo with `sase repo open sase-core`
  (linked repo).
- Each epic `phases[]` entry becomes a phase bead; `phases[].description` becomes the bead description
  (`src/sase/bead/epic_from_plan.py`). When omitted, the fallback is "Phase `<id>` in approved epic plan `<ref>`."
  (`src/sase/bead/phase_description.py`).
- `sase bead show` on a phase bead already prints an EPIC PLAN section with the plan path
  (`src/sase/bead/cli_query.py:141-155`), so a description that only points at the plan file is redundant. What the bead
  consumer actually lacks is which _section_ of the plan covers the phase and what that section says.
- Today the only authoring hint is the `phases[].description` schema text ("Phase bead description. A deterministic plan
  pointer is generated when omitted."), and the human renderer prints the schema table **only when validation fails**. A
  plan that validates cleanly surfaces no guidance at all. Warning diagnostics, by contrast, always print (all
  diagnostics render even on success, and warnings never flip `ok` to false) and appear in `--json` output.

## Design

Deliver the guidance through three small, mutually reinforcing surfaces, each kept to a sentence or two — every extra
token in agent context hurts:

1. **Schema field text + example (Rust)** — reaches the failure path and all `--json` consumers. Replace the
   `phases[].description` spec description with, approximately: "Phase bead description: name this phase's section in
   the plan body and briefly summarize that section. Do not reference the plan file itself; `sase bead show` already
   displays it." Replace the example value with one that models the shape, e.g.:
   `"'Core validator' section: build the shared validation engine and its JSON wire."`
2. **New warning when a phase omits `description` (Rust)** — the only surface that reaches the success path. In
   `validate_phases`, for each phase mapping without a `description`, push a `warning` diagnostic (code
   `phase-description-missing`, field path `phases[<n>].description`, line of the phase entry) with a concise message
   such as: "phase `<id>` has no `description`; add one naming its plan-body section with a brief summary of that
   section". Warnings must not change `ok`, the validated-plan payload, or `PLAN_WIRE_SCHEMA_VERSION` (stays 2 — this is
   additive).
3. **`sase_plan` skill template (Python)** — reaches agents before authoring, saving a validate round-trip. In
   `src/sase/xprompts/skills/sase_plan.md`, replace the "A phase's `description` and `model` are optional." sentence
   with guidance that each phase should carry a `description` naming the phase's plan-body section plus a brief summary
   of it, and must not reference the plan file itself since `sase bead show` already displays it. Keep it to one or two
   sentences; `model` stays optional as written.

Wording above is directional: the implementer may tighten phrasing but must keep the two semantic points (reference the
phase's plan **section**, not the plan file; summarize that section's contents) and keep it terse.

## Implementation steps

1. **sase-core** (open via `sase repo open sase-core`; follow that repo's AGENTS.md conventions):
   - Update the `phases[].description` field spec text and example in `crates/sase_core/src/plan/validate.rs`.
   - Emit the `phase-description-missing` warning in `validate_phases` for phases lacking `description` (skip phases
     that already failed to parse as mappings; an explicitly empty string already errors via existing optional-string
     validation — confirm and leave that behavior alone).
   - Tests in the same file's test module: warning emitted with expected code, field path, line, and severity for a
     description-less phase; no warning when `description` is present; `ok` stays true and the plan payload is
     unchanged; update existing fixtures/assertions that assume description-less epics validate with zero diagnostics;
     keep the `phases[].model` description assertion pattern as precedent for asserting the new spec text if useful.
   - Run that repo's fmt/lint/test checks and commit there.

2. **sase repo — skill template**:
   - Edit `src/sase/xprompts/skills/sase_plan.md` per Design item 3.
   - Regenerate deployed skills: `sase skill init --force`, then `chezmoi apply` (generated-skill workflow; never edit
     deployed SKILL.md files directly).

3. **sase repo — test hardening for the future binding release**:
   - The pinned `sase-core-rs>=0.5.0,<0.6.0` wheel will pick up the new warning on its next release; no Python code
     change is needed to surface it (the renderer already prints warnings and the JSON envelope already carries
     diagnostics). Guard against latent breakage now: audit epic fixtures in tests that assert exact diagnostic lists,
     empty-diagnostics, or warning counts (`tests/test_plan_validate.py`, `tests/plan_validation_helpers.py`, plan
     approve / committed-validation tests) and add phase `description` values wherever description-absence is not the
     point of the test. Keep assertions that specifically exercise `description is None` rehydration.
   - Confirm nothing gates on warning-free validation: `src/sase/sdd/committed_plan_validation.py` and the epic
     consumption boundary only fail on `is_error` diagnostics.

4. Run `just check` in the sase repo.

## Testing

- Rust unit tests cover the new warning and spec text (step 1).
- Python suite runs against the currently released binding, so behavior is unchanged today; the fixture hardening in
  step 3 keeps the suite green when the new sase-core-rs 0.5.x wheel lands.
- Manual sanity: author a scratch epic plan with and without phase descriptions and run
  `sase plan validate <file> --tier epic` (plus `--json`) against a locally built binding if available; otherwise rely
  on Rust tests.

## Risks and non-goals

- Non-goal: validating description _content_ (no heuristic check that the text actually names a section) — guidance
  only.
- Non-goal: changing the `generated_phase_description` fallback text or `sase bead show` output.
- Risk: warning noise when older description-less epic plans are re-validated. Acceptable: warnings never fail
  validation or gates, and the nudge is exactly the behavior change requested.
