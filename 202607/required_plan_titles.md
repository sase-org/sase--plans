---
tier: tale
title: Require plan titles and show them in agent metadata
goal: Tale and epic plans require a non-empty frontmatter title at every strict validation
  boundary, and every visible Agents-tab SASE PLAN section displays that title through
  the existing responsive cached metadata path.
create_time: 2026-07-16 07:42:00
status: done
prompt: 202607/prompts/required_plan_titles.md
---

# Plan: Require plan titles and show them in agent metadata

## Context and product contract

The authoritative Rust plan validator currently treats `title` as epic-only: the tale schema omits it, a tale can
validate without it, and a supplied tale title is only an inert-field warning. The CLI's expected-schema table and
minimal example are generated from that core schema, while proposal, approval, epic creation, and post-cutover
committed-plan checks reuse the same validation result. Separately, the Agents-tab detail enrichment reads associated
plan frontmatter into an mtime/size-keyed cache, but its `SASE PLAN` summary carries and renders only the goal, tier,
path, and (for epics) phases.

Make frontmatter `title` a required, non-empty string for both authored tiers. Keep frontmatter authoritative: do not
derive a missing title from the Markdown heading or filename. Strict callers must reject a missing, blank, or non-string
title with the normal location-bearing field diagnostics. The existing pre-cutover committed-plan compatibility rule
remains intact; historical or damaged plans that are still discoverable by the TUI should show `unavailable` for the
title rather than hiding the whole section or inventing a value.

## Implementation

1. Update the authoritative validator in the `sase-core` repository so `title` is a common required field in the tale
   and epic schemas, is normalized once for either tier, and is populated in every successfully validated plan. Remove
   the tale inert-field treatment for `title` while leaving phases, ChangeSpec, and bug metadata epic-only. Preserve the
   current wire envelope and schema version unless the serialized shape itself must change; the already-nullable title
   slot can remain compatible while validation guarantees a value on success. Extend Rust schema, diagnostic, normalized
   result, and Python-binding parity tests to prove both tiers accept a valid title and reject missing, empty, and
   wrong-typed titles.

2. Synchronize the Python-facing validation contract and authoring guidance. Update tale fixtures and assertions across
   CLI validation, proposal/approval, committed-plan enforcement, and shared plan-chain tests so they represent the new
   valid document shape. Verify that the human expected-schema table, JSON schema payload, and generated minimal tale
   example all mark and include `title`, and that `sase plan validate --tier tale` now fails when it is absent. Update
   the source `sase_plan` skill's tale template and wording so future agents author titles before validation, cover that
   source in the skill-generation tests, and regenerate/deploy managed skill outputs through the documented
   `sase skill init --force` and chezmoi workflow rather than editing generated skill files directly. Refresh the SDD
   README templates so their plan-frontmatter guidance also names the required title.

3. Carry the title through the existing Agents-tab associated-plan enrichment path. Add an optional title to the cached
   file metadata and public summary, extract and whitespace-normalize it alongside the goal during the deferred
   frontmatter read, and propagate it without adding any I/O or parsing to the hot render path. Preserve all current
   association and visibility rules, including the deliberate omission of the `SASE PLAN` section for phase workers.

4. Render `Title:` as the first field in every visible `SASE PLAN` section, followed by Goal, Tier, and Path. Give
   present titles a clear restrained style, use the existing quiet `unavailable` treatment when metadata is missing or
   unreadable, and widen the shared label column for `Title:` so hanging indents remain aligned. Retain complete
   responsive folding at narrow widths and the section's 80-cell cap; long ASCII and wide-Unicode titles must not be
   truncated.

5. Expand model and renderer coverage for title extraction, cache reuse and invalidation, field ordering and styles,
   unavailable legacy/damaged data, and responsive wrapping. Update only the affected Agents-tab PNG goldens, inspect
   actual/expected/diff artifacts, and accept them only after confirming the added title row is the intentional visual
   change.

## Verification and acceptance

- In `sase-core`, run the focused plan-validator and binding tests plus the workspace Rust test/format checks. Confirm
  the ordered schema for both tiers reports `title` as required and successful normalized tale and epic payloads carry
  it without changing unrelated wire fields.
- In the main SASE repository, run targeted plan validation, proposal/approval, committed-plan, generated-skill,
  associated-plan model, metadata renderer, and visual snapshot tests. Then run `just install` followed by the required
  `just check` full gate.
- Manually exercise one valid and one title-less file for each tier through `sase plan validate`, including human and
  JSON output. A valid associated plan must display its full title in the visible `SASE PLAN` section; a legacy,
  missing, or damaged plan must retain the section and show `Title: unavailable`.
- Confirm plans before the committed-plan schema cutover retain their existing compatibility behavior, while all strict
  validation entry points use the new required-title contract.
