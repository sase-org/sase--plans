---
tier: tale
title: Group corresponding agent and bead wait indicators
goal:
  Waiting agent nodes present independent agent and bead counts in a separator-free,
  status-aligned sequence.
size: small
proposed_by: bbugyi200.athena.0au
create_time: 2026-08-22 13:41:32
status: wip
---

# Plan: Group corresponding agent and bead wait indicators

## Outcome

Make compact `WAITING` summaries in the ACE Agents tab read as one status-oriented
sequence instead of two domain groups. Remove the dim `·` between agent and bead
indicators, and when both domains have corresponding visible statuses, place the bead
token immediately after the matching agent token while retaining an independent count
beside each glyph.

For the requested example—one running agent, one done agent, one in-progress bead, and
one closed bead—the row becomes:

```text
(WAITING ▶1 ◐1 ✓1 ●1)
```

It must not become `▶2 ✓2`: agent and bead counts remain independent even when their
states correspond. Every token keeps its current glyph, count, and Rich style. Ordinary
single spaces remain between tokens; only the domain separator and its surrounding
separator-specific spacing disappear.

This is a presentation-only refinement of the designs in
`wait_dependency_status_counts`, `separate_agent_bead_wait_counts`, and
`compact_bead_wait_status_tokens`. It does not change wait satisfaction, persisted
metadata, dependency aggregation, bead status caching or warmup, selective row patching,
render-cache invalidation, Rust/core behavior, or detail-panel lanes.

## Product and ordering contract

### Corresponding status pairs

Reuse the correspondence that existed before agent and bead tallies were separated, but
use it only to order independently rendered tokens:

| Agent bucket | Agent token | Corresponding bead status | Bead token |
| ------------ | ----------- | ------------------------- | ---------- |
| `Starting`   | `◐N`        | `claimed`                 | `◎N`       |
| `Running`    | `▶N`        | `in_progress`             | `◐N`       |
| `Waiting`    | `⏳N`       | `open`                    | `○N`       |
| `Done`       | `✓N`        | `closed`                  | `●N`       |
| unknown      | `?N`        | unknown                   | `?N`       |

`Stopped`, `Failed`, and `Queued` agent buckets have no bead counterpart. The exact bead
statuses `ready` (`◇N`) and `snoozed` (`◈N`) have no agent counterpart. Do not invent a
match based only on color or lifecycle intuition.

### Stable, conditional grouping algorithm

Keep canonical agent order as the primary compact-row order. For each nonzero agent
token, append it and then immediately append its corresponding bead token when that bead
count is also nonzero. Mark that bead status as consumed. After all agent tokens, append
every unconsumed bead token in canonical bead display order.

This gives the requested “if any” behavior without hiding or merging anything:

- `▶1` plus `◐2` renders `▶1 ◐2`.
- `✓1` plus `●1` renders `✓1 ●1`.
- A corresponding bead whose agent token is absent remains visible in the trailing
  canonical bead remainder; bead-only summaries therefore preserve their current
  canonical order.
- `ready` and `snoozed` always remain in that unpaired remainder.
- Unknown agent and bead counts render as adjacent independent tokens (`?1 ?2`) when
  both exist. Their identical glyph is intentional under the requested separator-free
  contract; the counts remain separate in the model and the selected detail panel's
  `[agents]` and `[beads]` lanes remain the disambiguation surface.

Zero buckets remain suppressed and a count of `1` remains explicit. Continue to render
the sequence only on `WAITING` rows, directly after `WAITING` and before the existing
reserved-tribe `!`, relative duration, absolute time, or countdown annotation. Do not
add a replacement separator, bead-domain prefix, brackets, background chip, or extra
spacing convention.

## Implementation

### Pure presentation formatter

Refactor `src/sase/ace/tui/wait_status_presentation.py` so
`format_wait_dependency_status_counts()` emits one sequence from the existing nested
`WaitDependencyStatusCounts` value:

1. Define one explicit, presentation-owned agent-bucket-to-bead-status correspondence
   table for the five pairs above.
2. Append count tokens through small helpers that preserve the current complete-token
   Rich styles and immediate glyph/count attachment.
3. Walk the existing `WaitAgentStatusCounts.nonzero_buckets()` order, conditionally
   consuming a matching nonzero bead count after its agent count.
4. Walk `WaitBeadStatusCounts.nonzero_statuses()` and append only statuses not already
   consumed.
5. Remove the now-unused `WAIT_DOMAIN_SEPARATOR` and `WAIT_DOMAIN_SEPARATOR_STYLE`
   constants and exports.

Keep `WaitAgentStatusCounts`, `WaitBeadStatusCounts`, `WaitDependencyStatusCounts`,
`wait_dependency_status_counts()`, and the canonical agent/bead status presentation
modules unchanged. The outer nested value must continue to flow through list
construction, row context, render keys, and selective patch overrides exactly as today.
The formatter remains pure, memory-only, and free of file, store, subprocess, or
event-loop work.

### User-facing descriptions

Update the Agents help modal's existing `Waiting Badges` section and the compact-row and
detail-view explanations in `docs/ace.md`:

- replace mixed-domain `·` examples with paired examples such as `▶1 ◐2` and `✓1 ●1`;
- explain that matching bead statuses follow their present agent status, while unmatched
  bead tokens trail in bead order;
- retain agent-only, bead-only, unknown, tribe, and detail-lane legends; and
- keep the trailing gold `◆` bead-linked-agent badge explicitly distinct from waited-on
  bead status tokens.

Keep help labels within their established width and avoid implying that adjacent counts
were merged.

## Tests and visual acceptance

Update focused coverage for the following contracts:

1. The formatter removes `·` completely and produces the exact requested `▶1 ◐1 ✓1 ●1`
   sequence with separate counts and existing per-token styles.
2. Every supported correspondence is adjacent in canonical agent order:
   `Starting/claimed`, `Running/in_progress`, `Waiting/open`, `Done/closed`, and
   agent/bead unknown.
3. A bead token is consumed exactly once when paired; counts from the two domains are
   never summed, including same-glyph cases.
4. Corresponding beads with no visible matching agent, plus `ready` and `snoozed`,
   remain in the trailing canonical bead order. Agent-only and bead-only summaries stay
   byte-for-byte unchanged.
5. Zero suppression, explicit `1`, multi-digit counts, canonical colors, non-`WAITING`
   suppression, and ordering before `!`/duration/countdown annotations remain intact.
6. Existing aggregation, cache-key, warmup, cold/stale snapshot, and selective-patch
   tests continue to prove the unchanged two-domain data and refresh behavior; update
   only presentation expectations that contain the removed separator or reordered
   tokens.
7. Help and documentation assertions match the new examples and no longer describe
   positional groups separated by `·`.
8. Update only the dedicated `agents_waiting_missing_target_row_120x40.png` golden. Keep
   a mixed selected row that visibly proves running/in-progress and done/closed
   adjacency, update its plain/SVG assertion, and inspect the full-resolution PNG for
   spacing, styles, selection contrast, single-cell glyph rendering, row/detail
   agreement, and unrelated churn.

After implementation, install the ephemeral workspace and run focused formatter, row,
help, documentation, cache, patching, and waiting visual tests while iterating. Finish
with:

```bash
just install
just check
just test-visual
```

If `just check` escalates or reports unusual selection, run `just check-full` only
through `/sase_monitor` as required by project guidance. Do not accept unrelated visual
golden changes.

## Likely implementation surface

- `src/sase/ace/tui/wait_status_presentation.py` — correspondence-aware unified
  formatter and separator removal.
- `src/sase/ace/tui/modals/help_modal/agents_bindings.py` and `docs/ace.md` — updated
  visual grammar and examples.
- `tests/ace/tui/test_agent_wait_dependency_status_counts.py` and focused agent-row,
  help, render-cache, and patching assertions affected by output ordering.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_zoom.py` and the one dedicated
  waiting-agent PNG golden — visual acceptance.

Exact private helper names may vary during implementation, but conditional adjacency,
independent counts, canonical remainder order, separator removal, disk-free rendering,
and unchanged refresh/cache topology are required.

## Out of scope

- Merging agent and bead counts or changing the nested count projection.
- Changing `%wait` resolution, bead claims, lifecycle statuses, persisted fields, or
  cache/warmup behavior.
- Changing agent glyphs/colors, canonical bead glyphs/colors/order, or the detailed
  `[agents]` and `[beads]` lane rendering.
- Adding a bead prefix, replacement delimiter, feature flag, configuration option,
  filter, query syntax, persistent field, background task, or render-path I/O.
- Changing the trailing linked-bead `◆` identity badge.
