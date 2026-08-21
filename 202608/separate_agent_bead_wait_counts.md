---
tier: tale
title: Separate agent and bead wait dependency counts
goal:
  Waiting agent rows distinguish agent dependencies from bead dependencies with compact,
  status-aware, accessible visual tokens.
size: medium
proposed_by: bbugyi200.athena.09q
create_time: 2026-08-21 11:59:44
status: wip
---

# Plan: Separate agent and bead wait dependency counts

## Outcome

Make every compact `WAITING` summary on the ACE Agents tab distinguish waited-on agents
from waited-on beads without sacrificing the existing scan-friendly status language.
Agent counts retain their current status glyphs. Bead counts become their own group,
with each token carrying both the established `◆` bead-domain glyph and the canonical
bead-status glyph, followed by the count.

For example, the current mixed summary:

```text
(WAITING ✗1 ▶2 ⏳1 ✓2 ?1)
```

becomes:

```text
(WAITING ✗1 ▶1 ✓1 ?1 · ◆○1 ◆◐1 ◆●1)
```

The left group describes agents (`✗` failed, `▶` running, `✓` done, `?` unknown), and
the right group describes beads (`◆○` open, `◆◐` in progress, `◆●` closed). The quiet
middle dot separates domains only when both are present. This removes the current
ambiguity where an agent and a bead in equivalent normalized states are merged into one
number.

This remains a presentation-only ACE change. It must not alter `%wait` satisfaction,
agent or bead lifecycle state, bead claims, persisted wait metadata, Rust/core behavior,
or the existing post-paint cache-warmup schedule.

## Product and visual contract

### Two explicit dependency domains

Render agent and bead counts independently and never combine them, even when their
states are semantically similar. For example, one running agent and two in-progress
beads render as `▶1 · ◆◐2`, not `▶3`.

Agent tokens retain the shipped mappings, colors, order, zero suppression, and unknown
behavior:

| Normalized agent state | Token | Existing color |
| ---------------------- | ----- | -------------- |
| `Stopped`              | `▲N`  | grey-purple    |
| `Failed`               | `✗N`  | red            |
| `Starting`             | `◐N`  | sky            |
| `Running`              | `▶N`  | gold           |
| `Queued`               | `…N`  | blue           |
| `Waiting`              | `⏳N` | amethyst       |
| `Done`                 | `✓N`  | green          |
| unknown/unresolved     | `?N`  | amber          |

Bead tokens use `◆` to identify the dependency domain and reuse the canonical
`bead_status_presentation` glyph and Rich color for the exact bead status:

| Bead status      | Token | Canonical color |
| ---------------- | ----- | --------------- |
| `open`           | `◆○N` | sky             |
| `claimed`        | `◆◎N` | amethyst        |
| `ready`          | `◆◇N` | teal            |
| `snoozed`        | `◆◈N` | grey            |
| `in_progress`    | `◆◐N` | gold            |
| `closed`         | `◆●N` | green           |
| missing/unmapped | `◆?N` | amber           |

The domain glyph, status glyph, and decimal count form one unbroken styled token. The
status glyph is required in addition to color: `◆◐2` stays understandable in a
monochrome export, under low-contrast selection styling, and to readers who cannot
distinguish the palette. Reusing canonical bead presentation also prevents a new local
color/status table from drifting from the Beads tab and wait picker.

Use the canonical agent bucket order for the agent group and
`bead_status_display_order()` for the bead group, with that domain's unknown token last.
Always render the number, including `1`, and suppress zero entries. Render one dim `·`
separator only when both groups are nonempty; do not render a leading/trailing separator
for agent-only or bead-only waits. Examples:

```text
(WAITING ▶2 ✓1)
(WAITING ◆○2 ◆◐1)
(WAITING ▶1 · ◆◐2)
(WAITING ?1 · ◆?2 +5m)
(WAITING ▶1 · ◆●1 !)
```

Keep all dependency tokens directly after `WAITING` and before the existing reserved-
tribe `!`, relative duration, absolute time, or countdown annotation. Do not add nested
parentheses or chip backgrounds. Continue to render dependency counts only when the
displayed row status is `WAITING`; defensive callers must not leak them onto other
statuses.

### Row/detail agreement and discoverability

Update the selected agent's `[beads]` wait lane to use the same status-bearing bead
token beside each ID, without a count: for example `run-bead ◆◐` and `done-bead ◆●`.
Keep `[agents]` badges unchanged. An unknown bead is `bead-id ◆?`. This makes the detail
panel a direct legend for the compact row instead of showing an agent-shaped glyph for a
bead.

Promote the existing Agents-row `◆` bead glyph to one shared ACE presentation constant
used by both wait tokens and the trailing "bead-linked agent" badge. The linked-agent
badge keeps its existing fixed style; only waited-on bead tokens inherit status color.

Revise the Agents help modal's `Waiting Badges` section to show distinct agent and bead
examples, explain the mixed-domain separator, and document `?N` versus `◆?N`. Keep the
legend within its established width. Add the compact wait-summary contract to the
Agents-tab documentation in `docs/ace.md`, adjacent to the `Waiting` description, and
ensure the separate bead-token notation cannot be confused with the trailing linked-
bead badge.

## Data projection

Replace the flat, cross-domain `WaitDependencyStatusCounts` tally with an immutable,
hashable projection that contains two explicit value objects:

- agent counts: the seven supported agent buckets plus `unknown`; and
- bead counts: all six canonical bead statuses plus `unknown`.

The outer dependency-count value remains the single value threaded through list build,
rendering, render-cache keys, stored row context, selective row patches, and warmup
overrides. Its equality must include both domains so any agent or bead status transition
invalidates the rendered row. Provide small pure accessors/iterators rather than making
formatters inspect dictionaries with implicit keys.

Refactor `wait_dependency_status_counts()` to maintain independent tallies:

1. Resolve through `wait_display_agent()` so a family root and its concrete waiting
   shell use the same wait source.
2. Count ordinary agent targets exactly as today, including expanded visible clan
   members and agent-specific unknowns.
3. Count each warmed bead under its exact canonical bead status instead of normalizing
   it into an agent bucket.
4. Count an authoritative missing or unsupported bead status under bead `unknown`, not
   agent `unknown`.
5. Omit a cold bead-cache miss until the existing worker warms it; cold is still not
   evidence of an unknown bead.
6. Return empty domain values for rows without ordinary agent or bead waits.

Continue to exclude tribe references, runner-capacity waits, and time-only waits from
both count domains. Preserve clan-member expansion, ordinary family aggregation, and all
existing unknown-agent semantics.

Remove the bead-to-agent-bucket normalization from the shared wait presentation module;
the exact bead status is now the source of truth. Do not add store access or filesystem
work to the projection, list construction, formatter, render cache, keystroke path, or
Textual message pump.

## Cache warmup and selective rendering

Keep `WaitBeadStatusSnapshot`'s existing cold/concrete/authoritative-unknown distinction
and keep the current project-batched, deduplicated, off-thread warmup unchanged. The new
projection should consume the same snapshots, so this feature adds no second cache,
timer, task, store query, or startup work.

When bead statuses warm or revalidate, preserve the existing apply contract:

- rematch identities against the current agent snapshot after the await;
- compute one current in-memory `AgentWaitStatusMaps` for the apply batch;
- derive a fresh two-domain count override for each affected row;
- patch only affected rows and update `_row_render_ctx`; and
- request one ordinary display rebuild after the batch if any row grows beyond the
  cached aligned width or fails another conservative patch gate.

The complete nested count value must remain in `agent_render_key()`. A transition such
as `◆○1` to `◆●1`, a cold-to-known bead, an agent status change, or either domain's
unknown count changing must bust the cache even when the waiting agent record is
otherwise unchanged.

## Tests and visual acceptance

Add or update focused coverage for all of the following:

1. Pure aggregation keeps agent and bead counts separate when their statuses would have
   merged previously, covers all seven agent buckets and all six canonical bead
   statuses, and assigns unknown agents and unknown beads to different domains.
2. Formatting uses stable canonical order, suppresses zeroes, always displays `1`,
   handles multi-digit counts, emits the dim separator only for mixed domains, and
   applies the exact canonical Rich style across each complete `◆<status><count>` bead
   token.
3. Bead tokens remain semantically readable from plain text: `◆` identifies the domain,
   the second glyph identifies exact status, and `◆?N` is distinct from agent `?N`.
4. Cold bead-cache misses remain omitted, authoritative missing/unmapped statuses render
   `◆?N`, stale statuses stay visible during revalidation, and all current batched
   warmup/deduplication/error-containment tests continue to pass.
5. Clan expansion, ordinary family aggregation, `wait_display_source`, tribe exclusion,
   runner/time-only exclusion, and the ordering of `!`/duration/countdown annotations
   remain unchanged.
6. Non-`WAITING` rows never render either domain. Agent-only rows preserve their exact
   current output, while bead-only rows no longer disappear after warmup.
7. Render-key/cache tests prove independent agent and bead transitions invalidate a row.
   Selective-patch tests prove a fresh nested bead count override replaces stale row
   context and the width-growth path still requests only one rebuild per apply batch.
8. Metadata wait-lane tests cover every canonical bead glyph/color plus unknown and
   prove the unchanged `[agents]`, `[tribes]`, `[time]`, and `[runners]` lanes do not
   regress.
9. Update the dedicated waiting-agent PNG fixture so its selected row contains both
   agent and warmed bead dependencies, including at least one semantically similar
   cross-domain pair. Assert exact plain/SVG tokens such as `▶1 · ◆◐1`, regenerate only
   the intentional golden, and inspect the PNG at full resolution for group spacing,
   color hierarchy, selection contrast, single-cell glyph rendering, row/detail
   agreement, and absence of unrelated golden churn.
10. Add focused help-modal and documentation assertions where existing test structure
    supports them, so the user-facing legend stays synchronized with the formatter.

After implementation, run focused aggregation, row-rendering, cache, warmup, metadata,
help, and waiting visual tests while iterating. Then run:

```bash
just install
just check
just test-visual
```

If `just check` escalates or reports unusual selection, follow repository guidance and
run `just check-full` through `/sase_monitor`. Do not accept unrelated visual snapshot
changes.

## Likely implementation surface

- `src/sase/ace/tui/_agent_completion_wait.py` and
  `src/sase/ace/tui/agent_completion.py` — separate immutable domain counts and the
  memory-only projection.
- `src/sase/ace/tui/wait_status_presentation.py` — shared `◆` glyph, independent agent
  and bead formatters, canonical bead-status presentation, separator, and unknown
  tokens.
- `src/sase/ace/tui/widgets/_agent_list_styling.py` and `_agent_list_render_agent.py` —
  consume the shared glyph/formatter while preserving linked-bead placement and wait
  annotation order.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py` — render exact
  status-bearing bead tokens in the detail lane.
- Existing list build/cache/patch and bead-warmup modules — thread and compare the new
  nested value without changing task or I/O topology.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` and `docs/ace.md` — explain
  the two-domain visual language.
- Existing focused wait projection, row rendering, render-cache, patching, bead warmup,
  metadata, help, and waiting-agent visual tests plus the intentional PNG golden.

Exact private helper names may change during implementation, but the two-domain data
model, shape-plus-color bead tokens, canonical status reuse, memory-only render path,
cache invalidation, and selective-update behavior are required.

## Out of scope

- Changing `%wait` resolution, bead claims, or what statuses satisfy a bead wait.
- Changing agent status glyphs/colors or the detailed agent/tribe status badges.
- Counting tribe, runner, or time waits as agent or bead dependencies.
- Changing the trailing linked-bead badge's placement or fixed style.
- Adding persistent fields, wire fields, configuration, filters, feature flags, or new
  refresh tasks.
- Reading bead stores during startup, list construction, row rendering, navigation, or
  any Textual message/timer callback.
