---
tier: tale
title: Uncapped default query for Artifacts commits
goal: Keep the 24-hour Artifacts Commits default query free of an implicit 40-row
  cap while retaining explicit limits, truthful coverage reporting, and responsive
  navigation.
create_time: 2026-07-21 13:06:51
status: done
---

- **PROMPT:** [202607/prompts/uncap_default_commits_query.md](prompts/uncap_default_commits_query.md)

# Plan: Uncap the default Artifacts Commits query

## Context and intended contract

The bundled persistent query already renders as `sidecar:false since:24h`, but the commit-query value model assigns 40
when no `limit:` token is present. Once a 24-hour result reaches that cap, the pane truthfully exposes the hidden value
as `limit:40`, so the effective default is still bounded even though the configuration string does not contain the
token.

Make an omitted `limit:` mean unlimited for the Commits query language. This keeps the bundled query and its schema
default as the literal `sidecar:false since:24h`, sends an unlimited collection request for that bounded time window,
and prevents the persistent row from acquiring `limit:40`. Apply the rule consistently to startup configuration and live
edits rather than adding a bundled-query-only override; the same visible query must always parse to the same state when
ACE starts, when the user focuses it, and when the user submits it again.

Explicit positive limits remain supported and opt-in: `limit:40` and other `limit:N` values continue to bound
collection/display, surface capped lower-bound status when truncation is possible, and remain in canonical query text.
Keep accepting `limit:all` as an explicit spelling of the now-default unlimited state, canonicalizing it to the same
token-free representation as an omitted limit. Do not change the independent `sase vcs log` CLI limit contract.

## Query, collection, and UI behavior

- Update the Commits filter value/parser default from 40 to the existing unlimited sentinel used by the pane's
  collection adapter. Align canonical token rendering and filter chips so the unlimited default does not emit
  `limit:all` or a synthetic numeric limit, while explicit positive limits still round-trip and display.
- Preserve the existing collection pipeline: date/author/repository filters continue to be pushed to providers on the
  background worker, subject/exclusion filters still request enough candidates for correct in-memory matching, and the
  backend's established zero-to-unlimited translation remains the only unbounded-collection mechanism. No new VCS,
  config, or filesystem work should reach render, focus, or key-event paths.
- Keep truncation reporting honest. A default 24-hour collection containing more than 40 commits should show every match
  with an exact count and no active-limit annotation. A query with an explicit positive cap should retain the current
  `N+ matches · capped`, persistent `limit:N`, and header treatment. Provider/aggregate truncation metadata must not
  invent a query limit when the active query itself is unlimited.
- Retain lazy first activation, cache/snapshot coverage, relative-window re-anchoring, stable selection, sidecar and
  project-scope toggles, fetch/refresh behavior, and the current visual layout. This is a Python ACE query/presentation
  change; the Rust aggregation API already supports unlimited requests, so no duplicate core behavior or wire change is
  needed.

## Documentation and regression coverage

- Update the ACE guide and configuration reference to say that Commits queries are uncapped unless they include an
  explicit positive `limit:N`, and that the bundled 24-hour query therefore has no row cap. Retain guidance for using
  numeric limits on deliberately broad searches and clarify that `limit:all` is an accepted but canonically omitted
  synonym.
- Extend pure query tests to prove empty and bundled queries resolve to unlimited, token-free canonical text; explicit
  numeric limits parse, serialize, and round-trip; and `limit:all` normalizes to omission without changing semantics.
  Update property coverage and any default-limit constants/names so tests describe the new contract rather than merely
  changing an expected number.
- Extend configuration and Commits-pane coverage to prove the unchanged bundled string drives a first collection with
  `limit=0`, a 24-hour result above 40 rows is complete and exact, and an explicit `limit:40` still truncates and
  exposes its active cap. Preserve tests for custom default queries, invalid-query fallback, cache directionality,
  relative time re-anchoring, and provider truncation metadata.
- Refresh only the Commits PNG goldens whose intended default-limit chrome changes. Keep explicit-limit capped-state
  visual coverage by seeding that fixture with `limit:40`, while ensuring a normal bundled-default snapshot asserts the
  row remains `sidecar:false since:24h` with no limit token.

## Verification and performance

Run `just install` before repository checks. Run the focused query, configuration, collection, Commits pane/widget,
help, and Commits visual suites, then run `just test-visual` for intentional golden changes and the mandatory
`just check`.

Removing the cap increases the number of timeline options that may be rendered inside the bounded 24-hour window. Update
the existing Artifacts navigation benchmark so its 200-commit fixture waits for and exercises the full uncapped Commits
result instead of stopping at 40. Capture the benchmark before and after the implementation where practical and confirm
Commits navigation remains below the documented 16 ms key-to-paint p95 target. If the larger result reveals a
regression, optimize the established in-memory/list rendering path without moving collection onto the UI thread or
reintroducing a hidden default cap.

## Acceptance criteria

- A fresh Artifacts → Commits pane displays `sidecar:false since:24h`, collects all matching commits, and never adds
  `limit:40` solely because more than 40 commits exist in the window.
- Omitting `limit:` and entering `limit:all` both produce the same unlimited canonical query, while an explicit positive
  `limit:N` remains visible and enforces the requested cap.
- Counts and cap indicators remain truthful for both unlimited and explicitly bounded queries, without regressions to
  lazy loading, refresh/re-anchoring, cache coverage, selection, or sidecar/project filtering.
- Focused tests, intentional visual snapshots, the 200-row Artifacts navigation benchmark, and `just check` all pass.
