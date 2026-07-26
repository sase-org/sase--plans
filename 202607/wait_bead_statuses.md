---
tier: tale
title: Show per-bead status badges in the agent detail Wait field
goal: The agent metadata panel's `Wait:` field renders a status badge beside every
  bead id an agent is waiting for (a checkmark for closed beads), resolved off the
  event loop through a bounded TTL cache so the TUI stays responsive.
create_time: 2026-07-26 09:04:50
status: wip
---

- **PROMPT:** [202607/prompts/wait_bead_statuses.md](prompts/wait_bead_statuses.md)

# Plan: Per-bead status badges in the agent detail `Wait:` field

## Problem

The agent detail panel already annotates _agent_ wait targets with status badges, but bead wait targets are rendered as
a bare comma-joined list with no status information:

`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`, in `_append_wait_field`:

```python
if wait_agent.waiting_for_beads:
    if appended_dependency_names:
        text.append(" + ", style=_WAITING_VALUE_STYLE)
    text.append("beads: ", style="dim #AF87FF")
    text.append(
        ", ".join(wait_agent.waiting_for_beads),
        style=_WAITING_VALUE_STYLE,
    )
    appended_dependency_names = True
```

So a panel showing `Wait: beads: sase-9r.2` gives no hint that `sase-9r.2` is already closed and the wait is therefore
satisfied. Agent wait targets a few lines above already get this treatment via `_append_wait_status_badge()`; beads
should match.

## Outcome

`Wait: beads: sase-9r.2 ✓` — every bead id in the list is followed by a badge for that bead's live status, using the
same glyph/color vocabulary the panel already uses for agent wait targets.

## Design constraints

Bead-store reads touch the filesystem and the Rust bead binding, so they must never run on the Textual event loop, and
must not be repeated per render. Three existing facts make this straightforward:

1. **There is already an off-event-loop enrichment lane for exactly this kind of data.** `build_detail_header_summary()`
   in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_summary.py` runs only inside a worker thread
   (`AgentDisplayAgentWorkerMixin.start_agent_detail_header_enrichment` → `run_worker(enrich_task, thread=True)` in
   `_agent_display_async_agent.py`) and its result is cached per agent identity and handed to
   `build_header_text(..., summary=...)`. The existing `bead_display` field on `DetailHeaderSummary` is precedent: bead
   metadata already flows through this lane.
2. **A module-level TTL cache pattern for bead lookups already exists.** `src/sase/ace/tui/models/agent_bead.py` defines
   `_BeadDisplayCache` (TTL for hits, longer TTL for misses, `RLock`, bounded LRU eviction at 256 entries). The new work
   mirrors it rather than inventing a new caching approach. This matters because the detail-header summary cache TTL is
   `DIFF_CACHE_TTL_SECONDS == 1.0`, so without a second, longer-lived cache the store would be reopened roughly once per
   second for a selected waiting agent.
3. **The canonical bead store is already kept fresh by a chop.** `src/sase/scripts/sase_chop_bead_store_refresh.py`
   refreshes canonical bead stores for projects that have live bead waiters (every 30s). The TUI can therefore read that
   store passively and must never call `refresh_bead_store()` or any git-syncing path itself.

Correctness anchor: the resolver must read the **same** store that decides whether the wait actually clears.
`src/sase/scripts/sase_chop_wait_checks.py` resolves bead waits with `closed_bead_ids_for_project(project_name)`, which
reads `canonical_beads_dir_for_project(project)`. Using that same canonical store guarantees a rendered checkmark means
"this bead no longer blocks the agent" rather than "some store somewhere has it closed".

Rust core boundary: bead reads already route through `sase_core_rs` (`BeadProject.show()` →
`sase.core.bead_read_facade.show()` → the `bead_show` binding). The new code is a thin Python adapter that composes
existing Rust-backed reads, directly alongside the existing `closed_bead_ids_for_project()` adapter in the same module.
No `sase-core` change is required.

## Implementation

### 1. Batched status read helper

In `src/sase/bead/store_locator.py`, add a sibling to `closed_bead_ids_for_project()`:

```python
def bead_statuses_for_project(
    project: str, bead_ids: Iterable[str]
) -> dict[str, str] | None:
    """Return requested bead IDs mapped to status values, or ``None``.

    ``None`` means the project's canonical bead store was unavailable. IDs that
    do not exist in the store are omitted from the returned mapping.
    """
```

Behavior:

- Resolve `canonical_beads_dir_for_project(project)`; return `None` when it is `None`.
- Open the store **once** with `open_bead_project_for_beads_dir(beads_dir)` as a context manager and call
  `bead_project.show(bead_id)` per requested id, catching `KeyError` (omit that id) and letting the outer
  `except Exception` return `None`, matching the existing fail-closed style of `closed_bead_ids_for_project()`.
- Map each found issue to `issue.status.value` (`"open"`, `"claimed"`, `"in_progress"`, `"closed"` from
  `sase.bead.model.Status`).
- Deduplicate the input ids while preserving no ordering requirement (the caller re-orders).

Point `show()` lookups are used rather than `list_issues(...)` so cost scales with the (tiny) number of waited-for
beads, not with store size.

### 2. TTL-cached TUI resolver

New module `src/sase/ace/tui/models/agent_wait_beads.py`, modeled on `agent_bead.py`:

- `WaitBeadStatusCacheKey = tuple[str, str]` — `(project_key, bead_id)`.
- A `_WaitBeadStatusCache` class with the same shape as `_BeadDisplayCache`: `get()`, `should_resolve()`, `set()`,
  `clear()`, an `RLock`, LRU eviction bounded at 256 entries, a hit TTL (use 15 seconds — comfortably shorter than a
  human's read of the panel, far longer than the 1s header-summary TTL) and a longer miss TTL (use 60 seconds, since an
  id that is not in the store is usually a typo or a foreign-project bead and is not worth re-querying often).
- `cached_wait_bead_statuses(agent) -> tuple[tuple[str, str | None], ...] | None` — memory-only, safe to call from any
  thread, returns `None` when there is nothing to render.
- `resolve_wait_bead_statuses(agent) -> tuple[tuple[str, str | None], ...] | None` — the worker-thread entry point.
  Contract, documented in the docstring: **may touch bead stores; must only be called off the Textual event loop.**

`resolve_wait_bead_statuses()` logic:

1. `wait_agent = wait_display_agent(agent)` (from `sase.ace.tui.models.agent_time`) so synthetic family/clan/root rows
   resolve against the row that actually owns the wait, consistent with how `_append_wait_field` picks the ids it
   renders.
2. If `not wait_agent.waiting_for_beads`, return `None` immediately. **This gate is the main performance guarantee**:
   agents with no bead wait pay nothing beyond one attribute check, and that is nearly every agent.
3. Derive the project key from `wait_agent.project_file` via `Path(project_file).parent.name` (same derivation as
   `_agent_project_name()` in `agent_bead.py`), falling back to `agent.project_file`. If no key can be derived, return a
   tuple of `(bead_id, None)` pairs so the panel still renders the ids with unknown badges.
4. Partition the ids into cached-and-fresh vs. needing resolution using `should_resolve()`. If every id is fresh, return
   from cache with no I/O at all.
5. For the remaining ids, make a **single** `bead_statuses_for_project(project_key, stale_ids)` call and `set()` each
   result (found → status value, not found or store unavailable → `None`).
6. Return an ordered tuple aligned to `wait_agent.waiting_for_beads`.

Because step 5 is one store open per resolve regardless of how many bead ids are waited for, and step 4 collapses repeat
resolves inside the TTL window, a selected agent waiting on beads costs at most one store open per 15 seconds on a
worker thread.

### 3. Plumb through `DetailHeaderSummary`

- Add a field to `DetailHeaderSummary` in `src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py`:

  ```python
  wait_bead_statuses: tuple[tuple[str, str | None], ...] | None = None
  ```

  Defaulting to `None` keeps every existing `DetailHeaderSummary(...)` construction site valid.

- In `build_detail_header_summary()` (`_agent_display_header_summary.py`), populate it with
  `resolve_wait_bead_statuses(agent)` and pass it to the returned `DetailHeaderSummary`. This function is already
  documented and used as worker-thread-only, so no new threading is introduced.

### 4. Render the badges

In `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`:

- Add a badge table beside the existing `_WAIT_STATUS_BADGES`, reusing `AGENT_STATUS_BUCKET_GLYPHS` and the same colors
  so bead badges read identically to agent badges:

  | bead status                             | glyph | style                              | rationale                                                                        |
  | --------------------------------------- | ----- | ---------------------------------- | -------------------------------------------------------------------------------- |
  | `closed`                                | `✓`   | `bold #5FD75F`                     | same as the `Done` agent badge; the wait is satisfied                            |
  | `in_progress`                           | `▶`   | `bold #FFD700`                     | same as `Running`                                                                |
  | `claimed`                               | `◐`   | `bold #87D7FF`                     | same as `Starting`                                                               |
  | `open`                                  | `⏳`  | `bold #AF87FF`                     | same as `Waiting`; not started, still blocking                                   |
  | unknown / not found / store unavailable | `?`   | `_MISSING_WAIT_TARGET_GLYPH_STYLE` | matches the existing missing-wait-target convention already used for agent names |

- Add a `_append_wait_bead_status_badge(text, status)` helper next to `_append_wait_status_badge()`, with the same
  `text.append(" ")` + glyph shape.
- Change the `wait_agent.waiting_for_beads` branch of `_append_wait_field` from a single `", ".join(...)` into a loop
  that appends each id with `_WAITING_VALUE_STYLE`, a `", "` separator between entries, and a badge after each id.
- Give `_append_wait_field` a new `wait_bead_statuses: Sequence[tuple[str, str | None]] | None` parameter and thread it
  from `append_agent_metadata_fields`, which already receives `summary`:
  `summary.wait_bead_statuses if summary is not None else None`.
- Build a `dict(wait_bead_statuses)` lookup inside the branch rather than trusting positional alignment, so a stale
  summary (resolved for a slightly different bead list) degrades to `?` badges instead of mislabeling a bead.

**Graceful degradation is required**: when `summary` is `None` (cheap renders, first paint before enrichment lands) or
`wait_bead_statuses` is `None`, render exactly today's output — the plain comma-joined id list with no badges and no
trailing spaces. The panel re-renders once the worker posts `AgentDetailHeaderEnriched`, which is the same mechanism the
`Bead:` field already relies on.

### 5. Help modal

Per the ace subcommand guidelines (`src/sase/ace/CLAUDE.md`, "Help Popup Maintenance"), update the `Waiting Badges`
section in `src/sase/ace/tui/modals/help_modal/agents_bindings.py` so the new meaning is documented — e.g. add a
`("beads: id ✓", "Bead wait target status")` row, or extend the existing entry text to state that bead wait targets
carry the same badges. Keep descriptions within the 32-character limit noted in the ace guidelines.

## Explicitly out of scope

- **Agent-list row rendering.** `src/sase/ace/tui/widgets/_agent_list_render_agent.py` currently treats any bead wait as
  unsatisfied (`wait_deps_satisfied and not wait_agent.waiting_for_beads`), and `_agent_list_render_cache.py` keys rows
  on `waiting_for_beads`. Rows are rendered in bulk on the hot path, so putting bead status there needs its own
  warmup/patch design (like `warm_confirmed_bead_displays`). Leave both files untouched.
- **Changing wait semantics.** This is display-only; nothing about when an agent stops waiting changes.
  `wait_dependencies_satisfied()` in `src/sase/ace/tui/_agent_completion_wait.py` keeps returning `False` whenever
  `waiting_for_beads` is non-empty.
- **Clan-member bead aggregation** in the `(all clan members · …)` sub-list.

## Tests

Add `tests/ace/tui/widgets/test_agent_display_wait_bead_statuses.py`:

1. Rendering, per status — build an `Agent` with `waiting_for_beads=["sase-9r.2"]`, pass a
   `DetailHeaderSummary(wait_bead_statuses=(("sase-9r.2", "closed"),))` into `build_header_text()`, and assert the plain
   text contains `beads: sase-9r.2 ✓`. Parametrize over `closed`/`in_progress`/ `claimed`/`open` and their glyphs.
2. Unknown status — `(("sase-9r.2", None),)` renders `beads: sase-9r.2 ?`.
3. No summary — `build_header_text(agent)` with `summary=None` renders `beads: sase-9r.2` with no trailing badge,
   proving first paint is unchanged.
4. Multiple beads — mixed statuses render `beads: a ✓, b ⏳` with badges attached to the correct ids.
5. Stale/mismatched summary — `wait_bead_statuses` naming a bead not in `waiting_for_beads` yields `?` for the
   un-covered id and never crashes or mislabels.
6. Ids-only agents unaffected — an agent with `waiting_for` but no `waiting_for_beads` renders byte-identically to the
   current behavior.

Add `tests/ace/tui/models/test_agent_wait_beads.py`:

7. `resolve_wait_bead_statuses()` returns `None` and performs **zero** store calls for an agent with no
   `waiting_for_beads` — monkeypatch `bead_statuses_for_project` with a call counter and assert it is never invoked.
   This is the perf-guarantee regression test.
8. Two resolves inside the TTL window issue exactly **one** `bead_statuses_for_project` call.
9. A single resolve over three bead ids issues exactly **one** `bead_statuses_for_project` call (batching, not one open
   per id).
10. Store-unavailable (`bead_statuses_for_project` returns `None`) yields `(id, None)` pairs, and the negative result is
    cached so a second resolve inside the miss TTL does not re-query.
11. `wait_display_agent` indirection — an agent whose `wait_display_source` owns the bead wait resolves against the
    source row's beads and project.
12. Cache isolation — the module-level cache is cleared between tests via a fixture so ordering cannot leak state.

Add `tests/test_bead_statuses_for_project.py` (there is no existing `store_locator` test module):

13. `bead_statuses_for_project()` against a real temporary bead store returns correct status values, omits ids absent
    from the store, and returns `None` for an unknown project.

## Verification

- `just install` first (workspace virtualenvs can be stale), then `just check`.
- Manual check in `sase ace`: select an agent whose `Wait:` field shows `beads: <id>` and confirm a badge appears next
  to each id, and that a closed bead shows `✓`.
- Responsiveness spot-check while a bead-waiting agent is selected: with `SASE_TUI_PERF=1`, confirm the j/k key-to-paint
  p95 in `~/.sase/perf/tui_jk.jsonl` stays under the 16 ms target on the Agents tab, and that
  `~/.sase/logs/tui_stalls.jsonl` records no new stalls attributable to bead reads.
