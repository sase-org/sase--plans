---
tier: tale
title: Unread-aware agent clans
goal: 'Agent clan rows continuously show a counted unread indicator, unread navigation
  reveals and selects members hidden by a collapsed clan, and interacting with the
  synthetic clan container never acknowledges its members.

  '
create_time: 2026-07-18 07:24:36
status: wip
prompt: 202607/prompts/clan_unread_navigation.md
---

# Plan: Unread-aware agent clans

## Context and outcome

Agent clans are presentation-only containers over real agents and sequential agent families. The Agents tab already
projects a synthetic clan row, folds its direct members, and renders aggregate status/runtime metadata. Separately,
completed-agent unread state follows a one-to-one contract: active completion notifications are the source of truth,
`_unread_completed_agent_ids` is a render/navigation cache keyed by real agent identity, and reading a real agent row
dismisses only that agent's matching notification.

Those systems currently meet at the wrong boundary. Fold finalization reconciles and prunes unread identities against
the fold-filtered `_agents` list, so collapsing a clan removes its members from the unread projection. The synthetic
clan identity therefore has nothing to render, and `,j` only searches rows exposed by the current in-panel folds. Clan
selection happens to be harmless today only because the container has no unread identity; the feature should make that
invariant explicit rather than relying on an accident of matching.

The finished behavior should be:

- A clan with unread direct members shows the existing gold `U<n>` metric inside its compact status chip in collapsed
  and expanded states. This is the aggregate visual language already used by Agents panel titles and the info panel;
  individual member rows keep their existing completion glyphs. Successful unread members contribute to `U`, not `D`,
  while an unread failure may contribute to both `F` and `U`, matching existing aggregate-count semantics.
- The count is the number of distinct unread real clan-member identities for that exact clan name and generation.
  Synthetic clan rows never enter `_unread_completed_agent_ids`, and no aggregate notification is created.
- `,j` keeps its current newest-first ordering. If the next target is hidden only because its clan is collapsed, the
  action expands that exact clan, selects the real agent or family row, and then applies the existing targeted
  acknowledgement behavior. Other clans and other unread agents remain untouched.
- Selecting, navigating onto, or rendering a clan row does not mark any descendant read, dismiss any descendant's
  completion notification, or change a manual unread guard. The manual `U` action on a clan is likewise a no-op because
  a synthetic container cannot own unread state.

## Unread projection and clan aggregation

Keep the notification store as the only authority for real completion unread state. Reconcile active notification keys
and session-local manual unread guards against the complete loaded, display-eligible real-agent roster (the unfiltered
`_agents_with_children` projection, with a safe `_agents` fallback for narrow test/early-lifecycle contexts), not the
list after clan/workflow fold filtering. Exclude synthetic clan identities from reconciliation and pruning. This
preserves unread state across a fold change while still removing identities that genuinely leave the loaded roster
through dismissal, filtering at load time, or artifact disappearance.

Add a small pure clan-unread projection beside the existing clan status aggregation. It should use stable real-agent
identities, exact clan generation boundaries, and deduplication; extend `ClanStatusCounts`/`clan_member_counts` to
accept unread identities and expose an `unread` metric. Feed that metric into `format_agent_count_chip` for both the
clan list row and its aggregate detail header, so the two visible surfaces cannot disagree. Do not overload the
row-level `is_unread` boolean for clans: that boolean controls the individual completion glyph and would produce an
uncounted marker.

Thread the derived count through the existing render and patch inputs. Ensure the agent render-cache key contains the
complete derived count state, including transitions such as `D1 -> U1 -> D1`, so notification polling cannot return a
stale cached clan row. Keep the calculation entirely in memory over already-attached clan members; render paths must not
read notification files, inspect artifacts, or perform other I/O.

Centralize unread repainting around the before/after identity delta. For every changed real member, patch both the
member row when visible and its visible clan container when present. A collapsed clan therefore receives a selective row
patch even though the changed member is absent from `_agents`; an expanded clan updates both rows. Refresh panel titles,
the info panel, collapsed-panel summaries, and the selected clan detail header through their existing cheap paths. Fall
back to a list rebuild only when a structural/panel/cache-width guard rejects the selective patch. Route selection
acknowledgement, manual toggles, mark-all-read, notification polling, and dismissal cleanup through this ancestor-aware
repaint policy so no mutation path leaves the aggregate count stale.

## Reveal-aware `,j` navigation

Separate candidate discovery from current outer-clan visibility. Start with the normal rendered candidates, then use the
complete loaded projection to add only direct clan members that would render if their clan's outer fold were open. Keep
family/workflow child folds, hidden-step rules, active search/group filters, terminal-status predicates, identity
matching, completion-time ordering, panel partitioning, manual-unread behavior, and wrap semantics authoritative. This
must not make an agent hidden by its own inner fold newly jumpable. Clan containers are never candidates. A candidate
outside a clan follows the current path unchanged; a candidate in an expanded clan remains a normal visible-row jump.

For a candidate hidden by a collapsed clan:

1. Save the existing Agents jump anchor before changing any structure.
2. Resolve the candidate's exact clan fold key using both clan name and generation, and expand it through the existing
   fold manager/persistence path. If its tag panel is also collapsed, retain the current panel auto-expansion behavior.
3. Refilter once through the established in-memory fast path, then resolve the target again by stable identity in the
   new `_agents` list rather than retaining a stale index across the rebuild.
4. Move panel focus and row focus to the revealed real member, clear banner/clan focus, and acknowledge only after the
   target has been successfully resolved and selected. A disappeared or no-longer-unread target must not dismiss a
   notification; fail safely and let the next refresh/retry choose from current state.

Keep this path synchronous and in memory. Do not add disk access, notification loads, subprocesses, or awaited work to
the keystroke handler. Reuse the normal refilter and detail-panel debounce behavior, invalidate navigation/panel caches
when the fold changes, and persist the programmatic expansion exactly like a manual `l` expansion so refreshes do not
immediately recollapse the clan. Back/forward jump history should continue restoring selection anchors; it need not
silently undo the user's now-visible clan expansion.

## Selection and command boundaries

Make synthetic-row behavior explicit at the narrow unread action boundary: `_acknowledge_agent_unread` and
`_toggle_agent_unread` should reject clan containers before mutating local sets or calling notification dismissal. This
protects mouse selection, ordinary `j`/`k`, jump hints, panel navigation, and future callers without duplicating special
cases in every event handler. Preserve all existing real-agent behavior, including the manual unread guard and
one-to-one dismissal by `(cl_name, raw_suffix)`.

The existing `,j` key and default configuration remain unchanged. Its help/footer wording remains accurate, but update
any focused help text or command documentation that describes the action as limited to already-visible rows. Do not add
a clan-level mark-read action: selecting a container is inspection, not acknowledgement.

## Verification and visual quality

Add focused model, interaction, and rendering coverage for:

- clan aggregation in both fold states, including `U1`/multi-unread counts, `D` exclusion for unread successes, unread
  failures, exact generation isolation, and removal of the chip when the final member is acknowledged;
- finalization and notification polling retaining fold-hidden member identities while pruning truly absent real agents,
  plus ancestor-row patching in collapsed and expanded clans without an unnecessary full rebuild;
- mouse selection, ordinary keyboard selection, jump hints, and manual `U` on a clan leaving every descendant identity
  and matching notification untouched;
- `,j` choosing the most recent unread across multiple clans, auto-expanding only the target clan (and a collapsed tag
  panel when applicable), resolving by identity after refilter, landing on both an ordinary agent and a family row,
  acknowledging only the selected target, preserving manual unread guards, and handling a stale/disappearing target;
- render-cache invalidation and aggregate detail/header consistency when unread counts change.

Add dedicated ACE PNG snapshots for a collapsed unread clan and its expanded form (or an interaction snapshot showing
the `,j` reveal) so the gold `U<n>` token, existing status metrics, indentation, selection highlight, and right-aligned
runtime column are reviewed together. Prefer these focused fixtures over changing unrelated baseline snapshots. The
count chip should remain compact at narrow widths and use the existing `U` background treatment rather than a new glyph
or color.

Run `just install` before repository checks, then the focused unread/clan/render-cache/detail tests, `just test-visual`,
and finally `just check`. Use the existing TUI trace/performance assertions where available to confirm ordinary row
acknowledgement stays on the selective patch path and `,j` incurs a structural refilter only when it actually has to
reveal a collapsed clan.

## Scope and risks

This is a TUI presentation/navigation change; no notification-store or Rust-core API change is needed. The main risks
are losing unread identities by confusing "fold-hidden" with "gone," counting across two generations that share a clan
name, acknowledging before a structural refilter successfully lands, or updating the child glyph while leaving the
ancestor chip cached. Stable identities, exact clan fold keys, loaded-roster reconciliation, post-refilter identity
resolution, and ancestor-aware selective patches address those failure modes directly.
