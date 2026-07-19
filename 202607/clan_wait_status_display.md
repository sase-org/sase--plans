---
tier: tale
title: Clan-aware Wait field in the Agents-tab metadata panel
goal: 'When an agent waits on an agent clan, the Agents-tab metadata panel''s Wait:
  field shows every clan member''s live status and makes it explicit that the wait
  covers all current (and future) members of the clan, instead of rendering the clan
  name with the unknown "?" badge.

  '
create_time: 2026-07-19 14:02:06
status: wip
prompt: 202607/prompts/clan_wait_status_display.md
---

# Plan: Clan-aware `Wait:` field in the Agents-tab metadata panel

## Problem

When an agent declares `%wait:<clan>` (e.g. `%w:sase-7g`), the Agents-tab metadata panel renders `Wait: sase-7g ?` — the
clan name followed by the orange unknown-status badge. Two things are wrong with this:

1. The `?` badge is misleading: the clan is known and its members are visible in the same TUI snapshot, but the badge
   lookup misses because clan names are never indexed.
2. Nothing communicates the wait semantics: a clan wait resolves only when **every** member of the clan's newest
   generation is done, and the member set is dynamic (new agents can join the clan while the waiter is parked). The
   field should show the per-member statuses and say clearly that all members must complete.

## Verified: `%wait` on clans is already supported end-to-end

No resolution-engine work is needed. This was verified up front so the plan can stay display-only:

- Directive parsing accepts clan names: `%wait` positional args are free-form names
  (`src/sase/xprompt/_directive_values.py::resolve_wait_agent_args`); no launch-time existence check rejects clan names.
- The wait engine resolves clan names: `WaitDependencyIndexQueries.is_resolved`
  (`src/sase/core/wait_dependency_resolution/_index_queries.py`) consults `clan_candidate` first, which aggregates the
  **newest generation** of the clan and reports done only when **all** members are resolved and done.
- The `wait_checks` chop (`src/sase/scripts/sase_chop_wait_checks.py`) and the runner fast path
  (`src/sase/axe/run_agent_wait.py::_initial_dependencies_resolved`) both go through that same index, so `ready.json` is
  written correctly for clan waits.
- Existing regression tests: `tests/test_clan_wait_dependency.py` (`test_wait_on_clan_requires_every_member`,
  `test_wait_on_clan_uses_newest_generation`).
- The prompt-bar `%wait` autocompletion already offers clans as first-class wait targets
  (`src/sase/ace/tui/widgets/directive_completion.py`, `_TARGET_KIND_ORDER` includes `"clan"`).

## Root cause of the display bug

`_collect_agent_status_buckets` (`src/sase/ace/tui/agent_completion.py`) maps only prompt-referenceable **agent** names
and **family** reference names to status buckets. Clan names never get an entry, so:

- `_append_wait_field` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`) falls back to
  `_UNKNOWN_WAIT_AGENT_GLYPH` (`?`) for the clan name.
- `wait_dependencies_satisfied` (`src/sase/ace/tui/agent_completion.py`) can never return `True` for a clan wait, which
  suppresses the "time-floor countdown" rendering on the agent row
  (`src/sase/ace/tui/widgets/_agent_list_render_agent.py`) after the clan actually completes.

## Design

All changes are presentation-layer Python in this repo. Per the Rust core boundary rule this does not touch `sase-core`:
the wait _resolution_ behavior already lives behind the shared engine and is unchanged; only how already-loaded TUI rows
are summarized for display changes.

### 1. Clan-aware wait-status collection (`src/sase/ace/tui/agent_completion.py`)

Extend the status-bucket collection so clan names resolve like other wait targets, and expose the per-member detail the
metadata panel needs:

- Reuse `_visible_clan_completion_groups(...)` (already in this module) to derive each visible clan's **newest
  generation** and its deduped real member rows. This mirrors the engine's `max(generations)` rule so the display and
  the resolver judge the same member set.
- Flat aggregate entries: after the existing per-row pass, add one entry per clan name to the returned bucket map. The
  aggregate bucket must reflect **clan** semantics (all members done), not the family "one success satisfies"
  precedence: derive it by feeding the member statuses through `aggregate_clan_status`
  (`src/sase/ace/tui/models/_agent_clan.py`) and mapping the result with `status_bucket_for_values`. That yields `Done`
  only when every member is done, `Failed` when any member failed, and the active bucket otherwise — matching how clan
  rows already summarize elsewhere in the TUI.
  - Collision rule: only add a clan entry when the name is not already present from a real agent/family row (clan names
    are reserved by `%clan`, so collisions indicate legacy data; prefer the existing behavior for them).
- Member detail map: add a sibling collector (one shared pass with the aggregate computation — do not walk the agent
  list twice) returning `clan name -> ordered ((member label, bucket), ...)` for every visible clan, where member order
  matches the clan section's member-row order used by `_clan_group_members`. Member labels are the short in-hood form:
  strip the leading `<clan>.` prefix and render as `.<suffix>` (every member is named inside the clan's hood, so this is
  always well-formed); fall back to the full name if a legacy row is not inside the hood.
- Extend `agent_status_buckets_for_app(...)` (or add one sibling helper next to it) so header render call sites can
  obtain both maps from the same `_agents_with_children` / `_agents` snapshot without a second pass over the app state.

Performance constraints (per the TUI perf rules): everything stays pure in-memory over already-loaded rows — no disk
I/O, no subprocess, no new refresh paths. The collection is invoked from the existing list-build and (debounced)
detail-panel paths only; keep it a single pass plus the existing clan-group derivation.

### 2. `Wait:` field rendering (`_agent_display_header_metadata.py`)

Thread the new clan-member map through the header pipeline:

- `build_header_text(...)` (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py`) and
  `append_agent_metadata_fields(...)` gain an optional `clan_wait_member_statuses` mapping parameter (default `None`),
  passed to `_append_wait_field`. The existing `agent_status_buckets` parameter keeps its `Mapping[str, str]` shape so
  current callers/tests keep working.
- Update both call sites that currently fetch buckets via `agent_status_buckets_for_app`:
  `src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py` and `.../_agent_display_hints.py`.

Rendering rules in `_append_wait_field`:

- Non-clan names: unchanged (`name` + single status badge, `?` when unknown).
- Clan names (i.e. names present in the member map): render the clan name in the existing wait value style, then an
  expansion instead of a single badge:

  ```
  Wait: sase-7g (all clan members · 2/4 done: .f0 ✓ · .f1 ✓ · .w0 ▶ · .w1 ⏳)
  ```

  - The `all clan members` label is the explicit signal that the wait covers every member of the clan — including
    members that join after the wait started — so the list shown is a live snapshot, not a fixed dependency list. Keep
    this exact intent even if the copy is tweaked (e.g. `all clan members` vs `every clan member`); it must not read
    like a static agent list.
  - `2/4 done` is the done-count over the currently visible newest generation.
  - Each member renders as its short `.suffix` label plus the standard status badge from `_WAIT_STATUS_BADGES` (reuse
    the existing glyph/style tables; member badges use the same styles as the aggregate badges so the panel stays
    consistent).
  - Members render in clan-section order, separated by `·`. Do not cap the list; the panel wraps. (If a cap is later
    wanted it must say `+N more` — never truncate silently.)

- A clan wait whose clan is not in the visible snapshot (fully archived/cleaned) keeps the current `name ?` fallback.
- Time-floor / runner-slot suffixes (`+ 5m`, `(2m left)`, `runners ≤ N`) are appended after the dependency list exactly
  as today.

### 3. `wait_dependencies_satisfied` correctness for clan waits

No signature change: with the aggregate clan entries from (1) in the flat bucket map,
`all(status_buckets.get(name) == "Done" ...)` becomes correct for clan names (the aggregate is `Done` only when every
member is done, matching the engine). This restores the row-level countdown display for clan waits with a time floor.

## Testing

Extend `tests/ace/tui/widgets/test_agent_display_waiting_warning.py` (reuse its `make_agent` helper patterns; fabricate
clan rows via the same fields `_visible_clan_completion_groups` reads — `agent_clan`, `agent_clan_generation`,
container/member flags):

- Bucket collection: clan aggregate entries — all members done → `Done`; any failed → `Failed`; active mix → active
  bucket; only the newest generation counts; agent/family entries win name collisions.
- `Wait:` rendering: clan wait shows the `all clan members` label, the done-count, and one badge per member in order;
  unknown clan still renders `?`; mixed `waiting_for` (clan + plain agent) renders the plain agent unchanged; time-floor
  suffix still appends after the expansion.
- `wait_dependencies_satisfied`: `True` once the clan aggregate is `Done`, `False` while any member is active.

Run `just install` then `just check` (this workspace may have stale deps). Also run `just test-visual`: the metadata
panel is covered by PNG snapshot suites — if any golden legitimately changes because of the new Wait rendering, inspect
`.pytest_cache/sase-visual/` artifacts and update goldens intentionally with `--sase-update-visual-snapshots`.

## Non-goals

- Tribe waits (`%wait:@tribe`): these resolve against "first entity launched after the waiter", which a visible-rows
  snapshot cannot faithfully mirror; they keep today's rendering. Worth a follow-up bead if the `?` badge on tribe waits
  bothers anyone.
- Family-member expansion in the `Wait:` field: family waits already show a correct single badge via family reference
  names; per-member expansion is clan-only for now.
- Wait modal / edit-wait flows, the engine, chop, and `%wait` parsing: verified working, unchanged.
- No Rust core (`sase-core`) changes: presentation-only per the core-boundary litmus test.

## Risks / notes

- The member list can be long for large clans; the panel wraps text, and the format keeps per-member cost to a short
  label plus one glyph. No disk or O(archive) work is added to render paths.
- Display aggregates come from the _visible_ snapshot while the engine judges the _artifact_ index; brief divergence
  (e.g. filtered panels) is acceptable — the field is informational, and the unknown fallback still covers names with no
  visible rows.
