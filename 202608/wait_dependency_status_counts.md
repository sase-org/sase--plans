---
tier: tale
title: Count waited-on statuses in agent nodes
goal:
  Waiting agent nodes summarize the live status counts of their agent and bead
  dependencies without compromising first-paint or navigation responsiveness.
size: medium
proposed_by: bbugyi200.athena.08l
create_time: 2026-08-20 10:57:16
status: wip
---

# Plan: Count waited-on agent and bead statuses in agent nodes

## Outcome

Give every `WAITING` row in the ACE Agents tab the same dependency-status visibility
that already exists in the selected agent's `Wait:` metadata. Instead of showing only
the current bare `?` when an ordinary agent target is unknown, the row will render a
compact, colored, zero-suppressed count for every represented waited-on agent or bead
status. For example:

```text
visual-waiter (WAITING ▲1 ✗1 ▶2 ✓3 ?1)
```

The row remains a summary; selecting it remains the way to see names, `[agents]` and
`[beads]` lanes, clan-member expansion, timing, and the reason for any unknown target.
The summary should let a user scan a panel and immediately distinguish work that is
progressing, work that completed, and waits that need attention.

This is a presentation-only ACE change. It does not alter wait satisfaction, agent or
bead lifecycle state, completion candidates, persisted wait metadata, or Rust/core
behavior.

## Product and visual contract

### One shared status language

Use the established agent wait-badge mappings, glyphs, and colors rather than inventing
a second legend:

| Normalized status | Glyph | Existing color | Waited-on bead statuses mapped here         |
| ----------------- | ----- | -------------- | ------------------------------------------- |
| `Stopped`         | `▲`   | grey-purple    | —                                           |
| `Failed`          | `✗`   | red            | —                                           |
| `Starting`        | `◐`   | sky            | `claimed`                                   |
| `Running`         | `▶`   | gold           | `in_progress`                               |
| `Queued`          | `…`   | blue           | —                                           |
| `Waiting`         | `⏳`  | amethyst       | `open`                                      |
| `Done`            | `✓`   | green          | `closed`                                    |
| unknown/unmapped  | `?`   | amber          | missing, unavailable, or unsupported status |

Render nonzero counts in the canonical `AGENT_STATUS_BUCKETS` order shown above, then
unknown last. A fixed order prevents badges from jumping around when dependencies change
status and matches the Agents tab's existing grouping vocabulary. Render each token as
the colored glyph immediately followed by its colored decimal count (`▶2`), with one
ordinary space between tokens. Do not add another bracket/chip boundary inside the row's
existing status parentheses.

Always render the number, including `1`. The existing `(WAITING ?)` therefore becomes
`(WAITING ?1)`: a count is unambiguous, scales naturally to multiple dependencies, and
keeps every token visually consistent. Suppress zero buckets entirely.

Place the count tokens directly after `WAITING`, before the existing reserved-tribe `!`,
relative-duration (`+5m`), absolute-time, or countdown annotation. Examples:

```text
(WAITING ▶2 ✓1)
(WAITING ✗1 ?2 +5m)
(WAITING ▶1 !)
```

### What is counted

Count only the entities represented by the selected detail panel's `[agents]` and
`[beads]` wait lanes:

- An ordinary named agent/family target contributes one count using the same effective
  bucket already used for its per-name metadata badge.
- A visible clan target contributes each member shown by the metadata's existing
  `all clan members` expansion, rather than collapsing the whole clan to one status.
  This makes the node total agree with the concrete agents the wait is gating.
- A bead ID contributes one count after its cached status is available, normalized by
  the existing bead-to-wait-badge mapping in the table above.
- An ordinary name absent from the current full in-memory agent snapshot contributes to
  `?`. A bead lookup that authoritatively resolves missing/unavailable, or returns a
  status for which the current wait metadata has no supported badge, also contributes to
  `?`.
- A cold bead-cache miss is not evidence that the bead is unknown. Omit that bead from
  the first-paint summary, resolve it in the existing post-paint worker, then patch the
  row. Expired entries keep their stale value while revalidation runs, matching the
  established TUI cache policy.

Do not count tribe references. Bound and pending tribe waits are a different `[tribes]`
lane, and reserved/unresolvable tribes keep their existing red `!` node marker. Runner
capacity waits and time-only waits likewise keep their current compact annotations and
do not manufacture dependency counts.

For family roots that mirror a waiting child through `wait_display_source`, use that
same source for agent names, bead IDs, project identity, and counts. The family root and
its concrete waiting shell must not disagree.

### Shared presentation source

Extract the current private agent-bucket and bead-status badge tables from
`widgets/prompt_panel/_agent_wait_section.py` into a small shared ACE TUI presentation
module. That module should own:

- the canonical bucket/status-to-glyph-and-style mappings;
- the bead-status-to-agent-bucket normalization used by both surfaces;
- the fixed count display order; and
- a pure formatter for a dependency-count value object.

Keep this module presentation-only and disk-free. Refactor the metadata `Wait:` renderer
to consume the shared mappings without changing its existing per-name/per-bead output.
This makes the metadata panel and row summary incapable of silently drifting to
different glyphs or colors.

## Data projection and rendering design

### Pure dependency-count projection

Add a frozen, explicit count value object (one field per supported agent bucket plus
`unknown`) and a pure helper alongside the existing wait dependency aggregation in
`src/sase/ace/tui/_agent_completion_wait.py`. Given one agent, the already-computed
`AgentWaitStatusMaps`, and an optional memory-only bead-status snapshot, the helper
should:

1. resolve through `wait_display_agent()`;
2. walk ordinary agent targets, using `buckets` for normal targets and
   `clan_member_statuses` for expanded clans;
3. normalize warmed bead statuses through the shared mapping;
4. count missing ordinary names and authoritative unknown/unmapped bead statuses as
   `unknown`; and
5. return an all-zero value for rows without agent/bead waits.

Reuse the one `AgentWaitStatusMaps` instance that `build_list()` already computes for
the complete loaded snapshot. Do not scan files, query bead stores, or rebuild global
maps in `format_agent_option()` or any keystroke/render path.

Thread the resulting value through `build_list()`, `format_agent_option()`,
`cached_format_agent_option()`, `agent_render_key()`, the stored per-row render context,
and `patch_row()`. Replace the boolean `has_missing_wait_target` rendering input with
the richer counts: unknown ordinary names are now simply the `unknown` bucket. Keep the
separate `has_unresolvable_wait_target` input for the tribe `!` contract.

Include the complete count value in the render-cache key. A dependency moving from
running to done, a bead becoming closed, or the unknown count changing must invalidate
the row even when the waiting agent record itself is otherwise unchanged. Continue to
render counts only when the row's displayed status is `WAITING`; defensive callers that
pass counts to another status must not leak wait badges.

### Memory-only bead snapshots and batched warmup

Extend `models/agent_wait_beads.py` with a public memory-only cache snapshot that
distinguishes these three cases for every waited-on bead:

- cold cache miss (omit for now);
- cached concrete status (count it); and
- cached authoritative `None` (count it as unknown).

Keep `resolve_wait_bead_statuses()` compatible for detail-header enrichment, but factor
the cache/store work so a new batch warmup can resolve all stale waited-on bead IDs for
the visible waiting rows. Group candidates by project and deduplicate bead IDs so one
warmup performs at most one store query per project, not one query per row. Return the
agent identities whose cache-visible bead projection actually changed, including
miss-to-known, status transitions, and known-to-missing transitions.

Reuse `AgentBeadWarmupMixin` and its existing post-first-paint, pump-free, navigation-
gated, coalesced task. Generalize its candidate scan and worker body to warm both the
existing inferred agent-bead display cache and the waited-on bead-status cache. Do not
add a second timer/task family or delay startup/agent loading. Preserve the existing
fresh-cache and negative-cache TTL behavior, coalescing guards, trailing request, task
registry, exception containment, and teardown semantics.

When warmed wait-bead results return:

- match identities against the current agent snapshot, since an agent refresh may have
  interleaved with the worker;
- build one current in-memory `AgentWaitStatusMaps` snapshot for the whole apply batch;
- recompute each changed row's counts from its current wait source and warmed cache;
- pass those counts as an optional override through `_try_patch_agent_row()` and
  `AgentList.patch_agent_row()`, updating the stored render context before rendering;
  and
- patch only affected rows. If any patch cannot fit the existing aligned width (or fails
  another existing conservative patch gate), request exactly one normal display rebuild
  after the batch so the badge cannot remain absent indefinitely.

This selective-first/fallback-once behavior is required for both responsiveness and
reliability: warming usually touches a handful of rows, but adding the first count token
can legitimately increase the panel's measured width.

## Help and discoverability

Update the Agents help modal's existing `Waiting Badges` section rather than adding a
new section. Make it clear that the glyph is the dependency state and the adjacent
number is the number of waited-on agents/beads in that state; keep `?` documented as
unknown/unresolved. Include the queued `…` glyph, which the current help line omits.
Keep descriptions within the help modal's established width limits.

The metadata panel remains the explanation surface: names keep their calm magenta value
style, and individual glyphs remain attached to names. Do not status-tint whole names or
add labels/tooltips to the compact agent row.

## Tests and visual acceptance

Add or update focused tests for all of the following:

1. Pure aggregation covers every agent bucket, mixed agents and beads, merged counts
   when both domains normalize to the same bucket, multiple unknowns, zero suppression,
   stable canonical order, and multi-digit counts.
2. Clan waits count the expanded member statuses, while ordinary family/name targets use
   their established aggregate bucket. `wait_display_source` makes a family root match
   its waiting child.
3. Tribe references, runner waits, and time-only waits do not enter the count summary;
   the reserved-tribe `!` and duration/countdown layouts remain correctly ordered.
4. Row rendering emits each glyph/count with the established Rich style, always shows
   `1`, suppresses absent buckets, and never renders counts on non-`WAITING` rows.
5. The row render key/cache invalidates when counts change. Full list rebuilds pick up
   changes in agent dependency status and missing-target resolution, and selective row
   patches consume a fresh count override rather than stale `_row_render_ctx` data.
6. The bead cache snapshot distinguishes cold from authoritative unknown, retains stale
   values during TTL revalidation, batches/deduplicates lookup IDs by project, handles
   missing stores without raising, and reports only identities whose visible status
   projection changed.
7. The generalized bead warmup remains coalesced, runs store access off the Textual
   event loop, respects the navigation gate and interleaved-refresh identity lookup,
   patches only affected rows, and performs one rebuild fallback for any width-growth
   failures. Preserve or extend the existing pump-responsiveness regression test.
8. Metadata-header tests prove that extracting the shared presentation mapping leaves
   all existing per-agent, per-clan-member, and per-bead badges and styles unchanged.
9. Update the dedicated waiting-agent PNG fixture to include a visually balanced mix of
   known agent states, an unknown agent, and warmed bead states. Make at least one
   agent+bead pair share a normalized status so the displayed count proves cross-domain
   aggregation. Assert the key plain/SVG tokens, regenerate the intentional golden, and
   inspect the PNG at full resolution for spacing, color hierarchy, single-cell glyph
   rendering, selection contrast, and row/detail agreement.

After implementation, run focused unit/integration tests while iterating, then:

```bash
just install
just check
just test-visual
```

If `just check` escalates or reports unusual selection, follow project guidance and run
`just check-full` through `/sase_monitor`. Do not accept unrelated visual golden churn.

## Likely implementation surface

- `src/sase/ace/tui/_agent_completion_wait.py` and
  `src/sase/ace/tui/agent_completion.py` — count model/projection and public facade.
- A focused shared module under `src/sase/ace/tui/` — canonical wait status presentation
  and count formatting.
- `src/sase/ace/tui/models/agent_wait_beads.py` — tri-state memory snapshot and
  project-batched warmup.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_wait_section.py` — consume the shared
  presentation contract without changing detailed output.
- `src/sase/ace/tui/widgets/_agent_list_build.py`, `_agent_list_render_agent.py`,
  `_agent_list_render_cache.py`, and `agent_list.py` — compute, render, cache, retain,
  and selectively refresh counts.
- `src/sase/ace/tui/actions/agents/_loading_bead_warmup.py` and
  `_display_panel_patches.py` — warm statuses after paint and apply current count
  overrides with a one-rebuild fallback.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` — counted-badge legend.
- Existing focused wait rendering/cache/build/warmup tests plus the dedicated waiting
  visual fixture and golden.

Exact private helper names may change during implementation, but the shared mapping,
disk-free render path, post-paint batching, cache invalidation, and selective-update
contracts are required.

## Out of scope

- Changing how `%wait` determines satisfaction or resumes an agent.
- Adding statuses or glyphs beyond those the existing `Wait:` metadata already supports;
  unsupported bead statuses continue to use `?`.
- Summarizing `[tribes]`, `[runners]`, or `[time]` lanes as dependency counts.
- Adding new persistent fields, wire fields, feature flags, configuration, filters, or
  query syntax.
- Changing agent/bead names or the expanded detail layout.
- Performing bead-store reads during startup, list construction, row rendering,
  navigation, or any Textual message/timer callback.
