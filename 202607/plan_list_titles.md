---
tier: tale
title: Readable titles in sase plan list
goal: 'Make every plan pipeline row easy to recognize by surfacing its canonical frontmatter
  title in human and JSON output without sacrificing paths, compatibility with historical
  plans, or inventory performance.

  '
create_time: 2026-07-16 08:07:36
status: done
prompt: 202607/prompts/plan_list_titles.md
---

# Plan: Readable titles in `sase plan list`

## Context

Plan titles are now required and normalized by the authoritative plan validator, and the Agents view already uses them
to identify an associated plan. The `sase plan list` inventory still presents only paths, however, so users must decode
filenames to distinguish proposed, approved, and inferred-rejected plans.

The list command is a Python-owned inventory assembled from three sources: pending plan notifications, approved-agent
metadata, and archived proposal files. Each row already resolves to a concrete Markdown plan path and reads its
frontmatter to determine the tier. This feature should extend that existing best-effort metadata path; it does not need
a new CLI option or a Rust binding.

## Product and presentation design

- Show the canonical frontmatter `title` for proposed, approved, and rejected rows. Normalize embedded whitespace to one
  display line, matching the Agents view, while preserving Unicode and meaningful punctuation.
- Replace the human table's `Plan path` column with a `Plan` cell that uses clear visual hierarchy: the title is the
  prominent first line and the existing shortened path is a dim secondary line. Let both wrap instead of truncating
  them, so narrow terminals lose neither identity nor provenance.
- Preserve the compact, status-colored panels and all operational columns, especially the proposed-plan ID used by
  approve/reject commands. Do not add a separate title column that squeezes IDs, agent/project, model, tier, or age.
- When a historical, malformed, wrong-typed, missing, unreadable, or blank-title plan is encountered, keep the row and
  path visible and render a subdued `title unavailable` label. Listing inventory must remain best-effort rather than
  becoming another validation gate.
- Reclaim useful width in the rejected panel by showing its shared inference explanation once as a dim panel note
  instead of repeating the same note in every row. Keep the per-row note in JSON for compatibility and provenance.
- Add no flag: titles become the default because they are the primary human identity of a plan. Existing filters,
  limits, ordering, deduplication, section counts, and empty states remain unchanged.

## Data and reliability design

- Add a nullable `title` to each public inventory row model and therefore to each proposed, approved, and rejected JSON
  object. A normalized canonical title is a string; unavailable metadata is JSON `null`. Keep all existing fields and
  summary semantics stable.
- Introduce one best-effort plan metadata read that derives both `title` and `tier` from the same parsed frontmatter
  snapshot. Reuse the established plan frontmatter parser and tier normalizer rather than implementing a title-only YAML
  scanner.
- Update all three collectors to consume that shared metadata result. Avoid a second file read per row, preserve the
  current scan limits and early exits, and do not allow an I/O, UTF-8, YAML, or schema-shape problem in one plan to
  abort the full inventory.
- Keep title collection independent of strict validation. New plans should be valid, but archived history predates the
  title requirement and must still be inspectable.
- Do not infer canonical titles from filenames or headings in the machine payload. Such guesses look attractive but are
  ambiguous; the path already supplies a reliable fallback identity in human output.

## Implementation areas

- Extend the immutable inventory row types and stable JSON projection in `src/sase/main/plan_inventory_models.py` and
  `src/sase/main/plan_inventory.py`.
- Centralize the single-read title/tier extraction alongside inventory path metadata in
  `src/sase/main/plan_inventory_paths.py`, then thread it through `src/sase/main/plan_inventory_collectors.py` for every
  pipeline status.
- Add a reusable Rich plan-cell renderer and update the three status tables in `src/sase/main/plan_inventory_render.py`.
  Consolidate the rejected inference note there without changing its JSON representation.
- Update inventory helpers and tests to author titled plan fixtures by default, while retaining explicit legacy and
  malformed fixtures for fallback coverage. Keep the scope on `sase plan list`; the Artifacts Plans pane and plan search
  already have their own title contracts and need not change.

## Verification

- Model and JSON tests cover title propagation for proposed, approved, and rejected rows; whitespace normalization;
  nullable fallback behavior; and preservation of existing fields, filters, counts, limits, ordering, and deduplication.
- Collector tests cover missing frontmatter, blank and non-string titles, malformed YAML, invalid UTF-8 or
  unreadable/disappearing files, and confirm title extraction does not add duplicate reads or bypass approved scan caps.
- Rich rendering tests exercise all three non-empty sections at narrow and wide console widths. Assert that canonical
  titles lead, paths remain visible, unavailable titles are honest and subdued, proposed IDs remain usable, and the
  rejected inference note appears once rather than once per row.
- Run the focused inventory, rendering, parser/handler, and Artifacts Plans consumer tests to catch row-model
  compatibility. Manually inspect human output at representative narrow and normal widths and verify `--json` emits
  `title` as a string or `null` as designed.
- Because the implementation changes files in the SASE repo, run `just install` followed by the required full
  `just check` gate.

## Acceptance criteria

- A user can identify every titled plan from the first line of its row without reading or mentally expanding its
  filename.
- The shortened path remains visible for navigation and debugging, and pending proposal IDs remain easy to copy into
  approval or rejection commands.
- Human output stays readable across terminal widths and avoids repeated explanatory clutter.
- JSON consumers receive a stable `title` field on every row without losing or changing existing fields.
- One bad or historical plan cannot make `sase plan list` fail, and title support does not regress the command's
  bounded-scan behavior.

## Out of scope

- New title filtering, sorting, searching, or CLI switches.
- Revalidating or rewriting historical archived plans.
- Changing the strict title schema, proposal lifecycle, approval semantics, or Rust plan-search wire format.
