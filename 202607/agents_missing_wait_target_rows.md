---
tier: tale
title: Surface missing wait targets on Agents-tab rows
goal: 'WAITING agent rows reveal at a glance when at least one agent dependency does
  not exist yet, using an accurate, compact, and visually consistent indicator that
  clears as soon as the dependency becomes known.

  '
create_time: 2026-07-21 11:38:04
status: wip
prompt: '[202607/prompts/agents_missing_wait_target_rows.md](prompts/agents_missing_wait_target_rows.md)'
---

# Plan: Surface missing wait targets on Agents-tab rows

## Product intent and visual contract

The Agents tab already explains wait dependencies in the selected agent's metadata: each known target receives its
status glyph and each target absent from the current agent namespace receives an amber `?`. Bring that signal into the
list without duplicating the dependency list or competing with the row's identity hierarchy.

Render one amber, bold `?` inside the existing purple WAITING status chip, immediately after `WAITING` and before any
runner-slot, duration, or absolute-time annotation. A normal row remains `build (WAITING)`; a row with one or several
missing agent targets becomes `build (WAITING ?)`. Combined states remain readable as, for example,
`build (WAITING ? +5m)`. Reuse the detail panel's glyph and warm-amber style so the compact marker has an immediate,
learnable relationship to the per-target explanation. Keep the marker singular regardless of the missing-target count:
the row answers “does this wait reference something that does not exist?”, while the metadata panel remains the place to
identify exactly which names are missing.

This marker means only “at least one named agent dependency is absent.” Do not show it for a known dependency that is
running, waiting, failed, stopped, or done; bead-only, time-only, and runner-slot-only waits; unavailable status data;
or non-WAITING rows with stale wait metadata. Do not add a legend, warning toast, count, animation, or configurable
color. The existing selected-agent metadata supplies the drill-down and keeps the list calm.

## Authoritative missing-target semantics

Add a pure shared wait-classification helper alongside the existing completion/wait status aggregation. It should use
`wait_display_agent()` so synthetic family/clan/root rows inherit the same effective wait source already used by row
timers and metadata. Given the already-built status map, return the ordered missing names while preserving a distinct
“status snapshot unavailable” result. A target is missing only when a usable map exists and the exact wait name is not
one of its keys; do not introduce fuzzy, prefix, filesystem, or process-based existence checks.

Continue building that map from the app's full `_agents_with_children` snapshot, with the current `_agents` and local
list fallbacks. This is important for folded children and tribe-split panels: an agent in another panel still exists.
Retain the existing prompt-referenceable namespace and precedence rules, including direct agent names, family aliases,
and visible clan aggregates. Any known status proves existence, even when it does not satisfy the wait. Refactor the
detail `Wait:` badges to consume the same missing-name classification when deciding where `?` appears, so the row-level
summary and per-target detail cannot drift semantically.

The classifier must be in-memory and side-effect free. It must not stat, glob, read artifacts, invoke subprocesses, or
otherwise add work to the Textual render/event path.

## Row rendering, refresh, and cache integration

During each normal AgentList build, compute the full status map once, derive both the existing “all dependencies done”
state and the new “one or more dependencies missing” state for each row, and pass the latter into the pure row
formatter. Append the shared amber glyph only in the formatter's `WAITING` branch. Store the derived flag in the row
render context so selective and periodic clock patches reproduce the same row without rescanning all agents per patched
row. Normal agent refreshes rebuild the context, making the marker disappear when a formerly missing dependency is
launched and appear when wait metadata begins referencing an absent name.

Thread the flag through the memoized formatter and include it explicitly in `agent_render_key`; a cache hit must never
reuse a pre-marker row after the classification flips. Preserve the existing alignment-width guard: a selective update
that cannot accommodate the extra cells should fall back to the established full rebuild, and a full build should size
group rules and runtime suffix alignment from the marked text. Do not add another refresh path or force broad rebuilds
solely for the indicator.

Centralize the missing-wait glyph/style with the other shared Agent-list visual constants and use it in both row and
metadata rendering. Keep selection bolding, tree gutters, mark/jump prefixes, family count chips, unread styling, and
right-aligned activity/runtime suffixes unchanged.

## Verification and acceptance coverage

Add focused classifier tests for a partially missing dependency list, all-known dependencies across nonterminal
statuses, a missing target with another dependency done, multiple missing names, an unavailable map, and effective wait
data supplied by `wait_display_source`. Preserve and extend family/clan aggregation coverage so a valid aggregate name
is never mislabeled missing, including when the known target is outside the currently rendered panel.

Add row-rendering tests that assert the exact compact text and amber bold span for `WAITING ?`, one marker for several
missing targets, absence for every non-agent wait kind and non-WAITING status, and stable ordering with runner-slot,
relative-duration, and absolute-time annotations. Add build/cache coverage proving that the render key changes with the
derived flag and that a normal refresh removes the marker when the target appears without leaving a stale cached row.
Retain detail-panel assertions that each missing name receives its own `?` and known targets keep their status badges.

Create a dedicated Agents-tab PNG snapshot using a selected WAITING agent whose dependency set includes done, running,
failed, and absent targets. The snapshot should show the compact row marker and the detailed per-target badges together,
making color, spacing, selection treatment, hierarchy, and list/detail consistency reviewable at the standard terminal
size. Inspect the rendered actual/expected/diff artifacts before accepting the intentional golden update.

After implementation, run `just install`, the focused wait-classification, AgentList rendering/cache, detail-header, and
PNG snapshot tests, then `just test-visual` and the required `just check`. No default configuration, keymap,
persistence, wait execution, or notification behavior should change.

## Risks and boundaries

The main correctness risk is conflating “not done” with “does not exist”; membership in the authoritative status map,
not its bucket value, is the boundary. The main freshness risk is omitting the derived flag from cache/context state;
explicit flip tests cover that. The main performance risk is rebuilding the full status namespace from every row patch;
compute it once per list build and reuse the result instead. The main visual risk is making WAITING rows noisy; a single
shared `?` inside the status chip keeps the signal subordinate to the row name while remaining scannable.

This tale does not change how waits resolve, attempt to launch missing dependencies, expose missing names in filters or
group banners, or generalize the indicator to missing beads and other resources.
