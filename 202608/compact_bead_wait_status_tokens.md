---
tier: tale
title: Compact bead wait status tokens
goal:
  Remove the redundant bead-domain glyph from ACE wait-status tokens while preserving
  canonical bead status glyphs, colors, independent counts, and mixed-domain ordering.
size: small
proposed_by: bbugyi200.athena.09u.f0
create_time: 2026-08-21 18:59:17
status: wip
---

# Plan: Compact Bead Wait Status Tokens

## Outcome

Make waited-on bead statuses consume one fewer terminal cell per nonzero status bucket
by rendering the canonical bead-status glyph directly, without the leading `◆`.

The current mixed summary:

```text
(WAITING ✗1 ▶1 ✓1 ?1 · ◆○1 ◆◐1 ◆●1)
```

becomes:

```text
(WAITING ✗1 ▶1 ✓1 ?1 · ○1 ◐1 ●1)
```

Apply the same visual language to the selected agent's `[beads]` detail lane, so
`run-bead ◆◐` becomes `run-bead ◐`. Keep the trailing `◆` badge on an agent launched by
`sase bead work`; that is a separate linked-bead identity marker, not a waited-on bead
status.

This is a presentation-only ACE refinement. It must not change the separate agent/bead
count model, wait satisfaction, lifecycle status, bead claims, persisted metadata, cache
warmup, selective row patching, Rust/core behavior, or event-loop work.

## Product Contract

### Status-only bead tokens

Render every known bead wait with the canonical `bead_status_presentation` TUI glyph and
Rich style, followed immediately by its decimal count in compact rows:

| Bead status   | Compact token | Detail token | Canonical color |
| ------------- | ------------- | ------------ | --------------- |
| `open`        | `○N`          | `○`          | sky             |
| `claimed`     | `◎N`          | `◎`          | amethyst        |
| `ready`       | `◇N`          | `◇`          | teal            |
| `snoozed`     | `◈N`          | `◈`          | grey            |
| `in_progress` | `◐N`          | `◐`          | gold            |
| `closed`      | `●N`          | `●`          | green           |
| unknown       | `?N`          | `?`          | amber           |

The status glyph and count remain one unbroken styled token. Preserve canonical bead
display order, zero suppression, multi-digit counts, and the always-visible count for
`1`. Do not substitute a new bead prefix, bracket, letter, or chip; the space saving is
the intended product change.

Keep agent wait tokens exactly as shipped. Keep the two data domains independently
counted even when their rendered glyphs coincide. In a mixed summary, the left group is
agents and the right group is beads, separated by the existing dim `·`:

```text
(WAITING ▶1 · ◐2)
(WAITING ◐1 · ◐2)
(WAITING ?1 · ?2 +5m)
```

The second and third examples deliberately accept that status-only glyphs can coincide
across domains; separator position carries the domain in mixed rows. A bead-only summary
such as `(WAITING ◐2)` intentionally prioritizes compactness over encoding the domain in
every token. The selected detail view remains explicit through its `[beads]` lane tag.
Document this positional convention rather than reintroducing a per-token domain marker.

Preserve all existing placement rules: dependency groups follow `WAITING`, the separator
appears only when both groups are nonempty, and the reserved-tribe `!`, relative
duration, absolute time, or countdown annotations follow the groups. Non-`WAITING` rows
must not render either group.

### Keep the linked-bead badge distinct

Retain the fixed-style gold/green `◆` identity badge rendered beside agents launched by
`sase bead work`, with its current placement and warm-cache behavior. It should become
the only `◆` involved in these Agents-row presentations.

Remove the misleading shared-wait-glyph abstraction from `wait_status_presentation.py`.
Restore or introduce a narrowly named private linked-bead glyph/style constant in the
agent-list styling layer and consume it only from the linked-agent render branch. Do not
change other uses of `◆` elsewhere in ACE.

## Implementation

1. Refactor the bead wait token helper in `src/sase/ace/tui/wait_status_presentation.py`
   to return only the canonical status glyph and style, with `?` plus the existing amber
   style as the unsupported/missing fallback. Use that helper for both compact counts
   and count-free detail badges.
2. Remove `BEAD_WAIT_GLYPH` from the wait-presentation API. Move or rename the remaining
   linked-agent `◆` constant so its name, ownership, and comment describe only that
   identity badge; update `_agent_list_render_agent.py` accordingly.
3. Leave `WaitAgentStatusCounts`, `WaitBeadStatusCounts`, `WaitDependencyStatusCounts`,
   their aggregation, cache keys, warmup snapshots, and patch overrides untouched. The
   formatter continues to receive the same immutable two-domain value and uses the same
   separator logic.
4. Update the Agents help modal and every affected `docs/ace.md` description/example.
   Use concise help examples such as `○2 ◐1`, `▶1 · ◐2`, `?1 · ?2`, and `[beads] id ◐`.
   Explain that group order disambiguates mixed rows and that the trailing linked-agent
   `◆` is not a wait token. Avoid separate duplicate `?N` help keys for agent and bead
   unknowns.

Exact private helper and constant names may vary during implementation, but the
status-only token shape, canonical styling, positional group semantics, and linked-badge
separation are required.

## Tests and Visual Acceptance

Update focused tests to cover the new presentation contract without weakening the
existing two-domain data assertions:

1. Formatter tests cover all six canonical bead statuses plus unknown, stable order,
   zero suppression, counts of one and multiple digits, and exact Rich styling across
   each complete `<status-glyph><count>` token. Assert that waited-on bead tokens
   contain no `◆`.
2. Mixed-domain tests retain independent counts and the dim separator, including
   same-glyph cases (`◐1 · ◐2`) and unknowns (`?1 · ?2`). Agent-only output remains byte
   for byte unchanged; bead-only output becomes status-only.
3. Row tests preserve annotation ordering and non-`WAITING` suppression while updating
   bead-only, mixed, and unknown expected text.
4. Detail-lane tests cover every canonical bead glyph/color plus unknown, with the
   `[beads]` tag providing domain context. Confirm `[agents]`, `[tribes]`, `[time]`, and
   `[runners]` lanes remain unchanged.
5. Render-cache and selective-patch tests continue proving that bead status transitions
   invalidate and replace rendered rows even though the displayed token is shorter.
   Warmup/cold/stale status semantics remain unchanged.
6. Help and documentation assertions use the new examples and continue distinguishing
   wait statuses from the trailing linked-bead `◆` badge.
7. Update only the dedicated `agents_waiting_missing_target_row_120x40.png` golden. Its
   selected row must contain mixed agent and bead groups with exact plain/SVG text
   `WAITING ✗1 ▶1 ✓1 ?1 · ○1 ◐1 ●1`. Inspect the full-resolution PNG for the expected
   three-cell reduction, group spacing, canonical bead colors, selection contrast,
   single-cell glyph rendering, and no unrelated visual drift. The detail `[beads]` lane
   should remain identifiable even when narrow-panel wrapping hides target values.

## Verification

Install the ephemeral workspace before running tests:

```bash
just install
```

Run the focused wait formatter/projection, row, detail, cache, patching, and help tests
while iterating. Regenerate only the intentional waiting-row golden, then rerun that
visual test without the update flag. Finish with:

```bash
just check
just test-visual
```

If `just check` escalates or reports unusual selection, run `just check-full` only
through `/sase_monitor` as required by repository guidance. Do not accept unrelated PNG
snapshot changes.

## Out of Scope

- Changing the agent/bead dependency projection or merging the two domains again.
- Changing `%wait` resolution, bead claims, lifecycle statuses, or persisted fields.
- Changing agent status glyphs/colors, canonical bead status presentation, or Beads-tab
  rendering.
- Removing or restyling the trailing linked-bead `◆` identity badge.
- Adding a feature flag, configuration option, cache, store query, timer, background
  task, startup work, or render-path I/O.
