---
tier: epic
title: Beads sub-section in the agent metadata panel's ARTIFACTS lane
goal: Selecting a sase agent in the Agents tab shows a Beads sub-section inside SASE
  CONTEXT's ARTIFACTS lane that lists every bead that agent read, created, or changed,
  with one row per bead, the verbs it performed, and the bead's title; the underlying
  agent-to-bead touch facts are reduced once in sase-core, queryable from the CLI,
  and cheap enough for the panel's hot path.
phases:
- id: core-index
  title: Reduce bead event streams into an actor-keyed touch index in sase-core
  depends_on: []
  size: medium
  description: 'core-index: add the actor/bead touch reduction over beads/events/streams,
    its versioned index wire, incremental signature-cached refresh, and the read-only
    touch query, exposed through PyO3 bindings with fixture and parity tests.'
- id: host-refresh
  title: Adopt the touch index in Python and keep it fresh off the hot path
  depends_on:
  - core-index
  size: medium
  description: 'host-refresh: add the Python facade over the new bindings, normalize
    actors onto agent identities, and refresh the index incrementally from bead mutation
    commits, post-sync refresh, and a bounded lumberjack routine, with a doctor check.'
- id: bead-cli
  title: sase bead touched
  depends_on:
  - host-refresh
  size: small
  description: 'bead-cli: add the agent-scoped touch listing subcommand with colored
    and JSON output, and wire its help, ordering, and option contract to the CLI rules.'
- id: panel-data
  title: Resolve per-agent bead touches for the metadata panel
  depends_on:
  - host-refresh
  size: medium
  description: 'panel-data: add the mtime-cached per-agent touch loader, the summary
    field and artifacts-lane resolution that carries it, and the merge that folds
    audited bead reads and the agent''s own assigned beads into one ranked per-bead
    view.'
- id: panel-render
  title: Render the Beads sub-section
  depends_on:
  - panel-data
  size: medium
  description: 'panel-render: paint the per-bead rows, verb chips, glyph and palette,
    lane counts, hints, and clan aggregation, stop double-listing bead refs under
    Reads, and cover the result with header and visual tests.'
- id: bead-views
  title: Record agent bead views so unaudited reads are not silently missing
  depends_on:
  - panel-render
  size: small
  description: 'bead-views: record agent-attributed sase bead show invocations as
    a local viewed touch, merge them into the query behind the durable mutation and
    audited-read facts, and render them as a visibly weaker signal.'
proposed_by: bbugyi200.athena.0oa
create_time: 2026-09-20 16:31:02
status: wip
bead_id: sase-14j
---

- **PROMPT:** [prompts/202609/agent_bead_touches.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_bead_touches.md)
- **BEAD:** [sase-14j](https://github.com/sase-org/sase--beads/blob/main/pages/sase-14j/README.md)

# Beads sub-section in the agent metadata panel's ARTIFACTS lane

## Outcome and scope

Selecting an agent in the Agents tab renders `SASE CONTEXT` → `ARTIFACTS`. That lane
today has four sub-sections: `Reads:`, `Commits:`, `Deltas:`, and `Files:`. This epic
adds a fifth, `Beads:`, which answers one question completely: **which beads did this
agent touch, and what did it do to each of them?**

One row per bead, newest touch first, carrying the bead id, the set of verbs the agent
performed on it with repeat counts, and the bead's title. A bead the agent was launched
to work on is marked as its own. The row's hint opens that bead's page.

In scope: the durable reduction of agent-to-bead facts, its refresh, a CLI surface over
it, the panel data path, and the rendering. Out of scope: any change to bead semantics,
the bead store's on-disk format, the existing `BEAD` lane above `ARTIFACTS`, and the
Artifacts tab's own beads/agents panes.

This is an epic because the work crosses a Rust core contract, host refresh plumbing on
a store-mutation path, a new CLI subcommand, and a performance-sensitive TUI lane, and
because the substrate must exist and be trustworthy before any of its three consumers
can be written. Plan approval authorizes implementation; this authoring turn changes
only this scratch plan.

Repositories are the current SASE checkout and the linked `sase-core`. A worker touching
`sase-core` must use `/sase_repo` with `sase repo open sase-core -r '<specific reason>'`
first, read that checkout's `AGENTS.md`, use only the returned path, and point the
existing `SASE_CORE_DIR`/Justfile override at it for local builds. No memory-file edits
are part of this plan; see "Memory and follow-ups".

## What is already tracked, and what is not

This was verified against the live `sase` bead store (1,586 streams, 35,601 events) and
the live artifact-link aggregate (19,662 rows).

- **Every bead mutation is already attributed.** `beads/events/streams/<bead-id>.jsonl`
  is the canonical per-bead event log, and every event carries `actor`, `operation`,
  `issue_id`, and `timestamp`. `src/sase/bead/attribution.py` already resolves the
  acting agent's globalized name for new beads, and the runner writes the same identity
  for mutations. There are 13,069 distinct `(actor, bead)` pairs today. **No new
  mutation-side tracking is needed.**
- **Bead titles are already available from the same streams.** `issue_created` carries
  the full issue payload and `issue_updated` carries field changes, so a reduction over
  the streams can carry each bead's title, type, and last-known status without ever
  opening `issues.jsonl` or querying the bead store.
- **Audited bead reads are already tracked twice.** `sase artifact read bead:<id>`
  writes `~/.sase/projects/<key>/artifact_reads.jsonl` and an
  `agent:<name> read bead:<id>` artifact-link row. The metadata panel **already loads
  that log for this exact lane** (`load_artifact_reads_for_agent_context`), so bead
  reads cost no new I/O.
- **Assignment is already tracked.** `agent:<name> implements bead:<id>` rows are
  projected from published agent metadata by
  `src/sase/artifact_links/projection/_agent_bead.py` (7,256 rows), and the same fields
  (`bead_id`, `epic_bead_id`, `phase_bead_id`) are already on the in-memory `Agent` row.
- **What is missing is only the reverse index.** Nothing can cheaply answer "which beads
  did actor X touch". Answering it today means replaying 1,586 streams (29 MB), which is
  legal for a background job and illegal on the panel's path.
- **`sase bead show` is not recorded at all.** It is the dominant way agents look at a
  bead, and it leaves no trace. The `bead-views` phase addresses this deliberately and
  separately, because a convenience peek is weaker evidence than an audited read and
  must not be conflated with one.
- **Two attribution defects are load-bearing for this design.** First, actors are not
  uniformly globalized: `note_appended` alone has 2,904 distinct actors mixing
  globalized names (`bbugyi200.athena.0gk`) with bare local names (`013`, `uc`,
  `sase-zt.6.5.land`). Second, 46% of all events record the store owner
  (`bryanbugyi34@gmail.com`) rather than an agent, and `link_added` records the owner
  100% of the time. Matching must handle the first; the second means some verbs are
  simply absent from the corpus today and the UI must not imply otherwise.

## Why the reduction belongs in sase-core

`sase/memory/` core memory states the boundary test: if another frontend would need the
behavior to match the TUI, it is core backend logic. "Which beads has this actor
touched, and how" is exactly that — a CLI (`sase bead touched`), the TUI lane, and any
later web surface must return the same answer from the same reduction. sase-core already
owns the event streams (`bead/events.rs`, `reduce_event_streams`) and their per-bead
history projection (`bead/history.rs`). A second Python reducer over the same files
would be a duplicate domain implementation with its own drift.

Python keeps what it already owns: agent identity resolution, project paths and locks,
CLI presentation, and TUI caching.

## Public contract

### The touch record

A **touch** is one `(actor, bead)` pair with aggregated verbs. Its wire shape:

```text
actor        globalized or raw actor string exactly as the stream recorded it
bead_id      full bead id
title        bead title at the newest event that carried one, else ""
issue_type   plan | phase | task, else ""
status       last-known status from the reduced events, else ""
verbs        map of verb -> positive count
first_at     earliest contributing event timestamp (RFC 3339 Z)
last_at      newest contributing event timestamp (RFC 3339 Z)
```

Verb mapping from `operation`, exhaustive over the operations present in the live store:

| verb       | operations                                                  |
| ---------- | ----------------------------------------------------------- |
| `created`  | `issue_created`                                             |
| `updated`  | `issue_updated`                                             |
| `noted`    | `note_appended`, `note_edited`, `note_removed`              |
| `closed`   | `issue_closed`                                              |
| `reopened` | `issue_opened`                                              |
| `+1`       | `task_plus_one_recorded`                                    |
| `ready`    | `ready_marked`, `ready_unmarked`                            |
| `snoozed`  | `task_snoozed`, `task_snooze_woken`, `task_snooze_canceled` |
| `dep`      | `dependency_added`, `dependency_removed`                    |
| `linked`   | `link_added`, `link_removed`                                |
| `ref`      | `reference_added`, `reference_removed`                      |
| `removed`  | `issue_removed`                                             |

`epic_work_preclaimed` is **excluded**. It is the runner reserving a phase bead at epic
launch, not an agent editing a bead, and including it would attach 4,184 meaningless
touches to launching agents. An operation this table does not name contributes no verb
and no touch; adding a new bead operation must not make the index reject a stream.

### Identity matching

An agent matches a touch when the touch's `actor`, after trimming, equals any of:

1. the agent's globalized name,
2. the agent's local `agent_name`,
3. `globalize_owned_agent_name(actor)` compared against the agent's globalized name,
   which recovers the legacy bare-local-name rows.

An actor that is an email address, or that `validate_new_agent_name` rejects, is not an
agent: it is dropped from the index entirely rather than shown as a nameless toucher.
Matching is exact after those normalizations; no prefix or suffix matching, because bead
ids and agent names both use dotted suffixes and a loose match would cross-attribute.

### Index file and freshness

The index is a derived cache, never a source of truth, and is always safely rebuildable
from the streams:

```text
~/.sase/projects/<key>/agent_bead_touches.json
{"schema_version": 1, "generation": "<RFC 3339 Z>",
 "streams": {"<bead-id>": [mtime_ns, size], ...},
 "touches": [ <touch record>, ... ]}
```

Refresh is incremental: a stream whose `(mtime_ns, size)` is unchanged contributes its
cached touch rows verbatim; only changed, new, and vanished streams are re-reduced.
Refresh is write-locked with the existing project file lock, is idempotent, and is
recomputed from scratch when `schema_version` does not match. A missing, truncated, or
unparseable index is a cache miss, never an error: the query returns nothing and the
next refresh rebuilds it.

Refresh runs only where it cannot be felt: after a bead-store mutation commits, after a
bead-store sync pulls other machines' streams, and on a bounded lumberjack tick. Nothing
on a keystroke, render, or completion path may trigger a refresh —
`sase/memory/tui_perf.md` rules 1, 8, and 11.

### Honest counts

The lane reports what the corpus contains, never what it wishes it contained. Because
`link_added` is not agent-attributed today, `linked` will not appear; because
`sase bead show` is unrecorded until the last phase, a bead an agent only peeked at will
not appear. The lane must therefore never render a total that implies completeness (no
"all beads", no "N/N"), and the `bead-views` phase must render `viewed` as a visibly
weaker signal than `read`.

## Presentation design

### Placement and shape

`Beads:` is the **first** sub-section inside `ARTIFACTS`, above `Reads:`. The lane
already reads consult-then-produce; beads are the unit of work the agent was doing, so
they belong at the top, and putting them first lets `Reads:` become purely non-bead
consultation.

Rows reuse the lane row vocabulary in `_agent_context_common.py` unchanged
(`append_lane_row` for `HH:MM:SS`, glyph, and primary; `append_context_reason` for the
wrapped `↳` line), so the new sub-section aligns column-for-column with `Reads:`,
`MEMORY`, `SKILLS`, and `WORKSPACES`:

```text
▸ ARTIFACTS · 4 beads · 2 reads · 1 commit · 5 files
  Beads:
    16:41:02  ✚ sase-14g · created · noted ×2
              ↳ Custom gate with a failed command becomes unreachable
    16:12:55  ✓ sase-l6.4 · own · closed · noted ×3
              ↳ Stream SASE CONTEXT lanes progressively
    15:58:10  ✎ sase-zl.3 · updated · dep
              ↳ Prevent workspace startup failures after a pull
    15:31:44  ← sase-su · read ×2
              ↳ Drain the provider outbox before finalization
    + 3 more · 14:02 earliest
```

The timestamp is the bead's newest touch. The primary is the bead id. Verb chips follow
in a fixed order (`own`, then durable verbs newest-first by their own last touch, then
`read`, then `viewed`), joined by `·`, with `×N` only when N > 1. The bead's title is
the `↳` line, which is what makes the list legible at a glance — a list of bare ids is
not.

### Palette and glyphs

The sub-section borrows the `BEAD` lane's amber so the eye connects it to the bead
context above, while living inside the blue `ARTIFACTS` lane: glyph in
`COLOR_BEAD_SUBHEADER`, bead id in `COLOR_BEAD_PRIMARY`, verb chips in `COLOR_SUMMARY`,
title in `COLOR_REASON`, `own` chip in `COLOR_ROLE`. `Beads:` itself is `COLOR_SUMMARY`,
matching the four existing sub-section headers exactly.

The glyph is the row's single strongest verb, chosen by that precedence:

```text
✚ created   ✓ closed   ↻ reopened   ✎ updated/noted/ready/snoozed/dep/linked/ref/+1
← read (audited only)   ◇ viewed only   ⌫ removed
```

New glyphs must be single-cell; add them to `_agent_context_common.py` next to the
existing lane glyphs rather than inline, and assert their cell width so the shared
column contract cannot drift.

### Ranking, caps, hints, folding

Newest-touch-first. `MAX_VISIBLE_BEADS = 5`, matching `MAX_VISIBLE_READS`, with the same
`+ N more · HH:MM earliest` footer. An agent's own assigned beads sort to the top of
their timestamp position only by virtue of being touched; no special ordering, because a
stale own-bead should not outrank live work. A bead the agent was assigned but never
touched still appears, with the `own` chip and no verbs, because its absence would make
the list look wrong next to the `BEAD` lane.

Each row takes a hint number whose target is `bead_page_path(bead_id)` when that page
exists, so `Beads:` hints behave exactly like `Reads:` and `Files:` hints. Rows whose
page is missing take no hint rather than a dead one.

`Beads:` is part of the existing `ARTIFACTS` fold section and digest; it is not
separately foldable. Its count joins the lane header details as the first entry.

### No double listing

`Reads:` stops rendering `bead:` refs, because `Beads:` now owns every bead interaction
including audited reads. The lane header counts each bead read once, under `beads`. A
regression test must pin that a bead read appears in exactly one sub-section.

## Implementation phases

### core-index

Add the reduction, index, and query to `sase-core`, alongside the existing bead event
and history modules rather than inside them. Provide: a pure function from a stream's
lines to that stream's touch rows; an incremental refresh that stats each
`events/streams/*.jsonl`, reuses cached rows for unchanged streams, and writes the index
atomically under a lock; and a read-only query that loads the index and returns the
touches for a requested set of actors. The query performs no directory scan and no
stream parse — a stale index returns stale answers rather than paying for freshness on
the caller's clock.

Follow the crate's established wire conventions: a versioned wire struct, a
`*_wire_schema_version()` accessor, serde types mirrored by the Python side, and
`BeadError` for genuine failures only. Malformed lines, unknown operations, unknown
payload shapes, and a stream whose filename does not match its events are skipped with
the event's own resilience posture, never fatal. Expose refresh and query through PyO3
in `crates/sase_core_py`, documented in that crate's binding list, and add parity tests
for the complete returned snapshot. Cover: empty store, a stream with only excluded
operations, mixed globalized and bare-local actors, owner-actor dropping, title carried
from `issue_created` and then overwritten by `issue_updated`, verb counting across
repeats, `first_at`/`last_at` boundaries, an unchanged-stream cache hit, a changed
stream re-reduction, a vanished stream's rows disappearing, and a `schema_version` bump
forcing a full rebuild. Run the core repository's `just check`, including PyO3 tests;
Cargo-only tests are insufficient. Do not hand-edit release-plz-owned crate versions.

### host-refresh

Add a thin Python facade beside the other bead facades in `src/sase/core/` that wraps
the two new bindings, resolves the index path from `sase_projects_dir()`, and owns
nothing the Rust side already owns. Add the identity normalization from "Identity
matching" in one place, with the agent-name helpers it already has in
`sase.core.agent_identity_facade`, so the CLI, the panel, and any later caller share one
matcher.

Refresh at three points, each incremental and each already off every interactive path:
`bead_store_mutation` in `src/sase/bead/cli_common.py` after a mutation commits, so an
agent's own edit is visible in the panel within one poll; `refresh_bead_store` in
`src/sase/bead/_sync_refresh.py` after a pull, so other machines' streams are picked up;
and a bounded step on the existing `artifact_link_backfill` lumberjack routine in
`src/sase/default_config.yml` (or a sibling routine if that job's budget is already
spent), so a store that has drifted for any other reason converges. Every refresh is
best-effort: a failure logs and is swallowed, never breaks a bead mutation, a sync, or a
job tick. Add a `sase doctor` check that reports index staleness and schema mismatch,
and tests that a mutation refreshes exactly the streams it wrote.

### bead-cli

Add `sase bead touched` between `task-type` and `update` in the subcommand list and in
the parser's alphabetical ordering. It takes the agent name as a positional argument,
because the command cannot run without one, and optional modifiers only — `-j/--json`,
`-l/--limit`, `-v/--verb` (repeatable), and `-a/--all` to include the store-owner actor.
Every long option gets a short alias. Output is colored and scannable: bead id, verb
chips, title, and relative last-touch time, using the same glyph vocabulary the panel
uses so the two surfaces read as one feature. `-h` output must be complete enough that
`sase_beads.md` needs no new prose. Follow `sase/memory/cli_rules.md`; read it with
`/sase_memory_read` before writing the parser.

### panel-data

Add `src/sase/ace/tui/bead_touches.py` modeled directly on
`src/sase/ace/tui/artifact_reads.py`: a frozen display-event dataclass, an
mtime-and-size keyed cache with the same `_MIN_REREAD_INTERVAL_S` throttle, a bounded
snapshot cache across projects, and a per-agent loader plus an agent-family context
loader that carries the member role label. The loader reads the index only; it never
refreshes it.

Carry the result as a new `DetailHeaderSummary` field and add that field to
`_LANE_FIELDS["artifacts"]`. Do **not** add a new `DetailContextLane`: the sub-section
must land in the same frame as the rest of `ARTIFACTS`, the lane-batch tests require
every lane to sit in exactly one resolution batch, and the index read is a small
mtime-cached JSON load that belongs in the same batch-2 budget as the artifact-read log
it sits next to. Resolve it inside the existing `"artifacts" in lanes` block, under its
own `tui_trace` span.

Add the merge that produces the ranked per-bead view the renderer consumes, as a pure
function over three inputs so it is testable without a store: index touches, the
already-loaded audited artifact reads filtered to `bead:` refs, and the agent's own
`bead_id`/`epic_bead_id`/`phase_bead_id`. It emits one entry per bead with merged verb
counts, the newest timestamp across all sources, the title from whichever source has
one, and the `own` mark. Bead ids are compared after canonicalization so a shorthand ref
and a full id never split into two rows.

### panel-render

Add `src/sase/ace/tui/widgets/prompt_panel/_agent_bead_touches.py` holding only the row
painting, mirroring `_agent_artifact_reads.py`'s division of labor: the `ARTIFACTS` lane
keeps owning sub-section order and the header counts. Add the new glyphs and reuse the
existing colors from `_agent_context_common.py`. Wire the sub-section into
`_agent_artifacts_lane.py` first in order, add `beads` first in the header details, and
thread the new summary field through `_agent_context.py`, the header builders, and the
family/clan paths, including a `BeadTouch` branch in `_agent_display_clan_context.py`'s
hint-target resolution that returns the bead page path.

Filter `bead:` refs out of the `Reads:` rows and out of the `reads` count in the same
change, so the two sub-sections never overlap at any point in history. Cover with
header-text tests in `tests/ace/tui/widgets/` following the existing
`test_agent_artifact_reads.py` and `test_agent_display_bead_section.py` shape: exact row
and chip order, palette spans via `assert_span_covers`, the `own` chip, single-cell
glyph widths, the overflow footer, the cap, the migration of a bead read out of
`Reads:`, alignment at 120 and 28 columns with no truncation of the bead id, an empty
index rendering no sub-section and no header count, and a cheap header doing no index
read at all. Read `sase/memory/tui.md` and `tui_perf.md` with `/sase_memory_read` first.
Rendered output changes, so run `just fix-tui-screenshots` for the affected scenes and
inspect every golden diff — generation is not approval.

### bead-views

Close the `sase bead show` gap. A view is not a bead event, so it cannot come from the
stream reduction and must not be written into the audited artifact-read log, whose rows
require an authored reason and carry link-recording weight. Record it instead as its own
small local log, `~/.sase/projects/<key>/bead_views.jsonl`, written by `sase bead show`
only when an agent identity is present, so interactive use pays nothing and the audited
corpus stays clean. Merge it in the Python facade at query time, behind the durable
mutation facts, and render it as `viewed` with the weakest glyph and no promotion to
`read`.

State the limitation in the module docstring and in `sase bead touched -h`: the view log
is machine-local, so a remote agent's views are not visible while its mutations are.
This phase is deliberately last and deliberately separable — if review prefers the
existing doctrine that an audited `sase artifact read bead:<id>` is the sanctioned way
to read a bead as context, dropping this phase leaves the feature coherent.

## Integrated acceptance and landing

Acceptance is behavioral, not structural. On a real store: launch or pick an agent that
created a task bead, noted its own phase bead, closed it, and audited-read another bead;
confirm `Beads:` lists exactly those four beads, with the right verbs, counts, glyphs,
titles, `own` mark, and ordering; confirm the bead it audited-read no longer appears
under `Reads:`; confirm each row's hint opens that bead's page; confirm
`sase bead touched <agent>` agrees with the panel row for row. Confirm an agent that
touched no beads renders no `Beads:` sub-section and no `beads` count. Confirm a bead
mutation is reflected after one poll without any TUI restart, and that j/k navigation
across agents shows no regression in the `SASE_TUI_PERF=1` p95 for the Agents tab.

No feature flag is required. `sase/memory/sase_flags.md` scopes a beta flag to a landed
phase that would expose part of an unfinished feature; here `core-index`,
`host-refresh`, and `panel-data` expose nothing, `bead-cli` lands a complete CLI surface
over a complete reduction of existing history, and `panel-render` lands the UI whole. No
user-reaching behavior is deprecated, so no sunset flag applies either. A worker that
nonetheless finds itself about to land a half-state must read `sase_flags.md` through
`/sase_memory_read` and use `sase flag new` rather than landing it bare.

Each worker reads `sase/memory/lint_and_test.md` and runs SASE `just check` (through
`sase tool run check`) when it changed tracked files in this repo; `core-index` also
runs the core repository's `just check` including binding tests. `panel-render` runs the
focused visual suite for its scenes. The land agent verifies the combined tree with
`just check-full` exclusively through `/sase_monitor` using the `TESTING`/`TESTED`
status pair, and rechecks every changed repository's required gates. Follow the
approved-plan and host-owned finalization workflows: no worker creates commits,
branches, or PRs.

## Memory and follow-ups

No memory-file edits are part of this plan. `sase_beads.md` documents only what
`sase bead -h` cannot, and `bead-cli` is required to make its own help complete, so no
prose update should be needed. If the land agent judges otherwise, it files a `memory`
task bead through `/sase_new_task` rather than editing memory inside this epic.

Two attribution defects found while designing this are out of scope and belong in their
own task beads, filed by the land agent through `/sase_new_task` with the evidence in
"What is already tracked, and what is not":

- `link_added` and `link_removed` events record the bead store owner as actor 100% of
  the time, so bead link writes are unattributable and the `linked` verb is dead on
  arrival.
- Bead event actors are not uniformly globalized. `note_appended` alone carries 2,904
  distinct actors mixing globalized names with bare local names, which this epic works
  around in its matcher instead of fixing at the write site.
