---
tier: tale
title: Warn when custom model aliases shadow builtin aliases
goal: 'The ACE Models panel immediately and persistently identifies builtin aliases
  configured under llm_provider.model_aliases.custom and directs users to move those
  overrides to llm_provider.model_aliases.builtin, without changing alias resolution
  or configuration automatically.

  '
create_time: 2026-07-21 09:23:31
status: wip
prompt: 202607/prompts/warn_custom_builtin_aliases.md
---

# Plan: Warn when custom model aliases shadow builtin aliases

## Context and intent

Model alias resolution deliberately merges `model_aliases.custom` over `model_aliases.builtin`. The existing doctor
check reports a builtin alias name placed in the custom map, but the Models panel currently presents the winning value
as an ordinary configured alias. This makes an effective but misplaced override easy to miss, especially for `coder` and
phase-worker aliases hidden inside collapsed builtin buckets.

Keep the change presentation-only. Do not alter merge precedence, schema validation, alias resolution, persistent edit
behavior, or the user's config. The warning should cover valid custom entries whose alias name is already classified as
a builtin (`default`, a role alias, or a registered `<provider>_coder` alias). Legitimate user-defined aliases in
`custom`, and ordinary builtin overrides already stored in `builtin`, must remain calm. Other doctor-only migration
problems remain outside this focused warning.

## Alias warning state

Extend the display-ready alias model in `sase.llm_provider.alias_view` with a clear semantic predicate for “custom entry
shadows a builtin alias,” derived from the alias snapshot's existing `configured_source == "custom"` and non-`user`
kind. This reuses the same builtin classification already used by the doctor instead of introducing a second hard-coded
alias list. Add a bucket aggregate that exposes the affected member names/count so collapsed `coders` and `phase_worker`
rows cannot hide the problem.

Derive all panel warnings from the alias views already built for the panel. Opening or navigating the panel must not
invoke the full doctor check, re-read configuration, stat files, or perform any other synchronous I/O on the Textual
event loop. A normal row refresh should naturally recompute the warning state from the new view snapshot so repaired
config stops rendering as problematic.

## Models panel warning experience

When the panel mounts with one or more affected aliases, emit one warning-level toast for that panel opening. List the
affected `@alias` names in deterministic order and give actionable singular/plural guidance: move each custom entry's
`model` value from `llm_provider.model_aliases.custom` to `llm_provider.model_aliases.builtin`. Do not emit a toast for
a clean config, and do not repeat the opening toast during navigation or internal row refreshes.

Also make the condition persistent and local in the panel rather than relying only on the transient toast:

- Mark every affected alias row with a warning glyph/style while preserving the existing configured/reference or
  temporary-override state. The warning must remain visible when a temporary override is active.
- Propagate a warning marker/count to a collapsed bucket containing affected members, and retain the marker when the
  bucket is opened or closed.
- When an affected alias or bucket is highlighted, prioritize a concise warning in the two-line description strip that
  names the affected alias or aliases and says to move them from `llm_provider.model_aliases.custom` to
  `llm_provider.model_aliases.builtin`. Normal alias descriptions and bucket model summaries remain unchanged when there
  is no warning.

Use the existing fixed-width row layout and description area; avoid a new always-present banner or data-dependent modal
height. Update the Models-panel documentation in `docs/ace.md` to describe the opening warning, row/bucket indicator,
and recommended config location.

## Coverage and verification

Add focused data-layer tests for builtin role and provider-coder aliases sourced from `custom`, bucket aggregation, and
negative cases for real custom aliases and correctly sourced builtin overrides. Add rendering tests proving the alias
and bucket indicators appear, the exact config-field advice is shown, and an active temporary override does not suppress
the warning. Add mounted-panel tests for one deterministic warning toast on open, no toast for clean views, and warning
visibility through bucket navigation and refresh.

Add a dedicated Models-panel PNG snapshot for the warning state, including a misplaced builtin member inside a collapsed
bucket, while keeping the existing calm and override snapshots unchanged. Regenerate only the intentional new or changed
golden and inspect the visual artifacts for clipping at the production 120x40 size and the existing narrow-width
geometry coverage.

Before handoff, run `just install`, the focused alias-view and Models-panel unit tests, the dedicated Models-panel
visual snapshot test, and finally the required `just check` suite. Confirm the final diff contains no alias-resolution,
schema, default-config, or automatic config-migration changes.
