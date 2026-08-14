---
tier: epic
title: SASE CONTEXT — stop re-parsing three whole stores per agent and stream the
  section lane by lane
goal: 'The SASE CONTEXT section in the Agents metadata panel shows commit context
  on the first paint and every remaining lane within a few tens of milliseconds instead
  of after one all-or-nothing enrichment pass, the per-agent cost stops scaling with
  the size of the artifact-file index and the memory/skill audit logs, and none of
  it moves work onto the Textual event loop.

  '
phases:
- id: trace
  title: Per-lane enrichment telemetry
  depends_on: []
  size: small
  description: 'trace: add a per-lane trace span and a repeatable measurement script
    for detail-header enrichment, record the baseline in a real terminal, and document
    the capture recipe so every later phase has a real before/after.'
- id: stores
  title: One parse per store change, not per agent
  depends_on:
  - trace
  size: medium
  description: 'stores: give the artifact-file index, the memory-read log, and the
    skill-use log process-wide revalidating snapshot caches so N agents share one
    parse instead of paying a full re-read each, and invalidate them from the write
    paths.'
- id: lanes
  title: Per-lane resolution, caching, and freshness
  depends_on:
  - trace
  size: medium
  description: 'lanes: split the monolithic detail-header summary into independently
    resolved and independently cached lanes with per-lane freshness policies, replacing
    the blanket 1 s whole-summary TTL, with no user-visible change yet.'
- id: stream
  title: Publish and render lanes as they resolve
  depends_on:
  - lanes
  size: medium
  description: 'stream: resolve lanes cheapest-first and merge/publish each one as
    it lands so the section renders partially, with stable lane order, a pending affordance,
    and coalesced repaints that do not disturb hint mode or scroll position.'
- id: immediate
  title: Zero-I/O context on the first paint
  depends_on:
  - stream
  size: small
  description: 'immediate: render the in-memory commit rows and any already-cached
    lanes on the cheap header path so SASE CONTEXT is present in the first paint after
    selection instead of after the debounce plus a worker round trip.'
- id: land
  title: Land the epic
  depends_on:
  - trace
  - stores
  - lanes
  - stream
  - immediate
  size: small
  description: 'land: re-measure the full budget against the trace phase baseline
    in a real terminal, file the named follow-ups with /sase_new_task, and close the
    epic with an honest reading of what each phase bought.'
proposed_by: bbugyi200.athena.zw
create_time: 2026-08-13 15:23:27
status: done
bead_id: sase-l6
---

- **PROMPT:** [prompts/202608/sase_context_incremental.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/sase_context_incremental.md)
- **BEAD:** [sase-l6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-l6/README.md)

# Plan: SASE CONTEXT — stop re-parsing three whole stores per agent and stream the section lane by lane

## Why this plan exists

Selecting an agent on the Agents tab shows the metadata header immediately, but the
`SASE CONTEXT` section — `PLAN`, `BEAD`, `ARTIFACTS` (commits, deltas, artifact files),
`MEMORY`, `SKILLS`, `WORKSPACES` — stays absent for a noticeable beat and then appears
all at once.

That is not incidental. `build_header_text` renders the section only when
`not cheap and summary is not None`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header.py:276`), and `summary` is
produced by one monolithic worker task, `build_detail_header_summary`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_summary.py:147`), which
resolves twelve separate things before returning anything. The section is gated on the
slowest of the twelve, and three of them re-parse an entire append-only store on every
call.

The work is already off the event loop — `run_worker(..., thread=True)` at
`_agent_display_async_agent.py:128` — so this is not a freeze. It is latency plus a
standing CPU tax that grows every day the machine is used. Both are fixable without
moving anything onto the loop.

## What was measured while writing this plan

All figures were taken on athena at `sase` master `b4542139a`, in a numbered workspace's
own `.venv` (Python 3.14), against live `~/.sase` state. Agents came from
`load_tiered_agents(full_history=False)` (518 rows loaded in 0.72 s); the first 20
non-clan rows were timed. Absolute numbers move with host load; the **ratios and the
warm/cold contrast are the portable result**.

### Per-resolver cost inside `build_detail_header_summary`

Milliseconds per agent, cold = first touch in a fresh process, warm = second touch after
every in-process cache is populated.

| Resolver                      | cold p50 | cold max | warm p50 | warm max |
| ----------------------------- | -------: | -------: | -------: | -------: |
| `load_skill_uses_*`           |    207.2 |    225.1 |      4.7 |      9.2 |
| `load_memory_reads_*`         |    145.1 |    166.1 |      4.9 |      7.5 |
| `artifact_file_paths`         | **86.6** |    386.3 | **88.9** |    349.6 |
| `resolve_agent_plan_*`        |      0.0 |    181.3 |      0.0 |      0.2 |
| `build_slow_tool_sources`     |      0.0 |    113.9 |      0.0 |     13.2 |
| `agent_delta_entries`         |      0.0 |     42.5 |      0.0 |     23.9 |
| `load_opened_workspaces_*`    |      0.3 |      4.5 |      0.2 |      5.3 |
| `load_xprompts_used`          |      0.1 |      2.1 |      0.1 |      2.1 |
| `agent_commit_groups`         |  **0.0** |  **0.0** |      0.0 |      0.0 |
| `agent_commit_linked_delta_*` |      0.0 |      0.0 |      0.0 |      0.0 |
| `resolve_wait_bead_statuses`  |      0.0 |      0.0 |      0.0 |      0.0 |
| `resolve_agent_page_url`      |      0.0 |      0.0 |      0.0 |      0.0 |

Two things stand out. First, `artifact_file_paths` **does not get cheaper when warm** —
it is the only resolver with no cache at all. Second, `agent_commit_groups` is free, and
it is the lane the user most often waits on.

### Where `artifact_file_paths` spends its time

| Component                                    | p50 ms | p90 ms | max ms |
| -------------------------------------------- | -----: | -----: | -----: |
| `synthesize_default_artifact_files`          |    0.9 |    7.2 |   11.2 |
| `list_indexed_artifact_files`                |   83.7 |   91.9 |   94.3 |
| `list_artifact_files` (total)                |   88.4 |   93.3 |   95.0 |
| `list_artifact_files` (immediately repeated) |   89.0 |   96.3 |  106.1 |

`list_indexed_artifact_files` (`src/sase/core/artifact_file_explicit.py:211`) calls
`read_artifact_file_index()` (`:195`), which takes a shared file lock and reads plus
JSON-parses the **entire** global index, then discards every row that does not match one
agent's association. There is no memoization anywhere on that path.

### The three stores, and why this gets worse over time

| Store                                                   |    Size |  Rows | Re-parsed per… |
| ------------------------------------------------------- | ------: | ----: | -------------- |
| `~/.sase/artifacts/index.jsonl`                         | 7.68 MB | 7,747 | every call     |
| `~/.sase/projects/gh_sase-org__sase/memory_reads.jsonl` | 2.73 MB | 4,164 | agent          |
| `~/.sase/projects/gh_sase-org__sase/skill_uses.jsonl`   | 3.09 MB | 6,222 | agent          |

All three are append-only and unpruned. Every hour of SASE usage makes the SASE CONTEXT
section slower for every agent.

## The four defects

**1 — Three whole-store parses, repeated per agent.** The memory-read and skill-use
loaders cache correctly against the log's mtime, but their cache key includes the agent
(`_cache_key(project, agent)`, `src/sase/ace/tui/memory_reads.py:68` and
`src/sase/ace/tui/skill_uses.py` alongside it), so a miss calls
`read_memory_read_events(project=...)` / `read_skill_use_events(project=...)`
(`src/sase/memory/read_log.py:391`, `src/sase/skills/use_log.py:148`) and re-reads the
whole log. Selecting N distinct agents costs N full parses of the same bytes. The
artifact index is worse: it has no cache at all, so even re-selecting the same agent
re-parses 7.7 MB.

**2 — All-or-nothing rendering.** `append_agent_context_section`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_context.py:40`) receives every lane's
data as fields on one `DetailHeaderSummary`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_state.py:71`). Until the whole
dataclass exists there is no section at all, so the free commit rows wait behind a 7.7
MB JSON parse.

**3 — A 1-second TTL on a ~90 ms recompute.** `should_refresh_detail_header_summary`
(`_agent_display_header_summary.py:102`) expires the summary after
`DIFF_CACHE_TTL_SECONDS`, which is **1.0**
(`src/sase/ace/tui/widgets/file_panel/_diff.py:53`), unless a hint session is active.
Every repaint of a stationary selection therefore re-runs all twelve resolvers. Repaints
arrive from the 10 s auto-refresh (`src/sase/ace/tui/app.py:234`), the 5 s slow-tool
render tick (`src/sase/ace/tui/widgets/prompt_panel/__init__.py:28`), and live-reply
updates. This is a standing background CPU cost per selected agent, and it contends for
the GIL with the UI thread. `sase/memory/tui_perf.md` rule 10 names this exact
anti-pattern: ticks revalidate, recomputes get a longer cadence.

**4 — Nothing survives the panel.** The summary cache is panel-local, memory-only, and
cleared wholesale by `show_empty()`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_display_render.py:494`).

## Reuse the clan pattern; do not invent a new one

The synthetic clan panel already solves the staged version of this problem, and the epic
should be read as extending that mechanism to ordinary agent rows rather than adding a
second one:

- `ClanDiskSection` names independently loadable sections
  (`src/sase/ace/tui/models/_agent_clan_sections.py:33`).
- `ClanSectionSnapshot` carries `disk`, `loading_sections`, and a `revision`, so a
  renderer can draw what has landed and mark what has not
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_clan_aggregation.py`).
- The worker requests only the sections the current fold state needs, merges partial
  results into the cached snapshot, and posts `ClanSectionSnapshotLoaded`
  (`_agent_display_async_groups.py:108`, `:317`).
- The app debounces the resulting repaint
  (`src/sase/ace/tui/actions/agents/_display_detail_render.py:296`).

Every one of those pieces has a per-agent analogue in this plan.

## Explicitly not the problem — do not spend effort here

- **Textual, widget mounting, and rendering.** The section is text appended to one
  `Text`; building it is microseconds once the data exists.
- **The 150 ms detail debounce** (`DetailPanelDebouncer`). It exists to protect j/k and
  must stay. Phase `immediate` gets context onto the _pre-debounce_ paint instead of
  shortening the debounce.
- **Moving enrichment onto the event loop, or awaiting it from a pump callback.** It is
  already a thread worker. `sase/memory/tui_perf.md` rules 1 and 2 apply to every new
  scheduling decision in this epic.
- **Speculative prefetch of neighbouring rows.** Rule 13 defers non-urgent work during
  navigation, and phase `stores` makes a second agent nearly free anyway, which is the
  better fix.
- **Pruning or rotating the three stores.** Worth doing (filed in `land`), but it treats
  the symptom; a store one tenth the size still gets re-parsed per agent today.
- **Bead lookups.** `resolve_agent_plan_enrichment` measures 0.0 ms at p50, with a 181
  ms cold tail already covered by `_PLAN_ASSOCIATION_CACHE`. It rides along with the
  per-lane work in `lanes`; it does not need its own attack.

---

## Phase `trace` — Per-lane enrichment telemetry

Everything below is measured from a script against library functions. None of it is
visible from a running TUI today, because `build_detail_header_summary` emits no trace
span and the panel records no timing. This phase runs first so `stores`, `lanes`,
`stream`, and `immediate` each have a real before/after rather than a component A/B.

### Deliverables

- **A `tui_trace` span per resolver** inside `build_detail_header_summary`, using the
  existing recorder (`src/sase/ace/tui/util/trace.py`, enabled by `SASE_TUI_TRACE=1`).
  Follow the shape already used at `_agent_display.py:54`. One parent span for the whole
  enrichment carrying the agent identity and a cold/warm marker, plus one child span per
  lane. Spans must be free when tracing is off — check the existing guard rather than
  assuming.
- **A committed measurement script** under `tests/perf/` following the conventions of
  the neighbouring `bench_*.py` files: load real agents through `load_tiered_agents`,
  time each resolver cold and warm over the first N non-clan rows, and print the two
  tables reproduced above. It must run read-only against live `~/.sase` state and
  default to a hermetic or opt-in mode for CI, the way `bench_agent_scan.py` gates its
  `home` workload behind `--include-home`.
- **A section in `docs/perf_runbook.md`** describing how to capture SASE CONTEXT
  latency: how to run the script, what the spans are called, and how to read
  `~/.sase/perf/tui_trace.jsonl` for the enrichment spans.
- **Record the baseline in the phase close note.** That is this phase's real output:
  per-lane cold and warm p50/max, the three store sizes, and the observed time from
  selection to a complete SASE CONTEXT section in a real terminal.

### Verification

`just check`, plus a real-terminal run with `SASE_TUI_TRACE=1` confirming the spans
appear with plausible durations and that `~/.sase/logs/tui_stalls.jsonl` stays quiet.
Confirm tracing-off adds no measurable cost.

---

## Phase `stores` — One parse per store change, not per agent

The largest measured win, and the only phase that needs no rendering change at all. It
is independent of `lanes` and can run in parallel with it.

### The artifact-file index

`read_artifact_file_index()` (`src/sase/core/artifact_file_explicit.py:195`) is the
whole cost of the ARTIFACTS lane: 7.68 MB and 7,747 rows re-read and re-parsed per call,
~84 ms each, with no cache warm or cold.

- **Add a process-wide revalidating snapshot cache in that module**, keyed on the
  resolved index path and `(st_mtime_ns, st_size)`, holding the parsed
  `list[ArtifactFile]`. `read_artifact_file_index` returns from it on a hit. Keep the
  shared-lock read on a miss.
- **Do not build this cache inside the TUI.** `sase/memory/*` records that shared
  backend behavior belongs in the core, not in a frontend adapter, and the read already
  lives in `sase/core/`. Caching it there means the CLI and any future frontend get the
  same benefit; reimplementing index parsing under `sase/ace/tui/` would be the boundary
  violation. Equally: **do not move the index into `sase-core` Rust in this phase** —
  that is a schema decision, filed as a follow-up in `land`.
- **Invalidate explicitly from the write paths in the same module**
  (`store_explicit_artifact_file`, `store_default_artifact_file`, and the index-writing
  helpers), which already hold the exclusive lock. `(mtime_ns, size)` revalidation is
  the cross-process backstop; explicit invalidation is what makes a same-process
  write-then-read correct regardless of filesystem timestamp granularity. Say that in
  the docstring — a reader must not "simplify" it away later.
- **Return defensively.** Callers must not be able to mutate the cached list and corrupt
  a later reader; return a copy of the list, and state in a comment that `ArtifactFile`
  rows are treated as immutable.
- **Consider grouping by association** while you are there.
  `list_indexed_artifact_files` filters the full row list by `matches_association` on
  every call; a snapshot-side index from association key to rows turns that from O(rows)
  into O(matching rows). Do it only if it stays simple — the parse, not the filter, is
  the measured cost.

### The memory-read and skill-use logs

Both loaders (`src/sase/ace/tui/memory_reads.py:114` and `:149`;
`src/sase/ace/tui/skill_uses.py` at the mirrored functions) already revalidate on log
mtime, but under a per-agent cache key, so every newly selected agent pays a full parse
of the same file: 145 ms and 207 ms at p50 here.

- **Add a shared per-project snapshot keyed on `(project, st_mtime_ns, st_size)`**
  holding the parsed event tuple, and have both the single-agent and the family-context
  paths filter that snapshot in memory. Keep the existing per-agent result caches on top
  — they are still worth having; they just must no longer be the only thing standing
  between a selection and a whole-file parse.
- **Keep the existing `_MIN_REREAD_INTERVAL_S` throttle** so a rapidly appended log does
  not cause a re-parse per repaint.
- **Bound the snapshot cache** (a small `OrderedDict` keyed by project is enough; there
  are few projects) and reuse the existing `_stat_mtime_ns` helpers rather than adding a
  third stat idiom.
- **Do the same audit for `opened_workspaces.py`.** It measures cheap today (0.3 ms p50)
  because it stats markers rather than parsing a log — confirm that, and if it is
  already correct, say so in the close note instead of changing it.

### Tests

- The load-bearing assertion in each case is a **call-count** test: monkeypatch the
  underlying reader (`read_artifact_file_index`'s file read, `read_memory_read_events`,
  `read_skill_use_events`), resolve the lane for M distinct agents, and assert exactly
  **one** underlying parse — then touch the file and assert exactly one more. Without
  these the caches silently regress.
- A same-process write-then-read test for the artifact index proving the explicit
  invalidation works even when mtime does not move.
- Existing tests that patch these readers may currently rely on being called per agent;
  port them rather than weakening them.

### Verification

`just check`, plus the phase `trace` script before and after. Expect warm
`artifact_file_paths` to drop from ~88 ms to near zero and the second and later agents'
`MEMORY`/`SKILLS` cold cost to drop from ~350 ms combined to near zero. Report the
first-agent cost separately — one parse per store per change is the floor this phase
targets, not zero.

---

## Phase `lanes` — Per-lane resolution, caching, and freshness

Structural phase. No user-visible change; it makes `stream` and `immediate` possible and
kills defect 3 on its own.

### Deliverables

- **Name the lanes.** Add a `DetailContextLane` literal set modelled on
  `ClanDiskSection` (`src/sase/ace/tui/models/_agent_clan_sections.py:33`), covering the
  six SASE CONTEXT lanes (`plan-bead`, `artifacts`, `memory`, `skills`, `workspaces` —
  `PLAN` and `BEAD` share one resolver, `resolve_agent_plan_enrichment`, and must stay
  one lane) plus the non-context summary fields the same worker produces (`slow-tools`,
  `xprompts`, `page-url`, `wait-beads`). Reuse `CONTEXT_LANE_ORDER`
  (`_agent_context.py:30`) as the render order; the lane set is a resolution concern,
  the order is a presentation concern, and they must not be conflated.
- **Make `build_detail_header_summary` lane-selective.** It takes a requested lane set
  and returns a partial summary plus the set of lanes it actually resolved. Preserve the
  two existing `include_*` flags' behavior for the clan aggregation caller
  (`_agent_display_header_summary.py:150`) by expressing them as lane selection.
- **Carry readiness on the summary.** `DetailHeaderSummary`
  (`_agent_display_state.py:71`) gains `ready_lanes: frozenset[DetailContextLane]` (and
  the loading counterpart if `stream` needs it). A lane that resolved to "nothing" must
  be distinguishable from a lane that has not resolved — that distinction is the whole
  point, and getting it wrong shows an empty section that should say "loading".
- **Merge instead of replace.** `cache_detail_header_summary`
  (`_agent_display_header_summary.py:123`) merges newly resolved lanes into the cached
  summary, in the shape of `cache_clan_disk_snapshot`. Keep the existing
  `associated_plan_cache_key` invalidation
  (`src/sase/ace/tui/models/agent_associated_plan.py:65`) — it is a memory-only key and
  stays correct.
- **Per-lane freshness replaces the blanket TTL.**
  `should_refresh_detail_header_summary` becomes a per-lane decision returning the set
  of stale lanes. Constraints rather than a prescription: no lane may keep a 1 s
  recompute cadence; lanes whose inputs are now snapshot-cached (`artifacts`, `memory`,
  `skills`) should revalidate cheaply and often but recompute only when their store
  actually changed; lanes over in-memory data (`wait-beads`, the commit part of
  `artifacts`) need no TTL at all; `slow-tools` keeps a cadence matched to its existing
  5 s render tick, not a 1 s one. `sase/memory/tui_perf.md` rule 10 is the governing
  rule — read it before choosing the numbers, and write the reasoning for each lane's
  policy into the module docstring.
- **Keep `DIFF_CACHE_TTL_SECONDS` alone.** It belongs to the diff cache; the detail
  header borrowing it is the bug. Do not change its value.

### Tests

Per-lane resolution with a partial request set; merge semantics (a later partial result
must not blank an already-resolved lane); resolved-empty versus unresolved; per-lane
staleness including the assertion that a stationary selection does **not** re-resolve
every lane once per second. The existing async header tests
(`tests/ace/tui/widgets/test_agent_display_header_enrichment_async.py`) and the lane
render tests (`test_agent_context.py`, `test_agent_display_bead_section.py`,
`test_agent_display_plan_section.py`, `test_agent_display_commit_metadata.py`) are the
regression surface — port them, do not relax them.

### Verification

`just check`, plus phase `trace` spans confirming that a stationary selection settles
into revalidation rather than repeated full recomputes. Rendered output must be
byte-identical to today once every lane is resolved; a visual snapshot run
(`just test-visual`) is the cheap way to prove that.

---

## Phase `stream` — Publish and render lanes as they resolve

This is the phase the user actually asked for.

### Deliverables

- **Resolve cheapest-first.** Order the worker's lane resolution by measured cost:
  in-memory lanes first (`wait-beads`, commits), then `plan-bead`, then `workspaces`,
  then the store-backed lanes (`artifacts`, `memory`, `skills`), then `slow-tools`. Put
  the ordering in one named constant with the measured costs in a comment so a later
  reader can re-derive it.
- **Merge and publish per lane.** Instead of one `AgentDetailHeaderEnriched` at the end
  (`_agent_display_async_agent.py:303`), merge each lane into the cached summary as it
  lands and post a message then. Do this **from the worker thread through the existing
  message path**, not by mutating widget state directly, and keep the
  `request.is_current(...)` generation/identity guard on every publish — a stale lane
  from a previous selection must never land on the current one.
- **Coalesce the repaints.** Six publishes must not mean six full document rebuilds
  during a j/k burst. Route the repaint through the existing debouncer the way
  `on_clan_section_snapshot_loaded` does
  (`src/sase/ace/tui/actions/agents/_display_detail_render.py:296`), and/or batch lanes
  that complete within the same short window inside the worker. Rule 2 applies: the
  message handler stays thin.
- **Render ready lanes and mark the rest.** `append_agent_context_section` renders every
  ready lane in `CONTEXT_LANE_ORDER` and shows a dim, non-interactive pending affordance
  for lanes still resolving. The section heading's count (`_agent_context.py:129`) must
  reflect what is shown. Do not reorder lanes by completion time — stable order is what
  stops the section from visibly reshuffling.
- **Hint mode must not renumber under the user.** File hints are numbered in render
  order (`_agent_context.py:59-82`, `_artifact_files.py:61`), so a lane appearing later
  shifts every subsequent number. Today `on_agent_detail_header_enriched` already
  declines to re-render when the hint input has a value
  (`_display_detail_render.py:285`); with six publishes that guard must additionally
  suppress mid-session renumbering — the safe rule is that hint-mode documents only
  rebuild when the lane set is complete or the hint input is empty. Also update
  `header_enrichment_pending` (`_agent_display_hint_render.py:226`) to mean "some lane
  is still pending" rather than "no summary yet", since it drives re-triggering
  enrichment from the hint cache (`_agent_display_hints.py:115`).
- **Scroll and section navigation must stay stable.** The panel reconciles a section
  cursor across updates (`prepare_section_document_for_agent` and
  `preserve_missing_section_on_next_update`,
  `src/sase/ace/tui/widgets/prompt_panel/__init__.py:60`, `:70`). Growing the document
  under the user is exactly the case those exist for; verify the cursor and scroll
  position survive a lane landing while the user is reading, and use
  `preserve_missing_section_on_next_update` where a partial paint is deliberate.

### Tests

A partially-resolved summary renders the ready lanes and the pending affordance; lane
order is independent of completion order; a lane from a superseded selection is dropped;
hint numbering does not change while a hint session has typed input; repaints are
coalesced under a burst. Add a PNG snapshot for the partial state next to the existing
`tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py`, and refresh the
complete-state goldens only if the complete rendering genuinely changed.

### Verification

`just check`, plus a real-terminal run: select a cold agent and confirm lanes appear
progressively rather than at once, with no visible reshuffle, no scroll jump, and the
stall watchdog quiet. Report time-to-first-lane and time-to-complete-section separately
against the `trace` baseline.

---

## Phase `immediate` — Zero-I/O context on the first paint

After `stream`, the first SASE CONTEXT paint still waits for the 150 ms debounce plus a
worker round trip. Commit rows do not need either.

`agent_commit_groups` / `agent_commit_diffs`
(`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py:498`, `:520`) parse only
`step_output` metadata; the docstring states outright that they avoid file-existence
checks "so render paths can call it without disk I/O", and they measure 0.0 ms. That
makes them legal on the cheap header path under `sase/memory/tui_perf.md` rule 11.

### Deliverables

- **Render the in-memory part of SASE CONTEXT in `update_header_only`**
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display.py:181`), which currently
  passes `cheap=True` and therefore skips the section entirely
  (`_agent_display_header.py:276`). The cheap path renders: the commit rows of the
  `ARTIFACTS` lane, plus **any lane already present in the panel's cached summary**, and
  nothing that would touch disk.
- **Replace the `cheap` boolean gate with the lane-readiness check** from `lanes`, so
  the cheap path is "the lanes that are free right now" rather than a separate code path
  with its own rules. One renderer, two readiness inputs.
- **Do not start workers from the cheap path.** `update_header_only` deliberately
  cancels rather than starts (`_agent_display.py:207-209`); keep that. Enrichment still
  starts from the debounced `update_display` (`:74-76`).
- **Verify against the navigation gate.** Adding rows to the immediate paint must not
  regress j/k. `SASE_TUI_PERF=1` targets p95 < 16 ms key-to-paint on every tab; measure
  it, do not assume it.

### Tests

The cheap header contains the commit rows when `meta_commits` is present and no summary
is cached; the cheap header does no disk I/O (assert against patched loaders); a cached
lane from a previous selection of the same agent renders immediately; the cheap path
starts no worker.

### Verification

`just check`, plus `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` and a
`SASE_TUI_PERF=1` capture on the Agents tab before and after. Report the p95 both ways —
if the immediate paint costs measurable key-to-paint latency, say so and reconsider the
scope rather than shipping it quietly.

---

## Phase `land` — Land the epic

Runs after all five implementation phases are committed and `just check-full` is green
on the combined tree.

### Re-measure, then report honestly

Using the phase `trace` recipe, capture the per-lane table cold and warm, plus these
three end-to-end numbers in a real terminal, before and after:

1. selection to first SASE CONTEXT content,
2. selection to complete SASE CONTEXT section,
3. steady-state enrichment CPU for one stationary selection over 60 s.

The target is context content on the first paint, a complete section within roughly 100
ms warm, and steady-state enrichment work that is revalidation rather than
recomputation. If a number misses, state it and by how much.

### Follow-ups to file

Use `/sase_new_task` for each, one at a time, naming the proposing bead, and record
every outcome — including duplicates the skill corroborates and any declined — in the
close note.

1. **Prune or rotate the three append-only stores.** 7.7 MB / 2.7 MB / 3.1 MB today and
   growing; this epic makes them cheap to read but not bounded.
2. **Move the artifact-file index into the Rust core's indexed storage.** A JSONL file
   re-parsed in every process is a schema decision that outlived its scale. Take this
   epic's measurements as the input.
3. **`show_empty()` clears the whole detail-header summary cache**
   (`_agent_display_render.py:494`), so deselecting throws away every lane. Evaluate
   whether it can be scoped to the current agent.
4. **Cold-cache behavior is unmeasured.** Every number here is warm-page-cache; dropping
   caches needs root. The store-backed lanes are the cold-sensitive ones.
5. **Audit the remaining callers of `read_artifact_file_index`** outside the TUI for the
   same per-call re-parse pattern, now that a cache exists.

Close the epic with a note covering the re-measured budget, which of the four defects
each phase actually removed, and every follow-up outcome. Run `just symvision` after the
close.
